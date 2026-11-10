# State system

Files are authoritative; SQLite indexes are disposable projections that can always be rebuilt. Reads may derive a view but never change state; repair paths make the durable changes. Contracts: the source checkout's `src/meridian/lib/state/AGENTS.md` and `.context/CONTEXT.md`.

The project root is resolved as [config precedence](../../concepts/config-precedence.md) describes. Package discovery can walk to an ancestor independently of project identity. Runtime roots under `~/.meridian/projects/` derive from identity; work items live in the separately configured context root because they are shared task artifacts.

- [Project identity](project-identity.md): committed vs ad-hoc identity, control sockets
- [Spawn identity and launches](spawn-launches.md), [spawn state](spawn-state.md), [derived status](derived-status.md)
- [Session state](session-state.md) and [native session binding](../native-session-binding.md)
- [Portable history](portable-history.md) and [native transcript reads](../native-transcript-reads.md)
- [Durability and locking](durability-and-locking.md)
- [Work items and checkout links](work-items.md)
- [Compatibility between builds](../../concepts/compatibility.md)
