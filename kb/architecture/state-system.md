# Architecture: State System

Meridian state is files. No database, no service, no hidden in-memory state. The state system enforces this by making writes atomic, reads crash-tolerant, and keeping recovery derivable from disk. Read paths can project a reconciled view without side effects; repair paths make the durable changes.

See [concepts/state-model.md](../concepts/state-model.md) for the mental model. This page explains the mechanics.

## Split State Layout

State divides across committed configuration, a user-local runtime root, and a configured context work root:

```
meridian.toml
  [project]
  id = "three-word-id"              — committed authoritative identity

~/.meridian/projects/.locks/<id>.lock — project-lifetime gate
~/.meridian/projects/<id>/          — user runtime, never committed
  sessions.jsonl                    — session events
  session-id-counter · spawn-id-counter
  sessions/ · locks/
  spawns/
    v2-format.json · .staging/<unique>/
    <spawn-id>/
      state.json                    — authoritative spawn row
      starting-prompt.md · prompt.md · report.md · heartbeat
      history.jsonl · stderr.log · params.json · tokens.json
      attempt-N/ · runner-lifecycle.jsonl · process_scopes.json
  artifacts/ · cache/ · .migrations.json

<context.work root>/<slug>/         — context-resolved work state and artifacts
```

`[project].id` selects the runtime directory. `user_paths.py` still reads a
legacy `.meridian/id` when config has no ID; the first write migrates that value
(or generates a three-word ID) into `meridian.toml` by atomic replacement.
Repo-local `.meridian/` is not an active state root.

Control sockets live outside spawn directories in the per-user POSIX temp
root. Work items live under `[context.work]` and archive beside that work root.
See `docs/configuration.md` in meridian-cli for context-path resolution.

## Spawn State: V2 Per-Spawn state.json

Since 2026-05 (spawn-state-v2 migration), spawn state lives in individual `state.json` files — one per spawn — rather than a single global `spawns.jsonl` event log.

**Why the migration:** the production global event log had grown enough that
every status read replayed substantial project history. Per-spawn `state.json`
makes an individual read O(1), independent of total event history. The dataset
and before/after measurements are recorded once below.

**Measured migration result:** in the production dataset that triggered the
migration (about 189 MB / 35,000 events), primary launch fell from 12–13 seconds
to 0.67 seconds and `list_spawns()` measured about 386 ms across roughly 4,000
spawn files. Provenance: `work:spawn-state-v2`, benchmark notes from the May
2026 state-v2 campaign. These historical measurements are not current
performance guarantees.

### Spawn Status Machine

Status progression is v2 per-spawn state plus the current terminal set:

```mermaid
stateDiagram-v2
    [*] --> queued: start_spawn()
    queued --> running: mark_running()
    running --> finalizing: mark_finalizing() CAS
    finalizing --> succeeded: finalize_spawn(succeeded)
    finalizing --> failed: finalize_spawn(failed)
    finalizing --> timed_out: finalize_spawn(timed_out)
    running --> failed: orphan reap
    running --> timed_out: timeout
    running --> cancelled: cancel()
    finalizing --> cancelled: cancel() (rare)
    finalizing --> failed: orphan_finalization reap
```

Terminal statuses are `succeeded`, `failed`, `cancelled`, and `timed_out`. `timed_out` is a failure class distinct from generic `failed`, so user-facing filters and statistics can separate deadline failures from other errors. `SpawnStatus` is a `StrEnum` (`core/domain.py`); lifecycle sets are derived from a member→class map.

Lifecycle evidence is nested into frozen sub-models (`RunnerExitFacts`, `TerminalFacts`). Top-level `status` is the sole status authority; `TerminalFacts` carries exit code, timestamps, metrics, error, and origin but does not repeat status. When `status` is terminal, `terminal` must not be `None`; active rows must not carry `terminal`. `StoredSpawnState` uses `extra="forbid"`, so persisted rows with a nested `terminal.status` or stale flat fields are quarantined. Collection reads partition valid rows from quarantine reports in immutable `SpawnScan` envelopes. See [spawn-finalization.md](spawn-finalization.md) for the discriminated facts schema and quarantine contract.

**Terminal writes use the projection authority rule**: a runner-origin terminal write supersedes a reconciler-origin write on the same spawn. See [spawn-finalization.md](spawn-finalization.md) for the full authority lattice.

`mark_finalizing()` is a compare-and-swap from `running` → `finalizing`. It narrows the reaper's target from the full execution window to the drain/report window, enabling `orphan_finalization` vs `orphan_run` distinction.

### Locked Mutation Seam

