# Work items

## WorkScope: Named vs Ambient

Named work survives sessions; ambient work belongs to an execution. Stable per-execution links let running agents follow a work item when it moves. The source checkout's `src/meridian/lib/state/.context/CONTEXT.md` describes the mechanism; [work and checkout links](../decisions/session-work-links.md) explains the choices.

Probe runtimes belong under `/tmp`, not in an autosynced work directory: large scratch runtimes can stall synchronization. See [probe runtime placement](../lessons/probe-runtime-placement.md).

Work activation means someone chose an item, not that a maintenance operation repointed an attachment. See [hook event contract](../decisions/hook-event-contract.md#d-work-activation--activation-means-someone-chose-the-item).

