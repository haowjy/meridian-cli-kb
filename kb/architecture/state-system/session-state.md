# Session State


Sessions track harness session IDs, work-item attachment, primary-spawn relationships,
and lifecycle (created → active → closed). `sessions.jsonl` is the sole authority
for those facts. `sessions-index.sqlite3` is a metadata-only projection used for
direct chat-ID and requested-subset reads. Cross-record browse and history discovery
use the separate disposable history index; transcript bodies and full-text content
are in neither SQLite projection.

`session_journal.py` certifies ordinary appends with a derived epoch in
`sessions-append-state.json`. The certificate records source identity and file state,
not session facts. The index consumes only complete JSONL records and trusts suffix
growth only when the current certificate matches both the source and the index's epoch.
Replacement, truncation or rewrite, an absent or stale certificate, schema mismatch,
and corruption force replay from authoritative JSONL. A busy or unavailable index
falls back promptly to truncation-tolerant journal projection and is not deleted merely
for being busy.

New primary launches persist their canonical `spawn_id` in the journal. Historical
sessions are enriched by `session_aggregate.py`, which joins missing relationships from
authoritative spawn rows. The projection records the published-spawn generation around
that join; publication changes the generation, so a cached negative result cannot hide
a relationship that appears later. This aggregate owns the cross-store join and keeps
the session and spawn persistence leaves from importing each other.

Normal browse uses the history index's recent-session view, which can include
available ZIP and inert historical records. Preview and re-entry still resolve
authoritative session/spawn facts before acting. Scoped search uses indexed
associations to plan its corpus, then reads transcript authority. Legacy sessions
without a recorded relationship may require one generation-aware batch scan. A
recorded but unreadable relationship (missing primary row, wrong spawn kind, or wrong
owning chat) is a separate exceptional case: transcript target resolution performs
at most one batch-wide legacy scan so deep search can recover related histories.
Readable recorded relationships never take that global-scan path.

Native capture preparation is narrower than presentation discovery. It binds the
exact completed primary aggregate and resolves normalized harness/native identity
from state, primary metadata, and that generation's session facts. Ambiguous or
conflicting selection refuses capture; known same-runtime owners are a conservative
read precondition, not permission to resume or proof against external writers. Native
identity never changes re-entry authorization. See
[Portable history](portable-history.md) for the remaining snapshot divergence.

`sessions.jsonl` also records `model_selection` events (legacy `initial_seed` or
`invocation_started`). That is Meridian-selected intent for later continue, not
an observation of the last model that executed. Writes happen only at
accepted-running. Native ID may bind later without moving the log row. The
original launch-policy snapshot is not rewritten.

Session-ID counter (`session-id-counter`) is monotonically incremented under `platform.locking.lock_file()` so concurrent spawns never collide.

Per-session files under `sessions/<chat_id>/`:
- `<chat_id>.lock` — held for active session duration
- `<chat_id>.lease.json` — PID + generation token for staleness detection


## Related Pages

- [State system overview](overview.md) — state roots and subsystem map
- [Spawn state](spawn-state.md) — per-spawn authority and artifact lifetime
- [Portable history](portable-history.md) — cross-record discovery, exact identities, ZIP retention, and inert restore
- [Reconciliation](reconciliation.md) — read-time projection and durable repair
