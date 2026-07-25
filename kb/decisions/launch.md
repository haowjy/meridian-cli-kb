# Decisions: Launch Composition and Policy

This page owns decisions about preparing, binding, and composing a launch.
Managed-process ownership, session/wait behavior, and harness compatibility now
have separate records so independently changing concerns remain discoverable.

## Decision Map

| Concern | Record |
|---|---|
| Prepare/bind and composition | This page |
| Managed-primary and process ownership | [Launch Process Ownership](launch-process-ownership.md) |
| Wait, goals, output, continue/fork/from | [Launch Session Initiation](launch-session-initiation.md) |
| Harness and platform compatibility history | [Launch Harness Compatibility](launch-harness-compatibility.md) |

## Launch Pipeline

### Single composition seam: all driving adapters call `build_launch_context()`

**Decision:** All three driving adapters (primary CLI, spawn subprocess, CLI streaming-serve) call `build_launch_context()` for all composition. No driving adapter re-derives policy, permissions, prompt, argv, or environment.

**Why:** Separate composition paths drift over time. Security properties (permission resolution, environment sanitization) and behavioral properties (prompt assembly, workspace projection) need a single place to audit. The 13 composition invariants are checkable in one code location.

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
- Split only for the primary path — partial split creates two patterns to maintain; the full split allows all three driving adapters to benefit

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

### Resolve-before-persist for the streaming-serve path

**Decision:** The CLI streaming-serve path calls `build_launch_context()` before creating the spawn row (`SpawnApplicationService.prepare_spawn()`). The spawn row is created only if resolution succeeds.

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

### D-continue-replays-recorded-launch-contract: same-session continue is not live policy recomputation

**Decision (2026-07, work:continue-replay-contract):** Primary `meridian --continue <ref>`
and spawn `meridian spawn --continue <id>` replay the recorded launch contract for
tracked source sessions/spawns instead of resolving a new launch from the caller's
current CWD, environment, or config. The replay boundary is launch-owned:
`continue_replay.py` builds a `ContinueReplayContract` from the resolved source
reference and its persisted `LaunchPolicySnapshot`; primary and spawn continue
consume that same normalized contract.

**Behavior:** Continue inherits the source work attachment and task directory
(`task_cwd`, surfaced as `MERIDIAN_TASK_DIR`) when Meridian recorded them. The
absence of source work is also recorded state: exact continue suppresses the
caller's ambient `MERIDIAN_ACTIVE_WORK_*` instead of attaching it implicitly. When
a `LaunchPolicySnapshot` exists, continue reuses the snapshot's model, harness,
agent, agent opt-out, agent profile, skills, skill paths, loaded skill content,
execution policy, tool/MCP tool policy, terminal surface mode, matched policy rule,
fallback chain, rendered inventory prompt, env, and passthrough args. Legacy
tracked sessions without a snapshot still preserve recorded work/task context and
source harness metadata where present.

Explicit agent opt-out is authoritative. If the source launch opted out of an
agent, replay must keep `agent=None` and avoid falling back to configured default
agents. When there is no opt-out, snapshot replay preserves the source agent
identity even if current config or environment would select a different default.

