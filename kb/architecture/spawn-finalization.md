# Spawn finalization

## One writer, first terminal wins

The runner, or a repairer that claimed its free launch lock, writes the terminal outcome once. A write pinned to an earlier launch can't finalize a later continue. This replaced ranking competing terminal writers: a free lock already proves the previous runner is gone ([derived status](state-system/derived-status.md)).

## Durable completion evidence

A report can resolve a dead launch as successful despite a cancel request. Nonempty protocol events and synthetic failure wrappers don't count as completion, because harnesses emit them during cancellation. On the live-runner path, execution facts such as a nonzero exit still outrank report text. Contract: the source checkout's `src/meridian/lib/core/.context/CONTEXT.md`.

## Terminal arbitration

Several triggers can become ready together, so arbitration gives them a fixed order. A watchdog that merely woke on completion must not win as if it had stopped the spawn ([cost cuts and safety checks](../lessons/cost-cuts-and-safety-checks.md)).

Signal masking must cover everything that has to finish before a pending signal is redelivered; a mask inside a callee doesn't protect its caller's later cleanup.
