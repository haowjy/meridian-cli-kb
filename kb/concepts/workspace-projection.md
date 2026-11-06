# Workspace Projection

Workspace entries grant filesystem access; [context paths](context-resolution.md) tell agents where working memory lives. The two can name the same directory but serve different purposes. Missing committed and locally declared roots produce distinct diagnostics: an absent checkout can be normal while a missing local path may be a configuration mistake.

Codex's remote TUI and managed backend have distinct permission surfaces. An approval prompt for a file change alone does not prove a root is missing: sandbox scope and approval policy are independent. See [workspace decisions](../decisions/workspace.md) and `docs/codex-tui-passthrough.md`.

Claude can discover skills in projected roots; granting access can therefore expose instructions from those roots. See `src/meridian/lib/harness/AGENTS.md`.
