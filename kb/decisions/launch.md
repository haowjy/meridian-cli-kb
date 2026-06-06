# Decisions: Launch Pipeline

Launch decisions cover the single composition seam, driving-adapter boundaries, harness identity propagation, and spawn-wait semantics. See [state-and-launch.md](state-and-launch.md) for the split-domain map.

## Launch Pipeline

### Single composition seam: all driving adapters call `build_launch_context()`

**Decision:** All four driving adapters (primary CLI, spawn subprocess, REST app, CLI streaming-serve) call `build_launch_context()` for all composition. No driving adapter re-derives policy, permissions, prompt, argv, or environment.

**Why:** Four separate composition paths would drift over time. Security properties (permission resolution, environment sanitization) and behavioral properties (prompt assembly, workspace projection) need a single place to audit. The 13 composition invariants are checkable in one code location.

**Alternatives rejected:** Per-adapter composition was the prior approach and caused drift bugs. Shared utility functions without a single factory were considered, but utility functions don't enforce the "sole callsite" property that invariants require.

See [architecture/launch-system.md](../architecture/launch-system.md) — architecture model and driving adapters.

---

### Prepare/bind split: expensive resolution separated from cheap materialization

**Decision (2026-05, launch-primary-context-dedupe):** `build_launch_context()` is decomposed into two public phases: `prepare_launch_surface()` (expensive: model/harness/skill resolution, composition, prompt assembly) and `bind_launch_context()` (cheap: env, cwd, spec, argv, permissions given a spawn ID and report path). The existing `build_launch_context()` entry point is preserved as a backward-compat wrapper.

**Why:** The primary CLI path previously called the full factory twice — once for `--dry-run` preview, once for real execution. Each call repeated all the expensive Mars alias lookups and skill resolution. The split makes the prepare-once/bind-N pattern explicit and enforced, not accidental. Any future driver that needs preview+real or multi-variant binding gets the same efficiency for free.

**`PreparedLaunchSurface`** — promoted from the internal `_SurfaceResolution` to a public frozen dataclass. This is the in-memory contract between preparation and binding. It carries everything that is spawn-stable (resolved request, harness, composition warnings, agent inventory, context prompt) and explicitly excludes runtime-only values (spawn ID, report path, env, argv, permissions). The exclusion is the invariant — bind-phase code cannot accidentally depend on preparation-phase values and vice versa.

**`RuntimeBindings`** — frozen dataclass for the bind-phase inputs: `spawn_id`, `report_output_path`, `runtime_work_id`, `chat_id`, `forked_harness_session_id`, `plan_overrides`, `dry_run`. Naming these explicitly prevents bind-phase parameters from silently expanding over time.

**Alternatives rejected:**
- Keep calling `build_launch_context()` twice — semantically equivalent but obscures the intent and makes the expensive/cheap boundary invisible to reviewers
- Memoize inside `build_launch_context()` — requires cache invalidation policy and shared mutable state, which violates the "no derived state on DTOs" invariant
- Split only for the primary path — partial split creates two patterns to maintain; the full split allows all four driving adapters to benefit

See [architecture/launch-system.md](../architecture/launch-system.md) — Prepare/Bind Split.

---

### CatalogSession: operation-scoped Mars result cache

**Decision (2026-05, launch-primary-context-dedupe):** `CatalogSession` is an operation-scoped collaborator that holds a `MarsResultCache` and wraps the catalog resolution methods (`resolve_model`, `load_aliases`, `alias_map`, `list_all_models`). It is created once per launch operation and discarded at operation end. `prepare_launch_surface()` receives a `CatalogSession` from its caller.

**Why:** Without this, `resolve_policies()` and `prepare_launch_surface()` called `mars models list` independently on every invocation. The prepare/bind split compounds this — calling `prepare_launch_surface()` for preview + real would double-call Mars. `CatalogSession` solves the problem at the right abstraction level: the cache lifetime matches the operation, not the module or the process. This avoids global shared state (process-lifetime cache) and the "pass `MarsResultCache` explicitly through every call site" antipattern.

**Alternatives rejected:**
- Process-level `MarsResultCache` singleton — survives beyond the operation; stale if the catalog changes mid-session; wrong lifetime
- Pass `MarsResultCache` as an explicit parameter to every resolution function — the refactor showed this creates 8+ call sites that all need the extra parameter; `CatalogSession` groups them cleanly
- Remove caching entirely — Mars calls are subprocess launches (~100ms each); uncached, a single spawn involves 3–5 redundant calls

---

### D-control-root-task-cwd-split: Split execution_cwd into control_root + task_cwd

