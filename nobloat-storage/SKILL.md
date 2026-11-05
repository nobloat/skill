---
name: nobloat-storage
description: How nobloat apps store config, entities and data without a database: key=value config with env overrides, entity .conf files with hot reload, JSON per type or per object, CSV or JSONL rows, blobs, atomic writes, locks, CAS, multi-file transactions, strict startup checks, audit logs, seeding from prepared files, trivial and incremental backup, export, migration, and when SQLite is justified. Use when planning or reviewing an app's storage, config or data layout.
---

# nobloat-storage

Read with the `nobloat` skill. Distilled from bill, track, tax, up and auth (all stdlib only, no database). Decide every point below in the plan's data-layout section.

## The ladder

Pick the first shape that fits. Write down the expected size, and add a `ponytail:` comment where it stops fitting.

| Data | Shape | Example |
|---|---|---|
| Settings of the app | `<app>.conf`, `key = value` | every app |
| Entities that admins or IaC edit, maybe live | one `.conf` per entity in a directory, hot reload | auth users/roles/apps, up monitors/notifiers/pages |
| Master data, up to a few thousand objects, edited in the UI (see Per type or per object) | one JSON array per kind, sorted by id | bill companies/customers/products, tax vendors/assets |
| Documents edited one at a time, many of them | one JSON file per object: `<kind>/<id>.json` | bill `docs/`, tax `receipts/`, outboxes |
| Rows, mostly appended, read in ranges | CSV, partitioned by owner or year when it grows | track `entries.csv`, tax `bank/<company>.csv`, up `<monitor>.csv` |
| History | `audit.csv`, append only | all |
| Originals and generated files | blobs: `files/<id>/<name>`, versions `vN` (never overwritten) | bill PDFs, tax receipts and packages |
| Credentials and state the app writes alone | `secrets/<id>.json`, 0600 | auth |

**SQLite only if absolutely necessary.** Allowed only when all of these hold: data that outgrows a few hundred MB or ~100k rows queried by several keys, ad-hoc filtering or aggregation over the whole history on every request, and splitting by year or owner does not help. The plan names the size that forces it. Then use a pure-Go driver (no cgo), one file in the data dir, schema migrations in code, and keep the backup story "copy the data dir" (with `VACUUM INTO` for a consistent copy). Until then: files.

## Per type or per object

Decide per kind, in the plan's data-layout table, with these questions. One "per object" answer usually decides it.

| Question | Per type: `<kind>.json` (all objects in one file) | Per object: `<kind>/<id>.json` or `<id>.<kind>.conf` |
|---|---|---|
| How many objects? | up to a few thousand, small | many, or objects of real size (documents with lines, receipts) |
| Who writes them? | the app (UI, API) | also humans and IaC: one file per object is what they can create, review and delete |
| Seeded or managed from outside? | rarely | yes: dropping a file in is the seed (see Seeding) |
| How are they changed? | rarely, one at a time | often, concurrently: a lock per object instead of one per kind |
| Rules across objects (unique number, references) | cheap: one read, one validated write | need a startup scan and a lock on the scope (the series, the company) |
| Listing | one read | a directory scan, parse every file (cache by mtime+size if hot) |
| Lifecycle | edited in place | created, frozen, archived (documents), deleted by removing the file |
| Damage | one bad file stops the kind | one bad file names the object; startup still refuses it |
| Hot reload | not needed | per file, by fingerprint |

Rows (entries, bank rows, check results) are neither: CSV per kind, partitioned by owner or year.
Blobs always sit next to their object: `files/<id>/…`, never inside JSON.

Examples: bill keeps companies, series, customers, products and projects per type (few, app-edited, unique numbers per series) and documents per object (many, frozen, concurrent). auth keeps users, roles and apps per object (`.conf`, IaC-managed, hot reload). up keeps monitors, notifiers and pages per object for the same reason.

## Rows: CSV or JSONL

Both are one record per line in one file per kind (or per partition), read sequentially, appended or rewritten. Pick per kind:

