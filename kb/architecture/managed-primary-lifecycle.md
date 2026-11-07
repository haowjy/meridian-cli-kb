# Managed primary lifecycle

Codex and OpenCode managed primaries have a Meridian launcher, a backend and a user-facing TUI. The TUI shares the user's foreground process group; the backend has its own process scope. Guardians stop those processes if the launcher dies. A primary stays open until the human ends it, so the [completion hold](drain-plans.md#resident-completion) does not apply. Claude primaries have no managed backend/TUI split. See [process scope](process-scope.md) and [primary terminal ownership](primary-terminal-ownership.md).

Startup and stop share an ownership gate. Stop waits for an in-flight start or its cleanup, preventing it from reporting no child while startup subsequently publishes one. Cleanup retains the gate even if bounded foreground failure handling stops waiting; a second runtime or replacement-only lock would leave the initial-publication race open.

## Repair and cleanup boundary

A dead launcher's free lock makes its row read as crashed; reads do not signal or write. The reaper claims the free lock and records the outcome and cleanup claim before terminating recorded scopes. `spawn cancel` has its own liveness and process-identity checks. A PID alone is not durable authority: it can be reused. See [derived status and cleanup claims](state-system/derived-status.md#the-reapers-cleanup-claims).