Every update to a published spawn calls `write_state_locked()`. It acquires the stable per-spawn lock at `locks/spawns/<id>.lock`, re-reads current `state.json`, applies a pure mutator function, and writes atomically. There is no public unlocked write path — the prior two-tier model (owner writes without lock / external writes with lock) was collapsed in PR #422 to eliminate the convention-enforced split that was the root cause of every reproduced lost-update bug.

`start_spawn()` creates the initial `state.json` under the global `spawns_flock`, where ID reservation is also serialized. Once a spawn row is published, all subsequent mutations go through `write_state_locked()`.

The same mutate-under-lock shape applies across all stores:
- **Spawn state**: `write_state_locked()` — `locks/spawns/<id>.lock`
- **Archived spawns**: `mutate_archived_spawns()` — `locks/archived-spawns.lock`
- **Work items**: `_mutate_item()` in `work_repository.py` — `work-store.flock`
- **Hook intervals**: `run_if_due()` — `locks/hooks/<name>.lock`
- **Scope projections**: `_mutate_scope_projection()` (private) — `locks/process-scopes/<id>.lock`
- **Autosync**: `transaction()` — canonical sync-root lock path
- **Published-spawn deletion**: `delete_published_spawn()` — same per-spawn lock

These seams are behavior-preserving contracts: a planned future store rewrite (typed state, store scaling) inherits the same lock-acquire / re-read / pure-mutate / atomic-write shape.

### Published Spawn Artifact Lifetime

The published `state.json` row owns the lifetime of every artifact under its
spawn directory. Once that row is gone, no signal, journal, history record,
diagnostic, metadata file, heartbeat, connection, or process-scope registration
may recreate or change the aggregate.

`spawn_aggregate.py` gives deletion and late writers one ordering boundary:

1. `mutate_published_spawn_artifact()` acquires
   `locks/spawns/<id>.lock`.
2. It re-reads `spawns/<id>/state.json` while holding the lock.
3. It optionally evaluates a current-row predicate, such as `status == failed`.
4. It performs the supplied artifact mutation or returns `False` without
   touching the spawn directory.
5. `delete_published_spawn()` takes the same outer lock before removing the
   directory.

If the writer wins, deletion waits and then removes the complete aggregate. If
deletion wins, the writer observes no published row and fails closed. Artifact-
specific locks remain nested inside the spawn lock. Heartbeats are the lighter
case: they never create their parent and stop when deletion makes the path
disappear. Harness connection startup and process adoption likewise require an
already-published directory rather than creating one.

