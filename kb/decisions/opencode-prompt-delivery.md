# OpenCode Prompt Delivery

Meridian projects a reserved config agent at bind for the prompt, permissions and workspace grants. This single channel works across OpenCode 1 and 2 and the managed and fallback paths. A file reference avoids environment-string size limits. OpenCode merges same-named agents across config documents, so the reserved prefix reduces accidental collisions. See meridian-cli `src/meridian/lib/harness/.context/CONTEXT.md` and [OpenCode config semantics](../research/opencode-config-semantics.md).

## Config agent {#d-opencode-config-agent-deliver-the-prompt-as-a-config-agent-projected-at-bind}

A v2 plugin hook is v2-only and depends on plugin loading; session instruction entries are experimental and unavailable to the bare-TUI fallback. The config agent replaces OpenCode's built-in base prompt, although its other instruction sources remain. A Meridian-created OpenCode 2 session can name an agent unavailable outside Meridian: `opencode run --session` then fails, while the TUI switches to `build`.

## Explicit agent selection {#d-opencode-explicit-agent-select-the-agent-on-every-session-not-only-through-default_agent}

Select the reserved agent on open and resume: `default_agent` does not override a session's committed agent. OpenCode 2's bare TUI has no agent-selection flag, so a fallback resuming a different agent can still miss the prompt. This gap is tracked in the harness `.context/FUTURE`.

## Model channel {#d-opencode-model-off-agent-the-model-stays-at-top-level-config-and-on-the-session}

The agent carries no model because OpenCode 2 ignores that field in favor of session and top-level configuration. Model selection has its own channel; permission rules still belong to the agent.