An empty snapshot model (`model=""`) is valid legacy persisted JSON when Mars
cleared an incompatible model because a higher-precedence harness override won. It
means "no managed model override; let the recorded harness use its default."
In-memory replay represents that absence as `None`; normalization belongs to
policy snapshot replay (`policy_snapshot.py`), not to continue-specific code.
Continue must replay that contract for any harness instead of recomputing from
current config/env or rejecting the snapshot. Empty harness remains invalid.
See [model-resolution: model optional](model-resolution.md#model-optional-empty-model).

**Why:** Continue means re-entering the same conversation. Recomputing from current
CWD/config/env can change the system prompt, projected prompt payload, workspace
projection, task directory instructions, or prompt-cache key even though the user
asked for same-session behavior. That breaks cache locality and can send the agent
to the wrong checkout or worktree.

**Override boundary:** Policy-changing options are rejected on continue: model,
agent, agent opt-out, skills, harness, execution policy, passthrough args,
environment overrides, `--work`, and `--task-dir`. Agent opt-out is a launch
identity mutation because it changes whether configured default agents can fill in.
Changing work location or launch identity belongs to a divergent mode: `--fork`,
`--fork-fresh`, `--from`, or a fresh session.

**Runtime verification:** Probe `spawn:p4871` covered primary continue dry-run,
ambient model/agent drift, primary and spawn mutation rejection (including
`--agent ""` as identity mutation), non-continue smoke launches, ambient task-dir
suppression, and the focused continue regression suite. This verifies the
documented contract at both CLI and operation seams.

**Session ID authority:** Continue/fork consume
`ResolvedSessionReference.authoritative_harness_session_id`, not only the raw value
stored on the source row. Authoritative recovery from session records, spawn rows,
or primary metadata is valid launch input; unverified detected IDs are not. See
[session-reference-resolution.md](session-reference-resolution.md).

**Non-goal:** Untracked raw harness sessions with no Meridian metadata may still
fall back to current launch context; there is no compatibility shim for metadata
Meridian never recorded.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) and
[../architecture/launch-system.md](../architecture/launch-system.md#control_root--task_cwd-split).

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

### Spawn-field coverage from adapter `handled_fields` (supersedes StrategyMap)

**Current decision:** each adapter declares the launch fields it handles, and
coverage validation uses the union of those declarations against the typed
launch spec. Bundle registration and launch-spec guards fail when a field has no
owner.

The earlier decision required a per-adapter `STRATEGIES: StrategyMap` of
`FlagStrategy` entries. That command-assembly generation was deleted; retaining
its exhaustive-coverage goal through `handled_fields` avoids coupling every
field to a flag-strategy abstraction.

See [Harness Abstraction](../concepts/harness-abstraction.md).

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

**Why:** `MERIDIAN_HARNESS_COMMAND` was a backdoor that bypassed the adapter's command assembly, including exhaustive spawn-field coverage and security checks. Any legitimate need to customize a launch command belongs in the adapter's `build_command()` method — where it goes through review, testing, and invariant checking.

**Alternatives rejected:** Keep as an advanced escape hatch — rejected because there was no documented use case that couldn't be served by adapter customization.

---

### D-spawn-capability-gate: Capability-gated spawn knowledge injection (PR #328, 2026-06)

**Decision:** Spawn knowledge (agent inventory + harness-templated spawn usage contract) is injected into the system prompt, gated on the agent profile's `subagents` field. Opt-out via `meridian-capabilities: {spawn: false}`. There is intentionally **no positive `spawn: true` override** — having `subagents` IS the capability.

**Why:** The safety-critical spawn usage contract (`meridian spawn --bg` rules, double-backgrounding warning, `spawn wait` workflow) was previously only in the opt-in `meridian-spawn` skill. Across three separate incidents, the model never loaded that skill — it skipped skill discovery when the prompt surface was saturated or context prioritization buried obscure skills. The consequence was models hitting the double-backgrounding footgun, background-wrap timeout, and lost spawn tracking.

Making the contract always-injected when the agent is spawn-capable eliminates the opt-in gap. The gate on `subagents` ensures leaf agents (which have no spawn delegation to do) don't waste context on spawn instructions they can't use.

**Contract as first-class composition block:** The spawn contract is delivered as `spawn_contract_prompt` — a first-class block in `ComposedLaunchContent`, rendered separately from the inventory string. This ensures continue/fork re-applies it exactly once. If it were appended to the inventory string, snapshot→replay→recompose could duplicate or drop it.

**Harness-templated contracts:** Claude gets a harness-specific contract referencing Claude's `run_in_background`; all other harnesses get a generic contract referencing "your harness's background execution." Policy lives in `launch/spawn_guidance.py`.

**meridian-base v0.7.20:** Removed the default orchestrator and subagent profiles. The orchestrator was the one spawn-capable agent with an empty `subagents` roster, so post-gating it would have shipped a spawn contract with an empty inventory — confusing and wasteful. The profiles may be re-added with proper subagent rosters later.

**Alternatives rejected:**
- Keep spawn contract in the skill only — three real incidents proved the model won't load it reliably.
- Add a positive `spawn: true` override in `meridian-capabilities` — rejected because it creates two sources of truth (`subagents` roster AND a boolean flag). `subagents` already captures the intent: if you list subagents, you can spawn.
- Append contract to inventory string — rejected because snapshot round-trip would duplicate/drop it.

See [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md#spawn-capability-gating).

---

### D-reserve-before-prep: Reserve spawn row before preparation for crash recovery (c2076, PR #328, 2026-06)

**Decision:** Background spawns write a durable `queued` row **before** preparation (model/harness resolution, prompt composition, workspace projection). Preparation succeeds → announce the row (create work item, emit lifecycle/telemetry/subrun events). Preparation fails gracefully → nothing to clean up (no work item, no events).

**The crash window being closed:** A launcher SIGKILLed during preparation — after work-item creation but before the spawn row was written — left a forever-invisible spawn. The work item existed but `spawn list` showed nothing; the agent never knew it launched. By reserving the row first, even a mid-prep kill leaves a reconcilable `queued` row that the existing reaper handles via `missing_runner_pid`.

**Two-phase reserve/announce:**
1. **Reserve** — `lifecycle.start(dispatch_events=False)` writes only the spawn row (status `queued`). No work item, no events, no telemetry.
2. **Announce** — `lifecycle.announce_started()` + `_emit_spawn_start_subrun_event()` runs only after preparation succeeds. Creates the work item and emits lifecycle/telemetry/subrun events.

If preparation fails anywhere, the reserved row is cleaned up via `spawn_store.remove_spawn_events()` (rmtree of the spawn directory) before returning the failure — graceful prep failures are side-effect-free: no leaked work items, no phantom start events.

**The rejected "detach-first" reorder:** The original implementation reordered `Popen` before row creation (~9ms theoretical gain, as preparation typically completes before the detached worker process starts). This was rejected for two correctness regressions:
1. Work-item creation and start events leaked on graceful preparation failure — the earlier setup steps had already run.
2. The `announce=False` sentinel couldn't retroactively suppress already-emitted events.

The two-phase split (reserve → announce) fixes both: reserve creates *nothing* but the row; announce emits everything only on confirmed prep success.

**Detached worker survivability:** The background worker uses `Popen` with `start_new_session=True` (POSIX) / `DETACHED_PROCESS` (Windows), so it survives the launcher being killed. The only irreducible window is between row write and Popen — if killed exactly there, the row exists but no worker will claim it. The reaper resolves this via `missing_runner_pid`.

**Reconciler behavior unchanged:** No new states, no reaper changes, no schema changes. The existing `missing_runner_pid` path covers the reserved-but-unclaimed row.

See [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md#reserve-before-prep-crash-window).

---

## Related

- [Launch architecture](../architecture/launch-system.md)
- [Launch concepts](../concepts/composition-pipeline.md)
- [Decision index](../decisions.md)
