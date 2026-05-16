# Codebase Guide

How to navigate the Meridian codebase, where to start reading, and how to add new things.

## Package Structure

```
src/meridian/
  cli/            CLI entry point (cyclopts), command groups, ext_cmd.py, ext_registration.py
  server/         MCP stdio server (FastMCP) — extension_list_commands + extension_invoke
  lib/
    extensions/   Unified command system: ExtensionCommandSpec, registry, dispatcher
    ops/          Operation handler implementations + commands.py
    harness/      Subprocess adapters: Claude, Codex, OpenCode, Direct + connections/
    state/        JSONL event stores, work items, path resolution, reaper, atomic writes
    catalog/      Model resolution, agent/skill loading, default agent policy
    launch/       Spawn lifecycle: build_launch_context() + four driving adapters
    config/       Settings model, precedence chain, TOML read/write, workspace config loader
    context/      Context path resolver: work/kb path resolution
    hooks/        Hook dispatch: lifecycle events → builtins or shell commands
    core/         Shared primitives: ID types, OutputSink, depth, ResolvedContext, lifecycle
    platform/     Cross-platform: file locking, process termination, deferred OS imports
    app/          REST server: FastAPI + SSE/WebSocket streaming
    streaming/    HarnessConnection abstractions + SpawnManager
    observability/ Structured logging context, debug tracing
    safety/       Budget, guardrails, permissions, redaction
    kg/           Knowledge graph analysis: document graph, broken links
    mermaid/      Mermaid diagram validation
    markdown/     Markdown extraction (headings, links, fenced blocks)
    utils/        Additional utilities beyond core/
    ignores.py    Shared gitignore-style pattern loader (pathspec)
  plugin_api/     Stable plugin API v1: Hook/HookContext types, state/git/config/fs helpers
  dev/            Token-efficient pytest wrapper
```

## What to Read First

**Understanding a spawn:** start with `ops/spawn/api.py` (the handler), then `lib/launch/context.py` (the factory), then `lib/state/spawn_store.py` (how it lands).

**Understanding command routing:** start with `ops/commands.py` (all operations declared), then `cli/ext_registration.py` (how CLI commands are generated), then `lib/extensions/registry.py` (the registry).

**Understanding state:** start with `lib/state/paths.py` (path resolution), then `lib/state/spawn_store.py` (event store), then `lib/state/reaper.py` (reconciliation).

**Understanding harness integration:** start with `lib/harness/adapter.py` (protocols), then one adapter (e.g., `lib/harness/claude.py`), then `lib/harness/common.py` (shared command assembly). See [harness-adapters.md](harness-adapters.md) for the capability matrix.

**Understanding active spawn runtime:** start with `lib/streaming/spawn_manager.py` (connection registry, drain loop), then `lib/streaming/drain_policy.py` (when to stop), then `lib/streaming/control_socket.py` (inject endpoint).

**Understanding session log / transcript reading:** start with `lib/ops/session_log.py` (entry point), then `lib/harness/transcript.py` (message extraction), then `lib/ops/reference.py` (reference resolution). See [session-operations.md](session-operations.md).

**Understanding work items:** start with `lib/state/work_store.py` (directory-authoritative storage), then `lib/ops/work_lifecycle.py` (lifecycle operations). See [work-items.md](work-items.md).

## How to Add Things

### A new CLI command

1. Define a handler function in the appropriate `ops/` module
2. Add an `ExtensionCommandSpec` entry in `ops/commands.py` via `ExtensionCommandSpec.from_op()`
3. The CLI auto-generates the command via `cli/ext_registration.py` — no explicit registration needed
4. The command is automatically available via the MCP server and HTTP server too

### A new harness

1. Create an adapter file in `lib/harness/` implementing `SubprocessHarness` (or `InProcessHarness`)
2. Define `HarnessCapabilities` — set capability flags conservatively
3. Implement `STRATEGIES: StrategyMap` for command assembly (every `SpawnParams` field must be mapped or added to `_SKIP_FIELDS`)
4. Implement `project_content()` for semantic content routing (both PRIMARY and SPAWN_PREPARE surfaces)
5. Register the adapter in `lib/harness/registry.py:with_defaults()`
6. For bidirectional streaming: add a `HarnessConnection` implementation in `lib/harness/connections/`

No shared code should need modification.

### A new operation / extension command

Same as adding a CLI command above — the operation handler + manifest entry is all that's needed. The extension system dispatches to it from all surfaces.

### A new lifecycle hook

1. Define the hook name and payload type in `lib/hooks/`
2. Emit the hook event from `SpawnLifecycleService` at the appropriate transition
3. Register built-in handlers or document the shell hook interface
4. See `plugin_api/` for the stable hook API surface

### A new config field

1. Add to the settings model in `lib/config/`
2. Add to `RuntimeOverrides` in `lib/core/overrides.py` if it's a per-spawn override
3. Add a classmethod constructor update to `RuntimeOverrides.from_*` methods
4. Ensure `resolve_policies()` picks it up in `lib/launch/policies.py`

## Testing Strategy

**Smoke tests first.** Integration smoke tests (`tests/smoke/`) are the primary quality gate. They test real CLI invocations and catch what unit tests miss. Run with `uv run meridian`.

**Unit tests for hard-to-smoke logic:** signals, concurrency, security/env sanitization, sync engine algorithms, parsing edge cases. Run with `uv run pytest-llm`.

**Platform coverage required:** when touching paths, process launching, signals, shells, filesystem semantics, or config discovery — consider Windows semantics explicitly. See `CLAUDE.md` for the full platform policy.

**Lint and type checks:**
```bash
uv run ruff check .    # lint
uv run pyright          # type check (must be 0 errors)
```

## Important Invariants

- **Never edit `.mars/` or native generated harness dirs** — generated output, overwritten by `meridian mars sync`
- **Never delete untracked files** without asking — may be another agent's in-progress work
- **Never revert changes** — assume it's someone else's work
- **Commit after each verified step** — don't accumulate changes across multiple steps
- **Launch composition invariants** — 13 invariants at `.meridian/invariants/launch-composition-invariant.md`; check on every PR touching `launch/`, `harness/`, `ops/spawn/`, `app/`, or `cli/streaming_serve.py`

## Related Pages

- [../architecture/system-overview.md](../architecture/system-overview.md) — subsystem map
- [../architecture/launch-system.md](../architecture/launch-system.md) — launch factory details
- [../architecture/state-system.md](../architecture/state-system.md) — state internals
- [harness-adapters.md](harness-adapters.md) — harness capability matrix
