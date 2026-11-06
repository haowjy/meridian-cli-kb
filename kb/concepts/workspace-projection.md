# Workspace Projection

Workspace entries grant filesystem access; [context paths](context-resolution.md) tell agents where working memory lives. The two can name the same directory but serve different purposes. A missing committed workspace root can mean an optional checkout is absent, while a missing locally declared root suggests a configuration mistake.

Codex's remote TUI can narrow the backend's workspace scope unless the attach client receives the same roots. An approval prompt for a file change alone does not prove a root is missing: sandbox scope and approval policy are independent. See [workspace decisions](../decisions/workspace.md) and `docs/codex-tui-passthrough.md`.

Claude can discover skills in projected roots; granting access can therefore expose instructions from those roots. See [workspace decisions](../decisions/workspace.md) and `src/meridian/lib/harness/AGENTS.md`.
