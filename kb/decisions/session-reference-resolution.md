# Decision: Centralize Session Reference Resolution

## Context

Meridian had two divergent code paths for resolving the same session/spawn/chat IDs:

- `session_log.py::resolve_target()` — robust, with spawn-output detection and primary-meta fallback
- `reference.py::resolve_session_reference()` — thin, no fallbacks, broke `--from p123` when `harness_session_id` was missing

This meant `session log p123` could work while `--from p123` failed for the same spawn.

## Decision

Extract the shared "harness session ID recovery" logic into a new read-only module (`reference_recovery.py`) and make `resolve_session_reference()` use it. Keep transcript-source policy (native transcript vs spawn output vs live output) in `session_log.py`.

## Recovery Provenance

Four levels, in order of authority:

1. `SESSION_STORE` — from session records (chat refs)
2. `SPAWN_ROW` — from spawn record directly
3. `PRIMARY_META` — from primary spawn metadata
4. `DETECTED_UNVERIFIED` — from harness adapter detection

The first three are authoritative enough for `--continue`/`--fork`. `DETECTED_UNVERIFIED` is NOT — it is only usable by `session_log` for transcript verification.

## API Changes

- `ResolvedSessionReference` gains `recovery: RecoveryResult | None`
- `effective_harness_session_id` — any recorded or recovered ID
- `authoritative_harness_session_id` — excludes `DETECTED_UNVERIFIED`
- `missing_harness_session_id` — now considers authoritative recovery

## Consumer Behavior

| Consumer | Uses | Accepts detected? |
|----------|------|-------------------|
| `session log` | `effective_harness_session_id` | Yes |
| `--continue` / `--fork` | `authoritative_harness_session_id` | No |
| `--from` | `effective_harness_session_id` (best-effort) | Yes (omitted if unavailable) |

`--continue` and `--fork` treat authoritative recovery as launch authority, not
diagnostic metadata. If the source row lacks a raw harness session ID but recovery
finds one from session records, the spawn row, or primary metadata, the follow-up
launch uses that recovered ID. `DETECTED_UNVERIFIED` remains excluded because a
same-session operation must not resume or fork against a guessed session.

## Quit Message

`PrimaryLaunchOutput` now carries `continue_chat_id` separately from `continue_ref`. The resume command prefers the chat ID:

```
To continue with meridian:
meridian --continue c123
```

The harness UUID is still available for diagnostics and plumbing.

## Date

2026-05-06
