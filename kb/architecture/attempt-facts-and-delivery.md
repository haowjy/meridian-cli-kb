# Attempt Facts and Delivery

How usage, failure, "produced output" and the first session ID are computed from
the events an attempt saw live, and how those events reach subscribers. No runner
`history.jsonl` stream is involved; Meridian no longer writes one. Why the stream stopped being the
source: [native-only history](../decisions/native-only-history.md). How a chat's
conversation is read from the harness's native transcript: [native transcript
reads](native-transcript-reads.md).

**State:** implemented in combined PR #534 (`feat/native-session-identity` @
`08499af0`, against `main`); #520, #526 and #531 are closed as superseded. Runner
history is neither read nor written.

## The emit path

Each event reaches three consumers, all in memory, in this order:
1. **inline event hooks**, through `core/event_hooks.run_event_hooks`, which logs and
   isolates each hook's exception. The attempt fold and harness event sinks (the Pi
   lifecycle sidecar) are hooks.
2. **subscriber fan-out**;
3. for drained spawns, **`coordinator.note_event_delivered(event)`**, then terminal
   handling.

`SpawnManager._run_event_hooks(spawn_id, event)` runs step 1 and returns nothing.
The drain loop receives it as `SpawnDrainLoop(run_event_hooks=…)` and does fan-out,
the coordinator note and terminal classification itself
(`streaming/spawn_drain_loop.py`). Manager-authored events go through `emit_event`:
hooks, fan-out, then a trace. Managed primary attach
(`launch/process/primary_attach._consume_live_events`) runs the same hooks, touches
the heartbeat and folds attempt facts.

Nothing can fail between hooks and fan-out, so there is no outcome to match on.
**Deleted in PR 3:** `EmitOutcome` (`NoWriter | Written | WriteFailed`), the writer
registry, the drain loop's abort after ten consecutive write failures, and the
primary-attach writer. `note_event_persisted` was renamed `note_event_delivered`
because nothing is persisted. The unused `EventObserverRegistry` and its lossy queued
observer were deleted in PR 2.

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
3. the exact native reply named by this attempt's events. OpenCode V2 looks up its
   message ID exactly. OpenCode 1.x has no V2 `assistantMessageID`, so its fallback
   reads the final assistant response from this attempt's exact bound native session;
   it never selects an ambient or merely newest session.
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

## The runner stream is retired

PR 3 deleted every writer of runner `spawns/<id>/history.jsonl`:
- `state/history.py` as a whole: `HarnessHistoryWriter`, sequence envelopes, tail
  repair, causal rehydration, the `last-observed-event.json` checkpoint,
  `write_retained_child_stream` and `ingest_portable_history`;
- the writer-only managed-primary causal tracker (`state/managed_primary.py`);
- the retry header write and the `meridian.attempt.completed` marker. A retry now
  rotates only `runner-lifecycle.jsonl`, `stderr.log`, `tokens.json` and `report.md`
  into `attempt-N/`;
- the reaper's `last_observed_event` orphan evidence. Liveness evidence is unchanged.

New spawns and primaries create neither file. Old files stay on disk until the user
prunes them ([runner-history prune](../operations/session-archive-pruning.md#runner-history-prune)).

`launch/constants.RETIRED_RUNNER_STREAM_FILENAMES` (`history.jsonl`,
`last-observed-event.json`) names the retired files once. Its owners are the
`session log --file` rejection (option C), legacy ZIP inventory, the atomic-temp
detection in `retention_archive`, and prune. No new code should use it to write.

**History-blind test mode.** `pytest --runner-history=off` traps every read of a
spawn, attempt or artifact `history.jsonl` (`tests/support/runner_history_blind/`).
Only `state.retention_archive` may read one, to hash legacy ZIP members.
Subprocesses inherit the trap through a test-only `sitecustomize`, which also covers
`python -m meridian` children. There are no writers left, so PR 3 retired the
writer patches and the import hook that applied them. The suite passes in both modes
with no known-failure list. A test that compares runner fixtures before and after must
compare `lstat` results, not bytes, or it trips the trap
([lesson](../lessons/native-session-identity.md#a-test-helper-that-reads-runner-bytes-trips-the-blind-trap)).

## Related Pages

- [Native transcript reads](native-transcript-reads.md) — resolver, reader and
  search projection that read the transcript this page's report precedence points at
- [Native session binding](native-session-binding.md) — the runner pipeline that
  produces the events this page's folds consume
- [Native-only history decision](../decisions/native-only-history.md) — why the run
  stream stopped being the source of run facts and then stopped being written
- [Pi lifecycle](pi-lifecycle.md) — Pi's spawned-session lifecycle and quiescence

**Provenance:** same work item as [native transcript
reads](native-transcript-reads.md), which lists the full lane and review provenance
for PR 2: `work:native-harness-session-identity`, code checked at `feat/native-reads`
@ `3ae3fce8`. PR 3: `evidence/pr3-deletion-list.md`, `evidence/pr3-p3a-report.md`
(`spawn:p7155`), `review/pr3-review.md` (`spawn:p7161`), fix lane E
`evidence/pr3-fix-e-report.md` (`spawn:p7162`); code checked at
`feat/stop-runner-history` @ `c1fa08e4`.
