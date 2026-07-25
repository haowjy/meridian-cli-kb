# Decisions: Launch Process Ownership

These decisions govern managed-primary lifetime, durable process scope,
reconciliation, cancellation, and cleanup.

## Decision Map

| Concern | Status | Canonical mechanism |
|---|---|---|
| Managed-primary passive reaping | Current | [Managed-primary lifecycle](../architecture/managed-primary-lifecycle.md) |
| Process-scope ownership and containment | Current | [Process scope](../architecture/process-scope.md) |
| Codex interactive approval routing | Current | [Launch architecture](../architecture/launch-system.md) |

### D-managed-primary-reaper: Managed-primary passive reaper safety

*2026-05-06; refined 2026-05 PR #184*

**Decided:** Passive reconciliation may finalize Codex/OpenCode managed
primaries as `failed/orphan_primary`. When it finalizes (not skips), it also
terminates tracked runtime children as a cleanup safety net via
`terminate_managed_primary_processes()`. Runtime children are NOT terminated on
Skip decisions — those mean the spawn may still be active.

**Why:** The generic reaper assumption is "dead runner means dead harness." That
is true for child spawns and Claude's single black-box process, but false for
managed Codex/OpenCode primaries. Their launcher wrapper, backend, and TUI are
separate processes. A dead launcher is abnormal evidence about Meridian's
wrapper, not proof that the TUI/backend are gone.

**The skip/finalize distinction is load-bearing:**
- **Skip**: reaper cannot prove the spawn is dead (recent activity, alive runner).
  No signals — the TUI may still be usable.
- **Finalize as failed**: reaper has proven the spawn is orphaned. Leaving
  backend/TUI processes running causes accumulation. PR #184 added the safety
  net: finalize decision → also clean up runtime children with PID-reuse guards.

**Damage prevented before this boundary:** Read surfaces (`meridian work`, `spawn
show`, `spawn list`, `spawn wait`, `session log`) run reconciliation. Before the
original fix, a user inspecting state could cause the reaper to terminate a
still-running backend/TUI. The skip-vs-finalize distinction preserves
non-destructive behavior for active sessions.

**Conservative metadata rule:** Missing or corrupt `primary_meta.json` for a
`kind="primary"` Codex/OpenCode spawn is treated as a managed-primary candidate.
It records `orphan_primary` but does NOT invoke the safety-net termination
(cannot identify PIDs without metadata). This prevents metadata corruption from
becoming permission to kill unknown processes.

**Explicit cleanup paths:**
- `spawn cancel` terminates tracked managed-primary processes on request.
- `meridian doctor --kill-orphans` triggers the same managed-process cleanup
  for orphaned spawns identified during that doctor run.

**Alternatives rejected:**
- Keep passive child termination for all decisions including Skip — violates the
  expectation that dashboards and query commands are safe.
- Add a new daemon to own cleanup — increases owned complexity.
- Treat missing metadata as generic `orphan_run` — unsafe for managed-primary
  candidates because the topology cannot be proven.

See [architecture/managed-primary-lifecycle.md](../architecture/managed-primary-lifecycle.md)
and [lessons/arch-refactor-pitfalls.md](../lessons/arch-refactor-pitfalls.md).

---

### D-process-scope-ownership: Durable process-scope ownership over POSIX-only or psutil-only cleanup

*2026-05 PR #184*

**Decided:** Adopt a shared process containment layer (Option D) with OS-specific
mechanisms, durable ownership metadata, and lease-aware cleanup policy. Rejected:
Option A (POSIX process-group only), Option B (psutil tree kill as primary), Option C
(harness-specific cleanup).

**Why Option A was insufficient:** `CREATE_NEW_PROCESS_GROUP` on Windows is for
console signal routing, not tree reclamation. Child tools can still survive root
termination. The model also cannot express managed-primary preservation.

**Why Option B was insufficient as primary:** psutil tree snapshots are race-prone —
if an intermediate exits before the snapshot, deeper descendants disappear from the
tree. No positive containment boundary; crash recovery depends on rediscovery after
the fact.

**Why Option C was rejected:** Duplicates logic across Claude/Codex/OpenCode adapters;
misses generic tool-child leaks launched outside the harness; violates policy/mechanism
separation. Reaper remains inconsistent.

**Why Option D:** Gives normal teardown and the reaper the same mechanism seam.
Durable ownership metadata turns managed-primary preservation from an implicit
exception into an explicit policy class (`session_owned` vs `spawn_owned`). Allows
adding a new OS platform as one adapter file without editing policy.

**Options A and B are kept as sub-components:**
- POSIX group kill is the primary POSIX mechanism inside the scope adapter.
- psutil tree kill is the degraded fallback when containment setup fails or scope
  metadata is absent (legacy spawns, corrupt records, Job Object assignment denied).

**The three-layer split:**
- `lib/platform/process_scope/` — mechanism (how to kill a tree; four sub-modules:
  base, posix, windows_job, fallback)
- `lib/core/process_cleanup.py` — policy (whether to kill; consults spawn record,
  session leases, managed-primary metadata)
