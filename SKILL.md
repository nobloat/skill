---
name: nobloat
description: cinemast's nobloat software taste for small, self-hosted Go tools (workq, blog, tinyviper, poll, up/pingpong, split). Use when starting, specifying, building, reviewing or extending a nobloat project, or when asked for a minimal, dependency-free Go web app, CLI or library in this style.
---

# nobloat

Distilled from workq, blog, tinyviper, poll, up (pingpong) and split. Read before starting or specifying a nobloat project; most iterations on past projects were about points listed here.

## Philosophy

- A minimal alternative to one well-known tool (River, viper, Doodle, Uptime Kuma, Hugo, Splitwise). Name the tool in the first README sentence.
- Do exactly what is needed, nothing more. Write a **Non-features** list as deliberately as the features list.
- Fewer *complexity tokens*: no framework, no plugin system, no config DSL, nothing that breaks on its own after a year.
- Small enough to read in one sitting. State the size or the dependency count as a feature ("under 400 lines", "no dependencies").
- A little copying is better than a little dependency. Reuse what the OS and browser already do (printing, `<dialog>`, `<input type="date">`, `localStorage`) before writing it.

## Stack

- Go, standard library only, current Go release. A dependency needs a reason and goes behind a build tag or a separate module (blog `-tags watch`, workq `pgxq`).
- One statically compiled binary (`CGO_ENABLED=0`), `package main` for apps, a single file for libraries. All assets via `embed`.
- Every timestamp shows its timezone (logs, UI, files); build with `-tags timetzdata` so `TZ` works in `scratch` images.
- Logging with `log/slog`. Log activity, not only errors: startup (version + clickable URL), each unit of work at INFO, state changes at WARN, failures at ERROR.
- Storage: files first. JSON per item, append-only CSV, written via temp file + fsync + rename; one lock per item (`sync.Map` of mutexes), not a global one. A database only when the problem is a database problem (relational queries, reports across many records): then SQLite through a pure-Go driver, isolated in one file (`store.go`), plain SQL, schema in one `.sql` file, no ORM, no migration framework.
- Frontend: server-rendered `html/template`, one CSS file, a little inline or single-file JS. No node, npm, bundler or SPA framework.

## Code

- One file per concern; once there are several variants of something (monitor types, notifiers, themes), one file per variant behind a small interface + registry map, so adding one is one file and one entry.
- Defaults through `cmp.Or`, small helpers inline; no interfaces with one implementation, no factories.
- Inject only what tests need (dialer, listener, `now func() time.Time`, clock via `synctest`), nothing else.
- IDs: `strings.ToLower(rand.Text()[:16])`, validated by regexp everywhere they enter; names that become filenames are restricted to `[a-z0-9-]+`.
- Anything that leaves the system and may fail (mail, webhooks) goes through a durable file outbox with retries and backoff, resumed on startup (poll `outbox.go`).
- Public create endpoints are protected by a stateless HMAC proof-of-work challenge (poll/split `pow.go`), not captchas or accounts.
- Comments explain why, not what. Mark deliberate shortcuts with their limit and upgrade path.

## HTTP and API

- Plain HTTP, TLS and authentication belong to the reverse proxy (caddy). Listen on `127.0.0.1` by default; trust forwarded headers only from configured proxies (see Auth).
- An uncommon default port (poll `8737`, split `8738`, pingpong `7493`), not 8080.
- JSON APIs are strict and documented in one contract file (split `BACKEND.md`): `Content-Type` required, unknown fields, duplicate keys, trailing data and invalid IDs rejected, body size limited, empty lists are `[]` never `null`, errors as `{"error":"snake_case_code"}` with a fitting status.
- Concurrent edits: revision numbers + compare-and-swap (409 with the current state), mutation IDs make retries idempotent.

## Auth (optional, for multi-user services)

Single-user tools keep one static basic auth credential. Once users matter, no service implements login or user management: one IdP (Authelia today, a nobloat IdP later) behind caddy `forward_auth`, swappable by proxy config only.

- Fixed header contract: `Remote-User` (stable key), `Remote-Email`, `Remote-Name`, `Remote-Groups` (comma-separated). Other IdPs are mapped in caddy (`copy_headers X-Forwarded-User>Remote-User`), never via configurable header names.
- Read them only from `trusted_proxies` (CIDRs in config), otherwise 401; a fixed user only with `-dev`. Users are created on first sight as a reference for audit log and permissions, no credentials stored.
- No login/logout UI, only a `logout_url` link.
- API keys (CLI, scripts, integrations) are the one thing a service checks itself: `Authorization: Bearer <key>`, stored hashed in the seedable config with a username and a role; caddy skips `forward_auth` when the header is present.
- Access control in layers:
  1. Who may reach the app at all: IdP rules per domain, no code.
  2. What they may do: a few fixed roles in code (`viewer < editor < admin`, each implying the lower), mapped to IdP groups in config (`role.editor = billing`). One check per handler (`need(r, editor)`), default deny, UI hides what the role cannot do but the server enforces it.
  3. Per object, only where needed: `groups` (optionally `users`) as fields on the object itself (e.g. a project), visible when a group is shared, admins see all. Seedable, audited, no ACL table.
- No custom roles, permission matrices or RBAC engine until a real case needs more than this.

## Configuration

