# Meridian KB — Index

Knowledge base for the Meridian ecosystem. Covers the CLI, web frontend, prompt packages, and domain extensions — oriented toward agents and developers who need to understand, operate, or extend Meridian.

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
| What's still unresolved? | [open-questions/](#open-questions) |
| What external research applies? | [research/](#research) |
| What other repos are in the ecosystem? | [ecosystem/](#ecosystem) |
| How should I maintain the KB? | [AGENTS.md](AGENTS.md) |

---

## Root Pages

- [overview.md](overview.md) — System identity, mental model, major subsystems, harness-agnostic coordination model
- [vocabulary.md](vocabulary.md) — Alphabetical shared terminology contract; every domain term defined in one place
- [decisions.md](decisions.md) — Chronological index of durable architectural decisions; entries link to domain pages for full rationale
- [AGENTS.md](AGENTS.md) — KB maintenance guide: five-layer model, writing conventions, FLAG protocol, diagram rules, validation checklist (`CLAUDE.md` is a symlink)

---

## Decisions

Durable decision rationale, clustered by domain. Start with [decisions.md](decisions.md) for the chronological index; follow anchors into domain pages for full reasoning.

- [decisions/overview.md](decisions/overview.md) — Domain page map and naming guidance for the decisions layer
- [decisions/state.md](decisions/state.md) — Why dual-root state, JSONL event sourcing, crash-only reads/writes, read-only UUID behavior, and presentation vs storage field naming
- [decisions/launch.md](decisions/launch.md) — Why `build_launch_context()` is the composition seam, how harness identity propagates, and spawn wait semantics
- [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) — Why descriptor-driven startup, doctor tiering, and sandbox projection policy work the way they do
- [decisions/state-and-launch.md](decisions/state-and-launch.md) — Compatibility map for the previous combined state/launch decision page
- [decisions/dev-frontend.md](decisions/dev-frontend.md) — Dev frontend decisions DF-D1 through DF-D7: portless default, explicit exposure, PORTLESS env scrub, allowed-hosts, launcher strategy
- [decisions/model-resolution.md](decisions/model-resolution.md) — Why Mars owns alias authority, routing architecture, profile schema, policy matching, harness derivation from model, and candidate-chain semantics
- [decisions/package-management.md](decisions/package-management.md) — Why `.mars/` replaced `.agents/`, targeting design, compiler pipeline choices, universal skill schema, bootstrap doc model, upgrades-available direct-deps fix
- [decisions/managed-command-references.md](decisions/managed-command-references.md) — `managed_cmd()` pattern: how mars-agents surfaces `mars` vs `meridian mars` in user-facing output based on `MERIDIAN_MANAGED`
- [decisions/telemetry.md](decisions/telemetry.md) — Three-layer telemetry design: local JSONL (v1), error reporting (v2), feature tracking (v3); retention, envelope schema, dead-zone taxonomy
- [decisions/workspace.md](decisions/workspace.md) — Why named workspace entries, permission-grant vs context-surfacing split, missing-path behavior, migration strategy
- [decisions/spawn-cwd-worktree-anchor.md](decisions/spawn-cwd-worktree-anchor.md) — Authority/task domain split: single reference anchor, kb: resolution from authority_root, stale worktree hard error, managed vs manual worktree ownership

---

## Concepts

Mental models for key abstractions. Read before diving into architecture or code.

- [concepts/overview.md](concepts/overview.md) — Concepts domain overview and reading map
- [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) — What a spawn is, the full status machine from `queued` to terminal, spawn capability gating, reserve-before-prep crash window, resident turn boundaries, heartbeat and reaper mechanics, cancel-all subtree scoping
- [concepts/spawn-wait-barrier.md](concepts/spawn-wait-barrier.md) — `meridian spawn wait` semantics: chat lineage scoping, descendant-scoping for nested spawns, wait-set display, yield-checkpoint, harness-aware yield defaults
- [concepts/spawn-output-contract.md](concepts/spawn-output-contract.md) — Report-first default output, transcript pointer as primary result, `--metadata` vs `--verbose` split, progressive disclosure hierarchy, agent-mode text default
- [concepts/state-model.md](concepts/state-model.md) — Dual-root state split (repo + user), event sourcing via JSONL, crash-only as a state property, UUID keying
- [concepts/harness-abstraction.md](concepts/harness-abstraction.md) — Policy/mechanism split, adapter contract, capability flags, terminal status semantics per harness, why harness-agnostic is load-bearing
- [concepts/composition-pipeline.md](concepts/composition-pipeline.md) — Semantic IR → adapter projection: how `ComposedLaunchContent` becomes harness-specific argv and env
- [concepts/hooks-and-plugins.md](concepts/hooks-and-plugins.md) — Lifecycle hook system, built-in hooks (git-autosync), shell hooks, plugin API boundary, `ignore_env` for CWD-relative root resolution in hooks
- [concepts/extension-system.md](concepts/extension-system.md) — Unified command definition, extension registry, surface membership (CLI/MCP/HTTP), single dispatcher
- [concepts/skill-schema.md](concepts/skill-schema.md) — Universal SKILL.md frontmatter schema, invocability compilation matrix per harness, skill variants and 4-step specificity ladder
- [concepts/bootstrap-docs.md](concepts/bootstrap-docs.md) — Two-tier bootstrap doc discovery (skill resources + package-level), `meridian bootstrap` command design
- [concepts/config-precedence.md](concepts/config-precedence.md) — File-backed settings loader vs per-spawn runtime override stack; which system handles which setting
- [concepts/context-resolution.md](concepts/context-resolution.md) — Named context paths (work/kb/archive/strategy) surfaced to agents as `MERIDIAN_CONTEXT_*_DIR` env vars
- [concepts/session-initiation.md](concepts/session-initiation.md) — Four-mode session initiation: `--continue`, `--fork`, `--fork-fresh`, `--from`; identity lock; continue replay of recorded work/task-dir and launch policy; four-layer content composition; bare flag inference
- [concepts/workspace-projection.md](concepts/workspace-projection.md) — Filesystem permission grants for sibling repos; contrast with context resolution (permission vs guidance)
- [concepts/reference-resolution.md](concepts/reference-resolution.md) — How `-f` reference file paths resolve: relative paths from task_cwd, kb: prefix for KB-relative paths (authority_root-derived), @ removal

### Model Resolution

- [concepts/model-resolution/overview.md](concepts/model-resolution/overview.md) — How a model alias or profile name becomes a concrete harness invocation; pipeline overview
- [concepts/model-resolution/aliases-and-routing.md](concepts/model-resolution/aliases-and-routing.md) — Mars alias entries, pattern fallback, identity/routing split, harness selection priority cascade; `mars models prompting` agent-first lookup via canonical launch policy
- [concepts/model-resolution/agent-profiles.md](concepts/model-resolution/agent-profiles.md) — Profile loading from `.mars/agents/`, frontmatter fields, skill attachment, runtime fallback chain
- [concepts/model-resolution/model-policies.md](concepts/model-resolution/model-policies.md) — `model-policies` typed selector rules, visibility flags, superseded models, candidate-chain transform semantics
- [concepts/model-resolution/vocabulary.md](concepts/model-resolution/vocabulary.md) — Model resolution glossary: aliases, policies, prompting guidance, agent-first resolution

### Package Management

- [concepts/package-management/overview.md](concepts/package-management/overview.md) — Mars as package manager: `mars.toml`, `.mars/` compiled store, harness native dir projection
- [concepts/package-management/compiler-pipeline.md](concepts/package-management/compiler-pipeline.md) — Reader → compiler → target sync flow; `compiler::compile()` entry point, module map, collision resolution
- [concepts/package-management/resolution-algorithm.md](concepts/package-management/resolution-algorithm.md) — Trait-based resolver: version selection, constraint validation, `ResolvedGraph` production
- [concepts/package-management/targeting.md](concepts/package-management/targeting.md) — How compiled content lands in native harness dirs; conditional agent emission, per-harness target rules
- [concepts/package-management/sync-model.md](concepts/package-management/sync-model.md) — `mars sync` pipeline: load → resolve → target → plan → apply; atomic, idempotent, lockfile behavior

---

## Architecture

How the system realizes the concepts — subsystem boundaries, invariants, data flows, and concrete component structure.

- [architecture/overview.md](architecture/overview.md) — Architecture domain overview, subsystem map, and reading paths
- [architecture/system-overview.md](architecture/system-overview.md) — Three external surfaces + one in-process adapter, shared extension registry, end-to-end data flow diagram
- [architecture/startup-pipeline.md](architecture/startup-pipeline.md) — Descriptor-driven startup: CommandDescriptor catalog, thin entrypoint, bootstrap split, unified telemetry install, help profiles; lazy import strategy in `main.py`
- [architecture/launch-system.md](architecture/launch-system.md) — `build_launch_context()` central seam; prepare/bind split (`prepare_launch_surface` + `bind_launch_context`), `PreparedLaunchSurface`, `CatalogSession`; four driving adapters, composition invariants
- [architecture/state-system.md](architecture/state-system.md) — JSONL event stores, atomic tmp+rename writes, `fcntl.flock` locking, reaper reconciliation, migration strategy
- [architecture/spawn-finalization.md](architecture/spawn-finalization.md) — terminal write policy (authority lattice), store-level finalization under flock, TerminalArbitrator, StreamingRunConclusion, PreparedExecutionHandoff, failure_policy consolidation
- [architecture/completion-drain-coordination.md](architecture/completion-drain-coordination.md) — shared Pi/resident completion mechanism, transitive descendant authority, Pi private-work boundary, and publish-before-cleanup invariant
- [architecture/pi-lifecycle.md](architecture/pi-lifecycle.md) — current Pi spawned-session lifecycle, quiescence, extension integration, and cleanup behavior
- [architecture/pi-runtime/overview.md](architecture/pi-runtime/overview.md) — Pi runtime vocabulary for background work and extension coordination
- [architecture/atomic-child-row-publication.md](architecture/atomic-child-row-publication.md) — nested staging and directory replacement for complete child-row visibility; Linux/POSIX proof and remaining platform gates
- [architecture/managed-primary-lifecycle.md](architecture/managed-primary-lifecycle.md) — Managed Codex/OpenCode primary process roles, passive reconciliation safety, explicit cleanup boundary, and `orphan_primary` diagnosis
- [architecture/process-scope.md](architecture/process-scope.md) — Process-scope ownership and cleanup: Option D design decision, spawn_owned vs session_owned, platform adapters (POSIX/Windows/psutil), module boundary map, reaper behavior, and known gaps (PROC-004, PROC-007)
- [architecture/sandbox-projection.md](architecture/sandbox-projection.md) — Sandbox permission projection policy: fixed constraint (no global user home projection), path classification framework, context-root existence filtering (codex bwrap safety), gap-by-gap behavior
- [architecture/app-server.md](architecture/app-server.md) — FastAPI layer: REST endpoints, WebSocket/SSE streaming, MCP stdio server, connection management
- [architecture/mars-compiler.md](architecture/mars-compiler.md) — Compiler internals: module map, config-entry pipeline, MCP/hook collision resolution, provenance and stale cleanup
- [architecture/mars-targeting.md](architecture/mars-targeting.md) — Why `.agents/` was eliminated, `.mars/` as Meridian's compiled read surface, native harness dir emission per target
- [architecture/mars-launch-bundle.md](architecture/mars-launch-bundle.md) — Cross-repo launch-bundle: Mars scaffold, Meridian injection, bundle `routing` contract, schema v2
- [architecture/mars-routing.md](architecture/mars-routing.md) — Mars-internal routing: slug primitive, default harness_order, routing parity with models CLI, acceptance layer (PR #58 + #72)
- [architecture/mars-model-refresh.md](architecture/mars-model-refresh.md) — Models.dev catalog `ensure_fresh`, probe `ProbeRefreshMode`, `--refresh-models` / `--no-refresh-models` CLI surfaces
- [architecture/claude-session-isolation.md](architecture/claude-session-isolation.md) — Upstream Claude shared-config limitation, isolated overlay mechanism, transcript materialization lifecycle, primary vs child behavior, `--continue` flow
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
- [codebase/harness-adapters.md](codebase/harness-adapters.md) — `lib/harness/`: capability matrix, primary event scope, and cross-harness comparison
- [codebase/observability.md](codebase/observability.md) — Telemetry spine framing and relationship to durable state log
- [codebase/tools.md](codebase/tools.md) — `lib/kg/`, `lib/mermaid/`, `lib/markdown/`: KG analysis, Mermaid validation, Markdown extraction exposed via `meridian kg` / `meridian mermaid`
- [codebase/work-items.md](codebase/work-items.md) — Cross-module work attachment, scratch directories, rename propagation, hook coordination
- [codebase/session-operations.md](codebase/session-operations.md) — `session log / search / export`: compaction segments, transcript extraction workflow, user-facing command guidance
- [codebase/session-log-rendering.md](codebase/session-log-rendering.md) — Internal rendering pipeline: ToolCall normalization, clean vs raw output, `--raw`/`--no-truncate` flag design, content pipeline order
- [codebase/test-determinism.md](codebase/test-determinism.md) — Test determinism infrastructure: `AsyncDeterminism` + `FakeClock`, `process_race.py` cross-process harness, advance-past-boundary principle, behavioral vs safety-bound timeouts

---

## Operations

Operational knowledge for running, configuring, and diagnosing Meridian in practice.

- [operations/overview.md](operations/overview.md) — Operations domain overview and runbook map
- [operations/health-checks.md](operations/health-checks.md) — `meridian doctor`: two-tier design (cheap per-project default vs explicit global), background per-project repairs, stale pruning, live-spawn warnings
- [operations/troubleshooting.md](operations/troubleshooting.md) — Common failure patterns (`orphan_run`, `orphan_finalization`, locked files, harness startup failures) and recovery procedures
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
- [research/t3-code-reference.md](research/t3-code-reference.md) — T3 Code architecture analysis: what was taken (single dispatch, event taxonomy, invariant library) vs. rejected (CQRS, AG-UI/ACP, Vercel AI SDK)
- [research/vite-portless-funnel.md](research/vite-portless-funnel.md) — Vite host-check CVE (CVE-2025-24010), portless v0.12.0 `--force` behavior and `PORTLESS_*` env catalog, Tailscale Funnel prerequisites
- [research/tailscale-serve-semantics.md](research/tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), MagicDNS lookup

---

## Lessons

Hard-won knowledge from building the system — failures, surprises, and approaches tried and abandoned.

- [lessons/overview.md](lessons/overview.md) — Lessons domain overview and learning map
- [lessons/state-design-lessons.md](lessons/state-design-lessons.md) — Why dual-root, why JSONL, what broke before the current design, what we'd do differently
- [lessons/harness-integration.md](lessons/harness-integration.md) — Non-obvious discoveries from integrating Claude, Codex, and OpenCode: PTY capture, capability gaps, behavioral surprises, patterns that generalized
- [lessons/chat-normalization-repair.md](lessons/chat-normalization-repair.md) — Lessons from repairing chat normalization drift: harness compatibility mapping, completion dedupe, replay obligations, and smoke-test caveats
- [lessons/mars-compiler-cleanup.md](lessons/mars-compiler-cleanup.md) — Lessons from the Mars compiler cleanup: Windows config artifacts, lock indexing, integration-test split, diagnostic routing
- [lessons/source-simplification.md](lessons/source-simplification.md) — Lessons from Phase 8.6 source-seam and test-collapse work: deletion-first simplification, seam ownership moves, test contract discipline, over-collapse recovery
- [lessons/arch-refactor-pitfalls.md](lessons/arch-refactor-pitfalls.md) — Implementation pitfalls from PR #184/#200: GIT_DIR hook leaks, structlog cache test blindspot, CheckpointService git add -A danger, process-group kill requirement, Windows asyncio signal handler silent no-op, Windows Claude cancel SIGINT unreliable, Windows audit false-positive discipline, no-arg spawn wait full-chat-tree hang bug

---

## Open Questions

Tracked uncertainties, deferred work, and known future cleanup.

- [open-questions/overview.md](open-questions/overview.md) — Open questions domain overview and deferred-work map
- [open-questions/mars-feature-gaps.md](open-questions/mars-feature-gaps.md) — Known Mars capability gaps: permissions, tools, MCP integration, hook materialization (as of April 2026)
- [open-questions/process-scope.md](open-questions/process-scope.md) — Process-scope cleanup follow-ups: dead-wrapper/live-child escapes and metadata-only lifecycle projection
- [open-questions/future-work.md](open-questions/future-work.md) — Deferred items by domain: Jupyter Workbench structural follow-ups, decision follow-ups, longer-horizon design questions

---

## Ecosystem

The broader Meridian ecosystem: companion repos, their roles, and how they compose.

- [ecosystem/overview.md](ecosystem/overview.md) — Ecosystem repo map: meridian-cli, mars-agents, prompt packages, domain packages, and their relationships
- [ecosystem/domain-packages.md](ecosystem/domain-packages.md) — Domain-specific packages (creative-writing-skills, microct-analysis, mouse-ct): purpose, ownership, interface rules

### Prompt Packages

- [ecosystem/prompt-packages/overview.md](ecosystem/prompt-packages/overview.md) — Package model: agent profile + skill bundles, versioned, composable, materialized via `meridian mars sync`
- [ecosystem/prompt-packages/meridian-base.md](ecosystem/prompt-packages/meridian-base.md) — Core package: orchestrator, subagent, KB agents, and foundation skills that all other packages build on
- [ecosystem/prompt-packages/meridian-dev-workflow.md](ecosystem/prompt-packages/meridian-dev-workflow.md) — Dev workflow package: design leads, planners, coders, reviewers, testers, documentation writers
- [ecosystem/prompt-packages/meridian-prompter.md](ecosystem/prompt-packages/meridian-prompter.md) — Prompt engineering package: prompt writers, reviewers, testers; dependency of both base and dev-workflow
- [ecosystem/prompt-packages/meridian-benchmark-agents.md](ecosystem/prompt-packages/meridian-benchmark-agents.md) — Benchmark package: unattended verifier-driven benchmark orchestration, bounded attempts, Claude-native subagent materialization, and reporting guardrails
- [ecosystem/prompt-packages/prompt-principles.md](ecosystem/prompt-packages/prompt-principles.md) — Core prompt-engineering doctrine: four levels of principles tied to research evidence, not stated as preferences

### Jupyter Workbench

- [ecosystem/jupyter-workbench/overview.md](ecosystem/jupyter-workbench/overview.md) — Two-package split (jupyter-workbench + microct-analysis), FastMCP adapter, key design decisions, why pyvista-cli was eliminated
- [ecosystem/jupyter-workbench/architecture.md](ecosystem/jupyter-workbench/architecture.md) — Hexagonal architecture, Ports and Adapters pattern, service boundaries (Session/Execution/Snapshot/Lineage), DTO contract, microct-analysis boundary rules
- [ecosystem/jupyter-workbench/runtime-model.md](ecosystem/jupyter-workbench/runtime-model.md) — Why Jupyter over plain REPL, no-daemon design, session lifecycle state machine, append-only notebook model, PyVista+trame in-kernel visualization

### MicroCT Analysis

- [ecosystem/microct-analysis/overview.md](ecosystem/microct-analysis/overview.md) — Agent architecture (analyst + specialists vs rejected alternatives), domain-agnostic design, workflow note system, slice-examination-loop skill, confidence gating contract
- [ecosystem/microct-analysis/measurement-design.md](ecosystem/microct-analysis/measurement-design.md) — Two measurement domains (femoral 3D surface vs tibial 2D slice), root cause of prior errors, OA6-1RK acceptance oracle, PCA orientation as design extrapolation, Amira SOP growth plate separation, SCANCO vs Amira threshold distinction
- [ecosystem/microct-analysis/toolkit-implementation.md](ecosystem/microct-analysis/toolkit-implementation.md) — Generic tools layer, 3DMedAgent pattern, watershed failure root cause and fix (sphere markers), CoordinateFrame contract, multi-plane rendering, OA6-1RK validation results (all 6 indices pass)
