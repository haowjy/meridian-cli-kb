# Testing Decisions

Meridian assigns tests by the behavior they own, not by how many dependencies
are mocked. The unit tier is the pure functional core; every real boundary has a
more honest home.

## Test tiers follow behavioral boundaries

| Tier | Owns |
|---|---|
| Unit | Pure policy, parsing, state transitions, algorithms, and other functional-core decisions |
| Integration | Behavior across real seams such as filesystems, subprocesses, persistence, and module composition |
| Contract | API payloads, response shapes, retry policy, and type contracts that must not drift |
| Platform | OS behavior such as signals, process scope, encoding, and locking |
| Smoke | CLI-visible workflows exercised through human-readable scenarios |

Mocking a filesystem, process, or HTTP collaborator does not turn its
choreography into unit behavior. A test belongs in integration when it proves a
real seam, with only the final external boundary stubbed. It belongs in contract
when the stable behavior is the request, response, or retry shape rather than a
live service interaction.

The named exception is a coherent security suite. Light in-process dry-run
composition and temporary configuration scaffolding may remain in unit when
keeping the suite together makes one fail-closed boundary legible and
reviewable. This is not a general exception for composition tests.

## Aggressive deletion requires named evidence

Deleting mock-heavy tests is safe only after the deletion risk register has
been resolved. For every gated file:

1. Close its prerequisite coverage gaps before deletion.
2. Verify each named invariant at its surviving tier; broad claims such as
   “integration covers this” are insufficient.
3. Accept and record any remaining risk explicitly rather than preserving a
   low-value suite as reassurance.

Re-tiering means consolidating the named behavior into the destination tier.
Do not copy a suite and leave duplicate coverage behind. A moved test that
still asserts mocked collaborator choreography has changed directories, not
tiers; rewrite it to cross the real seam or place a stable payload/retry
contract in the contract tier.

## Rejected alternatives

**Conservative pruning was superseded by the aggressive tier correction.**
The conservative audit retained tests because they covered an edge in
isolation. That bar preserved filesystem, subprocess, signal, network, polling,
and timing choreography under `unit`. The adopted bar asks whether the test
protects pure project logic or a named boundary invariant, then deletes or
re-tiers everything else.

**A local HTTP server was rejected for the OpenCode session API suite.** The
suite owns payload, retry, and timeout shapes, not the behavior of an HTTP
server. Contract-tier placement states that ownership directly without
inventing a service integration that adds setup and timing failure modes.

**The nested-deny security suite was not fragmented across tiers.** A small
amount of in-process launch composition is cheaper than splitting one
fail-closed policy proof across unit and integration. Its coherence makes
omissions visible during security review.

## Fake executables must become observable before teardown

Process tests must not assume that starting a child means its shim has executed.
On a contended CI runner, even a cold interpreter boot can lose a race with
process teardown before the shim records its argv.

Prefer a POSIX `sh` shim when the fake only needs to log arguments, emit fixed
output, or stay alive. Poll the expected observation with a bounded deadline
before stopping the process. The bound is a hang guard; the recorded command or
state transition is the assertion.

## Provenance

- `work:workstream-roadmap`, test-pruning slice, sprint PRs 3 and 4
- Approved prune specification: `aggressive-prune-list.md`
- Decision log: `DECISIONS.md`, decisions 2, 5, 6, and 8
- Execution record: `EXECUTION-DAG.md`
