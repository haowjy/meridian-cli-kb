# State system

Files are authoritative; SQLite indexes are disposable projections that can always be rebuilt. Reads may derive a view but never change state; repair paths make the durable changes. Contracts: the source checkout's `src/meridian/lib/state/AGENTS.md` and `.context/CONTEXT.md`.

`-C` selects the project root ahead of an inherited `MERIDIAN_PROJECT_DIR` and the cwd. Package discovery can walk to an ancestor independently of [project identity](project-identity.md). Runtime roots under `~/.meridian/projects/` derive from identity; work items live in the separately configured context root because they are shared task artifacts.

- [Project identity](project-identity.md): committed vs ad-hoc identity, control sockets
- [Spawn identity and launches](spawn-launches.md) and [spawn outcome](spawn-outcome.md)
- [Session state](session-state.md) and [native session binding](native-session-binding.md)
- [Portable history](portable-history.md) and [native transcript reads](native-transcript-reads.md)
- [Durability and locking](durability-and-locking.md)
- [Work items and checkout links](work-items.md)
- [Compatibility between builds](compatibility.md) and [per-operation cost](per-operation-cost.md)
