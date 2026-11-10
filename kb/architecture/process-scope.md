# Process-scope ownership

Harnesses can leave descendants after their root exits, including processes reparented to PID 1. Durable scope metadata lets recovery identify owned work, while session-owned scopes can survive a disposable launcher. A process-tree snapshot alone can miss descendants when an intermediate exits. See [process ownership decisions](../decisions/launch-process-ownership.md) and the source checkout's `src/meridian/lib/platform/process_scope/.context/CONTEXT.md`.

## Parent-death linkage

The start gate prevents a harness from executing before its scope is recorded. A guardian receives runner-death evidence through a pipe and stops the owned group; a TUI sharing the shell's group receives root-only termination to avoid signalling the shell. Descendants that create another session or group can escape group cleanup. See [spawn identity](process-scope.md).

## Continue sweep

Continue claims the prior launch lock and sweeps its remaining scopes before starting another launch. It refuses a live owner and rechecks process identity before signalling. See [spawn launches](state-system/spawn-launches.md#the-launch-lock-fences-each-launch).

PID and process-group reuse make cleanup hazardous. Some fallback paths can still signal when root birth is unknown or inaccessible; a reachability probe can also see zombies or a reused group. Reachability is diagnostic, not proof that a scope is safe to kill. See [process-scope follow-ups](../open-questions/process-scope.md) and [managed primary lifecycle](managed-primary-lifecycle.md).
