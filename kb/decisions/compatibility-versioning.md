# Decisions: Compatibility and Format Versioning

> **Status:** Shipped in v0.11.0 (PR #695, with the hook contract in PR #688).
> The user-facing rules are meridian-cli `docs/releasing.md` "Compatibility and
> versioning"; what changed for users is `docs/upgrading.md` "Compatibility
> policy and state version checks".

Until v0.11.0 the project rule was "no real users, no backwards
compatibility": a change could rename a config key or reshape a stored file and
expect every reader to be the same build. That rule never matched how
Meridian runs. Two builds touch the same data whenever a primary keeps running
across an upgrade, a work directory syncs through git to a machine on another
release, or a separately installed component (Mars, a harness extension) reads
what Meridian wrote. The decisions below replace the rule with a policy that
says what each side of that skew must do. See [../decisions.md](../decisions.md)
for the chronological index.

What went wrong under the old rule, found while auditing for this policy
(`work:versioning-policy` `versioning-inventory.md`):

- A work item's `__status.json` had no version and its reader rejected unknown
  fields. On a parse failure the reader fell back to defaults built from the
  directory's mtime, so a file written by a newer build lost its description,
  goal and status, and the item still looked normal.
- Telemetry and session records carried `v`, but readers parsed any value as
  version 1.
- Several version fields were written and never compared:
  `plugin_api.__version__`, the launch-policy snapshot's `schema_version`, the
  Pi runtime sidecar's `schema_version`, and a copy of the native-search schema
  inside its SQLite file.
- An old-build primary and a new-build CLI treated each other's idle files as
  absent, silently.

---

## D-compat-policy — one-minor deprecation; new builds read old data, old builds refuse new data

**Decision (2026-10-19, owner):** A removed or renamed config key, hook name or
CLI flag keeps working for one minor release, warning with its replacement,
then becomes an error. A new build reads anything an older build wrote,
migrating on read. An older build that finds newer data refuses it loudly with
a remedy and never substitutes defaults. Rollback is supported only where that
version's upgrade note says so. A break still needs `release:minor` and an
upgrade note; a deprecation that still accepts the old input is not itself a
break.

**How it was reached.** The question started narrow: what Plugin API version
the permission-notice event should bump. The owner pressed on what the version
string did ("how are u making the comparisons tho...?"); nothing read it. The
owner then widened the question to "any versioning / semver for our project",
and, after the lead framed it as server/client skew, accepted that the skew is
real without a server: "well, i guess we don't have a server, but the same
stuff applies i think". The owner then said "I think we should get rid of the
no backwards compatibility rule now" and chose the lead's one-release
deprecation option over the alternatives (`chat:c8397`). The owner gave no
further rationale; the reasoning above is the lead's, which the owner accepted.

**Why one minor release.** It is the shortest window that lets every machine
sharing one config upgrade before the old input stops loading, and it keeps
compatibility code bounded: each window is one table or one branch to delete
in the next minor. The hook contract is the first use; its window lives in
`NEXT_MINOR_HOOK_CONFIG_BREAKS` (meridian-cli `lib/hooks/schema.py`), whose
`mode` flips to `"error"` in the next minor release
([D-hook-deprecation-window](hook-event-contract.md#d-hook-deprecation-window--old-names-load-for-one-minor-release)).
Removals are tracked in the owning directory's `.context/TODO`.

**Why refuse instead of guess.** The `__status.json` fallback shows the cost of
substituting defaults: the data is not lost on disk, but the reader presents
something false as normal, and a later write makes the loss real.

**Applied to the release itself.** The breaking 0.11 changes (#688, #689,
#695) merged into a `release/0.11` branch while patches (v0.10.29, v0.10.30)
kept shipping from `main`; one `release/0.11` → `main` PR (#697) shipped v0.11.0. The
lead first proposed holding the breaking PRs on `main` under `release:skip`,
which would have blocked patches; the owner objected ("I also still want to
submit patches", then "isn't it just a minor version branch...?"). This was how
0.11 shipped, not a standing rule: `docs/releasing.md` still describes
releasing from PR merges to `main`.

---

## D-version-only-where-read — integer format versions where another build reads; delete the rest

**Decision (2026-10-19, lead):** Version a stored format only when a different
build can read it. Store an integer in the data and check it by membership in
the reader's supported set. Semver is for software people install (`meridian`,
`mars`, package `requires-*`). Delete a version nothing reads.

**Why:** "Version every file to be safe" is how write-only versions
accumulated. A version nobody checks promises compatibility handling that does
not exist, and readers trust the promise. Integers checked by set membership
let a reader say exactly which formats it handles; semver ordering adds nothing
to that question.

**Applied in v0.11.0:**

| Format | Outcome |
|---|---|
| Telemetry envelope `v`, `sessions.jsonl` event `v` | Kept; readers now check it |
| `plugin_api.__version__` | Deleted. The Plugin API is not installed separately; the hook contract is identified by payload `schema_version` (3) and Meridian's own version |
| `LaunchPolicySnapshot.schema_version`, Pi `pi_runtime_meta.json` `schema_version`, native-search SQLite metadata copy | Deleted; the native-search file name derives from its one `SCHEMA_VERSION` |
| MCP and `ext list` JSON `schema_version`, Claude `plugin.json` `version` | Kept: external output contracts, not stored formats |
| Legacy native-import marker `schema_version` | Kept, contrary to the first plan |

The legacy import marker is the one divergence. v0.10.29's reader requires the
field and later `doctor` passes read the marker, so omitting it would make new
markers unreadable to the previous build. The field stays, the new reader
treats an absent field as 1, and removal waits in meridian-cli
`src/meridian/lib/state/.context/TODO` until the strict reader leaves the
compatibility window.

**Not fixed here:** Mars skips a package's `requires-meridian` check silently
when `MERIDIAN_VERSION` is unset (haowjy/mars-agents#178).

---

## D-staged-version-add — readers learn a version one release before writers write it

**Decision (2026-10-19, lead):** Adding a version to an unversioned format
takes two releases. v0.11.0 reads `v` in work `__status.json` (absent means 1),
refuses a newer `v`, and tolerates unknown fields with a warning, but does not
write `v`.

**Why:** v0.10.29's work reader rejects unknown fields. Writing `v` in v0.11.0
would trigger the silent mtime fallback on every machine still on v0.10.29, the
exact failure this policy exists to stop. Once every supported reader accepts
`v`, a later release can write it.

**Unknown fields survive rewrites.** The work reader keeps fields it does not
know and writes them back, so an older build's `work update` does not strip a
newer build's field. That creates the rule for dependent fields: a new field
whose meaning depends on another field requires bumping `v`, because an older
build that changes the field it depends on would carry the dependent value
forward stale.

A newer `__status.json` shows in listings as `requires-newer-meridian`, kept
separate from the item's own status, and every mutation refuses it.

---

## D-check-before-deciding — version checks guard decisions and mutations, not blind appends

**Decision (2026-10-19, lead, across three review rounds):**

- **Work items:** the repository's single write seam loads and validates every
  copy of an item, active and archived, before any mutation runs. Rename, done,
  reopen, delete and duplicate cleanup cannot touch disk and then refuse.
  Per-command checks were tried first and missed rename and duplicate cleanup.
- **Sessions:** code that reads existing rows to decide what to append
  (starting a chat, binding a native session, deduplicating model records,
  adopting an observed native session) refuses when an unsupported row names
  the same chat or native session. Blind appends of this build's own rows read
  nothing and check nothing.
- **Legacy native import:** reads only the chats it binds and defers, through
  a dedicated `UnsupportedSessionFormat`, when one has an unsupported row.

**Why blind appends skip the check:** appending interprets nothing, and a newer
build reads older rows by policy. The check as first built re-read all of
`sessions.jsonl` under the lock whenever another process had appended,
measured at +0.25 s per append at 40k rows and +0.45 s at 73k, growing with
history. That broke the [per-operation cost](../architecture/per-operation-cost.md)
bar for no safety gain. Rejected: a format marker file (a new concept for a
check blind appends don't need) and a partial tail read (still a full read per
process).

**Why the import is scoped:** unscoped, one unsupported row anywhere made the
import fail, and because the import runs at startup until its marker exists,
every command failed with it.

**Known limit:** a newer row that names no chat or native session is invisible
to scoped decisions. It refuses only decisions that check every chat, such as
repair binds.

---

## D-index-named-by-inputs — a rebuildable index's file name carries every input format

**Decision (2026-10-19, lead):** A disposable projection puts its own schema and
every independently versioned input format in its file name, and rebuilds on a
mismatch instead of migrating. The history index became
`history-v10-sessions-v1.sqlite3`. It skips session rows it can't read, keeps
indexing later ones, and records the skipped versions in its cursor so every
indexed read warns once per command.

**Why:** the earlier rule named the file by index schema alone
([D-history-index-schema-namespace](history-storage.md#d-history-index-schema-namespace)).
A session format change would then leave an index built from rows the new
build reads differently. Naming by both inputs makes a format change force a
rebuild from authority. Skip-and-record replaced a first design that froze the
index at the first unsupported row, which hid every later session.

**Rollback:** after rolling back to v0.10.29, its `history-v10.sqlite3` was not
updated while the new file was active; `docs/upgrading.md` tells the user to
run `meridian session index rebuild --metadata-only`.

---

## D-idle-absent-exception — idle state warns and behaves as absent

**Decision (2026-10-19, lead):** An idle file with an unknown format warns,
naming the file and saying another build wrote it, and is treated as absent.
This is the one documented exception to refusing newer data.

**Why:** idle files are short-lived and already expected to be absent.
Refusing one after a mid-session upgrade would disable idle handling until the
running primary and the CLI were the same build, a worse outcome than losing
one stretch's state. The previous behavior was the same minus the warning.

---

**Provenance:** `work:versioning-policy` `decisions.md` V1–V12 and its
`DIVERGENCE/legacy-import-report-version.md`; `work:hook-event-names` D7, D9,
D10; owner turns in `chat:c8397` (mined by `spawn:p8692`); reviews
`spawn:p8665`, `spawn:p8670`, `spawn:p8676`.
