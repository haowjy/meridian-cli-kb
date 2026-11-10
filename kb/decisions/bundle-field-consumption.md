# Launch-Bundle Field Consumption

## Applied, reported, or deliberately unread

Every launch-bundle field is applied, reported as unsupported, or deliberately left unread with a reason. Silent drops make a profile appear to work when it does not. The unread fields and reasons are in `src/meridian/lib/launch/bundle_adapter.py`; harness behavior is in `docs/agent-profiles.md`.

## Native config

Codex applies `harness-overrides` except for Meridian-reserved settings; other harnesses report it unsupported. Reserved keys prevent passthrough from undoing compiled policy. A package's native config can change other Codex settings, including providers and MCP commands, and is trusted like a hook.

## Compaction thresholds

Token and percentage thresholds project to the corresponding native harness settings. See `docs/agent-profiles.md` for supported ranges and harnesses.

## MCP permission names

`tools.mcp` names permissions, not server definitions. Passing the names as server configuration would configure nothing. Per-launch server injection is not built; it would need real server definitions in the bundle.
