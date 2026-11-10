# Spawn wait barrier

`meridian spawn wait` blocks on a fixed set of launches and yields a checkpoint when work remains. A fixed set lets the caller choose when to start another barrier, instead of letting new work extend the wait forever. A yield is not a failure; `--timeout` is a separate hard limit. Contract: the source checkout's [`ops/spawn/.context/CONTEXT.md`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/.context/CONTEXT.md).

A no-argument wait finds active descendants in the caller's scope and adds owed finished results ([child result delivery](child-result-delivery.md#no-argument-wait-includes-owed-results)). Nested callers exclude siblings and ancestors, so a child never waits on its own parent. A waiting process pins the launch it awaits across checkpoints, so continuing the same spawn can't replace the result ([spawn launches](../architecture/state-system/spawn-launches.md#results-are-owed-per-launch)). A busy history index can yield a discovery checkpoint; retry rather than treating it as an empty set.

Spawned leads normally end their turn instead of waiting ([completion hold](../architecture/completion-drain.md#waiting-has-one-number)).
