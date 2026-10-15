# Contention Diagnosis

Lessons from #586 / PR #588, where `meridian spawn wait` failed daily with
"History index deadline exhausted". The decision and the mechanism are recorded in
[D-history-index-readers-never-wait-on-session-writers](../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers).
This page covers how the diagnosis went wrong twice before it went right.

## A deadline error is a contention symptom: find the holder before designing recovery

The first #588 design looked reasonable. It classified the error as a typed
`HistoryIndexBusy`, retried catch-up in 2 s attempts inside a 15 s budget, and gave
no-arg wait a "discovery pending" checkpoint. It took three coder rounds, and two
review rounds restructured it (`spawn:p7952`, then the tech lead). Every reproduction in
that work held a lock by hand. The repros proved that the retry logic handled
contention, but none of them measured what was holding the lock in real use.

The user's objection reset the work: even 2 s is too long for a metadata read. The
tech lead then measured. `meridian spawn list` took 0.82–1.06 s in total, of which
about 0.8 s was CLI startup, and `HistoryIndex.spawns()` took 0.126 s. On a copy of real state,
catch-up took 4–169 ms for 0–100 changed sources (`spawn:p7970`). An uncontended read
was 20× under the deadline. So every timeout meant a reader queued behind some
holder, and a longer budget would only have made the queue longer. The holder was a
full fold of a 46.6 MB `sessions.jsonl` under its writer lock, on every session
append. That took 1,463 ms per append in the runtime where 14 of the 15 incidents
happened (`spawn:p7975`). Removing the fold, and letting readers stop waiting on
that lock, brought reader P95 from 2,001 ms to about 100 ms. The retry budget was
deleted.

**The rule.** When an operation exhausts a deadline on a lock or gate:

1. Measure the uncontended cost first. If it is far below the deadline, the deadline
   is not the problem.
2. Inventory the holders: what each lock guards, what work runs under it, and how
   long it is held on real data, from copies of real runtime state, not synthetic
   fixtures. Correlate them with the incident ledger (which runtime, which sources).
3. Only then design recovery. A held-lock repro is the right test *for the recovery
   path*. It says nothing about why the lock is held.

The first design wasn't wasted: the typed `HistoryIndexBusy`, the distinction between
contention and exhaustion, and the wait checkpoint all stayed. What changed was
the order of work: shorten the holder, then report whatever contention remains
honestly. Do not wait it out.

## Check a cause ranking against the protocol

The lock-holder investigation (`spawn:p7970`) measured the session writer's
0.7–1.5 s lock hold and then ranked it out. The slow fold runs *before*
`HistoryChanges.mark()`, and marker-to-release took only 5.45 ms, so "it does not
explain a reader waiting on that marker." Each measurement was right. The
conclusion missed one protocol detail.

The `sessions` marker is one file per source, and it stays pending until a reader
acknowledges it under the source lock. Under steady session churn it is almost
always pending. A reader that drains it must take `sessions.jsonl.flock`, and so
waits through *every later writer's* pre-mark fold, not just the one that marked.
It also holds the exclusive catch-up gate while it waits, so every other reader
queues behind it. "Slow before mark" is harmless for a writer whose marker is
consumed once. It is the whole problem for a per-source marker that never settles.
The tech lead caught this by tracing the reader's path through the marker protocol.
The fold probe (`spawn:p7975`) then confirmed it: with the fold, 5 of 30 readers
failed; without it, 0 of 30.

**The rule.** Before acting on an investigator's ranking, walk at least the top
dismissed candidate through the protocol it was dismissed by. Ask who waits on what,
for how long, and while holding what. A ranking built from per-phase timings can be
accurate phase by phase and still wrong about who blocks whom.

## A pending label can name a bystander

The incident errors listed unresolved sources, and nearly every set included
`sessions`. That pointed the right way for the wrong reason. Under churn the
sessions marker is pending most of the time, so it appears in busy messages it didn't
cause. In the review's stress run, a message blamed `sessions` when the real
timeout was on the catch-up lock (`spawn:p7987`). #588 changed busy messages to name
the lock that timed out (`FileLockTimeout.path`) instead of listing pending sources.
When an error lists "what was pending", treat that list as context, not as the cause.

## Related

- [Verification and review discipline](verification-and-review-discipline.md#measurement-discipline):
  provenance for measurements.
- [Review convergence gate](review-convergence-gate.md)
- [Session state](../architecture/state-system/session-state.md#sessionsjsonl-is-the-remaining-global-on-log):
  the remaining O(log) holders (#591).
