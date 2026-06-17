# Codebase: Harness Adapters

`lib/harness/` is the mechanism side of the policy/mechanism split. Adapters translate a typed `SpawnParams` struct into a runnable subprocess and extract results back. Adding a harness touches only the adapter file and registry — no shared code.

## Connection-Level Resident and Scope Facts

Streaming adapters expose three optional facts on `HarnessConnection`:

- `resident_backend`: the resident follow-up seam used by `ResidentDrainCoordinator`.
  Codex/OpenCode return a `ResidentBackendControl`; Claude, Cursor, and Pi spawned
  RPC return `None`.
- `primary_event_scope`: the parent-conversation identity used when one event stream
  can include both parent and child work. Codex returns the main turn `threadId`;
  OpenCode returns the launched parent `sessionID`. The drain loop passes this scope
  to `terminal_outcome()`, `activity_transition()`, and `clears_signal()`.
- `scope_snapshot`: process-scope facts for a managed backend subprocess. Primary
  attach reads this snapshot instead of reconstructing containment from raw PID
  fields.

Primary event scope is a classification boundary, not an event filter. Child Codex
thread events and child OpenCode task-session events are still persisted to
`history.jsonl` and visible through `meridian session log`; they just cannot complete,
fail, or clear signals for the parent spawn. `PrimaryEventScope` is the single drain
contract for this behavior; semantic helpers and coordinators should not grow
harness-specific compatibility parameters. OpenCode report extraction uses the same
boundary so child assistant text cannot become the parent `report.md`.

The unscoped fallback is harness-specific. Codex keeps the legacy conservative behavior:
an unscoped `turn/completed` still counts when the main thread is known. OpenCode does
not treat unscoped `session.idle` / `session.error` as parent events once the parent
session is known, because OpenCode's global SSE stream can include child task sessions.
OpenCode session-id and report artifact parsing is centralized in
`harness/opencode_report.py`; the extractor delegates there rather than duplicating
OpenCode event-shape logic in generic helpers.

Managed Codex/OpenCode backends launch through `launch_managed_backend()`, which
records the backend scope and links the detached backend to the worker lifetime where
the platform can enforce it (Linux parent-death signal, Windows Job Object; other
POSIX platforms degrade to isolated-session containment plus recorded scope cleanup).
Adapter code should expose the resulting `scope_snapshot`, not write a separate
backend lifecycle sidecar.

`BackendLivenessPolicy` is shared by managed Codex/OpenCode connections and owns the
classification of quiet streams, active turns, in-flight requests, dead backend PIDs,
and intentionally suppressed waits. Adapter code feeds it signals; callers consume
its decisions.

## Adapter-Owned Spawn Contract

Each harness adapter declares its spawn-usage-contract phrasing variants via
`RunPromptPolicy.spawn_usage_contract_variants()` returning a `SpawnUsageContractVariants`.
This is the adapter's *semantic interface* — it tells `build_guidance_blocks()` how
to phrase harness-specific spawn instructions (e.g., Claude's `run_in_background` vs
a generic "your harness's background execution").

The contract variants carry:
- `intro_line` — how the harness launches background work
- `double_wrap_bullet` — warning about double-backgrounding (harness-specific)
- `timeout_bullet` — timeout behavior phrasing

`build_guidance_blocks()` composes the caller-supplied contract with no `HarnessId`
conditional — it receives a pre-built contract string and places it as a first-class
composition block (`spawn-contract`, `GuidancePhase.GUIDANCE`, ordinal 10). The policy
(`has_spawn_capability()`) gates whether the contract is injected; the adapter owns
the phrasing.

See [../concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md#spawn-capability-gating).

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

## Claude Native Agent Routing

Claude Code's `Agent` tool is gated by Mars `[settings.meridian.agent_copy]`. When
`harnesses = ["claude"]` and `.claude` is in targets, generic `Agent` is allowed
(the native agent surface is Meridian-owned). Without agent copy, `Agent` is denied
and delegation routes through `meridian spawn`. Built-in subagents (`Explore`,
`Plan`, `General-purpose`/`general-purpose`) are always denied unconditionally.

Detection runs in `bind_launch_context()` via `project_has_claude_agent_copy()` (reads
`mars.toml` and `mars.local.toml`). The result flows through
`ResolvedLaunchSpec.claude_native_agents_enabled` to `project_claude.py`, which strips
`Agent` and `Agent(...)` from all allowed-tool sources when disabled. Parent-inherited
allowed-tool tails cannot re-add denied agents — the projection layer filters them.

Delegation guidance is rendered by Mars into the `prompt_surface.inventory_prompt` bundle
field alongside the `# Meridian Agents` block. Meridian embeds the Mars-rendered string
verbatim — there is no Python-side guidance renderer.

See [../concepts/harness-abstraction.md](../concepts/harness-abstraction.md#claude-native-agent-routing) for the full policy.

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
