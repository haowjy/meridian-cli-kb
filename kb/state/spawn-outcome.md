# Spawn outcome

Readers derive whether a launch is live from its lock and never write or signal; one writer publishes a launch's terminal outcome, and only the holder of a claimed lock writes a dead launch's outcome. Contracts: the source checkout's `src/meridian/lib/state/AGENTS.md` and `src/meridian/lib/core/.context/CONTEXT.md`.

## The launch lock is the liveness evidence

For a lock-backed launch, the runner holds `launch.lock` until it exits. A free lock lets a reader derive the dead launch's outcome without writing it; a stored terminal stays authoritative. Activity mtimes and heartbeats were removed because they can't establish liveness: a quiet live runner looks dead, and a killed runner looks alive until a grace period expires. The harness must not inherit the lock: a descendant could keep it after the runner dies, or drop it during re-exec. The lock says nothing about whether descendants have exited; that is [process scope](../launch/process-scope.md).

A status read holds the shared launch lock across the row read. Probing and releasing first would let a new runner publish an active row in between and be mistaken for dead ([race-test evidence](../codebase/testing.md#race-tests-must-go-red-against-the-split-implementation)).

## One writer, first terminal wins

The runner, or a repairer that claimed its free launch lock, writes the terminal outcome once. A write pinned to an earlier launch can't finalize a later continue. This replaced ranking competing terminal writers: a free lock already proves the previous runner is gone.

Every reader, repair and cancel path applies one rule to a dead launch, so they converge on the same outcome. A report can resolve a dead launch as successful despite a cancel request. Nonempty protocol events and synthetic failure wrappers don't count as completion, because harnesses emit them during cancellation. On the live-runner path, execution facts such as a nonzero exit still outrank report text.

Several triggers can become ready together, so arbitration gives them a fixed order. A watchdog that merely woke on completion must not win as if it had stopped the spawn: making a wake faster can reorder a `FIRST_COMPLETED` race, so check each one when changing wake timing. Signal masking must cover everything that has to finish before a pending signal is redelivered; a mask inside a callee doesn't protect its caller's later cleanup.

## Select by stored status, report by derived status

Destructive selection and repair use stored status: an active row whose runner died still needs finalization, and archiving or hiding it as terminal would keep the writer responsible for repairing it from seeing it. Liveness queries use derived status. Doctor reports a dead but unwritten launch under `dead_spawns_unfinalized` ([troubleshooting](../operations/troubleshooting.md#a-spawn-shows-crashed)).

## Only a claim authorizes the repair write

The repairer must claim the free lock, and the claim rechecks the observed launch number, so a cancel or repair aimed at an earlier launch can't finalize a reopened one. Cancel likewise pins the observed launch and uses the claim or a birth-checked runner signal ([cancel contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/core/.context/cancel.md)).

The reaper records cleanup targets before finalizing and keeps failed claims for retry: finalization and process cleanup are separate, and a crash after writing the outcome must not erase evidence of remaining work. Exactly-once cleanup gating was rejected for the same reason: a process that dies after recording the gate but before terminating the target would leave the target running forever. Unknown process identity and access errors never authorize terminating a primary's scope ([process scope](../launch/process-scope.md)).

## Known gaps

- Parent finish or cancel does not always converge descendants ([#671](https://github.com/haowjy/meridian-cli/issues/671)).
- Dead-launch teardown is spread across callers ([#838](https://github.com/haowjy/meridian-cli/issues/838)).
