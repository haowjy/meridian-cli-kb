# Decisions

Decision records explain why durable architectural choices were made — alternatives considered, constraints that drove the choice, and when revisiting is warranted. Architecture pages describe mechanism; decision pages explain the reasoning behind it.

See [decisions/overview.md](decisions/overview.md) for naming conventions and how to add new decisions.

---

## Foundational Decisions

These cross-cutting decisions shape the entire system. Everything else inherits their constraints.

**JSONL append-only event stores over SQLite or mutable JSON** (session state; spawn state migrated in 2026-05)
Session state is an append-only JSONL event log. Spawn state was migrated in 2026-05 to per-spawn `state.json` files for O(1) read performance (12–13s → 0.67s primary launch). The crash-only and files-as-authority principles apply to both formats. See [state decisions](decisions/state.md).

**Spawn state v2: per-spawn state.json with locked mutation seam (2026-05, superseded two-tier model in 2026-07)**
Global `spawns.jsonl` replaced with `spawns/<id>/state.json`. Lazy one-shot migration on first access, no quiescence gate. The original two-tier write model (owner writes without lock / external writes with lock) was collapsed in PR #422 (2026-07) into a single locked mutation seam: every mutation calls `write_state_locked()` under a stable per-spawn lock at `locks/spawns/<id>.lock`. See [state decisions](decisions/state.md#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05).

