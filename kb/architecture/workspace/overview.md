# Workspace Architecture — Overview

The workspace system controls which filesystem directories are projected into harness subprocess scope. It is a **filesystem permission** mechanism, not a working-memory mechanism — workspace roots become `--add-dir` arguments, never env vars or system prompt content.

See [decisions/workspace.md](../../decisions/workspace.md) for the full rationale behind this boundary.

---

## The Trust Boundary

```
Workspace = filesystem scope.  Gets --add-dir only.  No env vars.  No prompt mentions.
Context   = working memory.    Gets MERIDIAN_CONTEXT_*_DIR env vars + system prompt.
```

Workspace and context are separate systems with separate config keys, separate loaders, and separate consumers. See [concepts/context-resolution.md](../../concepts/context-resolution.md) for the context side.

---

## Two-File Config Model

Workspace config lives in the same TOML files as other project config:

| File | Scope | Committed to VCS? |
|---|---|---|
| `meridian.toml` | Team-wide conventional layout | Yes |
| `meridian.local.toml` | Per-developer path overrides and additions | No (gitignored) |

The committed section declares the *expected* multi-repo layout. Developers whose checkouts differ override or extend with their own `meridian.local.toml`. The loader merges both layers at resolution time — local entries win on name conflict.

---

## Pages in This Subtree

| Page | What it covers |
|---|---|
| [config-schema.md](config-schema.md) | TOML format, `[workspace.NAME]` entry structure, validation rules |
| [resolution.md](resolution.md) | Merge algorithm, `WorkspaceSnapshot`, path evaluation, projectable roots |
| [migration.md](migration.md) | Legacy `workspace.local.toml` fallback and `workspace migrate` command |

---

## How Workspace Roots Reach the Harness

At launch time, `resolve_workspace_snapshot_for_launch()` (`src/meridian/lib/launch/workspace.py`) resolves the snapshot and raises immediately on `status == "invalid"`. The snapshot then flows to `get_projectable_roots()`, which filters to enabled, existing paths. Those paths are passed to harness adapters:

- **Claude / Codex subprocess:** `--add-dir <path>` per root
- **Codex app-server:** `-c sandbox_workspace_write.writable_roots=[...]` config flag
- **OpenCode:** injected via `OPENCODE_CONFIG_CONTENT` with `permission.external_directory` entries

Missing paths (non-existent directories) are silently excluded from projection — a partial checkout does not prevent launch. Invalid config (schema errors, unparseable TOML) blocks launch entirely.

### `projected_roots` structural seam

Roots travel through `ResolvedLaunchSpec.projected_roots`, not through `extra_args` as synthetic `--add-dir` flags. This keeps Meridian-owned policy data separate from user passthrough. Each harness projection consumes the field directly, and the Codex remote TUI attach command (`codex resume --remote ws://...`) receives matching `--add-dir` flags so the TUI's per-turn sandbox overrides do not narrow scope below the app-server's original configuration. See [decisions/workspace.md — D47](../../decisions/workspace.md#d47).

---

## Key Source Files

| File | Role |
|---|---|
| `src/meridian/lib/config/workspace.py` | Core parser, types, snapshot builder |
| `src/meridian/lib/launch/workspace.py` | Launch gate — raises on invalid, returns snapshot |
| `src/meridian/lib/ops/workspace.py` | `workspace init` and `workspace migrate` commands |

---

## Related

- [decisions/workspace.md](../../decisions/workspace.md) — why this architecture
- [concepts/context-resolution.md](../../concepts/context-resolution.md) — context vs workspace trust boundary
- [architecture/launch-system.md](../launch-system.md) — how workspace roots feed into launch composition
- [operations/configuration-guide.md](../../operations/configuration-guide.md) — practical setup