- `lib/state/process_scope_projection.py` — persistence (scope ownership events
  through the existing spawn lifecycle state stream)

**Key implementation constraints discovered:**
- Scope snapshots must be persisted **before** `connection.stop()` is called. A crash
  between `stop()` and snapshot persistence would leave no scope record.
- `terminate()` must be exception-safe — cleanup must not throw.
- PID-reuse guards are required: compare `root_created_at_epoch` before any signal.

**Known follow-up gaps:** PROC-004 (dead wrapper / live reparented child surviving
snapshot) and PROC-007 (metadata-only lifecycle spawns not yet covered by scope
projection). See [open-questions/process-scope.md](../open-questions/process-scope.md).

See [architecture/process-scope.md](../architecture/process-scope.md) for mechanism,
module boundary, and platform adapter detail.

---

### D-reaper-escape-fix: Reaper and cancel paths rewired to use scope containment

*2026-05 — commits f5405a80, 82f871f5, 1c4c915f, 1e0c5020*

**The bug:** Sync cleanup paths (reaper, cancel, session-exit) ignored scope containment
metadata and fell through to PID-tree fallback (`terminate_tree_sync` on `worker_pid`).
PID-tree fallback cannot find processes that reparented to PID 1 after their parent
harness exited. Result: Vite dev servers, Codex app-server instances, and tool workers
survived as orphans after session exit or cancel. Session-exit scope reclamation
(`reclaim_session_owned_scopes_for_chat`) was also dead code — written but never called.

**Fix:** All three sync cleanup paths now route through `terminate_scope_sync()` when
scope metadata is available. Legacy spawns fall back to PID-tree termination. The
session-exit path was wired to its already-existing `reclaim_session_owned_scopes_for_chat()`.

**Six implementation decisions in this fix:**

#### D-REF-1: `terminate_scope_sync()` in platform layer, not core layer

`terminate_scope_sync()` is a public function in `platform/process_scope/__init__.py`,
not a private helper in `core/process_cleanup.py`.

**Why:** Single canonical sync dispatch — reaper, cancel, and session-exit all route
through one place. Mirrors the async `ScopedProcessHandle.terminate()`. Placing it in
`platform/` keeps mechanism co-located with the adapters it dispatches to. Placing it
in `core/` would invert the dependency direction (platform is below core; callers
importing from core to reach platform logic muddies the layer boundary).

#### D-REF-2: `degraded_fallback` flag semantics

`CleanupResult.degraded_fallback` is set as: `scope.containment != "pid_tree_fallback"`.

This means:
- Scope says `pid_tree_fallback` and PID-tree termination runs → `degraded_fallback=False` (expected path for this scope)
- Scope says `posix_pgid` but falls through to PID-tree → `degraded_fallback=True` (unexpected)

The same logic applies in both async (`ScopedProcessHandle.terminate`) and sync
(`terminate_scope_sync`) paths. Consistency was deliberate: the async path was already
correct; the sync path was written to match exactly.

#### D-REF-3: Reclaim by `chat_id`, not `harness_session_id`

`reclaim_session_owned_scopes_for_chat()` reclaims by `chat_id`, not `harness_session_id`.

**Why:** A single meridian session (`chat_id`) spans multiple harness sessions —
resume after compaction creates a new harness session ID. Reclaiming by
`harness_session_id` would miss scopes registered under earlier harness session IDs
in the same meridian session. `chat_id` is the full meridian session lifetime boundary.

#### D-REF-4: Session-exit reclamation in `session_scope()` finally block

`reclaim_session_owned_scopes_for_chat()` is called from the `session_scope()` finally
block, after `stop_session()`, in a **nested** try/finally so reclamation runs even if
`stop_session()` raises.

This was previously dead code — the function existed but was never called. The fix
was not a new mechanism but wiring the existing one from the right call site.

#### D-REF-5: Cancel path manages scope cleanup inline, not via `core/process_cleanup.py`

`signal_canceller.py` reads scope sidecars, calls `terminate_scope_sync()`, and marks
scopes released directly — it does not delegate to `core/process_cleanup.py`.

**Why:** `signal_canceller.py` has a live event loop and needs `asyncio.to_thread` for
blocking calls. `core/process_cleanup.py` is sync-only and called from the reaper.
Delegating from cancel to core would require either making core async-aware or
introducing a thread-context dependency. Self-contained cancel maintains the
`signal_canceller → platform/ + state/` dependency direction cleanly.

#### D-REF-6: Legacy fallback preserved in cancel path (not deprecated)

Both the legacy (`runner_pid` + `terminate_tree_sync`) and the new (scope-sidecar +
`terminate_scope_sync`) paths are preserved in the cancel path.

**Why:** Legacy spawns without scope metadata still exist during the transition period.
Corrupt or missing scope records can occur. The fallback handles these cases. It is not
deprecated — it covers the gap between old spawns and new ones until all spawns carry
scope metadata.

---

## Related

- [Launch composition decisions](launch.md)
- [Launch session decisions](launch-session-initiation.md)
