# Architecture: State System

Meridian's authoritative state is files. There is no state service or hidden
in-memory authority; SQLite is allowed only as a disposable projection that can be
rebuilt from authoritative files. The state system makes writes atomic, reads
crash-tolerant, and recovery derivable from disk. Read paths can project a reconciled
view without side effects; repair paths make the durable changes.

See [concepts/state-model.md](../../concepts/state-model.md) for the mental model. This page explains the mechanics.

## Split State Layout

State divides across committed configuration, a user-local runtime root, and a configured context work root:

```
meridian.toml
  [project]
  id = "three-word-id"              — committed authoritative identity

~/.meridian/projects/.locks/<id>.lock — project-lifetime gate
~/.meridian/projects/<id>/          — user runtime, never committed
  sessions.jsonl                    — session events
  legacy-native-import-v1.json      — one-time legacy key import outcome
  history-index/                    — disposable projections, named by schema
    history-v<N>.sqlite3            — metadata index (N = SCHEMA_VERSION; 10 on main)
    native-search-v1.sqlite3        — native-keyed FTS5 search projection
    pending-v<N>/                   — dirty-source intent for schema N
    history.sqlite3 · pending/      — 0.6.7's schema-2 index; left untouched
  history-index-init-failure-v<N>.json — latched genuine build failure
  history-archives/                 — ZIP receipts and private restore stages
  locks/history-*-v<N>.lock         — projection coordination (per schema)
  locks/history-mutation.lock       — authority mutation gate (shared by all builds)
  session-id-counter · spawn-id-counter
  sessions/ · locks/
  spawns/
    v2-format.json · .staging/<unique>/
    <spawn-id>/
      state.json                    — authoritative spawn row (schema v3)
      starting-prompt.md · prompt.md · report.md · launch.lock
      stderr.log · params.json · tokens.json
      pi-lifecycle.json · runner-lifecycle.jsonl · process_scopes.json
      attempt-N/                    — legacy retry evidence, read-only
  cache/ · .migrations.json

<context.work root>/<slug>/         — context-resolved work state and artifacts
```

Conversation content is the harness's native file, not a spawn artifact
([native-only history](../../decisions/native-only-history.md)). The runner's old
event stream `history.jsonl` and its `last-observed-event.json` checkpoint are no
longer written (PR 3). Older spawn directories may still hold them as inert bytes
until `session archive --prune-runner-history` removes them
([prune](../../operations/session-archive-pruning.md#runner-history-prune)).
`pi-lifecycle.json` holds the last Pi phase and cleanup status for the current turn
([attempt facts and delivery](../attempt-facts-and-delivery.md#pi-lifecycle-sidecar)).

Older runtimes may still contain `artifacts/<id>/`, the former second copy of
per-spawn files. Launch, finalization, and query paths never use it. Cleanup code
recognizes it only to remove legacy bytes: runner-history pruning can delete retired
stream files inside it, and stale-spawn pruning deletes the whole legacy directory.

The project id selects the runtime directory: `[project].id` for a committed
project, `adhoc.<slug>.<hash12>` for a marker-less directory, whose root also
holds `origin.json`. `lib/state/project_identity.py` is the only resolver
([project identity](project-identity.md)). Repo-local `.meridian/` is not a
state root and `.meridian/id` is not read.

Control sockets live outside spawn directories in the per-user POSIX temp
root. Work items live under `[context.work]` and archive beside that work root.
See `docs/configuration.md` in meridian-cli for context-path resolution.

## Pages in This Domain

- [Spawn state](spawn-state.md) — per-spawn rows, status transitions, publication lifetime, and legacy migration
- [Session state](session-state.md) — the global `sessions.jsonl` log (sessions are not per-chat yet), index projection, session files, its remaining full-log readers, and the accepted per-chat plan (#418)
- [Native session binding](../native-session-binding.md) — immutable chat-to-native-key binding across harnesses
- [Attempt facts and delivery](../attempt-facts-and-delivery.md) — emit path, per-harness one-turn folds, and the Pi lifecycle sidecar
- [Portable history](portable-history.md) — transcript identity, dirty-source projection, verified ZIP retention, and inert restore
- [Durability and locking](durability-and-locking.md) — atomic publication, lock semantics, and lock order
- [Derived status](derived-status.md) — status from the runner-held `launch.lock`: `read_derived`, stored vs derived, the one dead-launch finalize path, the reaper's cleanup claims (identity-v2 PR 3)
- [Project identity](project-identity.md) — committed and ad-hoc tiers, `origin.json`, ad-hoc retention
- [State roots and work items](roots-and-work-items.md) — work-item store and project/runtime root resolution

## Related Pages

- [System overview](../system-overview.md) — where state fits in the overall architecture
- [State model](../../concepts/state-model.md) — mental model for dual-root and event sourcing
- [State decisions](../../decisions/state.md) — rationale and superseded alternatives
