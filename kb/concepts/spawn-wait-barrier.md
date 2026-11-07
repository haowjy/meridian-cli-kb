# Spawn wait barrier

`meridian spawn wait` blocks on a fixed set of launches and yields a checkpoint when work remains. A fixed set lets the caller choose when to start another barrier instead of allowing new work to extend the wait indefinitely. A yield is not a failure; an explicit timeout is a separate hard limit.

A no-argument wait finds active descendants within the caller's scope and adds unconsumed direct background results. Nested callers exclude siblings and ancestors, avoiding a wait on their own parent. Newly discovered work is advisory and does not enlarge the original barrier. See the source checkout's [src/meridian/lib/ops/spawn/.context/CONTEXT.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/.context/CONTEXT.md) for the contract and [child result delivery](child-result-delivery.md) for consumption and its failure windows.

An observing wait pins a launch across checkpoints, so continuing the same spawn cannot replace the result being awaited. See [spawn launches](../architecture/state-system/spawn-launches.md). A busy history index can yield a discovery checkpoint; retry rather than treating it as an empty set.
