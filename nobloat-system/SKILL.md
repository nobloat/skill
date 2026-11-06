---
name: nobloat-system
description: Building a whole nobloat system of several cooperating deployables (e.g. nobloat/bill + nobloat/track) end to end in one pass: service integrations, money and documents, own renderers instead of libraries, security baseline, metrics, the local auth stack, verification with real tools, the ponytail/code/security review loop and a recorded demo video. Use together with the nobloat skill when a plan spans more than one service, handles money or documents, or when asked to build, verify, review or demo such a system.
---

# nobloat-system

Extends the `nobloat` skill (read it first) with what building bill + track in one session needed beyond a single tool. Everything here was decided or learned the hard way there.

## Shape

- One repo per deployable, **no shared code**: copy store/auth/audit/i18n/strict-JSON helpers instead of a shared module. Name the duplication in the ponytail review as intentional.
- The integration lives in the plan of the depending service: who calls whom, exact requests, failure and retry behavior, sequence sketch. The other side stays ignorant (track knows nothing about invoices or money).
- The local stack lives in [nobloat/system](https://github.com/nobloat/system): every service is a submodule, `make local` builds them from there, `make local SRC=..` from the sibling checkouts.

## Integrations between services

- **One direction.** The dependent service calls; the other never calls back.
- **Identity:** a service user in nobloat/auth (no password, a seeded key) whose groups decide what it may reach. No special case in code (see `nobloat-auth`).
- **Idempotent marks:** the caller sends a stable ref (`bill:<doc id>`) with the `{id: rev}` it read. All-or-nothing under the locks of every affected file; `409 {"error":"entries_changed","entries":[…]}` on any stale rev, foreign ref or invisible item; the same ref again succeeds. Marking does **not** bump `rev`, or retries break.
- **Crash windows:** do the remote call inside the local transaction (lock held, ponytail-comment the limit), write locally after it succeeds, and make re-pulls accept items already marked with the caller's own ref. `check` lists half-done states.
- **Unavailable = refuse,** never "finalize anyway". Undo paths (storno) release the marks first.
- **Provenance:** data that only the server may produce (e.g. a line's pulled track entries) must not be accepted from clients. Accept it only when it equals what the server produced before (`422 invalid_track`).
- Call the other service through caddy (`https://track.<system>.localhost`), never the container directly, so auth checks the key. Caddy's local CA root is `0600`: copy it readable in a one-shot container and point `SSL_CERT_FILE` at it; give caddy network aliases for the hostnames.

## Money and documents

- Integers only: cents, quantities in hundredths, VAT in basis points. Round half away from zero so negation (storno) is exact. VAT per rate group; per-line shares get the remainder on the largest line. A table test with the nasty cases plus a symmetry check.
- **Gap-free numbers:** drafts have none; finalize takes the series lock, computes `max(counter, start)+1`, writes the document, then the counter. Startup reconciles counters from documents (crash between the writes) and checks uniqueness. Test with 100 concurrent finalizes.
- Finalize freezes a snapshot of every referenced object into the document; afterwards only status, payments, archive and new file versions. Never delete finalized documents: cancel with a credit note in its own series.
- Derived states (paid, partial, overdue) are computed, not stored.
- Imports of history check every row first and compare computed totals with the source to the cent; nothing is written on any error.
- Formats are configured **by example** (`number_format = 1.234,56`, `currency_format = {symbol} {amount}`), with per-language overrides for documents; one helper each (`fmtMoney`, `fmtQty`, `fmtDate`). Inputs parse both conventions.

## Own renderers instead of libraries

- A PDF writer is ~400 lines of stdlib: standard-14 fonts in WinAnsiEncoding, widths generated once from the Adobe core-14 AFM files into a Go table, deflated content streams, JPEG passthrough, PNG with alpha as SMask, link annotations, correct xref. Restrict text to what the encoding prints and refuse other characters **at input** (`422 unsupported_character`).
- Designs are values (font, rules, accent, markers), not code paths. Lay out pages first, draw footers last (page count known).
- Small third-party code is vendored into a subpackage with its LICENSE and a header naming the source and what was left out (rsc.io/qr).
- XML formats (EN 16931 CII) as one `text/template` that mirrors the schema's element order.

## Security baseline (every service)

- `http.NewCrossOriginProtection()` around the handler: the IdP cookie is valid on all sibling subdomains, `SameSite=Lax` does not stop same-site form posts. Non-browser clients pass.
- A request with `Authorization` never reads `Remote-*` headers (only basic auth for the single user); caddy strips `Authorization` after forward auth. Test: basic + forged header must be 401.
- `/metrics` has its own `metrics_key = sha256:…` that opens nothing else; empty = off. Gauges computed per scrape, counters in memory, Prometheus text by hand.
- Never read server files from request data (a stateless renderer clears logo paths). Redirect targets: local paths only, no `//` and no backslash. Preference cookies `HttpOnly`.
- Mails to customers carry only the templated text and the document, no links into internal tools; internal notices link back.
- Security headers (`nosniff`, `X-Frame-Options`, `Referrer-Policy`) in one middleware.

## Local stack

- caddy (`local_certs`) + nobloat/auth + nomail (SMTP + IMAP sink, `../nomail`) + the services, `*.<system>.localhost` (no `/etc/hosts`). The stack lives in nobloat/system (`make local`) and builds the apps from its submodules (or `../<repo>` with `SRC=..`); each app's config stays in its own `example/`; seed users are auth's `*.user.conf` files with committed argon2 hashes and public TOTP secrets (local only).
- Seed one user per role plus a denied one, and the service users of the integrations with their keys (`keys = <app>:<sha256>`).
- `demo.sh`: builds the CLIs, uses `SSL_CERT_FILE` with caddy's root, is idempotent via a marker object **created last**, and reopens closed periods it needs to book into.
- Verify layer 1–3 with a curl script (`--resolve host:443:127.0.0.1 -k`, a session per user from auth's login): expected status per user and app, a viewer's POST 403, keys (lost group 403, revoked 401), forged headers, metrics key.

## Verification toolbox

- Run the real cross-service flow in the stack, not only unit tests: pull → finalize → mark → send → nomail → storno → release.
- PDFs: Ghostscript (`gs -dNOPAUSE -dBATCH -sDEVICE=nullpage -dPDFSTOPONERROR`), poppler (`pdftotext` reports operator errors, `pdftoppm` to look at pages), plus a Go test that checks xref offsets and the shown text. XML: `xmllint --noout`; schema/schematron validators are Java jars the user may have to run.
- `golangci-lint` with gosec, staticcheck, ineffassign, unused, bodyclose; fix the real findings, list the false positives (admin-controlled paths, CLI URLs) in the review.
- `synctest` pitfalls: channels used by the code under test must be created **inside** the bubble, and the test's own shared variables still need a mutex under `-race`.
- `http.ServeMux` pattern conflicts panic at startup: a test that builds the handler catches them; prefer explicit routes per kind over `/{kind}/{id}` next to `/static/`.

## UI patterns for bigger apps

- Master data: one generic list and one generic form driven by a per-kind field list (text, textarea, int, money, percent, bool, select, color); nested names (`texts.de.intro`, `footer.0`) map to JSON.
- Autosave: the form posts on `change`; the response re-renders server-side and a small `morph` patches the DOM, keeping focus and caret and updating only unfocused inputs.
- Form endpoints and JSON API share the Go functions; parsers for user input (`930`, `+1:30`, `10%`, `1.234,56`) exist once, in Go.
- No-JS: table rows edit through inputs with `form="row-id"` and a hidden submit; dialogs are forms that post.
- Shortcut buttons carry `data-key`; only visible ones fire. `g`+letter nav underlines the letter, or appends it when the translation lacks it.

## Process

- Commit per milestone; before a likely interruption (usage limit), commit the work in progress with an honest message.
- Changes requested mid-build go into dated **Addendum** sections of `PLAN.md` (and `PROMPT.md`), including conflicts with the earlier plan and how they were decided.
- Definition of done adds: the review loop and the demo video.

## Review loop (in this order, each leaves `reviews/<name>.md` per repo)

1. **ponytail.md**: what to delete or replace with stdlib; apply the safe ones directly (no TODO comments), table of findings with *done* or *kept, because …*.
2. **code.md**: correctness; method used, fixed (with the failure it caused), verified without change, known limits.
3. **security.md**: severity, finding, fix and test; verified controls; accepted risks and operator notes.
Run `make test` after each and commit each review separately.

## Demo video

- Drive headless Chrome over the DevTools protocol with Node's built-in `WebSocket` and `fetch` (no npm): `Page.startScreencast` frames with timestamps, `Input.dispatchMouseEvent`/`dispatchKeyEvent`/`insertText` at human pace.
- Inject an overlay via `Page.addScriptToEvaluateOnNewDocument`: caption bar, a visible cursor (headless draws none), key badges for every shortcut; re-apply the caption after each navigation.
- Record against the local stack with real logins per role; show denials too. Title cards between services.
- Things headless cannot show (the PDF viewer, a terminal) are pre-rendered: PDFs to PNG with `pdftoppm`, CLI runs captured with their real output and replayed in a page with a typing effect.
- Encode with ffmpeg concat (each frame lasts until the next), pad to a fixed canvas, H.264, `+faststart`. Keep the script in the repo (`video/`, `make video`).
