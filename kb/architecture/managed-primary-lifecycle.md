# Managed primary lifecycle

A managed Codex or OpenCode primary has a Meridian launcher, a backend and the user-facing TUI. The TUI shares the user's foreground process group; the backend has its own [process scope](process-scope.md), and guardians stop both if the launcher dies. A primary stays open until the human ends it, so the [completion hold](completion-drain.md) doesn't apply. Claude primaries have no backend/TUI split. Terminal handling is in [primary terminal ownership](primary-terminal-ownership.md).

## Start, stop and cleanup share one gate

Stop waits for an in-flight start or its cleanup: otherwise stop could report no child while the start then publishes one, or a new start could overlap cleanup of the old. Cleanup keeps the gate even after bounded foreground failure handling stops waiting for it. A second runtime lock, or a lock that guards only replacement, would leave the first-publication race open.

## Repair and cleanup

A dead launcher's free lock makes its row read as crashed; reads neither signal nor write. The reaper claims the free lock and records the outcome and a cleanup claim before terminating recorded scopes ([derived status](state-system/derived-status.md#only-a-claim-authorizes-the-repair-write)). `spawn cancel` makes its own liveness and process-identity checks. A PID alone is never durable authority.
