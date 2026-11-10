# Spawn finalization

## One writer, first terminal wins

The runner or a repairer that claims its free launch lock writes the terminal once. A write pinned to an earlier launch cannot finalize a later continue. This replaces a ranking of competing terminal writers: a free lock establishes the previous runner is gone. See [spawn identity](state-system/spawn-launches.md) and the source checkout's `src/meridian/lib/state/AGENTS.md`.

## Durable completion evidence classification

A report may resolve a dead launch as successful despite a cancel request. Nonempty protocol events and synthetic failure wrappers are not completion: harnesses can emit them during cancellation. Execution facts such as a nonzero exit still outrank report text on the live-runner path. See `src/meridian/lib/core/.context/CONTEXT.md`.

## Terminal arbitration (TerminalArbitrator)

Several triggers can become ready together; arbitration gives them a defined order. A watchdog that merely woke on completion must not win as if it had stopped the spawn. See [cost cuts and safety checks](../lessons/cost-cuts-and-safety-checks.md#a-faster-wake-changed-arbitration-order-605) and `src/meridian/lib/launch/streaming/terminal_arbitrator.py`.

Signal masking must cover the work that must finish before a pending signal is redelivered; a mask inside a callee does not protect its caller's later cleanup. See the source checkout's `src/meridian/lib/core/lifecycle.py` and `src/meridian/lib/launch/streaming_runner.py`.
