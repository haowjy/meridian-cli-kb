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

### Project identity in `meridian.toml`; no repo-local state (Decision 3, 2026-07)

**Decision:** The committed, machine-managed `[project] id` in `meridian.toml` is the precedence-exempt project identity. A directory is a project when it has `meridian.toml` or `mars.toml`; zero-TOML directories remain runnable. Read-only commands create nothing, while the first durable write creates identity atomically.

The ID keys `~/.meridian/projects/<id>/` for runtime and `~/.meridian/context/<id>/` for default context. No state, lock, cache, generated `.gitignore`, or fallback context directory lives under repo-local `.meridian/`. Clones and worktrees retain shared history because identity remains committed and immutable.

**Alternatives rejected:** repo-local identity/state created a second root; local-only identity split clones and worktrees; path or VCS-derived identity broke moves or the no-VCS contract; precedence overrides could silently split one project across state roots.

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

### `_directory_env_scope` eliminates `MERIDIAN_DIRECTORY_EXPLICIT` {#directory-env-scope-eliminates-meridian_directory_explicit}

**Decision (2026-05, global-directory-flag):** When the `-C` / `--directory` flag is active, `main.py` directly unsets `MERIDIAN_RUNTIME_DIR` via a `_directory_env_scope` context manager, rather than introducing a signal env var (`MERIDIAN_DIRECTORY_EXPLICIT`) that the resolution layer would check to skip `MERIDIAN_RUNTIME_DIR`.

**Alternative eliminated:** A `MERIDIAN_DIRECTORY_EXPLICIT` flag that the resolution layer would check on every project-root read. Problems:
- Two env vars to coordinate (`MERIDIAN_PROJECT_DIR` + the signal flag).
- `MERIDIAN_DIRECTORY_EXPLICIT` itself would need cleanup propagation logic identical to what it was trying to avoid.
- The coalesce function reading both vars would grow complexity with each new override env var added.

**Approach chosen:** `_directory_env_scope` context manager in `main.py`. On entry:
- Sets `MERIDIAN_PROJECT_DIR` to the `-C` path.
- If `-C` is active, unsets `MERIDIAN_RUNTIME_DIR` entirely.

On exit: restores the original values of both env vars.

**Governing principle:** "If you can remove the value that would be misread, you don't need a flag saying 'ignore that value'." The signal var, its coalesce function, and all cleanup sites disappear. Generalizes: before adding a "skip/ignore" flag, ask whether the skippable value can simply be absent.

