# Mars compiler

The mars-agents checkout's `src/compiler/AGENTS.md` describes the compiler and ownership seams. Sync and plugin builds share target selection and lowering so a plugin contains the same selected content that sync installs. See [package management decisions](../decisions/package-management.md) and [sync model](../concepts/package-management/sync-model.md).

## Config-entry selection and collisions

Hooks and MCP entries use one target selection so a server's configuration and installed directory cannot come from different winners. An entry requiring a packaged directory is withheld when an unmanaged or modified destination prevents installation. Exact lock evidence controls removal; matching content alone does not establish ownership. See [sync model](../concepts/package-management/sync-model.md) and [package management decisions](../decisions/package-management.md).

Dependency exclude filters do not currently match MCP server names ([#237](https://github.com/haowjy/mars-agents/issues/237)); agent and skill include filters also do not narrow MCP servers ([#228](https://github.com/haowjy/mars-agents/issues/228)).
