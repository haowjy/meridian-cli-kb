# Workspace Config Schema

Workspace config is declared using `[workspace.<name>]` sections in `meridian.toml` (committed) and `meridian.local.toml` (gitignored). This page describes the TOML format, entry naming rules, and validation behavior.

See [decisions/workspace.md](../../decisions/workspace.md#d42) for why named entries were chosen over anonymous arrays.

---

## TOML Format

```toml
# meridian.toml — committed team conventions
[workspace.frontend]

[workspace.prompts]
path = "../prompts/meridian-base"

# meridian.local.toml — per-developer overrides (gitignored)
[workspace.frontend]

[workspace.local-data]
path = "/data/large-dataset"             # local-only root not in committed config
```

The committed file declares the *expected* multi-repo layout for the project. Developers who match the convention need no local file. Those with non-standard checkouts override just the entries that differ.

---

## Entry Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `path` | `string` | Yes | Relative paths anchor to `project_root`; tilde expanded |

Only `path` is recognized. Unknown keys within an entry are captured as `workspace_unknown_key` findings but do not block loading.

---

## Entry Name Rules

Entry names must match `^[a-z][a-z0-9_-]*$` — lowercase ASCII, digits, hyphens, underscores, starting with a letter.

Names are:
- **Stable merge keys** — the name is how local entries override committed entries. A local `[workspace.frontend]` replaces the committed `[workspace.frontend]` entirely.
- **CLI identifiers** — shown in `meridian config show` output and doctor findings.
- **Not exported as env vars** — workspace is filesystem scope, not working memory. Names never become `MERIDIAN_CONTEXT_*_DIR` variables.

A name that violates the pattern fails validation and produces a `workspace_invalid` snapshot, blocking launch.

---

## Validation Rules

Validation runs during `resolve_workspace_snapshot()` in `src/meridian/lib/config/workspace.py`. The loader validates per-layer before merging.

| Condition | Outcome |
|---|---|
| `path` field absent | `workspace_invalid` — blocks launch |
| `path` is empty or whitespace-only | `workspace_invalid` — blocks launch |
| `path` is not a string | `workspace_invalid` — blocks launch |
| Entry name violates `^[a-z][a-z0-9_-]*$` | `workspace_invalid` — blocks launch |
| Unknown keys in an entry | `workspace_unknown_key` finding — warning only |
| TOML parse error in either file | `workspace_invalid` — blocks launch |
| `[workspace]` value is not a table | `workspace_invalid` — blocks launch |

`workspace_invalid` produces `WorkspaceSnapshot(status="invalid")`. The launch gate in `src/meridian/lib/launch/workspace.py` raises `ValueError` on invalid status, surfacing the finding message to the user.

---

## Path Resolution

Relative paths in `path` are resolved against `project_root` (the directory containing `meridian.toml`). Tilde (`~`) is expanded. Absolute paths are used as-is.

Resolution is done in `_resolve_named_workspace_root_path()`:

```python
candidate = Path(declared_path).expanduser()
if not candidate.is_absolute():
    candidate = project_root / candidate
return candidate.resolve()
```

The resolved absolute path is stored in `ResolvedWorkspaceRoot.resolved_path`. Whether the path exists on disk is checked at resolution time via `resolved_path.is_dir()`.

---

## WorkspaceEntryConfig Type

```python
class WorkspaceEntryConfig(BaseModel):
    model_config = ConfigDict(frozen=True)

    path: str                            # required; no default
    extra_keys: dict[str, object]        # unknown keys captured for findings
```

Defined in `src/meridian/lib/config/workspace.py`.

---

## Where Workspace Config Does Not Live

- **`~/.meridian/config.toml`** (user config) — no workspace section. Workspace expands filesystem access and must be declared at repo level. See [decisions/workspace.md#d46](../../decisions/workspace.md#d46).
- **`MeridianConfig`** — workspace bypasses the main settings model entirely. See [decisions/workspace.md#d45](../../decisions/workspace.md#d45).
- **`workspace.local.toml`** — legacy format, deprecated. See [migration.md](migration.md).

---

## Related

- [resolution.md](resolution.md) — how entries from both files are merged into a `WorkspaceSnapshot`
- [migration.md](migration.md) — legacy `workspace.local.toml` format
- [decisions/workspace.md](../../decisions/workspace.md) — all workspace design decisions
