# Launch Waiting, Goals, and Session Initiation

See [session initiation](../concepts/session-initiation.md) for the four modes and [spawn wait barrier](../concepts/spawn-wait-barrier.md) for waiting behavior.

Headless deny policy is checked during preparation, before a background spawn creates a row and worker. Checking only in the worker delays an actionable refusal and leaves a transient row for callers to wait on. Bind also checks persisted worker requests that may not have passed through preparation.

A spawn's goal is structured spawn-level state, not work-level state or only prompt text: different spawns in the same work item can have different goals, and consumers should not have to parse a prompt to discover one. It does not select model, harness or permissions. Goal text is trimmed without template substitution so persisted and delivered text agree.

### `completion_contract` placed after `context_prompt` in `SYSTEM_INSTRUCTION_BLOCK_ORDER`

A spawn's per-launch goal contract now goes in the first user turn; a primary's goal is system guidance. See [stable system prompt](launch.md#the-system-prompt-depends-only-on-what-defines-the-conversation-2026-10).

`--fork` preserves identity-shaping inputs for cache locality; `--fork-fresh` permits identity changes. Prior context from `--from` goes in the user turn: prior reports can contain untrusted content, and placing them in the system prompt would give them authority and disturb the shared cache prefix.

Session browse decides whether to resume, fork or block at the operations boundary, and rechecks when the user activates a row. A live session forks rather than double-attaching. The listed action is advisory because liveness may change between display and activation.
