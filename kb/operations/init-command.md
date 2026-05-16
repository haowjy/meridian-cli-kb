# operations/init-command — `meridian init --add`

How `meridian init --add` works: the sequence of operations, auto-link behavior, primary agent handling, and idempotency properties.

---

## Command Contract

`meridian init --add <package> [<project_dir>]` runs a fixed sequence:

1. **Bootstrap `meridian.toml`** — creates the file if absent, idempotent if present
2. **`mars init`** — initializes `mars.toml` if absent, skipped if it exists
3. **`mars add <sources>`** — installs packages into `.mars/`
4. **Determine link targets** — explicit `--link` overrides; otherwise uses package-declared targets
5. **`mars link <target>`** for each resolved target — materializes content into harness directories
6. **Set `primary.agent`** — conditionally updates `meridian.toml` based on package declaration

Bare `meridian init` (no `--add`) runs only step 1 and exits. It creates `meridian.toml` without touching `mars.toml` or installing anything.

---

## Auto-Link Behavior

Packages declare required targets in their `mars.toml`:

```toml
[package.targets]
required = [".claude"]
```

When `meridian init --add` completes `mars add`, it reads the `declared_targets` from the mars JSON response and links each one automatically. This is the default path — users don't need to know which harness directories a package wants.

**`--link` fully overrides the declaration.** If the user passes `--link .codex`, the explicit list replaces the package-declared targets entirely. The package-declared `.claude` is NOT linked. This is intentional: `--link` is an expert override for users who know what they want.

---

## Primary Agent Handling

Three outcomes when a package declares `primary_agent` in its `mars.toml`:

| `meridian.toml` state | Outcome |
|---|---|
| `primary.agent` unset | Writes the package value; reports `action: set` |
| `primary.agent` = package value | No-op; reports `action: already_set` |
| `primary.agent` = different value | Prints recommendation message; reports `action: differs` |

The "differs" case does not overwrite. It prints: `Package recommends '<agent>' as primary agent. Current primary is '<current>'. Run meridian -a <agent> to try it, or meridian config set primary.agent <agent> to change your default.`

This is deliberate: `meridian init --add` should not silently override a user-configured default. The user's explicit config wins; the package declaration is advisory.

---

## Idempotency

Running `meridian init --add <same-package> <same-dir>` twice is safe. Each step is individually idempotent:
- `meridian.toml` bootstrap is a no-op if the file exists
- `mars init` is skipped when `mars.toml` exists
- `mars add` is idempotent — reinstalling the same package produces the same result
- `mars link` is idempotent — re-linking an already-linked target is safe
- Primary agent: already_set case is a no-op

---

## JSON Output

Pass `--json` (as a top-level meridian flag) to get machine-readable output:

```bash
meridian --json init --add <package>
```

Returns an `InitResult` object:

```json
{
  "ok": true,
  "project_root": "/path/to/project",
  "config_created": false,
  "packages_added": ["<package>"],
  "packages_requested": ["<package>"],
  "targets_linked": [".claude"],
  "content": {"agents": ["coder", "reviewer"], "skills": ["spawn"]},
  "primary_agent": {"action": "set", "agent": "coder"},
  "next_step": "Run `meridian` to start."
}
```

**`packages_requested`** holds the user-supplied specifiers (what was passed to `--add`), not the full resolved dependency set. The resolved set — including transitive deps — is internal to mars. See [decisions/package-management.md](../decisions/package-management.md#d79).

**`content`** is dynamically discovered from `.mars/` subdirectories, not a fixed schema. New content types added by future mars versions appear automatically. See [decisions/package-management.md](../decisions/package-management.md#d78).

---

## Cross-References

- [decisions/package-management.md](../decisions/package-management.md#init-command) — D78 (content scan pattern) and D79 (honest naming)
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — mars package model
- [concepts/package-management/targeting.md](../concepts/package-management/targeting.md) — how targets work
- [operations/configuration-guide.md](configuration-guide.md) — meridian.toml and project config