See [../concepts/state-model.md](../concepts/state-model.md#meridian_runtime_dir-override) for the runtime-root derivation context and [../concepts/config-precedence.md](../concepts/config-precedence.md#the--c----directory-flag) for how `-C` works at the CLI boundary.

---

### D-WorkScope: Explicit WorkScope model over implicit path-or-None (PR #328, 2026-06)

**Decision:** Work scope is modeled as an explicit `WorkScope` frozen dataclass with two kinds — `work_item` (named, durable, repo scratch dir) and `ambient_spawn` (ephemeral, child spawn dir). Resolution is explicit (`resolve_bound_work_scope()` at bind time, `resolve_work_scope_from_parts()` from already-derived context). The raw-`Path` overload (fake-scope fabrication from a bare directory) is removed.

**Why:** Before this, work scope was implicit — a `Path | None` that could be a named work item directory, an ambient spawn directory, or a bare path with no semantic tag. Code that needed to know *what kind* of scope it was had to infer from directory structure or env vars. The explicit model makes:
- Leave-scope warnings properly worded by kind ("left work item directory" vs "left spawn working directory")
- Dashboard grouping by kind unambiguous
- Artifact counting bounded (excluded top-level names per kind)
- Bind-time scope resolution a single function call, not scattered path-construction

**WorkScope is a pure value object:** Warning wording and display logic live in the ops layer, not in `WorkScope` itself. The state layer provides the canonical scope facts; the ops layer decides what to say.

**Session work-switch precedence fix (same PR):** A same-process `work start`/`switch` sets `RuntimeContext.work_id` — this takes precedence over the launch-bound `MERIDIAN_ACTIVE_WORK_DIR`. The `work_id` provenance gates which source is trusted: an in-session active work_id wins over a bound-dir that may be stale. Child-spawn binding is preserved.

See [concepts/../codebase/work-items.md](../codebase/work-items.md#workscope-named-vs-ambient).

---

### Concurrency by construction: mutate-under-lock seams over convention-enforced write tiers (PR #422, 2026-07)

**Decision:** The prior two-tier write model (owner writes without lock / external writes with lock) was collapsed into a single locked mutation seam: every mutation of published state calls a `write_state_locked`-shaped function that acquires a stable lock, re-reads current state, applies a pure mutator, and writes atomically. The public unlocked `write_state()` was deleted. This applies across all stores: spawn records, archived spawns, work items, hook intervals, scope projections, and autosync transactions.

**Why:** The two-tier split was the root cause of every lost-update bug the thermo-nuclear audit reproduced. The "convention" that owner-writes skip the lock was unenforceable: any process calling the public `write_state()` could race the runner's unlocked write path, and five stores instantiated the same absent abstraction independently. Collapsing to one locked seam makes the lost-update shape structurally unwritable.

**Seam shape as a durable contract (user decision):** these seams are behavior-preserving contracts that a planned future store rewrite (#423 typed-state, #376 store scaling) inherits. The shape (lock-acquire, re-read, pure-mutate, atomic-write) is the invariant; the implementation details may change.

---

### Stable lock inodes with validated-exclusive GC seam (PR #422 + PR #444, 2026-07)

**Decision (PR #422):** All coordination locks live under `locks/<domain>/` (e.g. `locks/spawns/`, `locks/process-scopes/`, `locks/reaper-cleanup/`, `locks/launch-boundary/`, `locks/hooks/`) outside the directories they protect.

**Why:** POSIX `flock` is per-open-file-description, not per-path. If one process unlinks a lock file and creates a new one while another holds the old inode, both processes believe they hold "the lock" on different inodes — split-brain. The audit reproduced this failure.

**Amendment (PR #444, closes #427):** Lock inodes are never unlinked except through `unlink_validated_lock()`, which unlinks only while holding a fresh, non-reentrant exclusive flock whose `fstat` matches the current `stat` of the path, immediately before release. The existing acquire-side revalidation loop makes this provably split-brain-free: a blocked acquirer that finds the inode gone (or changed) retries onto the recreated inode.

Two call sites use this primitive:
- `lock_gc.py::gc_orphaned_locks()` sweeps orphaned per-spawn locks (four classes) when the corresponding spawn directory no longer exists. Wired into doctor background repairs and post-prune. A non-blocking meta-lock (`locks/gc.lock`) serializes sweeps for churn control; safety is per-inode, not meta-lock dependent.
- `cleanup_stale_sessions()` unlinks cleaned session locks before release, bounding the session-lock rescan with zero new state.

**The forbidden pattern:** unlinking while the lock remains held afterwards. Concrete example: `delete_published_spawn` uses `reentrant=True`; an inline unlink would strand the outer frame on an orphaned inode while a new acquirer validates against a recreated path — split-brain. No unlink at ordinary release time, no unlink inside reentrant contexts.

**Striping rejected (PR #444 design doc):** A bounded lock-stripe namespace (hash spawn IDs onto a fixed set of lock files) was evaluated and rejected on three grounds: (1) reentrant holds would silently lose exclusion between stripe-colliding spawns; (2) session locks are lifetime-held liveness beacons whose acquirability is the staleness signal — striping collapses this; (3) larger surface than the validated-EX sweep for the same correctness goal. Other rejected alternatives: opportunistic unlink at release time (forbidden pattern above), unlink without holding (meta-lock alone does not exclude holders), epoch/generation lock directories (moves the problem without solving it).

**Alternatives rejected (PR #422, original decision):**
- Unlink-while-holding: still allows a third process to create a new inode after unlink, producing split-brain between holder and newcomer.
- Revision-CAS without re-read-and-reapply: requires the caller to hold the complete previous state, which is fragile across process boundaries and doesn't compose with pure mutators.

---

### Project-lifetime shared/exclusive gate (PR #422, 2026-07)

**Decision:** `~/.meridian/projects/.locks/<uuid>.lock` is a project-lifetime coordination lock outside the deletable project root. Sessions hold a shared lock for their lifetime; global pruning acquires exclusive + revalidates before removal.

**Why:** The audit's gate-1 review reproduced global pruning destroying a runtime root while sessions held spawn locks inside it. The shared/exclusive gate prevents this without requiring pruning to enumerate all active sessions.

---

### Permission policy split: preserve-mode for user files, strict 0600 for runtime state (PR #422, 2026-07)

**Decision:** `atomic_replace()` in `lib/platform/atomic.py` accepts `permissions="preserve"` (default, keeps existing file mode) or `permissions=0o600` (strict mode for runtime state). State-facing writes enforce 0600; user-owned project files and context work-item metadata preserve the original mode.

**Why:** The initial atomic-replace primitive unconditionally set 0600, which flipped user-owned files (like `mars.toml`) to unreadable modes. The split policy was driven by the gate-2 review reproducing Codex rollout permission broadening.

---

### AST conformance guard over ruff TID251 for raw-write ban (PR #422, 2026-07)

**Decision:** Raw writes to authoritative state are rejected by `tests/contract/test_state_write_conformance.py`, a repo-wide AST test with a documented single-entry allowlist. Stale allowlist entries are detected. The failure message names the offending call site and guides toward the correct primitive.

**Why ruff TID251 was rejected:** TID251 matches import paths, not method calls on inferred types. It cannot flag `some_path.write_text()` because it does not know `some_path` is a `Path`. The AST-based test walks call nodes and matches method names against a banned set, which is the correct granularity.

---

### Autosync AGENTS.md notice removal (PR #422, 2026-07)

**Decision:** Per-conflict rewrites of user-owned AGENTS.md at the sync root were removed. Conflict JSON (`<sync-root>/.meridian/autosync/conflicts/<id>.json`) is the durable signal.

**Why:** No agent ingestion contract existed to consume the AGENTS.md notices. The notices were committed locally and never propagated while behind origin (by design), but they modified user-owned files without a consuming contract — a write without a reader. The conflict JSON was already the authoritative record that CLI and dashboard read from.

---

### Conftest git-config guard deletion (PR #422, 2026-07)

**Decision:** The session-wide git-config snapshot/restore guard in `tests/conftest.py` was deleted.

**Why:** Investigation (spawn p5328) proved no test writes the shared repo-local `.git/config`. The guard falsely attributed writes by VS Code's Git extension (which writes `branch.*.vscode-merge-base` to the shared common config on worktree creation) and restored stale bytes — itself an unlocked read-modify-write on an unowned file. The actual test isolation already lives at the correct ownership boundary: each test strips inherited `GIT_*` state and points global config to a temp file.

---

### Torn-tail-proof JSONL appends via parse-before-discard and atomic inode replacement (PR #444, 2026-07)

**Decision:** All durable JSONL consumers (session events, launch-boundary events, permission journals, control-action journals) use `append_durable_jsonl_line`, which repairs any torn tail before appending. A complete row missing only its trailing newline is preserved (readers already accepted it); a genuinely torn partial row is dropped. Repair writes the corrected content via atomic inode replacement (`atomic_replace` with `permissions="preserve"`), so unlocked buffered readers can never splice a fabricated hybrid event from a partial overwrite.

**Why:** A SIGKILL mid-append leaves a partial last line. The next append previously concatenated its content onto the torn tail, making both the torn row and the new row unreadable. For session events, this made the session permanently unstopped after its lock was unlinked (the stop event was the new row). For permission/control journals, torn tails allowed `transition_seq` reuse.

**Design choices:**
- *Parse-before-discard:* the tail is parsed as JSON before deciding whether to keep or drop it. A complete object row missing only `\n` is common (valid write, missing only the delimiter) and must not be discarded.
- *Atomic inode replacement over in-place truncation:* `os.truncate` would mutate the file in place; an unlocked reader with a buffered file descriptor could read bytes from the old content followed by bytes from a new append, producing a fabricated hybrid event. Atomic replacement creates a new inode, so the reader sees either the old file (complete) or the new file (repaired), never a splice.
- *O(1) fast path:* the clean-tail check reads only the last byte (`seek(-1, SEEK_END)`); no whole-file read per append when the tail is intact.
- `history.jsonl` is deliberately excluded; its repair is tracked under #376.

---

### Spawn deletion composition owner: spawn_aggregate.py (PR #444, 2026-07)

**Decision:** `delete_published_spawn()` and its precondition type live in `spawn_aggregate.py`, which composes the spawn repository and process-scope projection persistence leaves. The dependency is one-way: `spawn_aggregate` imports from both leaves; neither leaf imports the other or the aggregate.

**Why:** `spawn_store.py` had grown to 988 lines by hosting both per-spawn mutation and cross-leaf deletion. The thermo-nuclear consistency review identified the extraction as a structural fix: the deletion seam composes two leaves (spawn lock then projection lock) and should not live inside either leaf.

---

### Published-row lifetime owns spawn artifacts (issue #437, 2026-07)

**Decision:** A spawn-owned artifact may be created or changed only while that
spawn has a published `state.json` row. Parent-creating or otherwise late
mutations use `mutate_published_spawn_artifact()` in `spawn_aggregate.py`. The
seam acquires the stable external per-spawn lock, re-reads the row while locked,
optionally checks a current-row predicate, and runs the artifact mutation only
if the row is still eligible. `delete_published_spawn()` uses the same outer
lock, so deletion and a late artifact write have a total order.

Simple writers that do not need to create parents, such as heartbeats, may fail
closed on `FileNotFoundError` instead of taking the aggregate seam. Connection
startup and child-process adoption treat the published spawn directory as a
precondition; they never manufacture it. Late diagnostics are best-effort and
may be dropped rather than outlive their authoritative row.

**Why:** The spawn directory was both the aggregate being deleted and the place
where low-level helpers recreated parents. Retention could remove a terminal
spawn, then an awaited callback could append a journal, write metadata, or
touch a heartbeat and reconstruct `spawns/<id>/` without `state.json`. The
result was a ghost directory: invisible to authoritative listing, confusing to
cleanup, and able to delay external lock GC. Checking existence before the
write was insufficient because deletion could win between the check and the
mutation.

**Rejected:**

- Guard only the launch-boundary writer. Harness callbacks, history drains,
  signals, scope registration, metadata, reaper evidence, and other
  asynchronous writers share the same lifetime race.
- Teach atomic-write and JSONL helpers about spawn publication. Those helpers
  are deliberately domain-blind and serve state outside the spawn aggregate.
- Preserve direct connection startup without a published row by recreating the
  directory. Missing publication is a lifecycle violation, not a compatibility
  case.

The resulting lock order keeps the per-spawn lifetime lock outside specialized
artifact locks: spawn lock before launch-boundary append lock, and spawn lock
before process-scope projection lock.

Provenance: `work:launch-boundary-resurrection`, issue #437.

---

### Mutate-under-lock seams stay store-specific; generic mutation framework rejected (PR #444, 2026-07)

**Decision:** The six mutate-under-lock seams (spawn records, archived spawns, work items, hook intervals, scope projections, autosync) stay store-specific. A generic mutation framework (e.g. a `LockedStore[T]` base class or protocol) was evaluated in the thermo-nuclear consistency review and rejected.

**Why:** The seams share a shape (lock, re-read, pure-mutate, atomic-write) but diverge in lock-path computation, read/write codecs, file layout, and error handling. A generic abstraction would need enough parameters to reconstruct each store's specifics, providing the cost of an abstraction without reducing the decision space. The shared mechanism is already `lock_file` + the atomic-write canon; the seam shape is a documented behavioral contract, not a code-level inheritance hierarchy.

---

## Typed State Contracts (PR #423, 2026-07)

Six decisions made invalid states unparseable or unconstructible, eliminating bug classes that existed only because the types permitted them.

### Single `SpawnStatus` StrEnum authority with member-derived lifecycle sets

**Decision:** `SpawnStatus` is a `StrEnum` in `core/domain.py`. Lifecycle sets (`ALL_SPAWN_STATUSES`, `ACTIVE_SPAWN_STATUSES`, `TERMINAL_SPAWN_STATUSES`) are derived from a `dict[SpawnStatus, SpawnLifecycleClass]` map. `TerminalSpawnStatus` is a `Literal` alias verified against the terminal set at import time.

**Rejected:** Positional ordering with magic count (the sets were previously hand-curated frozensets of strings whose membership could drift from the Literal duplicate). Keeping the Literal as an independent authority (it still exists for static type-checking, but the import-time guard ties it to the StrEnum).

---

### Quarantine, not coercion, for out-of-vocab persisted rows

**Decision:** `StoredSpawnState` validates vocabulary fields in a `model_validator(mode="before")`. Rows with unknown `status`, `kind`, `launch_mode`, or nested fact vocabularies are quarantined: single reads raise `SpawnStateQuarantined`; collection reads partition into `SpawnCollection` (valid rows + quarantine reports). Migration and retention fail closed on quarantine — an unreadable row may contain active work.

**Rejected:** Silently omitting unknown rows (loses data). Coercing to `failed` (misrepresents a parse failure as a lifecycle outcome). The quarantine seam validates type before string operations so a non-string value routes to quarantine rather than crashing with `AttributeError`.

---

### Enforced-equivalence discriminant for terminal lifecycle facts

**Decision:** Runner-exit evidence and finalized-outcome evidence are frozen sub-models (`RunnerExitFacts`, `TerminalFacts`) nested under `runner_exit` and `terminal`. A model validator enforces `status == terminal.status` when terminal, and `terminal is None` when active. Stale flat lifecycle fields are rejected at parse.

`_RevalidatedFrozenModel.model_copy(update=)` round-trips through `model_validate` instead of Pydantic's default shallow copy, closing the escape hatch where `model_copy` would bypass validators.

**Rejected:** Sole-carrier (a single discriminated union carrying both the status and the facts). The model-copy revalidation approach was chosen after discovering that frozen-model immutability alone did NOT close the copy escape hatch — Pydantic's `model_copy` skips validators by default.

---

### Apply/Decline typed mutation outcome

**Decision:** Locked mutators return `SpawnRecord | Decline(reason)`. `write_state_locked()` returns `MutationOutcome(wrote, snapshot, reason)`. The lifecycle layer adds `transitioned` for status-change awareness.

**Rejected:** Four ad-hoc exception classes and two `nonlocal` smuggles that encoded mutation outcomes as control flow. The typed outcome makes the wrote/declined distinction part of the return type, not exception handling.

---

### Open `RawHarnessEvent` envelope to closed `SemanticEvent` union

**Decision:** Raw harness events stay open (unknown event types pass through). Each adapter's `HarnessBundle` registers a `HarnessSemantics` port with an event-name→`SemanticClass` table and optional payload resolver. `HarnessSemantics.normalize()` dispatches by `HarnessId` before `event_type`, producing a closed `SemanticEvent` union. Shared `semantics.py` contains no harness event names.

**Rejected:** Strict rejection at raw parsing — harness CLIs are unpinned; new event types arrive across minor releases. The open envelope / closed semantic union preserves forward compatibility without ignoring classification responsibility.

---

### Directory location as sole archived-ness authority for work items

**Decision:** A work item's `archived_at` timestamp is stored but never decides whether the item is archived. Directory location (`work/<slug>/` vs `archive/work/<slug>/`) is the sole authority. Archive and reopen operations move the directory first; `__status.json` is updated inside the moved directory. `StoredWorkItemState` is the typed codec for `__status.json`; work-item `status` is an open string vocabulary (not a closed enum) because custom labels exist.

**Rejected:** Closed `WorkStatus` enum — would delete custom labels. Relocating the reconciliation heuristic (the 117-line heuristic that previously reconciled `archived_at` with directory location was deleted).

---

## Related

- [../architecture/spawn-finalization.md](../architecture/spawn-finalization.md) — full subsystem architecture for the 2026-05 refactor and typed state contracts
- [../architecture/state-system.md](../architecture/state-system.md) — state-system mechanism
- [../concepts/state-model.md](../concepts/state-model.md) — state mental model
- [state-and-launch.md](state-and-launch.md) — compatibility map for the previous combined decision page
