# research/cursor-sandbox-architecture — Cursor Sandbox and Approval Surface

Investigated 2026-06-03 during meridian-cli#309 / mars-agents#91. Cursor's
sandbox and approval surfaces were undocumented; this captures what was found
by reproducing the permission behavior and inspecting Cursor internals.

---

## Cursor Sandbox

Cursor exposes a binary sandbox toggle via CLI:

```
cursor agent --sandbox enabled    # restricts file access
cursor agent --sandbox disabled   # unrestricted
```

Internally, Cursor uses three modes: `WORKSPACE_READWRITE`, `WORKSPACE_READONLY`,
`INSECURE_NONE`. The CLI flag maps `enabled` → `WORKSPACE_READWRITE` (or
`WORKSPACE_READONLY` depending on config), `disabled` → `INSECURE_NONE`.

### Fine-grained path control

Cursor's sandbox supports `additionalReadwritePaths` and `additionalReadonlyPaths`
via `.cursor/sandbox.json`. This is an internal config format — not documented for
CLI agent use. We decided NOT to materialize `.cursor/sandbox.json` because:
- Conflicts with user's own sandbox config
- Couples mars to Cursor's internal config format
- No cleanup mechanism when agents are removed

### Key finding: file deletion is approval-gated, not sandbox-gated

Reproduced in a toy repo: with sandbox enabled, agents CAN read/write workspace
files. File **deletion** is blocked by the approval gate (`--force` / `--yolo`),
not the sandbox. This was the root cause of the c1005 incident where a Cursor
Composer subagent couldn't delete files.

---

## Cursor Approval

Cursor supports approval via CLI flags:

| Flag | Behavior |
|---|---|
| (none) | Cursor default (user approves tool calls) |
| `--force` | Auto-approve safe operations |
| `--yolo` | Approve everything including destructive ops |

There is no `--confirm` or equivalent — Cursor's default behavior IS confirm.

---

## Mars Mapping Decision

### Sandbox mapping (Approximate)

| Mars sandbox | Cursor CLI | Notes |
|---|---|---|
| `default` | (omit) | |
| `read-only` | `--sandbox enabled` | |
| `workspace-write` | `--sandbox disabled` | Overshoot — no fine-grained workspace-write |
| `danger-full-access` | `--sandbox disabled` | |

Classification is `Approximate` because Cursor's binary toggle can't express
`workspace-write` vs `danger-full-access` — both map to `disabled`.

### Approval mapping (Approximate)

| Mars approval | Cursor CLI | Notes |
|---|---|---|
| `default` | (omit) | |
| `auto` | `--force` | |
| `confirm` | (omit) | No equivalent; Cursor default is confirm-like |
| `yolo` / `never` | `--yolo` | |

Classification is `Approximate` because `confirm` has no explicit flag —
it falls back to Cursor's default, which is behaviorally similar but not
explicitly set.

---

## Implementation Split

- **mars-agents#91**: Compilation matrix fix — sandbox and approval classified
  as `Approximate` (was `Dropped`) in `lower_to_cursor_with_model`.
- **meridian-cli#309**: Runtime projection fix — wire `--sandbox` and approval
  flags in `project_cursor.py`. Not yet implemented.

The mars side is classification metadata only — no values are emitted into the
Cursor native YAML artifact. Meridian's harness projector reads from the
`.mars/agents/` canonical artifact and projects the appropriate CLI flags at
spawn time.
