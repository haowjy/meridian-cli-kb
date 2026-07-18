# Vocabulary

Shared terminology contract for Meridian. This page defines the ~20 most important cross-cutting terms. Domain-specific terms live in the domain vocabulary pages listed in the [Domain Index](#domain-index) below.

When documentation uses a term, it means exactly what the relevant vocabulary page says. "See also" links point to the concept or architecture page covering the topic in depth.

## Core Terms

| Term | Definition | See also |
|---|---|---|
| **Adapter** | A harness-specific module that translates a generic `SpawnParams` struct into a concrete subprocess command (argv + env), and extracts session IDs, token usage, and reports from harness output. One file per harness; registered in `HarnessRegistry`. | [concepts/harness-abstraction.md](concepts/harness-abstraction.md) |
| **Agent profile** | A YAML-fronted markdown file declaring a named agent: model, harness, skills, approval mode, sandbox, effort, fanout, and system prompt body. Loaded from `.mars/agents/` at launch time. Distinct from a skill. | [concepts/model-resolution/agent-profiles.md](concepts/model-resolution/agent-profiles.md) |
| **Atomic write** | Crash-safe file write implemented as write to a temp file + fsync + rename. The sole pattern for state mutations in Meridian; crash during write leaves the old value intact. | [principles/design-principles.md](principles/design-principles.md) |
| **Composition** | The process of assembling all launch-time inputs (prompt, skills, permissions, argv, env) into a `LaunchContext`. Performed exclusively by `build_launch_context()` in `lib/launch/context.py`. No driving adapter may re-derive what the factory already decided. | [architecture/launch-system.md](architecture/launch-system.md) |
| **Crash-only design** | An architectural property: every write is atomic (tmp+rename), every read tolerates truncation, and recovery is startup behavior rather than a shutdown hook. Meridian has no graceful shutdown path. If killed mid-spawn, the next read path triggers the reaper. | [principles/design-principles.md](principles/design-principles.md) |
| **Extension** | A named, surface-aware command unit declared as an `ExtensionCommandSpec`. Extensions are the single source of truth for all user-facing operations. The CLI, MCP server, and REST API all consume the same extension registry. | [concepts/extension-system.md](concepts/extension-system.md) |
| **Harness** | A concrete AI CLI tool (Claude, Codex, OpenCode) or in-process API (Direct) that Meridian adapts to execute spawns. Meridian is harness-agnostic — it never assumes which harness is in use. | [concepts/harness-abstraction.md](concepts/harness-abstraction.md) |
| **Invariant** | A rule that must hold across all code changes. The launch system has 13 documented composition invariants (e.g., "no driving adapter bypasses `build_launch_context()`"). | [principles/invariants.md](principles/invariants.md) |
| **JSONL** | The append-only log format used for session stores and chat event logs. Each line is one JSON event. Readers skip malformed or truncated lines. Session state is derived by replaying from the beginning; spawn state moved to per-spawn `state.json` in 2026-05. | [concepts/state-model.md](concepts/state-model.md) |
| **Launch context** | The `LaunchContext` frozen dataclass produced by `build_launch_context()`. Contains the full resolved launch state: argv, typed spec, env overrides, run params, permissions, child cwd, report output path, warnings, and resolved request. Complete and immutable at construction. | [architecture/launch-system.md](architecture/launch-system.md) |
| **Mars** | The standalone agent package manager (Rust CLI) that compiles source packages into `.mars/` for Meridian and emits skills to harness-native directories. Invoked via `meridian mars ...`. Owns model alias resolution. | [concepts/package-management/overview.md](concepts/package-management/overview.md) |
| **Precedence chain** | The resolution order applied independently to every config field: CLI flag > ENV var > agent profile > project config > user config > harness default. A CLI model override forces harness re-derivation from the overridden model, never from the profile's harness. | [concepts/config-precedence.md](concepts/config-precedence.md) |
| **Primary** | The top-level interactive agent launched directly by a human via `meridian` (without `spawn`). Has depth 0. Its chat ID is propagated to all descendant spawns as `$MERIDIAN_CHAT_ID`. Distinct from a spawn, which is a delegated child-agent task. | [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) |
| **Prompt package** | A mars source package containing agent profiles, skills, and bootstrap docs. Compiled into `.mars/` by `mars sync` and emitted to harness-native directories. Examples: `meridian-base`, `meridian-dev-workflow`, `meridian-prompter`. | [ecosystem/prompt-packages/overview.md](ecosystem/prompt-packages/overview.md) |
| **Reaper** | The component in `lib/state/reaper.py` that detects and reconciles orphaned spawns on read paths. Checks heartbeat recency (primary signal), `runner_pid` liveness (secondary), and durable report completion. Only applies root-level side effects when `$MERIDIAN_DEPTH` is absent, empty, or `0`. | [concepts/state-model.md](concepts/state-model.md) |
| **Session** | A tracked conversation unit between Meridian and a harness. Identified by a chat ID (`c42`). Events are recorded in `sessions.jsonl`. The top-level primary session's ID is `$MERIDIAN_CHAT_ID`. | [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) |
| **Skill** | A markdown file loaded into a spawn's system prompt to provide domain-specific instructions or operational doctrine. Resolved from `.mars/skills/` at launch time. Distinct from an agent profile. | [concepts/skill-schema.md](concepts/skill-schema.md) |
| **Spawn** | A delegated child-agent task created by a primary agent or orchestrator via `meridian spawn`. Executed by a harness process, tracked in `spawns/<id>/state.json`, and associated with a spawn ID. The core work unit in Meridian. | [concepts/spawn-lifecycle.md](concepts/spawn-lifecycle.md) |
| **State root** | The coordination boundary containing spawn/session/filesystem state. Committed `meridian.toml` `[project] id` anchors identity; mutable runtime lives under `~/.meridian/projects/<id>/` and default context under `~/.meridian/context/<id>/`. | [concepts/state-model.md](concepts/state-model.md) |
| **Work item** | A named unit of work tracked as a directory under the context work root, with mutable metadata in `__status.json`. Has a title, status, description, and associated spawns. The active work item is the default attachment point for new spawns. Directory location determines active-vs-archived. | [concepts/state-model.md](concepts/state-model.md) |

## Domain Index

Domain vocabulary pages cover terms specific to one subsystem. Each page has a brief intro and the same table format.

| Domain | File | What's covered |
|---|---|---|
| **Model resolution** | [concepts/model-resolution/vocabulary.md](concepts/model-resolution/vocabulary.md) | Alias entries, model policies, fanout, approval mode, skill registry, skill variants, resolved policies, prompting guidance, agent-first resolution |
| **Package management** | [concepts/package-management/vocabulary.md](concepts/package-management/vocabulary.md) | Mars compiler pipeline, source resolution, targeting, sync model, lock file, MVS |
| **Telemetry** | [architecture/telemetry/vocabulary.md](architecture/telemetry/vocabulary.md) | Telemetry sinks, envelope format, router, local persistence |
| **Workspace** | [architecture/workspace/vocabulary.md](architecture/workspace/vocabulary.md) | Workspace entries, multi-repo projection, snapshot merging |
| **Pi runtime** | [architecture/pi-runtime/vocab.md](architecture/pi-runtime/vocab.md) | `bash_id`, `bash_manage`, `managed-bash`, `meridian-spawn-watch`, `originating_bash_id`, env-var correlation, quiescence rule |
| **Codebase** | [codebase/vocabulary.md](codebase/vocabulary.md) | Harness infrastructure, hooks and plugins, safety and permissions, launch system, state management, config loading, platform abstractions |
