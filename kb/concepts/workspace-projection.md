# Workspace Projection

**Workspace projection** extends the filesystem scope available to an agent beyond the current project root. When an agent needs to read or write files in a sibling repository, workspace projection grants that access at the harness sandbox level.

Workspace roots are **not** mentioned in the agent's system prompt and do not receive `MERIDIAN_CONTEXT_*_DIR` env vars. They are pure permission grants — the agent gets filesystem access, but Meridian provides no guidance on where or how to use it. This is the sharpest contrast with [context resolution](context-resolution.md), which surfaces paths explicitly to agents.

## When to Use It

Workspace projection is for multi-repo development where an agent needs file access across repo boundaries — for example, a coder agent that needs to edit both the backend and a frontend in a sibling directory. Without projection, harness sandboxes typically restrict the agent to the project root.

## Config Location

Workspace entries live in `[workspace.<name>]` tables in `meridian.toml` and/or `meridian.local.toml`. The two files serve different purposes:

```toml
# meridian.toml (committed) — declares the conventional layout for this project.
# Assumes standard checkout structure; developers with non-standard layouts override in local.
[workspace.frontend]

[workspace.prompts]
path = "../prompts/meridian-base"
```

```toml
# meridian.local.toml (gitignored) — machine-specific path overrides and additions.
[workspace.frontend]

[workspace.local-data]
path = "/data/large-dataset"             # local-only root not in committed convention
```

Entry names must match `^[a-z][a-z0-9_-]*$`. The `path` key is required; unknown keys are warned and ignored.

## Merge Semantics

`resolve_workspace_snapshot()` in `src/meridian/lib/config/workspace.py` merges committed and local entries:

- Committed entries from `meridian.toml` establish the baseline.
- Local entries from `meridian.local.toml` override committed entries by name and add new ones.
- Merge order for display: committed entries first, then local-only entries appended.

Each resolved entry carries a `source` field: `committed`, `local`, or `merged` (present in both).

### Missing path behavior

| Condition | Finding |
|---|---|
| Committed-only path does not exist on disk | Silently skipped (normal partial-checkout condition) |
| Local or merged path does not exist on disk | `workspace_local_missing_root` diagnostic emitted |

The asymmetry reflects intent: a committed path that's absent means the developer didn't clone that repo. A local path that's absent is likely a misconfiguration — the developer explicitly declared it but the directory isn't there.

## Snapshot Status

`WorkspaceSnapshot.status` has three values:

| Status | Meaning |
|---|---|
| `"none"` | No `[workspace]` section found in either file |
| `"present"` | Parsed successfully; roots may or may not exist on disk |
| `"invalid"` | TOML parse error or schema violation (`path` missing, name invalid) — **blocks launch** |

`"invalid"` is terminal: meridian refuses to launch rather than silently drop entries, since a broken workspace config may leave agents unable to complete their work.

## Projection at Launch

`get_projectable_roots()` returns only roots that are both enabled and exist on disk. These are passed to `project_workspace_roots()` in the launch layer, which translates them into harness-specific access grants:

| Harness | Mechanism |
|---|---|
| Claude | `--add-dir <root>` per root |
| Codex | `--add-dir <root>` per root |
| OpenCode | `OPENCODE_CONFIG_CONTENT` env var with external directory list |

## Key Types

From `src/meridian/lib/config/workspace.py`:

- **`WorkspaceEntryConfig`** — one parsed `[workspace.<name>]` entry (path + extra keys)
- **`ResolvedWorkspaceRoot`** — evaluated entry with `name`, `declared_path`, `resolved_path`, `enabled`, `exists`, `source`
- **`WorkspaceSnapshot`** — shared read model: `status`, `source_paths`, `roots`, `findings`
- **`WorkspaceFinding`** — structured diagnostic (code + message + optional payload)

## Legacy Format

The old `workspace.local.toml` with `[[context-roots]]` arrays is read as a fallback when no new `[workspace]` entries are present. A `workspace_legacy_file_present` finding is emitted when both are present.

Migrate with:

```bash
meridian workspace migrate
```

This copies enabled legacy roots into `meridian.local.toml` using the new named-entry format. Disabled legacy roots are dropped (the new schema has no `enabled` field — exclusion is by omission). Entry names are generated from the path basename with numeric suffixes for collisions.

## Codex-Specific Behavior

### Remote TUI Attach Root Mirroring

When a user attaches the Codex TUI via `codex resume --remote ws://...`, the TUI runs in `ThreadParamsMode::Remote`. In this mode, the TUI sends per-turn `sandbox_policy` constructed from its own local `PermissionProfile` — built from the TUI's CLI flags (including `--add-dir`) and local config. If Meridian doesn't pass `--add-dir` flags on the attach command, the TUI's per-turn overrides **narrow** the sandbox scope below what the app-server was originally configured with.

This is Codex working as designed — the remote-TUI security model assumes the connecting client may have a different posture than the original launcher. Meridian passes the same roots to both sides by appending one `--add-dir` flag per `projected_roots` entry to the `codex resume` attach command.

### Sandbox Scope vs Approval Policy Are Independent

Codex treats sandbox write scope (`sandbox_workspace_write.writable_roots`) and approval policy (`approval_policy` / `approvals_reviewer`) as separate concerns. Writing to a fully-writable sandbox path can still trigger an `item/fileChange/requestApproval` prompt if the active approval policy requires it.

Practical implication: observing an approval prompt for a write to a projected workspace root is **not proof** that workspace roots are missing from projection. An `item/permissions/requestApproval` is stronger evidence of missing scope; `item/fileChange/requestApproval` alone is not. See [decisions/workspace.md — D49](../decisions/workspace.md#d49).

## OpenCode-Specific Behavior

### Parent Env Merge

OpenCode workspace projection uses `OPENCODE_CONFIG_CONTENT` — a JSON env var with `permission.external_directory` entries. When the parent environment already has `OPENCODE_CONFIG_CONTENT`, Meridian parses it, preserves existing keys, and deep-merges new workspace entries into `permission.external_directory`.

This avoids the old nested-spawn failure mode where inherited env config caused child spawns to lose workspace root access. The merge is additive only; existing entries remain. The `/*` wildcard suffix on each root path is required by OpenCode's permission schema.

## `projected_roots` as First-Class Field

Workspace roots flow through `ResolvedLaunchSpec.projected_roots`, not through synthetic `--add-dir` entries in `extra_args`. This keeps Meridian-owned policy data separate from user-owned passthrough args.

Each harness projection consumes `projected_roots` in its native mechanism: Claude and Codex subprocess commands become `--add-dir` flags, Codex app-server commands become `sandbox_workspace_write.writable_roots` config, Codex remote TUI attach commands become `--add-dir` flags, and OpenCode receives merged `OPENCODE_CONFIG_CONTENT`. See [decisions/workspace.md — D47](../decisions/workspace.md#d47) for full rationale.

## Related

- [concepts/context-resolution.md](context-resolution.md) — context paths surfaced explicitly to agents via env vars
- [concepts/config-precedence.md](config-precedence.md) — how meridian.toml and meridian.local.toml load relative to each other
- [architecture/workspace/overview.md](../architecture/workspace/overview.md) — workspace config schema and resolution mechanics in full detail
- [decisions/workspace.md](../decisions/workspace.md) — D47 (projected_roots), D48 (OpenCode merge), D49 (scope vs policy independence)
- [codebase/harness-adapters.md](../codebase/harness-adapters.md) — Codex managed-primary approval routing and remote TUI attach root mirroring
