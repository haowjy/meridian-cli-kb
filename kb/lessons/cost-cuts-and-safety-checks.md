# Cost Cuts That Weakened Safety Checks

Lessons from the constant-time program (`work:v010-followups`, 2026-10). Several
changes made an operation cheaper, and in the process changed what it guaranteed.
None of them was caught by the cost tests, which all passed. The program and its
bar are on [per-operation cost](../architecture/per-operation-cost.md).

The shared rule comes first, then the four cases it comes from.

**Before you narrow a scan, a wait or a poll, write down every guarantee the slow
version gave, and keep a test that fails if the fast version drops one.** A cost
test (for example "`list_spawns` is never called") proves the work is gone. It can't
prove the check still refuses what it used to refuse. The combined identity
review found exactly this gap: many new cost guards, and no test that a narrowed
guard still rejected a non-linked observer, a cycle through an older run, or a lost
pointer combined with a gap (`review-combined-correctness.md` N5).

## A faster wake changed arbitration order (#605)

The finalize report watchdog polled once a second, so a normal finish waited up to
1 s for it. #605 made the watchdog wait on the completion event instead, which
removed about 1 s of every finalize.

The terminal arbitrator races completion against the watchdog and checks the
watchdog first. That priority order had been safe only because the watchdog
almost never finished in the same loop tick as completion. With the event wake,
completion's done-callback woke the watchdog before `asyncio.wait`'s own callback
ran. Both tasks were therefore done when the arbitrator looked, and the watchdog
branch won on **every normal finish**. It recorded a no-op watchdog decision
instead of a completion (`spawn:p8060`).

The fix made the trigger mean causation: `WATCHDOG` and `INACTIVITY` win only when
that task actually stopped the spawn (its result is `True`). Otherwise arbitration
falls through to completion. The `watchdog_noop` flag became unreachable and was
deleted.

**The rule.** A priority-ordered race assumes its triggers are separated in time.
Removing latency removes that separation. When you make a task finish sooner, check
every `FIRST_COMPLETED` race it joins. A task that merely *woke* must not win over
the task that *caused* the outcome.

## A keyed lookup covered only the linked case (combined #587/#590, B1)

Reopening a chat must refuse if any live run is using it. The base check scanned
every spawn for one whose `current_chat_id` was that chat. The cost fix checked
only the run linked on the chat's session record, which removed an O(spawns) pass
per continue.

A run that natively `/resume`s into a chat sets its `current_chat_id` to that chat,
but nothing links it on the chat's record. With the narrowed check, `start_session`
accepted the reopen, and two harness processes would write one native session
(`spawn:p8096`, reproduced).

The decision was to revert to base parity, the full scan. The cheaper check is not
an index query either, because index catch-up can itself re-read every active row
until #594 lands. It belongs to #418 B3, which records observer rows in the chat.

**The rule.** A keyed lookup is complete only if every path that creates the fact
also writes the key. List those writers (here: launch links the run; native
observation does not) before replacing a scan with a lookup.

## A cycle check followed one edge per node (B2)

The base cycle check treated every run's `parent_id` as an edge. The fast walk
followed only each ancestor's *latest* run. When a child is reopened under a new
parent, its latest edge changes and the original edge disappears from the walk, so
`reserve_run` accepted a reopen that closed a cycle. The walk had also moved outside
`spawns_flock`, so two concurrent cross-reopens could both pass.

The fix walks every surviving run of each ancestor. That is bounded by the
ancestor's own runs, not by the store, so it stays within the cost bar. It repeats
the walk under the lock before allocating.

**The rule.** When identity becomes multi-valued (a spawn now has many runs), every
graph check over it must follow every value's edges. A cost cut that picks a
representative value silently changes the graph.

## A "disposable" pointer that nothing could rebuild (B3)

The branch added `spawn-heads/<pN>` pointers and documented them as "never
authority, rebuilt by scan". No code rebuilt them. With no head, `latest_run`
assumed run 1 and probed upward only through contiguous runs. Gaps between runs are
normal, because prune and archive retire runs that aren't retained. So a lost
head plus a gap made continue resume the wrong generation, made the next
reservation reuse an existing K, and let the busy check read a terminal run while
a live one existed. Restore published runs without writing a head, so the trigger
was not hypothetical.

The fix collapsed two heads into one, the highest reserved run:
- a missing or unreadable head triggers one logged scan of that spawn's runs and an
  atomic rewrite;
- continue reads rows downward from the head, skipping gaps;
- restore writes the head before publishing;
- retiring the last run removes it.

**The rule.** A derived file is disposable only if deleting it at any moment gives
the same answers after the rebuild. Test exactly that: delete it on top of a state
with gaps. Two pointers to the same fact double the ways they can disagree (the
review's S2 found the second head could move backwards).

## Related

- [Contention and cost diagnosis](contention-diagnosis.md): finding the costs.
- [Per-operation cost](../architecture/per-operation-cost.md): the measured bar and
  acceptance evidence.
- [Spawn/run identity decision](../decisions/state.md#d-spawn-run-identity)
- [Spawn finalization](../architecture/spawn-finalization.md#terminal-arbitration-terminalarbitrator):
  the arbitrator as it is now.
- [Review convergence gate](review-convergence-gate.md)

**Provenance:** `work:v010-followups` `decisions.md` D24, D26, D30, D31;
`review-combined-correctness.md` (`spawn:p8096`); `review-combined-structure.md`
(`spawn:p8098`); #605 review `spawn:p8060`; `combined-progress.md` pass 2;
`chat:c7736`.
