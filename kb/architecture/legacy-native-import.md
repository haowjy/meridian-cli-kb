# Legacy Native Session Import

## Legacy import

Implemented in combined PR #534 (`feat/native-session-identity` @ `08499af0`, against
`main`); #520, #526 and #531 are closed as superseded. The rules and user decision are in the
[decision](../decisions/legacy-native-import.md).

**Trigger.** `ops/runtime.py`'s `resolve_runtime_authority_for_read` and
`resolve_runtime_authority_for_write` call
`ops/legacy_native_import.maybe_import_legacy_native_sessions(runtime_root)`.
Commands that never resolve a runtime root, such as `--help`, do not trigger it. A
root without `sessions.jsonl` is left alone, so an untouched project gets no state
files. When the marker `<runtime_root>/legacy-native-import-v1.json` exists, the
importer returns after checking it; late binding is a separate repair described below.

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
not blocked behind the Codex walk or the OpenCode reads.

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
| OpenCode | Resolved DB path from recorded env, else the default | The adapter's exact reader, in place, opened `mode=ro`; SQL errors raise |
| Pi | Meridian's unscoped Pi sessions root (where interactive primaries live), plus `<root>/<spawn_id>/` for the chat's own spawns | Pi `session` header `id`; a match in more than one candidate is `ambiguous`. 0.6.7 recorded no Pi ID, so its Pi chats are `no_session_id` ([open decision](../decisions/legacy-native-import.md#open-old-pi-chats-that-067-never-bound-decision-pending)) |
| Cursor, historical records | none: `unsupported` | — |

**OpenCode reads in place.** The import calls the same exact `mode=ro` reader that
live reads use, on the live database. It writes the WAL shared-memory read marks
that any SQLite reader writes, OpenCode included. An earlier cut copied the DB and
WAL to a private snapshot and fingerprinted it to detect a torn copy. That cost about
6 GB of scratch and 17–26 s per attempt on the 6.3 GB live DB, and it re-ran on every
deferred attempt while OpenCode kept writing. What still matters from that cut: a
lookup that hits `sqlite3.Error` raises instead of reporting "absent". A swallowed
error once made 201 of 251 sessions look absent, and the marker would have made that
permanent ([lesson](../lessons/native-session-identity.md#a-once-only-marker-turns-transient-failures-into-permanent-ones)).

**Deferral.** `SpawnStateQuarantined`, `OSError` and `sqlite3.Error` are caught
around the whole import. Then:
- no marker is written;
- the command prints `Native session import deferred for …` and continues;
- an atomic `legacy-native-import-deferral.json` note starts a 15-minute backoff,
  so commands inside the window skip the import quietly;
- success removes the note and writes the marker.

Quarantined spawn rows are not skipped, because they could carry a conflicting ID.
Journal rows with the wrong shape are skipped rather than raised: non-dict records,
non-string cwds, null ID arrays.

## Late binding after the marker

An old build can finish writing a native session ID to a chat after the once-only
import has already recorded that chat under `unbound.no_session_id`. The ordinary
import does not scan or repair it on later reads. `bind_late_legacy_sessions()` uses
the typed `ImportReport` marker to consider only those recorded misses, resolves an
exact store and validates the native header, then rechecks the chat identity and
generation while binding. Each eligible row is recorded as attempted, including
non-matches; malformed marker data causes no mutation. This repair runs only in
`meridian doctor` and primary-launch background repairs. It does not run on every
command, browse row, or transcript read.

**Report mode.** `python -m meridian.lib.ops.legacy_native_import RUNTIME_ROOT` prints
the report JSON. It skips CLI startup, runtime resolution, telemetry, and the automatic
import, so it writes nothing under Meridian state or native stores. It is a
development entry point, not a CLI command.

**Cost.** The first cut appended each bind separately, with three fsyncs per bind
while holding the sessions lock. A synthetic run of 3,000 Claude imports took 180.9 s.
Batching the append in one `SessionBindings` commit brought that to about 0.5 s.
On a copy of the real meridian-cli root, the whole import, 2,133 of 7,004 chats,
took 3.75 s.


**Provenance:** `work:native-harness-session-identity` (user decision "Auto-import
once" in `decision.md`; brief `prompts/pr1-legacy-import.md`); commits `96e146d0`,
`ce8b6df5`, `8e8fe485`; review `spawn:p7091` (`evidence/pr1-legacy-review.md`),
recheck `spawn:p7093` (`evidence/pr1-legacy-recheck.md`); in-place OpenCode read,
deferral backoff and Pi root candidates `evidence/pr1-install-readiness-report.md`
(commits `5695f5c9`, `38e40862`, `a3769c39`).
