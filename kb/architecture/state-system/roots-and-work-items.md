# State Roots and Work Items


Work items use a different storage pattern from spawns: **one `__status.json` file per work directory** under the context work root (e.g. `<context.work>/<slug>/__status.json`). The work store is split into three modules: `work_state.py` owns models (`WorkItem`, `StoredWorkItemState`), the metadata codec, slug normalization, and shared directory-location primitives; `work_store.py` provides pure read projections; `work_repository.py` serializes all mutations (status updates, healing, directory-namespace operations) behind the stable project-level `work-store.flock`.

Archiving moves the entire directory to the archive root. Directory location is the **sole** authority for active-vs-archived state; `archived_at` is stored but never decides. Archive and reopen operations move the directory first, then update `__status.json` inside the moved directory. Work-item `status` is an open string vocabulary (not a closed enum) because custom labels exist; `"done"` is reserved for archived items; empty status is rejected.

## ID Generation

**Project IDs** come from the identity seam ([project identity](project-identity.md)).
A committed project's three-word id is minted by `ensure_project_identity()`,
which atomically edits `meridian.toml` under the project-config transaction;
concurrent writers converge by re-reading under that lock. An ad-hoc id is a
path hash and is never written into the directory.

**Spawn IDs** and **session IDs** come from monotonic counters under their
store locks and render as `p1`, `p2`, … and `c1`, `c2`, … respectively.

## Read vs Write Resolution

Read paths call `resolve_project_runtime_root_or_none()` when zero state is a
valid result, or `resolve_project_runtime_root()` when identity is required.
Neither creates identity: both go through the read authority, which uses
`resolve_project_identity()`. Write paths call
`resolve_project_runtime_root_for_write()`, which goes through
`ensure_project_identity()`.

| Resolver | Mutates identity? | Result |
|---|---|---|
| `resolve_project_runtime_root_or_none()` | No | Runtime root, or `None` (unminted committed id or unmaterialized ad-hoc place) |
| `resolve_project_runtime_root()` | No | Runtime root or error |
| `resolve_project_runtime_root_for_write()` | Yes: mints a committed id or materializes the ad-hoc root | Runtime root |

## Related Pages

- [State system overview](overview.md) — the complete state-root layout
- [Project identity](project-identity.md) — tiers, ad-hoc keys and retention
- [State model](../../concepts/state-model.md) — why committed identity is separate from runtime state
- [Workspace architecture](../workspace/overview.md) — context resolution that selects the work root
