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
remains untested. Do not claim four-harness runtime success. PR #494 remains draft.

Provenance: `work:next-minor-planning/design/native-capture-boundary-decision.md`;
`reviews/frontier-contract-audit.md`; `probes/frontier-opencode-claude.md`;
`probes/frontier-codex-pi.md`; `probes/native-content-scope-followup.md`.


## Intended History Storage

### D-history-file-authority: readable transcript files are authoritative; SQLite is a derived index (2026-09-11) {#d-history-file-authority}

**Status:** Settled intent. Retention and bounded-preview implementations are
approved on the feature branch but are not merged or released.

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

**Rejected:** SQLite-authoritative transcripts, locator-only retention, SQLite FTS,
a resident indexer, full-conversation caching, and a second transcript parser.
Database-file portability does not make the transcript independently readable, and
each additional interpretation or service becomes another correctness dependency.

### D-history-bounded-preview: previews are disposable projections of canonical history (2026-09-14)

**Decision:** The browser caches a bounded recent-message projection produced by
the same canonical normalizer as full reads. It does not cache grouped render
entries as conversation authority. Selection is cache-first; refresh is selected-only
and latest-only. Ordinary metadata catch-up does not parse transcript bodies, while
explicit rebuild can warm previews through the same projector.

A cached offline preview is usable only for the same selected verified digest and
must be labeled offline. Selection, preview, and direct ZIP reads never restore or
launch history.

Controlled append-only streams may resume from a complete-line checkpoint. The
checkpoint records both consumed extent and observed source size: the former is the
parsed boundary, while the latter includes any incomplete suffix seen at capture.
Witness changes are accepted only for real controlled growth. Replacement,
truncation, same-size rewrite, or other external editing requires rebuild; mutable
native sources use fresh snapshots.

**Why:** Responsive selection does not justify a second history model. Separating
consumed extent from observed size prevents an incomplete tail from being mistaken
for already-consumed content, while retaining incremental work for the writer shape
Meridian controls.

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
`c3ffcaa0`; unreleased. PR #494 remains draft.

**Decision:** A first operation that needs a missing or outdated history index gets
one 15-second automatic initialization phase, separate from the ordinary two-second
query budget. Workspace/global initialization shares one absolute 15-second deadline
across roots. An all-warm operation never resets its ordinary deadline; only a real
initialization phase permits a fresh ordinary deadline afterward. Cache-only preview
peeks remain bounded independently and do not initialize.

Automatic and manual construction reuse `history-catchup.lock` and the same
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

**Why:** A real 1,093-record / 6,077-session corpus repeatedly exceeded the ordinary
query budget while explicit metadata rebuild completed in 5.68 seconds. Two
concurrent missing-index callers also performed two serialized builds because the
second caller did not recheck under the existing lock. Globally raising query timeouts
would make warm contention slower without fixing ownership or retry behavior.

**Rejected:** silently incomplete results, automatic retry loops, progress UI,
sticky busy/cancellation state, per-root 15-second resets, and destructive transcript
conversion. The index remains disposable metadata, not history authority.

### D-native-transcript-snapshot: preserve stream evidence and publish a separate canonical snapshot (2026-09-14) {#d-native-transcript-snapshot}

**Status:** Partially implemented on the feature branch through `e2fea094`; unreleased.
Pi grammar and preview compatibility, exact completed-primary handoff, archived-child
warming, OpenCode raw-row preservation/shared interpretation, and exact native-identity
selection with same-runtime known-owner checks are implemented and independently
reviewed. Qualified snapshot publication, canonical validation, portability, model
observation, four-harness workflows, and PR readiness remain open. PR #494 remains
draft.

**Decision:** `history.jsonl` keeps its existing stream/retry-evidence meaning.
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
