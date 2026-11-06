# Extension System

CLI and MCP share command specifications, with surface access set per command. This prevents separately implemented surfaces from drifting. The source guidance in `src/meridian/lib/extensions/AGENTS.md` describes registration, capability checks, and the first-party boundary.

See [startup](../architecture/startup-pipeline.md) for command registration.
