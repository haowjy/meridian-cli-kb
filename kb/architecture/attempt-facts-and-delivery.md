# Attempt Facts and Delivery

How usage, failure, "produced output" and the first session ID are computed from
the events an attempt saw live, and how those events reach subscribers, without
depending on the runner's `history.jsonl` stream. Why the stream stopped being the
source: [native-only history](../decisions/native-only-history.md). How the
resulting report is surfaced when reading a chat's transcript: [native transcript
reads](native-transcript-reads.md).

**State:** landed in PR 2, draft PR #526 (`feat/native-reads` @ `3ae3fce8`, stacked on
PR #520). It is not on `main`. Writers below stay until PR 3 deletes them.

## The emit path

**The emit path.** `SpawnManager._emit(spawn_id, event)` is the one emit path for the
drain loop and the manager's own events. Managed primary attach has the same shape. It
runs three steps in order:
1. inline event hooks, through `core/event_hooks.run_event_hooks`, which logs and
   isolates each hook's exception;
2. the history write, only when a writer exists;
3. subscriber fan-out.

`_emit` returns `EmitOutcome = NoWriter | Written | WriteFailed(error)`
(`streaming/spawn_drain_loop.py`). The drain loop matches on it:
- `NoWriter` and `Written` fan out;
- `WriteFailed` counts the failure, traces the real error and skips fan-out, which is
  the pre-PR 2 policy for a failed write.

Manager-authored `emit_event` fans out whatever the outcome. The unused
`EventObserverRegistry` and its lossy queued observer are deleted. Managed primary
attach also touches the spawn heartbeat, and it no longer raises without a writer.
PR 3 deletes the `Written` and `WriteFailed` arms.

## Attempt folds

**Attempt folds.** Each registered extractor is stateless. Its `create_fold()` returns a
per-harness `AttemptFold` subclass (`ClaudeFold`, `CodexFold`, `OpenCodeFold`,
`PiFold`, `CursorFold`) in `harness/extractors/`. The fold holds one attempt's
`AttemptFacts` (`harness/attempt_facts.py`):
- `first_session_id` and `output_seen`;
- `final_text`, capped at 1 MiB, with `text_capped`;
- `native_turn_ids` and `usage`;
- `failure` and `incomplete`.

The base `AttemptFold.__call__` is the one entry point. It normalizes the event kind
once, then applies `accepts`, session observation, generic usage and the harness's
`fold_event`. `bind_scope(session_id)` is the only scope mutation. It is used by the
streaming runner, managed primary attach and `streaming serve`. Claude `--print`
captured stdout (`output.jsonl`) is folded after exit by `fold_stdout`, through the same
hook isolation. A bad byte decodes with replacement, and an unparseable line is skipped.
Both mark the facts incomplete.

**Usage when a fold step fails.** A fold exception marks the facts `incomplete`, and
finalization (`launch/extract.enrich_finalize`) logs `facts_incomplete`.
- If the harness has already set a specific total (`usage_is_specific`, as with a
  Claude `total_cost_usd` result or Pi's totals), that total survives.
- Otherwise the generic usage is dropped to `None` for the rest of the attempt
  (`generic_usage_lost`), never a partial sum passed off as a total.
- Unparseable stdout lines mark `incomplete` but do not drop usage.

The streaming budget check reads the same usage.

**Report precedence** (`launch/report.extract_or_fallback_report`):
1. an explicit `report.md`;
2. a Pi typed failure (`facts.failure`);
3. the exact native reply named by this attempt's events. Only OpenCode V2
   implements `read_native_turn`; it looks up a message ID exactly and never takes the
   latest message in the DB.
4. the fold's `final_text`;
5. the failure reason;
6. nothing.

Usage is `None` when it is unknown, never zero. `streaming serve` guards enrichment:
a failure is logged, and the terminal row is still written, with no usage.

A crashed runner loses its in-memory fold. It is finalized from `report.md` and
lifecycle facts only.

## Pi lifecycle sidecar

**Pi lifecycle sidecar.** A harness bundle declares `event_sinks(runtime_root,
spawn_id)`, which defaults to none. Pi's sink writes phase events to
`spawns/<id>/pi-lifecycle.json` through `state/pi_lifecycle.record`:
- it stores the last phase and cleanup status per attempt;
- the whole read-modify-write runs under `mutate_published_spawn_artifact`, so a late
  phase cannot recreate a deleted spawn.

Hooks are removed only after teardown finishes, so cleanup phases survive shutdown.
`spawn show` reads the file through the typed `pi_lifecycle.read`. The manager and
attach have no Pi branch.

## Other stream readers removed

**Other stream readers removed:**
- the reaper's history-mtime liveness leg. The activity artifacts are `heartbeat`,
  `bash-records.json`, `stderr.log` and `report.md`, plus process and report evidence.
- the empty-output failure artifact written into the history path;
- `_MERIDIAN_GUARDRAIL_OUTPUT_LOG`, replaced by `_MERIDIAN_GUARDRAIL_REPORT` and
  `_MERIDIAN_GUARDRAIL_CHAT_ID`;
- the transcript-available check, which now tests whether `continue_chat_id`'s record
  has a native key. It filters events before one per-chat fold (493 → 71 ms).
- artifact-mode extraction (`harness/extractor.py`), `current_attempt_lines`, the
  `LocalStore` history redirect, and OpenCode's ambient-DB report fallbacks.

`streaming serve` prints `Transcript: meridian session log pN`.

## What is still written

Writers stay until PR 3 (`evidence/pr3-deletion-list.md`):
- `HarnessHistoryWriter` in the drain loop and primary attach;
- the retry header write and `meridian.attempt.completed` marker;
- `write_retained_child_stream`;
- `last-observed-event.json` and the reaper's diagnostic read of it.

**History-blind test mode.** `pytest --runner-history=off` makes writers absent and
traps every read of a spawn, attempt or artifact `history.jsonl`. Subprocesses inherit it
through a test-only `sitecustomize`. It installs the read trap and a meta-path hook that
patches the writer modules (`state/history`, `spawn_manager`, `primary_attach`) when they
are imported. This covers `python -m meridian`, console scripts and `runpy` alike. At
`3ae3fce8`, the whole suite fails only on the 16 writer tests PR 3 deletes.

## Related Pages

- [Native transcript reads](native-transcript-reads.md) — resolver, reader and
  search projection that read the transcript this page's report precedence points at
- [Native session binding](native-session-binding.md) — the runner pipeline that
  produces the events this page's folds consume
- [Native-only history decision](../decisions/native-only-history.md) — why the run
  stream stopped being the source of run facts, and the retained rationale for what
  is still written
- [Pi lifecycle](pi-lifecycle.md) — Pi's spawned-session lifecycle and quiescence

**Provenance:** same work item as [native transcript
reads](native-transcript-reads.md), which lists the full lane and review provenance
for PR 2: `work:native-harness-session-identity`, code checked at `feat/native-reads`
@ `3ae3fce8`.
