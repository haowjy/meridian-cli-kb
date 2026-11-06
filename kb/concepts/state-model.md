# State Model

Committed project identity survives a repository move and is shared by clones that carry its config. An ad-hoc identity follows a canonical path, so it does not survive a move or merge into a later committed project. Work items live in a separate configured context root because they are shared task artifacts. See [project identity](../architecture/state-system/project-identity.md).

Files are authoritative; indexes are rebuildable projections. Read-only observation derives status without authorizing destructive cleanup. See the [state architecture](../architecture/state-system/overview.md) and [state decisions](../decisions/state.md) for the storage and recovery contracts.

## Root Resolution

The project root is selected from `-C`, `MERIDIAN_PROJECT_DIR`, then the literal cwd. Package discovery can walk to an ancestor independently of project identity. Runtime roots derive from identity, with an internal override for managed execution; reads do not mint identity. The source guidance is `src/meridian/lib/state/AGENTS.md`.
