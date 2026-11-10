# Mars launch bundle

Mars resolves package content, routing and portable tool policy into a launch bundle; Meridian applies the returned policy and composes its own prompt from the selected profile, loaded skills and per-run context. Mars owns capability-to-native tool names; Meridian owns harness permission mechanisms ([tool permissions](tool-permissions.md)). Field handling: the source checkout's `src/meridian/lib/launch/bundle_adapter.py` and `docs/agent-profiles.md`.

## Every field is applied, reported or deliberately unread

A silently dropped field makes a profile look like it works when it doesn't. So each bundle field is applied, reported as unsupported, or left unread with a stated reason in `bundle_adapter.py`. The bundle's precomposed prompt and scaffold slots are deliberately unread so there is one instruction surface; per-run prompt files stay with Meridian.

## `harness-overrides` passes native config through

`harness-overrides.<harness>` carries target-native configuration beside portable policy, without claiming its keys mean anything on other harnesses. Codex applies it; other harnesses report it unsupported. Meridian reserves the settings it owns, such as model, sandbox and approval, so passthrough can't undo compiled policy. A Codex override can still change other settings, including providers and MCP commands, so a package carrying one is trusted like a hook. Native configuration never goes into model instructions.

## Other fields

- **Compaction thresholds** project to each harness's native token or percentage setting.
- **`tools.mcp` names permissions, not servers.** Passing the names as server configuration would configure nothing. Per-launch MCP server injection isn't built; it would need real server definitions in the bundle.
