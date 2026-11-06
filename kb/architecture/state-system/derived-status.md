# Derived spawn status

## Stored and derived status

For a lock-backed launch, the runner holds `launch.lock` until exit. A free lock allows readers to derive the dead launch's outcome without mutating it; a stored terminal remains authoritative. The report-first dead-launch rule makes every repair or cancel path converge on the same result. Control frames and synthetic failures do not count as completion reports. See [spawn finalization](../spawn-finalization.md#durable-completion-evidence-classification) and the source checkout's `src/meridian/lib/state/AGENTS.md`.

A status read holds the shared launch lock across the row read. Probing and releasing the lock first would allow a new runner to publish an active row before the read and be mistaken for dead. See [race-test evidence](../../codebase/test-determinism.md#race-tests-must-go-red-against-the-split-implementation).

## Selection versus liveness

Derived status answers whether work appears live. Destructive selection and repair use stored status: an active row whose runner died still needs finalization, and archiving it as terminal would prevent that write. See [state decisions](../../decisions/state.md).

## Who persists a dead launch's outcome

Only a successful exclusive claim authorizes the repair writer. The claim rechecks the observed launch number so a cancel or repair for an earlier launch cannot finalize a reopened one. Readers never persist or terminate processes. See the source checkout's `src/meridian/lib/state/AGENTS.md`.

### The reaper's cleanup claims

The reaper records cleanup targets before finalization and retains failed claims for retry. Finalization and process cleanup are separate: a crash after writing the outcome must not erase evidence of remaining work. Unknown identity and access errors do not authorize primary-scope termination. See [process scope](../process-scope.md).

## Cancel on the lock

Cancel pins the observed launch and uses the claim or a birth-checked runner signal, so a concurrent continue cannot redirect the action to the next launch. See the source checkout's `src/meridian/lib/core/.context/cancel.md`.

## Deleted mechanisms: do not bring back

Activity mtimes and heartbeats cannot establish liveness: a quiet live runner may appear dead and a killed runner can appear alive until a grace expires. The launch lock supplies the evidence directly. See [spawn identity](../../decisions/spawn-identity.md).

## Known gaps

- Parent finish or cancel does not always converge descendants ([#671](https://github.com/haowjy/meridian-cli/issues/671)).
- Derived finish time for a crashed streaming child can be close to start when no activity files changed ([#841](https://github.com/haowjy/meridian-cli/issues/841)).
- Streaming serve teardown can exceed its intended stop budget ([#842](https://github.com/haowjy/meridian-cli/issues/842)).
- Dead-launch teardown remains spread across callers ([#838](https://github.com/haowjy/meridian-cli/issues/838)).
