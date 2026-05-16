# Decisions: State Layer

State-layer decisions cover how Meridian stores project identity, runtime state, and crash recovery data. See [state-and-launch.md](state-and-launch.md) for the split-domain map.

## State Layer

### Crash-only design: no graceful shutdown path

**Decision:** Meridian has no graceful shutdown protocol. The design assumption is that any process can be killed at any time, and the system must reach consistent state on the next startup.

**Constraints:** Graceful shutdown adds complexity (shutdown hooks, drain timeouts, cleanup responsibilities) and is unreliable in practice — SIGKILL bypasses all shutdown hooks. The alternative is "crash-only" design: instead of "we always clean up before exiting," the guarantee becomes "the next startup always reconciles correctly."

**Why this works:** The reconciliation path is exercised on every startup regardless of whether a crash occurred. This keeps the recovery path warm and tested, not a rarely-exercised special case.

**Alternatives rejected:**
- Graceful shutdown with SIGTERM handler — works unless SIGKILL is used; SIGKILL is common (OOM killer, user force-quit, container restart)
- Separate crash-recovery daemon — unnecessary; the reaper running on every read path achieves the same result without an additional process

See [principles/design-principles.md](../principles/design-principles.md) for full rationale.

---

### JSONL append-only event stores over SQLite or mutable JSON

**Decision (original):** Spawn and session state is stored as append-only JSONL event logs, not a database or mutable JSON files.

**Session state** still uses JSONL (see below). **Spawn state** was subsequently migrated to per-spawn `state.json` files for performance reasons — see the spawn-state-v2 decision below. The original JSONL rationale remains active for session state.

**Constraints that drove the original JSONL design:**
1. **Crash tolerance** — JSONL appends are atomic at the OS level: a partial write corrupts only the partial line, not any previous lines.
2. **Inspectability** — `cat sessions.jsonl | jq` works with no tooling.
3. **Simplicity** — no service dependency, no connection management, no schema versioning in the hot path.

**Alternatives rejected:**
- SQLite — richer query capability, but introduces a service dependency (locked DB files on crash), requires schema migrations, and breaks the "files as authority" principle
- Mutable JSON per-spawn — reconsidered and adopted for spawn state in 2026-05 once the performance problem was measured (see spawn-state-v2 below)

---

### Dual-root state: repo `.meridian/` + user `~/.meridian/projects/<uuid>/`

**Decision:** State is split between a repo-local committed root (`.meridian/`) and a user-level runtime root keyed by UUID (`~/.meridian/projects/<uuid>/`). Runtime state (spawn history, session events, artifact directories) is never committed.

**Problems solved:**
- **Repo moves** — if runtime state lived in `.meridian/`, renaming or moving the repo would orphan all prior spawn history. UUID keying means history survives repo renames.
- **Privacy** — spawn history, report content, and conversation artifacts are private to the developer. Committing them would leak into shared history.
- **Shareability** — committed scaffolding (KB, work directories) benefits from being in the repo. Runtime state doesn't benefit from sharing.

**Alternatives rejected:**
- All state in `.meridian/` (original design) — broke on repo renames, committed private runtime data
- All state in `~/.meridian/` — lost the committable scaffolding advantage for KB and work directories

---

### UUID generation skipped for read-only commands

**Decision:** State bootstrap (UUID creation + runtime directory setup) is skipped for read-only commands. `resolve_project_runtime_root()` never creates a UUID; only `resolve_project_runtime_root_for_write()` does.

**Why:** Without this, commands like `meridian spawn list` in an untouched CI checkout would create a `.meridian/id` file and `~/.meridian/projects/<uuid>/` directory — a surprising side effect from a read command that pollutes CI environments and makes the first `meridian` invocation in a new checkout unexpectedly stateful.

---

### `mark_finalizing` CAS enables `orphan_finalization` distinction

**Decision:** After the harness process exits, `streaming_runner.py` atomically transitions the spawn from `running` to `finalizing` (compare-and-swap) before beginning drain and report extraction. The reaper uses this state to distinguish `orphan_finalization` from `orphan_run`.

**Why:** Without the distinction, all crashes look the same to the reaper. `orphan_finalization` (harness completed, output likely present, only metadata persistence failed) is much less severe than `orphan_run` (harness died mid-execution). The distinction enables better diagnostics and recovery guidance.

