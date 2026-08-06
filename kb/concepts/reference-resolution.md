# Reference Resolution

How `-f` (reference file) paths are resolved during spawn operations. Reference files are passed to agents as additional context. Resolution depends on the path prefix and the current task domain.

## Resolution Table

| Prefix | Resolves from | Domain |
|--------|--------------|--------|
| (relative, no prefix) | `reference_anchor` = `task_cwd` | Task |
| `/absolute/path` | Direct pass-through | — |
| `~/path` | HOME expansion | User |
| `kb:path` | KB directory (from `authority_root`) | Authority |
| `@path` | **ERROR** — use `kb:path` instead | — |

## The Two Domains

Reference resolution separates two concerns:

**Authority domain** (from `authority_root` = project config root):
- `kb:` paths always resolve here
- KB dir is `resolve_kb_dir(authority_root)`, independent of task_cwd
- KB is project infrastructure — it follows the repo, not the worktree

**Task domain** (from `task_cwd`):
- Relative paths resolve from `reference_anchor = task_cwd`
- `task_cwd` is where the agent works (may be a git worktree, may equal authority_root)

This separation is intentional and critical: a naive implementation that changes the base directory for relative paths must NOT also change the KB lookup base — these are separate resolution paths.

## Single Anchor

All relative `-f` paths share one anchor (`task_cwd`). There is no per-file fallback (try task_cwd, then project root, then cwd). Single-anchor resolution matches every major tool that separates workspace from config root: Docker build context, Terraform `-chdir`, Git `GIT_WORK_TREE`, AutoGen `work_dir`.

## `kb:` Semantics

```bash
meridian spawn -a coder -f kb:architecture/launch.md
```

- Resolves `architecture/launch.md` relative to the KB root directory
- KB root comes from `authority_root`'s KB configuration (`MERIDIAN_CONTEXT_KB_DIR`)
- Path must be relative (no leading `/`)
- Path must stay within the KB root — `kb:../outside.md` is rejected with a confinement error

## `@` Removal

The `@path` syntax for `-f` KB-relative references was removed in PR #248. Attempting to use it produces a helpful error:

```
The '@' prefix for KB paths is no longer supported.
  Use 'kb:architecture/page.md' instead.
```

**Note:** `@` in other contexts is unchanged — spawn/session references in `context_ref.py` and `query.py` use `@` with different semantics. Only `-f` file reference `@` was removed.

## Error Messages

When a reference path cannot be found, the error includes:
- The resolved path (what Meridian tried to open)
- The raw input (what was passed on the command line)
- The reference anchor (what directory relative paths resolved from)

This makes it clear whether the problem is a wrong path, a wrong anchor, or a missing file.

## task_cwd Resolution

`task_cwd` (= `reference_anchor`) follows this priority chain (highest wins):

1. Explicit `--task-dir` override
2. Explicit `--work <item>` task_dir (hard boundary — ambient work NOT consulted as fallback)
3. Inherited task-dir (`MERIDIAN_TASK_DIR`)
4. Ambient work item task_dir
5. Caller cwd when outside the project tree
6. Default → `authority_root`

**Stale task_dir** (directory no longer exists) → warning + fallback to the next valid tier (inherited `MERIDIAN_TASK_DIR`, then `authority_root`). Work state is not mutated on read. See [D-stale-task-dir-graceful-fallback](../decisions/spawn-cwd-worktree-anchor.md#d-stale-task-dir-graceful-fallback-stale-work-item-task_dir-degrades-with-warning-not-hard-error).

## Related

- [concepts/workspace-projection.md](workspace-projection.md) — harness filesystem permission grants (separate from reference resolution)
- [architecture/launch-system.md](../architecture/launch-system.md) — full authority/task domain split
- [decisions/spawn-cwd-worktree-anchor.md](../decisions/spawn-cwd-worktree-anchor.md) — design decisions with rejected alternatives
- [codebase/work-items.md](../codebase/work-items.md) — `work set-worktree` / `work clear-worktree` commands