**Decision (2026-05-13, PR #210):** `LaunchContext`, spawn records, and session start events now carry two distinct path fields — `control_root` and `task_cwd` — instead of the single overloaded `execution_cwd` / `project_root` field.

- **`control_root`** — the project config/authority root. Always anchored to the directory that contains `meridian.toml`. Used for spawn log directory construction, config loading, and harness `--add-dir` workspace root injection. This is what `project_root` / `execution_cwd` meant before the split.
- **`task_cwd`** — the task's intended working directory. Non-null only when the spawn was requested from a subdirectory of the project root. `None` in the common case where task and config roots coincide.

**Why:** `execution_cwd` was overloaded — one field served double duty as both config authority and task directory. When a spawn originates from a nested directory (e.g., `packages/auth/`), these two concerns diverge: the config authority should stay at the repo root (`control_root`) while the agent needs to know it should operate in the subdirectory (`task_cwd`). A single field cannot express both truths simultaneously.

**Behavior (updated 2026-06, PR #318):** When `task_cwd` differs from `control_root`, `bind_launch_context()` communicates this to the agent in two ways:
1. `MERIDIAN_TASK_DIR` env var set to the task_cwd path in the child process environment.
2. A `# Source-edit directory` system prompt block injected stating the absolute task dir path, that the shell cwd is the project root (not the task dir), and to `cd` in or use absolute paths.

A `_is_task_cwd_covered_by_projection()` check prevents redundant workspace root additions when the task_cwd is already covered by the projected roots.

**continue/fork authority:** `ClaudeSessionAccessSource` (and the equivalent structs for other harnesses) now carries `source_control_root` and `target_control_root` instead of `source_cwd`. `resolve_session_reference()` uses `source_control_root` from persisted spawn records for continue/fork authority. Legacy records that predate this split fall back to the current launch `control_root`.

**`execution_cwd` legacy field:** Retained as an alias for the actual process cwd (`child_cwd`) in `PreparedExecutionHandoff` for backward compatibility with existing call sites that read this field. New code should use `control_root` and `task_cwd`.

**Alternatives rejected:**
- *Keep using process cwd as project root* — spawns launched from nested directories would pick up the wrong `meridian.toml`, breaking config loading and spawn log placement.
- *Keep a single "project cwd" concept* — loses the task directory intent when the two diverge; the agent would have no way to know its process cwd is not where the user intended work.
- *Pass task_cwd only as env var, not in spawn records* — loses the durable intent across continue/fork sessions; the next launch would not know what directory the task was originally scoped to.

See [architecture/launch-system.md](../architecture/launch-system.md#control_root--task_cwd-split).

**Extended by PR #248 (2026-05-22):** The two-field model was extended into a full authority/task domain architecture with worktree-based task_cwd resolution, reference_anchor for `-f` paths, `kb:` prefix, and adapter cwd policy. See [spawn-cwd-worktree-anchor.md](spawn-cwd-worktree-anchor.md).

---

### Resolve-before-persist for REST app and streaming-serve paths

**Decision:** Both the REST app path and the CLI streaming-serve path call `build_launch_context()` before creating the spawn row (`SpawnApplicationService.prepare_spawn()`). The spawn row is created only if resolution succeeds.

**Why:** The alternative — create the row first, then resolve — produces phantom active rows when resolution fails (bad model name, missing profile, permission error). Phantom rows require doctor to clean up and confuse `spawn list` output. Resolve-before-persist eliminates this class of bug: if you see a row, it has real resolved metadata.

**Known gap:** The spawn subprocess path (`ops/spawn/execute.py`) still creates the row before calling `build_launch_context()`. Unifying it to the resolve-before-persist seam remains a follow-up. The 2026-05 refactor (below) eliminated the secondary resolution problem without changing row-creation order.

---

### Background worker trusts pre-resolved `BackgroundWorkerLaunchRequest`; no inline re-resolution

**Decision:** The background worker in `ops/spawn/execute.py` removes its inline model/harness resolution fallback chain. It trusts the `BackgroundWorkerLaunchRequest` written by `execute_spawn_background()`, which is populated from `build_create_payload()` → `build_launch_context()` → `resolve_policies()`. The worker's job is execution, not re-resolution.

**Context:** The worker previously did:
```python
resolved_model = (request.model or spawn_record.model or "").strip()
```
This merged the persisted request with the spawn record — both populated from the same resolved source — creating a redundant fallback chain that hard-rejected empty model. Orphan-run spawns with model-optional profiles failed this check because neither source contained a model string.

**Why remove, not fix:** Both the spawn record and the persisted worker request are copies of the same resolved data (from `build_create_payload()`). The fallback was redundant, not a cross-source safety net. Removing it makes the data flow explicit: one source of truth (persisted request), one consumer (background worker). The foreground path already trusts `build_create_payload()` output without re-validating — the background path now matches.

**Alternatives rejected:**
1. Fix the inline resolution to accept empty model → still maintains two resolution paths, still has the redundant fallback
2. Re-run `resolve_policies()` inside the background worker → expensive and wrong-in-principle (the request is already resolved; re-running at a different time could produce different results if alias catalog changes)

---

### `pre_launch_complete` flag separates pre-launch failure domain from execution failure domain

> **Superseded by the R4-R6 layered finalization ownership refactor (2026-05).** The flag-based approach below was replaced by `launch_prepared_spawn()` as a shared helper with structural ownership. Documented here as historical rationale for the new design.

**Decision (historical):** `_execute_existing_spawn()` used a `pre_launch_complete: bool` flag (initially `False`, set to `True` just before `execute_with_streaming()` is called) to determine whether an exception in the `except` block should write a `launch_failure` terminal event or be treated as an unexpected execution exception.

**Why the flag was insufficient:** The flag approximated ownership with boolean state — it couldn't express partial-setup failures cleanly, required both foreground and background paths to duplicate the pattern independently, and the boundary between "pre-launch" and "in-runner" was implicit. The replacement uses function scope as the ownership unit.

---

### Layered finalization ownership: runner → helper → surface backstop (R4-R6 refactor)

**Decision:** Finalization responsibility is divided across three concentric function scopes, each defined by what it can observe:

1. **Runner layer** (`execute_with_streaming()`): Owns terminal finalization from function entry. Uses sentinel locals (initialized to `None`/safe defaults at top of function, set to real values during setup). The `finally` block reads these sentinels to handle partial-setup failures — if a local is still `None`, the setup step that sets it never completed. Clock calls (`resolved_clock`, `started_at`) are left **outside** the try block because `stdlib` monotonic clock calls effectively never raise.

2. **Helper layer** (`launch_prepared_spawn()`): Shared pre-run setup function called by both foreground (`execute_spawn_blocking`) and background (`_execute_existing_spawn`) paths. Owns `launch_failure` finalization for exceptions that occur **before** `execute_with_streaming()` is called. Its broad `except` is intentional — the runner's `complete_spawn()` idempotency guarantee means a double-fire from this catch is safe and produces no corruption.

3. **Surface backstop layer** (`execute_spawn_blocking`, `_execute_existing_spawn`): Last-resort catch around the entire post-row section (everything after the spawn row is created). Logs and returns exit code 1. By this point the runner or helper should have already written terminal state; the backstop ensures an unexpected exception from the helpers themselves doesn't leave the spawn permanently stuck in `queued`.

**Why structural layers over boolean flags:**
- Each layer is defined by function scope, not by a flag state that must be threaded carefully through complex setup code
- `launch_prepared_spawn()` eliminates code duplication that existed because both foreground and background paths independently re-implemented the `pre_launch_complete` pattern
- Ownership is inspectable from the call graph; flag ownership requires reading the flag-set logic

**Shape of the helper layer:**
```python
async def launch_prepared_spawn(...) -> int:
    try:
        # ... all pre-launch setup (session, fork materialization, context build) ...
        return await execute_with_streaming(spawn, ...)
    except Exception as exc:
        # pre-run failure: helper owns launch_failure finalization
        await SpawnApplicationService(runtime_root, lifecycle_service).complete_spawn(
            spawn.spawn_id, "failed", 1, origin="launch_failure", error=str(exc),
        )
        logger.exception("Pre-launch setup failed.", spawn_id=str(spawn.spawn_id))
        return 1
```

---

### Sentinel-local pattern for partial-setup failure in the runner

**Decision:** `execute_with_streaming()` initializes all result-bearing locals to `None` or safe defaults at the top of the function body, before any setup code runs. The `finally` block reads these sentinels to decide what work to do:

```python
extracted: FinalizeExtraction | None = None  # set after enrich_finalize()
lifecycle_service: SpawnLifecycleService | None = None  # set after create_lifecycle_service()
manager: SpawnManager | None = None           # set after SpawnManager()
```

If setup fails partway through, whichever locals are still `None` are skipped in cleanup. This handles partial-setup failures gracefully without needing explicit rollback logic.

**Clock calls as sentinels:** `resolved_clock = clock or RealClock()` and `started_at = resolved_clock.monotonic()` are called **before** the main try block — not inside it. `stdlib` time functions never raise in practice. If they were inside the try, a clock failure would prevent the `started_at` sentinel from being set, and the `finally` block would fail computing `duration_seconds`. Keeping them outside preserves the sentinel contract.

**Why not a flag:** A flag requires understanding which half of the function body you're in. Sentinel locals make the failure boundary explicit in the type system — `None` means the step didn't complete; a real value means it did.

---

### `complete_spawn()` idempotency as the enabler for safe layered catches

**Decision:** `SpawnApplicationService.complete_spawn()` reads the current spawn record before writing. If the record is already in a terminal state, it returns `False` and performs no write. This idempotency contract is what makes layered catches safe — neither the helper layer nor the surface backstop can corrupt state that the runner has already finalized.

**Known limitation (issue #152):** `complete_spawn()` currently blocks the reconciler from overriding a finalization it has already set. The authority rule in the projection system should allow a later `runner`-origin event to supersede a `reconciler`-origin event, but the current service implementation treats all origins equally once a terminal event exists. This is a pre-existing design tension exposed by the refactor, not introduced by it.

**Related known issues from the refactor:**
- **#153 — Concurrent finalizer metrics:** When two concurrent finalization paths complete near-simultaneously, the second path's metrics (cost, tokens) are silently dropped because the record is already terminal. Merge semantics for metadata are unimplemented.
- **#154 — Teardown mislabeled as `launch_failure`:** Exceptions from teardown code inside `launch_prepared_spawn()` (e.g., from session cleanup) are caught by the helper's broad `except` and recorded as `launch_failure`. They're actually post-execution failures, not pre-launch failures. The label is misleading but benign in practice.

---

### `complete_spawn()` async call replaces `_complete_spawn_sync()` in background worker

**Decision:** Background worker pre-launch failure handling uses `SpawnApplicationService.complete_spawn()` (async, called via `asyncio.run()` at the top level of `_execute_existing_spawn()`) rather than the synchronous `_complete_spawn_sync()` wrapper.

**Why:** `_complete_spawn_sync()` called `asyncio.run()` internally. Calling it from inside `_execute_existing_spawn()` (which is itself called via `asyncio.run()` from `_background_worker_main()`) would nest two `asyncio.run()` calls, which raises `RuntimeError` in Python's asyncio. The fix is to make the worker's main function async throughout, so all completion calls use `await service.complete_spawn()` instead.

**Broader implication:** `_complete_spawn_sync()` remains available for synchronous calling contexts (like `execute_spawn_background()` in the foreground path). The async service method is preferred whenever the caller is already async.

---

### "unknown" sentinel removed from `PreparedSpawn.resolved_model`

**Decision:** `SpawnApplicationService.prepare_spawn()` previously set `resolved_model = ... or "unknown"` to prevent empty strings in the spawn row. The sentinel is removed. Empty model string is stored as-is.

**Why:** `"unknown"` is a lie — it asserts a model was specified when none was. Empty string correctly represents "no model specified; harness uses its own default." The sentinel made `spawn list` output misleading and defeated any filter or report that tries to group spawns by model. The spawn record reflects the resolved state; empty is a valid resolved state for model-optional profiles.

**Relationship to I-7:** Invariant I-7 says "resolved values in the row, never placeholder strings." `"unknown"` violated I-7 by being a placeholder, not a real model name. Empty string is not a placeholder — it means no model was requested.

---

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

### D-primary-approval: Managed-primary Codex approval routing uses `InteractiveHandler`, not `AutoAcceptHandler`

*2026-05-05*

**Decided:** For managed-primary Codex sessions, `CodexConnection` is constructed with an `InteractiveHandler` (passive — surfaces requests as events without auto-answering). Spawn/subagent paths keep `AutoAcceptHandler` (approves everything). The blanket `_primary_observer_mode` rejection of all server requests is removed.

**The defect:** `CodexConnection._handle_server_request()` short-circuited when `_primary_observer_mode` was set, rejecting ALL server requests — including `requestApproval` and `requestUserInput` — with JSON-RPC error `-32601`. This caused the Codex app-server to mark operations as `Rejected("rejected by user")`, the exact symptom observed in detached managed-primary sessions.

**Why the TUI cannot answer:** In managed-primary mode, the app-server has two WebSocket clients — Meridian (primary ws client, turn owner) and the TUI (secondary observer via `codex resume --remote ws://...`). The app-server routes `requestApproval` to the **turn owner** (Meridian), not to the TUI. The TUI receives only notifications. This means the TUI cannot answer approval requests in this architecture; Meridian must.

**Three defects in the same cluster:**
1. **D-PA1:** `_primary_observer_mode` blanket rejection — removed; handler-based dispatch handles approval/user-input methods.
2. **D-PA2:** Managed-primary attach construction injects `InteractiveHandler`.
3. **D-PA3:** `_reject_confirm_mode_approval_request()` is conditional on whether the active handler declares `no_runtime_hitl`.

**Spawn/subagent isolation:** Only managed-primary sessions get the interactive handler. Subagent/spawn connections keep `AutoAcceptHandler` unchanged. This is a deliberate policy boundary — spawn approvals are not routed to the parent by default.

**Crash-only observability:** No second durable request ledger. `InteractiveHandler` already emits `request/opened` events, and `PrimaryAttachLauncher._run_event_writer()` persists connection events to `history.jsonl`. After a crash, a pending request appears in durable history and the live-response registry shows it unavailable/orphaned.

**Also:** `_hitl_requests` must be cleaned up on incoming `serverRequest/resolved` notifications — currently only cleared when Meridian itself sends the answer. If the TUI or another client answers the request, Meridian must clear its pending map to stay consistent.

**Alternative rejected:** Full chat-runtime convergence (treating managed primary as a first-class `ChatRuntime` execution) — correct direction long-term but out of scope for this defect fix; expands boundary beyond the immediate problem.

**Alternative rejected:** Passive observer only, relying on the TUI to answer — does not work for detached-primary control; fails when no local TUI is present.

---

### D-model-invocable: Filter at inventory prompt boundary, not at catalog scan

**Decision (2026-05, PR #208; rewritten 2026-06, PR #314):** `model-invocable: false`
in agent profile frontmatter is enforced by Mars at inventory render time (inside the
launch-bundle `prompt_surface.inventory_prompt` field). The catalog scan
(`scan_agent_profiles()`) returns all profiles regardless of this field. Meridian does
not maintain a Python-side inventory renderer — the bundle is the sole source of truth
and Meridian embeds the rendered string verbatim.

**Why the rewrite (2026-06):** The original `build_agent_inventory_prompt()` Python
renderer was deleted in PR #314 when prompt.py was decomposed. Inventory rendering is
now exclusively Mars-owned — Mars computes the `# Meridian Agents` block (harness-aware
delegation guidance, native-agent sections, model metadata, `model-invocable` filtering)
and Meridian passes through the result. This eliminates the dual-renderer risk where
Python and mars could produce divergent inventory.

**Original Why (still holds):** `scan_agent_profiles()` is a neutral scanner with
multiple consumers — inventory prompt, explicit load, listing commands. Filtering there
would conflate model-facing visibility with general catalog availability. The inventory
prompt is the only model-facing boundary, so that is the correct and narrowest seam.

**Rejected alternative:** Add `model_invocable_only=True` parameter to
`scan_agent_profiles()`. Rejected because it pushes visibility policy into the
catalog layer, which should remain policy-free.

**Default True, backward compatible:** When `model-invocable` is absent from
frontmatter, the field defaults to `True`. All existing profiles remain visible
without any migration.

---

### D-model-invocable-vs-user-invocable: Two separate visibility surfaces

**Decision (2026-05, PR #208; updated 2026-06, PR #314):** `model-invocable` controls
only the model-facing agent inventory prompt (now Mars-rendered in the launch bundle).
It does not restrict explicit user invocation via `meridian spawn -a <name>`. These
are separate surfaces with different consumers.

**Why:** A deprecated or internal agent that shouldn't appear in the model's option
menu may still be explicitly invoked by a user who knows what they're doing. An
internal orchestration tool might need `model-invocable: false` to stay out of the
agent inventory while remaining available to humans directly.

**`user-invocable` deferred:** Filtering explicit CLI invocation and `meridian mars
list` by a `user-invocable` field is tracked separately. This change scopes only to
model-facing prompt injection.

**Explicit invocation ignores the field:** `load_agent_profile()` does not check
`model_invocable`. If a user explicitly requests an agent by name, it loads.

---

### D-mars-owns-inventory: Mars renders agent inventory; Meridian embeds verbatim

**Decision (2026-06, PR #314):** The agent inventory prompt is bundle-only. Mars renders
the `prompt_surface.inventory_prompt` field (harness-aware delegate preference guidance,
native-agent sections, model metadata, `model-invocable` filtering). Meridian parses this
field from the launch-bundle JSON in `bundle_adapter.py` and embeds it verbatim into the
composed system prompt. There is no Python fallback renderer.

**Context:** Before PR #314, Meridian maintained a Python-side `build_agent_inventory_prompt()`
function in `prompt.py` that built the inventory string. When `prompt.py` was decomposed
into text_utils/resolve/prompt_context during PR #314, this function was deleted. The
inventory already came from Mars for all active surfaces; the Python renderer was legacy
fallback code that was no longer exercised in production.

**Why single-owner:** A dual-renderer (Python + Mars) creates a divergence risk — the model
could see inconsistent inventory depending on which path a spawn took. Mars already owns
the `.mars/` directory schema, profile scanning, and agent-copy; inventory rendering is a
natural extension of that ownership. Meridian's job is composition, not rendering.

**Resume/snapshot persistence:** Resume and snapshot replay carry `bundle_inventory_prompt`
on `LaunchPolicySnapshot` so inventory survives without re-calling mars.

**Rejected alternative:** Keep the Python renderer as a backup — rejected because it
allows silent divergence when the two renderers drift. The bundle-only contract makes
divergence impossible.

---

### D-headless-claude-deny: Headless Claude denied by default with 2026-06-15 driver

**Decision (2026-06, PR #314):** Meridian denies headless Claude by default. The
`[spawn] deny_headless_harnesses = ("claude",)` setting ships in auto-scaffolded
`meridian.toml`. Users who override with `deny_headless_harnesses = []` receive a
startup warning. The driver is Anthropic's announced 2026-06-15 deprecation of
headless Claude — Meridian preempts this by denying headless proactively rather than
waiting for `claude -p` to start returning errors.

**Startup warning:** When `deny_headless_harnesses` is empty (user overriden), the
primary CLI path emits a config warning: headless Claude will stop working on
2026-06-15 per Anthropic's deprecation notice. This is informational — Meridian
does not block the override, but ensures the user is aware.

**Why deny-by-default:** Headless Claude `-p` is an Anthropic API surface, not a
local tool. Anthropic controls its lifecycle. Meridian cannot fix a broken headless
path — only warn and route users to alternatives. The deny default ensures users
encounter a clear Meridian error ("headless Claude is denied") rather than an opaque
Anthropic API rejection.

**Rejected alternative:** Allow headless Claude until Anthropic actually removes it —
rejected because the opaque error users would encounter gives no path to recovery.
Meridian's deny message tells users which harnesses are available and why Claude is
denied.

---

### D-agent-copy-key: Mars `settings.meridian.agent_copy` key renamed; auto-scaffolded

**Decision (2026-06, PR #314):** The Mars config key for Claude agent copy was renamed to
`[settings.meridian.agent_copy] harnesses = ["claude"]` (was `[settings.agent_copy]`). On
init, `mars.toml` is auto-scaffolded with the new key structure. Meridian reads
this key via `project_has_claude_agent_copy()` in `permissions.py` — the reader was
updated to match the new key path.

**Why explicit key:** Agent copy is a Meridian concept (not a generic mars feature),
so it belongs under `settings.meridian.agent_copy`. The old `settings.agent_copy` key
was ambiguous — it didn't name which platform owned the copy boundary.

**Auto-scaffold:** New projects get `[settings.meridian.agent_copy] harnesses = ["claude"]`
written into the initial `mars.toml`. Existing projects with the old key continue to
work through mars backwards-compat reading, but Meridian only reads the new key path.

**Why init scaffold is `["claude"]`:** Claude is the only harness that currently supports
agent copy. The scaffolded list is the common case and avoids configuration ceremony.

**Rejected alternative:** Support both old and new key paths in Meridian — rejected
because dual-read creates ambiguity about which key is authoritative when both exist;

---

### D-skill-doc-suppression: Skip supplemental_documents for native-skill harnesses

**Decision (2026-05, PR #208):** When a harness declares `supports_native_skills=True`,
`_resolve_spawn_prepare_projection()` and `_resolve_primary_projection()` in
`context.py` produce an empty `supplemental_documents` tuple instead of calling
`compose_skill_prompt_documents()`.

**Why:** All three active harnesses (Claude, Codex, OpenCode) declare
`supports_native_skills=True`. Before this change, Meridian injected skill content
into `supplemental_documents` for all of them — duplicating content that Mars already
delivers through native channels. This wasted context tokens and contradicted the
harness's declared capability.

**Two delivery channels exist for skill content:**
- `supplemental_documents` → flows through `render_system_instruction_blocks()` into
  inline/user-turn content. Suppressed for all `supports_native_skills=True` harnesses.
- `compose_skill_injections()` → `--append-system-prompt` CLI flag, Claude spawn
  surface only. Preserved — it is the intended mechanism for Claude, not the redundant one.

For Codex and OpenCode, skill content reaches agents only through Mars-materialized
native channels after this change. Claude continues receiving skill content via
`--append-system-prompt`.

**`snapshot_from_resolved_policy()` follows the same gate:** Skill document snapshotting
in `chat/policy.py` is also conditioned on `supports_native_skills`.

See [lessons/arch-refactor-pitfalls.md](../lessons/arch-refactor-pitfalls.md) for the
`prepare_prompt_payload()` or-chain pitfall discovered while implementing this.

---

### Semantic IR + adapter projection for prompt composition

**Decision:** The launch factory produces `ComposedLaunchContent` (harness-agnostic semantic IR), and each harness adapter's `project_content()` method maps this to `ProjectedContent` (harness-specific channel assignments). No conditional branches exist in the composition layer for per-harness content routing.

**Why:** With a shared composition layer and multiple adapters, the alternative is conditional branches in the factory: `if harness == "claude": ... elif harness == "codex": ...`. This violates Open/Closed — adding a new harness requires editing composition code rather than implementing an adapter. The semantic IR pattern lets the factory stay harness-agnostic and adapters own their channel semantics.

**Alternatives rejected:** The old `reference_input_mode` capability flag was an attempt to parameterize routing without per-adapter code. It was removed when the model proved insufficient — harnesses need to express routing logic (not just a single mode value), and the factory shouldn't branch on capability flags for content placement decisions.

See [concepts/composition-pipeline.md](../concepts/composition-pipeline.md).

---

### StrategyMap invariant enforces exhaustive SpawnParams handling per adapter

**Decision:** Each harness adapter declares a `STRATEGIES: StrategyMap` dict mapping every `SpawnParams` field name to a `FlagStrategy`. Fields not in the map must be in the `_SKIP_FIELDS` set. Missing mappings raise `ValueError` at build time.

**Why:** Without this, adding a new field to `SpawnParams` silently ignores it in adapters that don't handle it. Subtle omissions (forgetting to pass `report_output_path` to a harness that supports it) are hard to catch in review. The invariant makes the gap a runtime error at the earliest possible moment.

See [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — command assembly.

---

### Config precedence: CLI flag > ENV var > agent profile > project config > user config > harness default

**Decision:** Every config field resolves independently through a fixed precedence chain. A CLI model override also forces harness re-derivation from the overridden model (not from the profile's harness).

**Why:** Independent field resolution is necessary because different fields have different override sources. A user might set a default model in user config but override approval mode on a specific spawn — each field must resolve to the "nearest" (highest precedence) source independently.

**Forced harness re-derivation:** If the user specifies `-m codex` but the profile specifies `harness: claude`, the harness must follow the model, not the profile. Without re-derivation, a model override silently uses the wrong harness.

---

## Harness Identity and Environment

### D32: `MERIDIAN_HARNESS` is a one-hop env var; does not cascade to grandchildren

**Decision:** `build_launch_context()` writes `MERIDIAN_HARNESS = harness.id.value` into every spawned process's environment. The variable is NOT in `ALLOWED_CHILD_ENV_KEYS` and does not propagate to grandchildren. Each spawn level gets its own value derived from its own harness resolution.

**Why:** A grandchild spawn may use a different harness than its parent (resolved from its own profile, model override, or config). Inheriting the grandparent's harness value would silently override the grandchild's resolved harness. One-hop semantics ensure each process knows its own harness, not its ancestor's.

**Purpose of the variable:** "What harness am I running inside?" — used by wait-yield timing logic, not as a routing input.

See [architecture/launch-system.md](../architecture/launch-system.md) — MERIDIAN_HARNESS child env injection.

---

### D33: Wait-yield uses the orchestrator's own `MERIDIAN_HARNESS`, not child spawn rows

**Decision:** `spawn_wait_sync()` determines the yield interval by reading `os.getenv("MERIDIAN_HARNESS")` — the harness the orchestrator is running inside — rather than inspecting the harness fields of the spawn rows being waited on. All harnesses now default to 3000 seconds (50 minutes), unified from the previous 900s for Claude/Codex and 240s for fallback.

**Why:** The yield protects the **orchestrator's** prompt-cache context window. The question is "how long until *my* cache expires?" — answered by the caller's own harness, not the harnesses of the tasks it delegated. Reading child spawn rows was wrong in principle: a mixed-harness wait set would use the shortest interval of the children, but the cache being preserved is the parent's.

**Unified 50-minute default:** Claude Code's prompt-cache TTL is 5 minutes by default, or 1 hour with `ENABLE_PROMPT_CACHING_1H=1` (recommended for all users). Codex and OpenCode also support extended cache up to ~1 hour. A unified 3000s default keeps Meridian comfortably inside the warm window for all harnesses without requiring per-harness tuning.

**Alternatives rejected:** Scanning child spawn harness rows (wrong semantic); process tree inspection (fragile, platform-specific); keeping per-harness split defaults (unnecessary complexity once all harnesses support comparable cache TTLs).

See [concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) — harness-aware yield defaults.

---

### D57: `MERIDIAN_HARNESS` is spawn-local, not a user-facing policy override

**Decision:** `MERIDIAN_HARNESS` written to child process environments by `build_launch_context()` is an informational spawn-local value. It is NOT consumed as a user policy override by `resolve_policies()`. `overrides.py` explicitly ignores it when building policy.

**Why:** `MERIDIAN_HARNESS` tells the child process "which harness you are running inside" — used by wait-yield timing logic. If it were treated as a policy override, nested launches would inherit the parent's resolved harness, bypassing their own profile and config. This was identified as blocking issue H1 in the architecture review.

**Future work:** Rename the spawn-local env var to `MERIDIAN_SELECTED_HARNESS` (or similar) so the user-facing `MERIDIAN_HARNESS` env var can safely be consumed as a routing input per consumer routing spec S1.2.

See also [decisions/model-resolution.md](model-resolution.md) for the model routing context.

---

### D63 (launch): `MERIDIAN_HARNESS_COMMAND` removed; harness adapters are the only launch path

**Decision:** The `MERIDIAN_HARNESS_COMMAND` environment variable override mechanism was removed. Harness adapters are now the only way to specify how a harness is launched.

**Why:** `MERIDIAN_HARNESS_COMMAND` was a backdoor that bypassed the adapter's command assembly, including the StrategyMap invariant (exhaustive field handling) and security checks. Any legitimate need to customize a launch command belongs in the adapter's `build_command()` method — where it goes through review, testing, and invariant checking.

**Alternatives rejected:** Keep as an advanced escape hatch — rejected because there was no documented use case that couldn't be served by adapter customization.

---

## Spawn Wait Barrier

### `spawn wait` no-arg discovers pending spawns by MERIDIAN_CHAT_ID lineage

**Decision:** `meridian spawn wait` with no positional IDs collects all active spawns (`queued`, `running`, `finalizing`) whose `chat_id` matches `MERIDIAN_CHAT_ID`. The wait set is frozen at discovery time; spawns launched after discovery are not added.

**Why:** Orchestrators frequently forgot to run `spawn wait` or lost track of individual spawn IDs. No-arg wait removes the bookkeeping burden — the orchestrator launches all background spawns, then runs one barrier command with no arguments.

**Why chat lineage as scope:** All spawns in a tree share the same root `MERIDIAN_CHAT_ID` by inheritance at any depth. Chat lineage is the natural scope for "all the work I started."

**Alternatives rejected:** Work-item scope — considered but rejected because not all spawns are attached to a work item, and the user's stated preference was chat lineage as primary scope.

See [concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md).

---

### `--yield-after-secs` not `--timeout-secs`; checkpoint exits 0

**Decision:** The wait-yield interval flag is named `--yield-after-secs`. Yield (checkpoint expiry while spawns are still pending) exits with code 0 and prints continuation guidance. "Timeout" semantics (hard failure) are reserved for the existing `--timeout` flag. When `--timeout` is not explicit, `hard_deadline = None` and only the checkpoint governs.

**Why:** Naming the flag `--timeout-secs` would imply failure semantics, causing agents to treat a cache-preserving continuation as an error requiring retry logic. Keeping "timeout" reserved for failures makes the distinction unambiguous. Disabling the hard deadline in checkpoint mode prevents config-level `wait_timeout_minutes` from silently preempting checkpoint-mode waits.

---

### Spawn wait yield interval is harness-aware, not a global constant

**Decision:** The default wait-yield interval is harness-aware: all harnesses default to 3000 seconds (50 minutes), with a 30s minimum clamp. `--yield-after-secs` overrides per invocation.

**Why:** All supported harnesses (Claude Code with `ENABLE_PROMPT_CACHING_1H=1`, Codex extended cache, OpenCode) support cache TTLs of roughly 1 hour or more. A unified 50-minute default avoids unnecessary re-invocations for long-running sessions without requiring the user to tune the interval manually. Users who cannot extend harness cache TTL can lower the yield interval via `--yield-after-secs` or `MERIDIAN_WAIT_YIELD_AFTER_SECONDS`.

**Alternatives rejected:** A single configurable global — doesn't adapt to harness differences. TTL-minus-buffer modeling — too implementation-specific and brittle when TTLs change. Per-harness split defaults (240s/900s) — obsolete now that all harnesses support comparable extended cache TTLs.

> **Implementation updated (2026-05-05).** Values unified to 3000s for all harnesses. Previous split: unknown/default 240s, claude/codex 900s. Harness detection still uses `os.getenv("MERIDIAN_HARNESS")` (D33, 2026-04-30).

---

### Background spawn output uses MUST-run barrier language

**Decision:** The CLI output for a successful `--bg` spawn submission uses explicit obligation language ("you MUST run: `meridian spawn wait`") rather than weak suggestion language.

**Why:** Agents were reliably not running the barrier before reporting. Weak suggestion language was insufficient. Explicit MUST-run text on every background spawn submission is intended to drive habit formation in orchestrators. The no-arg wait command is shown as the primary form; a spawn-specific fallback is shown for non-session callers.

---

## Spawn Goal (`--goal`)

### Spawn-level goal authority: not work-level, not prompt sugar

**Decision (2026-05, spawn-goal):** `meridian spawn --goal TEXT` stores goal as a first-class flat field on the spawn record, not as a work-level property and not as additional prompt body text.

**Why spawn-level:** Work-level goal would create inheritance ambiguity across the multiple spawns (reviewer, tester, etc.) that belong to one work item, each with a different purpose. A future work-level default can materialize into `SpawnCreateInput.goal` at create time without changing v1's spawn-level authority.

**Why flat field, not sidecar:** `state.json.goal` is the simplest structured field and is enough for future hooks. A `completion-contract.json` sidecar becomes valuable only when the contract gains subfields (success criteria, required artifacts, hook state). Adding it before those needs exist creates a second persistence authority prematurely.

**Why not prompt sugar:** Goal as prompt body text would require future hooks to parse prompt artifacts — violating files-as-authority. A structured field remains queryable without prompt parsing.

**Alternatives rejected:**
- Work-level goal — inheritance ambiguity; couples the feature to work-item semantics prematurely
- `completion-contract.json` sidecar in v1 — no subfields yet; two persistence authorities before hooks exist
- Prompt-body injection only — future hooks would need to parse prompt artifacts

---

### Goal does not affect policy; it only affects content composition

**Decision:** `compile_prepared_policy_surface()` (in `launch/policies.py`) is goal-agnostic. Goal does not affect model, harness, approval mode, or routing. It affects only the semantic IR (`completion_contract` block) inside content composition.

**Why:** Policy decisions (model selection, harness routing) happen before content assembly. Goal is a bounded completion contract for the agent, not a routing signal. Keeping policy goal-agnostic preserves the policy/mechanism separation and makes goal a pure composition concern.

---

### `completion_contract` placed after `context_prompt` in `SYSTEM_INSTRUCTION_BLOCK_ORDER`

**Decision:** The new `completion_contract` field on `ComposedLaunchContent` is positioned after `context_prompt` in `SYSTEM_INSTRUCTION_BLOCK_ORDER`:

```python
SYSTEM_INSTRUCTION_BLOCK_ORDER = (
    "supplemental_documents",
    "agent_profile_body",
    "report_instruction",
    "inventory_prompt",
    "context_prompt",
    "completion_contract",   # ← goal injected here
)
```

**Why:** The goal contract is behavior-shaping and should be prominent after the agent has been oriented via context/inventory. Placing it after `context_prompt` ensures the agent reads its context before seeing the completion obligation. `passthrough_system_fragments` (user's explicit `--append-system-prompt`) remains last to allow deliberate user overrides.

---

### Goal text normalization: trim-only; no `--prompt-var` substitution

**Decision:** Goal normalization trims only leading/trailing whitespace. Interior whitespace — spaces, newlines, list formatting — is preserved exactly. `--prompt-var` substitution is not applied to goal text.

**Why:** The same goal text must appear both in `state.json` (persisted) and in the injected completion contract. If template substitution ran after persistence, the persisted text would differ from the injected text, making the state.json record unreliable as a source of truth. For v1, keeping them identical is correct.

**Error contract:** Normalization rejects explicitly provided empty strings with `--goal cannot be empty`. `None` is the only absent-goal value accepted by the prompt renderer; an empty or non-normalized string reaching the renderer indicates an upstream bug and fails fast.

---

### Continue/fork goal inheritance: concrete spawn refs only

**Decision:** `meridian spawn --continue SOURCE` inherits the source spawn's goal when SOURCE is a tracked spawn row; explicit `--goal TEXT` overrides. `meridian spawn --fork p<ID>` inherits goal when the reference matches the concrete spawn-ref shape (`p` followed by digits); chat IDs, raw harness session IDs, and untracked refs do not inherit a goal.

**Implementation:** A `_looks_like_spawn_ref(ref: str)` type-narrowing helper gates the inheritance lookup:

```python
def _looks_like_spawn_ref(ref: str) -> bool:
    normalized = ref.strip()
    return len(normalized) > 1 and normalized[0] == "p" and normalized[1:].isdigit()
```

**Why no chat-ref inheritance:** A chat can contain multiple child spawns (reviewer, tester, coder) with different goals. There is no authoritative single goal owner for a chat ID. Requiring explicit `--goal` for chat/raw refs prevents invented goal inheritance from surprising the user.

**SpawnStartMetadata carrier:** The start call passes goal via a dedicated `SpawnStartMetadata` dataclass (not loosely via `**kwargs`) to keep the call site explicit and auditable. This is the R-05 refactor from the plan.

---

## Windows Compatibility (2026-05, PR #200 / PR #184)

### D-win-signal: Portable signal handlers via `signal.signal()` + `call_soon_threadsafe()`

**Decision:** Signal handlers in async spawn runners use `signal.signal()` +
`loop.call_soon_threadsafe(event.set)` instead of `loop.add_signal_handler()`.

**Why:** `loop.add_signal_handler()` is not implemented on Windows
`ProactorEventLoop` — it silently no-ops or raises `NotImplementedError`. The
`signal.signal()` + `call_soon_threadsafe()` pattern is cross-platform and
produces identical behavior on POSIX. The cleanup callable pattern (store
previous handlers, restore on exit) is preserved.

**Constraint:** `signal.signal()` must be called from the main thread.
`_install_signal_handlers()` returns `None` when called from a worker thread
to signal "no cleanup needed." Callers handle `None` by skipping cleanup.

**Alternatives rejected:** Platform-specific branches (`if IS_WINDOWS: ...`) —
rejected because the portable pattern works on both platforms without branching.

---

### D-win-cancel: Claude cancel uses `terminate()` on Windows, `send_signal(SIGINT)` on POSIX

**Decision:** The Claude harness adapter's cancel path branches on platform:
POSIX sends `SIGINT` (graceful interrupt, allows shutdown handlers to run);
Windows calls `process.terminate()` (maps to `TerminateProcess`, immediate).

**Why:** `send_signal(SIGINT)` on Windows maps to `GenerateConsoleCtrlEvent`.
This only works for processes in the same console group. Harness subprocesses
launched with `CREATE_NEW_PROCESS_GROUP` — the isolation default — never
receive the event. The subprocess appears to ignore the cancel, continuing
to run until externally killed. On POSIX, `SIGINT` is reliable and allows
Claude to handle it gracefully (e.g., emit a final report before exiting).

**Why not `terminate()` on both platforms:** `terminate()` is `TerminateProcess`
on Windows but `SIGTERM` on POSIX. `SIGTERM` on POSIX gives Claude no
opportunity to run its SIGINT shutdown handler. The POSIX preference is SIGINT
for graceful shutdown; the Windows fallback is terminate because graceful
interrupt delivery is not available.

---

### D-win-output-log: `output_log_path` must be passed through; no hardcoded `None`

**Decision:** `WindowsConsoleLauncher` passes `output_log_path` from its
`bind_launch_context()` call through to the launch spec. Hardcoding `None`
was a one-line omission that caused spawn output to be discarded on Windows.

**Why:** `output_log_path` is how the streaming runner knows where to write
the session history artifact. A hardcoded `None` was a silent bug — the spawn
succeeded but no history was captured, and no error was raised.

---

### D-win-ci: Windows CI added as required check (no Windows-only code without CI gate)

**Decision:** Windows CI is a required check on all PRs that touch platform
behavior (paths, signals, processes, locks, file operations, config discovery).

**Why:** Three of the four Windows bugs in the audit were pre-existing but
only caught by explicit Windows testing. POSIX CI had silently passed through
Windows-specific regressions. CI is the only scalable gate for platform
correctness at PR review time.

**Scope rule:** When a change touches paths, process launching, signals, shells,
filesystem semantics, locking, or config discovery — explicitly consider Windows
behavior and add or update tests so behavior is verified on Windows, not just
inferred from POSIX coverage.

---

### D-win-stdout: `sys.stdout.reconfigure(encoding='utf-8')` at CLI entry on Windows

**Decision:** The CLI entry point calls `sys.stdout.reconfigure(encoding='utf-8')` on
Windows before any command runs. The call is Windows-only; POSIX is unchanged.

**Why:** On Windows, `sys.stdout` defaults to the active code page (often cp1252 or
cp850). Any Unicode output — agent names, file paths with non-ASCII characters, emoji
in status messages — raises `UnicodeEncodeError` or silently corrupts the output.
Reconfiguring stdout to UTF-8 at the entry point normalizes encoding for all downstream
code without requiring per-call `encode`/`errors='replace'` guards.

**Why not `PYTHONUTF8=1`:** The env var would require users to set it before running
Meridian. A startup call inside the CLI owns the fix unconditionally.

**Why Windows-only:** On POSIX, stdout is already UTF-8 (locale-determined in practice).
Reconfiguring on POSIX would change behavior for terminals that legitimately use
non-UTF-8 locales; the risk is asymmetric.

---

### D-win-spawn-wait-scope: No-arg `spawn wait` in nested spawns scoped to descendants only

**Decision:** When `MERIDIAN_SPAWN_ID` is set (the caller is a nested spawn,
not the primary session), `_resolve_wait_targets()` passes
`only_descendants_of=self_spawn_id` to `_discover_pending_spawns()`. The wait
set is restricted to the caller's subtree via BFS. Primary session behavior
(no `MERIDIAN_SPAWN_ID`) is unchanged — it still waits on all chat-scoped spawns.

**Why:** Without descendant scoping, no-arg `spawn wait` from a tech-lead or
orchestrator spawn would include the primary session in its wait set. The
primary session is perpetually active while spawns run — it never terminates
during the orchestrator's lifetime. This caused tech-lead and orchestrator
agents to hang indefinitely on `spawn wait`.

**Why BFS (not parent-only filter):** An orchestrator may spawn nested
orchestrators, which themselves spawn workers. A flat parent-only filter would
leave grandchild spawns in the active-but-invisible category — the orchestrator
waits for its direct children, but a direct child waiting for its own children
fails to drain because the grandparent's wait released too early. BFS through
`_collect_descendants()` captures the full subtree regardless of depth.

**Alternatives rejected:**
- Work-item scope filter — not all spawns have work items; primary session
  spawns are not work-item-scoped
- Explicit spawn ID tracking — the existing background-spawn output already
  gives explicit IDs; no-arg wait exists specifically to avoid tracking IDs
- Filter to direct children only — misses nested orchestration; still incomplete

See [../concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) —
descendant-scoping semantics and implementation detail.

---

---

## Structural Simplification (2026-05, PR #184)

### Wave 1: Indirection removal

**Decision:** Four proxy/wrapper indirections were deleted:
- `decision.py` proxy — callers now call target functions directly
- `prepare.py` wrapper pairs for the dry-run prepare path — inlined to direct calls
- `process/__init__.py` compat shim — removed
- `autocompact` validation — centralized

**Why:** Each removed indirection was a pure maintenance surface with no semantic value — it routed calls without adding logic, policy, or error handling. Extra layers obscured the actual call chain, made grep-based navigation harder, and created dead weight in reviews. Removing them makes the data flow explicit and auditable at a glance.

**Principle:** Indirections earn their keep by expressing a boundary, enabling testing, or isolating change. Proxy modules that just re-export the target do none of these.

---

### Wave 2: Session carrier consolidation

**Decision:** `session_scope()` and `lifecycle.start()` accept typed carriers (`SessionRequest` + `PrimarySessionMetadata`) instead of individual keyword arguments.

**Why:** `**kwargs` spreading at call sites made it impossible to see at a glance what was being passed, deferred type errors to runtime, and meant new fields silently vanished if a caller forgot to thread them. Typed carrier dataclasses make the call site explicit and auditable — what goes in is visible from the type, not inferred from reading the function body.

This is the same rationale as `SpawnStartMetadata` in the spawn-goal work: typed carriers over `**kwargs` is the consistent pattern for session and spawn boundary calls.

---

### Wave 3: LaunchSpec flattening

**Decision:**
- `build_child_env_overrides` wrapper deleted — call sites use the target directly
- `LaunchRequest` compat bridges removed
- Harness launch specs flattened into a single `ResolvedLaunchSpec`

**Why:** The compat bridges existed to ease migration from an older struct shape. Once the migration was complete, the bridges were pure ceremony — they copied fields without transforming them. Flattening into `ResolvedLaunchSpec` gives a single canonical struct that all downstream paths read, eliminating the question of which shape is authoritative.

## Session Initiation (2026-05, PR #216)

### D-fork-identity-lock: `--fork` locks identity; `--fork-fresh` allows changes

**Decision:** `--fork` rejects `-a`, `-m`, and `--skills` at the CLI. `--fork-fresh` is the distinct mode that permits identity overrides. Enforcement is CLI-only — the ops layer (`SpawnForkInput`) handles both modes without a separate field.

**Why:** Agent, model, and skills determine the system prompt fingerprint — the dominant factor in prompt cache locality. `--fork` guarantees identity-shaping inputs are unchanged, so the harness cache can warm from the shared prefix. Without the explicit split, callers would silently invalidate the cache by passing `-a` to `--fork`. The two modes capture two distinct user intents: "branch this conversation to explore a task variant" (`--fork`) vs "hand off to a different role" (`--fork-fresh`).

Task-scoped overrides (`--goal`, `-f`, `--prompt-var`, `--work`, `-p`) are allowed by `--fork`. These change task-scoped content within a preserved identity, not the identity itself. `--prompt-var` substitutes into the profile body — this is intentional customization, not an identity change.

**Why CLI-only:** `SpawnForkInput` already handles both fork behaviors. The ops layer does not need to enforce CLI ergonomics. Non-CLI callers (MCP, programmatic) may legitimately fork with identity changes.

**Alternatives rejected:**
- Reject `-a`/`-m`/`--skills` in the ops layer — couples an ergonomic CLI policy to a layer that has no semantic stake in it; breaks programmatic callers.
- Add a mode field to `SpawnForkInput` — adds coupling without correctness benefit; the CLI already validated before constructing the input.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) — full four-mode model.

---

### D-prior-context-user-turn: `--from` prior context goes in user turn, not system prompt

**Decision:** Prior spawn/session context delivered via `--from` is rendered into `UserTurn.context_blocks` (user turn), not into `SystemInstruction` (system prompt / `--append-system-prompt`).

**Why (four independent reasons):**

1. **Prompt injection defense.** Prior spawn reports may contain user-generated content or tool outputs. System prompt placement grants implicit authority status. User-turn placement means the model evaluates the content as evidence, not as instructions to obey. `sanitize_prior_output()` is defense-in-depth; channel placement is the primary trust boundary.
2. **Prompt cache locality.** The system prompt fingerprint determines the cache key. Variable prior-context material in the system prompt destroys cross-session cache sharing for sessions that share the same agent identity. User-turn injection preserves the stable system-prompt prefix.
3. **Harness consistency.** `--append-system-prompt` is reserved for agent identity material. Codex and OpenCode have no separate system-prompt channel; the user-turn channel is the only consistent cross-harness delivery path.
4. **Semantic consistency with `-f`.** File references (`-f`) go in user-turn context blocks. `--from` prior context is structurally the same kind of reference material. Same channel avoids a confusing distinction.

**Exception:** The completion goal (`--goal`) and report contract are authoritative stopping conditions set by Meridian — not prior agent output — and belong in `SystemInstruction`.

**Alternatives rejected:**
- System prompt injection for authority — grants untrusted prior output instruction status; invalidates the prompt cache.
- Harness-specific channels — inconsistent across Claude/Codex/OpenCode; user-turn is the only cross-harness consistent path.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) — full layer model and authority hierarchy.

---

### D-argv-normalization-sentinel: Pre-Cyclopts argv normalization for optional-value flags

**Decision:** `--fork`, `--fork-fresh`, and `--from` accept an optional positional REF. Because Cyclopts requires a value token for `str | None` parameters, bare invocations (e.g., `--fork` without a ref) would cause a parse error. The fix is pre-Cyclopts argv normalization: `normalize_optional_value_flags()` in `cli/argv_normalization.py` inserts a sentinel token (`__SELF__`) before Cyclopts sees the argv. Bootstrap uses `SYNTHETIC_VALUE_TOKENS` to detect and skip synthetic values. `resolve_optional_ref(raw_ref, flag_name)` maps the sentinel to `$MERIDIAN_SPAWN_ID` or raises with a flag-specific error.

**Why pre-Cyclopts:** Cyclopts does not support optional-value parameters natively. Post-Cyclopts transformation would require patching the parsed result after Cyclopts already rejected the bare form. Pre-normalization is transparent to Cyclopts — the sentinel looks like a normal value token.

**Sentinel design:** `__SELF__` is not a valid spawn ref (`pN`), chat ref (`cN`), or UUID. The sentinel is defined as a named constant (`SELF_FORK_REF_SENTINEL`), not a magic string, and checked explicitly in `resolve_optional_ref`.

**Bare ref semantics:** All three flags default to `$MERIDIAN_SPAWN_ID` — the current spawn context. `$MERIDIAN_CHAT_ID` is available as an explicit ref for session-level context. `--continue` was intentionally excluded: bare `--continue` means "resume this session" which is a no-op, and its semantics differ from the optional-ref pattern.

**Applies to both surfaces:** spawn and primary. The normalization runs in `main.py` before bootstrap and Cyclopts dispatch.

**Alternatives rejected:**
- Cyclopts optional-value parameter — not supported without custom type annotation plumbing.
- Post-parse sentinel injection — brittle; Cyclopts rejects before transformation can run.
- Require explicit ref always — adds ceremony for the common case; `$MERIDIAN_SPAWN_ID` is always set inside a Meridian-managed session.

---

### D-from-fork-mutual-exclusion: `--from` + fork rejected in MVP

**Decision:** Combining `--from` with `--fork` or `--fork-fresh` is rejected. If external context is needed on a forked conversation, pass it via `-f`.

**Why:** `--fork` already carries the full prior transcript as Layer 2 (harness-managed, Meridian-invisible). Adding `--from` on top is redundant — the model would see the same context twice (once as lived transcript, once as rendered reference) with ambiguous ordering between a harness-managed layer and a Meridian-rendered layer. `-f` composes cleanly with fork because file references are always Layer 3 content with no semantic overlap with transcript lineage.

**Future path if needed:** Allow `--from` + `--fork` with explicit ordering rule: `--from` blocks rendered into user turn after the forked transcript and before `-p` text. But the redundancy risk makes this opt-in, not default.

---

## Related

- [../architecture/launch-system.md](../architecture/launch-system.md) — launch architecture
- [../concepts/composition-pipeline.md](../concepts/composition-pipeline.md) — prompt composition mental model
- [../concepts/session-initiation.md](../concepts/session-initiation.md) — four-mode session initiation model
- [../concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) — wait mechanism
- [state-and-launch.md](state-and-launch.md) — compatibility map for the previous combined decision page
