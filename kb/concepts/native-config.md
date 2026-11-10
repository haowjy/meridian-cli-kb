# Harness Override Passthrough

`harness-overrides.<harness>` carries target-native configuration alongside portable policy without claiming that its keys have cross-harness meaning. Meridian reserves settings it owns so passthrough cannot silently override projected policy. A Codex override can still change other settings, including providers and MCP commands, so a package carrying one is trusted code.

See [bundle field decisions](../decisions/bundle-field-consumption.md#native-config) and `docs/agent-profiles.md` for the current application and reporting rules.
