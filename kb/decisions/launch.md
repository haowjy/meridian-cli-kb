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

**Why:** The primary CLI path previously called the full factory twice — once for `--dry-run` preview, once for real execution. The split makes the prepare-once/bind-N pattern explicit and enforced.

**`PreparedLaunchSurface`** — public frozen dataclass carrying everything spawn-stable (resolved request, harness, composition warnings, agent inventory, context prompt). Explicitly excludes runtime-only values (spawn ID, report path, env, argv, permissions). The exclusion is the invariant — bind-phase code cannot accidentally depend on preparation-phase values.

**`RuntimeBindings`** — frozen dataclass for bind-phase inputs: `spawn_id`, `report_output_path`, `runtime_work_id`, `chat_id`, `forked_harness_session_id`, `plan_overrides`, `dry_run`.

See [architecture/launch-system.md](../architecture/launch-system.md) — Prepare/Bind Split.

---

### CatalogSession: operation-scoped Mars result cache

**Decision (2026-05):** `CatalogSession` is an operation-scoped collaborator holding a `MarsResultCache`. Created once per launch operation, discarded at operation end. `prepare_launch_surface()` receives one from its caller.

**Why:** Without this, `resolve_policies()` and `prepare_launch_surface()` called `mars models list` independently on every invocation (~100ms subprocess per call, 3–5 redundant calls per spawn).

---

### D-control-root-task-cwd-split: Split execution_cwd into control_root + task_cwd

