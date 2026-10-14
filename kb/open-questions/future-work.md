# open-questions/future-work — Deferred Items and Known Gaps

Tracked follow-ups deferred from implementation or design sessions. Each entry names the issue, the context, and why it was deferred. This is not a backlog — it's a record of known state so future work doesn't re-derive the same conclusions.

Items are organized by domain.

---

## Managed Primary

### Live primary injection control endpoint (#498)

**Where:** managed primary launch/attach and `spawn inject` control routing.

**The issue:** OpenCode's connection adapter can project a later HTTP prompt using
the committed native agent/model/variant, but a running native primary TUI does
not expose a public control endpoint to `spawn inject`. A native probe therefore
could not reach the adapter path while the primary was live; it failed after the
primary ended. This is an endpoint/ownership gap, not evidence that model
projection reset the native choice.

**Boundary:** #498 owns any public primary control surface. Do not hide that work
inside #497 or claim public-primary injection coverage from adapter-level tests.
Unsent TUI-local choices remain unobservable, and GET followed by POST remains
non-atomic against simultaneous native submission.

**Related:** [OpenCode managed-primary model commitment](../codebase/harness-adapters.md#opencode-managed-primary-model-commitment)
documents the adapter-level boundary; [launch-system.md](../architecture/launch-system.md)
documents primary launch composition.

**Provenance:** `work:next-minor-planning/probes/followup-497-native/REPORT.md`;
`spawn:p6060`.

---

## State & Spawn

### Process-Scope Cleanup Gaps

Process-scope deferred work moved to [process-scope.md](process-scope.md) so the detailed PROC-004 and PROC-007 notes live with their shared containment/reaper context.

### Restore metadata hash still depends on the reader's models (#537)

**Status:** Open, [#537](https://github.com/haowjy/meridian-cli/issues/537).
`restored-from.json` stores `session_sha256` as a hash of the folded
`SessionRecord`, so a runtime that 0.6.7 restored into fails a repeat restore under
the new build ("Restored metadata changed"). It is the same model coupling that the
[portable-digest decision](../decisions/history-storage.md#d-history-portable-digest-stored-json)
removed for record digests. Fix: hash the stored `historical_import` session event
JSON. Reads and a first restore are unaffected.

### Spawn-list scaling

**Status:** Deferred until scale pressure recurs. Individual spawn reads are
O(1), while listing is O(spawns) file reads. The measured migration dataset and
timings live only in [Architecture: State System](../architecture/state-system/spawn-state.md); this page does not duplicate them.

An index would require invalidation rules under concurrent writers, so the
complexity is not justified without renewed evidence.

### D-003: Nested-Depth "After Reconciliation" Wording

**Where:** `meridian doctor` warning message for `live_active_spawns_remain`

**The issue:** When `is_root_side_effect_process()` is `False` (i.e., `MERIDIAN_DEPTH > 0`), reconciliation is skipped entirely. The `live_active_spawns_remain` warning message currently says "after reconciliation" even though reconciliation didn't run. This is technically correct — no reconciliation means all active rows remain — but it's confusing. A reader could infer that reconciliation ran and left these spawns alive, when actually it wasn't attempted.

**The fix:** When `MERIDIAN_DEPTH > 0`, the warning should say "reconciliation skipped at nested depth" rather than "after reconciliation."

**Why deferred:** Out of scope for the initial doctor bugfix. Small wording issue, no functional impact.

---

### D-005: Double Read in `_repair_orphan_runs` (stale framing)

**Where:** `src/meridian/lib/ops/diag.py`

**The issue:** `_repair_orphan_runs()` reads the spawn store to perform reconciliation, then `doctor_sync` re-reads to get the post-reconcile snapshot. In v2, each read is O(1) per spawn file rather than an O(n) JSONL replay, so the performance concern is diminished. The structural concern (two reads where one suffices) remains a minor cleanup target.

**Why deferred:** Minor refactor, no correctness issue. The v2 per-spawn reads are fast.

---

### v001 `uuid_state_split` Migration — Manual Helper, Not an Active Fallback

**Where:** `migrations/v001_uuid_state_split/`

**The issue:** Repo-local `.meridian/` is no longer an active runtime-state root.
`migrations/v001_uuid_state_split/migrate.py` contains a manual copy helper for moving
legacy runtime state to the user project root, but `migrations/registry.toml` still
marks v001 as `stub` and production startup does not discover or run this registry.

**Impact:** Legacy repo-local spawn/session files do not appear on current read paths
until copied into the user runtime root. The remaining `.meridian/id` compatibility
path migrates project identity only; it does not make repo-local runtime files active.

**What's needed:** Decide whether to validate and expose v001 as a supported manual
migration or remove the dormant registry path. If retained, align the helper with the
current project-ID model and atomic-write policy before marking it ready.

**Why deferred:** Migration tooling design was deprioritized relative to active state
work; there is no transparent legacy runtime fallback.

---

### CLI Subprocess Path: Resolve-Before-Persist Unification

**Where:** `ops/spawn/execute.py`

**The issue:** The streaming-serve path uses `SpawnApplicationService.prepare_spawn()`, which composes before creating the spawn row (resolve-before-persist, SEAM-1/2/3).

The CLI subprocess path (`ops/spawn/execute.py`) also composes before the row now (`build_create_payload()`), so it has no phantom-row risk, but it reserves the row with its own `lifecycle_service.start()` call instead of going through the shared policy seam. The background worker recomposes after the row exists.

Phase 8.6 removed `SpawnOperationServices` and inlined cancel/cancel-all root/service resolution, leaving this CLI subprocess path as the remaining spawn-pipeline unification target.

**What's needed:** Unify the CLI subprocess path to use `SpawnApplicationService.prepare_spawn()`, removing the direct `lifecycle_service.start()` call.

**Why deferred:** The current behavior is correct. Unification is architectural hygiene, and it is subsumed by the planned shared turn host (launch#1, phase 4 of the [cleanup program](../codebase/cleanup-program.md)).

---

## Streaming & Completion

### Resident Nudge Serialization and Test Seeding (#430)

**Where:** `src/meridian/lib/streaming/resident_drain.py:181-190`, `src/meridian/lib/streaming/drain_plan_factory.py:105-123`, `tests/integration/launch/test_spawn_manager_lifecycle.py:243,330`

**The issue:** Two residuals from the #322 re-audit (PR #375):

1. **Resident nudges bypass the serialized inject seam.** `resident_drain.py` calls `ResidentBackendControl.begin_followup_turn()` directly; the resident plan does not receive the `SerializedInject` callback from `drain_plan_factory.py`. Pi done nudges route through the factory's manager-owned callback, but resident nudges use a separate path. Unification belongs to the future completion-profile/factory redesign, not the seam extraction done in PR #375.

2. **Test `_sessions` seeding.** `test_spawn_manager_lifecycle.py` directly seeds `manager._sessions` to set up integration scenarios. This is a private-field dependency; the streaming test-suite reshaping deferred from #372 should replace it with the `PiDrainScenario` builder or a manager-owned setup path.

**Why deferred:** Both are known residuals deliberately left by PR #375. The nudge unification requires a completion-profile redesign; the test seeding belongs to a broader streaming test-suite reshaping.

---

### Store-Level Publication Ordering (#431)

**Where:** `src/meridian/lib/streaming/spawn_manager.py`, `src/meridian/lib/launch/streaming_runner.py`

**The issue:** Store-level `published_at` can trail drain cleanup rows. The probe-fix
cycle measured 56s between death and store publication on real Pi (p5390: cleanup
completed at 14:20:20.545Z, `published_at` at 14:20:21.810Z). The sub-second
`published_at` timestamp (added in #435) makes this ordering observable. Moving
durable publication ahead of cleanup is a separate store-pipeline concern.

**Resolved in probe-fix cycle:**
- Item 1 from the prior version of this entry (`prefer_drain_outcome` conflation)
  was fixed: `prefer_drain_outcome` was replaced with an explicit
  `resolve_terminal_outcome()` priority resolver and a manager-owned
  `_publish_terminal` idempotent barrier.
- Item 2 (publication timing lag) was partially addressed: `_publish_terminal`
  runs before cleanup, so the completion future resolves immediately. Store-level
  `published_at` still trails because the runner's `complete_execution()` runs
  after `stop_spawn()`.

**Remaining:**
- `streaming_runner.py` remains large; timeout state spans arbitration, drain
  normalization, failure projection, and lifecycle finalization. Extraction
  of turn-result normalization would prevent another ad-hoc boolean branch.

**Why deferred:** The probe-fix cycle resolved the structural conflation.
Store-level ordering is a pipeline concern tracked as #431 evidence.

---

### Wait Fail-Fast Reporting (#458)

**Where:** `src/meridian/lib/ops/spawn/api.py` — `spawn_wait_sync()`

**The issue:** `spawn wait` blocks until all pending spawns reach a terminal
state. When a denied or failed spawn finalizes quickly (e.g. after the
headless-deny fix in PR #460), the healthy spawns in the wait set continue
running. The caller sees the failure only after the entire wait set drains.
A fail-fast mode would report the first terminal failure immediately without
waiting for healthy spawns to complete.

**What's needed:** An eager/fail-fast option on `spawn wait` (and
`spawn_wait_sync`) that returns as soon as any spawn in the wait set reaches a
terminal failure status. Checkpoint mode already returns partial state, but it is
timer-driven, not event-driven. Fail-fast is conceptually distinct: it reacts to
a specific failure, not to a timer expiry.

**Why deferred:** Scoped out of PR #460 (#442 fix) as a separate concern.
The deny-at-prepare fix eliminates the most common trigger (denied spawns no
longer reach the wait set), reducing urgency. Filed as issue #458.

**Decision context:** `decisions-bugfix-sprint.md` in the workstream-roadmap work
directory.

---

### Typed Terminal Provenance (#438)

**Where:** `src/meridian/lib/streaming/pi_completion_profile.py`, `src/meridian/lib/harness/connections/pi_rpc.py`

**The issue:** `TerminalEventOutcome` loses provenance when
`error/connectionClosed` is normalized. The probe-fix cycle narrowly addressed
the Pi exit-classification defect by recognizing the canonical
`PI_SUBPROCESS_EXIT_ERROR_PREFIX` string in precedence logic. A typed terminal
origin (`pi_subprocess_exit`, transport failure, provider failure, timeout)
would make precedence explicit and delete string-prefix policy.

**Why deferred:** String-prefix recognition is correct and scoped. A typed
provenance taxonomy is a larger contract revision. Filed as #438.

---

### Cleanup Telemetry Categories (#439)

**Where:** `src/meridian/lib/streaming/pi_drain.py`, lifecycle phase events

**The issue:** `cleanup_running`/`cleanup_completed` lifecycle phases do not
distinguish connection teardown from tracked-work cleanup. The probe-fix root
cause analysis showed this ambiguity prevents over-reading cleanup telemetry.
A reason/category field on cleanup phases would make the distinction
observable.

**Why deferred:** Contract revision, not a correctness fix. Filed as #439.

---

---

## Routing & Model Resolution

### Agent Overlay Follow-Ups (2026-05-05, after agent-model-overrides shipped)

**Where:** `src/meridian/lib/launch/compiler.py`, `src/meridian/cli/config_cmd.py`; overlay config itself now lives in `mars.toml` (Meridian's loader rejects `[agents]`)

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

1. **`MERIDIAN_HARNESS` env var not a policy override (S1.2).** The spec describes `MERIDIAN_HARNESS` as a user-facing harness override. The spawn-local write has been renamed to the internal `_MERIDIAN_HARNESS` (PR #388), so the name is free, but nothing reads a plain `MERIDIAN_HARNESS` yet. Wiring it into `RuntimeOverrides.from_env()` is the remaining work. See [decisions.md](../decisions.md) D57.

2. **Layer-by-layer debug routing logging not implemented (S6.2/S6.3).** `--dry-run` output shows final provenance but not the full layer-by-layer resolution trace. Adding a debug log per layer would help diagnose unexpected routing behavior.

3. **Rich error messages for invalid model/harness not implemented (S6.2/S6.3).** Errors like "unknown harness" show a bare message. A richer format (showing what was requested, what was available, and which layer set the value) would reduce debugging time.

4. **Harness fallback informational log and error message not implemented (S9.5/S9.6).** When harness-availability fallback activates, there's no user-visible notification. The spec calls for an INFO log "falling back from X to Y" and a richer error if no fallback is found.

5. **List override fields in `model-policies` accepted but not functional (S3.3/S3.12).** `skills`, `tools`, `disallowed-tools`, `mcp-tools` in the `override:` block are parsed and emit a "not yet supported" warning. Implementing them requires threading list overrides through policy merge — a separate design effort.

6. **`--mode <value>` filter flag for agent listing not implemented (S8.6).** `meridian list` groups agents by mode but doesn't support filtering by mode via flag.

7. **Model routing logic spans multiple modules — extraction recommended.** The refactor reviewer (p2857 final gate) noted that model routing logic is spread across `policies.py`, `resolve.py`, `catalog/models.py`, and `catalog/model_aliases.py`. A dedicated routing module would improve locality and testability.

**Decision context:** [decisions.md](../decisions.md) — D50–D57

---

## Pi Result Delivery Limits {#pi-result-delivery-limits}

Known limits of the tool-boundary delivery change (PR #560, shipped in v0.9.4). Aggregate
message size and report-read concurrency are bounded by the change itself (32 members,
64 KiB, sequential reads) and are not deferred. None of the items below is a
reproduced defect or a must-do:

- **Unadmitted queue claim.** A watcher message queued for native steer or
  follow-up that Pi never admits (for example a cleared queue) keeps its IDs
  reserved, and because queueing closes the current opportunity it can also pause
  unrelated automatic publication until a real native lifecycle callback, reload or
  restart; timer and watcher wakeups do not reopen it. This predates boundary steer; confirm a real
  reproduction before designing a release path.
- **All-history scan per boundary.** Fresh canonical discovery at each tool
  boundary reads every project spawn row. Revisit only if boundary latency shows up
  in real parent/child workflows. Removal trigger: a measured boundary cost or a
  safe incremental index that does not become authority.
- **Cross-cutting efficiency audit.** Retained runner streams, doubled Bash log
  writes, per-drain history polling, cold CLI process cost and old projection
  namespaces are tracked in
  [issue #561](https://github.com/haowjy/meridian-cli/issues/561) alongside #414,
  #496, #500 and #473. Constraints carry over: an inventory is not proof of
  reclaimability, a foreign negative cache needs proof, and old-schema files remain
  mixed-version authority constraints. No production cleanup is authorized.
- **Foreground attachment ownership (managed-Bash).** Foreground model and human
  execution paths still duplicate settlement, abort/start-stop hooks and
  foreground-count UI bookkeeping. Consider one attachment owner only if it
  deletes that bookkeeping while preserving distinct model/human output APIs,
  physical stop versus publication, and reload/background handoff. Launch already
  separates the owned process from its creation observation, so this is only an
  optional structural cleanup, not needed for any correctness behavior.
- **Post-shell polling cost (managed-Bash).** Groups whose descendants outlive the
  shell are polled on a short cadence. #561 has only exploratory backoff
  measurements, not full-runtime or macOS qualification; qualify output drain,
  cancellation, ps failure, publication delay and ownership retirement before
  changing cadence, and never turn observed host lag into a delivery bound.
- **Large backlogs.** A backlog beyond one packet takes several genuine native
  opportunities, one message each; fair allocation fixed the avoidable per-result
  amplification, so only the unbounded-backlog case remains, with no
  constant-total-inference promise.

Provenance: `work:pi-result-delivery`. Source details:
`src/meridian/pi_runtime/.context/delivery-contract.md`.

## Telemetry

### Structural Refinements from v1 Final Gate

**Where:** `src/meridian/lib/telemetry/`, `src/meridian/cli/telemetry_cmd.py`

These structural findings were accepted as non-blocking after the v1 telemetry implementation shipped (tech-lead p3176). They improve long-term maintainability but don't affect v1 correctness or completeness.

1. **Observer ownership cleanup.** `SpawnTelemetryObserver` in `lib/telemetry/observer.py` creates a dependency from `telemetry` back to core lifecycle types. Future cleanup: consolidate lifecycle observer types into a neutral module (e.g. `lib/observers/`) to resolve the `core↔telemetry` module tension.

2. **Telemetry bootstrap deduplication.** Operation and background-spawn entry
points still call `setup_telemetry()` after they know the logical owner and
runtime root. A future change may converge these on one process seam, but the
current [startup architecture](../architecture/startup-pipeline.md#telemetry-as-shipped)
documents the distributed installation accurately; no `TelemetryBootstrap`
class exists today.

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
- [architecture/state-system/overview.md](../architecture/state-system/overview.md) — migration framework design
- [architecture/process-scope.md](../architecture/process-scope.md) — process-scope ownership design
- [operations/health-checks.md](../operations/health-checks.md) — doctor flow (D-003, D-005 context)
- [decisions.md](../decisions.md) — D27, D29
