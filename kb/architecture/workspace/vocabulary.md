# Workspace Vocabulary

Terms covering workspace configuration, multi-repo projection, and snapshot merging. For the full vocabulary index see [../../vocabulary.md](../../vocabulary.md).

| Term | Definition | See also |
|---|---|---|
| **Workspace entry** | A named workspace root configuration entry declared as `[workspace.<name>]` in `meridian.toml` or `meridian.local.toml`. Injects an additional repository root into harness subprocesses. | [../../concepts/workspace-projection.md](../../concepts/workspace-projection.md) |
| **Workspace projection** | The mechanism that injects additional repository roots into a harness subprocess so it can access sibling repos. Per-harness behavior: Claude and Codex receive `--add-dir` flags; OpenCode receives `OPENCODE_CONFIG_CONTENT`. Invalid workspace config blocks spawns before harness contact. | [../../concepts/workspace-projection.md](../../concepts/workspace-projection.md) |
| **Workspace snapshot** | The merged workspace configuration (`WorkspaceSnapshot`) from `meridian.toml` and `meridian.local.toml`, with local entries overriding committed entries. `get_projectable_roots()` returns only enabled, existing paths. | [overview.md](overview.md) |
