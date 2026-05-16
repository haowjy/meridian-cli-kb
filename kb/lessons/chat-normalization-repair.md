# lessons/chat-normalization-repair — Lessons from Repairing Harness Normalization Drift

The 2026 normalization repair was not a new chat design. It was a recovery from protocol drift: live harnesses were emitting real assistant and tool activity that no longer satisfied the existing frontend contract.

## What Broke

Three failure modes mattered:

- **Codex invisible turns:** raw turns could contain `turn/started`, `item/agentMessage/delta`, `item/completed`, and `turn/completed`, but assistant text or command results were not always projected into the normalized stream correctly.
- **Claude aggregated tool drops:** first-turn text worked, but persistent turns with aggregated `assistant` `tool_use` and `user` `tool_result` blocks lost tool lifecycle visibility.
- **OpenCode snapshot drift:** text, reasoning, and tool parts were split across `message.part.delta`, `message.part.updated`, `message.updated`, `session.status`, and `session.idle`; without part-role tracking, valid assistant output and tool completion snapshots were dropped.

The shared symptom was the same: `turn.started` and `turn.completed` existed, but the user-visible middle of the turn was missing or incomplete.

## Contract First, Harness Second

The repair only stabilized one contract:

- Assistant text -> `content.delta`
- Reasoning -> `content.delta`
- Tools -> `item.started` / `item.updated` / `item.completed`
- Exactly one `turn.completed` per actual turn

That contract is more durable than any one harness protocol. The mistake was letting normalizer behavior follow individual raw shapes too literally instead of forcing those shapes back into the existing event families.

## Raw Event Drift Is Normal

Harness event shapes changed without a frontend redesign:

- Codex started surfacing useful assistant text on both streamed `item/agentMessage/delta` and terminal `item/completed` `agentMessage` snapshots.
- Claude exposed important tool lifecycle state in aggregated `assistant` and `user` events, not only in streaming block events.
- OpenCode required state stitched across deltas, snapshots, and session status changes before a coherent turn could be projected.

The durable lesson: treat each harness normalizer as a compatibility adapter for evolving live protocols, not as a one-time parser for a fixed schema.

## Completion Dedupe Has To Be Defensive

Synthetic `meridian/turn_completed` exists because some persistent harness sessions do not always emit a clean terminal boundary Meridian can rely on. The bug class here was subtle: a synthetic completion could fix one turn while mutating state so the next real turn was partially suppressed.

Rules that held up:

- Real terminal events win over synthetic ones.
- Dedupe by actual raw turn ID when available.
- Synthetic completion may close a turn, but it must not register a fake new turn identity.
- Resetting completion state must unblock the next real `turn.started` and `turn.completed`.

Completion handling is stateful enough that regressions often appear only on second or later persistent turns. Unit coverage has to include those sequences.

## Recovery and Replay Are Part of the Normalization Contract

The bug is not fixed if live streaming works but replayed history diverges. Normalized event history feeds:

- `ChatEventLog` JSONL persistence
- `ChatEventIndex` rebuilds
- recovery on backend restart
- browser replay after reconnect

That means a normalizer change is not isolated to rendering. It changes durable history semantics. Captured raw histories need to replay through recovery paths, not just direct unit normalization calls.

See [architecture/chat/normalization.md](../architecture/chat/normalization.md), [architecture/chat/event-pipeline.md](../architecture/chat/event-pipeline.md), and [architecture/chat/recovery.md](../architecture/chat/recovery.md).

## Where to Look in the Code

- `src/meridian/lib/chat/normalization/codex.py`
- `src/meridian/lib/chat/normalization/claude.py`
- `src/meridian/lib/chat/normalization/opencode.py`
- `src/meridian/lib/chat/normalization/common.py`
- `src/meridian/lib/chat/event_pipeline.py`
- `src/meridian/lib/chat/runtime.py`
- `src/meridian/lib/chat/recovery.py`
- `src/meridian/lib/chat/event_index.py`

## Tests That Matter

- `tests/unit/chat/normalization/test_codex.py`
- `tests/unit/chat/normalization/test_claude.py`
- `tests/unit/chat/normalization/test_opencode.py`
- `tests/integration/chat/test_harness_parity.py`

The unit tests should include captured raw histories for:

- Codex assistant-only turn
- Codex tool + assistant turn
- Claude persistent second turn with `tool_use` / `tool_result`
- OpenCode text/reasoning/tool lifecycle with `message.updated`, `session.status`, and `session.idle`

## Live Smoke Lessons

The final live smoke produced two operational lessons worth keeping:

- **Short temp paths matter.** Long isolated temp directories triggered `AF_UNIX path too long` before any harness turn started. Use short temp roots for isolated local chat smokes.
- **Codex model support can look like normalization failure.** In this environment, `codex` and `gpt` were rejected by the provider/account combination, while a literal `gpt-5.4-mini` run succeeded in a separate focused smoke. Separate provider/model support failures from normalization regressions before debugging the chat pipeline.

Relevant reports:

- Work design brief: `work/codex-dev-tailscale-no-response/design/normalization-repair-brief.md`
- Final live smoke: `work/codex-dev-tailscale-no-response/final-live-smoke-report.md`
- Focused Codex literal-model smoke: `work/codex-dev-tailscale-no-response/smoke-codex-gpt-5.4-mini-literal.md`
