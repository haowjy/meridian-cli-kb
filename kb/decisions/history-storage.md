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
hold discovery metadata and bounded preview checkpoints, but no unique historical
fact. Losing the database may cost acceleration, not history or control authority.
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

### D-history-index-schema-namespace: each index schema owns its own files; never migrate in place (2026-09-26) {#d-history-index-schema-namespace}

**Status:** Implemented in combined PR #534 (`a3fd1439`, merged into
`feat/native-session-identity` @ `e194ceea`); unreleased.

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

**Why:** a real upgrade leaves 0.6.7 processes running. The round-3 probe started a
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

**Rollback.** The older build's index misses everything a newer build recorded, and
unreleased builds of this branch before `a3fd1439` had already converted
`history.sqlite3` to schema 6. After rolling back, run the older build's `meridian
session index rebuild --metadata-only`, or stop its processes and delete
`history-index/history.sqlite3*`. The older files can be deleted once no older
build uses the runtime.

**Revisit if:** a projection ever holds a fact the authoritative files lack. Then a
fresh build could lose data, and migration would need to come back.

**Provenance:** `work:native-harness-session-identity`, `decision.md` entries of
2026-09-26 "Round-3 probe of every command" (stuck old runner, p7222; fix lane
p7224); `evidence/probe3-upgrade-rerun.md`; investigation `spawn:p7222`; fix and
overlap test `spawn:p7225`, commit `a3fd1439`.

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
- [Portable history](../architecture/state-system/portable-history.md) — current history-storage mechanism
- [State-system overview](../architecture/state-system/overview.md) — state-system map
- [Native transcript reads](../architecture/native-transcript-reads.md) — live and retained transcript read paths
- [Native session identity lessons](../lessons/native-session-identity.md) — upgrade and cross-harness seam failures