---

### Terminal write policy as a pure function (terminal_policy.py)

**Decision (2026-05, spawn-finalization-refactor):** The finalization authority lattice is encoded as a pure function `decide_terminal_write(current_status, current_terminal_origin, incoming_origin)` in `state/spawn/terminal_policy.py`. The store calls this function under flock; the reducer calls it during event projection.

**Why:** The authority lattice had previously been duplicated between the store write path and the reducer projection path — two copies that could drift. Extracting it as a pure function makes the lattice a single testable artifact. It can be unit-tested without filesystem setup, making the full decision space verifiable (append / replace / reject × all status + origin combinations).

**Alternative rejected:** Inline conditionals in both the store and reducer — resulted in the two paths drifting and the projection authority rule being impossible to audit as a whole.

---

### CR1 fix: complete_spawn() always delegates to store, never short-circuits

**Decision (2026-05, spawn-finalization-refactor):** `SpawnApplicationService.complete_spawn()` no longer short-circuits when a pre-lock snapshot shows the spawn already terminal. It always delegates to `lifecycle.finalize()`, which calls `spawn_store.finalize_spawn()` under flock.

**Why:** The original short-circuit check (`if is_terminal: return early`) was a non-atomic read-then-skip. A concurrent writer could have appended a reconciler event between the read and the early return, allowing the caller to believe the spawn was already-terminal-by-authoritative when it was actually only-terminal-by-reconciler. The fix makes the store the sole decision authority: the read, decision, and write are all under flock.

**Effect:** Callers now receive a `CompleteSpawnOutcome` that accurately reflects whether _this_ call wrote the terminal event, whether it replaced a prior reconciler event, or whether a prior authoritative event already existed. The `snapshot` field carries post-write state regardless of outcome.

---

### Terminal arbitration extracted to a single module (terminal_arbitrator.py)

**Decision (2026-05, spawn-finalization-refactor):** Terminal trigger arbitration for streaming spawns is extracted from `streaming_runner.py` into `launch/streaming/terminal_arbitrator.py`. Both `run_streaming_spawn()` and `_run_streaming_attempt()` delegate to `arbitrate_terminal()`.

**Why:** Both functions previously contained near-identical `asyncio.wait()` races with slightly different priority orderings, making it easy for one path to be updated without the other. A single `arbitrate_terminal()` with documented priority order (terminal frame > budget > timeout > watchdog > completion-with-grace > signal) is the authoritative implementation. The 0.5-second completion grace — brief wait for a late terminal frame after process exit — is now a single named constant rather than an implicit behavior.

---

### StreamingRunConclusion replaces mutable sentinel locals

**Decision (2026-05, spawn-finalization-refactor):** Six mutable local variables that accumulated execution outcome across retry attempts in `execute_with_streaming()` are replaced by a `StreamingRunConclusion` dataclass with `absorb_attempt()` and `resolve_terminal_state()` methods.

**Why:** Scattered sentinels were error-prone across multiple code paths (retry loops, exception handlers, finally blocks). `resolve_terminal_state()` centralizes the "what terminal state does this outcome represent?" logic that was previously duplicated across multiple conditionals and hard to audit as a whole.

---

### PreparedExecutionHandoff with ExitStack ownership transfer

**Decision (2026-05, spawn-finalization-refactor):** `_prepare_execution_handoff()` in `execute.py` uses an ExitStack ownership transfer pattern: a local stack enters all context managers; on success, ownership is transferred to the returned `PreparedExecutionHandoff`; on failure, the local stack is closed before re-raising.

**Why:** Session scope (created in `_session_execution_context()`) must be cleaned up whether execution succeeds or fails, but the cleanup must happen in the caller's scope, not at preparation time. The transfer pattern guarantees no session scope leak on any failure path while ensuring the caller retains cleanup ownership during execution.

**Pattern summary:**
```python
local_stack = ExitStack()
try:
    local_stack.enter_context(...)
    handoff_stack = local_stack
    local_stack = ExitStack()   # disarm — caller owns cleanup
    return PreparedExecutionHandoff(session_exit_stack=handoff_stack, ...)
except Exception:
    local_stack.close()         # always cleans up on failure
    raise
```

---

### failure_policy.py consolidates launch-failure finalization sites

