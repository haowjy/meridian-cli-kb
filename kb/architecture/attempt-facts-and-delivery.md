# Attempt facts and delivery

## The emit path

In pending PR #547 (source checkpoint `c1b64bcf`), drained spawns normalize once before
this consumer ordering; the completion coordinator may drop duplicate events
or refine descriptors while preserving their raw event envelopes. Pi's `agent_end` frame therefore still
reaches raw hooks and subscribers; only its decoded attempt outcome is private
until `agent_settled`.

The Pi settlement refinement does not turn every Pi telemetry frame into activity:
`summarization_retry_*` and `queue_update` remain raw facts without activity semantics.
Automatic runs and ordinary compaction retain separate lifecycle facts, and arbitrary
third-party branch-summary lifecycles remain outside the supported safety boundary.

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
(`launch/process/primary_attach._consume_live_events`) runs the same hooks and folds
attempt facts.

Nothing can fail between hooks and fan-out, so there is no outcome to match on.
**Deleted in PR 3:** `EmitOutcome` (`NoWriter | Written | WriteFailed`), the writer
registry, the drain loop's abort after ten consecutive write failures, and the
primary-attach writer. `note_event_persisted` was renamed `note_event_delivered`
because nothing is persisted. The unused `EventObserverRegistry` and its lossy queued
observer were deleted in PR 2.

## Attempt folds

Attempt facts are folded from the live events of one turn; native transcripts serve conversation reads separately. If a fold fails, a known specific usage total survives but a generic partial sum is discarded rather than presented as a total. Unknown usage is not zero. A crashed runner loses its in-memory fold, so recovery relies on durable report and lifecycle evidence. See [native-only history](../decisions/native-only-history.md) and the source checkout's `src/meridian/lib/harness/extractors/`.

## Pi lifecycle sidecar

Pi's sidecar records phases through the published-spawn mutation boundary, so a late phase cannot recreate a deleted aggregate. Hooks remain until teardown finishes, preserving cleanup phases.
