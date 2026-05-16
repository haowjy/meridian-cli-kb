# Codebase: Harness Adapters

`lib/harness/` is the mechanism side of the policy/mechanism split. Adapters translate a typed `SpawnParams` struct into a runnable subprocess and extract results back. Adding a harness touches only the adapter file and registry — no shared code.

## Capability Matrix

| Capability | Claude | Codex | OpenCode | Direct |
|------------|--------|-------|----------|--------|
| `supports_stream_events` | ✓ | ✓ | ✓ | ✗ |
| `supports_session_resume` | ✓ | ✓ | ✓ | ✗ |
| `supports_session_fork` | ✓ | ✓ | ✓ | ✗ |
| `supports_native_skills` | ✓ | ✓ | ✓ | ✗ |
| `supports_native_agents` | ✓ | ✗ | ✗ | ✗ |
| `supports_programmatic_tools` | ✗ | ✗ | ✗ | ✓ |
| `supports_primary_launch` | ✓ | ✓ | ✓ | ✗ |
| `supports_native_file_injection` | ✗ | ✗ | ✗ | ✗ |

`supports_native_file_injection` is `False` for all current harnesses. References are routed as `"inline"` (rendered into prompt text) or `"omitted"` (empty-body files skipped).

## Related Pages

- [../architecture/launch-system.md](../architecture/launch-system.md) — how adapters plug into the launch factory
- [../architecture/app-server.md](../architecture/app-server.md) — how connections/ is used by the app server
- [../concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — policy/mechanism split mental model
- [../concepts/composition-pipeline.md](../concepts/composition-pipeline.md) — project_content() semantic IR pattern
- [../concepts/workspace-projection.md](../concepts/workspace-projection.md) — workspace root projection, Codex remote TUI attach, OpenCode env merging
- [../decisions/launch.md](../decisions/launch.md#d-primary-approval) — D-primary-approval: managed-primary Codex approval routing design
- [../decisions/workspace.md](../decisions/workspace.md#d47) — D47: projected_roots first-class field; D48: OpenCode merge-not-suppress
