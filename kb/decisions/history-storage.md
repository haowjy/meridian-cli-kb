# Decisions: History Storage

History-storage decisions govern transcript authority, disposable indexes, retention,
restore, and native capture.

## Native capture temporal boundary: return to post-stop observation (2026-09-14)

A real OpenCode 1.18.29 continuation overtook an immediate first-idle snapshot,
which already included the next user turn. The universal read-at-completion
qualification proposed for #499 is therefore not valid. The independent contract
audit traced exact launch-end semantics to repair-design strengthening, not an
explicit original product requirement.

After the user clarified that existing capture should be repaired rather than a
new snapshot system invented, the selected design returns to post-stop observation
with exact completed-record association and explicit temporal/content scope. It may
include later continuation and does not certify past launch-end state. Keep the
existing maintenance position; carry primary_spawn_id, not latest cN. No universal
frontier ledger or new native ownership system. Exact aggregate selection/byte
verification, active/dependent protection, stream preservation and inert provenance
remain unchanged. The exact completed-primary handoff and capture-purpose identity
selection now implement this temporal boundary. Provider qualification, snapshot
publication, canonical validation, and integrated runtime approval remain open.

Codex paginated forks may depend on bounded ancestor rollouts. OpenCode's pinned
MessageV2 transcript reader uses session/message/part tables; a raw snapshot of
that scope is not a complete native-state backup. Neither a leaf-only fork copy
nor lossy rendered OpenCode events can be certified as preserved native authority.

Evidence limits: the Codex/Pi prober exhausted account usage before reporting;
primary recovered evidence and cleaned its positively identified orphan backend
and owned scratch. Codex persisted one startup response; Pi's current native
probe had an isolated credential-path mismatch; Claude authenticated persistence
remains untested. Do not claim four-harness runtime success. PR #494 was later
merged to `main` (2026-09-18) and released in v0.5.0.

Provenance: `work:next-minor-planning/design/native-capture-boundary-decision.md`;
`reviews/frontier-contract-audit.md`; `probes/frontier-opencode-claude.md`;
`probes/frontier-codex-pi.md`; `probes/native-content-scope-followup.md`.


## Intended History Storage

### D-history-file-authority: readable transcript files are authoritative; SQLite is a derived index (2026-09-11) {#d-history-file-authority}

**Status:** Settled intent. Retention and bounded previews shipped with PR #494
(merged 2026-09-18, v0.5.0).

**Partly superseded 2026-09-25** by [native-only history](native-only-history.md):
- **Content authority.** For a bound chat, the transcript is the harness's own native
  file. Meridian stops keeping its own runner copy: reads leave it in PR 2, and
  writes stop in PR 3.
- **Retained files.** Retained files and ZIPs now carry native snapshots of the bound
  key. Old runner-history members restore as bytes but are not read.
- **Still current:** SQLite is disposable and holds no unique fact.
- **Reversed:** the "SQLite FTS" rejection below, for a disposable projection.

**Decision:** Retained transcripts are ordinary, independently readable and
transferable record files. Those files and verified immutable ZIPs preserve the
transcript plus the identity and relationship facts needed to recover and re-index
it. Harness-owned locators can aid ingestion but cannot be the only retained copy.

