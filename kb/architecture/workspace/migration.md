# Workspace Migration

`workspace.local.toml` was the original format for declaring workspace roots. It used an anonymous `[[context-roots]]` array and was always gitignored. The new design uses named `[workspace.NAME]` sections in `meridian.toml` and `meridian.local.toml`.

This page covers the legacy fallback, the migration command, and the migration phase plan.

See [decisions/workspace.md#d41](../../decisions/workspace.md#d41) for why the format changed.

---

## Legacy Format

`workspace.local.toml` used an anonymous array-of-tables:

```toml
# workspace.local.toml (legacy — gitignored, no committed equivalent)
[[context-roots]]

[[context-roots]]
path = "../prompts/meridian-base"
enabled = false   # could disable individual roots
```

Each entry had:
- `path` (required, string)
- `enabled` (optional boolean, default `true`)

Unknown keys were captured as findings. The file had no committed counterpart — the project topology could never be shared.

---

## Legacy Fallback Behavior

`resolve_workspace_snapshot()` uses `workspace.local.toml` only as a fallback:

```
if [workspace] entries exist in meridian.toml OR meridian.local.toml:
    → use new-style loader; workspace.local.toml is ignored
    → if workspace.local.toml also exists: emit workspace_legacy_file_present finding
else if workspace.local.toml exists:
    → parse with legacy reader; emit workspace_deprecated_legacy finding
else:
    → WorkspaceSnapshot(status="none")
```

The presence of any `[workspace]` section in either new file completely bypasses the legacy reader. The new system and the legacy system do not merge.

Legacy roots receive `source = "legacy"` in the snapshot. Their `enabled` field is preserved (can be `False`). They resolve paths relative to `workspace.local.toml`'s parent directory rather than `project_root`.

---

## Migration Command

```bash
meridian workspace migrate
```

Reads `workspace.local.toml` and appends the enabled entries to `meridian.local.toml` as named `[workspace.NAME]` sections.

### What it does

1. Parses `workspace.local.toml` with `parse_workspace_config()`.
2. Discards disabled roots (the new schema has no `enabled` field — see [decisions/workspace.md#d43](../../decisions/workspace.md#d43)).
3. Generates an entry name from each enabled root's path basename: lowercased, non-alphanumeric characters replaced with hyphens, leading/trailing hyphens stripped.
4. Deduplicates names by appending `-2`, `-3`, etc. for collisions.
5. If the basename produces an invalid or empty name, falls back to `root-N`.
6. Writes a migration block to `meridian.local.toml`, prefixed with an auto-migration comment.
7. Prints warnings for disabled roots dropped and a provisional-name advisory.

Example output:

```bash
$ meridian workspace migrate
migrated: 2 entries -> /project/meridian.local.toml
- workspace.meridian-base: ../prompts/meridian-base
warning: Review and rename auto-generated workspace entry names before adopting
  shared [workspace] conventions in meridian.toml; basename-derived names are
  provisional and may not match the canonical names chosen by the project.
```

### Why names are provisional


Names should be reviewed and aligned with whatever the project chooses to commit in `meridian.toml`.

### `--force` flag

By default, the command refuses if `meridian.local.toml` already has a `[workspace]` section. `--force` strips existing `[workspace.*]` sections from the file before writing the migration block.

### Disabled roots

Legacy disabled roots are dropped with a warning. The new schema represents disablement through the absence of a local override entry, not through an `enabled = false` flag. There is no direct migration path for a disabled root.

---

## `workspace init`

```bash
meridian workspace init
```

Creates `meridian.local.toml` with a scaffold comment if the file doesn't already exist (or has no workspace section or commented scaffold):

```toml
# Workspace topology — local path overrides and additions.
# Override committed [workspace] paths for your local checkout.
#
# [workspace.example]
# path = "../sibling-repo"
```

Also registers `meridian.local.toml` in `.git/info/exclude` so it's gitignored locally without touching `.gitignore`.

---

## Migration Phases

| Phase | State |
|---|---|
| **1 — Additive** | `[workspace]` loader added; `workspace.local.toml` still supported with `workspace_deprecated_legacy` finding |
| **2 — Tooling** | `meridian workspace migrate` and `workspace init` commands available |
| **3 — Removal** | Legacy reader removed; `workspace.local.toml` no longer read |

During Phase 1 and 2, both systems work. Running `meridian workspace migrate` moves a developer from legacy to new-style. Once the project commits entries in `meridian.toml`, the legacy file is ignored (the new loader takes precedence), even if not yet deleted.

---

## Key Source Files

| File | Role |
|---|---|
| `src/meridian/lib/config/workspace.py` | `parse_workspace_config()` — legacy TOML parser; `resolve_workspace_snapshot()` — fallback dispatch |
| `src/meridian/lib/ops/workspace.py` | `workspace_migrate_sync()`, `workspace_init_sync()` |

---

## Related

- [config-schema.md](config-schema.md) — new `[workspace.NAME]` entry format
- [resolution.md](resolution.md) — how legacy roots flow into the snapshot model
- [decisions/workspace.md#d41](../../decisions/workspace.md#d41) — why the format changed
- [decisions/workspace.md#d43](../../decisions/workspace.md#d43) — why `enabled` was dropped
