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
  history-index/                    — disposable projections
    history.sqlite3                 — metadata index (schema 6): discovery, aliases, previews
    native-search-v1.sqlite3        — native-keyed FTS5 search projection
    pending/                        — dirty-source intent
  history-archives/                 — ZIP receipts and private restore stages
  locks/history-*.lock              — stable projection/mutation coordination
  session-id-counter · spawn-id-counter
  sessions/ · locks/
  spawns/
    v2-format.json · .staging/<unique>/
    <spawn-id>/
      state.json                    — authoritative spawn row (schema v3)
      starting-prompt.md · prompt.md · report.md · heartbeat
      stderr.log · params.json · tokens.json
      pi-lifecycle.json · attempt-N/ · runner-lifecycle.jsonl · process_scopes.json
  artifacts/ · cache/ · .migrations.json

<context.work root>/<slug>/         — context-resolved work state and artifacts
```

Conversation content is the harness's native file, not a spawn artifact
([native-only history](../../decisions/native-only-history.md)). The runner's old
event stream `history.jsonl` and its `last-observed-event.json` checkpoint are no
longer written (PR 3). Older spawn directories may still hold them as inert bytes
until `session archive --prune-runner-history` removes them
([prune](../../operations/session-archive-pruning.md#runner-history-prune)).
`pi-lifecycle.json` holds the last Pi phase and cleanup status per attempt
([attempt facts and delivery](../attempt-facts-and-delivery.md#pi-lifecycle-sidecar)).

`[project].id` selects the runtime directory. `user_paths.py` still reads a
legacy `.meridian/id` when config has no ID; the first write migrates that value
(or generates a three-word ID) into `meridian.toml` by atomic replacement.
Repo-local `.meridian/` is not an active state root.

Control sockets live outside spawn directories in the per-user POSIX temp
root. Work items live under `[context.work]` and archive beside that work root.
See `docs/configuration.md` in meridian-cli for context-path resolution.

## Pages in This Domain

- [Spawn state](spawn-state.md) — per-spawn rows, status transitions, publication lifetime, and legacy migration
- [Session state](session-state.md) — authoritative session journal, index projection, and session files
- [Native session binding](../native-session-binding.md) — immutable chat-to-native-key binding across harnesses
- [Attempt facts and delivery](../attempt-facts-and-delivery.md) — emit path, per-harness attempt folds, and the Pi lifecycle sidecar
- [Portable history](portable-history.md) — transcript identity, dirty-source projection, verified ZIP retention, and inert restore
- [Durability and locking](durability-and-locking.md) — atomic publication, lock semantics, and lock order
- [Reconciliation](reconciliation.md) — read-time projections, explicit repair, and liveness checks
- [State roots and work items](roots-and-work-items.md) — work-item store and project/runtime root resolution

## Related Pages

- [System overview](../system-overview.md) — where state fits in the overall architecture
- [State model](../../concepts/state-model.md) — mental model for dual-root and event sourcing
- [State decisions](../../decisions/state.md) — rationale and superseded alternatives
