# Session Initiation Modes

`--continue` resumes the same transcript, while `--fork` branches it with identity locked and `--fork-fresh` allows identity changes. `--from` starts a fresh session and supplies prior output as reference material. See `docs/commands.md` and [launch decisions](../decisions/launch.md).

Continuation replays the recorded work, task and launch context rather than silently recomputing it from the caller. Primary and child continuation have different override rules; the user-facing flag contract is in `docs/commands.md` and the implementation guidance in `src/meridian/lib/launch/.context/CONTEXT.md`.

Prior reports and file references belong in the user turn: treating them as system instructions would grant untrusted content authority and changing them would disturb the shared prompt-cache prefix. A spawn's goal is also per-launch and belongs in that turn, while a primary's goal remains system guidance. See [launch decisions](../decisions/launch.md#the-system-prompt-depends-only-on-what-defines-the-conversation-2026-10).

The browse picker rechecks a session's binding and liveness at activation. It forks a live session instead of attaching twice, and blocks when no recorded native binding can authorize re-entry. The lifetime lock remains the guard against concurrent attachment. See [session browse decisions](../decisions/session-browse.md) and [native session identity](../decisions/native-session-identity.md).
