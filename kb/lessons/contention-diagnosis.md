# Contention and Cost Diagnosis

How lock contention and per-operation cost were diagnosed in `work:v010-followups`
(2026-10). The first three sections come from #586 / PR #588 (v0.10.4), where
`meridian spawn wait` failed daily with "History index deadline exhausted"; the
diagnosis went wrong twice before it went right. The decision and mechanism are in
[D-history-index-readers-never-wait-on-session-writers](../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers).
The later sections come from the constant-time program that followed. Its bar, its
violators and its harness are on
[per-operation cost](../architecture/per-operation-cost.md).

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

## Count work; don't time it on a shared box

The development box is shared, and its load swung between about 5 and 60 during
this work. The probe comparing base, #587 and #590 (`spawn:p8044`) started its
batches at 1-minute loads of 41.7, 50.7 and 15.5, against a quiet reference of 5.25.
Its wall times showed regressions and improvements of several seconds that were
just scheduler noise, and the whole timing batch was discarded. Its counts were
deterministic, and they were the useful result. They found that #587's `show
@latest` read every spawn row (2 → about 250–500 `state.json` reads), which neither
code reviewer had caught.

The instrument is a `sitecustomize.py` on `PYTHONPATH`. It counts:
- spawn-directory scans;
- `state.json` opens;
- full `sessions.jsonl` reads;
- lock acquisitions, with summed hold time.

It can capture a stack per event. Counts don't depend on load, they attribute cost
to a caller, and a test can pin them.

**The rules.**
- Make counts the acceptance evidence, and treat wall time as secondary.
- Record `uptime` with every batch, and redo any wall-time batch taken above about
  2× the quiet load.
- Measure several checkouts with one prober, one after another, so their batches
  don't load the box against each other (D23).
- Count one logical operation once. One keyed fold passed through three
  instrumented seams and showed up as three events, so an uncorrected count of
  30–39 "full-log reads" was really 9.

## Warm the index before you measure

A copied corpus starts with a cold history index. In the end-to-end probe, every
orange no-arg wait during a launch returned "discovery pending". That looked like a
launch-path lock held for more than 2 s, and an investigation was opened. It was
the cold catch-up after the copy: 3 of 3 pending cold against 0 of 8 warm (D21).

**The rule.** Run one catch-up on the measured root and check that coverage is
complete before you take samples. Measure the cold build separately, as its own
number (#592).

## Probe the whole command, not just the seams you suspect

The first constant-time audit (`spawn:p7993`) timed the seams it knew about. Its
end-to-end sub-spawn hung, so it estimated an orange launch at "0.8 s startup +
1.16 s bind". A real end-to-end probe with a fake harness then measured 9.12 s and 9
full-log reads (`spawn:p7997`). That was more callers than the audit had listed.
The same counting probe later showed 980 `state.json` reads per launch on base,
which nobody could explain at first. A stack capture traced them to session
teardown, `reclaim_session_owned_scopes_for_chat` → `list_spawns`. That is a path
no seam audit had considered, and it became #612 (`spawn:p8095`).

**The rule.** Run the real command end to end, with a fake harness, at two or
three history sizes. Look at what grows. An unexplained count is a finding to
chase, not noise to average away.

## Related

- [Verification and review discipline](verification-and-review-discipline.md#measurement-discipline):
  provenance for measurements.
- [Review convergence gate](review-convergence-gate.md)
- [Session state](../architecture/state-system/session-state.md#sessionsjsonl-is-one-global-log):
  the remaining full-log readers (#418).
- [Cost cuts that weakened safety checks](cost-cuts-and-safety-checks.md)
