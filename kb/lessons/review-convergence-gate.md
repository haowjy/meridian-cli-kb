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

## Related

- [Verification and Review Discipline](verification-and-review-discipline.md)
- [Package Management Decisions D91/D92](../decisions/package-management.md)
- [Residue Cleanup Discipline](residue-cleanup-discipline.md)
- [Verification Campaign History](verification-campaign-history.md)
