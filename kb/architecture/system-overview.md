# Architecture: System Overview

Meridian exposes one coordination layer through two external surfaces. Both share a single extension registry — no duplicated command definitions. User requests flow through launch and land in state; the state layer is the only durable record.

## Surface Map

```mermaid
graph TD
    subgraph Surfaces
        CLI["CLI\ncli/ · cyclopts\nstdio, human-readable"]
        MCP["MCP server\nserver/ · FastMCP\nstdio, JSON-RPC"]
    end

    REGISTRY["ExtensionCommandRegistry\nlib/extensions/\nsingle source of truth"]

    CLI -->|"ext_registration.py\nauto-generates per-group cmds"| REGISTRY
    MCP -->|"extension_list_commands\nextension_invoke"| REGISTRY

    REGISTRY --> OPS["ops/\noperation handlers\nmanifest.py declares all"]

    OPS --> LAUNCH["lib/launch/\nbuild_launch_context()\nexecution"]
    OPS --> STATE["lib/state/\nper-spawn state.json\nwork items"]
    OPS --> CATALOG["lib/catalog/\nmodel resolution\nagent/skill loading"]
    OPS --> CONFIG["lib/config/\nsettings precedence\nworkspace config"]

    LAUNCH --> HARNESS["lib/harness/\nadapters\nClaude · Codex · OpenCode · Cursor · Pi"]
    LAUNCH --> STATE
```

## Entry Points

| Surface | Entry point | Transport | Auth |
|---------|-------------|-----------|------|
| CLI | `meridian` (cyclopts) + `ext_cmd.py` | stdio | n/a |
| MCP | `meridian serve` (FastMCP) | stdio, JSON-RPC | n/a |

The MCP server exposes exactly **two tools**: `extension_list_commands` and `extension_invoke`. The CLI auto-generates per-group commands from the registry via `ext_registration.py`. Both call the same underlying `sync_handler` or `ExtensionCommandDispatcher`.

## Data Flow: User Request → State

```mermaid
sequenceDiagram
    participant U as User / caller
    participant S as Surface (CLI/MCP/HTTP)
    participant R as ExtensionCommandRegistry
    participant O as ops/ handler
    participant L as lib/launch/
    participant H as lib/harness/
    participant St as lib/state/

    U->>S: meridian spawn -a coder -p "..."
    S->>R: lookup command
    R->>O: dispatch to spawn_create_sync
    O->>L: build_launch_context(SpawnRequest, LaunchRuntime)
    L->>L: resolve policies, permissions, prompt, argv, env
    L->>St: start_spawn() — append queued event
    L->>H: subprocess / HarnessConnection
    H-->>L: stream events
    L->>St: mark_running → exited → finalize_spawn
    St-->>U: terminal status
```

## Key Subsystems

```mermaid
graph LR
    subgraph lib
        CORE["core/\nID types · OutputSink · depth\nSpawnLifecycleService · Clock"]
        LAUNCH["launch/\nbuild_launch_context()\nthree driving adapters"]
        STATE["state/\nper-spawn state.json · reaper\nwork items · paths"]
        HARNESS["harness/\nadapter protocol\nconnections subpackage"]
        CATALOG["catalog/\nmodel aliases · profile loading"]
        CONFIG["config/\nRuntimeOverrides · TOML\nworkspace config"]
        CONTEXT["context/\nwork/kb path resolution"]
        SAFETY["safety/\nbudget · guardrails\npermissions · redaction"]
        HOOKS["hooks/\nlifecycle event dispatch"]
        PLATFORM["platform/\nfile locking · process kill\ndeferred OS imports"]
        STREAMING["streaming/\nHarnessConnection\nSpawnManager"]
        EXT["extensions/\nExtensionCommandRegistry\ndispatcher · context"]
        OPS["ops/\nmanifest.py\nspawn · session · work · ..."]
    end

    LAUNCH --> CORE
    LAUNCH --> HARNESS
    LAUNCH --> STATE
    LAUNCH --> SAFETY
    LAUNCH --> CONFIG
    LAUNCH --> CATALOG
    STATE --> PLATFORM
    OPS --> LAUNCH
    OPS --> STATE
    OPS --> CATALOG
    HOOKS --> CORE
```

## Extension System as Single Source of Truth

`ops/manifest.py` declares every user-facing operation as an `ExtensionCommandSpec` via `ExtensionCommandSpec.from_op()`. The registry becomes the authority — no command can exist in the CLI but not the MCP server, or vice versa. Surface membership controls reachability per surface.

This design means:
- Adding a new command = define an `ExtensionCommandSpec` in `manifest.py`
- Removing legacy wiring = delete old CLI registration; registry takes over
- Command documentation = queryable via `extension_list_commands`

## Related Pages

- [launch-system.md](launch-system.md) — composition factory, three driving adapters
- [state-system.md](state-system.md) — per-spawn state, reaper, work items
- [../codebase/guide.md](../codebase/guide.md) — how to navigate and change things