**Decision (2026-05, spawn-finalization-refactor):** All launch-failure finalization routes through `ops/spawn/failure_policy.py`, which enforces a fixed terminal tuple: `status="failed"`, `exit_code=1`, `origin="launch_failure"`.

**Why:** Eight scattered call sites previously each chose these values independently. Some used different exit codes for the same failure class, making diagnostics inconsistent. A single module with a fixed tuple ensures all launch failures look the same in spawn history, are identifiable by origin, and are auditable at one location.

---

### Spawn state v2: per-spawn state.json over global JSONL (2026-05)

**Decision:** Spawn state migrated from a single global `spawns.jsonl` event log to individual `spawns/<id>/state.json` files. Each file holds the full current state of one spawn; no event replay needed.

**What drove the migration:** Production `spawns.jsonl` had grown to 189 MB / 35,000 events. Every spawn-status read required replaying the entire log — O(n) in total project history. Primary launch time had degraded to 12–13 seconds. Per-spawn `state.json` makes reads O(1) — one file read per spawn.

**Performance results:** Primary launch time dropped from 12–13s to 0.67s.

**Two-tier write model:**

- *Tier 1 (owner writes):* The spawn's own runner writes `state.json` directly via atomic tmp+rename — no per-spawn lock. The runner is the sole writer during active execution.
- *Tier 2 (external writes):* Other processes (reaper, cancel, `update_spawn()` callers) acquire `spawns/<id>/state.lock`, read the current state, apply a mutator, write atomically. `update_spawn(claude_config_dir=...)` used by session-bleed-isolation is always tier-2 because any process may call it.

**Migration design (lazy, one-shot, no quiescence gate):**

`ensure_v2_format()` in `state/spawn/migration.py` migrates on first access. A `spawns/v2-format.json` marker file signals completion. Multiple processes race safely via `migration.lock` — the second process reads the marker and skips.

**Quiescence gate rejected:** The migration does not wait for running spawns to finish. This was deliberate: users always have running spawns, so a quiescence gate would never trigger in practice. Safety net: the reconciler (reaper) handles any spawn whose runner died mid-migration by reading v2 state and finalizing via the normal path.

**Integration with spawn-finalization architecture:**
- `decide_terminal_write()` is preserved as a pure function — application point shifted from read-time (v1 reducer) to write-time (v2 `write_state_locked()` mutator). The authority lattice and `CompleteSpawnOutcome` contract are unchanged.
- Per-spawn locking in v2 eliminates the global `spawns.jsonl.flock` contention bottleneck.

**Known remaining gap:** `list_spawns()` is still ~386ms due to ~4,000 individual file reads. Per-spawn O(1) reads help individual spawn access, but listing all spawns is now O(spawns) in file reads rather than O(events). See [open-questions/future-work.md](../open-questions/future-work.md).

---

### Presentation vs storage field naming: harness-agnostic wire names over harness-specific storage names

**Decision (2026-05, quality fixes):** When a storage field has a harness-specific name for historical reasons, the corresponding wire/presentation field uses a harness-agnostic name. The mapping lives in the query layer (e.g., `detail_from_row()` in `query.py`). The storage field is not renamed to avoid migration complexity.

**Canonical example:** `SpawnRecord.claude_config_dir` (storage) maps to `SpawnDetailOutput.session_config_dir` (wire). The storage name predates harness-agnostic design; the wire name uses the correct abstraction level.

**Why not rename storage too:** Renaming a storage field requires a migration for all existing `state.json` files. The benefit — a consistent name in one internal layer — is low; the cost — a one-time migration touching every spawn record — is real. The mapping in the query layer is cheap and the inconsistency is local.

**Rule:** Storage field names may be harness-specific for historical reasons; wire/presentation field names must be harness-agnostic. When they diverge, map in the query layer, never in the model or the storage layer. Don't rename the storage field to cosmetically match — it is not worth the migration.

---

## Related

- [../architecture/spawn-finalization.md](../architecture/spawn-finalization.md) — full subsystem architecture for the 2026-05 refactor
- [../architecture/state-system.md](../architecture/state-system.md) — state-system mechanism
- [../concepts/state-model.md](../concepts/state-model.md) — state mental model
- [state-and-launch.md](state-and-launch.md) — compatibility map for the previous combined decision page
