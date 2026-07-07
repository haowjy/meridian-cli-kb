# Sync Model

`mars sync` runs the full package pipeline: load config, resolve, target, plan,
apply, and sync managed targets. Each phase produces a typed handoff struct.
The entire cycle is atomic at the file level and idempotent — running it twice
produces the same result.

Top-level entry: `sync::execute()` in `src/sync/mod.rs` lines 132–140.

## Phase Structs

```mermaid
graph LR
    L["LoadedConfig"] --> R["ResolvedState"] --> T["TargetedState"] --> P["PlannedState"] --> A["AppliedState"] --> S["SyncedState"]
```

| Struct | Produced by | Contains |
|---|---|---|
| `LoadedConfig` | `load_config()` | Parsed `mars.toml`, old lock, mutations, sync lock |
| `ResolvedState` | resolver | Version-pinned dependency graph |
| `TargetedState` | targeting layer | Compiler plan for all target directories |
| `PlannedState` | `sync/plan.rs` | Diff-based action list per item |
| `AppliedState` | `sync/apply.rs` | File operations completed, outcomes recorded |
| `SyncedState` | `target_sync` | Native target dirs updated, lock written |

`SyncRequest` carries the resolution mode (normal, maximize, frozen), optional
config mutation, and sync options (`src/sync/mod.rs` lines 59–81).

## Diff Classification

The diff phase (`src/sync/diff.rs`) compares the current planned state against
the installed state from `mars.lock`. Each item receives one of six
classifications:

| Classification | Condition |
|---|---|
| `Add` | Item is new — not in prior lock |
| `Update` | Item changed — content hash differs |
| `Unchanged` | Content hash matches lock |
| `Conflict` | Item was locally modified (dual-hash check) |
| `Orphan` | Item was in prior lock but is absent from current graph |
| `LocalModified` | On-disk content differs from the lock's installed hash |

Local modification detection uses dual checksums: the lock stores both the
content hash at install time and the on-disk hash at last check. If they
diverge, the item is flagged as `LocalModified` and protected from overwrite
unless `--force` is passed.

## Plan → Apply

`sync/plan.rs` maps each diff classification to an action:

| Diff state | Action (normal) | Action with --force |
|---|---|---|
| `Add` | Install | Install |
| `Update` | Overwrite | Overwrite |
| `Unchanged` | Skip | Skip |
| `Conflict` | Keep-local + warn | Overwrite |
| `Orphan` | Remove | Remove |
| `LocalModified` | Keep-local + warn | Overwrite |

`sync/apply.rs` executes the actions:

| Action | Operation |
|---|---|
| Install | Atomic copy (tmp+rename) |
| Overwrite | Atomic copy (tmp+rename) |
| Merge | Three-way merge via `git merge-file` |
| Remove | Safe removal |
| Skip | No-op |
| Keep-local | No-op, records warning |

All file operations go through `reconcile/fs_ops.rs`, which wraps the atomic
primitives.

## Config Mutations

`sync/mutation.rs` handles in-place config changes under sync lock:

- Batch upserts (add/update dependencies)
- Removes
- Overrides
- Rename rules

Mutations are applied to `mars.toml` before resolution runs, so the resolved
graph reflects the post-mutation config. Writes are atomic and round-trip
checked.

## Lock File and Provenance

`mars.lock` is the authority on installed state. It is schema v2, keyed by
logical item identity (`kind/name`):

```toml
version = 2

[items."agent/coder"]
source = "meridian-base"
url = "https://github.com/org/meridian-base"
commit = "abc123def456"
version = "1.2.0"
dest_path = ".mars/agents/coder.md"
content_hash = "sha256:..."

[config_entries."claude/.mcp.json"]
source = "meridian-base"
key = "some-mcp-server"
target_root = "claude"
```

The `items` section is the ownership registry: provenance and content hash per
managed file. The `config_entries` section records provenance for installed MCP
and hook entries.

On each sync, `lock::build()` in `src/lock/mod.rs` lines 420–639 reconstructs
the lock from the resolved graph plus apply outcomes. Skipped and kept-local
items are carried forward unchanged.

Lock writes are always atomic. Legacy v1 lock files are promoted transparently
at read time and re-written as v2 (`load_with_diagnostics()` in
`src/lock/mod.rs` lines 291–354).

