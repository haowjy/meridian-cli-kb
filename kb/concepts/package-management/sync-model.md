# Sync Model

Individual writes are atomic and sync runs are serialized, but a sync is not a transaction with rollback. Unreadable owned input must not be mistaken for an absent desired output: preserving ownership evidence allows a later retry instead of deleting an output whose desired state is unknown.

## Pending canonical writes

Durable write intent lets Mars recover ownership of a new canonical output after interruption. Recovery verifies both prior lock state and output identity before claiming it; matching bytes alone are insufficient evidence. Native and config outputs remain outside that recovery boundary. See [package management decisions](../../decisions/package-management.md), `src/sync/AGENTS.md`, and `src/target_sync/.context/CONTEXT.md` in mars-agents.
