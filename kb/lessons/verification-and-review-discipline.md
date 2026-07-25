# Verification and Review Discipline

A check is evidence only when it exercises the changed behavior, observes a
non-vacuous oracle, and reports failure from the environment where the product
actually runs. Review convergence is a separate question: recurring findings
of one class mean the seam or invariant is wrong, not that another local patch
is needed.

## Verification Validity

### Prove the check consumed the capability

A token, marker, or callback proves that *some* check ran. It does not prove the
call under test used it. Seed or replace the capability immediately before the
subject call, make that call the sole possible consumer, then assert both the
behavior and consumption. Otherwise earlier setup can spend the evidence and
leave the regression path untested.

### Make the oracle non-vacuous

An empty traversal, zero matched files, ignored exit status, or assertion on an
unrelated layer can all report green while examining nothing. Every verifier
must establish its preconditions: expected root, nonzero candidate count where
appropriate, the intended command, and a propagated failure status.

### Test below the gate that originally blocked the defect

A regression test is not protective when an earlier validation gate rejects the
fixture before the changed logic runs. Use the narrowest legitimate seam that
reaches the defect class, or add an integration case whose input passes all
preceding gates. Confirm the test fails before the fix when making a regression
claim.

### A diff is not runtime evidence

Static inspection proves source shape. It does not prove import order,
registration side effects, filesystem behavior, process races, or actual CLI
output. Pair structural review with a runtime probe at the real seam whenever
those behaviors matter.

### Verify the search scope contains the fact

A negative search result is evidence only when the search target can contain
the fact being searched for. Grepping root config files for declarations that
live inside compiled package internals cannot find them regardless of search
quality. This is not "should have searched harder" -- it is "searched a
location that could not hold the answer."

Before trusting a negative result, verify that the search scope covers the
domain structure where the fact would live. The cost of a false negative in
blast-radius assessment is an undetected breaking change; the cost of
checking the scope is one question.

The [release-sequencing](release-sequencing.md) lesson has a concrete instance:
a blast-radius assessment grepped sibling `mars.toml` files for `[[hooks]]`
and concluded no consumers existed. Hooks live in `hooks/<name>/hook.toml`
inside the resolved package, a location the search never reached.

### Record exit status and working directory

A copied success-looking line is not a validation record. Capture the command,
exit status, working directory, and the revision or artifact under test. A
wrong-CWD validator can inspect an empty or different tree and still exit zero;
a shell pipeline can hide an upstream failure unless pipe status is preserved.

## Measurement Discipline

Measurements need a recoverable provenance note: command or method, dataset,
environment/date, and source artifact or commit. Report scope with the number.
A historical benchmark is not a current guarantee, and repeated numbers without
a canonical evidence pointer become lore.

For source-size changes, state whether generated files, fixtures, and moves are
included. For performance, distinguish cold/warm runs and dataset size. Other
pages should link to the canonical measurement rather than copying it.

## From Repeated Finding to Redesign

When the same class recurs in one function or pipeline, stop repairing branches.
Write the total-intent invariant, identify where partial representations become
possible, and move enforcement to a boundary that makes omission impossible.
The [Convergence Gate](review-convergence-gate.md) owns this process, including
frozen-worktree review, severity trajectories, and old-state runtime probes.

## Historical Evidence

The rules above came from two multi-round campaigns involving Mars ownership,
lock-schema recovery, symlink-following hashes, wrong-CWD checks, and CI trigger
behavior. The chronology is retained in [Verification Campaign History](verification-campaign-history.md),
but it is evidence for these rules rather than the reading order of this page.

## Related

- [Review Convergence Gate](review-convergence-gate.md)
- [Source Simplification](source-simplification.md)
- [Residue Cleanup Discipline](residue-cleanup-discipline.md)