| | CSV | JSONL (one JSON object per line) |
|---|---|---|
| Shape | flat, fixed columns; lists as `;`-joined cells, ids as references | nested objects and lists, optional fields, records of different shapes |
| Readers | spreadsheets, the Steuerberater, `grep`, `awk`; doubles as the export | `jq`, other programs; needs a converter for spreadsheets |
| Schema | the header is the contract, checked exactly at startup; a new column = a format change (fixtures, sample, `migrate` after deployment) | struct with `DisallowUnknownFields` per line; a new optional field is backward compatible, a removed one is a format change |
| Types | everything is text: one parser per column (RFC 3339, cents as integers, `true`/empty), written in Go once | numbers, bools and null from JSON; still store money as integer cents and times as RFC 3339 strings |
| Size | compact | field names repeat on every line (≈2× CSV) |
| Append safety | a quoted field may hold newlines, so one record is not one physical line: a torn append is only detected by the CSV parser (unterminated quote, wrong field count) | one record is always one line: a torn last line is invalid JSON and easy to detect and report |
| Security | cells starting with `= + - @` (and tab/CR) are neutralized with a leading `'` on export (formula injection) | nothing extra |

Rules:
- **Default to CSV** for flat business rows people look at: time entries, bank rows, audit logs, check results. That is what bill, track, tax and up use; track replaced its earlier JSON/JSONL files with CSV.
- **Use JSONL** when records are nested or heterogeneous (events with varying details, webhook or mail logs, imported foreign records kept verbatim), or when each record is an existing JSON type of the API.
- **Hybrid** is fine for logs: fixed CSV columns plus one `details` column holding compact JSON (auth's `audit.csv`). Filterable columns stay columns.
- Never store the same rows in both formats; an export in the other format is computed.
- Both: header or field set checked at startup with file and line on error, strict per-record decoding, `encoding/csv` and `encoding/json` from the stdlib (no hand-rolled splitting), UTF-8 without BOM, `
` line endings written (CRLF inside quoted CSV fields preserved), sorted by time or id so diffs and partial reads stay cheap.
- Append (O_APPEND, one complete record, `Sync`) for logs and growing rows; a full rewrite under the kind's lock when rows are edited in place (track entries carry `rev`). Mark the size limit with `ponytail:` and the split plan (per year, per owner).

## Data dir rules

- One data dir per instance, one writer process. Backup is a copy or tar of the dir; restore is untar.
- Names: ids `[a-z0-9-]` (≤ 64), readable when hand-written (`smalldiff`) or when they appear in group names, `rand.Text()[:16]` lowercased otherwise. The file name is the id.
- Permissions: dirs 0750, data files 0600 (secrets always), files meant for IaC/humans (`.conf`) 0644. The image runs as uid 65534; the README says to chown the volume.
- JSON: `MarshalIndent` with 2 spaces, trailing newline, lists `[]` never `null`, arrays sorted by id, so diffs stay small and hand edits are easy.

## Writes

- **Atomic file write (copy this function):** `CreateTemp` in the same dir, write, `Sync`, close, chmod, rename over the target, sync the dir. `syncDir` per OS (`syncdir_other.go`: open + Sync; `syncdir_windows.go`: CreateFile with backup semantics + FlushFileBuffers), and on Windows retry the rename on sharing violations.
- **Locks:** a `sync.Map` of one mutex per relative path; `lock(rels...)` sorts and dedupes so multi-file writers never deadlock. Hold the lock across read-modify-write.
- **Concurrent edits:** every object has `rev`; PATCH needs it, a mismatch is `409 rev_mismatch` with the current object. For hand-edited `.conf` files use the sha256 of the content instead of a `rev` key, so files stay clean.
- **One change, several files:** write a `.transaction.json` with every new file content first (the commit point), then replace the files, then delete the record. Replay it on startup before reading anything (track). Without it, order the writes so a crash leaves something `check` can find (blob before its JSON, remote mark before local write) and say so in a `ponytail:` comment.
- **After a failed write:** answer 503 on every request until a restart has recovered (track). Never keep serving a state that is not on disk.
- **Appends:** `O_APPEND`, one complete line per write, `Sync`. The reader rejects an unterminated last record (`check` reports it) instead of guessing.
- **Rewrite vs append:** append for logs and rows that only grow; full rewrite only for small files. Mark the limit: `// ponytail: rewrites the whole file per event; split by year past a few MB`.

## Reads and startup

- Decode strictly everywhere, data files included: unknown fields, duplicate keys and trailing data are errors.
- Startup validates everything (`openData`): parse every file, check ids, duplicates and references, check uniqueness rules (numbers per series). On any error refuse to start with file and line. There are no silent skips.
- `<app> check -config <file>` runs the same code and prints every problem at once (warnings separate from errors). CI runs it on the sample config and data.
- Memory model, picked per app and written in the plan:
  - **Read per request** (bill): simplest; external edits show up immediately, but are not validated until restart.
  - **Cache keyed by mtime+size** (tax): read per request without re-parsing unchanged files; cap the size.
  - **Immutable snapshot swapped on write** (track, auth users): fastest reads; external edits need a restart or a reload loop.
- Indexes are maps built from the snapshot, never stored.

## Config

- `<app>.conf`: `key = value`, `#` starts a comment at line start or after whitespace (so `#1f4e79` stays a value), dotted keys for groups (`auth.user`, `de.number_format`, `company.<id>.uva`), comma lists, Go durations plus `d`.
- Unknown keys, duplicate keys and lines without `=` are errors with file:line.
- Env `<APP>_<KEY>` overrides every key (`.` → `_`). An unknown `<APP>_*` variable is an error, except the CLI's own (`<APP>_URL`, `_KEY`, `_USER`, `_PASSWORD`).
- Defaults live in one map. `example/<app>.conf` lists every key with its default and a comment and is the documentation; `check` must accept it.
- Secrets that are only compared are stored as `sha256:<hex>` (`metrics_key`, OIDC client secrets), and `<app> secret` prints a new one with its config line. Secrets the app must send (SMTP, IMAP, the other app's key) are plain and the config file is 0600.
- `-dev`: a fixed local admin, logged mails, no cookies `Secure`. It is never a production mode.

## Entities as files and hot reload

For entities that admins or IaC own (users, roles, apps, monitors):

- One `<id>.<kind>.conf` per entity, the file name is the id, in the config or data dir. Subdirectories are not read.
- A poll loop every 5–10 s fingerprints name+mtime+size of `*.conf`. On a change it loads everything, validates, and swaps the new state in atomically. Any error keeps the old state and logs ERROR. Keys that need a restart (`listen`, `data_dir`, the main `.conf`) log a WARN instead of applying.
- Audit file changes: keep `files.json` with the hash and non-secret values per file as last written or loaded, and write `file.change` with old/new hash and a value diff without secrets. Changes made by the app itself are not reported as external.
- UI edits write the same files (atomic write, CAS by content hash) and refuse values that would not read back.
- No fsnotify, no SIGHUP: polling works on every filesystem and in containers.

## Seeding

"There is no seeding code: the files are the configuration." Seeding means putting prepared files in the layout the app reads, before or while it runs. The layout decides how:

**Per-object files (preferred for anything seeded).** One file per object is one seed unit. Two ownership modes, chosen per kind and written in the plan:

1. **Managed from outside** (up's monitors, auth's apps): the files in the config dir are the truth, mounted read-only or written by IaC. The app reads them at startup and on every reload, never copies them. UI edits, if any, write the same file (atomic write, CAS by content hash, values that read back), so IaC and UI stay one source. Deleting the file deletes the object.
2. **Seeded, then owned by the app** (auth's users, roles): the files live in the data dir. IaC drops `<id>.<kind>.conf` there, and the app picks it up on startup or reload and owns it from then on: UI changes rewrite it, IaC may still edit it (the hash shows the change as `file.change`).
   - Values the app takes over and must not see again in a file (password hashes, TOTP secrets, keys) are **seed-only keys**: read once when the app's own state file (`secrets/<id>.json`) does not exist, copied there, never written back, audited `seed.applied`. `check` warns when they are ignored.
   - A seed file never overwrites state the app owns; the app never deletes a seed file it did not write.

**Per-type files.** A kind file is seeded as a whole: copy the prepared `<kind>.json` into the data dir while the app is stopped. The app validates it at startup and refuses invalid data. There is no merge of a seed into existing data: either the file is the seed or it is the app's.

**Data dirs that ship with a repo:**

| Dir | Purpose | How it gets used |
|---|---|---|
| `example/<app>.conf` (+ `example/*.<kind>.conf`) | every config key and one sample per managed kind | documentation; `check` must accept it; `make run` uses it |
| `example/data/` | test fixture, small, per type and per object as in production | `go test` copies it into `t.TempDir()` |
| `example/local/<app>/data/` | realistic sample (years, consistent with the sibling apps) | `make run`: `test -d data \|\| cp -R example/local/<app>/data data`; the local stack mounts or copies it into a volume |
| generated sample | when it depends on another app's sample (tax from bill) | `make sample`, deterministic for a date, writes files and runs `check`, never calls a running app |

- For managed kinds the seed dir can be the running dir (auth `make run` serves `example/` and `example/data/` directly; runtime files like `secrets/`, `outbox/`, `audit.csv` land next to the seed and are gitignored; `make local-reset` deletes them).
- `rm -rf data` (or deleting the volume) starts over from the seed. A format change of seeded files needs that reset; say so in the commit and README.
- Secrets in seed files are local only and marked as such in a comment at the top of each file.

## Audit log

- `audit.csv`, append only, one line per change: `object_ref,timestamp,user_ref,description` (`object_ref` = `<kind>/<id>`, description = action plus changed fields with old and new values). Startup checks the header.
- Audit failures are logged, never undo the change. Reading for a history page scans the file; partition by year when it outgrows a few MB (`ponytail:`).

## Backup and export

A nobloat app must be trivially backed up: **the data dir is the backup unit**, copied with any tool, while the app runs, full or incremental, and restorable by copying it back.

**Layout rules that make this true:**
- Everything the app writes lives in the data dir; nothing in the image, `/tmp` or a second place. Config is separate (read-only mount, owned by IaC) and backed up with the IaC repo, unless the UI writes it (up): then it is data and in the backup list.
- Every write is an atomic rename, so a file copied at any moment is the old or the new version, never half.
- A copy taken while the app runs must restore: multi-file changes go through the transaction record (copied along, replayed at the next start), single-file changes are consistent by themselves, and orphans from a crash window (a blob without its JSON) are `check` warnings, not start errors.
- Temp files are named `.tmp-*`, caches `*.cache.json`: both excluded from backups and recreated.
- Secrets in the data dir (`secrets/`, `keys/`) are part of the backup, so the backup is as sensitive as the instance.

**Incremental by construction:**
- Prefer files that never change once written: per-object files, frozen documents, versioned blobs (`v1.pdf`, `v2.pdf`, never overwritten), rows partitioned by year or owner (only the current partition changes).
- Prefer append over rewrite for growing files (logs, rows): rsync, restic and borg then move only the new tail or chunks. A file rewritten whole per event (bill's and tax's `audit.csv` today) is a full copy in every increment; partition it by year.
- Per-type files stay small (that is the rule for choosing them), so rewriting them costs little.
- Result: `restic backup /data`, `rsync -a --delete`, `borg create`, or a ZFS/btrfs/volume snapshot every hour moves only what changed. No app-specific backup tool, no dump step, no lock.

**Export through the app** (for people without shell access, and for a guaranteed consistent cut):
- `GET /api/export` (admin; auth, which has no REST admin API, serves it as `GET /admin/export`) streams a tar.gz under the storage read lock (writes wait; reads continue): `data/…` for the data dir and `config/…` for the config file or dir, without temp files and caches, with `manifest.json` first (app, version, created, every file with path, size, mtime and sha256). The CLI wraps it: `<app> export > backup-2026-11-05.tgz`.
- `GET /api/export?since=<manifest timestamp>` returns only the files changed since then plus the full manifest, so deletions are visible: an incremental export. Restore = the full export, then each increment in order, deleting files absent from that increment's manifest.
- `<app> restore <archive>…` (offline) unpacks into an empty data dir, verifies sha256 against the manifest, then runs `check`. Restoring by hand (untar) is equally valid.
- A test does the drill: export, restore into `t.TempDir()`, `check` passes, the data compares equal.

**Exportability (getting the data out):**
- The storage formats are the export formats: JSON, CSV, `key = value`, plain blobs, documented in the README's data-layout table with one example per file. Nothing binary, nothing that needs the app to read.
- Every list and report also downloads as CSV (formula cells neutralized); documents as their original files (PDF, e-invoice XML).
- Ids are stable and references are ids, so a file set exported from one instance imports into another or into a spreadsheet without mapping.

## Migration

Format changes before the first deployment: change the files, the fixtures and the sample, no migration code. After that: an offline `<app> migrate -data <old> -out <new>` that never touches the source, refuses an existing destination, reads the result back and compares it record by record. Retention is explicit: write down what grows forever (audit, rows, versions) and how it is split or pruned.

## Metrics

Storage shows up in `/metrics` only where an operator acts on it: outbox length and oldest age, write failures (the 503 state), size of the largest growing file if a limit is documented.