- Pick the medium for the audience and stick to it: env + `.env.sample` for a simple service (poll), one strict JSON file for infrastructure-as-code (pingpong), Go code for a personal tool (blog).
- Connection settings as one URL (`SMTP_URL=smtp://user:pass@host:587`, `smtp+insecure://` for local Mailpit).
- Strict: unknown fields are errors, everything validated at startup and via a `-check` flag (exit 1), defaults documented next to each option.
- Secrets live in the config; treat the whole file as secret; never render secrets in any UI (mask them).
- A public `base_url` whenever anything leaves the system (mails, notifications): links must always point back to the affected item.
- Ship a runnable sample config that demonstrates every interesting feature against real, public targets.

## Tests

- Stdlib `testing` only, integration tests that start the real thing in-process (`t.TempDir()` for data), plus small unit tests for parsers.
- `testing/synctest` for anything time-based; in-memory listeners/dialers instead of real sockets so the fake clock works.
- `make test` = `go vet` + `go test -race`. Every feature lands with its test; every bug fix gets one.
- UI changes are verified visually (`make screenshots`), not only by asserting HTML.

## UI

- Look: monospace (`ui-monospace, monospace`), generous whitespace, thin rules, no web fonts, no icon libraries. Colors carry meaning only; define each once as a CSS custom property with `light-dark()`.
- Light and dark are both first-class: follow the OS by default, offer a small toggle button (◐ auto → ☀ light → ☾ dark, never a dropdown) persisted in `localStorage`, applied before first paint.
- Mobile is first-class: usable at 360–500px without horizontal scrolling. Wrap the header, drop secondary columns, keep primary actions.
- Progressive enhancement: everything works without JS (forms post, links filter); JS adds live updates, live search, dialogs, in-place actions, shortcuts.
- State and status are visual (lights, icons with aria-label/title), not words.
- Lists get filter chips with counts, clickable tag pills, live search. Destructive or public actions get a confirmation dialog; actions that take time show a spinner.
- Keyboard first and discoverable: `<kbd>` hints on buttons and a per-page shortcut line; handled keys `preventDefault` so they never type into inputs; hints hidden on touch.
- i18n: one flat `i18n/<lang>.json` per language (`"area.key": "text"`), embedded, loaded at startup (panic on invalid file), `T(lang, key, args...)`, language from `Accept-Language` with an explicit override.
- Public pages (status pages, shared polls) show the minimum and never technical details.
- Accessible basics always: real buttons/links/forms, labels, focus kept across live updates, contrast >= 3:1 for meaningful graphics.

## Branding

- Name `nobloat/<project>`, glyph `][` + one project character (`][✓` poll, `][•` pingpong) used in `<title>` (`][• pingpong (2)`, counts at the end), the footer (plain text, no SVG, no color), the basic auth realm and `logo-full.svg` (brackets `#8b8b94`, character in the project color, monospace text).
- A project logo mark may express state (pingpong's ball); favicons are square SVGs that adapt to dark mode.
- One palette shared by UI, favicons, logos and outgoing messages (mail, Slack colors).
- Footer: glyph + `nobloat/<project>` · made by cinemast · nobloat.org · src · version.

## Project setup

Copy from poll or up, adjust names. Layout: Go files at the root, `web/` (templates + `static/`), `i18n/`, `example/` or sample config.

- `Makefile`: `build` (static binary, `-X main.release=$(TAG)`, TAG = exact git tag or `main`), `test`, `run`/`dev` (sample config), `package`, `push` (`ghcr.io/nobloat/<project>:$(TAG)`), `release` (next `vN` tag), `screenshots`, `clean`; local helpers like `mailpit` as needed.
- Version = release + short commit from `debug.ReadBuildInfo` (`+dirty` if modified), shown in footer and startup log.
- `Dockerfile`: `golang` builder runs vet + tests + `make build` (keep `.git` in the context), final `scratch` with CA certs, non-root user (`65534`), data as a volume.
- GitHub Actions: `ci.yml` (push to main + PRs: `make test`, `make build`), `release.yml` (tag push: `make push`, also `:latest`).
- `screenshots.sh`: start the app with deterministic data, headless Chrome (`--timeout`, not `--virtual-time-budget` when pages keep a stream open; `--force-prefers-reduced-motion`, headless freezes CSS animations on their first frame), desktop 1280 and phone **500** px (Chrome's minimum layout width), light and dark, ImageMagick overview committed for the README. Bash 3.2 compatible.
- `README.md`: `logo-full.svg`, one-sentence pitch naming the replaced tool, bullet features with the no-dependency claim, Non-features, Screenshots, Development (`make` targets), Deployment (docker run + compose), Configuration.
- `.gitignore`: binary, data dir, per-page screenshots.

## Before starting: answer these up front

Most iterations came from these being unspecified. Decide them in the plan:

1. Which tool does it replace, and what is explicitly out of scope?
2. Storage: files or (exceptionally) SQLite, and why? Backup = copy the data dir?
3. Config medium (env / JSON / Go), its strictness, defaults, secrets, includes/drop-ins?
4. Public URL, default port, auth: static credential or IdP headers; roles and per-object access?
5. What is logged during normal operation?
6. What leaves the system (mails, webhooks), through an outbox, and does every message link back?
7. Which pages and endpoints are public and what exactly may they show?
8. UI: list filters, search semantics, which actions where (list vs detail), confirmations, progress indicators, shortcuts?
9. Mobile, dark/light and languages (default: both themes first-class, en + de)?
10. Branding: glyph character, logo mark, palette?
11. Extension points: which things will get more variants later (types, themes, languages)?

## Definition of done

- `make test` green, `-check` passes on the sample config and README examples.
- Screenshots regenerated and looked at (desktop + phone, light + dark).
- README and plan updated; CI and release workflows in place.