**Concurrency by construction: mutate-under-lock seams as THE store contract (PR #422, 2026-07)**
All stores (spawn records, archived spawns, work items, hook intervals, scope projections, autosync) use the same mutate-under-lock shape: acquire a stable lock, re-read current state, apply a pure mutator, write atomically. These seams are behavior-preserving contracts a planned future store rewrite inherits. Lock-order invariants: `spawns_flock` before per-spawn lock before scope-projection lock. Orphaned per-spawn lock inodes are GC'd via a validated-exclusive unlink seam (PR #444, closes #427); cleaned session locks use the same primitive. Project-lifetime shared/exclusive gate prevents pruning from destroying live runtime roots. AST conformance guard rejects raw writes at CI. See [state decisions](decisions/state.md#concurrency-by-construction-mutate-under-lock-seams-over-convention-enforced-write-tiers-pr-422-2026-07).

**Typed state contracts: discriminated lifecycle facts, quarantine, and per-bundle semantics (PR #423, 2026-07)**
Six decisions made invalid states unparseable or unconstructible: single `SpawnStatus` StrEnum authority with member-derived lifecycle sets; quarantine (not coercion) for out-of-vocab rows; single status authority with non-duplicating `TerminalFacts` (top-level `status` only; `extra="forbid"` quarantines legacy nested status); `Applied | Declined | Missing` discriminated mutation result; normalize-once `EventSemantics` descriptor per raw event (replacing the deleted `SemanticEvent` union); directory location as sole archived-ness authority for work items. Three-way work store split (`work_state`/`work_store`/`work_repository`). See [state decisions](decisions/state.md#typed-state-contracts-pr-423-2026-07).

**Published-row lifetime owns spawn artifacts (issue #437, 2026-07)**
A spawn-owned artifact may change only while its `state.json` row is published.
Late parent-creating writers and deletion share the stable external per-spawn
lock; heartbeats, connection startup, and other simple boundaries never create
a missing spawn directory. See [state decisions](decisions/state.md#published-row-lifetime-owns-spawn-artifacts-issue-437-2026-07).

**Dual-root state: `.meridian/` + `~/.meridian/projects/<uuid>/`**
Project identity lives in the repo-local `.meridian/`; runtime state (spawns, sessions, work items) lives in the user home tree. This separates project configuration (committed, shared) from execution state (local, ephemeral). Migration toward user-level is intentional and ongoing. See [state decisions](decisions/state.md).

**Crash-only design — no graceful shutdown path**
Every write is atomic (tmp+rename). Every read tolerates truncation. Recovery is startup behavior. There is no shutdown path to get right. See [state decisions](decisions/state.md).

**Config precedence: CLI flag > ENV var > agent profile > project config > user config > harness default**
Each resolved field is independent. Derived fields inherit the precedence level of their source — a CLI model override forces harness re-derivation from that model, not from the profile. See [launch decisions](decisions/launch.md) and [model-resolution](decisions/model-resolution.md).

**`.agents/` eliminated — `.mars/` is Meridian's compiled read surface (D50)**
The `.agents/` directory was generated output that drifted from source. Mars now compiles directly into `.mars/`, a single authoritative surface for all harness adapters. Source lives in package repos; sync regenerates `.mars/`. See [package-management](decisions/package-management.md#d50).

**Mars package discovery is convention-based; hidden harness dirs require explicit import (D87)**
Mars discovers source items through a bounded convention walk over non-hidden package layers (`agents/`, `skills/`, `bootstrap/`, root `SKILL.md` fallback). Hidden harness directories such as `.claude/` are output/local-execution surfaces unless a dependency explicitly roots into them with `subpath` and `dialect`. See [package-management](decisions/package-management.md#d87-convention-based-source-discovery-and-explicit-hidden-foreign-import).

**Mars owns model alias resolution; Meridian calls `mars models list --json`**
Model alias expansion is delegated to Mars rather than duplicated in Meridian. Resolve-once pattern: aliases are expanded exactly once in `resolve_policies()`, never re-resolved downstream (D55). See [model-resolution](decisions/model-resolution.md).

**Managed-primary passive reaper safety (D-managed-primary-reaper, 2026-05-06)**
Passive reconciliation may finalize Codex/OpenCode managed primaries as `failed/orphan_primary`. It must not signal backend/TUI/runtime children on Skip decisions, but finalize-as-failed decisions may clean up tracked runtime children as a safety net. Missing metadata stays conservative: record `orphan_primary`, do not kill unknown PIDs. See [launch decisions](decisions/launch.md#d-managed-primary-reaper-managed-primary-passive-reaper-safety).

**Git autosync: merge over rebase, local-wins on conflict (D-autosync-1 through D-autosync-4, 2026-05; notice removal 2026-07)**
Integration strategy changed from `git pull --rebase` to `git merge origin/<branch>`. Default conflict policy changed to `"abort"`: on merge conflict, `git merge --abort` preserves local state (local-wins), writes per-conflict metadata. No conflict refs needed — local state is HEAD after abort. `autosync_store.py` owns all artifact layout and schema; both the hook and ops/CLI read through it. Per-conflict AGENTS.md notices were removed in PR #422 (2026-07): no agent ingestion contract existed to consume them; conflict JSON is the durable signal. See [decisions/git-autosync-merge-strategy.md](decisions/git-autosync-merge-strategy.md) and [decisions/state.md](decisions/state.md#autosync-agentsmd-notice-removal-pr-422-2026-07).

**Durable process-scope ownership over POSIX-only or psutil-only cleanup (D-process-scope-ownership, 2026-05 PR #184)**
Shared containment layer with OS-specific mechanisms, durable ownership metadata, and lease-aware policy. POSIX: setsid/process-group. Windows: Job Objects (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`). psutil tree kill is the degraded fallback. `spawn_owned` scopes die with the spawn; `session_owned` scopes persist while a session lease is active. Rejected: POSIX-only (Option A), psutil-primary (Option B), harness-specific (Option C) — all kept as sub-components. See [launch decisions](decisions/launch.md#d-process-scope-ownership-durable-process-scope-ownership-over-posix-only-or-psutil-only-cleanup) and [architecture/process-scope.md](architecture/process-scope.md).

**Completion drain convergence uses composition and reconciled descendant authority (2026-07, work:drain-convergence)**
Pi and resident drains use one completion coordinator composed from deep
evidence, profile/control, and cleanup collaborators; a shared base class was
rejected because Pi would override ordering-sensitive hooks. The reconciled
transitive spawn tree is authority for persisted Meridian descendants, while Pi
retains a private-work ledger for bash and the follow-up marker. Spawn rows publish atomically, so raw-directory allocation
inference is absent. Terminal outcomes publish before one async best-effort
cleanup; `done` is fail-closed on unreadable evidence. See
[completion drain coordination](architecture/completion-drain-coordination.md)
and [atomic child-row publication](architecture/atomic-child-row-publication.md).

**Terminal publication barrier: manager-owned idempotent `_publish_terminal` with explicit priority resolver (2026-07, probe-fix cycle)**
Terminal publication moved from the cancellable drain task to
`SpawnManager._publish_terminal()`, an idempotent barrier guarded by
`SpawnSession.terminal_published`. `resolve_terminal_outcome()` resolves
competing sources in explicit priority: success > authoritative stop > drain
classification. The former `preferred_stop_outcome` mutable override and
`prefer_drain_outcome` parameter were replaced. Cleanup tasks are per-spawn
keyed and drained at shutdown; `stop_spawn()` awaits only its own spawn.
Pi stream-exit classification is now category-complete via
`classify_outstanding_work()`; `pi_process_exited_with_tracked_children`
replaces only the canonical generic subprocess-exit outcome (shared constant
`PI_SUBPROCESS_EXIT_ERROR_PREFIX`), not unrelated specific failures. Root cause
(#433) was pre-existing since #225. See
[completion drain coordination](architecture/completion-drain-coordination.md)
and [drain plans](architecture/drain-plans.md).

**Timeout and completion policy: single carrier, opt-in ceilings, spawn-scoped rearm budget (2026-07, PR #375)**
`execution_policy.timeout` (minutes; seconds conversion at the runner edge only)
is the single timeout carrier for both CLI `--timeout` and `MERIDIAN_TIMEOUT`.
The former dual-carrier model (`ExecutionBudget.timeout_secs` alongside
`execution_policy.timeout`) was deleted because CLI armed one but not the other,
and env armed the other but not the first -- producing incorrect timeout
behavior in both paths. Resident rearm grants are signal-gated and unlimited by
default (`None`); users with 24h+ sessions need no default cap. Budget is opt-in
via `--resident-rearm-budget` / `MERIDIAN_RESIDENT_REARM_BUDGET` / profile
`resident-rearm-budget` / `timeouts.resident_rearm_budget`. The running count is
SPAWN-scoped (persisted as `resident_rearm_count` in `state.json`, monotonic
across streaming retries) so retry loops cannot bypass the bound. Pi is
intentionally unbounded while children live; `--timeout` / `MERIDIAN_TIMEOUT` is
the shared non-renewing opt-in absolute ceiling for both profiles, defaulting to
`None`. Rejected alternatives: default budget values (legit 24h+ sessions),
Pi default ceiling (unbounded-while-descendants-live is intentional), shared
bounded-extension vocabulary (larger rewrite planned). See
[architecture/drain-plans.md](architecture/drain-plans.md) and
[architecture/completion-drain-coordination.md](architecture/completion-drain-coordination.md).

**Authority/task domain split for spawn CWD and reference resolution (PR #248, 2026-05-22)**
Every spawn resolves two separate domains: authority domain (agent profiles, skills, config, KB — always from `control_root`) and task domain (agent working directory, reference file anchor — from worktree resolution). `kb:` prefix resolves KB-relative paths from the authority domain. Relative `-f` paths resolve from `task_cwd` (= `reference_anchor`). `@` removed for `-f` paths (use `kb:` instead). See [spawn-cwd-worktree-anchor decisions](decisions/spawn-cwd-worktree-anchor.md) and [architecture/launch-system.md](architecture/launch-system.md).

**Harness adapters are the only launch path (D63-launch)**
`MERIDIAN_HARNESS_COMMAND` env override was removed. All harness launches go through typed adapters in `src/meridian/lib/harness/`. Adding a harness is one adapter file plus registration — no other code changes. See [launch decisions](decisions/launch.md#d63-launch).

**Single composition seam: `build_launch_context()` for all driving adapters**
All launch paths — primary CLI, spawn subprocess, streaming-serve — compose prompts through one function. Semantic IR (`ComposedLaunchContent`) is projected to adapter-specific `ProjectedContent` by harness adapters, not in-line at call sites. See [launch decisions](decisions/launch.md).

**model-invocable agent visibility filter at inventory prompt boundary (D-model-invocable, 2026-05 PR #208; rewritten 2026-06 PR #314)**
`model-invocable: false` in agent profile frontmatter is enforced at inventory render time by Mars (in the `prompt_surface.inventory_prompt` bundle field). The Python fallback `build_agent_inventory_prompt()` was deleted in PR #314. The scanner remains policy-free; the inventory prompt is the model-facing boundary. Explicit CLI invocation ignores this field. See [launch decisions](decisions/launch.md#d-model-invocable-filter-at-inventory-prompt-boundary-not-at-catalog-scan).

**Skill document suppression for native-skill harnesses (D-skill-doc-suppression, 2026-05 PR #208)**
When `supports_native_skills=True`, `supplemental_documents` is empty — skill content is Mars-delivered natively. Claude's `--append-system-prompt` channel is preserved as the intended delivery path. All three active harnesses (Claude, Codex, OpenCode) declare this capability. See [launch decisions](decisions/launch.md#d-skill-doc-suppression-skip-supplemental_documents-for-native-skill-harnesses).

**Descriptor-driven startup: CommandDescriptor catalog owns startup policy**
CLI startup is driven by a lightweight command catalog that classifies invocations, plans bootstrap, and installs telemetry before any heavy ops modules import. Thin entrypoint handles `--help`/`--version` at ~16ms. `resolve_*` helpers are pure; `ensure_*` may mutate. See [startup decisions](decisions/startup-health-sandbox.md#startup-pipeline).

**Fixed sandbox projection constraint: no `~/.meridian/` root in child sandboxes**
Child sandboxes receive purpose-scoped projection only (project runtime roots, workspace/context roots, temp). Global user home is never projected. Each path class (automatic cache, cross-project coordination, global service, optional config, explicit maintenance) has a defined nested behavior: delete, wrap, gate, or keep. See [startup/sandbox decisions](decisions/startup-health-sandbox.md#sandbox-projection-policy).

**Codex turn-status ladder and TerminalEventOutcome succeeded invariant (PRs #459/#460, 2026-07)**
Codex `turn/completed` payload resolver reads nested `turn.status`: `completed`/absent → succeeded/0, `failed` → failed/1 with error, `interrupted` → cancelled/130, unknown → failed/1 diagnostic. `TerminalEventOutcome` enforces at construction that succeeded outcomes require exit_code=0 and no error. Headless deny policy moved from bind to prepare. See [launch decisions](decisions/launch.md#d-headless-deny-at-prepare-headless-deny-policy-moved-from-bind-to-prepare-pr-460-2026-07) and [harness abstraction](concepts/harness-abstraction.md).

**Behavior-owned test tiers and evidence-gated aggressive pruning (PRs #462/#463, 2026-07)**
Unit tests own pure functional cores; integration tests cross real seams;
contract tests own API payload/retry shapes; platform tests own OS behavior;
smoke guides own CLI-visible workflows. Coherent fail-closed security suites
may retain light composition in unit. Aggressive deletion requires closed
risk-register prerequisites and named-coverage verification; re-tiering
consolidates rather than copies. See [testing decisions](decisions/testing.md).

**Cross-harness death-shape normalization and Pi notification-machinery retirement (harness-hardening audit, PRs #446/#447, 2026-07-17)**
The four Pi defect classes from PR #443 (fake fidelity, inject truthfulness, terminal-reason shadowing, phantom telemetry) were audited across Claude, Codex, and OpenCode with runtime probes. Results: Codex resident terminal-shadowing confirmed live and fixed (typed `TerminalOutcomeCause.REPLACEABLE_TRANSPORT_CLOSE` scoped to Codex only); OpenCode phantom `response.completed` confirmed and removed from the accepted taxonomy; Claude death normalized to canonical `error/connectionClosed` carrying exit code + bounded stderr; Claude busy-turn inject investigated and found NOT a defect on Claude 2.1.212 (queues + acks correctly). Pi notification/subspawn lifecycle machinery (#440) confirmed dead and retired: zero canonical lifecycle events on real Pi; `meridian-spawn-watch` + `last-notification.json` is the functional contract. `PiSubspawnTracker`, `pi_process_cleanup.py`, and the notification-timeout/pending-ledger lifecycle event parsing were deleted; descendant discovery, `meridian-spawn-watch` follow-up, and `last-notification.json` were retained. The colocated contracts at `src/meridian/lib/harness/.context/CONTEXT.md` (Connection Death Shape, Inject Acknowledgment) and `tests/AGENTS.md` (generalized Fake Fidelity) are the per-adapter detail authority. See [architecture/pi-lifecycle.md](architecture/pi-lifecycle.md) and [concepts/harness-abstraction.md](concepts/harness-abstraction.md).

**POSIX-first supersedes Windows-first (platform stance, 2026-07-17)**
Linux and macOS are the supported platforms. Native Windows was never made to work and is not planned. This supersedes the prior "Windows Is First-Class" design principle (#12 in [design-principles.md](principles/design-principles.md)), which framed Windows support as a product requirement to design for upfront. Rationale: native Windows never reached a working state and maintaining parallel OS branches cost more than their value. Existing `os.name` / `sys.platform` branches in `lib/platform/` (locking, process-scope, signals, path resolution) stay as legacy, untested, best-effort — they must not be expanded. WSL is Linux and needs no special treatment. Canonical wording lives in root `AGENTS.md` "POSIX-first"; the doc sweep that aligned repo docs landed on branch `task/docs-posix-first`.

**Session initiation: four-mode model, identity lock, argv normalization (D-fork-identity-lock, D-prior-context-user-turn, D-argv-normalization-sentinel, D-from-fork-mutual-exclusion, 2026-05 PR #216)**
Four session-initiation modes unified across spawn and primary surfaces: `--continue` (resume in-place), `--fork` (fork transcript, identity locked), `--fork-fresh` (fork transcript, identity overridable), `--from` (fresh session, prior context → user-turn context blocks). Identity lock: `--fork` rejects `-a`/`-m`/`--skills` at CLI only; cache-preservation rationale. Prior context in user turn, not system prompt: four reasons (injection defense, cache locality, harness consistency, -f semantic consistency). Argv normalization sentinel (`__SELF__`): pre-Cyclopts normalization for optional-value flags; applies to all three flags on both surfaces. `--from` + fork rejected (MVP): redundancy via dual transcript coverage. See [launch decisions](decisions/launch.md#session-initiation-2026-05-pr-216) and [concepts/session-initiation.md](concepts/session-initiation.md).

**Continue replay: recorded launch contract, work, and task-dir are same-session state (D-continue-replays-recorded-launch-contract, 2026-07 work:continue-replay-contract)**
Primary and spawn `--continue` replay recorded work/task-dir and cache-shaping launch policy instead of resolving from current CWD/config/env, including recorded absence of work, explicit agent opt-out, recovered authoritative harness session IDs, and recorded `model=""` snapshots that mean no managed model override. Policy-changing overrides, including `--work` and `--task-dir`, are rejected because changing launch identity or work location is a divergent mode. See [launch decisions](decisions/launch.md#d-continue-replays-recorded-launch-contract-same-session-continue-is-not-live-policy-recomputation), [concepts/session-initiation.md](concepts/session-initiation.md), and [model-resolution: model optional](decisions/model-resolution.md#model-optional-empty-model).

**Spawn output contract: report-first default, transcript pointer as primary result, --metadata/--verbose split (2026-05)**
Foreground `spawn` and single-spawn `spawn wait` now default to compact report-first text: one-line status, blank line, report body (or `(no report)`), then the transcript inspection command. The transcript pointer (`meridian session log <id>`) is primary output, not metadata — it is the escape hatch when the report is incomplete. `--metadata` adds inline accounting detail (model, cost, tokens, duration, report path); `--verbose` remains debug verbosity. Both are separate concerns. Agent mode matches human mode by default (text, not JSON); explicit `--format json` includes the same primary content structurally. JSON wire shape unchanged. Multi-spawn wait, background submission, and `spawn show` unchanged. See [concepts/spawn-output-contract.md](concepts/spawn-output-contract.md).

## Domain Index

| Domain | Coverage | Page |
|---|---|---|
| **State** | State roots, JSONL event stores (sessions), spawn state v2 (per-spawn state.json), dual-root layout, crash-only design, presentation vs storage field naming (harness-agnostic wire names); **concurrency-by-construction (PR #422, 2026-07)**: mutate-under-lock seams, stable lock inodes with validated-EX GC seam, lock-order invariants, project-lifetime gate, permission policy split, conformance guard (AST over ruff TID251), autosync notice removal, conftest guard deletion; **cbc-followup (PR #444, 2026-07)**: lock GC (#427), torn-tail-proof JSONL appends, spawn_aggregate extraction, generic mutation framework rejected; **spawn artifact lifetime (issue #437, 2026-07)**: published-row ownership, delete/write serialization, fail-closed late writers and startup; **typed state contracts (PR #423, 2026-07)**: single SpawnStatus StrEnum authority, quarantine for out-of-vocab rows, single status authority with non-duplicating TerminalFacts, Applied/Declined/Missing mutation result, normalize-once EventSemantics descriptor, directory-authoritative work items, three-way work store split | [decisions/state.md](decisions/state.md) |
| **Launch** | Launch pipeline, prepare/bind split, CatalogSession, composition seam, harness identity env, spawn wait barrier, managed-primary Codex approval routing (D-primary-approval), managed-primary passive reaper safety (D-managed-primary-reaper), durable process-scope ownership (D-process-scope-ownership), spawn-goal authority + completion-contract composition (2026-05), Windows compat: portable signal handlers, Claude cancel platform split, output-log-path passthrough, stdout UTF-8 reconfigure, Windows CI gate (superseded: non-blocking), no-arg spawn wait descendant scoping (2026-05 PR #200); model-invocable agent inventory filter + user-invocable separation + skill-doc suppression for native-skill harnesses (D-model-invocable, D-skill-doc-suppression, 2026-05 PR #208); Mars-owns-inventory + headless-Claude deny default + agent-copy key rename (D-mars-owns-inventory, D-headless-claude-deny, D-agent-copy-key, 2026-06 PR #314); session initiation four-mode model, identity lock, argv normalization sentinel, prior-context-user-turn (D-fork-identity-lock, D-prior-context-user-turn, D-argv-normalization-sentinel, D-from-fork-mutual-exclusion, 2026-05 PR #216); continue replay of recorded work/task-dir and cache-shaping launch policy (D-continue-replays-recorded-launch-contract, 2026-07 work:continue-replay-contract); **spawn capability gating + reserve-before-prep crash recovery** (D-spawn-capability-gate, D-reserve-before-prep, 2026-06 PR #328); **Claude TUI trampoline session-ID reconciliation**: file-based over probe, adapter override over generic finalization (2026-06); **headless deny at prepare** (D-headless-deny-at-prepare, 2026-07 PR #460) | [decisions/launch.md](decisions/launch.md) |
| **Startup / health / sandbox** | Descriptor-driven startup, lazy import strategy, selective command registration, bootstrap split, doctor/reaper, sandbox projection policy | [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) |
| **Chat backend** | Custom protocol, event model, deferred acquisition, HITL, command layer, structural refactors, nested launch guard removal (D1–D32) | [decisions/chat-backend.md](decisions/chat-backend.md) |
| **Package management** | Mars compiler, skill schema, `.agents/` elimination, targeting, sync, bootstrap docs, collision resolution, convention-based source discovery, upgrades-available direct-deps fix, init command contract, content scan pattern, honest field naming (D35–D40, D50–D51, D58–D63, D71, D77–D79, D87); launch-bundle system: Mars/Meridian ownership, native-config passthrough, tool-policy preservation, Cursor experimental, Pi future, Gemini out-of-scope (D80–D85) | [decisions/package-management.md](decisions/package-management.md) |
| **Managed command references** | `managed_cmd()` helper for MERIDIAN_MANAGED-aware user-facing output; `mars` vs `meridian mars` in hints and errors (D76) | [decisions/managed-command-references.md](decisions/managed-command-references.md) |
| **Testing** | Behavior-owned tier boundaries, evidence-gated aggressive deletion, security-suite exception, contract placement, fake-executable observation | [decisions/testing.md](decisions/testing.md) |
| **Telemetry** | Envelope design, storage layout, readers-not-sinks pattern, liveness, segment retention (D61–D70); binary-mode tail_events() for stable Windows seek/tell (D-tel-binary-mode) | [decisions/telemetry.md](decisions/telemetry.md) |
| **Model resolution** | Alias resolution, routing context, profile schema, resolve-once, `MERIDIAN_HARNESS` spawn-local semantics (D52–D57); agent overlay config, canonical compiler, Meridian-vs-Mars ownership, candidate-chain semantics, RunnablePath harness-specific model IDs, explicit harness force, invalid override key skip (D72–D78); model prompting guidance with agent-first resolution, canonical `models prompting` command name (D86) | [decisions/model-resolution.md](decisions/model-resolution.md) |
| **Workspace** | `[workspace]` in `meridian.toml`, named entries, missing-path behavior, dedicated loader (D41–D46); `projected_roots` first-class field (D47); OpenCode merge-not-suppress (D48); sandbox-scope vs approval-policy independence (D49) | [decisions/workspace.md](decisions/workspace.md) |
| **Dev frontend** | Portless default, Vite fallback, exposure model, env scrub, launcher strategy (DF-D1–DF-D7) | [decisions/dev-frontend.md](decisions/dev-frontend.md) |
