# Codebase: Work Items

Work items are lightweight labels that group spawns and scratch directories. They're the mechanism behind `meridian work start/done/reopen` and the `MERIDIAN_ACTIVE_WORK_DIR` and `MERIDIAN_CONTEXT_WORK_DIR` env vars surfaced to agents.

## WorkScope: Named vs Ambient

Work items have two scope kinds, modeled explicitly by the `WorkScope` dataclass (`lib/state/work_scope.py`):

| Kind | `identifier` | `root` | Durability |
|------|-------------|--------|------------|
| `work_item` | work slug (e.g., `fix-auth`) | repo scratch dir | Durable — survives across sessions |
| `ambient_spawn` | spawn ID (e.g., `p1234`) | child spawn ambient dir | Ephemeral — lives with the spawn |

### Resolution at Bind Time

`resolve_bound_work_scope()` — two paths:
- **Named work item** (`requested_work_id` is set) → `resolve_work_scratch_dir_for_project()` → normal repo scratch path
- **Ambient (no work ID)** → `resolve_ambient_work_dir()` → `spawns/p<child>/work/` under the child spawn directory

### Resolution from Context Parts

`resolve_work_scope_from_parts()` resolves after context parts are already derived. Used by dashboards, `work current`, and env-var-based resolution. Respects work_id provenance: a `work_id` from an in-session switch is trusted; a `bound_work_dir` from `MERIDIAN_ACTIVE_WORK_DIR` is treated skeptically when there's no matching `work_id` provenance.

### Leave-Scope Warning

When an agent `cd`s out of the work scope root, a warning is emitted. The warning wording is in the ops layer (not the state layer), keeping `WorkScope` a pure value object. The warning uses the scope kind to phrase appropriately: "left work item directory" vs "left spawn working directory."

## Which Item a Session Is On

An execution's work item is the `active_work_id` on its own session row; an
explicit `--work` outranks it, and the launch env (`MERIDIAN_ACTIVE_WORK_ID`)
speaks only for a shell Meridian did not launch. `work switch`, `start` and
`clear` write the caller's own rows; child spawns inherit their parent's item at
launch. A row naming an item that no longer exists resolves to no item, with a
warning. Commands that act on an item default to the current one, and `current`
is accepted wherever a work ID is.

Spawns also record the `work_id` they launched with in their `state.json`. `work
rename` moves the item, then moves the live executions attached to it onto the new
name, and renames the `work_id` in every spawn's `state.json` that names it; the
session row of a chat that already ended keeps the old name.

Every execution reaches the item through its per-chat links,
`sessions/<chat>/work` and `sessions/<chat>/task`, which point at the work
store's own `work-items/<id>/{work,task}` links. That is why `work done` moves
the directory at once and running agents follow it. Why it is built this way:
[per-execution work and checkout links](../decisions/session-work-links.md).
Mechanism: meridian-cli `src/meridian/lib/state/.context/CONTEXT.md` ("Work
Item Directory Moves"). This describes meridian-cli PR #786, open and unmerged;
released builds name concrete paths in the env, and resolve a spawn's item from
the env before its row.

## Scratch Directories

Each work item has a **scratch directory** for agents to write design docs, plans, and artifacts:

```
~/.meridian/git/<repo>/work/<slug>/    (or wherever context.work_dir resolves)
```

Configured via `[context.work]` in `meridian.toml`, whose root is surfaced as `MERIDIAN_CONTEXT_WORK_DIR`; the current item's directory is `MERIDIAN_ACTIVE_WORK_DIR`. On `work done`, the scratch dir moves to the archive with its metadata. On `work reopen`, it moves back. A move refuses when both locations exist and never deletes either copy.

When the work dir lives in a git-synced context repo (as it does here), everything written to it is committed by autosync. It holds notes, scripts and result summaries only. Probe runtimes (scratch homes, copied state, environments, databases) go under `/tmp`. A 24 GB pile of them once stalled autosync for hours; see [probe runtime placement](../lessons/probe-runtime-placement.md).

See [../concepts/context-resolution.md](../concepts/context-resolution.md) for how `MERIDIAN_CONTEXT_*_DIR` env vars are populated.

## Checkout (`task_dir`)

A work item can record a source checkout, usually a git worktree, with
`work start --task-dir` or `work task-dir`. Meridian only records the path; it
does not create, rename or remove worktrees. An execution's checkout is its own
pin (from `--task-dir`, a launch outside the project, or its launcher's pin),
else the item's `task_dir`, else the project root
([D-checkout-pin-sources](../decisions/session-work-links.md#d-checkout-pin-sources)).

Relative `-f` reference paths resolve from the checkout; `kb:` paths still
resolve from `authority_root`. A `task_dir` that no longer exists (common after a
worktree is removed post-merge) is skipped with a warning, and work state is not
mutated on read. See [../architecture/launch-system.md](../architecture/launch-system.md)
for the authority/task split and
[../decisions/spawn-cwd-worktree-anchor.md](../decisions/spawn-cwd-worktree-anchor.md)
for the rationale.

## Hook and Event Coordination

Work items fire three hook events:

- `work.created`: `work start` made a new item.
- `work.activated`: someone chose an item. `work start` (new or existing), `work switch` (archived items included), and a primary launch attaching `--work` or `--from` work fire it, with or without a session. Child spawns inheriting work, `work rename` repointing attachments, and `work clear` do not: they write the session attachment through `session_store` directly.
- `work.done`: the item was archived as done.

`activate_work()` in `lib/ops/work_attachment.py` is the only path that fires `work.activated`. A "session" here is a Meridian chat record (`cN`), which a primary has as soon as it launches, before its harness reports a session ID. When `session_store.session_exists()` finds the chat record, `activate_work()` also writes the item as that session's active work; run from a plain shell with no chat, it fires the event and attaches nothing. `publish_work_event()` beside it builds every work event's hook context, resolves `work_dir` for active or archived items, and emits the paired telemetry. A hook failure is logged and the work operation continues. The default `git-autosync` hook syncs on `work.activated` and `work.done`, so a primary launch with work waits for one sync before its harness starts. Why activation excludes maintenance writes: [D-work-activation](../decisions/hook-event-contract.md#d-work-activation--activation-means-someone-chose-the-item).

A nested-agent warning is included in `WorkLifecycleOutput` when called from a nested agent (`RuntimeContext.is_nested`) — work coordination should go through the orchestrator.

See [../concepts/hooks-and-plugins.md](../concepts/hooks-and-plugins.md) for the hook dispatch model.

## Related Pages

- [../architecture/state-system/roots-and-work-items.md](../architecture/state-system/roots-and-work-items.md) — state roots and work-item storage
- [../concepts/context-resolution.md](../concepts/context-resolution.md) — MERIDIAN_CONTEXT_WORK_DIR surfacing
- [../architecture/workspace/overview.md](../architecture/workspace/overview.md) — workspace permission grants (different from work items)
- [../operations/configuration-guide.md](../operations/configuration-guide.md) — configuring work root