**Decision (2026-05-13, PR #210):** `LaunchContext`, spawn records, and session start events carry two distinct path fields — `control_root` (project config/authority root, anchored to `meridian.toml`) and `task_cwd` (task's intended working directory, `None` when it coincides with control_root).

**Why:** `execution_cwd` was overloaded — one field served double duty as both config authority and task directory. When a spawn originates from a nested directory these concerns diverge: config authority stays at repo root while the agent operates in the subdirectory.

**Behavior (updated 2026-06, PR #318):** When `task_cwd` differs from `control_root`, `bind_launch_context()` sets `MERIDIAN_TASK_DIR` env var and injects a system prompt block stating the task directory.

**continue/fork authority:** `ClaudeSessionAccessSource` carries `source_control_root` and `target_control_root` instead of `source_cwd`. Legacy records fall back to current launch `control_root`.

**Extended by PR #248 (2026-05-22):** Full authority/task domain architecture with worktree-based task_cwd resolution. See [spawn-cwd-worktree-anchor.md](spawn-cwd-worktree-anchor.md).

---

### Resolve-before-persist for the streaming-serve path

**Decision:** The CLI streaming-serve path calls `build_launch_context()` before creating the spawn row. The spawn row is created only if resolution succeeds, preventing phantom active rows when resolution fails.

**Known gap:** The spawn subprocess path still creates the row before resolution. Unification remains a follow-up.

---

### Layered finalization ownership (R4-R6 refactor)

> Supersedes the earlier `pre_launch_complete` boolean flag approach.

**Decision:** Finalization responsibility is divided across three concentric function scopes:
1. **Runner layer** (`execute_with_streaming()`): Owns terminal finalization from function entry. Uses sentinel locals (initialized to `None`, set during setup) so `finally` handles partial-setup failures.
2. **Helper layer** (`launch_prepared_spawn()`): Shared pre-run setup for both foreground and background paths. Owns `launch_failure` finalization for pre-execution exceptions.
3. **Surface backstop layer**: Last-resort catch ensuring no spawn is left permanently stuck in `queued`.

**Why structural layers over boolean flags:** Each layer is defined by function scope, not flag state. `launch_prepared_spawn()` eliminates code duplication between foreground and background paths.

**`complete_spawn()` idempotency** enables safe layered catches — if the record is already terminal, the call is a no-op. Known issue #153: concurrent finalizers silently drop the second set of metrics.

---

### D-primary-approval: Managed-primary Codex approval routing

*2026-05-05*

**Decided:** Managed-primary Codex sessions use `InteractiveHandler` (passive — surfaces requests as events). Spawn/subagent paths keep `AutoAcceptHandler`. The blanket `_primary_observer_mode` rejection of all server requests is removed.

**Why:** In managed-primary mode, the app-server routes `requestApproval` to the turn owner (Meridian), not the TUI. The TUI cannot answer approval requests; Meridian must handle them via the interactive handler.

**Three defects fixed:** (D-PA1) blanket rejection removed, (D-PA2) managed-primary uses InteractiveHandler, (D-PA3) `_reject_confirm_mode_approval_request()` conditioned on `no_runtime_hitl`.

---

### D-model-invocable: Filter at inventory prompt boundary, not at catalog scan

**Decision (2026-05, PR #208; rewritten 2026-06, PR #314):** `model-invocable: false` in agent profile frontmatter is enforced by Mars at inventory render time. The catalog scan returns all profiles regardless.

**Why (2026-06 rewrite):** Inventory rendering is now exclusively Mars-owned — Meridian passes through the bundle's `prompt_surface.inventory_prompt` verbatim. No Python fallback renderer.

**Original Why (still holds):** `scan_agent_profiles()` has multiple consumers. Filtering there conflates model-facing visibility with general catalog availability. The inventory prompt is the only model-facing boundary.

---

### D-model-invocable-vs-user-invocable: Two separate visibility surfaces

**Decision (2026-05, PR #208):** `model-invocable` controls only the model-facing agent inventory prompt. It does not restrict explicit user invocation via `meridian spawn -a <name>`.

**Why:** A deprecated or internal agent that shouldn't appear in the model's menu may still be explicitly invoked by a user who knows what they're doing.

---

### D-mars-owns-inventory: Mars renders agent inventory; Meridian embeds verbatim

**Decision (2026-06, PR #314):** The agent inventory prompt is bundle-only. Mars renders `prompt_surface.inventory_prompt`. Meridian embeds it verbatim. There is no Python fallback renderer.

**Why single-owner:** A dual-renderer creates divergence risk. Mars already owns the `.mars/` directory schema, profile scanning, and agent-copy; inventory rendering is a natural extension.

**Resume/snapshot persistence:** `bundle_inventory_prompt` on `LaunchPolicySnapshot` so inventory survives without re-calling mars.

---

### D-continue-replays-recorded-launch-contract: same-session continue is not live policy recomputation

**Decision (2026-07):** Primary `--continue` and spawn `--continue` replay the recorded launch contract for tracked source sessions/spawns instead of resolving a new launch from current CWD/env/config.

**Behavior:** Continue inherits source work attachment and task directory. Absence of source work is also recorded state: exact continue suppresses the caller's ambient `MERIDIAN_ACTIVE_WORK_*`. When a `LaunchPolicySnapshot` exists, continue reuses the snapshot's model, harness, agent, skills, execution policy, tool policy, terminal surface mode, env, and passthrough args. Explicit agent opt-out is authoritative — replay preserves `agent=None` without falling back to configured defaults.

**Why:** Continue means re-entering the same conversation. Recomputing from current CWD/config/env can change the system prompt, workspace projection, or prompt-cache key.

**Override boundary:** Policy-changing options are rejected on continue: model, agent, skills, harness, execution policy, passthrough args, env overrides, `--work`, `--task-dir`. Changing identity belongs to `--fork`, `--fork-fresh`, `--from`, or a fresh session.

**Session ID authority:** Continue/fork consume `ResolvedSessionReference.authoritative_harness_session_id`. See [session-reference-resolution.md](session-reference-resolution.md).

---

### D-headless-claude-deny: Headless Claude denied by default with 2026-06-15 driver

**Decision (2026-06, PR #314):** Meridian denies headless Claude by default. `[spawn] deny_headless_harnesses = ("claude",)` ships in auto-scaffolded `meridian.toml`. Users who override receive a startup warning.

**Why deny-by-default:** Headless Claude `-p` is an Anthropic API surface with announced deprecation. Meridian cannot fix a broken headless path — the deny default gives users a clear Meridian error with recovery guidance rather than an opaque API rejection.

---

### D-agent-copy-key: Mars `settings.meridian.agent_copy` key renamed; auto-scaffolded

**Decision (2026-06, PR #314):** The Mars config key was renamed to `[settings.meridian.agent_copy] harnesses = ["claude"]` (was `[settings.agent_copy]`). Agent copy is a Meridian concept, so it belongs under `settings.meridian`.

**Auto-scaffold:** New projects get the key written into initial `mars.toml`. Meridian only reads the new key path.

---

### D-skill-doc-suppression: Skip supplemental_documents for native-skill harnesses

**Decision (2026-05, PR #208):** When a harness declares `supports_native_skills=True`, composition produces empty `supplemental_documents` instead of calling `compose_skill_prompt_documents()`.

**Why:** All three active harnesses declare `supports_native_skills=True`. Before this change, Meridian injected skill content into `supplemental_documents` for all of them — duplicating content that Mars already delivers through native channels.

**Two delivery channels:** `supplemental_documents` (suppressed for native-skill harnesses) and `compose_skill_injections()` via `--append-system-prompt` (preserved for Claude).

---

### Semantic IR + adapter projection for prompt composition

**Decision:** The launch factory produces `ComposedLaunchContent` (harness-agnostic semantic IR), and each harness adapter's `project_content()` maps this to `ProjectedContent` (harness-specific channel assignments). No conditional branches in composition for per-harness routing.

**Why:** Adding a new harness should mean implementing an adapter, not editing composition code. The old `reference_input_mode` capability flag was removed when it proved insufficient.

See [concepts/composition-pipeline.md](../concepts/composition-pipeline.md).

---

### Spawn-field coverage from adapter `handled_fields` (supersedes StrategyMap)

**Decision:** Each adapter declares the launch fields it handles. Coverage validation uses the union of declarations against the typed launch spec. Bundle registration and launch-spec guards fail when a field has no owner.

The earlier `STRATEGIES: StrategyMap` of `FlagStrategy` entries was deleted; `handled_fields` retains its exhaustive-coverage goal without coupling every field to a flag-strategy abstraction.

See [Harness Abstraction](../concepts/harness-abstraction.md).

---

### Config precedence: CLI flag > ENV var > agent profile > project config > user config > harness default

**Decision:** Every config field resolves independently through a fixed precedence chain. A CLI model override also forces harness re-derivation from the overridden model.

**Why:** Different fields have different override sources. Independent resolution lets each field resolve to the highest-precedence source. Without forced re-derivation, `-m codex` with `harness: claude` in the profile silently uses the wrong harness.

---

## Harness Identity and Environment

### D32: `MERIDIAN_HARNESS` is a one-hop env var; does not cascade to grandchildren

**Decision:** `build_launch_context()` writes `MERIDIAN_HARNESS = harness.id.value` into every spawned process's environment. NOT in `ALLOWED_CHILD_ENV_KEYS` — does not propagate to grandchildren. Each spawn level gets its own value from its own harness resolution.

**Why:** A grandchild may use a different harness. Inheriting the grandparent's value would silently override the grandchild's resolved harness.

---

### D33: Wait-yield uses the orchestrator's own `MERIDIAN_HARNESS`, not child spawn rows

**Decision:** `spawn_wait_sync()` reads `os.getenv("MERIDIAN_HARNESS")` — the orchestrator's own harness — for yield interval. All harnesses default to 3000 seconds (50 minutes).

**Why:** The yield protects the orchestrator's prompt-cache context window. Reading child spawn rows was wrong — a mixed-harness wait set would use the shortest child interval, but the cache being preserved is the parent's.

---

### D57: `MERIDIAN_HARNESS` is spawn-local, not a user-facing policy override

**Decision:** `MERIDIAN_HARNESS` is informational spawn-local. NOT consumed as a policy override by `resolve_policies()`.

**Why:** If treated as a policy override, nested launches would inherit the parent's resolved harness, bypassing their own profile and config.

---

### D63: `MERIDIAN_HARNESS_COMMAND` removed

**Decision:** The env var override mechanism was removed. Harness adapters are the only launch path.

**Why:** Bypassed adapter command assembly including spawn-field coverage and security checks. No documented use case couldn't be served by adapter customization.

---

### D-spawn-capability-gate: Capability-gated spawn knowledge injection (PR #328, 2026-06)

**Decision:** Spawn knowledge (agent inventory + harness-templated spawn usage contract) is injected into the system prompt, gated on the agent profile's `subagents` field. Opt-out via `meridian-capabilities: {spawn: false}`. No positive `spawn: true` override — having `subagents` IS the capability.

**Why:** The spawn usage contract was previously only in the opt-in `meridian-spawn` skill. Across three incidents, the model never loaded that skill — context prioritization buried it. Making the contract always-injected eliminates the opt-in gap. The gate on `subagents` ensures leaf agents don't waste context on spawn instructions.

**Contract as first-class composition block:** Delivered as `spawn_contract_prompt` — a first-class block in `ComposedLaunchContent`, rendered separately from inventory to prevent snapshot round-trip duplication.

---

### D-reserve-before-prep: Reserve spawn row before preparation for crash recovery (PR #328, 2026-06)

**Decision:** Background spawns write a durable `queued` row **before** preparation. Preparation succeeds → announce (work item, lifecycle events). Preparation fails → reserved row cleaned up, side-effect-free.

**The crash window being closed:** A launcher SIGKILLed during preparation could leave a forever-invisible spawn (work item exists but `spawn list` shows nothing). Reserving the row first leaves a reconcilable `queued` row the reaper handles via `missing_runner_pid`.

**Two-phase reserve/announce:** Reserve writes only the spawn row. Announce creates work item and emits events only after preparation succeeds.

---

## Related

- [Launch architecture](../architecture/launch-system.md)
- [Launch concepts](../concepts/composition-pipeline.md)
- [Decision index](../decisions.md)