SQLite is a disposable, one-way projection from files and available ZIPs. It may
hold discovery metadata (bounded preview checkpoints too, until
[D-browse-transcript-on-demand](session-browse.md#d-browse-transcript-on-demand)),
but no unique historical fact. Losing the database may cost acceleration, not history or control authority.
Writers publish dirty-source intent before mutation and do not need SQLite to finish.

**Why:** History must remain readable and transferable without a database export,
the original harness store, or a running Meridian service. One authoritative file
model makes repair a rebuild instead of bidirectional reconciliation between two
durable truths.

**Rejected:** SQLite-authoritative transcripts, locator-only retention, SQLite FTS
(reversed for search; see the status note), a resident indexer, full-conversation
caching, and a second transcript parser.
Database-file portability does not make the transcript independently readable, and
each additional interpretation or service becomes another correctness dependency.

### D-history-bounded-preview: previews are disposable projections of canonical history (2026-09-14)

**Superseded 2026-10-21** by
[D-browse-transcript-on-demand](session-browse.md#d-browse-transcript-on-demand):
browse no longer previews on highlight, and the preview cache, accumulator and
offline labels are deleted (meridian-cli PR #714). Kept for the reasoning; the one-normalizer rule survives as the shared
transcript reader.

**Partly superseded 2026-09-25** ([native-only history](native-only-history.md)):
- **Checkpoints.** Previews read only native sources, so the append-only checkpoint
  path for Meridian's runner streams is deleted.
- **Rebuild.** `session index rebuild` no longer warms previews: the warm loop
  re-folded and re-resolved every chat. Previews refresh lazily.
- **Still current:** the rest (same normalizer, bounded cache, offline labeling).

**Decision:** The browser caches a bounded recent-message projection produced by
the same canonical normalizer as full reads. It does not cache grouped render
entries as conversation authority. Selection is cache-first; refresh is selected-only
and latest-only. Neither metadata catch-up nor rebuild parses transcript bodies for
previews; a preview is built when its row is selected.

A cached offline preview is usable only for the same selected verified digest and
must be labeled offline. Selection, preview, and direct ZIP reads never restore or
launch history.

*Superseded 2026-09-25 (runner streams are no longer a preview source); kept for
the reasoning:* controlled append-only streams could resume from a complete-line
checkpoint. The checkpoint recorded both consumed extent and observed source size: the former is the
parsed boundary, while the latter includes any incomplete suffix seen at capture.
Witness changes are accepted only for real controlled growth. Replacement,
truncation, same-size rewrite, or other external editing requires rebuild; mutable
native sources use fresh snapshots.

**Why:** Responsive selection does not justify a second history model. Separating
consumed extent from observed size prevented an incomplete tail from being mistaken
for already-consumed content, while keeping incremental work for the writer shape
Meridian controlled.

**Rejected:** full-conversation or FTS storage, render-only clipping, a second parser,
full hashing on every append, and treating stat/tail witnesses as proof of arbitrary
external mutation safety.

### D-history-reclaim-gate: hash under shared capture, revalidate and retire briefly under exclusive ownership (2026-09-14)

**Decision:** Archive capture brackets a full hash with exact source witnesses under
the shared root gate and source lock. The final exclusive phase recomputes protection
and dependency closure, revalidates the witness, publishes reclaim intent, and
atomically retires the aggregate. It does not repeat the hash. Recursive cleanup runs
after root/spawn/scope locks are released.

Both retirement parents are synchronized before garbage collection disposes of the
staging entry or recovery acknowledges an absent source. A persistent sync failure
leaves the prepared receipt and retirement residue recoverable; it does not require a
second ledger. Witnesses detect post-hash changes but never substitute for checksum
verification.

**Why:** Exact byte verification is necessary, but keeping repeated hashing and
recursive deletion inside a global exclusive gate serializes unrelated writers.
Shared capture plus a short exact revalidation preserves safety while narrowing the
exclusive ownership interval.

**Rejected:** stat-only verification, a second full hash at reclaim, recursive
cleanup inside the writer gate, blind reversal after uncertain rename durability,
and an additional failure ledger.

A narrow decoder may drop the single obsolete capture-fingerprint field from
previously published immutable ZIP metadata. New records neither compute nor emit it;
unknown fields remain invalid, and archive member bytes still verify. This preserves
immutable history rather than maintaining a legacy runtime.

### Identity and restore boundaries

History UUID is portable transcript identity. Local spawn/chat aliases, physical
locations, and runtime ownership are separate concepts. A selected portable digest
chooses content; an offline copy is interchangeable only if it verifies to that same
digest. An older reachable snapshot is never an implicit fallback.

Restore preserves raw portable session facts, assigns fresh local aliases, and marks
the result historical and inert. Original process IDs, leases, scopes, and harness
continuation identifiers remain provenance, not executable ownership. This keeps
historical identity from becoming authority to control a current runtime.

The mechanism is described in
[Portable history](../architecture/state-system/portable-history.md).

**Provenance:** `work:next-minor-planning/design/followup-495-497.md`;
`work:next-minor-planning/DIVERGENCE/2026-09-14-preview-reclaim-model-followup.md`;
`work:next-minor-planning/followup-implementation-progress.md`;
`work:next-minor-planning/reviews/496-implementation-followup.md`;
`work:next-minor-planning/reviews/495-final.md`; commits `a8592210`, `80c37741`,
`c44a3e81`; `spawn:p6053`; `spawn:p6062`.

### D-history-index-initialization: initialization has its own bounded gate (2026-09-14) {#d-history-index-initialization}

**Status:** Implemented and independently verified on the feature branch at
`c3ffcaa0`; shipped when PR #494 merged (2026-09-18, v0.5.0).

**Budget changed in v0.18.0:** the automatic phase now gets 60 seconds, the
same as an explicit rebuild. At about 12,000 spawns a first build ran past
15 seconds, and the failure latch then failed every later command until a
manual rebuild ([spawn launches](../architecture/state-system/spawn-launches.md#the-history-index)).
The rest of this decision stands.

**Decision:** A first operation that needs a missing history index gets
one 15-second automatic initialization phase, separate from the ordinary two-second
query budget. Workspace/global initialization shares one absolute 15-second deadline
across roots. An all-warm operation never resets its ordinary deadline; only a real
initialization phase permits a fresh ordinary deadline afterward. Cache-only preview
peeks remain bounded independently and do not initialize.

Automatic and manual construction reuse the schema's catch-up lock
(`history-catchup-v<N>.lock` since
[D-history-index-schema-namespace](#d-history-index-schema-namespace)) and the same
lock-owned projection/publication body. They recheck compatibility and failure state
after taking ownership, so a waiter consumes a peer's publication instead of
rebuilding again. There is no daemon, second index, progress display, or per-attempt
initialization ledger.

A genuine owned-build failure is latched durably outside the replaceable SQLite
directory, scoped to the target schema and coordination generation. Later automatic
calls report the bounded reason and manual repair command rather than retrying.
`session index rebuild --metadata-only` is the explicit retry; successful metadata
publication clears the latch. Lock contention, another initializer, cancellation,
and crash before a recorded failure are not sticky failures. Status inspection must
not initialize the index or coordination state.

**A repair of the cause re-arms the latch (PR 3, 2026-09-26).** An `authority`
failure means the authoritative files could not be projected. A quarantined spawn row
records the row's own bounded quarantine message as the reason, so the error names
the `state.json` path (and, for a dogfood row, `meridian doctor`). Other invalid
metadata keeps the fixed reason "Invalid authoritative history metadata". The printed
retry command is `meridian session index rebuild --metadata-only`. When the dogfood
migration rewrites at least one row, it calls `HistoryIndex.clear_authority_failure()`.
That takes the catch-up lock, so any initializer still publishing a pre-repair failure
is ordered first, and it clears only `authority` markers. The next index-backed command
then initializes on its own. Rejected: fixing only the message, which still left a
manual rebuild; clearing in each caller, which duplicates the rule. Open (#530): one
quarantined non-dogfood row still fails the index closed for every command; whether
to project with warnings instead is undecided.

**Why:** A real 1,093-record / 6,077-session corpus repeatedly exceeded the ordinary
query budget while explicit metadata rebuild completed in 5.68 seconds. Two
concurrent missing-index callers also performed two serialized builds because the
second caller did not recheck under the existing lock. Globally raising query timeouts
would make warm contention slower without fixing ownership or retry behavior.

**Rejected:** silently incomplete results, automatic retry loops, progress UI,
sticky busy/cancellation state, per-root 15-second resets, and destructive transcript
conversion. The index remains disposable metadata, not history authority.

### D-history-index-readers-never-wait-on-session-writers: shorten the holder, don't widen the wait (2026-10-15) {#d-history-index-readers-never-wait-on-session-writers}

**Status:** Shipped in v0.10.4 (PR #588, merge `c0db78058`). Fixes #586 and
supersedes the B7 slice of PR #546
([D-b7-via-546](codebase-cleanup.md#d-b7-via-546-the-spawn-wait-contention-failure-is-fixed-by-pr-546)).
#546's other half, scoped no-arg wait discovery, shipped separately in v0.10.8
(PR #599), which closed #546.

**The symptom.** From 2026-10-12, agents hit this daily on 0.10.x:

```
$ meridian spawn wait
error: History index deadline exhausted
error: History index has unresolved sources: (...)
```

The first #588 design treated this as a budget problem: it classified contention
as `HistoryIndexBusy` and retried catch-up in roughly 2 s attempts inside a 15 s
overall budget. The user objected that even 2 s is too long for a metadata read. Measurement
then showed that the budget was never the issue. An uncontended
`HistoryIndex.spawns()` takes about 0.1 s, and catch-up takes 4–169 ms for 0–100
changed sources (`spawn:p7970`). So a 2 s timeout always meant a reader was queued
behind a lock holder. The holders were:

1. **Every session update, stop and model-selection append replayed the whole
   `sessions.jsonl` under `sessions.jsonl.flock`.** `_append_session_event` called
   `list_session_generations()` (a full Pydantic fold) only to refuse mutation of
   restored historical sessions (added in v0.5.0, `120b6d44b`). Hold time: 1,463 ms
   at 46.6 MB, 786 ms at 18.9 MB, 77 ms at 1 MB, 16 ms with the fold patched out. That
   is about four folding appends per spawn. With two writers, 5 of 30 readers failed,
   and none failed without the fold (`spawn:p7975`). 14 of the 15 logged incidents
   came from the runtime with the 46.6 MB log.
2. **Readers waited on that lock while holding the exclusive catch-up gate.** The
   `sessions` marker is one file per source and stays pending until acknowledged, so
   a reader draining it waits through every later writer's fold, and every other
   reader queues behind that reader. The slow fold ran *before* `mark()`, which is
   why the first investigation ruled it out; see the
   [lesson](../lessons/contention-diagnosis.md#check-a-cause-ranking-against-the-protocol).
3. **Each reader catch-up re-validated every stored session row** (about 8k rows) under
   that gate. `HistoryIndex._sessions()` loaded all `session_chats` and `sessions`
   rows on any append.

**Decision.** Fix the holders. A history-index read never waits on a session writer
and does work proportional to what changed. Contention that remains is reported, not
retried. The parts:

- **An O(1) historical-session guard via a reserved generation prefix.** Restore is
  the only place that mints a historical generation, and it now mints
  `historical-<uuid hex>` (`session_fold.new_historical_generation`). The append guard
  checks the event's own normalized generation and folds nothing. This is exact:
  replay raises "Historical session authority contains a mutation" only when an event's
  generation matches the historical record's, and that generation now carries the
  prefix. Restore already minted a fresh chat ID and a uuid4 generation, so only
  the string changed. The real logs held 0 `historical_import` events, so nothing
  was migrated.
  - *Rejected: a sidecar set of historical keys.* It is new derived state that must
    stay consistent with the log.
  - *Rejected: a byte-level prefilter scan.* It is still O(log size), and it
    string-matches JSON.
  - *Rejected: dropping the guard, or making replay ignore historical mutations.*
    The documented invariant says restored generations are immutable at the append
    boundary as well as in replay.
- **The prefix rules apply at append time only, never in replay.** An import must
  carry the prefix and a live start must not. Both checks run in
  `_append_session_rows`, under the append seam's `append_chat_rows` /
  `append_native_rows` since #651
  ([D-session-read-seam](state.md#d-session-read-seam)). Replay keeps only the "historical import must be inactive"
  check and the mutation check. The review (`spawn:p7987`, `old_restore_bricks.py`)
  showed what a replay-side prefix check would do: an unprefixed restore appended by
  an older build would fail every indexed read and every `rebuild` on that runtime
  permanently, because replay cannot skip a row. The check would add nothing, since
  the mutation check already keys on `record_mode` plus a matching generation. An
  older build's unprefixed restore bypasses the O(1) append guard but stays inert,
  because operations refuse to continue or fork historical sessions.
- **The sessions source is projected without the writer lock; acknowledgement
  happens only under it.** This revises the invariant in the repo's
  `lib/state/.context/history.md`. That invariant used to say that catch-up reads
  every authority under its source lock. Now, when `sessions.jsonl.flock` is busy, catch-up projects to the committed end
  of one open inode (`iter_jsonl_file` with a fixed extent) and leaves the marker
  pending. The tech lead's ruling was that the invariant exists to protect
  acknowledgement: never acknowledge a change you didn't see. That still holds. It is
  safe because:
  - the log is append-only and newline-committed;
  - a read that overlaps an in-flight append can be linearized before that append;
  - every completed append lies below EOF;
  - the cursor advances only over committed rows.

  The only rewriter, tail repair, is an atomic replace. A reader still holding the
  old inode sees a consistent pre-rewrite file, and the cursor's inode/tail check
  forces a full replay next time. The reviewer's `stress_l.py` tested this with 3
  writers (split batches, torn tails) racing 3 readers: the incrementally caught-up
  tables equalled a fresh `rebuild(reset=True)` in 3 of 3 runs. *Accepted cost:* one
  read can see a committed prefix of a multi-line `SessionBindings.commit()` batch.
  The index is a discovery projection, and lifecycle and control decisions read
  session authority directly. Spawn and catalog sources keep locked reads. Their
  writers hold the lock for 22–38 ms, and reading them unlocked could give a torn
  multi-file view.
- **Touched-chat catch-up.** Incremental `_sessions()` loads only the stored rows of
  the chats that new events touch: 2 rows decoded instead of 40 in the pinning test.
  Projection reads only the event's own chat, because fork lineage is copied from the
  start event.
- **One bounded attempt, not a retry budget.** `query()` and `catch_up()` make one
  catch-up within the caller's deadline (`QUERY_TIMEOUT`, 2 s, by default).
  Contention raises `HistoryIndexBusy`, a subclass of `HistoryIndexIncomplete`, so
  existing handlers keep working. Incompatible baselines, coordination errors and
  real I/O or SQLite errors still fail closed. `_source_lock` tells a held lock
  (never latched) from an exhausted budget. *Rejected: the 15 s `_retry_busy`
  budget.* It hid the holder and lengthened the catch-up queue for every other reader.
- **Every caller gets an owned, sticky cold build.** An absent index always gets
  one owned `INITIALIZATION_TIMEOUT` build, as
  [D-history-index-initialization](#d-history-index-initialization) requires, for
  every caller. A caller's deadline bounds only the catch-up after it. *Rejected
  (reversed in round 3): letting a caller's deadline bound the cold build, unlatched.*
  Descendant evidence polls every 0.25 s with a 2 s deadline. On an absent index it
  would start and abort a cold build on every poll, holding the catch-up gate up to
  2 s each time, where `main` did one owned build.
  *Follow-up (#656, 2026-10-16):* the build was protected, but the caller's
  absolute deadline still ran during it. Descendant evidence then answered
  `unknown` on a cold index. It now builds first and starts its 2 s afterward
  ([rule](../architecture/state-system/portable-history.md#a-caller-with-a-deadline-builds-the-index-before-starting-its-clock)).
- **No-arg `spawn wait` caps discovery at `QUERY_TIMEOUT`.** Discovery used to
  receive the whole wait deadline (3000 s for Claude and Codex). One catch-up could
  then hold the exclusive gate, plus the shared mutation and database locks, for up
  to 50 minutes while it waited on a marked spawn lock. If discovery is still busy,
  wait exits 0 with a checkpoint: `Discovery still pending (history index busy); run
  \`meridian spawn wait\` again.` It never reports "no pending spawns" without
  complete coverage. Answering from a stale projection stays rejected: it could miss
  a just-created spawn.

**Result** (copy of the 46.6 MB log, 240 spawn states, 30 reads): with no writers,
P95 39 ms. With 2 update writers, P95 101 ms and 0/30 failed; `main` had P95 2,001 ms
and 5/30 failed. With 2 spawn-start bind writers that each still hold the lock for
1.16 s, P95 43 ms and 0/30 busy.

**Still open:**
- [#418](https://github.com/haowjy/meridian-cli/issues/418) (absorbed #591): the
  remaining per-spawn full-log reads, some under `sessions.jsonl.flock`. Since #651
  (B1) they decode only the selected chats' rows but still scan every byte; see
  [session state](../architecture/state-system/session-state.md#sessionsjsonl-is-one-global-log)
  and the accepted per-chat design,
  [D-session-per-chat-journals](state.md#d-session-per-chat-journals).
- [#592](https://github.com/haowjy/meridian-cli/issues/592): a cold build holds the
  catch-up gate for 2.4–5.7 s.
- [#594](https://github.com/haowjy/meridian-cli/issues/594): every reader takes the
  exclusive catch-up gate even when nothing changed. With 16 pollers, 1 of 37 CLI
  reads is busy, the same as on `main`.
- [#527](https://github.com/haowjy/meridian-cli/issues/527): about 0.8 s of each CLI
  command is startup imports and model construction, not index work.

**Revisit if:** a second rewriter of `sessions.jsonl` appears that is not an atomic
replace (the lock-free read assumes there is none), or #418 B2 moves session
authority to per-chat journals (index v12 makes each chat its own source, and this
lock-free projection of `sessions.jsonl` goes away).

**Provenance:** `work:v010-followups` `decisions.md` D11–D16; `lock-holders-588.md`
(`spawn:p7970`); `session-fold-probe-588.md` (`spawn:p7975`); review `spawn:p7987`;
coders `spawn:p7978`, `spawn:p7986`, `spawn:p7988`, `spawn:p7989`; `chat:c7736`.

### D-abandoned-generation-closure: a later start of the same chat closes earlier generations; capture defers to real owners (2026-10-15) {#d-abandoned-generation-closure}

**Status:** decided by the tech lead after investigation (`work:resume-after-crash`
R4, R6, R8, R11). The user's ruling was the premise, not the mechanism: an
expected "native session is live" refusal is not an error ("why is this an
error at all anyways...?", R3). Shipped: ported onto `main` after #616 closed
unmerged (R15) and merged as PR #631 on 2026-10-16, merge `8ecd44989`. The
original `#616`-based branch `fix/abandoned-session-generation` (`933385b15`) was
deleted unmerged.

**Problem:** a Claude primary died with its pane when systemd stopped the scope
after an OOM kill. Its launcher wrote no exit and no session stop. The user ran
`--continue` at once, and the new launcher took the chat lock before stale-lock
cleanup could append the old stop. Capture treated the old unstopped generation
as a live writer, and no operation ever stopped it. A deterministic replay showed
the refusal was permanent on `main` and on #616 (which ran the second launch as
`p1.2` of the same spawn, and still left the old generation open). Separately,
the refusal surfaced as a traceback, and on `main` that traceback was drawn over
the resumed TUI ([D-tui-quiet-launcher](primary-terminal-ownership.md#d-tui-quiet-launcher--launcher-output-never-reaches-a-live-tui)).

**Decision:**
1. **Fix the state, in the fold.** A later accepted start of the same chat
   closes every earlier unstopped generation at its `started_at`. The premise is
   that every start writer holds the chat's exclusive lock; the reviewer
   (`spawn:p8156`) verified it for both production writers, and the port
   re-verified it on `main` (`spawn:p8255`). History index schema 7 → 10 on
   `main`; 8 and 9 are skipped because unreleased #616 builds wrote them.
2. **Liveness comes from real signals.** The guard drops "`stopped_at is None`"
   as proof of a live writer. Liveness now comes from a non-terminal row, a
   birth-validated lease, a serving scope or an observer.
3. **A live owner is a typed deferral.** `NativeCaptureDeferred` names the real
   owner. Startup repair logs it at debug level, stop maintenance stays silent,
   and archive lists it as deferred.
4. **Retry when the owner stops.** Stop maintenance captures the terminal records
   bound to the same native session.

Mechanism: [abandoned generations](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations),
[deferral and retry](../architecture/state-system/portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops).

**Rejected:**
- **Writing stop events at start time.** `start_session` would append explicit
  stops for earlier unstopped generations under the chat lock. That works, but
  finding those generations needs a full `sessions.jsonl` replay on the start
  path, against the constant-time bar in
  [per-operation cost](../architecture/per-operation-cost.md), and it would not
  heal journals already written. The fold rule heals
  existing journals on replay at no launch cost. It was the fallback if the
  start-writer audit failed.
- **Lease-only liveness without the state fix.** Replacing the `stopped_at`
  check with an exact-generation lease check alone would have made capture
  succeed, but the leaked open generation would stay in state. Capture would
  stop looking at it, and every other reader would still see a chat with two
  open generations. The
  fix repairs the state first and then trusts real liveness signals
  ([lesson](../lessons/state-design-lessons.md#repair-the-state-before-relaxing-the-check)).
- **Keeping the `stopped_at` fence as defence in depth.** Not needed once
  starts and leases are written together under the lock. It would keep a latent
  permanent block when a lease write fails after its start is appended.
- **A durable "capture deferred" marker drained by a later pass.** The owner's
  stop is already the exact retry trigger, and the guard's query already yields
  the retry set. A marker would add a state and a drain path without adding
  durability.
- **Having stop maintenance give the guard a second job** (raise and also return
  related records, behind a `collect_related` flag). The owner query is split
  from the guard instead; capture returns its native key, and stop maintenance
  queries once.

**Revisit if:** a `SessionStartEvent` writer appears that doesn't hold the chat
lock, or a native writer can outlive its launcher outside a fenced process scope.

**Provenance:** `work:resume-after-crash` (`decisions.md` R3, R4, R6, R8, R11,
R12, R15, R19, R20; `evidence/incident-b1.md`, `lane-b-capture-guard.md`, `lane-b-fix.md`,
`review-lane-b.md`, `lane-b-shape-pass.md`, `lane-b-port-main.md`); investigation `spawn:p8139`;
implementation `spawn:p8150`, `spawn:p8163`; review `spawn:p8156`; port
verification `spawn:p8255`; `chat:c7524`.

### D-history-index-schema-namespace: each index schema owns its own files; never migrate in place (2026-09-26) {#d-history-index-schema-namespace}

**Status:** Shipped in v0.7.0 (PR #534, merge `3dae2434`; implementation commit
`a3fd1439`).

**Decision:** `SCHEMA_VERSION` (in `state/history_changes.py`) names every file the
metadata index owns: `history-index/history-v<N>.sqlite3` and its `.build-v<N>`
stage, the `history-index/pending-v<N>/` marker queue and its `GENERATION`,
`locks/history-{catchup,database,markers}-v<N>.lock`, and
`history-index-init-failure-v<N>.json`. A new schema builds a fresh file from the
authoritative files. No build opens, upgrades or deletes another schema's files.
0.6.7 and earlier keep the unversioned `history.sqlite3`, `pending/`, locks and
latch. The authority locks stay shared (`locks/history-mutation.lock` and the
per-source locks), because both builds write the same `sessions.jsonl` and
`state.json` files. The mechanism is in
[portable history](../architecture/state-system/portable-history.md#index-files-are-named-by-schema).

**Why:** a real upgrade can leave 0.6.7 processes running: interactive sessions
survive the reinstall, while a background spawn may die when the reinstall replaces
its tool environment (the upgrade guide says to finish or cancel those first). The round-3 probe started a
0.6.7 `--bg` Pi spawn and then ran the PR build, which at the time upgraded the shared
`history.sqlite3` to schema 6 in place. The old runner emitted `turn_completed` and
then stayed `running` forever. Its stack sat idle in the asyncio loop, with the Pi
child still alive. Old commands reported "History index is incompatible".
Investigation p7222 isolated the cause: copying a schema-6 index alone into a fresh
runtime reproduced the hang, and the legacy import marker alone did not. 0.6.7's Pi
drain treats any index error as unknown evidence and waits while that persists.
SIGTERM finalized the run as `cancelled`, not `succeeded`, so the reaper could not
recover the result either. With per-schema files, a real overlap run (p7225) finished
`succeeded`, and 0.6.7's `spawn list`, `session log` and `session index status` kept
working. The old file stayed at schema 2, byte-identical.

**Rejected:**
- **In-place migration.** It rewrites a file that a running older build still reads.
  All migration code (`SchemaStep`, `SCHEMA_STEPS`, the `outdated` status) was deleted:
  with only one schema ever writing its own file, a file whose version does not
  match its name is a stray copy. It reports `incompatible` and needs an explicit
  rebuild.
- **Lock coordination with the older build.** 0.6.7 cannot follow a protocol for a
  schema it has never seen.
- **Sharing the marker queue, catch-up locks or failure latch.** Each build would
  clear or reset the other's markers. A schema-6 failure would block 0.6.7, and an
  old failure record would block v6.

**Cost and gap.** Each build now marks only its own queue, so v6 stops seeing an
older build's writes. Catch-up closes most of the gap by re-reading active loose
spawns and the `sessions.jsonl` cursor without a marker (`_reread_unmarked`),
kept cheap by a partial `active_records` index, a skip for unchanged spawn rows and
an early return when the journal has not grown. Spawns that an older build *creates*
after v6 was built, and archive changes it makes, still need `session index rebuild`.
First-use build: 1.5 s on a 171-spawn runtime and 5.2 s on a 1,990-spawn one
(budget 15 s); warm reads about 0.9 s.

**Rollback.** Per-schema files keep an already-running older build working; they do
not make downgrade work. Once a newer build has run a spawn in a project, 0.6.7 cannot
parse its rows (`run_boundary`, `native_store`): `spawn list` fails for the whole
project and an older `session index rebuild` stops at the first new row. Downgrade is
unsupported (user-facing: `docs/upgrading.md`, "Rolling back to 0.6.7"). Deleting
`history-index/history.sqlite3*` only repairs an index that a pre-release build of
this branch (before `a3fd1439`) converted in place.

**Revisit if:** a projection ever holds a fact the authoritative files lack. Then a
fresh build could lose data, and migration would need to come back.

**Provenance:** `work:native-harness-session-identity`, `decision.md` entries of
2026-09-26 "Round-3 probe of every command" (stuck old runner, p7222; fix lane
p7224); `evidence/probe3-upgrade-rerun.md`; investigation `spawn:p7222`; fix and
overlap test `spawn:p7225`, commit `a3fd1439`.

### D-history-portable-digest-stored-json: verify archive digests over the stored record JSON (2026-09-26) {#d-history-portable-digest-stored-json}

**Status:** Shipped in v0.7.0 (PR #534, merge `3dae2434`; implementation commit
`3e041a23`).

**Decision:** The portable record digest is SHA-256 over canonical JSON of the record
object *as stored* in the manifest or `record.json`, minus local-only fields
(`state/retention_digest.py`, `stored_record_digest`). Verification never re-parses
the record through today's Pydantic models and re-serializes it before hashing.
Writers hash exactly the JSON they publish, so every digest already written still
matches. Restore stages the archived `record.json` bytes unchanged. Where a check asks
"are these the same facts?" (re-capturing a restored record, `_verify_existing`), both
sides go through current models (`model_record_digest`), so neither side carries an
older shape.

**Why:** the old verifier loaded the manifest into current models and hashed that
projection. Adding `run_boundary` to `SpawnRecord` (commit `91ccb3c0`) and
`native_store` to `SessionRecord` put new `null` fields into the hashed object, so
every 0.6.7 archive failed `session import` and `session restore` with "Portable
record digest mismatch", including the user's real 19-record archive. The digest had
been checking the reader's schema, not the archive's bytes. Member bytes were always
separately checked against the manifest's size and sha256; that check was never the
problem.

**What 0.6.7 archives give back.** Import and restore now succeed, and a repeat
import reports `Already imported`. The records stay inert: a 0.6.7 ZIP holds runner
`history.jsonl` and no `native-transcript.jsonl`, and runner history is not read
([option C](native-only-history.md#old-runner-history-option-c-drop)), so `session
log` and `export` of an imported or restored 0.6.7 record refuse with "is historical
and has no retained native snapshot". Chats still bound to a live native file read
normally through their binding. The member stays in the ZIP for manual extraction.

**Rejected:**
- **Per-field exclusions for each new model field.** Every schema addition would
  break old archives again until someone added another exclusion.
- **Dropping the portable digest** in favor of member hashes alone. Member hashes do
  not cover the record facts that restore publishes.

**Open follow-up:** `restored-from.json` `session_sha256` still hashes the folded
`SessionRecord`, so a repeat restore into a runtime that 0.6.7 restored into fails
([#537](https://github.com/haowjy/meridian-cli/issues/537)).

**Revisit if:** the stored record format itself changes shape; then the digest needs
an explicit version, not a model-dependent recipe.

**Provenance:** `work:native-harness-session-identity`; diagnosis
`evidence/investigate-067-archive-digest.md`; fix and real-archive checks
`spawn:p7236` (commit `3e041a23`).

### D-native-transcript-snapshot: preserve stream evidence and publish a separate canonical snapshot (2026-09-14) {#d-native-transcript-snapshot}

**Status:** Superseded as a future write-path decision by
[native-session-identity](native-session-identity.md) and
[native-only history](native-only-history.md): runner history is neither read (PR 2)
nor written (PR 3), and old runner-history files are not decoded. The stream-and-snapshot
model below records the earlier choice. It is kept because it explains why the
stream was once retained. Native snapshots survive in the replacement design as
archive members of the bound key. Legacy runner-history members stay inert bytes.

**Superseded decision:** `history.jsonl` keeps its existing stream/retry-evidence meaning.
Qualified native-primary history is published atomically as a separate snapshot
inside the same record aggregate and becomes canonical through one shared source
selection boundary. This is preferred to append-only begin/body/commit sections,
which contaminate runner evidence and require abandoned-section recovery. File
existence or nonempty output does not prove capture completeness. Exact association
comes from the completed record, while the descriptor states the native observation
interval/scope rather than pretending to prove a past stop-time cutoff.

Capture preserves provider-native authority rather than normalized display events.
OpenCode now supplies a harness-owned raw-row representation containing the exact
session plus ordered message/part rows, identities, relationships, unsupported
material, and original payloads from one read-only transaction. Shared normalization
interprets those records for display and report consumers. This raw input is not yet
a qualified capture: storage completeness and rendering support remain separate
outcomes.

New archives explicitly declare the canonical transcript member and bind a versioned,
immutable original capture descriptor into portable identity. Restore may assign
inert local aliases, but rearchive carries the original descriptor unchanged so
archive -> restore -> rearchive -> restore still validates without the original
native store or first ZIP. Old archives retain their original implicit
`history.jsonl` recipe and are never rewritten.

Canonical snapshot validation is incremental and deadline-aware. A prefix whose
seal/count/digest has not been fully checked is partial, never verified-empty or
complete; search, export, preview, and retention consume the same storage/rendering
outcome rather than inventing surface-specific fallbacks. Retention requires full
validation before reclaim.

**Scope correction:** The earlier exact-stop frontier requirement is superseded,
not proved. The user asked to repair existing post-stop capture; its selected scope
is a complete consistent native observation associated with the exact record. A
real OpenCode counterexample and independent contract audit justify rejecting a
new universal frontier/ownership protocol. Preserve known-incomplete/unavailable
failures and exact reclaim checks. One provider/materializer qualification result
distinguishes complete observation, known incomplete, unavailable and unsupported.
Existing lifecycle facts and supported native markers reject known unfinished input;
consistent bytes alone do not qualify it. Delayed preparation retries the same checks
and may qualify resolved output without discarding earlier interruption evidence;
valid captures remain immutable. Integrated implementation/runtime evidence is still
required.

Capture-purpose selection already binds post-stop maintenance to the exact completed
primary spawn rather than the latest chat generation. State, primary sidecar, and
exact-generation session facts feed one normalized candidate helper. Selection
requires singleton agreement; the owner guard conservatively refuses a possible
same-harness match when recorded facts conflict. A distinct harness may reuse the
same opaque native ID without aliasing. Exact-generation live leases and unreleased
scopes block reading even when the native ID exists only in spawn metadata, and a
second check runs before publication. These same-runtime observations are safety
preconditions, not an external-writer or cross-machine fence. Children prepare only
existing retained streams and never fall back to native-primary storage.

**Why:** Investigation separated two causes of issue #499. The repaired Pi grammar
had dispatched only RPC `message_end`, so native on-disk `type=message` records
normalized to zero. Independently, PR #494 introduced existence-only capture guards
that can certify a partial stream while fuller native history exists. The identity
and source-selection preconditions are now repaired, but the old ingest guard and
envelope writer still require the qualified snapshot implementation.

**Rejected:** overwriting the stream, append-section transactions, normalized-text
capture, per-surface parsers, fallback based on message count, and treating ZIP byte
integrity as proof that the selected transcript was complete.

**Provenance:** `work:next-minor-planning/investigation-499-500.md`;
`work:next-minor-planning/design/history-repair-plan.md`;
`work:next-minor-planning/design/index-initialization.md`;
`work:next-minor-planning/design/native-transcript-capture.md`;
`work:next-minor-planning/reviews/repair-design-review.md`;
`work:next-minor-planning/DIVERGENCE/post-stop-capture-scope.md`;
`work:next-minor-planning/reviews/post-stop-scope-review.md`; `spawn:p6109`; `spawn:p6110`.

## Related

- [State-layer decisions](state.md) — runtime state, durability, locking, and typed contracts
- [Per-operation cost](../architecture/per-operation-cost.md) — measured cost bar
  and acceptance evidence
- [Contention and cost diagnosis](../lessons/contention-diagnosis.md) — diagnosis
  method and measurement traps
- [Spawn wait barrier](../concepts/spawn-wait-barrier.md) — the discovery path
  whose contention motivated this decision
- [Portable history](../architecture/state-system/portable-history.md) — current history-storage mechanism
- [Session state](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations) — the fold rule that closes abandoned generations
- [State-design lesson](../lessons/state-design-lessons.md#repair-the-state-before-relaxing-the-check) — the state-first repair lesson
- [Crash troubleshooting](../operations/troubleshooting.md#primary-crashed-with-no-exit-or-stop-recorded) — the incident behind the decision
- [State-system overview](../architecture/state-system/overview.md) — state-system map
- [Native transcript reads](../architecture/native-transcript-reads.md) — live and retained transcript read paths
- [Native session identity lessons](../lessons/native-session-identity.md) — upgrade and cross-harness seam failures
