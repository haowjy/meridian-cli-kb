# Workspace resolution

The source checkout's `docs/configuration/workspace.md` describes resolution and diagnostics. Missing committed roots can be normal in partial checkouts; a missing local override is more likely a stale or mistyped path. Both are reported and excluded from projection. See [workspace decisions](../../decisions/workspace.md#d44).

