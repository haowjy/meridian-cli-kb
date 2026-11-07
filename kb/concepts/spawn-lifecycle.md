# Spawn lifecycle

A spawn represents an execution and retains its identity across continuations; each continuation is a separate launch. A terminal status describes execution completion, not the correctness of the agent's work. See [spawn launches](../architecture/state-system/spawn-launches.md) and the source checkout's [src/meridian/lib/ops/spawn/AGENTS.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/AGENTS.md).

Readers may derive an outcome for a dead launch without writing it. A runner or repairer holding the launch lock persists the outcome; a durable report takes precedence over cancellation. This keeps display reads free of side effects and prevents competing terminal writers. The source checkout's [src/meridian/lib/state/.context/CONTEXT.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/.context/CONTEXT.md) owns the recovery contract.

A completion candidate can wait for descendants; a close can instead cancel remaining work. See [completion drain](../decisions/completion-drain.md) for the closure boundary and [spawn finalization](../architecture/spawn-finalization.md) for outcome arbitration.
