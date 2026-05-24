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

## Worktree Assignment

Work items can have an associated worktree path — a directory where spawns scoped to that work item will run. This determines the `task_cwd` for agent processes.

### Two Assignment Paths

**Managed** (`WorktreeMetadata.managed = true`): Meridian provisioned the git worktree and owns its lifecycle. Created by `work start --worktree`, `spawn --worktree`, or `work worktree --ensure`.
- Canonical path is always `<repo>.worktrees/<name>`, enforced by `managed_worktree_path()`. No `--path` override for managed worktrees.
- Removed on `work done` / `work delete` when not shared with other active work items
- Renamed if the work item is renamed
- Once metadata is persisted (`repo_path` + `name`), subsequent ensures reuse it — `--repo` cannot redirect an established managed worktree

**Manual** (`WorktreeMetadata.managed = false`): Assigned by `work set-worktree`. Accepts arbitrary paths. Meridian treats it as read-only:
- Never deletes the directory
- `work clear-worktree` removes only the assignment, not the directory

Multiple work items may point to the same `worktree_path`. No uniqueness constraint. Managed cleanup skips removal when the path is shared.

### Commands

```bash
# Managed worktree lifecycle
meridian work worktree                            # show active work item's worktree status
meridian work worktree --ensure                   # ensure managed worktree exists (or create temp if no active work)
meridian work worktree <work-id>                  # show explicit work item's worktree
meridian work worktree <work-id> --ensure         # ensure explicit work item's managed worktree
meridian work worktree --ensure --repo <path>     # select target repo (first ensure only)

# Manual assignment
meridian work set-worktree <work-item> <path>     # assign existing directory (manual, managed=false)
meridian work clear-worktree <work-item>          # remove assignment (not directory)
```

`work start --worktree` is the "provision and assign" path for a new work item (creates a managed git worktree). `work worktree --ensure` is the idempotent ensure path for an existing item. `set-worktree` is the "assign only" path for pre-existing directories.

### Temporary Managed Worktrees

When `work worktree --ensure` is invoked with no active or explicit work item, Meridian creates a **session-scoped temporary managed worktree**:
- Named deterministically from `spawn_id` (in a spawn, e.g. `temp-spawn-p2427`) or `chat_id` (in a primary session, e.g. `temp-chat-c2343`)
- Canonical path: `<repo>.worktrees/<worktree-name>`
- Tracked in `~/.meridian/projects/<uuid>/worktree-temp/<key>.json` — separate from work item state
- Crash-safe: record written with `status=pending` before `git worktree add`; set to `status=ready` on success; cleared (rollback) on failure

The explicit-work-id form (`work worktree <id> --ensure`) does NOT create a temp worktree if the named item is missing — it fails clearly.

### Effect on Spawns

`spawn --worktree` provisions (or recovers) the canonical managed worktree before launching, then runs the agent in it. This is an ensure operation — it creates the worktree if it does not exist. Implicit `--work <id>` without `--worktree` still falls back to `authority_root` when no worktree is configured.

When a work item has a `worktree_path`, spawns attached to that item run in it:
- Agent process cwd = `worktree_path` (for PI/OpenCode/Codex)
- Relative `-f` reference paths resolve from `worktree_path`
- `kb:` reference paths still resolve from `authority_root` (the config root)

If the `worktree_path` no longer exists on disk, spawn fails with a hard error rather than silently falling back to the authority root. Use `--no-worktree` to bypass this.

See [../architecture/launch-system.md](../architecture/launch-system.md) — Authority/Task Domain Split for the full model. See [../decisions/spawn-cwd-worktree-anchor.md](../decisions/spawn-cwd-worktree-anchor.md) for design rationale.

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
