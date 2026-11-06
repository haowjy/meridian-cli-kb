# Package Resolution

Mars resolves the dependency graph before selecting items. This lets sync, package checking, and plugin builds share one selection result rather than applying filters independently. Engine-incompatible versions are excluded across resolution restarts; otherwise a restart could repeatedly choose the same incompatible candidate. Path and pinned sources have no alternate version to try.

Discovery uses conventions while allowing explicit hidden foreign imports; see [package management decisions](../../decisions/package-management.md) for the rationale. The current resolver contract is in `src/resolve/AGENTS.md` in mars-agents.
