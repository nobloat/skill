---
name: nobloat-outbox
description: The durable file outbox for everything that leaves a nobloat system (SMTP mail, Slack/Mattermost/generic webhooks, pushes to other APIs): record format, same-transaction enqueue, frozen vs send-time values, worker loop, ordering per key, stdlib SMTP and MIME (text+HTML, attachments, inline logo), retry classification, backoff, give-up, delivery status, bulk rate limits, idempotency, -dev and nomail, metrics, tests, privacy. Use when planning, building or reviewing anything that sends mail or calls out to another system.
---

# nobloat-outbox

Read with `nobloat` and `nobloat-storage`. Distilled from auth (`outbox.go`, `mail.go`: text+HTML mail), bill (`outbox.go`: attachments, metrics, digest), tasq (`outbox.go`, `sync.go`: pushes, ordering per key, retry classes, PLAN addendum #20) and up (`notify*.go`: notifier registry, Slack/ilert formats). Copy the named functions, then adapt; decide every point below in the plan's integration section.

## Shape

- One file per pending send: `outbox/<id>.json` in the data dir, written atomically like any object (temp + fsync + rename), mode 0600 (records may hold reset links or customer text).
- No queue server, no goroutine per message, no in-memory queue. The directory is the queue; `ls outbox/` is the debugger; backup includes it.
- Remove the file on success; on give-up move it to `outbox/failed/<id>.json` (kept for the operator, who may move it back to retry). A record in `outbox/` is pending; nothing else marks state.

## Record

Copy tasq's `OutboxRecord`, add the payload of your kinds. Use these field names (the apps differ: auth `next`/`last_error`, bill `next_try`/`last_error`, tasq `next`/`error`; take tasq's).

```go
type OutboxRecord struct {
	ID       string    `json:"id"`      // newID(), the file name; also the Message-ID / idempotency key
	Kind     string    `json:"kind"`    // mail | hook | push
	Key      string    `json:"key"`     // ordering key: ticket ZK-000000042, task zk-42, issue id
	Created  time.Time `json:"created"`
	Attempts int       `json:"attempts"`
	Next     time.Time `json:"next"`    // not before
	Error    string    `json:"error"`   // last error, for the UI and `check`
	Mail     *Mail     `json:"mail,omitempty"`
	Hook     *Hook     `json:"hook,omitempty"` // {target: config name, body: json.RawMessage}
}
```

- Read strictly (`decodeStrict`), skip `.tmp-*` and names that fail the id regexp (tasq `readOutbox`). A record that does not parse is logged at ERROR, moved to `failed/` and reported by `check`; it never stops the app or the other records (see `nobloat-storage`, Reads and startup).
- Never store credentials or endpoint URLs in a record: store the config name (`desk`, `notifier`) and look up the SMTP URL or webhook URL at send time, so a rotated secret applies to pending records.

## Enqueue in the same transaction