`LockIndex` (`src/lock/mod.rs` lines 127–176) is a fast lookup overlay for
repeated dest-path queries during the diff phase.

## Rename and Rewrite Pass

After unmanaged-collision pruning, `sync/rewrite.rs` builds one `RenameIndex`
from explicit config renames and automatic collision renames, then applies a
single rewrite pass per agent. Each agent's `skills:` and `subagents:`
frontmatter is rewritten in one content update — no double-rewrite.

Resolution for which renamed variant to wire into an agent:
1. Same-source copy wins (the agent's own source)
2. If the agent's source still owns an unrenamed copy, the ref is left alone
3. Otherwise fall back to mars.toml declaration order (not `graph.order`,
   which is alphabetical)

After rewriting, `sync/validate.rs` checks config-side name references
(`[settings.meridian.fanout].agents`, `[agents.<name>]`, `[skills.<name>]`)
against installed names and emits a `config-rename-dangle` warning when a
referenced name was renamed away. See
[decisions/package-management.md#D88](../../decisions/package-management.md)
and
[#D89](../../decisions/package-management.md)
for the policy rationale.

## Sync Modes

| Flag | Behavior |
|---|---|
| (default) | MVS version selection, replay locked commits; models.dev catalog **Auto** + probe **Background** |
| `--force` | Overwrite locally-modified files |
| `--diff` | Dry-run: report what would change, no writes |
| `--frozen` | Do not fetch new versions; fail if lock is insufficient |
| `--refresh-models` | Force models.dev catalog refresh; run harness probes **synchronously** (no background `__refresh-probe` on stale cache) |
| `--no-refresh-models` | Disk-only catalog (`RefreshMode::Offline`); probe **Skip** (stale probe JSON still used when present) |

`--frozen` is the right mode for CI builds where reproducibility is required.

Model/probe refresh uses the same **`ModelsRefreshControl`** as `mars models list|resolve`
and `mars build launch-bundle`. Full matrix: [../../architecture/mars-model-refresh.md](../../architecture/mars-model-refresh.md).

## Filter Pass

`sync/filter.rs` applies the include/exclude/only-agent/only-skill modes from
each dependency declaration. Agents passing an include filter also pull their
declared skill dependencies transitively. Filter resolution runs before the diff
phase.

## Sync Lock

`load_config()` acquires a sync lock (file lock via `fcntl.flock` on POSIX,
`LockFileEx` on Windows) before any state modification. Concurrent `mars sync`
invocations block rather than racing. The lock is held for the duration of the
sync and released on completion or crash.

## Invariants

- **I-1: Atomic writes** — every file write is tmp+rename; partial writes do not
  corrupt state.
- **I-2: Lock guards concurrency** — only one sync runs at a time per project root.
- **I-3: Idempotent** — syncing twice with no source changes produces identical
  output and no warnings.
- **I-4: Local modifications are preserved by default** — `LocalModified` and
  `Conflict` items are kept unless `--force` is passed.
- **I-5: Orphan cleanup** — items that were installed in a prior sync but removed
  from the current graph are removed from both `.mars/` and native target dirs.
- **I-6: v2 lock is always written** — v1 is a read-time compatibility path only;
  any write produces v2.

## Key References

- Top-level entry: `src/sync/mod.rs` lines 132–140
- Phase structs: `src/sync/mod.rs` lines 83–130
- Diff classification: `src/sync/diff.rs`
- Plan mapping: `src/sync/plan.rs`
- Apply operations: `src/sync/apply.rs`
- Config mutations: `src/sync/mutation.rs`
- Filter pass: `src/sync/filter.rs`
- Skill rewrite: `src/sync/rewrite.rs`
- Lock build: `src/lock/mod.rs` lines 420–639
- Lock index: `src/lock/mod.rs` lines 127–176
- Atomic file ops: `src/reconcile/fs_ops.rs`

## Related

- [compiler-pipeline.md](compiler-pipeline.md) — what runs during the compile phase
- [targeting.md](targeting.md) — how the apply outputs are projected to harness dirs
- [resolution-algorithm.md](resolution-algorithm.md) — how the ResolvedState is produced
- [decisions/package-management.md](../../decisions/package-management.md) — why sync is manual, why the lock extends to config-entry provenance
- [../../architecture/mars-model-refresh.md](../../architecture/mars-model-refresh.md) — `ensure_fresh`, `ProbeRefreshMode`, refresh flags on sync
