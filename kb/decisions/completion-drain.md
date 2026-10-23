# Decisions: Completion Drain

When a managed spawn is allowed to publish its terminal status after the
harness reports a finished turn. The mechanism is in
[drain plans](../architecture/drain-plans.md) and
[completion drain coordination](../architecture/completion-drain-coordination.md);
Pi's own profile is in [Pi runtime decisions](pi-runtime.md).

## D-completion-report-contract: the confirmation says what the report is; `done` keeps the retained reply (2026-10-23) {#d-completion-report-contract}

**Decision status:** settled intent (tech lead, `work:completion-hardening` D2,
D4). **Implementation status:** shipped in PR #737 (v0.12.4), which closed
[#725](https://github.com/haowjy/meridian-cli/issues/725).

A child under the completion hold ([drain plans](../architecture/drain-plans.md#resident-completion))
reports the reply that formed the candidate `done` accepts.
When a successful turn becomes the first candidate, the drain captures that
turn's text, and finalization prefers it over anything the child writes later,
including its answer to the confirmation turn. A granted `rearm` drops the
candidate and its text, so the next candidate captures again. The confirmation
message says so: the previous reply is kept as the report; if it is
final, run `meridian spawn done` and end the turn; if more work remains,
including waiting on background tasks or spawns, run `meridian spawn rearm`
first, then finish with a complete report.

Pi is the exception. Its plan keeps the reply to its own confirmation message,
so the Pi message says that reply is the report and must be complete. Because
the two plans keep different text, only the confirmation message, which knows
its plan, states a report rule. The generic spawn guidance in every child's
system prompt does not.

**Why.** #725 reported a reviewer spawn "accepted on silence" while its
background suite ran, with the report "The suite is at about 34%". The spawn's
records showed the hold had worked: all three confirmation messages named the
running native tasks, the child rearmed twice, then killed the suite itself and
ran `done`. The defect was the report. The old message said "run `meridian spawn
done` and then provide your complete final report", but that later report was
discarded in favour of the 34% reply the candidate had captured. Every
confirmation turn also paid for a report rewrite nothing kept.

**Rejected.**

- *Latest text wins.* Brings back the loss fixed in `5c4cc7764`, where a child
  that answered the confirmation with "Done." replaced its whole report.
- *Recapture on any later successful turn without `done`.* The confirmation
  turn is itself such a turn. Telling continued work from a plain confirmation
  reply would need harness-specific tool tracking.

**Known gap.** With an exhausted `resident_rearm_budget`, a child cannot rearm,
so text from work it does after a confirmation never replaces the retained
report. The refusal tells it to run `done`. Tracked in meridian-cli
`src/meridian/lib/streaming/.context/FUTURE`; revisit if exhausted budgets appear in practice.

## D-completion-channel: a child that cannot signal completion gets no confirmation turn (2026-10-23) {#d-completion-channel}

**Decision status:** settled intent (tech lead, `work:completion-hardening` D1,
D4, D5). **Implementation status:** shipped in PR #737 (v0.12.4); its
read-only Codex exception is superseded on the unmerged `feat/sandbox-network` branch.

The connection declares whether the launched child can write Meridian's
completion signals (`completion_signals_available`). A hold without that
channel is still a hold: it waits on descendants and native tasks, holds on
unreadable evidence and times out at the deadline. It sends no confirmation
message, reads no `done` or `rearm` signal, accepts a ready candidate directly,
and publishes an empty accepted-signal set, so `meridian spawn done <id>` from
outside is refused. No maintained harness connection currently reports this
unavailable state, but the drain contract still handles it.

**Why #737 originally disabled the channel for read-only Codex.** `meridian spawn done`
takes Meridian's state lock, which the sandbox could not write (`Errno 30` on
`spawns.jsonl.flock`). Measured on
`gpt-6-luna`, each confirmation turn cost about 24–29k tokens and one failed
shell call, then the child's report described the failure instead of its
answer. Under the behavior available then, nothing was gained: a read-only child could not
start Meridian spawns,
and its background processes end with its turn, so it never has work worth
holding for. The shipped `explorer`, `cold-reader` and `session-miner` profiles
are read-only Codex and among the most-spawned.

**Rejected.**

- *Remove only the `done`/`rearm` sentence from the message.* The cost is the
  turn, not the sentence.
- *Give Codex a writable path or a non-filesystem signal channel.* This was
  rejected while the only known fix weakened the read-only sandbox. The later
  execpolicy probe found a narrower mechanism that runs only the standalone
  `meridian spawn` prefix outside it, so this rejection no longer controls.
- *Model the missing channel as requested-then-failed confirmation flags.*
  Capability is named state on the profile; confirmation settlement derives
  from it.

**2026-10-23 follow-up.** Meridian now maintains a user-wide Codex execpolicy rule for
resolved `read-only` and `workspace-write` launches. `meridian spawn done` runs outside the
sandbox through that rule, so `CodexConnection` no longer overrides
`completion_signals_available`; read-only Codex follows the ordinary confirmation protocol.
Live p9089 received the confirmation turn and wrote `done` with exit 0. The rules-file
decision and its user-wide cost are in
[D-codex-child-spawn](sandbox-platform-access.md#d-codex-child-spawn).

**Residual.** Meridian launches Codex even if it cannot write its rules file, reporting the
projection as partial. A read-only child can then receive a confirmation whose signal fails
until the file is repaired. A profile with no resolved `sandbox` has a related gap: Meridian
does not project the file even if the user's Codex config defaults to read-only, because it
does not infer sandbox policy from that config
([sandbox inference deleted](tool-permissions.md#d-codex-sandbox-inference-deleted)). No
shipped profile omits `sandbox`. Both cases belong to the same effective-sandbox/reporting
boundary rather than a read-only-only connection override.

**Provenance (both decisions):** `work:completion-hardening` `decisions.md`
D1, D2, D4, D5; `evidence/725-root-cause.md` (`spawn:p8966`),
`evidence/codex-readonly-completion.md` (`spawn:p8965`),
`evidence/rearm-same-turn.md`, `evidence/live-readonly-final.md`; reviews
`spawn:p8977`, `spawn:p8983`, `spawn:p8990`. The follow-up is
`work:harness-sandbox-network`, especially `spawn:p9089`.

---

## D-child-completion: confirm each retained candidate once (2026-10-20)

**Decision status:** settled intent. **Implementation status:** shipped in
v0.11.3 through [PR #708](https://github.com/haowjy/meridian-cli/pull/708).
Supersedes D-claude-native-hold's separate native policy, sticky resident rearm,
first-result leaf completion, and fixed deadline assumptions. The native
background-task observation remains valid.

A finished harness turn is a completion candidate, not proof that the child has
finished all its work. Claude, Codex and resident OpenCode children use one
best-effort confirmation turn per candidate. `done` accepts the retained
candidate; `rearm` rejects it and waits for a genuinely new successful/idle
candidate. Candidate-less signals are discarded, rather than carried forward.
Silence or unavailable confirmation accepts only ready evidence. Known blockers
hold without explicit `done`; unreadable evidence never permits success.

This keeps ordinary completion automatic without stranding a finished child
behind a mandatory signal. It also prevents a silent confirmation from abandoning
live descendants. Claude's native tasks and persisted Meridian descendants feed
the same hold policy instead of parallel policies with drifting precedence.
OpenCode 2.x has a resident backend and participates; Pi keeps its distinct
quiescence contract, and genuinely plain plans have no completion hold.

The default child completion deadline inherits the calling harness's
`wait_yield_seconds`; explicit `timeouts.resident_deadline_seconds` wins.
Only a granted rearm extends the absolute window. The caller's recursive wait
checkpoint remains independent. Their shared default duration bounds the
caller's idle/compaction exposure, not Codex prompt-cache retention.

Signal commands advertise the selected plan's real capability through a side
file, not a strict spawn-row field that older readers would quarantine.
`done` leaves Meridian descendants running; parent cancel cascade and
dead-descendant convergence remain separate work in [#671](https://github.com/haowjy/meridian-cli/issues/671). Signed-in Codex's
obsolete fixed idle cache warning is tracked separately by
[#707](https://github.com/haowjy/meridian-cli/issues/707).

### Spawn visibility is a convenience, not completion

The descendant-aware hold and `spawn wait` output are the correctness path.
The always-visible Claude status row in
[PR #706](https://github.com/haowjy/meridian-cli/pull/706) was rejected and
closed without merge. Occasional inspection belongs in a pull-based surface:
low-priority idea [#709](https://github.com/haowjy/meridian-cli/issues/709)
proposes an on-demand `/meridian-spawns` menu with no persistent row or
background polling while closed. It is not planned work; build it only if real
usage shows that the existing CLI views are insufficient.

**Rejected:** sticky rearm; repeating confirmation on a timer; treating signal
file presence as candidate authority; accepting silence with live or unknown
work; a deadline based on the child's harness rather than the caller's;
hard-coded 1800/3300-second defaults; Codex cache lifetime as the rationale.

Mechanism and edge conditions: [drain plans](../architecture/drain-plans.md).

**Provenance:** `work:child-completion`, `decisions.md` K6–K9 and round-three
ownership correction; `DIVERGENCE/confirmation-handshake.md`;
review reports `spawn:p8721`–`spawn:p8725`.

---


## D-claude-native-hold: a Claude child holds for its own background work through the resident coordinator (2026-10) {#d-claude-native-hold}

**Status:** superseded by [D-child-completion](#d-child-completion-confirm-each-retained-candidate-once-2026-10-20). The record below preserves the earlier decision and its rationale, not the current contract. Originally decided by the tech lead with the owner's direction (`chat:c8112`,
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
- Meridian's spawn guidance tells agents the no-arg `meridian spawn wait` may run
  in native background execution, and forbids background-wrapping only the `--bg`
  launch. It stays as written: with the hold, a Claude child that backgrounds its
  wait and ends its turn is kept alive and takes the follow-up turn (`spawn:p8385`:
  follow-up turn, `succeeded`, no nudges), and a TUI primary was never affected
  because Claude's own task notification drives its follow-up turn. Tightening it
  to "foreground wait only" would block the agent's turn for nothing
  (`work:claude-mod-inject` D5).
- A mod cannot replace the stdin stream-json pipe Meridian injects child messages
  through: plain `claude -p` exits about 2.5 s
  after going idle, so nothing outside can reach it, and Claude children keep
  `--input-format stream-json`. A TUI primary is injected through Claude Code's
  own inbox socket rather than a mod
  ([D-claude-primary-inbox](primary-inject.md#d-claude-primary-inbox)).

**Revisit the surviving native-task observation when:** Claude Code changes or
drops `background_tasks_changed`. The former descendant-hold revisit condition
was resolved by the shared
[D-child-completion](#d-child-completion-confirm-each-retained-candidate-once-2026-10-20)
policy.

**Provenance:** `work:claude-bg-quiescence` `decisions.md` D1–D14,
`verification-p8407.md`,
`verification-p8385.md`, `investigation-early-end-bg-task.md`, `bisect-early-end.md`;
investigator `spawn:p8374`; bisect `spawn:p8382`; implementer `spawn:p8385`,
`spawn:p8388`, `spawn:p8395`, `spawn:p8401`, `spawn:p8407`; reviewers
`spawn:p8390`, `spawn:p8391`, `spawn:p8397`, `spawn:p8398`, `spawn:p8406`,
`spawn:p8408`, `spawn:p8409`; `chat:c8112`.
