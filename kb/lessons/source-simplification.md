# Source Simplification Lessons

Lessons from the `architecture-refactor-roadmap` work item (2026-05), specifically the Phase 8.6 source-seam and test-collapse pass. This page documents what worked, what was over-corrected, and the principles that should guide future simplification work.

---

## Deletion-First Simplification Is Correct — but Deletion Is a Hypothesis

The Phase 8.6 goal was to reduce concepts and entropy, not merely move lines. The final result: −3,005 LOC net across 10 files, with all four reviewer gates passing (verifier, reviewer, refactor-reviewer, alignment-reviewer).

The key insight: **deleting a seam is a hypothesis that the seam's contract is either trivially enforced at call sites or not worth enforcing independently.** Test collapse proved the hypothesis for most of the targeted suites. One suite (primary_launch + compiler) disproved it — over-collapse was caught at review and corrected with minimal contract tests.

The correction pattern matters: reviewers found over-collapse, and the fix was to write small targeted behavior contracts — not to restore the original broad mock-heavy suites. The lesson is not "be conservative about deleting tests." It is "when a reviewer finds a gap, fill it with a contract test, not a suite."

---

## Unit Tests That Pin Implementation Shape Are Presumptively Deletable

The targeted test suites (`test_cli_main.py`, `test_cli_mars.py`, `test_cli_spawn.py`, `test_primary_launch.py`, `test_codex_ws.py`, `test_git_autosync.py`, `test_compiler.py`) shared a pattern:

- Mock-heavy: mocked the immediate collaborators, not the boundary behavior
- Implementation-pinning: broke on internal renames without catching real regressions
- Structural rather than behavioral: tested "was this internal function called with these args?" not "does this component do its job?"

These tests had negative long-term value because every refactor required updating them even when no behavior changed. The policy derived from this:

> A unit test is presumptively deletable if its primary failure mode is an internal rename rather than a behavioral regression. Retain it only when it covers a stable seam that genuinely needs an independent contract.

The positive rule: keep tests at stable seams with explicit contracts (compiler precedence, legacy override handling, timeout resolution, Codex WS fail-closed behavior, git autosync pure helper logic). These survived because they test behavior the caller cares about, not internal wiring.

---

## Smoke and Behavior Tests Are the Higher-Value Tier

After the collapse, the surviving coverage was behavior-first:

- Mars passthrough/env/agent-mode behavior (functional, not structural)
- Spawn CLI glue/output behavior (what the command produces, not how)
- Primary launch resume/fork/cross-harness wrapper contracts (cross-boundary behavior)
- Compiler precedence/legacy/timeout/child override contracts (policy, not plumbing)
- Codex WS fail-closed behavior (safety property)
- Git autosync pure helper logic (pure function, stable contract)

The consistent pattern: the retained tests specify what the component is *for*, not how it is built internally. This is the test tier that survives refactoring.

---

## Source Seam Removal Requires Moving Ownership, Not Just Deleting Code

Two seam removals in Phase 8.6 were structural:

1. **`SpawnOperationServices` / `resolve_spawn_operation_services` removed from `ops/spawn/api.py`**: cancel and cancel-all now inline root/service resolution directly. The seam was a service-locator pattern with one consumer — the indirection added no value and obscured what was actually happening.

2. **Workspace table parsing/merge moved from `state/paths.py` to `config/workspace.py`**: The state layer was loading and parsing workspace config — a config concern. The move enforced the correct ownership boundary: state reads files; config interprets their content.

The lesson: when removing a seam, identify which layer *should* own the logic and move it there. Deleting without re-homing leaves orphaned logic scattered at call sites.

---

## Over-Collapse Is Recoverable — Restore Contract Coverage, Not Full Suites

Phase 8.6 initially collapsed more than was warranted in the primary_launch and compiler test suites. The refactor reviewer caught this and p5491 restored minimal contract coverage.

The key discipline: when restoring collapsed tests, restore *contracts*, not *implementation pinning*. A contract test verifies observable behavior at a boundary. An implementation-pinning test verifies internal wiring. Only the former should survive.

If reviewers had not caught the over-collapse and it had been detected later, the correct fix would still be the same: add targeted behavior contracts, not restore the original suite.

---

## Cross-References

- [principles/design-principles.md](../principles/design-principles.md#14-behavior-tests-over-implementation-pinning-tests) — durable testing principle extracted from Phase 8.6
- [codebase/guide.md#testing-strategy](../codebase/guide.md#testing-strategy) — existing smoke-first testing guidance that this lesson refines
- [decisions/launch.md](../decisions/launch.md) — spawn pipeline seam decisions
- [decisions/workspace.md](../decisions/workspace.md) — workspace config ownership decisions
- [architecture/workspace/overview.md](../architecture/workspace/overview.md) — workspace config ownership boundary
- [architecture/workspace/resolution.md](../architecture/workspace/resolution.md) — workspace table ownership after Phase 8.6
- [open-questions/future-work.md](../open-questions/future-work.md#source-simplification--post-phase-86-remaining-targets) — remaining non-blocking simplification targets
