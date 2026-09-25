# Legacy Native Session Import

## Legacy import

Slice branch `slice/pr1-legacy-import` at `8e8fe485` (merging into PR #520). The
rules and the user decision behind them are in the
[decision](../decisions/native-session-identity.md#chats-from-before-the-key-existed-import-once).

**Trigger.** `ops/runtime.py`'s `resolve_runtime_authority_for_read` and
`resolve_runtime_authority_for_write` call
`ops/legacy_native_import.maybe_import_legacy_native_sessions(runtime_root)`.
Commands that never resolve a runtime root, such as `--help`, do not trigger it. A
root without `sessions.jsonl` is left alone, so an untouched project gets no state
files. When the marker `<runtime_root>/legacy-native-import-v1.json` exists, later
runs do nothing more than check for it.

**Sequence.**

1. Take `locks/legacy-native-import.lock`, then check the marker again. A concurrent
   process waits here and then finds the marker.
2. Build the report without the sessions lock. Read `sessions.jsonl` as raw rows,
   including historical `harness_session_ids` arrays and every recorded cwd
   (`execution_cwd`, `task_cwd`, `control_root`). Scan spawn rows for the chat's own
   `harness_session_id`. Validate candidates against native stores.
3. Enter `session_bindings()`. For each accepted chat, check again under the lock. A
   chat that gained a complete key concurrently is skipped. A chat whose recorded IDs
   or `session_instance_id` changed is counted as `ambiguous_id`. A bind conflict is
   also counted as `ambiguous_id`. Every other accepted chat is bound with
   `source="legacy_import"`, and the whole batch is appended with one fsync.
4. After the context commits, write the marker atomically. It holds the schema
   version, a timestamp, per-harness counts for `imported`, `missing`, `ambiguous`,
   `ambiguous_id`, `no_session_id`, and `unsupported`, the unbound chat IDs for each
   reason, and the bindings. Print one line to stderr:
   `Imported native sessions for N of M existing chats; K left unbound (details: …)`.

The lock order is import lock, then history-mutation lock, then sessions lock. Native
I/O and the spawn scan happen before the sessions lock is taken, so live launches are
not blocked behind the Codex walk or the OpenCode copy.

**Crash recovery.** If the process dies after the append but before the marker, the
next run starts again. Chats that already have a `legacy_import` row are counted as
imported and are not bound a second time. A torn tail from the single batch append is
repaired like any other torn JSONL line, and the chats it covered are bound again.

**Candidates** (`lib/harness/legacy_native_stores.py`). This is a single module that
dispatches on harness ID. It is deliberately not an adapter protocol hook, because
the logic runs once. Each store value must equal what PR 1 records for a new chat of
that harness. It is derived through the adapter's own `native_store_for_launch` from
recorded facts, and the ambient `CLAUDE_CONFIG_DIR`, `CODEX_HOME`, and `OPENCODE_DB`
are ignored.

| Harness | Candidate stores | Exact check |
|---|---|---|
| Claude | `<config root>/projects/<slug>` for each distinct recorded cwd of the chat and its spawns. The config root is the recorded `claude_config_dir`, else the default home. | First-line `sessionId` of `<store>/<id>.jsonl` |
| Codex | `<home>/sessions` from recorded launch-policy env, else the default home. Relative homes resolve against a recorded cwd. | One `rglob` per store builds an ID→paths index, then the shared `codex_rollout.resolve_exact_rollout` validates it (the same function live reads use): more than one file is `ambiguous_native_file`, otherwise `session_meta.payload.id` must match |
| OpenCode | Resolved DB path from recorded env, else the default | Strict `SELECT 1 FROM session_v2` (or `session`) on a private snapshot (see below) |
| Pi | `<meridian pi sessions root>/<spawn_id>/` for the chat's own spawns only | Pi `session` header `id` |
| Cursor, historical records | none: `unsupported` | — |

**OpenCode snapshot.** The import never opens the source database with SQLite. The
module's comment explains why: a `mode=ro` connection can still write WAL
shared-memory read marks. Instead, the import copies the DB, and the WAL if present,
into a temporary directory once per store per import. It records a fingerprint before
and after the copy: inode, size, and mtime for both files, plus the WAL's 32-byte
header with its salts. If the fingerprint changed, a checkpoint or WAL restart may have
torn the snapshot. The import then raises and is deferred. Row lookups use a strict
query that raises on `sqlite3.Error` instead of returning "absent". Before this fix,
a torn copy made 201 of 251 existing sessions read as absent in a reproduction, and
the marker would have made that result permanent. Cost on the live 6 GB database is
about 6 GB of scratch space and 17–26 s. Each runtime root pays this once, and only
when it has unbound OpenCode chats; the dev report pays it as well.

**Deferral.** `SpawnStateQuarantined`, `OSError`, and `sqlite3.Error` are caught
around the whole import. The command prints `Native session import deferred for …`,
writes no marker, and continues. Quarantined spawn rows are not skipped, because they
could carry a conflicting ID. A deferred import runs again on every command until the
source is repaired. Journal rows with the wrong shape (non-dict records, non-string
cwds, null ID arrays) are skipped rather than raised.

**Report mode.** `python -m meridian.lib.ops.legacy_native_import RUNTIME_ROOT` prints
the report JSON. It skips CLI startup, runtime resolution, telemetry, and the automatic
import, so it writes nothing under Meridian state or native stores. It is a
development entry point, not a CLI command.

**Cost.** The first cut appended each bind separately, with three fsyncs per bind
while holding the sessions lock. A synthetic run of 3,000 Claude imports took 180.9 s.
Batching the append in one `SessionBindings` commit brought that to about 0.5 s.


**Provenance:** `work:native-harness-session-identity` (user decision "Auto-import
once" in `decision.md`; brief `prompts/pr1-legacy-import.md`); commits `96e146d0`,
`ce8b6df5`, `8e8fe485`; review `spawn:p7091` (`evidence/pr1-legacy-review.md`),
recheck `spawn:p7093` (`evidence/pr1-legacy-recheck.md`).
