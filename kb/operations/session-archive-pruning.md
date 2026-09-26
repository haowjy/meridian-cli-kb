# Operations: Session Archive Pruning

This page documents the explicit cleanup operation for retired runner-history files. It is separate from transcript reads and from `meridian doctor`. The decision rationale and measured impact are in the [native-only history decision](../decisions/native-only-history.md#the-prune-rule).

## Runner-History Prune

`meridian session archive --prune-runner-history [--apply] [--after-days N]` deletes
runner-stream files that Meridian stopped writing in PR 3 and never reads, for spawns
whose exact native source makes them redundant. It lives in
`ops/runner_history_prune.py` and is wired from `session_archive_sync`. The output is
the `runner_history` field of `SessionArchiveOutput`.

- **Dry run by default.** It prints each spawn it would prune with bytes, a total, a
  count and bytes per skip reason, the quarantined IDs with a `meridian doctor`
  pointer, and errors. `--apply` deletes. `--after-days` defaults to 14.
- **Refused combinations:** refs, `--eligible`, `--list` and `--destination`.
- **Never automatic.** `session_stop_maintenance` and `history.archive.automatic` do
  not call it, and a test covers this.

**Qualification.** Only spawns that still have runner-stream files are considered.
`_judge` returns a typed `_Verdict`, either a skip reason or the native source paths.
The first failing check is the skip reason:

| Order | Check | Skip reason |
|---|---|---|
| 1 | Not a historical (restored) record | `historical` |
| 2 | Status is terminal | `running` |
| 3 | `terminal.finished_at` parses | `no_terminal_time` |
| 4 | Finished more than N days ago | `recent` |
| 5 | `run_boundary` is `None` or `verified` | `exit_unresolved` |
| 6 | No unreleased, likely-serving process scope | `live_scope` |
| 7 | `session_target.resolve_run_sources(row, sessions)` resolves the log chat and the entry chat exactly | the `NativeSessionUnavailable` reason (`unbound`, `missing`, `ambiguous_native_file`), or `error` for any other exception |

Check 7 runs the same `_spawn_target` chain as `session log pN`: `continue_chat_id` →
`native_key()` → `adapter.resolve_native_session_file`. Adapters validate the header
or DB row. The sessions projection is built once per pass.

**What gets deleted.** `runner_stream_files` looks under `spawns/<id>` and legacy
`artifacts/<id>`, plus their `attempt-<n>/` subdirectories. In each it takes the
`RETIRED_RUNNER_STREAM_FILENAMES` (`history.jsonl`, `last-observed-event.json`) and
their `.<name>.*.tmp` atomic temps. It takes only regular files under `lstat`, and
skips symlinked directories and `attempt-N.tmp` staging directories. `state.json`,
`report.md`, logs, `pi-lifecycle.json`, native data, ZIPs and `sessions.jsonl` are
never touched.

**Apply.**
- The pass holds `history-archives/archive.lock`.
- Each spawn is unlinked inside `mutate_published_spawn_artifact`, which holds the
  shared history-mutation lock and the spawn lock. `can_mutate` requires the record
  to equal the planned one (prompt excluded) and re-runs `resolve_run_sources`.
  Otherwise the spawn is kept and reported.
- Each spawn's body runs in its own `try`; a failure becomes `"<id>: <error>"` and
  the pass continues.
- Files are unlinked one by one with `missing_ok`, so a rerun after a crash converges.
- `HistoryIndex.catch_up()` runs afterwards. The index does not project these files.

Capture and ZIP archive still work after pruning: capture needs a sealed native
snapshot, and runner files were never an archive source. A prune → capture → archive
test checks that the ZIP holds `native-transcript.jsonl` and no runner members.
