---
name: nobloat
description: cinemast's nobloat software taste for small, self-hosted Go tools (workq, blog, tinyviper, poll, up/pingpong, split). Use when starting, specifying, building, reviewing or extending a nobloat project, or when asked for a minimal, dependency-free Go web app, CLI or library in this style.
---

# nobloat

Distilled from workq, blog, tinyviper, poll, up (pingpong), split, bill and track. Read before starting or specifying a nobloat project; most iterations on past projects were about points listed here. For a system of several cooperating services (or anything with money, documents, a local auth stack, reviews or a demo video) also read the `nobloat-system` skill; for anything about login, users, roles, API keys or the IdP read `nobloat-auth`. For storage, config, seed data, backup and export read `nobloat-storage`.

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
- Anything that leaves the system and may fail (mail, webhooks) goes through a durable file outbox with retries and backoff, resumed on startup (poll `outbox.go`); details in the `nobloat-outbox` skill.
- Public create endpoints are protected by a stateless HMAC proof-of-work challenge (poll/split `pow.go`), not captchas or accounts.
- Comments explain why, not what. Mark deliberate shortcuts with their limit and upgrade path.

## HTTP and API

- Plain HTTP, TLS and authentication belong to the reverse proxy (caddy). Listen on `127.0.0.1` by default; trust forwarded headers only from configured proxies (see Auth).
- An uncommon default port (poll `8737`, split `8738`, pingpong `7493`), not 8080.
- JSON APIs are strict and documented in one contract file (split `BACKEND.md`): `Content-Type` required, unknown fields, duplicate keys, trailing data and invalid IDs rejected, body size limited, empty lists are `[]` never `null`, errors as `{"error":"snake_case_code"}` with a fitting status.
- Concurrent edits: revision numbers + compare-and-swap (409 with the current state), mutation IDs make retries idempotent.
- `/metrics` in Prometheus text format, written by hand, behind its own `metrics_key` that opens nothing else (empty = off).
- Wrap the handler in `http.NewCrossOriginProtection()` whenever a cookie authenticates browsers; add `nosniff`, `X-Frame-Options`, `Referrer-Policy` in one middleware.

## Auth

Details in the `nobloat-auth` skill. In short:

- Two modes, both always supported: single user (`auth.user` + `auth.password`, basic auth, admin) and behind nobloat/auth via caddy `forward_auth` (`Remote-User`, `Remote-Name`, `Remote-Email`, `Remote-Groups`, trusted only from `trusted_proxies`). `-dev` is a fixed local admin.
- No service implements login, user management or API keys: auth issues keys and checks them on every request against the owner's current groups.
- Access only from groups: admission `access-<app>` (auth), `<app>-admin` global, otherwise the level per resource in the group name (`bill-company-<id>-editor`, `track-project-<id>-member`); one check per handler on the object's company or project, default deny. No role mapping and no access lists stored in the app.
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
- `synctest`: channels and timers the code under test waits on must be created inside the bubble, otherwise `synctest.Wait` hangs.
- A test that builds the full handler catches `ServeMux` pattern conflicts, which otherwise only panic at startup.
- Before calling it done: `golangci-lint` with gosec and staticcheck; fix real findings, name the false positives.
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
4. Public URL, default port, auth: single user and/or nobloat/auth; roles and resource groups?
5. What is logged during normal operation?
6. What leaves the system (mails, webhooks), through an outbox, and does every message link back?
7. Which pages and endpoints are public and what exactly may they show?
8. UI: list filters, search semantics, which actions where (list vs detail), confirmations, progress indicators, shortcuts?
9. Mobile, dark/light and languages (default: both themes first-class, en + de)?
10. Branding: glyph character, logo mark, palette?
11. Extension points: which things will get more variants later (types, themes, languages)?

## Plan first, then build in one go

Implementation is a single pass, so every decision is settled before code exists:

