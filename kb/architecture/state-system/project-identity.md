# Project identity

Committed project identity survives a checkout move; an ad-hoc identity is derived from the canonical path and leaves the directory untouched. Reads do not materialize either kind of identity. See [state decisions](../../decisions/state.md#d-two-tier-identity) and the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

Control sockets use a short, fixed per-user temporary directory because runtime paths can exceed Unix socket limits and per-process temporary directories may differ between caller and runner. A stale-socket sweep uses a datagram probe: a stream probe can refuse a live macOS listener with a full backlog. See `src/meridian/lib/streaming/.context/CONTEXT.md`.

## Retention

Global pruning is explicit. Ad-hoc state can become shadowed when a project is initialized, so the global listing keeps it discoverable. The pruning activity check uses multiple signals: relying on the hourly origin-file touch alone leaves a gap near its boundary. See [state decisions](../../decisions/state.md#d-two-tier-identity).

