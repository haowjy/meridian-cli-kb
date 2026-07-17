# open-questions/future-work — Deferred Items and Known Gaps

Tracked follow-ups deferred from implementation or design sessions. Each entry names the issue, the context, and why it was deferred. This is not a backlog — it's a record of known state so future work doesn't re-derive the same conclusions.

Items are organized by domain.

---

## Chat Backend

### Scoped Nested Chat Runtime Roots {#scoped-nested-chat-runtime-roots}

**Where:** `src/meridian/lib/chat/runtime.py` — `recover_all()` / `list_chats()`, `cli/chat_cmd.py` — runtime root construction

**Context:** Nested `meridian chat` launch is now allowed ([chat backend D32](../decisions/chat-backend.md#d32), 2026-05-07). Smoke and browser testers running at `MERIDIAN_DEPTH > 0` can launch real live chat instances to exercise the full chat shared-policy path. This unblocked E2E chat testing.

**The issue:** Nested chat instances share the global state root with their parent process. `ChatRuntime.recover_all()` at startup scans all on-disk chat sessions without scoping by process or spawn context. In a concurrent scenario — parent chat running + nested tester chat launching — the nested instance may discover and attempt recovery of the parent's active sessions, and the parent's `list_chats()` may expose the nested instance's sessions. State cross-talk is low-risk for short-lived smoke runs (start, run, exit), but is a latent correctness hazard for longer-lived concurrent nested chats.

**Architecture recommendation (design-lead p4971):** Key each chat instance's runtime root by spawn scope:
- When `MERIDIAN_DEPTH > 0`, derive a per-scope runtime root from `MERIDIAN_SPAWN_ID` or equivalent rather than using the global user project root.
- `recover_all()` and `list_chats()` operate only within the instance-scoped root, preventing cross-discovery.
- Session-state, event logs, and the event index are written under the scoped root, not the shared one.
- The parent process is unaffected — its runtime root remains the global one (`MERIDIAN_DEPTH == 0`).

**Why deferred:** The minimal guard removal ships the testing unblock now. State cross-talk is acceptable for short-lived smoke/test runs. The scoped-root design requires threading an instance-scoped path through `chat_cmd.py`, `ChatRuntime`, and `BackendAcquisitionFactory` — a non-trivial refactor that touches state-system and chat runtime boundaries. Design work deferred until the concurrent nested-chat scenario becomes a real operational concern.

**Decision context:** [decisions/chat-backend.md#d32](../decisions/chat-backend.md#d32)

---

### TUI Passthrough Deprecation

**Where:** `lib/harness/passthrough/` (Claude, Codex, OpenCode)

**The issue:** The `passthrough/` module is intentionally silo'd from the chat backend (D23), but the actual deprecation execution hasn't happened. Claude TUI passthrough is the highest priority: it's already the most constrained (separate process, no shared backend). Once the browser UI matures, Codex and OpenCode passthrough become redundant too.

**Deprecation order:** Claude first (most painful to maintain), then Codex/OpenCode.

**The clean path:** Delete `harness/passthrough/claude.py` when `meridian chat` is stable with Claude. `meridian launch` with Claude becomes unavailable (or is redirected to `meridian chat`). Then delete Codex/OpenCode passthrough when the browser UI handles interactive sessions.

**Why deferred:** Browser UI doesn't exist yet. Passthrough still serves `meridian launch` users. Deferred until Phase 6 (CLI packaging) or later.

**Decision context:** [decisions.md](../decisions.md) — D23

---

### SpawnManager `_history_writers` Leak

**Where:** `src/meridian/lib/streaming/spawn_manager.py` — `start_spawn()`

**The issue:** `start_spawn()` allocates `_history_writers[spawn_id]` before `control_server.start()`. If the control server fails (Unix socket binding failure), the history writer entry leaks. `stop_spawn()` also doesn't clean up the entry when `session is None`.

**Severity:** Medium. In practice `control_server.start()` failures are rare. The leaked entry is small and doesn't affect correctness — only memory in long-lived servers.

**What's needed:** In `start_spawn()`, wrap `control_server.start()` + subsequent initialization in try/except; clean up `_history_writers[spawn_id]` on failure. In `stop_spawn()`, ensure cleanup when `session is None`.

**Why deferred:** Pre-existing SpawnManager debt, not introduced by chat backend. The chat acquisition cleanup (observer unregister + stop spawn) is correct for its scope. SpawnManager internal state cleanup tracked separately.

**Decision context:** [decisions.md](../decisions.md) — D29

---

### Structural Refinements from Final Gate Review (D27)

**Where:** `lib/chat/` and `lib/harness/`

Two structural findings from refactor-reviewer p2499 remain after Phase 7–11 refactors addressed the others.

1. **Event interpretation deduplication — still open.** Despite R6 centralizing semantics in `harness/semantics.py`, some event interpretation is duplicated between normalizers and the semantic functions. Should be a single call-site.

2. **Speculative capability surface cleanup — still open.** `ConnectionCapabilities` has flags for capabilities that no current harness supports. Clean up or document clearly which flags are speculative vs. real.

**Why deferred:** Correctness bugs (D27 blocking fixes) took priority. These two improvements compound future maintenance but don't block shipping.

---

### Structural Notes from Phase 7–11 Final Gate (D31)

**Where:** `lib/chat/`, `lib/harness/`, `lib/streaming/`

These notes were accepted as next-iteration cleanup after the Phase 7–11 refactor. They are structural follow-ups, not regressions.

1. **`ColdSpawnAcquisition` carries launch policy.** It lives in `chat/` but assembles launch/connection policy that may deserve its own module or factory boundary.

2. **`runtime`/`recovery` bootstrap cycle.** Recovery delegation works, but the lazy import in `start()` hides a module relationship that should be made explicit or untangled.

3. **Harness chat policy scatter.** Chat-specific harness policy is spread across connection config, acquisition, normalizer registry, and passthrough boundaries. Future work should consolidate the policy map without re-coupling mechanisms.

4. **Passthrough naming ambiguity.** `passthrough/` is correctly silo'd from `meridian chat`, but names can still imply it participates in the chat backend. Rename or document locally when the passthrough deprecation path is executed.

**Decision context:** [decisions.md](../decisions.md) — D31

---

### Pre-existing Reliability Findings from Final Gate (D30)

**Where:** `lib/chat/event_log.py`, `lib/chat/ws_fanout.py`, `lib/chat/checkpoint.py`, `lib/chat/event_index.py`, `lib/chat/session_service.py`

Final gate review found concurrency/safety concerns that were not introduced by the Phase 7–11 refactor. They need their own design/planning cycle:

1. **Event log truncation race.** Review append/read/truncation behavior around JSONL repair and concurrent readers.

2. **Fanout teardown leak.** Ensure WebSocket fanout cleanup fully releases client references after close/error paths.

3. **SQLite connection lifecycle.** Audit derived-index connection ownership and closure under failure paths.

4. **Checkpoint cross-process locking.** Checkpoint/revert safety should hold across processes, not just in-process guards.

5. **Stop/acquisition race.** Session stop and backend acquisition should have a clear synchronization contract.

**Decision context:** [decisions.md](../decisions.md) — D30

---

### Dead Code: Chat Backend Refactor Residue (p2595)

**Where:** `lib/chat/`, `lib/launch/process/runner.py`, `lib/harness/passthrough/`

**Reviewed:** 2026-04-30 (p2595). Read-only audit of the chat backend / refactor area. Four categories of dead code found. All non-blocking.

**1. Dead model/harness fields in CreateChatRequest**

`CreateChatRequest.model` and `.harness` in `lib/chat/server.py:37-46` are threaded through transport → runtime (`lib/chat/runtime.py:155-169`) and explicitly discarded with `_ = (model, harness)`. Server startup fixes the backend at process boot (`cli/chat_cmd.py:82-89`), so per-chat overrides have no effect.

**Action:** Remove `model`/`harness` from the create-chat transport and runtime until per-chat backend selection is real, **or** actually thread them into acquisition.

**2. Unused function parameters and dead return slots in runner.py / backend_acquisition.py**

- `_execute_primary_process()` (`runner.py:200-216`) takes `session_mode` but never reads it.
- Same function returns a `bool` (used-managed-backend flag) that the only caller discards as `_used_managed_backend` (`runner.py:623-626`).
- `_run_primary_attach()` (`runner.py:357-371`) takes `project_root` and explicitly ignores it.
- `_build_normalizer()` (`backend_acquisition.py:172-179`) takes `harness_id` and explicitly ignores it.

**Action:** Collapse signatures/returns to only the values actually read.

**3. Unused barrel re-exports**

- `lib/chat/normalization/__init__.py:3-15` re-exports the whole normalization surface; no in-repo imports from this path.
- `lib/chat/event_observer.py:58` re-exports `EventNormalizer`; no in-repo imports of that symbol from this module.
- `lib/harness/passthrough/__init__.py:3-6` re-exports `TuiPassthrough`; no in-repo imports from the package root (`get_passthrough` is the live consumer).

**Action:** Delete unused re-exports/barrels or trim to the actually used symbols.

**4. Unreachable "draining" logic in recovery.py**

`lib/chat/recovery.py:68-69` checks `last_state in {"active", "draining"}`, but `_state_from_events()` can only derive `idle`, `active`, or `closed` from persisted events — `"draining"` is only set in-memory in `session_service.py:95-103` and never persisted. Recovery logic can never reconstruct a `"draining"` state from disk.

**Action:** Either persist a draining-transition event so recovery can reconstruct it, or remove `"draining"` from recovery-time state derivation.

---

## State & Spawn

### Process-Scope Cleanup Gaps

Process-scope deferred work moved to [process-scope.md](process-scope.md) so the detailed PROC-004 and PROC-007 notes live with their shared containment/reaper context.

### list_spawns() ~386ms — O(spawns) file reads in v2

**Where:** `state/spawn_store.py:list_spawns()`, `state/spawn/repository.py:scan_spawn_ids()`

**The issue:** V2 per-spawn `state.json` made individual spawn reads O(1) and eliminated the 12–13s primary launch bottleneck. But `list_spawns()` is now O(spawns) in file reads — each spawn requires reading one `state.json`. At ~4,000 spawns in production the measured time is ~386ms. Not blocking in practice (startup is already 0.67s), but will grow linearly with spawn count.

**Directions to consider:**
- A lightweight index file at `spawns/index.json` that caches status/key fields for all spawns — updated on every state write. Trades write overhead for list read speed.
- Separate hot-path list fields (status, agent, started_at) from cold fields (prompt, cost) so listing reads smaller files.
- Bloom-filter or bitmask for active-spawn check without reading all files.

**Why deferred:** 386ms is acceptable at current scale. The index approach requires careful invalidation under concurrent writers. Design deferred until scale pressure recurs.

---

### D-003: Nested-Depth "After Reconciliation" Wording

**Where:** `meridian doctor` warning message for `live_active_spawns_remain`

**The issue:** When `is_root_side_effect_process()` is `False` (i.e., `MERIDIAN_DEPTH > 0`), reconciliation is skipped entirely. The `live_active_spawns_remain` warning message currently says "after reconciliation" even though reconciliation didn't run. This is technically correct — no reconciliation means all active rows remain — but it's confusing. A reader could infer that reconciliation ran and left these spawns alive, when actually it wasn't attempted.

**The fix:** When `MERIDIAN_DEPTH > 0`, the warning should say "reconciliation skipped at nested depth" rather than "after reconciliation."

**Why deferred:** Out of scope for the initial doctor bugfix. Small wording issue, no functional impact.

---

### D-005: Double Read in `_repair_orphan_runs`

**Where:** `src/meridian/lib/ops/diag.py`

**The issue:** `_repair_orphan_runs()` reads and replays the spawn store to perform reconciliation, then discards the reconciled list. `doctor_sync` then re-reads the spawn store in the next step to get the post-reconcile snapshot for `active_spawn_ids`. This is two reads of `spawns.jsonl` where one would suffice.

> [!FLAG] **Needs human review** — Spawn state migrated to per-spawn
> `state.json` in 2026-05, so this deferred item may be stale or need
> rephrasing around v2 list/read behavior rather than `spawns.jsonl` replay.
> Flagged 2026-05-10.

**The cleaner design:** `_repair_orphan_runs()` returns the refreshed reconciled spawn list directly, and `doctor_sync` uses that list instead of re-reading the store.

**Why deferred:** Minor refactor, no correctness issue. The double-read is fast (JSONL replay) and the code is correct. Deferred to keep the initial bugfix minimal.

---

### v001 `uuid_state_split` Migration — Still a Stub

**Where:** `migrations/v001_uuid_state_split/`

**The issue:** This migration moves legacy runtime state from repo `.meridian/` to user `~/.meridian/projects/<uuid>/`. It was introduced as an intent in 0.0.34 when the dual-root split was deployed. The migration is registered in `migrations/registry.toml` with status `stub` — the framework exists but `migrate.py` is not yet implemented.

**Impact:** Users who had spawn/session state under the old repo-level structure will not have it automatically migrated. Read paths fall back to repo `.meridian/` when no UUID exists, so the old data is not lost. But it won't appear in `meridian spawn list` until migrated.

**What's needed:** Implement `migrations/v001_uuid_state_split/migrate.py`:
1. Find legacy spawn/session JSONL files under `.meridian/`
2. Create the user-level runtime root (UUID creation if needed)
3. Move or copy the JSONL files to `~/.meridian/projects/<uuid>/`
4. Update `.meridian/.migrations.json` to record completion

**Why deferred:** No urgent need — the fallback read path covers existing users. Migration tooling design was deprioritized relative to new feature work.

---

### CLI Subprocess Path: Resolve-Before-Persist Unification

**Where:** `ops/spawn/execute.py`

**The issue:** The REST app and streaming-serve paths use `SpawnApplicationService.prepare_spawn()`, which calls `build_launch_context()` before creating the spawn row (resolve-before-persist, SEAM-1/2/3). This eliminates phantom active rows from resolution failures.

The CLI subprocess path (`ops/spawn/execute.py`) predates this seam. It still calls `lifecycle_service.start()` directly after its own resolution phase. It passes resolved metadata (not `"unknown"` placeholders), so the phantom-spawn risk is lower, but it doesn't go through the shared policy seam.

Phase 8.6 removed `SpawnOperationServices` and inlined cancel/cancel-all root/service resolution, leaving this CLI subprocess path as the remaining spawn-pipeline unification target.

**What's needed:** Unify the CLI subprocess path to use `SpawnApplicationService.prepare_spawn()`, removing the direct `lifecycle_service.start()` call.

**Why deferred:** The current behavior is correct — resolved metadata is passed, no phantom-spawn risk in practice. Unification is architectural hygiene. Deprioritized relative to capability work.

---

## Routing & Model Resolution

### Agent Overlay Follow-Ups (2026-05-05, after agent-model-overrides shipped)

**Where:** `src/meridian/lib/launch/compiler.py`, `src/meridian/lib/config/settings.py`, `src/meridian/cli/config_cmd.py`

These items were scoped out of v1 agent-model-overrides and are documented here so they aren't re-derived from first principles.

1. **`timeout` in agent overlays not yet supported (v2).** `timeout` is explicitly excluded from `[agents.<name>]` in v1. The overlay compiler does not yet thread resolved timeout into `execution_policy.timeout`. Until then, adding `timeout` to overlay config would parse but silently have no effect. (`ExecutionBudget`, the former dual timeout carrier, was deleted in PR #375; timeout is now carried solely on `execution_policy.timeout` in minutes.) See [D72](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface).

2. **List override fields in `model-policies` not yet applied at runtime (v2).** `skills`, `tools`, `disallowed-tools`, `mcp-tools` are rejected with a warning in config overlay model-policies, and in profile model-policies they parse but are not applied. Implementing requires: (a) remove the rejection in overlay validation, (b) implement application in the compiler (merge or replace tool lists), (c) update behavioral spec. The three-state overlay semantics already accommodate this — no schema change needed.

3. **`config set/get/reset` for `agents.*` keys not implemented.** Users must edit TOML files directly. This is consistent with `[workspace]` and `[context]` management today, but `config set agents.tech-lead.model gpt55` would be more ergonomic. Deferred: complexity of teaching the config CLI about nested `agents.*` keys is not justified by the first-use pattern (most users set this once and leave it).

4. **`config show` provenance is coarse: "file" not "which file".** Agent overlay values from `config show` display `[source: file]` without distinguishing user/project/local. Per-file provenance is a v2 enhancement. The `FieldProvenance` data structure already tracks down to `AGENT_OVERLAY_DEFAULT` / `AGENT_OVERLAY_POLICY` — per-file attribution requires threading the file path alongside the value through normalization.

5. **`resolve_policies()` is still the primary caller interface.** The canonical compiler lives in `compiler.py`, but `resolve_policies()` remains a thin wrapper and callers still consume `ResolvedPolicies`. Future cleanup: migrate callers to consume `CompilerResult` or `MaterializedLaunch` directly, then deprecate `resolve_policies()`.

6. **Mars ownership of the compiler (long-horizon).** The `CompilerRequest`/`CompilerResult` contract is plain-data by design to make this transition mechanical if warranted. Revisit if: (a) a second consumer besides Meridian needs agent launch-parameter compilation, or (b) Mars gains runtime/launch semantics. See [D74](../decisions/model-resolution.md#d74-compiler-lives-in-meridian-not-mars-option-c-over-b).

---

### Known Gaps After MR1-MR6 (2026-05-02)

**Where:** `lib/launch/policies.py`, `lib/launch/resolve.py`, `lib/catalog/agent.py`, `cli/main.py`

These items were identified in the mars-capability-packaging final gate (alignment reviewer, p2857) as "partially delivered" — accepted as follow-up work, not blocking.

1. **`MERIDIAN_HARNESS` env var not a policy override (S1.2).** The spec describes `MERIDIAN_HARNESS` as a user-facing harness override. Currently it's spawn-local only (`overrides.py` explicitly ignores it). Before wiring it as a policy override, the spawn-local write should be renamed to `MERIDIAN_SELECTED_HARNESS` to avoid collision with nested launches inheriting the parent's resolved harness. See [decisions.md](../decisions.md) D57.

2. **Layer-by-layer debug routing logging not implemented (S6.2/S6.3).** `--dry-run` output shows final provenance but not the full layer-by-layer resolution trace. Adding a debug log per layer would help diagnose unexpected routing behavior.

3. **Rich error messages for invalid model/harness not implemented (S6.2/S6.3).** Errors like "unknown harness" show a bare message. A richer format (showing what was requested, what was available, and which layer set the value) would reduce debugging time.

4. **Harness fallback informational log and error message not implemented (S9.5/S9.6).** When harness-availability fallback activates, there's no user-visible notification. The spec calls for an INFO log "falling back from X to Y" and a richer error if no fallback is found.

5. **List override fields in `model-policies` accepted but not functional (S3.3/S3.12).** `skills`, `tools`, `disallowed-tools`, `mcp-tools` in the `override:` block are parsed and emit a "not yet supported" warning. Implementing them requires threading list overrides through policy merge — a separate design effort.

6. **`--mode <value>` filter flag for agent listing not implemented (S8.6).** `meridian list` groups agents by mode but doesn't support filtering by mode via flag.

7. **Model routing logic spans multiple modules — extraction recommended.** The refactor reviewer (p2857 final gate) noted that model routing logic is spread across `policies.py`, `resolve.py`, `catalog/models.py`, and `catalog/model_aliases.py`. A dedicated routing module would improve locality and testability.

**Decision context:** [decisions.md](../decisions.md) — D50–D57

---

## Telemetry

### Structural Refinements from v1 Final Gate

**Where:** `src/meridian/lib/telemetry/`, `src/meridian/cli/telemetry_cmd.py`

These structural findings were accepted as non-blocking after the v1 telemetry implementation shipped (tech-lead p3176). They improve long-term maintainability but don't affect v1 correctness or completeness.

1. **Observer ownership cleanup.** `SpawnTelemetryObserver` in `lib/telemetry/observer.py` creates a dependency from `telemetry` back to core lifecycle types. Future cleanup: consolidate lifecycle observer types into a neutral module (e.g. `lib/observers/`) to resolve the `core↔telemetry` module tension.

2. **Telemetry bootstrap deduplication.** Each process entry point should converge on one process-seam telemetry install contract. The startup docs now describe `TelemetryBootstrap.install()`, but the current source still has `setup_telemetry()` call sites in `src/meridian/lib/ops/spawn/api.py` and no `TelemetryBootstrap` class was found during the 2026-05-05 structural check.

> [!FLAG] **Needs human review** — Startup documentation claims operation-layer telemetry setup was removed, while current source still contains operation-layer `setup_telemetry()` calls. Decide whether the docs are ahead of implementation or the remaining call sites are stale. Flagged 2026-05-05.

3. **Reader JSONL parsing deduplication.** `read_events` and `tail_events` in `reader.py` each carry their own line-parsing logic. Future cleanup: extract a shared truncation-tolerant line parser.

> **Resolved:** CLI telemetry directory resolution is no longer deferred. D67
> moved telemetry segments under `<project_runtime_root>/telemetry/`; D68 added
> pre-root CLI buffering with in-place upgrade; D70 added point-in-time
> `--global` discovery across project telemetry directories plus legacy
> `~/.meridian/telemetry/` reads until old segments age out.

**Why deferred:** The remaining items are structural improvements, not correctness issues. v1 smoke tests pass, alignment review shows 40/42 requirements covered, and the two remaining gaps (OBS-TL07 consumer-aware retention) are v2-scoped.

**Decision context:** [architecture/telemetry/overview.md](../architecture/telemetry/overview.md)

---

## Codebase

### Source Simplification — Post-Phase-8.6 Remaining Targets

These non-blocking simplification targets were identified by the refactor reviewer (p5497) during the Phase 8.6 pass (2026-05-08). The spawn-pipeline unification target is recorded in the existing [CLI Subprocess Path](#cli-subprocess-path-resolve-before-persist-unification) entry above; the remaining targets are recorded here so the rationale is not re-derived.

1. **Session reference helper extraction.** Session reference lookups are repeated at call sites rather than extracted into a narrow helper. Extraction would improve locality and make the session lookup contract explicit.

2. **`service_context` / bootstrap carrier collapse.** `service_context` and bootstrap carrier objects carry overlapping state. These can be collapsed once the spawn pipeline is unified.

3. **`CodexConnection` decomposition.** `CodexConnection` accumulates multiple concerns (connection lifecycle, state tracking, output handling). Decomposing it into focused collaborators would improve testability and isolation.

4. **`session_target.py` collapse — deferred pending explicit read/repair design.** `session_target.py` has legacy state-reading paths that need an explicit read/repair contract before collapse. Do not simplify blindly; design the read/repair seam first.

**Why deferred:** These items were out of scope for Phase 8.6, which targeted `SpawnOperationServices`, workspace config loader ownership, and test-suite collapse. These targets compound future maintenance but have no known correctness impact.

---

### Dead Code Cleanup: General Codebase Compatibility Residue (p2596)

**Where:** `cli/main.py`, `cli/format_helpers.py`, `lib/launch/reference.py`, `lib/launch/prompt.py`, `lib/state/spawn_store.py`, `lib/harness/adapter.py`, `lib/launch/streaming_runner.py`, `server/__init__.py`

**Reviewed:** 2026-04-30 (p2596). Read-only full-codebase audit focused on compatibility residue, unused wrappers/aliases, and shim modules. All non-blocking.

**1. Dead CLI helpers in main.py (medium)**

Three symbols in `cli/main.py` have no callers: `_is_spawn_background_request` (:271), `_resolve_session_target` (:545), `_should_startup_bootstrap` (:699). Look like supported seams; are inert.

**Action:** Delete all three.

**2. Dead shim module: cli/format_helpers.py (medium)**

Compatibility shim re-exporting `kv_block` and `tabular`. No in-repo imports from `meridian.cli.format_helpers`. All live imports already go to `meridian.lib.core.formatting`.

**Action:** Delete `cli/format_helpers.py`.

**3. Dead ReferenceFile aliases (medium)**

`ReferenceFile = ReferenceItem` compatibility alias appears in both `lib/launch/reference.py:99-100` and `lib/launch/prompt.py:64-65`. No callsites — only the alias lines and their `__all__` exports.

**Action:** Remove both aliases and drop them from `__all__`.

**4. Dead deprecated wrappers in reference.py (medium)**

- `load_reference_files()` (`lib/launch/reference.py:464-482`) — deprecated wrapper with zero callers; live callsites use `load_reference_items()`.
- `render_reference_paths_section()` (`lib/launch/reference.py:560-579`) — deprecated renderer with zero callers; live code uses `render_reference_blocks()`.

**Action:** Delete both functions and remove their exports.

**5. Assorted dead aliases (low)**

- `_record_from_events = reduce_events` in `lib/state/spawn_store.py:283-284` — no callers.
- `BaseSubprocessHarness = BaseHarnessAdapter` in `lib/harness/adapter.py:418-419` — no callers; misleads future readers.
- `_terminal_event_outcome = terminal_event_outcome` in `lib/launch/streaming_runner.py:1218-1219` — no callers.

**Action:** Delete all three aliases.

**6. Dead server package re-exports (low)**

`server/__init__.py:1-5` re-exports `mcp` and `run_server`, but no in-repo code imports from `meridian.server` at the package level — all imports go directly to `meridian.server.main`.

**Action:** Delete the re-exports or collapse to a package marker only.

---

## Ecosystem: Jupyter Workbench

### Structural Follow-Ups from Final Gate

**Where:** `jupyter-workbench:src/jupyter_workbench/core/`, `jupyter-workbench:src/jupyter_workbench/adapters/`, `microct-analysis:src/microct_analysis/`

These items were accepted as non-blocking after tech-lead spawn p3005 shipped the initial `jupyter-workbench` / `microct-analysis` implementation. They are structural cleanup candidates, not known user-facing failures.

1. **Session manifest policy duplication.** Execution, snapshot, and lineage services repeat session-id resolution and manifest reads. Future cleanup should extract a `SessionStore` / `SessionRepository` boundary instead of letting every service carry manifest policy.

2. **Raw dict DTO fields.** `visualization_delta`, event payloads, and some summary fields use raw mutable dictionaries. Future cleanup should introduce typed DTOs for scene summaries, cell summaries, and event records while preserving CLI/MCP field semantics.

3. **Notebook writes are not atomic.** The per-session mutation lock serializes writes, but `active.ipynb` updates do not yet use tmp+rename. Low risk in current single-session mutation flow; should be made crash-only before heavier concurrent use.

4. **`LineageService.replay()` carries execution mechanics.** Replay currently invokes execution mechanics directly. A `NotebookOutputSerializer` or similar collaborator would keep lineage policy separate from execution details.

5. **Replay does not reconstruct live visualization state.** This is by design: browser/trame state is live process state. Replay should leave visualization `absent`; agents re-execute scene setup cells after replay.

**Decision context:** [ecosystem/jupyter-workbench/architecture.md](../ecosystem/jupyter-workbench/architecture.md) and [ecosystem/jupyter-workbench/runtime-model.md](../ecosystem/jupyter-workbench/runtime-model.md)

---

## State Record

### Regression Snapshot: 2026-04-30 (p2601)

Full verification pass run after the chat backend refactor:

| Check | Result |
|---|---|
| `ruff check .` | ✅ 0 errors |
| `pyright` | ✅ 0 errors, 0 warnings |
| `pytest-llm` | ✅ 1289 passed, 2 skipped, 1 xfailed |

All five targeted smoke tests passed (lifecycle, acquisition, normalizer parity, TUI passthrough, full suite). Build is clean on main as of this date.

---

## Ecosystem: MicroCT Analysis

### PCA Duplication Across Tool and Processing Layers

**Where:** `microct-analysis:src/microct_analysis/tools/orientation_tools.py`, `microct-analysis:src/microct_analysis/processing/orientation.py`

**The issue:** `orientation_tools.py` wraps `processing.orientation.pca_orient`, but both layers carry overlapping PCA logic. The tool layer was added to give agents a clean tool-API surface without importing directly from `processing/`. The duplication is structural, not a correctness issue — both implementations produce the same results.

**What's needed:** Define a single PCA implementation in `processing/`, with `orientation_tools.py` as a thin adapter. The adapter should own frame tracking (`CoordinateFrame`) while processing owns the numerical algorithm.

**Why deferred:** Both work correctly. Unification is structural hygiene. Revisit when a third PCA caller appears in the codebase.

---

### Slice Inspection Shared Primitive Extraction

**Where:** `microct-analysis:src/microct_analysis/tools/slice_renderer.py`, `src/microct_analysis/tools/slice_inspector.py`, `src/microct_analysis/tools/_plane_helpers.py`

**The issue:** `slice_renderer.py` and `slice_inspector.py` share slice-extraction logic through `_plane_helpers.py`, but the two functions differ enough that further abstraction risks over-engineering. Currently `_plane_helpers.py` is an internal module; it's not a stable public API.

**What's needed:** If a third consumer of plane-extraction logic appears, consolidate into a proper `plane_primitives` module with a defined interface.

**Why deferred:** Two consumers don't clearly establish the seam. Premature extraction would create an abstraction for its own sake.

---

### Prompt-Level Executable Tests for `slice-examination-loop`

**Where:** `microct-analysis:skills/slice-examination-loop/SKILL.md`

**The issue:** The `slice-examination-loop` skill has documented behavior contracts but no prompt-level tests that run the agent against a real session. Prompt testing for skill behavior requires an actual jupyter-workbench kernel session with real scan data — a workbench-integrated test harness that doesn't exist yet.

**What's needed:** A prompt testing framework that can launch a workbench session, load a scan, and drive the skill through its paces. The `meridian-prompter` test toolchain covers stateless skills; this one requires stateful kernel context.

**Why deferred:** Infrastructure doesn't exist. Tracked as a capability gap in the prompt testing toolchain.

---

## Cross-References

- [open-questions/mars-feature-gaps.md](mars-feature-gaps.md) — Mars package manager gaps
- [open-questions/process-scope.md](process-scope.md) — process-scope cleanup follow-ups (PROC-004, PROC-007)
- [lessons/state-design-lessons.md](../lessons/state-design-lessons.md) — why dual-root was introduced (v001 context)
- [architecture/state-system.md](../architecture/state-system.md) — migration framework design
- [architecture/process-scope.md](../architecture/process-scope.md) — process-scope ownership design
- [operations/health-checks.md](../operations/health-checks.md) — doctor flow (D-003, D-005 context)
- [decisions.md](../decisions.md) — D27, D29