1. Ask everything at once: a numbered list, each item with a recommended default. The owner replies only with deviations ("all ok except 4, 12: …"). Do not drip questions.
2. Check the answers against each other and against the stack rules. Name each conflict exactly (e.g. "browser-only PDF vs. PDF mail attachment") and list it as an open question with a default. Never resolve a conflict silently.
3. Write `PLAN.md` per deployable **as if the software already exists**: the full sample config with defaults, seed files, `compose.yaml` + `Caddyfile`, a roles table, the REST endpoint list, a CLI reference with a worked example, ASCII sketches of every page, the shortcut tables, the data layout with one example JSON, a file → purpose table pointing to the code layout and sample files (step 4), and the tests. Open questions go at the top, Non-features next to the decisions. Integrations between deployables get their own section: who calls whom, the exact requests, failure and retry behavior, and a sequence sketch.
4. Write the **code layout as real files**: every Go file that will exist, with its package doc, types, fields, constants and function signatures with doc comments. Bodies are `panic("todo")`, and `go vet ./...` must pass, so the signatures fit together. Each test file exists with one `t.Skip("todo")` test per planned check, named after what it checks. Files copied from a sibling repo are copied now and adapted (import paths, removed parts). Interfaces are written out only where an integration needs one: another service, or a variant registry such as monitor types or notifiers. That means the method set, doc comments, the error contract, and who implements it. The rule against one-implementation interfaces still applies; a plain HTTP client to another service is a struct with typed request and response types. If a file cannot be written this way, or two files want the same function, the design is not settled: add it to the open questions. `PLAN.md` gets a short file → purpose table and points to the files instead of repeating declarations.
   Also write every **sample file** for real: the sample config with defaults and comments, one file per seeded entity (users, roles, objects) with realistic data, both Caddyfiles (production with one block per pattern, local stack), the local `compose.yaml` and its init scripts, and the test-suite configs. The owner reads and runs these to judge how the system looks and feels, and to find missing or misread requirements, before any implementation. One config set serves `make run` and the local stack (env overrides in compose), with no second copy.
   For every third-party app the system integrates (OIDC clients, for example), the README gets a setup section with the app's own role model mapped to groups: a table of group → app role and the exact config of both sides. The local stack configures it the same way.
5. Write `README.md` **as if the software already shipped**, in the final README layout (see Project setup). Use real `make` targets, `docker run`/compose snippets, the config reference copied from the plan, and one worked example per main use case, with screenshot placeholders. Write it for a user who has never seen the plan. Anything that is hard to explain in the README, needs a caveat, or contradicts the plan is a misunderstood or unsettled requirement: add it to the open questions. Keep it in sync with each plan revision. The plan holds the decisions, the README holds the user's view, and neither repeats the other's details beyond the config reference.
6. Plan a **local stack** (`example/local/`, `make local`): docker compose with caddy + nobloat/auth in front, a mail sink (mailpit), and seed data with one user per role plus users who are denied. List the users, passwords, groups and what each one demonstrates.
7. **Wireframe review before any implementation.** Build `wireframes.html`: static, self-contained mockups in the real nobloat look (monospace, tokens, light + dark), with realistic example data, every page and dialog, the phone layout, and each important flow shown step by step (e.g. "add hours from track": empty draft → dialog → result). Publish it for the owner and iterate on their feedback until they stop changing it. Check every revision for:
   - shortcuts: vim `h j k l`, `g` + letter for every navigation item, `/` global search, no key bound twice on one page
   - density: few columns, generous whitespace, secondary info in muted small text or on the detail page
   - every amount with its currency, every date/time/number in the configurable local format (default AT: `€ 2.300,47`, `31.10.2026`, `14:30`)
   - labels that need no helper text; if a field needs an explanation, rename or remove it
   The plan names `wireframes.html` as the visual reference, which wins over its ASCII sketches.
8. Add a **manual acceptance checklist** to the plan: numbered items the owner clicks through at the end in the local stack, each naming the user to log in as and the expected result, grouped by area (operations, access, main flows, edge cases, output documents, UI/keyboard/mobile/a11y, CLI parity). Automated tests are listed separately and are still required.
9. Commit `PROMPT.md`, `PLAN.md`, `README.md`, `wireframes.html`, the stub code and the sample files as the first commit of a fresh repo per deployable. Start implementing only on the owner's explicit go, then build everything in one pass without asking back.

## Definition of done

- `make test` green, `-check` passes on the sample config and README examples, lint findings triaged.
- Screenshots regenerated and looked at (desktop + phone, light + dark).
- README and plan updated to what was built (README examples run as written; requests made during the build as dated *Addendum* sections); CI and release workflows in place.
- Reviews done and written to `reviews/`: ponytail (apply the safe deletions), code, security (see `nobloat-system`).
