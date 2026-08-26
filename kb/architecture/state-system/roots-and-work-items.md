# State Roots and Work Items


Work items use a different storage pattern from spawns: **one `__status.json` file per work directory** under the context work root (e.g. `<context.work>/<slug>/__status.json`). The work store is split into three modules: `work_state.py` owns models (`WorkItem`, `StoredWorkItemState`), the metadata codec, slug normalization, and shared directory-location primitives; `work_store.py` provides pure read projections; `work_repository.py` serializes all mutations (status updates, healing, directory-namespace operations) behind the stable project-level `work-store.flock`.

Archiving moves the entire directory to the archive root. Directory location is the **sole** authority for active-vs-archived state; `archived_at` is stored but never decides. Archive and reopen operations move the directory first, then update `__status.json` inside the moved directory. Work-item `status` is an open string vocabulary (not a closed enum) because custom labels exist; `"done"` is reserved for archived items; empty status is rejected.

## ID Generation

**Project IDs:** `get_or_create_project_id()` reads `[project].id`, migrates a
legacy `.meridian/id`, or generates a three-word ID. It atomically edits
`meridian.toml`; concurrent writers converge by re-reading under the config
lock.

**Spawn IDs** and **session IDs** come from monotonic counters under their
store locks and render as `p1`, `p2`, … and `c1`, `c2`, … respectively.

## Read vs Write Resolution

Read paths call `resolve_project_runtime_root_or_none()` when zero state is a
valid result, or `resolve_project_runtime_root()` when identity is required.
Neither creates identity. Write paths call
`resolve_project_runtime_root_for_write()`, which resolves write authority and
creates or migrates identity when needed.

| Resolver | Mutates identity? | Result |
|---|---|---|
| `resolve_project_runtime_root_or_none()` | No | Runtime root or `None` |
| `resolve_project_runtime_root()` | No | Runtime root or error |
| `resolve_project_runtime_root_for_write()` | Yes, when absent/legacy | Runtime root |

## Identity Compatibility Migration

The active migration is implemented in `lib/ops/migration.py`, not a stub in a
repo-side migrations registry. `get_project_id()` reads committed
`meridian.toml` first and falls back to `.meridian/id`. On the first write,
`get_or_create_project_id()` runs `migrate_legacy_project_identity()` under the
project-config transaction, atomically writes the legacy value into
`meridian.toml`, and then resolves the same user runtime root. Without a legacy
value it generates a collision-checked three-word ID.

The legacy file is compatibility input, not a second current identity store.


## Related Pages

- [State system overview](overview.md) — the complete state-root layout
- [State model](../../concepts/state-model.md) — why committed identity is separate from runtime state
- [Workspace architecture](../workspace/overview.md) — context resolution that selects the work root