- The record is written in the same multi-file commit as the change that causes it (tasq: `st.commit(transaction{Files: {task, outbox/<id>}})`, addendum #14). Crash before commit: neither exists; after: both.
- Writing the record separately (before or after the object) is a defect: a failed save can still send the mail, a crash after the change can lose it.
- After the commit, wake the worker with a non-blocking send on a `chan struct{}` of size 1 (all three apps).

## Frozen payload or values read at send time

| Freeze the payload in the record | Read at send time |
|---|---|
| Anything a human wrote, saw or approved, or that the audit must match: agent replies, confirmations, invoice mail text (auth, bill, asq, gazette) | Mirrors of current state where the latest value wins: issue title/body/state (tasq stores field names, not values) |
| Rendered subject, text and HTML (render once at enqueue, so a template edit never changes a queued mail) | Large immutable blobs, referenced by path: bill attachments `files/<doc>/<vN>.pdf`, asq ticket attachments, gazette issue body (frozen once the issue is sent) |

- Send-time records whose object is gone or already in the target state are dropped as obsolete (tasq `send`: deleted task, create with ext set). Mail is never obsolete except by max age.

## Config

- SMTP as one URL, validated at startup and by `-check`: `smtp://user:pass@host:587` (STARTTLS required), `smtps://…:465` (implicit TLS), `smtp+insecure://host:1025` (plain, the local nomail sink only; refuse credentials in it). Default ports by scheme (`cmp.Or(u.Port(), …)`, auth/bill). up uses `host`/`port`/`tls` fields instead; new apps use the URL.
- `mail_from` as an address; the display name is the brand or tenant (`mail.Address{Name, Address}.String()`, which RFC 2047-encodes; auth). The envelope sender is `mail_from`.
- Per tenant (asq desks, bill companies): `from` per tenant, and optionally `smtp_url` per tenant in its `.conf`, falling back to the global one. The record names the tenant; nothing is copied.
- `base_url` is required as soon as anything is sent: every message links back to its object (except bill's customer mails, which never link into bill).

## Worker

- One goroutine started in `serve`, `go a.outboxLoop(ctx)`. It drains on start (resume after restart), after each wake, and when the next record is due. Copy tasq's `outboxLoop`/`drainOutbox`: it returns the wait until the next due record instead of polling every 30 s (auth/bill).
- Order: oldest `Created` first, ties by kind then id. **Per key in order**: once a record of a key fails or is not yet due, skip the later records of that key in this round (tasq `blocked[rec.Key]`). Mail replies on one ticket therefore arrive in order; different keys never block each other.
- Shutdown (`ctx` done) during a send is not an attempt (tasq `sendFailed`).

## Sending SMTP (stdlib)

Copy auth's `sendMail` (bill's is the same): `net.Dialer{Timeout: 20s}`, `tls.DialWithDialer` for smtps, `conn.SetDeadline(1 min)` for the whole session, `smtp.NewClient`, `StartTLS` required for `smtp://` (fail if not offered), `Auth(smtp.PlainAuth)` only over TLS (`net/smtp` refuses PLAIN on plaintext except to localhost; up tries AUTH even with `tls = none`), `Mail`, one `Rcpt` per recipient, `Data`, `Quit`. Inject a `send func(*OutboxRecord) error` hook for tests only (auth `a.send`, bill `a.sendFunc`).

## MIME (stdlib)

Extend auth's `buildMessage`; build with `mime/multipart`, `mime`, `mime/quotedprintable`, `net/textproto`.

- Structure: `multipart/mixed` { `multipart/alternative` { `text/plain`, `multipart/related` { `text/html`, logo `image/png` with `Content-ID: <logo>` referenced as `cid:logo` } }, attachments }. Omit layers you do not need (auth: alternative only; bill: mixed with text only).
- Text parts: UTF-8, `quoted-printable` (readable source, fewer spam points); attachments and images: base64 in 76-char lines (bill `writeB64`). auth/bill use base64 for text too; bill's doc comment says quoted-printable but the code writes base64.
- Headers: `From`, `To`, `Subject` via `mime.QEncoding.Encode("utf-8", …)`, `Date` (`time.RFC1123Z`), `MIME-Version: 1.0`, `Message-ID: <record-id@from-domain>` (auth/bill use `@nobloat-auth`/`@nobloat-bill`; use the From domain). Reject CR/LF in any header value, at enqueue and again in `buildMessage`.
- `Auto-Submitted: auto-generated` on notices and confirmations (none of the apps set it; it stops auto-reply loops, which matters for asq's mailbox). Human agent replies carry no `Auto-Submitted` but `In-Reply-To` and `References` with the customer's Message-IDs, so the thread stays together on both sides.
- Bulk (gazette): `List-Unsubscribe: <https://…/u/<token>>`, `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058), `List-Id`, `Precedence: bulk`.

## HTML and text from one source

- Render both from the same data at enqueue (auth `renderMail`: greeting, paragraphs, optional link, brand line, color bar in the brand color). Never derive text from HTML by stripping tags.
- Customizable template (asq desks): one `html/template` file per tenant with a fixed data struct (brand, color, logo, paragraphs or rendered body, link, footer), inline styles only (clients drop `<style>`), max width ~34rem, the logo as `cid:` (no remote images; they are blocked and track). The text part uses a fixed built-in layout.
- asq markdown replies: HTML from the app's markdown renderer (escaped, sanitized), text = the markdown source, which reads fine as is.

## Retry, backoff, give-up

| Outcome | Action |
|---|---|
| Network, DNS, TLS, timeout, SMTP `4xx`, HTTP `5xx` / `408` / `429` | retry with backoff |
| any SMTP `5xx` (bad recipient, rejected content, or our `AUTH`/`MAIL FROM` refused), HTTP other `4xx` | give up (to `failed/`, so a config error loses nothing: fix it, move the records back) |
| HTTP `401`/`403` (our key is wrong, not the message) | retry and alert, until the max age |
| HTTP `404`/`410` on update of a remote object | mark the object (tasq `ext.gone`), drop the record |

- SMTP codes come from `errors.As(err, &*textproto.Error)`; note the stage in the error. HTTP: tasq `sendFailed` with a typed `remoteError{Status}`. Nothing retries forever.
- Backoff 1 min doubling to 1 h, `backoff(attempts)` (identical in all three).
- Max age: give up when `Created` is older than 3 days, or than the link's validity if shorter (a 1-hour reset link is dead after 1 h).
- Give-up: ERROR log (id, key, attempts, error), an audit record (`<kind>.failed` as tasq `sync.error`), a visible status on the object (ticket message, subscriber, invoice, project page), move the record to `failed/`, count it.

## Delivery status

- The object carries the latest delivery state, e.g. `mail: {id, state: queued|sent|failed, at, error}` on an asq message or a bill doc. Writing it is bookkeeping: no `rev` or `updated` bump, so it never causes a 409 (tasq `ext.synced`, addendum #18).
- bill sets `sent_to` at enqueue and never records delivery; new apps write `queued` in the enqueue transaction and `sent`/`failed` from the worker.

## Bulk and rate limits (gazette)

- One record per recipient (`key` = subscriber id, payload = issue id + address + unsubscribe token; body read from the frozen issue file). Per-recipient status, purge and retry come free; a 10k list is 10k small files, fine.
- `send_rate` per minute in config (the relay's limit); the worker waits `time.Minute/send_rate` between bulk sends. Transactional records (double opt-in confirmations) go first and are not rate limited. `// ponytail:` one SMTP connection per mail; reuse one connection with `RSET` once the rate exceeds about one per second.

## Idempotency

- At-least-once: a crash between a successful send and the remove sends again (auth/bill log "sent but not removed, it will be sent again"). Accept it and make duplicates harmless:
- Mail: the stable `Message-ID` = record id; most clients and archives fold duplicates.
- Webhooks: put the record id in the body (`"id"`) or `Idempotency-Key` header; Slack/Mattermost just show a second post.
- Creates on other APIs: tasq may create a duplicate issue if it crashes after the create and before its commit; name that limit in a `ponytail:` comment.

## Webhooks

- Slack and Mattermost incoming webhooks: `{"text": "…"}` is enough for both; add `username`, `icon_url` and `attachments` with the palette colors if wanted (up `notify_webhook.go`). Generic JSON: `{"id","event","key","url","at",…}`, documented in the contract file. ilert-style APIs get their own file behind the registry (up `notifierTypes` map + `validate`/`send` interface), one file per target type.
- `http.Client{Timeout: 30 s}`, `Content-Type: application/json`, configured extra headers (up `httpSend`). A webhook URL is a secret: log the host only (`req.URL.Host`, as up), never the URL, headers or body.
- State alerts where only the latest state matters (up: red/green per monitor) need no outbox: up keeps the last reported state per notifier on disk and resends the diff on the next evaluation until it succeeds.

## Dev and local testing

- `-dev` without `smtp_url`: log the mail at INFO instead of sending, then remove the record (auth). Without `-dev` and without `smtp_url`, records stay pending (until the max age) and `-check` warns.
- Local stack: [nobloat/nomail](https://github.com/nobloat/nomail) in compose (`smtp+insecure://nomail:1025`, IMAP `nomail:1143`, UI and API on 8025, `wait=` long poll for test assertions), `make nomail` (shared `nobloat-nomail` container, bound to `127.0.0.1`) for `make run`. nomail holds every mail of the stack, auth's reset and activation links included, so it is an account-takeover tool for whoever reaches it: local only, never published, apps reach only its SMTP port, `NOMAIL_USERS` set, its UI and API behind auth (see `nobloat-system`, Local stack); webhooks go to a local echo endpoint (a `-dev` route or any request bin container).

## Metrics

Hand-written, behind `metrics_key` (bill `metrics.go` has the full set; auth only pending):

- `<app>_outbox_pending` gauge, `<app>_outbox_oldest_seconds` gauge (alert on this), `<app>_outbox_sent_total`, `<app>_outbox_failed_total` (attempts), `<app>_outbox_given_up_total` counters (records moved to `failed/`). Label by `kind` at most; never by recipient or tenant address.
- tasq's HELP/TYPE say `tasq_outbox_pending` but the sample line is `tasks_outbox_pending`: test the exact metric names.

## Tests

- `synctest`: enqueue, failing send hook, advance 1 + 2 min, assert sent once and attempts counted; channels (wake) created inside the bubble (bill `TestOutboxRetryAndDigest`).
- Real SMTP path against a fake server on the in-memory listener (up `fakeSMTP`, `textproto` answering 220/250/354): assert STARTTLS refusal, headers, and parse the message back with `net/mail` + `mime/multipart` (both parts, encoded subject, attachment bytes).
- Restart resume: enqueue, stop, start, drained (tasq sync_test). Ordering per key, give-up per class, max age, obsolete drop, status written without `rev` bump.

## Privacy

- Logs carry record id, key, kind, attempts and error, never recipient addresses, subjects or bodies. The `-dev` mail log is the only exception.
- GDPR erasure of a person also deletes their pending and failed records (scan `outbox/` and `outbox/failed/` for the address) and their delivery history; the audit keeps the record id, not the address.

## Not done

- No queue server, broker or database; no worker pool.
- No DKIM, SPF or DMARC handling in the app: the relay signs, the README tells the operator to set it up.
- No bounce or complaint processing in the outbox: a 5xx at send time is the only bounce it sees. Asynchronous bounces arrive by mail (asq's IMAP poller or gazette's bounce mailbox) and are handled there.
- No provider HTTP APIs for mail (SES, Mailgun); plain SMTP works with all of them.
- No open or click tracking.
