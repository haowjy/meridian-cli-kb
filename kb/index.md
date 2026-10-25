# Meridian KB — Index

Knowledge base for the Meridian ecosystem. Covers the CLI, the mars package compiler, prompt packages, and domain extensions — oriented toward agents and developers who need to understand, operate, or extend Meridian.

**Where to start:** Read [overview.md](overview.md) for the system mental model, then navigate by what you need to answer:

| Question | Go to |
|---|---|
| What is X? | [concepts/](#concepts) |
| How is X built? | [architecture/](#architecture) |
| Where does X live in the codebase? | [codebase/](#codebase) |
| Why was X decided? | [decisions.md](decisions.md) → [decisions/](#decisions) |
| What does term Y mean? | [vocabulary.md](vocabulary.md) |
| How do I operate or diagnose X? | [operations/](#operations) |
| What principles guide X? | [principles/](#principles) |
| What did we learn the hard way? | [lessons/](#lessons) |
| What do agents keep getting wrong here? | [corrections.md](corrections.md) |
| What's still unresolved? | [open-questions/](#open-questions) |
| What external research applies? | [research/](#research) |
| What other repos are in the ecosystem? | [ecosystem/](#ecosystem) |
| How should I maintain the KB? | [AGENTS.md](AGENTS.md) |

---

## Root Pages

- [overview.md](overview.md) — System identity, mental model, major subsystems, harness-agnostic coordination model
- [vocabulary.md](vocabulary.md) — Alphabetical shared terminology contract; every domain term defined in one place
- [decisions.md](decisions.md) — Chronological index of durable architectural decisions; entries link to domain pages for full rationale
- [corrections.md](corrections.md) — Register of the human's corrections of agents, grouped by mistake class, with what enforces each and dated occurrences
- [AGENTS.md](AGENTS.md) — KB maintenance guide: five-layer model, writing conventions, FLAG protocol, diagram rules, validation checklist

---

## Decisions

Durable decision rationale, clustered by domain. Start with [decisions.md](decisions.md) for the chronological index; follow anchors into domain pages for full reasoning.

- [decisions/overview.md](decisions/overview.md) — Domain page map and naming guidance for the decisions layer
- [decisions/completion-drain.md](decisions/completion-drain.md) — shipped candidate-scoped confirmation for Claude/Codex/resident OpenCode, caller-derived completion windows, truthful signal capability, the report contract (the retained reply is the report; latest text wins rejected), channel-less holds for children that cannot signal; keeps spawn visibility outside the correctness handshake and records the rejected persistent row
- [decisions/child-result-delivery.md](decisions/child-result-delivery.md) — Claude's core-owned `collect`/`ack` delivery, header-only fallback, and the bounded owed no-arg wait; why cancelled results and terminal reports printed by show surfaces consume under their caller while internal/status/search/session/context reads do not; plus Pi consumption evidence, observation leases, and the `observed-spawns.json` compatibility transition
- [decisions/primary-inject.md](decisions/primary-inject.md) — D-claude-primary-inbox (#665, merged in PR #668 and v0.10.22; flagged for owner review): a Claude primary is injected through Claude Code's own inbox socket, recorded by the session-boundary hook, not through a mod; children keep stdin; no token, no forged attestation, no `--settings` override; "posted", never "delivered"; optional `HarnessBundle.primary_inbox` port; PID-reuse guard on `recorded_at`. Codex/OpenCode/Pi stay on #498
- [decisions/pi-runtime.md](decisions/pi-runtime.md) — Why Pi keeps live shell authority with its owner and uses exact admission plus public-event evidence for completion delivery
- [decisions/idle-notifications.md](decisions/idle-notifications.md) — Core idle policy (push and cache warning; Meridian idle compaction removed), Claude's stateless permission-prompt notice, running work and the push (which shells hold it, which harnesses report running work and the open Codex question, why permission-prompt tracking was withdrawn), lifecycle, adapter-hosting rationale, session role, config, Pi retention, structural debt, and verification gaps; current mechanism is in [architecture/idle-adapters.md](architecture/idle-adapters.md).
- [decisions/idle-notify-channels.md](decisions/idle-notify-channels.md) — Notification leaf-package boundary, reason-first notice context and cache countdowns, notifications as hook events (routes, hook-row validation owner, fallback, route vs transport reports, `HookResult.details`, `hooks run` one-action rule, `--attach`), SMTP password handling, and deliberate spawn notifications.
- [decisions/hook-event-contract.md](decisions/hook-event-contract.md) — D-hook-event-names, D-hook-deprecation-window, D-work-activation, D-autosync-default-events, D-hook-failure-policy-removed, D-hook-fact-owners (v0.11.0, #687): ten past-tense hook events, old names loading with a warning for one minor release, activation means someone chose the item, git-autosync defaults `spawn.finished`/`work.activated`/`work.done`, `failure_policy` and event classes deleted, `when.kind`, one owner per event fact
- [decisions/compatibility-versioning.md](decisions/compatibility-versioning.md) — The compatibility policy (v0.11.0, #695) that replaced "no backwards compatibility": one-minor deprecation of config keys, hook names and CLI flags; new builds read old data, old builds refuse newer data; integer format versions only where another build reads; `__status.json` `v` staged; version checks on decisions, not blind appends; history index named by schema and session format; idle state as the absent-state exception
- [decisions/idle-opencode-v2.md](decisions/idle-opencode-v2.md) — OpenCode 2.x sensor selection, inbox return semantics, primary stream lifetime; its durable compaction confirmation is history (Meridian idle compaction removed).
- [decisions/idle-compaction-actuator.md](decisions/idle-compaction-actuator.md) — Superseded by D-idle-compaction-removed (Meridian no longer compacts idle sessions). Compaction over unsent drafts, including the Codex screen-gated actuator decision.
- [decisions/idle-compaction-growth-floor.md](decisions/idle-compaction-growth-floor.md) — Superseded by D-idle-compaction-removed. Idle compaction only when growth past a baseline beats a per-harness break-even (r = 0.5; Claude, Codex and OpenCode 2.x costs; Pi never until #593; OpenCode 1.x push only); baseline lifecycle, arm and fire context reads, override, patch release despite the key rename; v0.10.6
- [decisions/idle-ttl-detection.md](decisions/idle-ttl-detection.md) — Idle TTL detection: launch-recorded native-store resolution and Pi provider TTL mapping, including estimated and unknown lifetimes.
- [decisions/state.md](decisions/state.md) — Why dual-root state, JSONL event sourcing, crash-only reads/writes, concurrency by construction, typed state contracts, the single JSONL commit rule, and two-tier project identity (committed vs ad-hoc, no carry-over at `init`, ad-hoc retention); also the per-chat session journals (#418 design B; B2 not built) and D-session-read-seam, the shipped B1 read/append seam and its invariants (#634, #651).
- [decisions/spawn-identity.md](decisions/spawn-identity.md) — D-spawn-identity-v2 (owner-approved 2026-10-16, 7 PRs; PRs 1–3 merged, #633, #647 and #672): one ID per spawn, internal launches, status derived from a runner-held `launch.lock` plus a once-written outcome, gate + guardian + sweep containment, continue resolving an old spawn ID to the chat's current owner, why #616's `pN`/`pN.K` runs were rejected; D-start-gate-guardian (merged in #633, v0.10.13): the `sh` guardian replacing PDEATHSIG and `parent_watchdog.py`
- [decisions/rust-rewrite.md](decisions/rust-rewrite.md) — What is deferred to the Rust rewrite: D-update-crash-waits-for-rust (#628) and the requirements it carries; sequencing after identity-v2 (recommendation, pending the owner)
- [decisions/history-storage.md](decisions/history-storage.md) — Why file/ZIP retained-history authority, disposable indexes named by schema (never migrated in place), index readers that never wait on session writers (#588), retention and restore, archive digests over stored record JSON, post-stop native capture, and closing abandoned session generations with a typed capture deferral (PR #631); which parts native-only history superseded
- [decisions/native-session-identity.md](decisions/native-session-identity.md) — Each chat binds one immutable native key; exact-target entry; refusals before row creation; one binding and drift rule; run-boundary outcomes (Pi exit); Claude last-observed current-chat tracking replacing final-exit proof (unreleased) and its limits
- [decisions/legacy-native-import.md](decisions/legacy-native-import.md) — Why pre-key chats get one bounded exact-key import, its limits, and measured result; content-proven recovery of 0.6.7 Pi chats with no recorded ID (primaries manual via `session repair`); implementation details are in the architecture page
- [decisions/native-only-history.md](decisions/native-only-history.md) — Conversation history is the harness's native transcript: one read path, old runner history dropped (option C), rebuildable search and live run facts, combined-PR archive/browse/read fixes, one-time migration and late-binding repair policy, and prune rationale
- [decisions/native-source-argument-admission.md](decisions/native-source-argument-admission.md) — Superseded raw-argument admission design
- [decisions/launch.md](decisions/launch.md) — Why prepare/bind is the single composition seam, how harness identity propagates, and spawn wait semantics
- [decisions/connection-seams.md](decisions/connection-seams.md) — Harness connection seams that outlived the removed chat backend: single event-stream consumer, HITL handler protocol, managed-primary passthrough boundary, and spawn `max_depth` cap
- [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md) — Managed-primary and process-scope ownership decisions
- [decisions/primary-terminal-ownership.md](decisions/primary-terminal-ownership.md) — D-exit-text-append, D-tui-quiet-launcher (PR #621, v0.10.11), D-native-terminal, D-shared-foreground-job, D-launcher-watchdog, D-cancel-by-order: why Codex/OpenCode go native and Claude/Pi stay relayed, why no tcsetpgrp, why a watchdog instead of PDEATHSIG (mechanism superseded by the start gate and guardian), why cancellation is decided by order; rejected PTY override, `ESC[J`, exit-code cancel rule, Python-level handler swaps, a signal barrier in the window, the "log is non-empty" pointer rule
- [decisions/launch-session-initiation.md](decisions/launch-session-initiation.md) — Wait, goal, and session-mode decisions
- [decisions/launch-harness-compatibility.md](decisions/launch-harness-compatibility.md) — Harness/platform compatibility history
- [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) — Why descriptor-driven startup, doctor tiering, and sandbox projection policy work the way they do
- [decisions/model-resolution.md](decisions/model-resolution.md) — Why Mars owns alias authority, routing architecture, profile schema, policy matching, harness derivation, target-constrained fallback, and OpenCode native model commitment
- [decisions/package-management.md](decisions/package-management.md) — Why `.mars/` replaced `.agents/`, targeting design, compiler pipeline choices, source discovery, declared-package self-sync, canonical write recovery, and user-scope packages with `--fallback-root` (D97)
- [decisions/testing.md](decisions/testing.md) — Test-tier ownership, aggressive deletion safeguards, security-suite exception, rejected alternatives, and fake-executable observation discipline
- [decisions/telemetry.md](decisions/telemetry.md) — Three-layer telemetry design: local JSONL (v1), error reporting (v2), feature tracking (v3); retention, envelope schema, dead-zone taxonomy
- [decisions/workspace.md](decisions/workspace.md) — Why named workspace entries, permission-grant vs context-surfacing split, missing-path behavior, migration strategy
- [decisions/spawn-cwd-worktree-anchor.md](decisions/spawn-cwd-worktree-anchor.md) — Authority/task domain split: single reference anchor, kb: resolution from authority_root, stale task-dir graceful fallback (supersedes hard error), managed vs manual worktree ownership
- [decisions/codebase-cleanup.md](decisions/codebase-cleanup.md) — Why the cleanup program runs phases 1–2 now and tracks 3–5 in #569, why B7 was assigned to PR #546 (superseded by #588), why the spawn-archive overlay and `auto_init_cwd` were deleted, why state#1 was accepted short of its estimate, the G2 verdict on relocated config validation, unknown-key warnings from Pydantic's `extra="allow"` traversal (#580), batch telemetry durability, and the JSONL inventory
- [decisions/static-typing.md](decisions/static-typing.md) — Why pyright strict runs with warnings fatal and no demotions, the one allowed suppression form, where untyped data gets narrowed, and why nested CLI handlers are registered by call
- [decisions/tui-framework.md](decisions/tui-framework.md) — prompt_toolkit full-screen Application for session browse; hand-rolled, textual, and fzf rejected on picker-specific evidence
- [decisions/session-browse.md](decisions/session-browse.md) — Session picker: recognition columns without chat id or live marker, transcript on demand (preview subsystem deleted), staleness from idle state, one search matcher before `LIMIT`, primary-only lineage view, Python before Rust
- [decisions/instruction-files.md](decisions/instruction-files.md) — AGENTS.md is the instruction file; no CLAUDE.md mirrors, `qi claude-md-fix` deleted
- [decisions/opencode-prompt-delivery.md](decisions/opencode-prompt-delivery.md) — D-opencode-config-agent, D-opencode-explicit-agent, D-opencode-model-off-agent (shipped in v0.11.6, PR #710): the composed prompt reaches OpenCode as a bind-projected config agent selected on every session; model stays off the agent; the agent also carries the profile's permission rules; rejected channels; accepted costs (base prompt replaced, `Agent not found` outside Meridian)
- [decisions/tool-permissions.md](decisions/tool-permissions.md) — Profile tool permissions (meridian-cli v0.12.0 PR #730, mars-agents PR #179): Mars owns harness tool names (`native`, `native_tools`); each harness enforces natively, no Meridian hook; gaps reported in `--dry-run`, not warned; profiles bind primaries; headless `ask` refused and journaled; scoped-allow allowlists; list form deprecated (grants only); platform rules; permission sets add across surfaces; Codex sandbox inference deleted; user's own harness settings stay in force
- [decisions/sandbox-platform-access.md](decisions/sandbox-platform-access.md) — What Meridian grants a sandboxed launch: exact project lifetime lock, narrow command/child escapes, PID-namespace compound-launch refusal, the sandbox/spawn/projected-network child guardrail (not adversarial confinement; user harness config only warns), `network` modes and host sets with a sandbox-derived default, the bundle object and per-harness projection, and the profile follow-up still owed (unmerged: PRs #750, #752, #754; Mars #182 release pending)
- [decisions/bundle-field-consumption.md](decisions/bundle-field-consumption.md) — Every launch-bundle field applied, reported or deliberately unread; `harness-overrides` applied on Codex as `-c`, reported elsewhere; `autocompact`/`autocompact_pct` projection; `tools.mcp` is permission names (Meridian `mcp_tools` deleted)

---

## Concepts

Mental models for key abstractions. Read before diving into architecture or code.

- [concepts/overview.md](concepts/overview.md) — Concepts domain overview and reading map
- [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) — What a spawn is, the full status machine from `queued` to terminal, spawn capability gating, reserve-before-prep crash window, resident turn boundaries, crash recovery (lock-derived `crashed`, who persists it), cancel-all subtree scoping
- [concepts/spawn-wait-barrier.md](concepts/spawn-wait-barrier.md) — `meridian spawn wait` semantics: chat and descendant scoping, separate owed and blocked-on result classes, wait-set display, fail-fast, descendant blockers, discovery checkpoints, and harness-aware yield defaults
- [concepts/child-result-delivery.md](concepts/child-result-delivery.md) — how a background child's result reaches its lead through explicit wait or the Pi/Claude watchers, what a lead is owed, the cross-route consumption union, and the existing observation leases that coordinate wait with automatic delivery
- [concepts/spawn-output-contract.md](concepts/spawn-output-contract.md) — Report-first default output, transcript pointer as primary result, `--metadata` vs `--verbose` split, progressive disclosure hierarchy, agent-mode text default
- [concepts/state-model.md](concepts/state-model.md) — Dual-root state split (repo + user), event sourcing via JSONL, crash-only as a state property, UUID keying
- [concepts/harness-abstraction.md](concepts/harness-abstraction.md) — Policy/mechanism split, adapter contract, capability flags, terminal status semantics per harness, why harness-agnostic is load-bearing
- [concepts/composition-pipeline.md](concepts/composition-pipeline.md) — Semantic IR → adapter projection: how `ComposedLaunchContent` becomes harness-specific argv and env
- [concepts/hooks-and-plugins.md](concepts/hooks-and-plugins.md) — Spawn, work and `notify.*` hook events (what fires each, what activation excludes), `when` conditions checked against what each event supplies, layering with the `default` notify routes, one owner for hook-row validation, `hooks run` one-action rule, built-in hooks (git-autosync, notify), `HookResult.details`, shell hooks, plugin API boundary, `ignore_env` for CWD-relative root resolution in hooks
- [concepts/extension-system.md](concepts/extension-system.md) — Unified command definition, extension registry, surface membership (CLI/MCP/HTTP), single dispatcher
- [concepts/skill-schema.md](concepts/skill-schema.md) — Universal SKILL.md frontmatter schema, invocability compilation matrix per harness, harness-level skill variants (model-level variants not supported)
- [concepts/skill-loading.md](concepts/skill-loading.md) — How skills reach an agent: profile `load` and `--skills` compose full text on every harness whatever `model-invocable` says; `available` lists names; native skill lists show only model-invocable skills; per-harness gaps (OpenCode v2)
- [concepts/bootstrap-docs.md](concepts/bootstrap-docs.md) — Two-tier bootstrap doc discovery (skill resources + package-level), `meridian bootstrap` command design
- [concepts/config-precedence.md](concepts/config-precedence.md) — File-backed settings loader vs per-spawn runtime override stack; which system handles which setting
- [concepts/context-resolution.md](concepts/context-resolution.md) — Named context paths (work/kb/archive/strategy) surfaced to agents as `MERIDIAN_CONTEXT_*_DIR` env vars
- [concepts/session-initiation.md](concepts/session-initiation.md) — Four-mode session initiation: `--continue`, `--fork`, `--fork-fresh`, `--from`; identity lock; continue replay of recorded work/task-dir and launch policy; four-layer content composition; bare flag inference; session re-entry model (Resume/Fork/Blocked)
- [concepts/continuation-model-selection.md](concepts/continuation-model-selection.md) — Target-constrained model selection on continuation and accepted startup selection/transport
- [concepts/workspace-projection.md](concepts/workspace-projection.md) — Filesystem permission grants for sibling repos; contrast with context resolution (permission vs guidance)
- [concepts/reference-resolution.md](concepts/reference-resolution.md) — How `-f` reference file paths resolve: relative paths from task_cwd, kb: prefix for KB-relative paths (authority_root-derived), @ removal

### Model Resolution

- [concepts/model-resolution/overview.md](concepts/model-resolution/overview.md) — How a model alias or profile name becomes a concrete harness invocation; pipeline overview
- [concepts/model-resolution/aliases-and-routing.md](concepts/model-resolution/aliases-and-routing.md) — Mars alias entries, pattern fallback, identity/routing split, harness selection priority cascade; `mars models prompting` agent-first lookup via canonical launch policy
- [concepts/model-resolution/agent-profiles.md](concepts/model-resolution/agent-profiles.md) — Profile loading from `.mars/agents/`, frontmatter fields, skill attachment, profile backup candidates
- [concepts/model-resolution/model-policies.md](concepts/model-resolution/model-policies.md) — `model-policies` typed selector rules, visibility flags, per-entry `no-fallback`, whole-profile backup scan
- [concepts/model-resolution/vocabulary.md](concepts/model-resolution/vocabulary.md) — Model resolution glossary: aliases, policies, prompting guidance, agent-first resolution

### Package Management

- [concepts/package-management/overview.md](concepts/package-management/overview.md) — Mars as package manager: `mars.toml`, `.mars/` compiled store, harness native dir projection
- [concepts/package-management/compiler-pipeline.md](concepts/package-management/compiler-pipeline.md) — Reader → compiler → target sync flow; `compiler::compile()` entry point, module map, collision resolution
- [concepts/package-management/resolution-algorithm.md](concepts/package-management/resolution-algorithm.md) — Trait-based resolver: version selection, constraint validation, `ResolvedGraph` production
- [concepts/package-management/self-source-selection.md](concepts/package-management/self-source-selection.md) — How `.mars-src`, declared-package content, dependency renames, discovery, default ownership refusal, bounded explicit-force takeover, and interrupted-write recovery combine
- [concepts/package-management/targeting.md](concepts/package-management/targeting.md) — How compiled content lands in native harness dirs; conditional agent emission, per-harness target rules
- [concepts/package-management/sync-model.md](concepts/package-management/sync-model.md) — `mars sync` pipeline: load → resolve → target → plan → apply; serialized runs, lock behavior, and pending canonical write recovery

---

## Architecture

How the system realizes the concepts — subsystem boundaries, invariants, data flows, and concrete component structure.

- [architecture/overview.md](architecture/overview.md) — Architecture domain overview, subsystem map, and reading paths
- [architecture/system-overview.md](architecture/system-overview.md) — Two external surfaces (CLI + MCP), shared extension registry, end-to-end data flow diagram
- [architecture/startup-pipeline.md](architecture/startup-pipeline.md) — Descriptor-driven startup: CommandDescriptor catalog, thin entrypoint, bootstrap split, unified telemetry install, help profiles; lazy import strategy in `main.py`
- [architecture/launch-system.md](architecture/launch-system.md) — prepare/bind central seam (`prepare_launch_surface` + `bind_launch_context`), `PreparedLaunchSurface`, `CatalogSession`; three driving adapters, composition invariants; interactive starting-prompt delivery per harness
- [architecture/idle-adapters.md](architecture/idle-adapters.md) — Current idle adapter locations, per-harness coverage (push and warn), Claude's stateless permission-prompt signal, the boundary that keeps spawn visibility out of the idle adapter, runtime contracts, background-task reporting by Claude, Pi and OpenCode 2.x and core's `wake.py`, close/reopen behaviour, and known fragility
- [architecture/state-system/overview.md](architecture/state-system/overview.md) — Per-spawn state.json, context-resolved work items, atomic tmp+rename writes, `fcntl.flock` locking, lock-derived spawn status
- [architecture/state-system/derived-status.md](architecture/state-system/derived-status.md) — identity-v2 PR 3 (#672, v0.10.24): the runner-held `launch.lock` as the only liveness fact; `read_derived`'s three steps; stored vs derived status and the rule that selection for convergence or destruction uses stored status; `claim_dead_launch`/`finalize_dead_launch` (reaper, background repair, cancel); the reaper's cleanup claims; lockless legacy rows; lenient quarantine; the deleted guesses (heartbeat, graces, ghost rule, launch-boundary journal, finalize-evidence) and why they stay gone
- [architecture/state-system/project-identity.md](architecture/state-system/project-identity.md) — Committed vs ad-hoc tiers, `resolve`/`ensure_project_identity`/`mint_committed_identity`, `adhoc.<slug>.<hash12>` and `origin.json`, read purity, ad-hoc retention, `doctor --global` listing and prune locking; control-socket directory and startup recovery of abandoned sockets (datagram probe)
- [architecture/state-system/portable-history.md](architecture/state-system/portable-history.md) — Retained-history authority: file/ZIP retention, index files named by schema and session format, mixed-version overlap, bounded index initialization, deadline-bound callers build before starting their clock (#656), one-attempt reads and the lock-free sessions projection (#588), no transcript content in the index, native-only archive capture, capture deferral to a live owner and retry on its stop (#631), reclaim, inert restore, retained-snapshot reads for restored and imported refs
- [architecture/state-system/session-state.md](architecture/state-system/session-state.md) — Session authority today: one global event-sourced `sessions.jsonl` (not per-chat), the bind rule, the O(1) restored-session guard (#588), the keyed read/append seam (`chats_rows`, byte prefilter, scoped `SessionBindings`; #634/#651) and what it still scans, the accepted per-chat journal plan (#418 B2, not built), and the fold rule that closes abandoned generations (#631)
- [architecture/per-operation-cost.md](architecture/per-operation-cost.md) — The constant-time bar for per-spawn operations; what #588/#599/#605/#634/#651 fixed; known O(history) and O(spawns) violators with issue numbers (#418, #612, #594, #600, #601, #592, #595/#527, the untracked archiver); the cold-index deadline rule (#656); resident memory from full scans kept as Pydantic objects (#627), with MEM-1/B1 before/after and the ~230 MiB close residual; count-based measurement harness and acceptance gate
- [architecture/spawn-finalization.md](architecture/spawn-finalization.md) — terminal write policy (authority lattice), store-level finalization under flock, TerminalArbitrator, StreamingRunConclusion, PreparedExecutionHandoff, failure_policy consolidation
- [architecture/drain-plans.md](architecture/drain-plans.md) — shared resident confirmation, native plus persisted evidence, candidate-scoped signals, absolute completion windows and publish-before-cleanup boundary
- [architecture/completion-drain-coordination.md](architecture/completion-drain-coordination.md) — shared Pi/resident completion mechanism, cached indexed descendant refresh, Pi private-work boundary, and publish-before-cleanup invariant
- [architecture/pi-lifecycle.md](architecture/pi-lifecycle.md) — current Pi spawned-session lifecycle, quiescence, extension integration, and cleanup behavior
- [architecture/pi-runtime/coordination.md](architecture/pi-runtime/coordination.md) — Pi Bash owner recovery, canonical child membership, exact completion delivery, bounded RPC reception, usage accounting, and failure limits
- [architecture/legacy-native-import.md](architecture/legacy-native-import.md) — One-time exact import of legacy chat native keys: trigger, locking, recovery, candidate stores (incl. Pi interactive root), in-place OpenCode reads, deferral backoff, report mode, cost, late binding, legacy Pi recovery, and `session repair`
- [architecture/pi-native-sessions.md](architecture/pi-native-sessions.md) — Pi exact identity (mint/verify, exact argv), session-boundary exit observation, historical Pi 0.87.1 readback behaviors (lazy persistence, ctx invalidation, one-turn create, agent-dir credential route), limits, reopen-lineage readback; managed launches are gated to pending PR #547's stable 1.1.x contract
- [architecture/native-session-binding.md](architecture/native-session-binding.md) — Cross-harness identity after the PR 1 restructure: types, one bind rule and fold, adapter template and primitives, runner pipeline (bind_entry → observe → conclude_native_run after teardown), spawn-row run_boundary and continue_chat_id
- [architecture/native-transcript-reads.md](architecture/native-transcript-reads.md) — Combined PR #534 implementation: one resolver and `NativeRead` reader, chat-named pN views, native-keyed FTS5 search projection (witness, `complete`, rebuild/status, measured cost)
- [architecture/attempt-facts-and-delivery.md](architecture/attempt-facts-and-delivery.md) — Emit path (hooks → fan-out → `note_event_delivered`), per-harness attempt folds and report precedence, Pi lifecycle sidecar, the retired runner stream, history-blind read trap
- [architecture/pi-runtime/overview.md](architecture/pi-runtime/overview.md) — Pi runtime vocabulary for background work and extension coordination
- [architecture/atomic-child-row-publication.md](architecture/atomic-child-row-publication.md) — nested staging and directory replacement for complete child-row visibility; Linux/POSIX proof and remaining platform gates
- [architecture/managed-primary-lifecycle.md](architecture/managed-primary-lifecycle.md) — Managed Codex/OpenCode process roles, startup/stop ownership gate, launcher death (`crashed`, guardians), reaper and cancel cleanup with exact-birth PID checks
- [architecture/process-scope.md](architecture/process-scope.md) — Process-scope ownership and cleanup: Option D design decision, spawn_owned vs session_owned, platform adapters (POSIX/Windows/psutil), parent-death linkage through the start gate and guardian (#633), cleanup triggers (reaper, continue sweep, cancel, session exit), scope liveness
- [architecture/primary-terminal-ownership.md](architecture/primary-terminal-ownership.md) — Who owns the terminal in an interactive primary: Codex/OpenCode native inherit vs Claude/Pi PTY relay (adapter default, no override, not persisted), shared foreground job and TUI-initiated Ctrl+Z, the SIGINT window via `SignalCoordinator` and reentrancy rule, cancellation by order of observation, never writing over a TUI or its exit text (`ESC[?1049l` root cause, PR #607), the fd-level `launcher.log` window and its one pointer (PR #621), black-box SIGTERM/SIGHUP cancel through a latch-only receiver (PR #658, #623) and why receivers must not touch the gate or an Event, `termios.error` is not an `OSError`, known gaps #610/#611/#657 and the finalize-window seam handed to identity PR 4
- [architecture/sandbox-projection.md](architecture/sandbox-projection.md) — Sandbox permission projection policy: fixed constraint (no global user home or shared project-lock projection), exact project lifetime lock, path classification framework, context-root existence filtering (Codex bwrap safety), gap-by-gap behavior
- [architecture/mars-compiler.md](architecture/mars-compiler.md) — Compiler internals: module map, config-entry pipeline, MCP/hook collision resolution, provenance and stale cleanup
- [architecture/mars-targeting.md](architecture/mars-targeting.md) — Why `.agents/` was eliminated, `.mars/` as Meridian's compiled read surface, native harness dir emission per target
- [architecture/mars-user-scope.md](architecture/mars-user-scope.md) — User Mars root `~/.meridian/mars/`, `resolve_mars_roots`, content layering through `--fallback-root` (Mars 0.16.0 pin, `--help` probe as safety net, `mars_fallback: active|unsupported|n/a`), user-agent alias resolution, `meridian mars --user` and the mutation gate
- [architecture/mars-launch-bundle.md](architecture/mars-launch-bundle.md) — Cross-repo launch-bundle: Mars scaffold, Meridian injection, schema v4 routing/tool contracts, and Mars-owned native tool names
- [architecture/mars-routing.md](architecture/mars-routing.md) — Mars-internal routing: slug primitive, default harness_order, routing parity with models CLI, acceptance layer (PR #58 + #72)
- [architecture/mars-model-refresh.md](architecture/mars-model-refresh.md) — Models.dev catalog `ensure_fresh`, probe `ProbeRefreshMode`, `--refresh-models` / `--no-refresh-models` CLI surfaces
- [architecture/claude-native-sessions.md](architecture/claude-native-sessions.md) — Claude native sessions: shared-store hazard, header-validated exact source seeding/reads, launch-owned SessionStart journal with current-chat tracking (unreleased), and TUI trampoline successor as diagnostic only
- [architecture/cursor-harness.md](architecture/cursor-harness.md) — Cursor probe: raw-slug prefix routing, build-time `harness_model` effort resolution, legacy Meridian projector path

### Telemetry

- [architecture/telemetry/overview.md](architecture/telemetry/overview.md) — Three-layer observability spine: v1 local JSONL, v2 error reporting, v3 feature tracking; dead-zone coverage
- [architecture/telemetry/local-persistence.md](architecture/telemetry/local-persistence.md) — `LocalJSONLSink` write path, `BufferingSink` pre-root buffering, per-process segment files, compound naming
- [architecture/telemetry/event-catalog.md](architecture/telemetry/event-catalog.md) — `TelemetryEnvelope` schema, six event domains, `EVENT_REGISTRY`, concern classification, validation rules
- [architecture/telemetry/reader-and-query.md](architecture/telemetry/reader-and-query.md) — Segment discovery, truncation-tolerant reads, live tailing, query filtering, status summarization, retention cleanup

### Workspace

- [architecture/workspace/overview.md](architecture/workspace/overview.md) — Workspace as filesystem permission mechanism (not working memory); `--add-dir` arguments, scope boundary
- [architecture/workspace/config-schema.md](architecture/workspace/config-schema.md) — `[workspace.<name>]` TOML sections in `meridian.toml` / `meridian.local.toml`, entry naming, validation
- [architecture/workspace/resolution.md](architecture/workspace/resolution.md) — `resolve_workspace_snapshot()`: merge algorithm, `WorkspaceSnapshot` types, path evaluation, flow to launch
- [architecture/workspace/migration.md](architecture/workspace/migration.md) — Legacy `workspace.local.toml` fallback, `[[context-roots]]` → named sections migration command and phase plan

---

## Codebase

High-level orientation for making changes — where to start, what owns what, how to extend without breaking invariants.

- [codebase/overview.md](codebase/overview.md) — Codebase domain overview and ownership map
- [codebase/guide.md](codebase/guide.md) — How to navigate the codebase, where to start reading, how to add new subsystems
- [codebase/cleanup-program.md](codebase/cleanup-program.md) — The six structural problems from the 2026-10 thermo-nuclear review, the five-phase cleanup plan, what phases 1 and 2 fixed, deleted and made explicit, and what is tracked in #569
- [codebase/harness-adapters.md](codebase/harness-adapters.md) — `lib/harness/`: capability matrix, primary event scope, and cross-harness comparison
- [codebase/tools.md](codebase/tools.md) — `lib/kg/`, `lib/mermaid/`, `lib/markdown/`: KG analysis, Mermaid validation, Markdown extraction exposed via `meridian kg` / `meridian mermaid`
- [codebase/work-items.md](codebase/work-items.md) — Cross-module work attachment, scratch directories, rename propagation, hook coordination
- [codebase/session-operations.md](codebase/session-operations.md) — `session browse` views and seams, `session log / search / export`: compaction segments, transcript extraction workflow, user-facing command guidance
- [codebase/session-log-rendering.md](codebase/session-log-rendering.md) — Internal rendering pipeline: ToolCall normalization, clean vs raw output, `--raw`/`--no-truncate` flag design, content pipeline order
- [codebase/test-determinism.md](codebase/test-determinism.md) — Test determinism infrastructure: `AsyncDeterminism` + `FakeClock`, `process_race.py` cross-process harness, advance-past-boundary principle, behavioral vs safety-bound timeouts; flake classes: cold index builds inside a test deadline (five instances; shared product cause fixed by #656, warm-ups kept) and real-time deadlines in fake-clock tests (#632); waiting on disk when memory leads it (#662); fixtures that mean "live" hold the launch lock rather than ride the 15 s legacy grace; race tests red-proven against split implementations; patches stop at the interpreter boundary (test-launched children need the socket root re-applied)

---

## Operations

Operational knowledge for running, configuring, and diagnosing Meridian in practice.

- [operations/overview.md](operations/overview.md) — Operations domain overview and runbook map
- [operations/health-checks.md](operations/health-checks.md) — `meridian doctor`: two-tier design (cheap per-project default vs explicit global), background per-project repairs (dogfood migration, late legacy binding, legacy Pi recovery), stale pruning, live-spawn warnings
- [operations/session-archive-pruning.md](operations/session-archive-pruning.md) — Explicit, dry-run-first cleanup of redundant retired runner-history files
- [operations/troubleshooting.md](operations/troubleshooting.md) — Common failure patterns (a `crashed` spawn and `dead_spawns_unfinalized`, a primary killed with its pane by systemd's `OOMPolicy=stop` (now a `continue` drop-in on tmux scopes), a dead managed-primary launcher, locked files, harness startup failures) and recovery procedures
- [operations/configuration-guide.md](operations/configuration-guide.md) — Practical config setup: TOML file locations, `[workspace]` entries, env vars, profile overrides, resolution verification
- [operations/init-command.md](operations/init-command.md) — `meridian init --add` contract: bootstrap sequence, auto-link, primary agent handling, idempotency, JSON output
- [operations/bootstrap-command.md](operations/bootstrap-command.md) — `meridian bootstrap --add`: first-run setup alias that runs init steps then launches a guided session with the primary agent

---

## Principles

Engineering principles that govern Meridian's design, extracted from practice and documented for durability.

- [principles/overview.md](principles/overview.md) — Principles domain overview and guardrail map
- [principles/design-principles.md](principles/design-principles.md) — Core principles (policy/mechanism split, files-as-authority, crash-only, harness-agnostic, extend-don't-modify, op layer purity, smoke-tests-as-markdown) with rationale
- [principles/invariants.md](principles/invariants.md) — Launch composition invariants (I-1 through I-13), state invariants, op layer identity invariant, and where they are enforced

---

## Research

Synthesized external research with implications for Meridian's design and KB structure.

- [research/overview.md](research/overview.md) — Research domain overview and source map
- [research/llm-wiki-pattern.md](research/llm-wiki-pattern.md) — Karpathy's LLM wiki concept: what it means for how this KB should be structured and maintained
- [research/tailscale-serve-semantics.md](research/tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), MagicDNS lookup
- [research/prompt-cache-retention.md](research/prompt-cache-retention.md) — retention by harness, provider and authentication; signed-in Codex versus API-key policy, the separate #707 idle-warning divergence, compaction cache reads and Pi cache warming
- [research/claude-code-mods-and-inbox.md](research/claude-code-mods-and-inbox.md) — Claude Code 2.1.302 input paths, probed: `-p` exits about 2.5 s after idle whatever a mod holds open; `$.prompt.submit` queues until the turn ends; `session.start` fires once per process and again per fresh plugin load; 2.1.307 idle-delivery hooks (`prompt.submit` resolves on queue, `prompt.edit` carries typing, `agentId` on subagent turns, no cross-plugin imports); a broken mod fails silently; the cross-session inbox socket (wire format, no reply, mid-turn absorption between tool calls, bypass receivers hold unattested posts, `crossSessionInbound` and the `--settings` precedence trap)
- [research/opencode-config-semantics.md](research/opencode-config-semantics.md) — OpenCode 2.0.29 and 1.19.3, probed: v2 ignores `instructions` and agent `model`, `{file:}` substitutes raw text before JSON parsing, `default_agent` misses sessions with a stored agent, the v2 bare TUI rejects `--model`/`--agent` and needs `--standalone`, no v2 effort channel

---

## Lessons

Hard-won knowledge from building the system — failures, surprises, and approaches tried and abandoned.

- [lessons/overview.md](lessons/overview.md) — Lessons domain overview and learning map
- [lessons/state-design-lessons.md](lessons/state-design-lessons.md) — Why dual-root, why JSONL, what broke before the current design, repairing state before relaxing a safety check, what we'd do differently; spawn identity lessons: a name with two meanings generates bugs (bare `pN`), several status writers make late writes, exact lock liveness vs PID/heartbeat guesses (M1 vs M2), a deleted rule surviving in a second writer and in test fixtures (report ⇒ succeeded), "every reader" sweeping in selectors
- [lessons/process-containment.md](lessons/process-containment.md) — Keeping harnesses from outliving their runner (#633): PDEATHSIG reaches one process, put the killer in the group, root-only mode sends one SIGTERM, dash's single-digit fds, a guardian holding harness stdio deadlocks EOF readers, hand-forked PTY children inherit fds, "proven dead" shares the alive check's ±1 s tolerance, POCs must match production fd allocation and I/O topology
- [lessons/harness-integration.md](lessons/harness-integration.md) — Non-obvious discoveries from integrating harnesses: a terminal relay kept "for observation" that observed nothing, signal-handler reentrancy, capability gaps, behavioral surprises, and seam-design pitfalls (optional defaults, function-local imports hiding layering cycles), and a fresh `CLAUDE_CONFIG_DIR` hiding the owner's hooks from probes
- [lessons/native-session-identity.md](lessons/native-session-identity.md) — Native-session work lessons: seam defects green suites missed, terminal status versus process exit, removing carriers, once-only markers, restructure before stacking, projections that re-derive the authority's rule, `pytest -x` in slice gates, argv-gated `sitecustomize` missing `python -m` children, a diagnostic flag discarding authoritative cost, and correcting live-probe briefs and input protocol; argv reorder let a variadic flag swallow the prompt; in-place projection upgrade broke the running old build; model-coupled hashes break old archives; measure recovery rules for wrong binds
- [lessons/spawn-lane-operations.md](lessons/spawn-lane-operations.md) — Spawn lanes that fail invisibly: declared backups do not cover a mid-run auth or model-capacity failure (pin `-m` to a declared backup), a Claude lane's background gate was killed and reported `succeeded` for weeks because a prompting workaround stood in for the code fix (#655), parallel lanes colliding on test basenames after merge
- [lessons/probe-runtime-placement.md](lessons/probe-runtime-placement.md) — Probe runtimes go under `/tmp`, never a work directory: the 24 GB autosync outage with a stale `index.lock` (2026-10-15), the earlier near miss, a scratch Codex self-update replacing the global install, `git config` in a throwaway worktree leaking a fake identity into the shared repo config, an inherited `MERIDIAN_PROJECT_DIR` sending a scratch-home probe's `work` commands to the real project, `.gitignore` backstops, open fixes meridian-dev-workflow#7 and #538
- [lessons/dogfooding-pr-builds.md](lessons/dogfooding-pr-builds.md) — Running unreleased builds safely: never on a real runtime root, schema bumps and rollback, nested-env pinning and `-C`, `uv tool install` relinking, writing over `.venv/bin/*` rewriting uv's shared cache (three occurrences; new-inode swap, `--link-mode=copy`, repair), Codex's login shell running the installed `meridian`, scratch outside synced repos, masked push exit codes and SSH keepalive; probing the upgrade path with a genuinely old build (`uvx --isolated`) and mixed-version overlap
- [lessons/mars-compiler-cleanup.md](lessons/mars-compiler-cleanup.md) — Lessons from the Mars compiler cleanup: Windows config artifacts, lock indexing, integration-test split, diagnostic routing
- [lessons/source-simplification.md](lessons/source-simplification.md) — Lessons from Phase 8.6 source-seam and test-collapse work: deletion-first simplification, seam ownership moves, test contract discipline, over-collapse recovery
- [lessons/arch-refactor-pitfalls.md](lessons/arch-refactor-pitfalls.md) — Implementation pitfalls from PR #184/#200: GIT_DIR hook leaks, structlog cache test blindspot, CheckpointService git add -A danger, process-group kill requirement, Windows asyncio signal handler silent no-op, Windows Claude cancel SIGINT unreliable, Windows audit false-positive discipline, no-arg spawn wait full-chat-tree hang bug
- [lessons/thermo-nuclear-audit.md](lessons/thermo-nuclear-audit.md) — Thermo-nuclear audit method: two-source adversarial panel, rejected alternatives with reasons, peer-benchmark defended strengths and dropped steals
- [lessons/release-sequencing.md](lessons/release-sequencing.md) — Cross-repo release sequencing failure: breaking change shipped while consumer migration sat unpushed; structural gap, blast-radius search-scope error, runtime smoke as the only catch; landing a stacked chain (ship the producer before the pin, CHANGELOG misfiles after release merges and rebases, now caught by the push hook and CI, `merge-tree` exit codes)
- [lessons/residue-cleanup-discipline.md](lessons/residue-cleanup-discipline.md) — When mars changes what it emits, removal paths must recognize prior versions' actual output shapes. Twice-learned: sweep ordering, legacy format misses, real-shape verification discipline
- [lessons/verification-and-review-discipline.md](lessons/verification-and-review-discipline.md) — Verification validity, pre-import process/network denial for fake-only launch tests, non-vacuous oracles, CI-prerequisite parity (including Pi bundle rebuilds), running every cross-language adapter suite in a gate, `pytest-llm` not being the full suite, checking a reviewer's claim about main against main's code, a loads-silently oracle for diagnostic paths with the real-config sweep and old-build differential audit, comparing a failing probe scenario against the base before calling it a regression, runtime evidence and measurement discipline
- [lessons/review-convergence-gate.md](lessons/review-convergence-gate.md) — Same-class recurrence, redesign escalation, total-intent invariant, old-state runtime probe, and keeping the lead's verdict out of review briefs
- [lessons/hook-event-contract.md](lessons/hook-event-contract.md) — Hook names accepted at load but never fired (`spawn.start`, `work.start`, inert `failure_policy`, impossible `when.status` values) and the producer-coverage test that guards the class; three fix rounds on work activation traced to parallel owners of one fact
- [lessons/contention-diagnosis.md](lessons/contention-diagnosis.md) — Contention and cost diagnosis: measure the uncontended cost and find the lock holder before widening a timeout (#586/#588); check a cause ranking against the protocol; pending labels name bystanders; count work instead of timing it on a shared box; count every pass per operation, filtered or not (#418 B1); warm the index before measuring; probe the whole command end to end
- [lessons/cost-cuts-and-safety-checks.md](lessons/cost-cuts-and-safety-checks.md) — Cost cuts that silently weakened guarantees: an event wake that changed arbitration order (#605), a keyed live-observer check that missed `/resume` observers, a cycle check following one edge per spawn, a "disposable" pointer nothing could rebuild; equivalence corpora with strictly increasing clocks that hid an ordering change (#418 B1); keep a refusal test for every guarantee a narrowed check gave
- [lessons/verification-campaign-history.md](lessons/verification-campaign-history.md) — Collapsed historical chronology and incident evidence

---

## Open Questions

Tracked uncertainties, deferred work, and known future cleanup.

- [open-questions/overview.md](open-questions/overview.md) — Open questions domain overview and deferred-work map
- [open-questions/mars-feature-gaps.md](open-questions/mars-feature-gaps.md) — Known Mars capability gaps: permissions, tools, MCP integration, hook materialization (as of April 2026)
- [open-questions/process-scope.md](open-questions/process-scope.md) — Process-scope cleanup follow-ups: dead-wrapper/live-child escapes and metadata-only lifecycle projection
- [open-questions/future-work.md](open-questions/future-work.md) — Deferred items by domain, including the separate #498 live-primary injection-control gap, Jupyter Workbench structural follow-ups, decision follow-ups, and longer-horizon design questions

---

## Ecosystem

The broader Meridian ecosystem: companion repos, their roles, and how they compose.

- [ecosystem/overview.md](ecosystem/overview.md) — Ecosystem repo map: meridian-cli, mars-agents, prompt packages, domain packages, and their relationships
- [ecosystem/domain-packages.md](ecosystem/domain-packages.md) — Domain-specific packages (creative-writing-skills, microct-analysis, mouse-ct): purpose, ownership, interface rules

### Prompt Packages

- [ecosystem/prompt-packages/overview.md](ecosystem/prompt-packages/overview.md) — The package graph: which packages exist, published vs local dependencies, where inventories live, version-pin traps
- [ecosystem/prompt-packages/meridian-base.md](ecosystem/prompt-packages/meridian-base.md) — Root package: shared workers, coordination skills, model aliases, hooks; why it has no lead and spawning isn't a skill
- [ecosystem/prompt-packages/meridian-dev-workflow.md](ecosystem/prompt-packages/meridian-dev-workflow.md) — Dev package: leads, specialists, lifecycle skills; how the fixed pipeline became risk-staffed leads, and retired pieces not to revive
- [ecosystem/prompt-packages/meridian-prompter.md](ecosystem/prompt-packages/meridian-prompter.md) — Prompt engineering package: prompt-dev, reviewer, tester, researchers; proportional verification; depends on base (others reference it locally only)
- [ecosystem/prompt-packages/meridian-benchmark-agents.md](ecosystem/prompt-packages/meridian-benchmark-agents.md) — Benchmark package: unattended verifier-driven benchmark orchestration, bounded attempts, Claude-native subagent materialization, and reporting guardrails
- [ecosystem/prompt-packages/prompt-principles.md](ecosystem/prompt-packages/prompt-principles.md) — How the `/prompt-principles` skill is organized (body plus level resources), and claims it dropped

### Jupyter Workbench

- [ecosystem/jupyter-workbench/overview.md](ecosystem/jupyter-workbench/overview.md) — Two-package split (jupyter-workbench + microct-analysis), FastMCP adapter, key design decisions, why pyvista-cli was eliminated
- [ecosystem/jupyter-workbench/architecture.md](ecosystem/jupyter-workbench/architecture.md) — Hexagonal architecture, Ports and Adapters pattern, service boundaries (Session/Execution/Snapshot/Lineage), DTO contract, microct-analysis boundary rules
- [ecosystem/jupyter-workbench/runtime-model.md](ecosystem/jupyter-workbench/runtime-model.md) — Why Jupyter over plain REPL, no-daemon design, session lifecycle state machine, append-only notebook model, PyVista+trame in-kernel visualization

### MicroCT Analysis

- [ecosystem/microct-analysis/overview.md](ecosystem/microct-analysis/overview.md) — Agent architecture (analyst + specialists vs rejected alternatives), domain-agnostic design, workflow note system, slice-examination-loop skill, confidence gating contract
- [ecosystem/microct-analysis/measurement-design.md](ecosystem/microct-analysis/measurement-design.md) — Two measurement domains (femoral 3D surface vs tibial 2D slice), root cause of prior errors, OA6-1RK acceptance oracle, PCA orientation as design extrapolation, Amira SOP growth plate separation, SCANCO vs Amira threshold distinction
- [ecosystem/microct-analysis/toolkit-implementation.md](ecosystem/microct-analysis/toolkit-implementation.md) — Generic tools layer, 3DMedAgent pattern, watershed failure root cause and fix (sphere markers), CoordinateFrame contract, multi-plane rendering, OA6-1RK validation results (all 6 indices pass)
