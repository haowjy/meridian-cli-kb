# Review Convergence Gate

Repeated review findings are a redesign signal when they share a defect class,
not merely a file. The gate below distinguishes healthy convergence from a
patch loop and forces schema-breaking changes to prove old-state behavior.

## The Gate

1. **Freeze the review target.** Reviewers inspect the same commit or worktree.
   A moving target makes recurrence and severity impossible to interpret.
2. **Classify the defect, not just the symptom.** Name the violated invariant:
   partial desired intent, ownership without evidence, wrong-root verification,
   unreadable old state treated as absence, or another precise class.
3. **Track severity and location.** Falling severity across independent seams is
   convergence. The same or rising severity at one seam means the abstraction
   is still wrong.
4. **State total desired intent.** For any replacement or retention pipeline,
   define what the complete desired output means. Omission must not silently
   mean deletion when omission can also mean unreadable or unprocessed input.
5. **Escalate the boundary.** After same-class recurrence, replace branch-level
   exceptions with one gate or type boundary that removes the invalid state.
6. **Probe old state at runtime.** A schema-breaking release is incomplete until
   the prior installed shape has been exercised by the new binary.

## Old-State Runtime Probe

Construct the actual prior-release artifact—lock, config surface, generated
file, or persisted row—then run the new command end to end. Observe disk after
failure as well as after success.

The minimum matrix is:

| Case | Required observation |
|---|---|
| Readable old state | It upgrades or is retained according to the declared migration contract. |
| Unreadable owned state | The command halts before destructive compilation/apply, or preserves exact ownership evidence. |
| Partial failure | Unrelated desired outputs and prior records are not silently dropped. |
| Retry after repair | The next run converges without manual cleanup beyond the documented recovery step. |

This probe must run against the built artifact or real command path. A unit test
of the decoder alone cannot prove gate placement, write ordering, or exit code.

## Why the Pattern Worked

In the Mars hook-fragment campaign, repeated attempts carried exceptions deeper
into diff, target, and lock persistence. Each new persistence site could forget
the exception. The converged design instead halted recovery commands before
compilation when removed-schema hook surfaces were unreadable. Deleting the
exception state eliminated an entire class of omissions.

The same reasoning applies outside package management: if every caller must
remember a special state, move the rule to the boundary all callers cross or
change the representation so forgetting is impossible.

## Lane Reviews Do Not Review the Composition

Parallel lanes each get a review and a recheck on a frozen head, but every lane
reviewer is scoped to its lane. In the native-session-identity work (PR #520) three
cycles ran: lane review, recheck of each fix pass, then one review of the integrated
whole change against the full contract. Every cycle found real defects, and the
whole-change review found three P1s that no lane review could see because each sat
between lanes: a runner that no lane owned, a merge that promoted one lane's
diagnostic into another lane's allocator, and validation that each lane assumed
another had done.

Apply the gate across cycles, not just within one:

- **New seams each cycle is convergence.** The findings shrank in kind from design to
  enforcement (make the typed route and exact-key rule hold at the remaining seams),
  and the reviewer said the architecture need not change. That justified another fix
  pass instead of a redesign. A P1 on a seam already fixed would have meant stop and
  write a redesign brief rather than patch a fourth time.
- **Review the integrated head before release,** against the whole contract (the
  review above scored coverage per contract item: 2 covered / 5 partial / 1 drift,
  then 8/8 on recheck). Green lane gates and green merge gates are not that review.
- **Evidence that decided it:** installed-wheel probes with decoys and an unrelated
  concurrent session, plus one real-process run of the external harness. Each found
  something the suite did not.
- **Steering a running implementer.** Reserve `spawn inject` for direction that comes
  from the human. A hint the orchestrator generated itself, such as "your step 10
  should also read this new report", goes into the next brief or a follow-up spawn,
  so the running agent's instructions stay the user's.

## A review brief must not carry the lead's verdict

**What happened (2026-10-14, PR #581).** The tech lead asked a thermo-nuclear
reviewer whether a deeper restructure should block the PR and wrote "my default is
no". The reviewer prototyped the restructure and reported the following:
- it was smaller than the PR's own patch (net −44 lines);
- it fixed the root cause in the same function;
- it closed a second issue.

Its verdict was still "No — I agree with your default." The evidence pointed one
way and the verdict echoed the brief. The lead read the evidence and folded the
restructure into the PR
([decision](../decisions/codebase-cleanup.md#d-unknown-keys-from-pydantic-pydantic-does-the-traversal-for-unknown-key-warnings)).

**The rule.** On a block-or-defer question, the brief should ask for evidence:
size, risk, what the change fixes and what it costs. It should leave out the lead's
lean. Share the constraints that bind the answer, such as "one issue, one PR" or a
scope budget. Withhold the answer you expect. The lead owns the verdict on a strict
review and reaches it from the report. A default stated in the brief becomes the
reviewer's starting point, and a strict reviewer that starts from the lead's answer
is no longer an independent check.

## Delete the obsolete work before adding a cancellation layer

When review keeps finding races between a slow path and its cancellation or
preemption, ask whether the slow work should run at all. The Pi result watcher
accumulated a budget, preemption and replay flags around always-on scanning;
removing the scanning (finite collector, no polling during active runs, direct reads) removed the race class.
Positive hints such as a candidate cache may narrow work but must not authorize
a conclusion of absence. (Provenance: `work:pi-result-delivery`; see
[Pi runtime decisions](../decisions/pi-runtime.md).)

## Related

- [Verification and Review Discipline](verification-and-review-discipline.md)
- [Package Management Decisions D91/D92](../decisions/package-management.md)
- [Residue Cleanup Discipline](residue-cleanup-discipline.md)
- [Verification Campaign History](verification-campaign-history.md)
- [Native session identity: green suites did not find the seam defects](native-session-identity.md#green-suites-did-not-find-the-seam-defects)
