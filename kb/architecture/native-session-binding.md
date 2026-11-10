# Native session binding

## Binding

A chat binds a native key: harness, recorded store and native session ID. A conflicting nonempty field never overwrites the accepted binding; explicit relocation is the only path that changes its store. This prevents a concurrent or guessed session from silently taking over a conversation. See [native session identity](../decisions/native-session-identity.md) and the source checkout's `src/meridian/lib/harness/AGENTS.md`.

## Runner pipeline

The launch plans identity before execution and verifies exact sources for resume and fork. Owned observations can establish an unassigned identity; a contradictory first signal fails the run. Later switches are observations, not permission to rewrite the entry chat. The child must exit and teardown must join before conclusion, because Pi writes its exit evidence during shutdown. See [Pi sessions](pi-native-sessions.md#exit-observation) and [Claude sessions](claude-native-sessions.md#launch-owned-sessionstart-journal-and-current-chat).

### Conversation-owned settings

Final binding persists policy on the captured entry generation before native
execution. The primary's final snapshot is not available when its session scope
first opens: recording preview policy there, or waiting until post-run
`record_started`, would give live observations the wrong settings. Observed aliases
inherit the selected conversation's policy and work. Older generations without a
snapshot read only their recorded spawn link; there is no newest-owner fallback.

A `pN` continuation or fork resolves the selected chat's current generation, just
like `cN`, including settings, work and source history. It does not combine that
chat's native identity with the original run's policy. Exact entry-generation
lookup remains a separate requirement for capture and provenance.

**Why teardown comes first.** Exit evidence (Pi's boundary record) is written by the child as it shuts down.
Real Pi showed the failure mode: the streaming runner read the boundary record about
4 s before Pi wrote `quit`. The fix orders the read after `SpawnManager.join_teardown`;
there is no polling or timeout
([lesson](../lessons/native-session-identity.md#terminal-status-is-not-process-exit)).

### One-attempt causality

A failure finalizes the same harness turn; a replay could repeat model work or side effects. See [launch decisions](../decisions/launch.md#harness-turn-failures-finalize-without-replay).
