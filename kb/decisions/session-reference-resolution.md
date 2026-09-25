# Decision: Centralize Session Reference Resolution

**Status:** Current. The recovery chain was narrowed on 2026-09-24 by the
[native session identity decision](native-session-identity.md): only exact recorded
state recovers an ID, and native discovery and primary metadata are no longer
recovery levels.

## Context

Meridian had two divergent code paths for resolving the same session/spawn/chat IDs:

- `session_log.py::resolve_target()`: robust, with spawn-output detection and primary-meta fallback
- `reference.py::resolve_session_reference()`: thin, no fallbacks, broke `--from p123` when `harness_session_id` was missing

This meant `session log p123` could work while `--from p123` failed for the same spawn.

## Decision

Extract the shared "harness session ID recovery" logic into one read-only module
(`reference_recovery.py`) and make `resolve_session_reference()` use it. Keep
transcript-source policy in the session-target layer.

## Recovery Provenance

Two levels, both exact recorded state:

1. `SESSION_STORE`: the chat's immutable native binding (chat refs)
2. `SPAWN_ROW`: the spawn record's mirrored ID (spawn refs)

Primary metadata (`PRIMARY_META`) and adapter file detection (`DETECTED_UNVERIFIED`)
were removed. A mirror cannot supply an identity the chat never bound, and a
detected file is exactly the discovery the identity decision forbids. A tracked
reference whose recorded harness is empty has no authoritative ID. `--continue`
and `--fork` refuse it instead of inferring a harness from adapters.

## API

- `ResolvedSessionReference.recovery: RecoveryResult | None`
- `effective_harness_session_id`: recorded or recovered ID
- `authoritative_harness_session_id`: `None` for a tracked ref without a recorded harness
- `missing_harness_session_id`: tracked and not authoritative

`--continue`/`--fork` use the authoritative ID. An unbound or missing one fails with an
explicit error, not a discovery hint. `--from` uses the effective ID as a
best-effort pointer and omits it if unavailable.

## Quit Message

`PrimaryLaunchOutput` carries `continue_chat_id` separately from `continue_ref`. The resume command prefers the chat ID:

```
To continue with meridian:
meridian --continue c123
```

The harness UUID is still available for diagnostics and plumbing.

## Date

2026-05-06; recovery narrowed 2026-09-24.

## Related

- [../architecture/native-session-binding.md](../architecture/native-session-binding.md)
