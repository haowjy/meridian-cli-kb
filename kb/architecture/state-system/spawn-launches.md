# Spawn launches

A continue reopens the same spawn as a new launch so its identity and child relationships remain stable. Each launch has its own files and the runner holds the launch lock throughout, preventing an old report or signal from being attributed to a later run. See [spawn identity](../../decisions/spawn-identity.md) and the source checkout's `src/meridian/lib/state/AGENTS.md`.

## The reopen

The lock holder settles the previous launch and its owned scopes before committing the next. A continue can reparent the spawn and its subtree; cycle detection must walk the full stored ancestor chain because that chain can exceed the spawn depth limit.

## The fence

Operations that do not hold the launch lock pin the launch they observed. Otherwise a cancel, wait or late artifact writer could act on a newly reopened launch. A descendant that escapes cleanup while retaining the spawn ID may still issue spawn-level commands, so the fence is not full containment.

## Usage across launches

User-facing usage aggregates launches. Claude and Codex report session-cumulative totals, so the next launch subtracts the prior recorded cumulative value rather than double-counting it. A v3 row lacks that seed on its first reopen; its first launch can be counted twice. Forks also need the matching source session's recorded seed. See the source checkout's `src/meridian/lib/state/AGENTS.md`.

## Waits, owed results and delivery

Every consumption record used to be keyed by child ID, so once launch 1 of a
child was consumed, launch 2 would never be delivered. Every key is now
`(spawn_id, n)`, written as `pN:n` inside the files. The CLI never shows that
key.

**Pinned waits.** `spawn wait pN` pins the launch it resolved. Once `launch.n`
moves past the pin, it reads `launches/<n>/launch.json` once and returns that
launch's result. Waits re-run as new processes (`--yield-after-secs`, Pi
checkpoints), so the pin is stored in the caller's ledger as
`wait_pins: {pC: {n, expires_at}}`. It dies when launch n is consumed by any
path, when the caller continues pC, or when it expires: each poll refreshes it to
one effective yield interval ahead. A wait that lost its pin pins the current
launch, and the earlier result is still owed, so nothing is lost.

**Owed launches.** Owed discovery selects children by their current parent and
launch mode, so after a reopen the row shows only n+1, and an unconsumed
background launch n would vanish. Whenever launch n was a background launch,
`reopen_spawn` appends `{n, parent_id}` to `owed_launches`; the history index
(v13) projects the list into a side table, and owed discovery unions current
children with those entries.

**Ledger files.** New files, because bumping the version in place fails both
ways: an older Python build reads an unknown version as absent and rewrites the
whole document, and an older Pi extension throws on it and stops delivering.

| Read, never written | Read and written |
|---|---|
| `spawns/<p>/observed-spawns.json` | `spawns/<p>/observed-launches.json` (observed, waiting, leases, `wait_pins`, `delivered`) |
| `pi-bash/<p>/delivery-receipts.json` | `pi-bash/<p>/launch-receipts.json` |
| `pi-bash/<p>/cleared-spawns.json` | `pi-bash/<p>/cleared-launches.json` |
| `pi-bash/<p>/observed-spawns.json` (mirror) | retired |

A bare legacy entry `pC` reads as `(pC, 1)`, which is exact: older builds can't
read a v4 row, so they only ever consumed launch 1. Retiring the legacy reads
and collapsing `owned_parent_ids` to the caller waits on a deprecation window
(#836).

**`spawn collect` and `spawn ack` keep bare IDs.** Launch keys on the wire break
Claude delivery, because the mod's ID pattern rejects `:`, and they let a bare
`ack` consume a running launch. `collect`
returns bare, de-duplicated IDs and records what it handed out in the caller's
`delivered` map; `ack pN` consumes exactly those launches, or with no record the
spawn's finished, owed, unconsumed launches, never a running one. A delivered
launch that is no longer current carries its report inline (64 KiB shared
budget), because `spawn report show pN` now shows the next launch. Contract:
`src/meridian/pi_runtime/.context/delivery-contract.md`; Claude's side is
[child-result delivery](../../concepts/child-result-delivery.md).

**Hooks and telemetry.** `spawn.created` fires on creation only; every launch
emits `spawn.started` and `spawn.finished`. Lifecycle event IDs hash
`(spawn_id, event_type, n − 1)`, so launch 1 keeps the ID older builds gave it
(hook dedupe still holds) and launch 2 doesn't dedupe against it. Payloads carry
`spawn_id` and no launch number.

## Older builds on the same runtime

Older readers can reject newer rows or repeat delivery after an upgrade. An older Pi extension still loaded after a continue can mark launch 1 consumed without showing it; restart Pi sessions on upgrade. See the source checkout's `docs/upgrading.md` for the compatibility procedure.
