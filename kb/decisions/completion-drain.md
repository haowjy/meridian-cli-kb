# Decisions: Completion Drain

When a managed spawn is allowed to publish its terminal status after the
harness reports a finished turn. The mechanism is in
[drain plans](../architecture/drain-plans.md) and
[completion drain coordination](../architecture/completion-drain-coordination.md);
Pi's own profile is in [Pi runtime decisions](pi-runtime.md).

---

## D-claude-native-hold: a Claude child holds for its own background work through the resident coordinator (2026-10) {#d-claude-native-hold}

**Status:** decided by the tech lead with the owner's direction (`chat:c8112`,
`work:claude-bg-quiescence` D1–D14). Issue #655, merged as PR #659
(`25e3ac345`), released in v0.10.21. **Before v0.10.21:** a Claude child was
finalized at its first `result`, its process group killed, and it was published
`succeeded`, even when its own background shells or subagents were still running.

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
2. **Reuse the resident machinery; no Claude-only copy.** Pending Claude work
   runs under the same `ResidentDrainCoordinator` Codex/OpenCode use, with the same
   deadline, nudges, `meridian spawn done` / `meridian spawn rearm`, rearm budget,
   then `timed_out`. The owner: "having the same pi/codex rearm system would be
   better." Claude reaches it through a connection capability,
   `completion_evidence()`, not a harness-id branch.
   **As merged (D11–D13):** the policy is split rather than flagged.
   `build_drain_plan()` picks one profile per spawn: `NativeTaskHoldProfile`
   (Claude, straight-line) when the connection supplies completion evidence,
   otherwise main's `_ResidentCompletionProfile` (Codex/OpenCode). Both compose the
   shared `RearmWindow` for signals, the absolute deadline, the rearm budget and
   the nudge interval. Native tasks reach the profile as ordinary `blocked`
   assessments with `native_task` blockers, so `WorkAssessment` keeps main's shape.
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
   at the 1800 s default). Everything else is the native profile's alone, so
   "Codex unchanged" holds by construction: main's resident tests pass verbatim.

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
| A `native_hold` flag on `WorkAssessment` branching inside the shared resident profile (round 2) | Feature checks scattered across shared code; Codex gained unused state and stayed safe only through early returns (`spawn:p8406`) |
| A base class with abstract `evaluate` hooks for the two profiles | Leaks each profile's phases into the base (`spawn:p8406`); composition through `RearmWindow` instead |
| Move child completion into a Claude Code mod (D14) | The finish decision must live outside the process it watches: Meridian owns the deadline and the kill, and a mod's timers die with a wedged Claude. It would be a second, JS-only copy of the deadline/nudge/rearm policy; the documented mod API lists subagents but not background shell tasks, which `background_tasks_changed` covers; and mod loading depends on version and org policy, so the stream-json path would still be needed |

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
  `run_in_background`. With the hold (v0.10.21+) that is safe; re-read the
  guidance against the shipped behavior.
- Mods remain open for **TUI primaries**, where Meridian has no inject and no
  background-task view (D14): `$.prompt.submit` for inject, `$.tool.register` for a
  native done/rearm tool, `$.ui.status` for child status. Additive only. A
  follow-up probe (`work:claude-mod-inject`) is checking whether a mod can replace
  the stdin inject pipe.

**Revisit when:** Claude Code changes or drops `background_tasks_changed`, or
#224 gives Claude children a descendant-spawn hold, which would join this one in
the same profile.

**Provenance:** `work:claude-bg-quiescence` `decisions.md` D1–D14,
`verification-p8407.md`,
`verification-p8385.md`, `investigation-early-end-bg-task.md`, `bisect-early-end.md`;
investigator `spawn:p8374`; bisect `spawn:p8382`; implementer `spawn:p8385`,
`spawn:p8388`, `spawn:p8395`, `spawn:p8401`, `spawn:p8407`; reviewers
`spawn:p8390`, `spawn:p8391`, `spawn:p8397`, `spawn:p8398`, `spawn:p8406`,
`spawn:p8408`, `spawn:p8409`; `chat:c8112`.
