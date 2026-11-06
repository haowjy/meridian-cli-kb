# State system

Files are authoritative; SQLite indexes are disposable projections. Read paths may derive a view without changing state, while repair paths make durable changes. See [state model](../../concepts/state-model.md), [state decisions](../../decisions/state.md), and the source checkout's `src/meridian/lib/state/AGENTS.md` and `.context/CONTEXT.md`.

- [Spawn state](spawn-state.md) and [spawn launches](spawn-launches.md)
- [Session state](session-state.md) and [native session binding](../native-session-binding.md)
- [Portable history](portable-history.md)
- [Durability and locking](durability-and-locking.md)
- [Derived status](derived-status.md)
- [Project identity](project-identity.md) and [work items](roots-and-work-items.md)