This seam belongs above the persistence leaves because low-level atomic and
JSONL writers cannot decide whether a spawn is still published. See the
[state decision](../decisions/state.md#published-row-lifetime-owns-spawn-artifacts-issue-437-2026-07)
for the rejected alternatives and rationale.

### Migration: ensure_v2_format()

`state/spawn/migration.py:ensure_v2_format()` performs a one-shot lazy migration on first access to a runtime root:

1. If `spawns/v2-format.json` marker exists → already migrated, return immediately (in-process cache hit after first check).
2. If no legacy `spawns.jsonl` exists → write marker and return (fresh install, nothing to migrate).
3. Under `spawns/migration.lock`: replay legacy `spawns.jsonl`, write `state.json` + `starting-prompt.md` for every spawn, write marker, rename legacy files to `spawns.legacy-v1.jsonl`.

**No quiescence gate.** The migration does not wait for active spawns to finish before migrating. Stragglers are handled by reconciliation: read surfaces can project a stale runner as terminal, and explicit repair paths can finalize it durably. The decision to drop the quiescence gate was deliberate: users always have running spawns, so a gate that requires a quiet runtime would never trigger in practice.

**Migration lock for process safety.** Multiple processes starting simultaneously converge: second process reads the marker after first writes it and skips migration. The `migration.lock` file prevents double-migration, not quiescence.

### Legacy V1 JSONL (Reference)

The original design used a global `spawns.jsonl` event log. Events were appended and state was derived by replaying all events for a spawn. This made crash tolerance structural (truncated lines are skippable) but O(n) in total spawn history. V1 files are archived to `spawns.legacy-v1.jsonl` on migration and are no longer read by active code.

## Session State

Sessions track harness session IDs, work-item attachment, and lifecycle (created → active → closed). Session events in `sessions.jsonl` link `meridian_session_id` (c1, c2, …) to `harness_session_id` (harness-native identifier) and `work_id`.

Session state remains event-sourced JSONL (no v2 migration for sessions). The session log is much smaller than spawn history and does not suffer the same O(n) performance problem.

Session-ID counter (`session-id-counter`) is monotonically incremented under `platform.locking.lock_file()` so concurrent spawns never collide.

Per-session files under `sessions/<chat_id>/`:
- `<chat_id>.lock` — held for active session duration
- `<chat_id>.lease.json` — PID + generation token for staleness detection

## Atomic Writes

Every file write goes through one of three patterns:

**JSONL append** (`state/event_store.py`): acquire `lock_file()` on `.flock` sidecar → repair any torn tail → append line → release. If the process dies mid-append, the next locked append repairs the torn tail before writing: a complete row missing only its delimiter is preserved; a genuinely torn partial row is dropped via atomic inode replacement (so unlocked readers never splice a fabricated hybrid event). The O(1) fast path (check last byte for newline) avoids a full-file read on clean tails. Session events, launch-boundary events, permission journals, and control-action journals all use `append_durable_jsonl_line`, which calls the shared repair before appending. `history.jsonl` is excluded (tracked under #376). Spawn state uses atomic overwrite (v2).

**Atomic file replacement** (`lib/platform/atomic.py:atomic_replace()`): the dependency-neutral platform primitive that `state/atomic.py`, `plugin_api/fs.py`, autosync, and the Codex streaming rewriter all delegate to. Writes to a same-directory temp, optionally fsyncs, then `os.replace()`. Permission policy: `permissions="preserve"` (default) keeps existing file mode; `permissions=0o600` enforces strict mode for runtime state. `AtomicReplaceDurabilityError` surfaces post-commit fsync failures so callers know the write is committed but not yet durable.

State-facing writes use `state/atomic.py:atomic_write_text()` which sets mode `0600` for runtime state. User-owned project files and context work-item metadata use the preserve-mode platform atomic writer.

**Atomic directory publication** (`state/atomic.py:atomic_publish_dir()`): rename a complete same-volume staging directory into a destination that must not exist, then fsync the publication parent.

**Work item renames:** `work-items.rename.intent.json` is written before any rename begins. Leftover intent is replayed on startup/reconciliation — crash-safe two-phase rename.

### Conformance Guard

`tests/contract/test_state_write_conformance.py` is a repo-wide AST test rejecting raw file writes (`Path.write_text`, `Path.write_bytes`, `open(..., "w")`) to authoritative state. It enforces that all state mutations route through the atomic primitives. A documented single-entry allowlist covers the one justified exception (telemetry cooldown marker). Stale allowlist entries are detected. The failure message names the offending call site and guides toward the correct primitive.

## Platform Locking

`platform.locking.lock_file(path, mode, timeout, reentrant)` is the single cross-process locking primitive:

- **POSIX:** `fcntl.flock(LOCK_EX | LOCK_SH)` — advisory, kernel-backed
- **Windows:** `msvcrt.locking(LK_NBLCK, 1)` with retry loop (50 ms sleep)

**Modes:** `exclusive` (default) or `shared`. Shared mode uses `LOCK_SH`; a held shared lock cannot be upgraded to exclusive in place.

**Reentrancy:** thread-local by default. A thread that already holds the lock re-enters safely; the OS lock releases only on outermost exit. Non-reentrant mode (`reentrant=False`) is used for mutation seams where nesting would let an inner run invalidate the outer's state snapshot.

**Fork safety:** acquired handles are tracked in a process-wide registry. On `fork()`, the child closes every inherited descriptor (releasing the parent's open-file-description lock without explicit unlock) and clears the reentrancy state. Release-window descriptors are also registered so a fork during the gap between OS release and handle close does not leak.

**Stable lock inodes with GC seam:** all coordination locks live under `locks/<domain>/` outside the directories they protect. This prevents the split-brain failure where one process unlinks a lock file and creates a new inode while another still holds the old one (POSIX `flock` is per-open-file-description, not per-path). Lock inodes are never unlinked except through a validated GC seam: `unlink_validated_lock()` unlinks only while holding a fresh, non-reentrant exclusive flock on the inode currently linked at that path, immediately before release. The acquire-side revalidation loop (`open → flock → compare fstat(fd) vs stat(path) → retry on mismatch`) makes this provably split-brain-free. Two GC call sites use this primitive: `lock_gc.py` sweeps orphaned per-spawn locks (four classes under `locks/`) when the corresponding spawn directory no longer exists; `cleanup_stale_sessions()` unlinks cleaned session locks before release. Both run on episodic paths (doctor, prune, cleanup), never on hot paths. The forbidden pattern — unlinking while a lock remains held afterwards (e.g. inside a reentrant context) — is never used.

`spawn_aggregate.py` owns published-row lifetime coordination. Artifact writers
use `mutate_published_spawn_artifact()` to check publication under the spawn
lock; deletion uses `delete_published_spawn()` to acquire the spawn lock then
the scope-projection lock, check for pending cleanup claims, and remove the
spawn directory. Cross-leaf spawn operations belong in the aggregate, not in
either persistence leaf.

### Lock-Order Invariants

When multiple locks are needed, acquire in this order to prevent deadlocks:

1. `spawns_flock` (global spawn-ID allocation and publication)
2. Per-spawn lock (`locks/spawns/<id>.lock`)
3. Scope-projection lock (`locks/process-scopes/<id>.lock`)

`delete_published_spawn()` (in `spawn_aggregate.py`) acquires the per-spawn lock then the scope-projection lock, and checks for pending cleanup claims before deletion. Pruning acquires `spawns_flock` first.

### Project-Lifetime Gate

`~/.meridian/projects/.locks/<project-id>.lock` sits outside the deletable project root. Sessions hold a **shared** lock for their lifetime; global pruning acquires **exclusive** + revalidates the target before removal. This prevents pruning from destroying a runtime root while sessions hold spawn locks inside it.

See `lib/platform/locking.py` for implementation details.

## Read-Time Projection and Explicit Reconciliation

Meridian no longer lets ordinary read surfaces terminate processes as a side effect.
The state layer has two reconciliation shapes:

- **Read-time projection** — `reconcile_spawns()` and
  `peek_reconciled_active_spawn()` return an in-memory view of stale active spawns
  for list/show/wait/dashboard and descendant-work checks. They do not write
  `state.json`, mark scopes released, or send process signals.
- **Explicit reconciliation repair** — `reconcile_active_spawn()` writes terminal
  state and runs process-scope cleanup. It is called by `meridian doctor
  --kill-orphans` and by the primary-launch background repair thread, both gated to
  root side-effect processes.

Both paths use the same liveness decision rules. Side effects run only from the
explicit path and only at clear root depth — `MERIDIAN_DEPTH` absent, empty, or
`"0"`. Nested processes and malformed depth values fail closed.

**Decision/IO split:** Reconciliation separates the decision step (pure, no I/O) from the action step (writes terminal state and cleans scopes). This lets read-time projection reuse the same decision logic without filesystem mutation.

**Finalize-first cleanup claims:** `reconcile_active_spawn()` snapshots exact cleanup targets into `reaper_cleanup_claim.json` under the spawn lock before persisting terminal state, then terminates the claimed scopes using birth-validated signals. This finalize-first order makes state convergence independent of slow cleanup: a crash leaves a durable claim for the next doctor pass. A separate stable cleanup lock (`locks/reaper-cleanup/<id>.lock`) prevents concurrent reapers from double-signalling. Terminal rows retain failed claims for retry. A runner-origin terminal write clears a reconciler claim without signalling because runner authority supersedes reconciler cleanup intent.

### Liveness Check Sequence

```mermaid
graph TD
    A[Read spawn row] --> B{Status active?}
    B -->|No| DONE[Skip — already terminal]
    B -->|Yes| C{is_root_side_effect_process?}
    C -->|No| DONE2[Skip — nested process, fail closed]
    C -->|Yes| D{Heartbeat age less than 120s?}
    D -->|Yes — recently alive| DONE3[Skip — spawn is running]
    D -->|No — stale heartbeat| E{Status = finalizing?}
    E -->|Yes| F{Completion, cancel, or runner-exit tuple?}
    F -->|Completion| G[Mark succeeded — report proves completion]
    F -->|Runner tuple| G2[Use recorded terminal status]
    F -->|Cancel intent| G3[Mark cancelled]
    F -->|None| H[Mark failed — orphan_finalization]
    E -->|No running or queued| I{runner_pid alive?}
    I -->|Alive| DONE4[Skip — process still running]
    I -->|Dead or not found| J{Completion/cancel/runner tuple?}
    J -->|Yes| K[Use resolved terminal tuple]
    J -->|No| L[Mark failed — orphan_run]
```

PID reuse guard: the runner records `runner_pid` and `runner_created_at_epoch`. If `psutil` finds a process with that PID but a different birth time, it is a different process — treat the original runner as dead.

`has_durable_report_completion(report_text)` returns True for non-empty report that is not a terminal control frame (`cancelled`/`error` JSON). Used by both reconciliation paths and runner-side terminal resolution to determine if a report artifact proves success.

## Work Item Store

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

- [system-overview.md](system-overview.md) — where state fits in the overall architecture
- [../concepts/state-model.md](../concepts/state-model.md) — mental model for dual-root and event sourcing
