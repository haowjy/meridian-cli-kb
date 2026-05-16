# Codebase: Work Items

Work items are lightweight labels that group spawns and scratch directories. They're the mechanism behind `meridian work start/done/reopen` and the `MERIDIAN_CONTEXT_WORK_*_DIR` env vars surfaced to agents.

## Cross-Module Work Attachment

The "current" work item for a session is tracked in two places:

1. **Persisted** (`current_work.py`): `~/.meridian/projects/<uuid>/current-work` — survives process restarts.
2. **In-session override**: `RuntimeContext.work_id` — set by `work switch` or `work start` during a session; takes precedence over the persisted value.

Nested agents inherit the work ID from their parent's `MERIDIAN_WORK_ID` env var. `work clear` clears both.

Spawns record their `work_id` in `state.json`. When a work item is renamed, `work_store.rename_work_item()` rewrites all spawn `state.json` files that reference the old ID. The rename is atomic at the directory level; spawn rewrites are best-effort.

## Scratch Directories

Each work item has a **scratch directory** for agents to write design docs, plans, and artifacts:

```
~/.meridian/git/<repo>/work/<slug>/    (or wherever context.work_dir resolves)
```

Configured via `[context.work]` in `meridian.toml`. Surfaced to agents as `MERIDIAN_CONTEXT_WORK_DIR`. On `work done`, the scratch dir is archived alongside the metadata. On `work reopen`, it's restored.

See [../concepts/context-resolution.md](../concepts/context-resolution.md) for how `MERIDIAN_CONTEXT_*_DIR` env vars are populated.

## Hook and Event Coordination

`work start` fires a `work.started` lifecycle hook. `work done` fires `work.done`. These are dispatched through `lib/core/lifecycle.py`'s `get_hook_dispatcher()`. Hooks can trigger downstream automation (e.g., git-autosync after a work item closes).

`work done` emits warnings (not errors) when there are active sessions or spawns associated with the work item.

A nested-agent warning is included in `WorkLifecycleOutput` when called from a nested agent (`RuntimeContext.is_nested`) — work coordination should go through the orchestrator.

See [../concepts/hooks-and-plugins.md](../concepts/hooks-and-plugins.md) for the hook dispatch model.

## Related Pages

- [../architecture/state-system.md](../architecture/state-system.md) — full state layout including work dirs
- [../concepts/context-resolution.md](../concepts/context-resolution.md) — MERIDIAN_CONTEXT_WORK_DIR surfacing
- [../architecture/workspace/overview.md](../architecture/workspace/overview.md) — workspace permission grants (different from work items)
- [../operations/configuration-guide.md](../operations/configuration-guide.md) — configuring work root
