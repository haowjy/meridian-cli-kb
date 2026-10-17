# Decisions: Completion Drain

When a managed spawn is allowed to publish its terminal status after the
harness reports a finished turn. The mechanism is in
[drain plans](../architecture/drain-plans.md) and
[completion drain coordination](../architecture/completion-drain-coordination.md);
Pi's own profile is in [Pi runtime decisions](pi-runtime.md).

---

## D-claude-native-hold: a Claude child holds for its own background work through the resident profile (2026-10) {#d-claude-native-hold}

**Status:** decided by the tech lead with the owner's direction (`chat:c8112`,
`work:claude-bg-quiescence` D1–D8). Issue #655. Built on branch
`fix/claude-bg-quiescence` (head `29f93150d`), with no PR opened or merged yet.
**On `main` today:** a Claude child is finalized at its first `result`, its
process group is killed, and it is published `succeeded`, even when its own
background shells or subagents are still running.

### Problem

Claude Code lets an agent start background work: `run_in_background` shells and
background subagents (the Agent tool). The agent ends its turn saying it will
continue when the work reports. Meridian had never given Claude a drain plan, so
`drain_plan_factory.py` fell through to the plain plan: the first `result`
finalized the spawn, teardown killed the group, background work included, and the
status was `succeeded`.

Coders' verification gates (`pnpm check`) were cut off mid-suite while their
reports said "the gate is still running", and leads saw `succeeded`
(meridian-flow p11457, p11949, p13303, p13328, 2026-10-12 to 2026-10-17). A bisect
(`spawn:p8382`) found every meridian version back to v0.6.7 bad, on Claude Code
2.1.301 and 2.1.302 alike: not a regression. Raw `claude -p` handled the same
repro correctly. Why it went unfixed for weeks is a
[lesson](../lessons/spawn-lane-operations.md#a-claude-lanes-background-gate-was-killed-and-reported-succeeded).

### Decision

1. **Hold on Claude's own signal.** Claude Code's stream-json emits
   `system/background_tasks_changed` with the full current `tasks` array whenever
   it changes. The spawn holds while the latest snapshot is non-empty and finalizes
   on Claude's own `result` after it empties. The facts behind the rule are in
   [drain plans](../architecture/drain-plans.md#claude-native-holds).
2. **Reuse the resident profile; no Claude-only copy.** Pending Claude work is a
   hold in the same completion profile Codex/OpenCode use: deadline, nudges,
   `meridian spawn done` / `meridian spawn rearm`, rearm budget, then `timed_out`.
   The owner: "having the same pi/codex rearm system would be better." Claude
   reaches it through a connection capability, `completion_evidence()`, not a
   harness-id branch.
3. **A native hold is not sticky.** A rearm on a Claude task hold only extends
   the deadline. Claude's natural follow-up `result` with an empty snapshot always
   finalizes `succeeded`. Codex/OpenCode explicit holds keep requiring `done`.
4. **An explicit `done` always wins; the deadline bounds only waiting.** A `done`
   consumed before the deadline is completion authority. A Claude `done` sent
   mid-turn waits for the turn's closing `result` up to the deadline, then
   publishes `succeeded` with the latest report. A hold that never gets `done`
   ends `timed_out` at the deadline.
5. **Cut-off work is never `succeeded`.** Cancel, the outer timeout, the deadline
   and harness exit while held all give a non-success status.
6. **Codex/OpenCode keep main's behavior** except three shared fixes: stale
   validation is cleared for each new candidate; nudge urgency uses the resolved
   interval, not a fixed 270 s; the interval is capped at half the deadline (inert
   at the 1800 s default). Everything else is gated on `native_hold`.

### Rejected

| Alternative | Why rejected |
|---|---|
| Prompt agents to run gates in the foreground | Tried since 2026-09-25; the bug recurred 4+ times anyway |
| Report honestly but keep finalizing at the first result | Stops the false `succeeded` but still kills the gate |
| Scan Claude's task directories, or infer tasks from tool calls | Guesses, and an exact stream signal exists |
| "Just let Claude finish", with no bound | Raw `claude -p` waits forever on a never-ending task such as a dev server (lead probe: `sleep 600` still waiting at a 90 s cap) |
| A one-shot Claude-specific nudge (`spawn:p8386`, cancelled) | A Claude-only copy of hold and nudge logic the resident profile already owns |
| The deadline beats a late `done` for every resident harness (implementer, round 3) | Never a lead decision; reports `timed_out` after the agent declared completion and cancels a Codex/OpenCode spawn's descendants |
| The deadline beats `done` for native active turns (`spawn:p8398`) | Same: `timed_out` after an explicit completion |
| Native rearm makes the hold sticky, as for Codex | Natural completion would then need a `done` the agent has no reason to send |

### Consequences and residuals

- Every Claude child now runs under the resident coordinator. Review
  (`spawn:p8390`) confirmed no regression for `--bg` grandchild spawns, but the
  parity gap stands: Codex children hold until their descendant *spawns* finish
  and Claude children don't (#224).
- `done` authorizes cleanup: leftover tasks are killed by normal managed-scope
  teardown.
- If a Claude Code version stops emitting snapshots, Claude children fall back to
  first-result finalization, as before this decision.
- Meridian's spawn guidance tells Claude agents to run `meridian spawn wait` with
  `run_in_background`. With the hold that becomes safe; re-read it once the fix
  ships.

**Revisit when:** Claude Code changes or drops `background_tasks_changed`, or
#224 gives Claude children a descendant-spawn hold, which would join this one in
the same profile.

**Provenance:** `work:claude-bg-quiescence` `decisions.md` D1–D8,
`verification-p8385.md`, `investigation-early-end-bg-task.md`, `bisect-early-end.md`;
investigator `spawn:p8374`; bisect `spawn:p8382`; implementer `spawn:p8385`,
`spawn:p8388`, `spawn:p8395`, `spawn:p8401`; reviewers `spawn:p8390`,
`spawn:p8391`, `spawn:p8397`, `spawn:p8398`; `chat:c8112`.
