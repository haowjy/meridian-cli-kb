# OpenCode harness

Meridian delivers the prompt, permissions and workspace grants to OpenCode through one reserved config agent projected at bind. Contract: the source checkout's `src/meridian/lib/harness/.context/CONTEXT.md`; config behavior: [OpenCode config semantics](../research/opencode-config-semantics.md).

## One config agent

A single config agent works across OpenCode 1 and 2 and across the managed and fallback paths. The alternatives each failed one of those: a v2 plugin hook is v2-only and depends on plugin loading, and session instruction entries are experimental and unavailable to the bare-TUI fallback. The agent's file is passed by reference, avoiding environment-string size limits, and its reserved name prefix avoids collisions, because OpenCode merges same-named agents across config documents. The agent replaces OpenCode's built-in base prompt; OpenCode's other instruction sources still apply.

Meridian selects the agent explicitly on every open and resume, because `default_agent` doesn't override an agent a session has already committed to. OpenCode 2's bare TUI has no agent flag, so a fallback that resumes a session committed to another agent can still miss the prompt (tracked in the harness `.context/FUTURE`). A Meridian-created OpenCode 2 session can name an agent that doesn't exist outside Meridian: `opencode run --session` then fails, and the TUI switches to `build`.

The agent carries no model, because OpenCode 2 ignores that field in favor of session and top-level configuration; the model has its own channel ([continuation model selection](../concepts/continuation-model-selection.md#opencode)). Permission rules stay on the agent.

## Managed primaries

The managed primary's model selection can't see an unsent choice made in the TUI, and its read-then-write over HTTP isn't atomic against a concurrent TUI submission. On OpenCode 1, a `spawn message` follows the last committed agent, not a TUI agent switch that hasn't produced a turn yet. The observer also carries messages ([primary messaging](primary-messaging.md#one-socket-for-every-primary)).

OpenCode lowers `ask` to native `deny` in spawns ([tool permissions](tool-permissions.md#headless-ask)).
