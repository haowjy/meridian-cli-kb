# Derived spawn status

Readers derive whether a launch is live from its lock and never write or signal; only the holder of a claimed lock writes a dead launch's outcome. Contract: the source checkout's `src/meridian/lib/state/AGENTS.md`.

## The launch lock is the liveness evidence

For a lock-backed launch, the runner holds `launch.lock` until it exits. A free lock lets a reader derive the dead launch's outcome without writing it; a stored terminal stays authoritative. Activity mtimes and heartbeats were removed because they can't establish liveness: a quiet live runner looks dead, and a killed runner looks alive until a grace period expires.

A status read holds the shared launch lock across the row read. Probing and releasing first would let a new runner publish an active row in between and be mistaken for dead ([race-test evidence](../../codebase/test-determinism.md#race-tests-must-go-red-against-the-split-implementation)).

## One rule for a dead launch's outcome

A dead launch with a durable report derives success even when cancellation was requested; otherwise cancellation wins over crash. Control frames and synthetic failures don't count as reports ([spawn finalization](../spawn-finalization.md#durable-completion-evidence)). Every reader, repair and cancel path applies this rule, so they converge on the same outcome.

## Select by stored status, report by derived status

Destructive selection and repair use stored status: an active row whose runner died still needs finalization, and archiving it as terminal would prevent that write. Liveness queries use derived status.

## Only a claim authorizes the repair write

The repairer must claim the free lock, and the claim rechecks the observed launch number, so a cancel or repair aimed at an earlier launch can't finalize a reopened one. Cancel likewise pins the observed launch and uses the claim or a birth-checked runner signal ([cancel contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/core/.context/cancel.md)).

The reaper records cleanup targets before finalizing and keeps failed claims for retry: finalization and process cleanup are separate, and a crash after writing the outcome must not erase evidence of remaining work. Unknown process identity and access errors never authorize terminating a primary's scope ([process scope](../process-scope.md)).

## Known gaps

- Parent finish or cancel does not always converge descendants ([#671](https://github.com/haowjy/meridian-cli/issues/671)).
- Dead-launch teardown is spread across callers ([#838](https://github.com/haowjy/meridian-cli/issues/838)).
