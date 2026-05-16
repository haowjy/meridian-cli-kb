# operations/bootstrap-command — `meridian bootstrap`

How `meridian bootstrap --add` works: first-run project setup followed by a guided launch with the installed primary agent.

---

## What It Does

`meridian bootstrap --add <package> [<project_dir>]` runs the same init sequence as `meridian init --add`, then launches a guided session with the package's primary agent:

1. **Bootstrap `meridian.toml`** — creates the file if absent, idempotent if present
2. **`mars init`** — initializes `mars.toml` if absent, skipped if it exists
3. **`mars add <package>`** — installs the package into `.mars/`
4. **`mars link <targets>`** — materializes content into harness directories
5. **Set `primary.agent`** in `meridian.toml`
6. **Launch guided bootstrap session** — starts a session with the installed primary agent

Steps 1–5 are identical to `meridian init --add`. Step 6 is the key addition: after setup completes, bootstrap launches the primary agent's guided onboarding session so new users land directly in a working environment rather than having to run a separate command.

---

## Difference from `meridian init --add`

| | `meridian init --add` | `meridian bootstrap --add` |
|---|---|---|
| Bootstraps `meridian.toml` | ✓ | ✓ |
| Installs package via mars | ✓ | ✓ |
| Links harness directories | ✓ | ✓ |
| Sets primary agent | ✓ | ✓ |
| Launches guided session | — | ✓ |
| Primary audience | Re-run or scripted setup | New users, first run |

`init` is designed for idempotent re-runs and scripted/CI use. `bootstrap` is designed as the new-user entry point — run it once, get a working project plus an immediate guided session.

---

## Rootless Startup Class

`meridian bootstrap` uses a `SERVICE_ROOTLESS` startup class (no pre-handler runtime bootstrap). This matters because:

- Normal command startup calls `prepare_for_runtime_write()` in the pre-handler, which resolves and validates the user's meridian state root before the command runs.
- Setup flows (`bootstrap`, `init`) do their own root resolution inside `run_init_flow()`. Running the pre-handler bootstrap first would try to validate state that doesn't exist yet for a brand-new user.
- `SERVICE_ROOTLESS` lets dry-run rejection and root resolution happen correctly inside the command's own logic rather than the pre-handler.

In practice: bootstrap works correctly even when called from a directory with no prior meridian setup, which is exactly the first-run case.

---

## Idempotency

Running `meridian bootstrap --add <same-package>` after a project is already initialized is safe. The init steps (1–5) are individually idempotent — see [init-command.md](init-command.md#idempotency) for details. The guided session launch (step 6) will start a new session, but no state is corrupted. Idempotency means bootstrap can be re-run if interrupted.

---

## Cross-References

- [operations/init-command.md](init-command.md) — `meridian init --add` sequence and idempotency details
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — mars package model
- [concepts/package-management/targeting.md](../concepts/package-management/targeting.md) — how targets work
- [operations/configuration-guide.md](configuration-guide.md) — meridian.toml and project config
