# Codebase: Harness Adapters

`lib/harness/` is the mechanism side of the policy/mechanism split. Adapters translate a typed `SpawnParams` struct into a runnable subprocess and extract results back. Adding a harness touches only the adapter file and registry — no shared code.

## Capability Matrix

| Capability | Claude | Codex | OpenCode | Pi | Direct |
|------------|--------|-------|----------|----|--------|
| `supports_stream_events` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `supports_session_resume` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `supports_session_fork` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `supports_native_skills` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `supports_native_agents` | ✓ | ✗ | ✗ | ✗ | ✗ |
| `supports_programmatic_tools` | ✗ | ✗ | ✗ | ✗ | ✓ |
| `supports_primary_launch` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `supports_native_file_injection` | ✗ | ✗ | ✗ | ✗ | ✗ |

`supports_native_file_injection` is `False` for all current harnesses. References are routed as `"inline"` (rendered into prompt text) or `"omitted"` (empty-body files skipped).

`supports_native_skills` is `False` for Pi — Pi has a native skills system but Meridian skill projection into Pi is deferred. Skills are currently delivered via the system prompt channel.

## Pi-Specific Notes

**Dual launch path.** Pi is the only harness with fully separate launch configurations per role:
- **Primary:** native Pi TUI (`pi [--model ...] [--session/--fork ...]`), no `--mode rpc`, `meridian-spawn-watch` extension only
- **Spawned:** Pi RPC (`pi --mode rpc ... --no-extensions -e managed-bash.js -e meridian-spawn-watch.js`), prompt written to stdin, JSONL events drained from stdout

The split is enforced at projection time: `project_pi_native_tui.py` for primary, `project_pi_rpc.py` for spawned.

**Disk-backed quiescence.** Pi RPC stdout is the JSONL protocol stream only. Background
bash, nested child spawns, and implicit-wait notification markers are observed through
disk state (`spawns/<child>/state.json`, `pi-bash/<parent>/bash-records.json`,
`pi-bash/<parent>/last-notification.json`) by the streaming layer.

**Extension-based permission routing.** Pi returns an empty tuple from its permission-flag projector — Pi uses extension event hooks for permissions rather than CLI flags. This differs from Claude (`--dangerously-skip-permissions`) and Codex (its own flag set).

**Session isolation.** Pi isolates session storage via `PI_CODING_AGENT_SESSION_DIR` set in env_overrides. `MERIDIAN_PI_SESSION_ROLE` (primary/spawned) is injected so extensions can gate quiescence machinery to spawned-only sessions.

**Runtime resolution.** `PiRuntimeResolver` probes the installed `pi` binary before launch (`pi --version`, `pi --help` surface check). Fails fast with install guidance if binary is missing or incompatible. `MERIDIAN_PI_BINARY` env var overrides PATH discovery.

See [../architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) for the quiescence model.

## Related Pages

- [../architecture/launch-system.md](../architecture/launch-system.md) — how adapters plug into the launch factory
- [../architecture/app-server.md](../architecture/app-server.md) — how connections/ is used by the app server
- [../concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — policy/mechanism split mental model
- [../concepts/composition-pipeline.md](../concepts/composition-pipeline.md) — project_content() semantic IR pattern
- [../concepts/workspace-projection.md](../concepts/workspace-projection.md) — workspace root projection, Codex remote TUI attach, OpenCode env merging
- [../decisions/launch.md](../decisions/launch.md#d-primary-approval) — D-primary-approval: managed-primary Codex approval routing design
- [../decisions/workspace.md](../decisions/workspace.md#d47) — D47: projected_roots first-class field; D48: OpenCode merge-not-suppress
- [../architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) — Pi quiescence model, extension architecture, disk-backed coordination
