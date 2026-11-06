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
| What design constraints guide X? | The meridian-cli root `AGENTS.md` and each subsystem's colocated `AGENTS.md` / `.context/CONTEXT.md` |
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
- [decisions/completion-drain.md](decisions/completion-drain.md) — A finished harness turn is a completion candidate, not proof that background work is finished
- [decisions/child-result-delivery.md](decisions/child-result-delivery.md) — See child result delivery for the current mechanism
- [decisions/primary-inject.md](decisions/primary-inject.md) — D-claude-primary-inbox (#665, merged in PR #668 and v0.10.22; flagged for owner review): a Claude primary is injected through Claude Code's own inbox socket, recorded by the session-boundary hook, not through a mod; children keep stdin; no token, no forged attestation, no `--settings` override; "posted", never "delivered"; optional `HarnessBundle.primary_inbox` port; PID-reuse guard on `recorded_at`. Codex/OpenCode/Pi stay on #498. D-spawn-message-frame: every message names its sender and reply command. D-spawn-message-address: another project's session is addressed `meridian -C <root> spawn message <chat>` (v0.16.4, #801); the reply names the sender's current chat (v0.17.1, #811). D-unified-delivery: every live launcher serves one control socket (not built, awaiting the owner's confirmation). In review: D-delivery-kind (PR #878), D-no-retry-after-write (PR #877)
- [decisions/chat-spawn-ownership.md](decisions/chat-spawn-ownership.md) — A chat reference names the run currently occupying that chat, otherwise its own latest linked session, never a child
- [decisions/ref-classification.md](decisions/ref-classification.md) — A reference's shape determines whether it names a spawn, chat, selector, or native session
- [decisions/pi-runtime.md](decisions/pi-runtime.md) — Why Pi keeps live shell authority with its owner and uses exact admission plus public-event evidence for completion delivery
- [decisions/idle-notifications.md](decisions/idle-notifications.md) — Adapters report facts; core decides scheduling and guards
- [decisions/idle-notify-channels.md](decisions/idle-notify-channels.md) — Notification delivery is independent of idle policy, so an agent can notify with idle automation disabled
- [decisions/hook-event-contract.md](decisions/hook-event-contract.md) — The current event catalog and payload are in `docs/hooks.md` and hooks and plugins
- [decisions/mars-portable-hooks.md](decisions/mars-portable-hooks.md) — Hook importers, WASM import exports and Cursor rendering are not built
- [decisions/release-process.md](decisions/release-process.md) — The release steps and compatibility policy are in meridian-cli `docs/releasing.md` and mars-agents' release guidance
- [decisions/compatibility-versioning.md](decisions/compatibility-versioning.md) — The compatibility policy is in meridian-cli `docs/releasing.md#compatibility-and-versioning`
- [decisions/idle-opencode-v2.md](decisions/idle-opencode-v2.md) — The sensor follows the transport's resolved dialect: the configured preference can be `auto`, and resolving it independently could disagree with the connection
- [decisions/idle-ttl-detection.md](decisions/idle-ttl-detection.md) — Idle TTL detection: launch-recorded native-store resolution and Pi provider TTL mapping, including estimated and unknown lifetimes.
- [decisions/state.md](decisions/state.md) — State-layer decisions cover project identity, runtime state, crash recovery, concurrency,
- [decisions/spawn-identity.md](decisions/spawn-identity.md) — A spawn has one public ID across continuations
- [decisions/rust-rewrite.md](decisions/rust-rewrite.md) — The rewrite is motivated by memory use and crashes when a running Python installation is updated underneath itself
- [decisions/history-storage.md](decisions/history-storage.md) — History Storage
- [decisions/native-session-identity.md](decisions/native-session-identity.md) — A chat is bound to one native key: harness, native store and native session ID
- [decisions/legacy-native-import.md](decisions/legacy-native-import.md) — The one-time import binds old chats only when recorded identity and a matching native store provide an unambiguous key
- [decisions/native-only-history.md](decisions/native-only-history.md) — A chat's conversation comes from the harness transcript selected by its recorded native binding
- [decisions/native-source-argument-admission.md](decisions/native-source-argument-admission.md) — Superseded raw-argument admission design
- [decisions/launch.md](decisions/launch.md) — See launch architecture and `src/meridian/lib/launch/.context/CONTEXT.md` for the mechanism
- [decisions/session-work-links.md](decisions/session-work-links.md) — Stable work and task links let a running execution follow work moves and checkout changes
- [decisions/connection-seams.md](decisions/connection-seams.md) — `SpawnManager` consumes the connection event stream and fans events out
- [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md) — Managed-primary and process-scope ownership decisions
- [decisions/primary-terminal-ownership.md](decisions/primary-terminal-ownership.md) — See terminal ownership for the current mechanism
- [decisions/launch-session-initiation.md](decisions/launch-session-initiation.md) — Wait, goal, and session-mode decisions
- [decisions/launch-harness-compatibility.md](decisions/launch-harness-compatibility.md) — Harness/platform compatibility history
- [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) — Why descriptor-driven startup, doctor tiering, and sandbox projection policy work the way they do
- [decisions/model-resolution.md](decisions/model-resolution.md) — Mars owns aliases and runtime routing
- [decisions/package-management.md](decisions/package-management.md) — Decisions about the `.mars/` store, source selection, targeting architecture,
- [decisions/testing.md](decisions/testing.md) — Test-tier ownership, aggressive deletion safeguards, security-suite exception, rejected alternatives, and fake-executable observation discipline
- [decisions/telemetry.md](decisions/telemetry.md) — Telemetry design and storage decisions
- [decisions/workspace.md](decisions/workspace.md) — Workspace grants, context separation, and missing-path behavior
- [decisions/spawn-cwd-worktree-anchor.md](decisions/spawn-cwd-worktree-anchor.md) — Relative file references resolve from the selected task directory, without per-file fallback: fallback makes the chosen file depend on what happens to exist on a machine
- [decisions/codebase-cleanup.md](decisions/codebase-cleanup.md) — Remaining cleanup work is tracked in #569
- [decisions/static-typing.md](decisions/static-typing.md) — Pyright strict mode with warnings fatal keeps uncertainty at the boundary where data enters the program
- [decisions/tui-framework.md](decisions/tui-framework.md) — prompt_toolkit full-screen Application for session browse; hand-rolled, textual, and fzf rejected on picker-specific evidence
- [decisions/session-browse.md](decisions/session-browse.md) — The picker shows recognizable session details and opens transcripts on request
- [decisions/instruction-files.md](decisions/instruction-files.md) — AGENTS.md is the instruction file; no CLAUDE.md mirrors, `qi claude-md-fix` deleted
- [decisions/opencode-prompt-delivery.md](decisions/opencode-prompt-delivery.md) — Meridian projects a reserved config agent at bind for the prompt, permissions and workspace grants
- [decisions/tool-permissions.md](decisions/tool-permissions.md) — Profiles express tool, sandbox and approval policy; harnesses enforce only what their native mechanisms can express
- [decisions/sandbox-platform-access.md](decisions/sandbox-platform-access.md) — Meridian projects its control-plane needs separately from profile permissions
- [decisions/bundle-field-consumption.md](decisions/bundle-field-consumption.md) — Every launch-bundle field is applied, reported as unsupported, or deliberately left unread with a reason

---

## Concepts

Mental models for key abstractions. Read before diving into architecture or code.

- [concepts/overview.md](concepts/overview.md) — Concepts domain overview and reading map
- [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) — What a spawn is (one ID across launches), the status machine (`running` to terminal once per launch), spawn capability gating, prepare before reserve, resident turn boundaries, crash recovery (lock-derived outcome, who persists it), cancel-all subtree scoping
- [concepts/spawn-wait-barrier.md](concepts/spawn-wait-barrier.md) — `meridian spawn wait` semantics: chat and descendant scoping, separate owed and blocked-on result classes, wait-set display, fail-fast, descendant blockers, discovery checkpoints, and harness-aware yield defaults
- [concepts/child-result-delivery.md](concepts/child-result-delivery.md) — how a background child's result reaches its lead through explicit wait or the Pi/Claude watchers, what a lead is owed, the cross-route consumption union, and the existing observation leases that coordinate wait with automatic delivery
- [concepts/spawn-output-contract.md](concepts/spawn-output-contract.md) — Report-first default output, transcript pointer as primary result, `--metadata` vs `--verbose` split, progressive disclosure hierarchy, agent-mode text default
- [concepts/state-model.md](concepts/state-model.md) — Dual-root state split (repo + user), event sourcing via JSONL, crash-only as a state property, UUID keying
- [concepts/harness-abstraction.md](concepts/harness-abstraction.md) — Policy/mechanism split, adapter contract, capability flags, terminal status semantics per harness, why harness-agnostic is load-bearing
- [concepts/composition-pipeline.md](concepts/composition-pipeline.md) — Semantic IR → adapter projection: how `ComposedLaunchContent`, capability guidance, and late-bound directories reach each harness
- [concepts/hooks-and-plugins.md](concepts/hooks-and-plugins.md) — Hooks observe lifecycle and notification events for side effects
- [concepts/extension-system.md](concepts/extension-system.md) — Unified command definition, extension registry, surface membership (CLI/MCP/HTTP), single dispatcher
- [concepts/skill-schema.md](concepts/skill-schema.md) — Universal SKILL.md frontmatter schema, invocability compilation matrix per harness, harness-level skill variants (model-level variants not supported)
- [concepts/skill-loading.md](concepts/skill-loading.md) — `skills.load` gives an agent a skill from the first turn; `skills.available` points to a skill it can load later
- [concepts/bootstrap-docs.md](concepts/bootstrap-docs.md) — Two-tier bootstrap doc discovery (skill resources + package-level), `meridian bootstrap` command design
- [concepts/config-precedence.md](concepts/config-precedence.md) — File-backed settings loader vs per-spawn runtime override stack; which system handles which setting
- [concepts/context-resolution.md](concepts/context-resolution.md) — Named context paths (work/kb/archive/strategy) surfaced to agents as `MERIDIAN_CONTEXT_*_DIR` env vars
- [concepts/session-initiation.md](concepts/session-initiation.md) — `--continue` resumes the same transcript, while `--fork` branches it with identity locked and `--fork-fresh` allows identity changes
- [concepts/continuation-model-selection.md](concepts/continuation-model-selection.md) — Target-constrained model selection on continuation and accepted startup selection/transport
- [concepts/workspace-projection.md](concepts/workspace-projection.md) — Filesystem permission grants for sibling repos; contrast with context resolution (permission vs guidance)
- [concepts/reference-resolution.md](concepts/reference-resolution.md) — How `-f` reference file paths resolve: relative paths from task_cwd, kb: prefix for KB-relative paths (authority_root-derived), @ removal

### Model Resolution

- [concepts/model-resolution/overview.md](concepts/model-resolution/overview.md) — How a model alias or profile name becomes a concrete harness invocation; pipeline overview
- [concepts/model-resolution/aliases-and-routing.md](concepts/model-resolution/aliases-and-routing.md) — Mars owns alias identity and launch routing
- [concepts/model-resolution/agent-profiles.md](concepts/model-resolution/agent-profiles.md) — Profile loading from `.mars/agents/`, frontmatter fields, skill attachment, profile backup candidates
- [concepts/model-resolution/model-policies.md](concepts/model-resolution/model-policies.md) — `model-policies` typed selector rules, visibility flags, per-entry `no-fallback`, whole-profile backup scan
- [concepts/model-resolution/vocabulary.md](concepts/model-resolution/vocabulary.md) — Model resolution glossary: aliases, policies, prompting guidance, agent-first resolution

### Package Management

- [concepts/package-management/overview.md](concepts/package-management/overview.md) — Mars as package manager: `mars.toml`, `.mars/` compiled store, harness native dir projection
- [concepts/package-management/compiler-pipeline.md](concepts/package-management/compiler-pipeline.md) — Reader → compiler → target sync flow; `compiler::compile()` entry point, module map, collision resolution
- [concepts/package-management/resolution-algorithm.md](concepts/package-management/resolution-algorithm.md) — Trait-based resolver: version selection, constraint validation, `ResolvedGraph` production
- [concepts/package-management/self-source-selection.md](concepts/package-management/self-source-selection.md) — Mars selects current-package content under a synthetic self owner
- [concepts/package-management/targeting.md](concepts/package-management/targeting.md) — How compiled content lands in native harness dirs; conditional agent emission, per-harness target rules
- [concepts/package-management/sync-model.md](concepts/package-management/sync-model.md) — `mars sync` pipeline: load → resolve → target → plan → apply; serialized runs, lock behavior, and pending canonical write recovery

---

## Architecture

How the system realizes the concepts — subsystem boundaries, invariants, data flows, and concrete component structure.

- [architecture/overview.md](architecture/overview.md) — Architecture domain overview, subsystem map, and reading paths
- [architecture/system-overview.md](architecture/system-overview.md) — Two external surfaces (CLI + MCP), shared extension registry, end-to-end data flow diagram
- [architecture/startup-pipeline.md](architecture/startup-pipeline.md) — Classification uses longest-prefix matching, so a missing command descriptor can silently inherit a parent's bootstrap requirements
- [architecture/launch-system.md](architecture/launch-system.md) — The launch pipeline separates preparation from binding so expensive resolution can be reused for preview and execution without separate composition paths drifting
- [architecture/idle-adapters.md](architecture/idle-adapters.md) — Harness adapters report idle and background-work facts; core owns the notification decision so policies do not drift among hosts
- [architecture/state-system/overview.md](architecture/state-system/overview.md) — Per-spawn state.json, context-resolved work items, atomic tmp+rename writes, `fcntl.flock` locking, lock-derived spawn status
- [architecture/state-system/derived-status.md](architecture/state-system/derived-status.md) — For a lock-backed launch, the runner holds `launch.lock` until exit
- [architecture/state-system/spawn-launches.md](architecture/state-system/spawn-launches.md) — A continue reopens the same spawn as a new launch so its identity and child relationships remain stable
- [architecture/state-system/project-identity.md](architecture/state-system/project-identity.md) — Committed project identity survives a checkout move; an ad-hoc identity is derived from the canonical path and leaves the directory untouched
- [architecture/state-system/portable-history.md](architecture/state-system/portable-history.md) — Readable files and verified immutable ZIPs are history authority; SQLite indexes are rebuildable projections
- [architecture/state-system/session-state.md](architecture/state-system/session-state.md) — A chat's native key identifies its harness, store and session
- [architecture/per-operation-cost.md](architecture/per-operation-cost.md) — Launch, close, wait, show, cancel and continue concern one spawn and chat; their work should not grow with project history
- [architecture/spawn-finalization.md](architecture/spawn-finalization.md) — The runner or a repairer that claims its free launch lock writes the terminal once
- [architecture/drain-plans.md](architecture/drain-plans.md) — shared resident confirmation, native plus persisted evidence, candidate-scoped signals, absolute completion windows and publish-before-cleanup boundary
- [architecture/completion-drain-coordination.md](architecture/completion-drain-coordination.md) — shared Pi/resident completion mechanism, cached indexed descendant refresh, Pi private-work boundary, and publish-before-cleanup invariant
- [architecture/pi-lifecycle.md](architecture/pi-lifecycle.md) — current Pi spawned-session lifecycle, quiescence, extension integration, and cleanup behavior
- [architecture/pi-runtime/coordination.md](architecture/pi-runtime/coordination.md) — Pi Bash owner recovery, canonical child membership, exact completion delivery, bounded RPC reception, usage accounting, and failure limits
- [architecture/legacy-native-import.md](architecture/legacy-native-import.md) — A once-only import binds legacy chats only when recorded facts identify an exact native source
- [architecture/pi-native-sessions.md](architecture/pi-native-sessions.md) — Pi can create a new identity when a requested session file is missing or empty, so Meridian validates the exact recorded source before resume or fork
- [architecture/native-session-binding.md](architecture/native-session-binding.md) — A chat binds a native key: harness, recorded store and native session ID
- [architecture/native-transcript-reads.md](architecture/native-transcript-reads.md) — Tracked chats read only their bound native source
- [architecture/attempt-facts-and-delivery.md](architecture/attempt-facts-and-delivery.md) — In pending PR #547 (source checkpoint `c1b64bcf`), drained spawns normalize once before
- [architecture/pi-runtime/overview.md](architecture/pi-runtime/overview.md) — Pi runtime vocabulary for background work and extension coordination
- [architecture/atomic-child-row-publication.md](architecture/atomic-child-row-publication.md) — nested staging and directory replacement for complete child-row visibility; Linux/POSIX proof and remaining platform gates
- [architecture/managed-primary-lifecycle.md](architecture/managed-primary-lifecycle.md) — Managed Codex/OpenCode process roles, startup/stop ownership gate, launcher death (`crashed`, guardians), reaper and cancel cleanup with exact-birth PID checks
- [architecture/process-scope.md](architecture/process-scope.md) — Harnesses can leave descendants after their root exits, including processes reparented to PID 1
- [architecture/primary-terminal-ownership.md](architecture/primary-terminal-ownership.md) — Codex and OpenCode TUIs use the native terminal; Meridian observes their backends through separate connections
- [architecture/sandbox-projection.md](architecture/sandbox-projection.md) — A child gets the current project's runtime and configured roots, not the whole shared Meridian home or project-lock directory
- [architecture/mars-compiler.md](architecture/mars-compiler.md) — The mars-agents checkout's `src/compiler/AGENTS.md` describes the compiler and ownership seams
- [architecture/mars-targeting.md](architecture/mars-targeting.md) — Why `.agents/` was eliminated, `.mars/` as Meridian's compiled read surface, native harness dir emission per target
- [architecture/mars-user-scope.md](architecture/mars-user-scope.md) — A user package root supplies agents and skills when no project package root exists, or as a content fallback for a project
- [architecture/mars-launch-bundle.md](architecture/mars-launch-bundle.md) — Cross-repo launch-bundle: Mars scaffold, Meridian injection, schema v4 routing/tool contracts, and Mars-owned native tool names
- [architecture/mars-routing.md](architecture/mars-routing.md) — Mars-internal routing: slug primitive, default harness_order, routing parity with models CLI, acceptance layer (PR #58 + #72)
- [architecture/mars-model-refresh.md](architecture/mars-model-refresh.md) — Models.dev catalog `ensure_fresh`, probe `ProbeRefreshMode`, `--refresh-models` / `--no-refresh-models` CLI surfaces
- [architecture/claude-native-sessions.md](architecture/claude-native-sessions.md) — Claude sessions share a store for a config root
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

---

## Codebase

High-level orientation for making changes — where to start, what owns what, how to extend without breaking invariants.

- [codebase/overview.md](codebase/overview.md) — Codebase domain overview and ownership map
- [codebase/guide.md](codebase/guide.md) — How to navigate the codebase, where to start reading, how to add new subsystems; source-tree rules as lints and when a text pre-filter is sound
- [codebase/cleanup-program.md](codebase/cleanup-program.md) — Remaining cleanup and deferred bugs are tracked in #569
- [codebase/harness-adapters.md](codebase/harness-adapters.md) — `lib/harness/`: capability matrix, primary event scope, and cross-harness comparison
- [codebase/tools.md](codebase/tools.md) — `lib/kg/`, `lib/mermaid/`, `lib/markdown/`: KG analysis, Mermaid validation, Markdown extraction exposed via `meridian kg` / `meridian mermaid`
- [codebase/work-items.md](codebase/work-items.md) — Cross-module work attachment, scratch directories, rename propagation, hook coordination
- [codebase/session-operations.md](codebase/session-operations.md) — `session browse` views and seams, `session log / search / export`: compaction segments, transcript extraction workflow, user-facing command guidance
- [codebase/session-log-rendering.md](codebase/session-log-rendering.md) — Internal rendering pipeline: ToolCall normalization, clean vs raw output, `--raw`/`--no-truncate` flag design, content pipeline order
- [codebase/test-determinism.md](codebase/test-determinism.md) — See the source checkout's `tests/AGENTS.md` and `tests/.context/CONTEXT.md` for test rules and helpers

---

## Operations

Operational knowledge for running, configuring, and diagnosing Meridian in practice.

- [operations/overview.md](operations/overview.md) — Operations domain overview and runbook map
- [operations/health-checks.md](operations/health-checks.md) — Doctor scope, status and pruning rationale
- [operations/session-archive-pruning.md](operations/session-archive-pruning.md) — Explicit, dry-run-first cleanup of redundant retired runner-history files
- [operations/troubleshooting.md](operations/troubleshooting.md) — Common failure patterns (a `crashed` spawn and `dead_spawns_unfinalized`, a primary killed with its pane by systemd's `OOMPolicy=stop` (now a `continue` drop-in on tmux scopes), a dead managed-primary launcher, locked files, harness startup failures) and recovery procedures
- [operations/configuration-guide.md](operations/configuration-guide.md) — Links to configuration reference, precedence and workspace architecture
- [operations/init-command.md](operations/init-command.md) — Project setup, preservation of user choices and identity
- [operations/bootstrap-command.md](operations/bootstrap-command.md) — `meridian bootstrap --add`: first-run setup alias that runs init steps then launches a guided session with the primary agent

---

---

## Research

Synthesized external research with implications for Meridian's design and KB structure.

- [research/overview.md](research/overview.md) — Research domain overview and source map
- [research/llm-wiki-pattern.md](research/llm-wiki-pattern.md) — Karpathy's LLM wiki concept: what it means for how this KB should be structured and maintained
- [research/tailscale-serve-semantics.md](research/tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), MagicDNS lookup
- [research/prompt-cache-retention.md](research/prompt-cache-retention.md) — How long each harness's provider keeps the prompt cache warm while a session
- [research/reaching-a-live-tui.md](research/reaching-a-live-tui.md) — What Codex 0.164, OpenCode 2.1.4 and Pi 1.4.0 accept from a second process while a human's TUI is attached; underpins the #498 message-route plan
- [research/claude-code-mods-and-inbox.md](research/claude-code-mods-and-inbox.md) — Claude Code 2.1.302 input paths, probed: `-p` exits about 2.5 s after idle whatever a mod holds open; `$.prompt.submit` queues until the turn ends; `session.start` fires once per process and again per fresh plugin load; 2.1.307 idle-delivery hooks (`prompt.submit` resolves on queue, `prompt.edit` carries typing, `agentId` on subagent turns, no cross-plugin imports); a broken mod fails silently; the cross-session inbox socket (wire format, no reply, mid-turn absorption between tool calls, bypass receivers hold unattested posts, `crossSessionInbound` and the `--settings` precedence trap)
- [research/harness-plugin-loading.md](research/harness-plugin-loading.md) — How Claude Code 2.1.310, Codex 0.164.0, OpenCode 2.1.3 and Cursor (unprobed) load plugins: formats and catalogs, plugin-root variables, Codex reading Claude-format trees but not loading agents or calling MCP, Claude plugin-agent ignored fields, session agents not preloading `skills:`, qualified names, plugin dependencies
- [research/opencode-config-semantics.md](research/opencode-config-semantics.md) — OpenCode 2.0.29 and 1.19.3, probed: v2 ignores `instructions` and agent `model`, `{file:}` substitutes raw text before JSON parsing, `default_agent` misses sessions with a stored agent, the v2 bare TUI rejects `--model`/`--agent` and needs `--standalone`, no v2 effort channel

---

## Lessons

Hard-won knowledge from building the system — failures, surprises, and approaches tried and abandoned.

- [lessons/overview.md](lessons/overview.md) — Lessons domain overview and learning map
- [lessons/state-design-lessons.md](lessons/state-design-lessons.md) — Committed identity travels with the repository; high-churn runtime state stays user-local
- [lessons/process-containment.md](lessons/process-containment.md) — A process group ID can be reused after its last member exits
- [lessons/harness-integration.md](lessons/harness-integration.md) — Native file-injection channels can give the same content different meanings across harnesses
- [lessons/native-session-identity.md](lessons/native-session-identity.md) — Native Session Identity Lessons
- [lessons/spawn-lane-operations.md](lessons/spawn-lane-operations.md) — Configured backup models are prelaunch candidates; an authentication or capacity failure after launch does not trigger fallback
- [lessons/probe-runtime-placement.md](lessons/probe-runtime-placement.md) — Keep scratch homes, copied runtime state, environments and databases under `/tmp`, outside autosynced work directories
- [lessons/dogfooding-pr-builds.md](lessons/dogfooding-pr-builds.md) — A build under test can write state that an installed older build cannot read, even during a nominally read-only operation
- [lessons/mars-compiler-cleanup.md](lessons/mars-compiler-cleanup.md) — Mars cache isolation and pointers to compiler guidance
- [lessons/source-simplification.md](lessons/source-simplification.md) — Removing a seam is a hypothesis about its contract
- [lessons/arch-refactor-pitfalls.md](lessons/arch-refactor-pitfalls.md) — Git subprocesses inherit `GIT_DIR` and `GIT_WORK_TREE` from hooks; those variables can redirect Git to another repository even when cwd is correct
- [lessons/thermo-nuclear-audit.md](lessons/thermo-nuclear-audit.md) — Rejected approaches to ownership, cleanup, parsing, and event schemas
- [lessons/release-sequencing.md](lessons/release-sequencing.md) — Independent producer and consumer releases can each pass their own gates while their combination fails
- [lessons/residue-cleanup-discipline.md](lessons/residue-cleanup-discipline.md) — When managed output changes shape, removal must recognize what prior versions actually wrote
- [lessons/verification-and-review-discipline.md](lessons/verification-and-review-discipline.md) — A check is evidence only when it reaches the intended behavior and its oracle can fail
- [lessons/review-convergence-gate.md](lessons/review-convergence-gate.md) — Recurring findings, integrated review, old-state probes, and unbiased review briefs
- [lessons/hook-event-contract.md](lessons/hook-event-contract.md) — A hook event can pass validation and never fire
- [lessons/contention-diagnosis.md](lessons/contention-diagnosis.md) — A deadline error can mean a lock holder is slow, not that the deadline is too short
- [lessons/cost-cuts-and-safety-checks.md](lessons/cost-cuts-and-safety-checks.md) — Before narrowing a scan, wait, or poll, identify the guarantees it provided and check that the faster path preserves them
- [lessons/verification-campaign-history.md](lessons/verification-campaign-history.md) — Ownership and verification hazards

---

## Open Questions

Tracked uncertainties, deferred work, and known future cleanup.

- [open-questions/overview.md](open-questions/overview.md) — Open questions domain overview and deferred-work map
- [open-questions/mars-feature-gaps.md](open-questions/mars-feature-gaps.md) — Open package capability, distribution, and model-routing questions
- [open-questions/process-scope.md](open-questions/process-scope.md) — Process-scope cleanup follow-ups: dead-wrapper/live-child escapes and metadata-only lifecycle projection
- [open-questions/future-work.md](open-questions/future-work.md) — Deferred items by domain, including the separate #498 live-primary injection-control gap, Jupyter Workbench structural follow-ups, decision follow-ups, and longer-horizon design questions

---

## Ecosystem

The broader Meridian ecosystem: companion repos, their roles, and how they compose.

- [ecosystem/overview.md](ecosystem/overview.md) — Ecosystem repo map: meridian-cli, mars-agents, prompt packages, domain packages, and their relationships
- [ecosystem/domain-packages.md](ecosystem/domain-packages.md) — Domain-specific packages (creative-writing-skills, microct-analysis, mouse-ct): purpose, ownership, interface rules

### Prompt Packages

- [ecosystem/prompt-packages/overview.md](ecosystem/prompt-packages/overview.md) — Package dependency boundaries and version-pin hazards
- [ecosystem/prompt-packages/meridian-base.md](ecosystem/prompt-packages/meridian-base.md) — Shared package, domain-neutral guidance and permissions hazard
- [ecosystem/prompt-packages/meridian-dev-workflow.md](ecosystem/prompt-packages/meridian-dev-workflow.md) — Development package and reasons for risk-scaled staffing
- [ecosystem/prompt-packages/meridian-prompter.md](ecosystem/prompt-packages/meridian-prompter.md) — Prompt engineering package: prompt-dev, reviewer, tester, researchers; proportional verification; depends on base (others reference it locally only)
- [ecosystem/prompt-packages/prompt-principles.md](ecosystem/prompt-packages/prompt-principles.md) — Prompt craft boundaries and unsupported claims to avoid

### Jupyter Workbench

- [ecosystem/jupyter-workbench/overview.md](ecosystem/jupyter-workbench/overview.md) — Notebook runtime and scientific-domain boundary
- [ecosystem/jupyter-workbench/architecture.md](ecosystem/jupyter-workbench/architecture.md) — Shared service layer and domain boundary
- [ecosystem/jupyter-workbench/runtime-model.md](ecosystem/jupyter-workbench/runtime-model.md) — Persistent kernel, replay, and notebook provenance

### MicroCT Analysis

- [ecosystem/microct-analysis/overview.md](ecosystem/microct-analysis/overview.md) — Current study-design, analysis, and review workflow
- [ecosystem/microct-analysis/measurement-design.md](ecosystem/microct-analysis/measurement-design.md) — Measurement domains, orientation hazards, and limits of validation
- [ecosystem/microct-analysis/toolkit-implementation.md](ecosystem/microct-analysis/toolkit-implementation.md) — Watershed markers, visual review, and coordinate-frame hazards
