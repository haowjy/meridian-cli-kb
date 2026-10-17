# lessons/state-design-lessons — Why the State System Is Built This Way

These are the choices that shaped Meridian's state layer, with the reasoning that drove each one. The "we tried X" context is what makes these valuable — without it, the decisions look arbitrary.

This page is historical: it records the reasoning and mistakes that shaped the
state system. For the shipped layout and identity mechanism, use
[Architecture: State System](../architecture/state-system/overview.md). Some early
choices below were later superseded; each section names what remains true.

## Why Separate Committed Identity from User Runtime

The durable lesson is separation by volatility: project identity must move with
the repository, while spawn records, transcripts, locks, and caches must not be
committed. The first implementation put a UUID in `.meridian/id` and keyed
`~/.meridian/projects/<uuid>/` from it. That solved path moves but made checkout
identity gitignored and left repo-local `.meridian/` carrying ambiguous roles.

The current design keeps the split but changed its binding: `[project].id` in
`meridian.toml` is authoritative for a committed project and the user runtime
remains `~/.meridian/projects/<id>/`. The legacy `.meridian/id` reader and its
`meridian migrate` command were deleted in 2026-10. Work items moved to the
configured context work root rather than either identity or runtime location.

The July 2026 rule that the first write creates identity added its own lesson:
**price the files a lazy default leaves behind.** Minting `meridian.toml` in
any directory that received a write kept every directory runnable, but a census
found 326 stray markers under `/tmp` and 158 runtime roots nothing could
attribute. The fix kept "runs anywhere" and moved scratch state out of the
directory: marker-less directories get a path-derived id, `origin.json`, and a
retention rule ([D-two-tier-identity](../decisions/state.md#d-two-tier-identity)).

**What remains true:** committed identity is not the absolute checkout path
(only unshared, ad-hoc state is path-keyed); high-churn runtime state stays
user-local; migration must land with the new write path rather than leaving two
active stores indefinitely.

## Why JSONL Over SQLite

**The temptation:** SQLite would give us transactions, queries, indexes, and a schema. Running `SELECT * FROM spawns WHERE status='running'` is cleaner than replaying a JSONL stream.

**Why we didn't:**

1. **Crash tolerance.** JSONL appends are atomic at the line level — a crash mid-write leaves an incomplete line that gets skipped on read. SQLite transactions require a write-ahead log that can get corrupted under hard kills. Crash-only design favors the dumbest possible write that can be safely recovered.

2. **Inspectability.** JSONL and per-spawn JSON files work with standard tools (`cat`, `jq`) without any Meridian tooling. This matters for debugging in CI, on bare machines, in restricted environments. SQLite requires `sqlite3` and schema knowledge.

3. **No service dependency.** SQLite files are exclusive-locked when open on some platforms. Multiple concurrent Meridian processes (nested spawns, parallel CLI invocations) need to read state concurrently. JSONL with `flock` sidecar scales to N readers + 1 writer without a connection manager.

4. **Simpler crash recovery.** A truncated JSONL file is recoverable by skipping the last partial line. A corrupt SQLite database requires `PRAGMA integrity_check` and may need manual reconstruction.

**The cost:** Querying requires reading and replaying the entire file. For spawn state this cost became real, so the global event log was superseded by per-spawn `state.json`. JSONL remains appropriate for sessions and journals where ordered history is the contract.

---

## Why Mutable JSON for Work Items

**The asymmetry:** Everything else is append-only JSONL. Work items are mutable per-directory JSON (`<context.work>/<slug>/__status.json`). Why the inconsistency?

**The constraint:** Work items have slugs, and the slug IS the directory name. When a work item is renamed, the directory must also rename. Directory location is the primary authority for active-vs-archived state.

With JSONL, rename would mean appending a `rename` event and somehow rebuilding the path — but the directory and the slug must stay in sync, and you can't atomically rename both via event append.

**The solution:** Mutable JSON (`os.replace()` for atomic overwrites) with a rename intent file. Before any rename:
1. Write `work-items.rename.intent.json` (the intent)
2. Rename the directory
3. Rename the JSON file
4. Delete the intent

If the process dies between steps 2 and 3, the intent file survives and the rename is replayed on next startup/reconciliation. This is crash-safe, inspectable, and keeps slug + directory in sync.

---

## The Projection Authority Rule

**The problem:** The reaper runs on every read path. It may stamp an orphaned spawn as `failed` with `orphan_run` before the actual runner process has a chance to report success. If the runner then reports `succeeded`, which one wins?

**The naive answer is wrong:** "Last write wins" means the reaper's `orphan_run` can never be corrected. "First write wins" means a legitimate runner success can be overridden by a reaper that jumped the gun.

**The actual answer:** Origins have authority levels. `runner` and `launcher` are authoritative (they were executing the harness). `reconciler` (the reaper) is non-authoritative (it's inferring from liveness signals).

- Authoritative vs. authoritative: first writer wins (no revision)
- Authoritative overrides reconciler: always — runner ground truth beats reaper inference
- Reconciler vs. reconciler: first writer wins (concurrent reap calls don't revise each other)

**How it manifests:** `finalize_spawn()` accepts an `origin` parameter. Projection replays all finalize events and applies the authority rules to determine the canonical terminal state.

**The lesson:** When multiple processes can write terminal state for the same record, you need an explicit authority ordering — not just a timestamp or a mutex. Timestamps require clock synchronization. Mutexes block. Authority rules are append-safe and correct without coordination.

---

## Reaper Lessons

These explain the reaper on `main`. Each guard below patched a guess, and the
windows the guesses left are why [identity-v2](../decisions/spawn-identity.md#d-spawn-identity-v2)
replaces the reaper's liveness with an exact lock. PR 3 (built, awaiting merge)
deletes every guard below; the list of what went and why it must not come back
is in [derived status](../architecture/state-system/derived-status.md#deleted-mechanisms-do-not-bring-back).
See [exact liveness beats guessed liveness](#exact-liveness-beats-guessed-liveness).

### Heartbeat-Based Liveness Over PID-Only

**Early design:** The reaper checked if the runner PID was alive. If dead, the spawn was orphaned.

**The problem:** PID reuse. A process finishes, its PID is recycled to a completely different process, and the reaper sees an "alive" PID and leaves the spawn in `running` forever.

**The fix:** Two guards:
1. Create-time comparison: if the process was created more than 30 seconds after the spawn's `started_at`, it's a PID reuse
2. Heartbeat recency: if any artifact in the spawn directory was touched within 120 seconds, the runner is considered active regardless of PID state

**The lesson:** PID liveness is a necessary but not sufficient condition. The heartbeat artifact provides a ground-truth liveness signal that doesn't depend on the OS PID table.

### Why Artifact-Agnostic Recency

**Original design:** The reaper checked only the `heartbeat` file for recency.

**The problem:** A runner that crashes while writing `stderr.log` or `history.jsonl` — just before writing the heartbeat — would be reaped immediately on the 120-second window, even though it was active a moment ago.

**The fix:** The recency check applies to a set of activity artifacts, and a write to any of them resets the recency window. The set is now `{heartbeat, bash-records.json, stderr.log, report.md}`. PR 2 dropped `history.jsonl` from it, because the runner stream is no longer read for anything and PR 3 stops writing it. Managed primary attach now touches the heartbeat as well, so no attached run relied on the stream for liveness ([native-only history](../decisions/native-only-history.md)).

**The lesson:** Use the broadest safe set of liveness signals. The heartbeat is the primary signal (written every 30s unconditionally), but last-gasp activity on other artifacts should prevent premature reaping.

### Why PID Probes Are Skipped for `finalizing`

**The `finalizing` status** means the harness process has already exited — the runner is in controlled post-exit drain (report extraction, finalization). The runner PID is still alive (it's the runner, not the harness), but the worker (harness) is done.

**The problem:** PID-probing the runner during `finalizing` gives "alive" (runner hasn't exited yet), which would suppress reaping forever for a runner that crashed mid-drain.

**The fix:** For `finalizing` rows, the reaper uses only heartbeat recency, not PID liveness. A stale heartbeat during `finalizing` means the runner crashed mid-drain → `orphan_finalization`.

## Spawn Identity and Status Ownership

*From `work:v010-followups` D45–D53 (2026-10-15/16): #616's run model, the
owner's questions that stopped it, and the lock probe.*

### A name with two meanings generates bugs

#616 kept `--continue pN` on the same spawn by making each launch a row: `pN`
for run 1, `pN.2` and later for the rest. A bare `pN` therefore meant both
"spawn pN" and "run 1 of pN". Every surface that forgot which one it held read
the wrong row, and it did so plausibly, with no error. Eight review passes over
217 files kept finding the same class:
- casts from spawn to run 1 at four sites;
- the Pi drain watching `pi-bash/<spawn>` while TypeScript wrote `pi-bash/<run>`
  (HIGH);
- children of a reopened run counted under the spawn's old parent, which needed
  schema v9's `parent_history` to fix;
- TypeScript/Python parser drift on bare `pN`;
- wait JSON putting a run ID in `spawn_id`.

Four follow-up issues (#617–#620) existed only because of the second identity.
The guard rails grew with them: a `SpawnId`/`RunId` type split, a CI grep gate,
head files with hole probing, a representative-run rule with six variants.

The format came from an earlier design rule, "no data migration", which the
project's no-backward-compatibility rule never asked for. **When a design needs
type splits and grep gates to keep one name from meaning two things, check
whether the second meaning is needed at all.** Here it wasn't
([D-spawn-identity-v2](../decisions/spawn-identity.md#d-spawn-identity-v2)).

### Several status writers make late writes; derive status instead

The second identity existed to fence late writes: an old launch writing its
result onto a newer one. Late writes existed because status was a mutable
field with three writers (the runner's finalize, the reaper and cancel), and
because no writer knew exactly whether the runner was alive. Each extra writer
needed an arbitration rule (the authority lattice, the `mark_finalizing` CAS,
cleanup-claim files), and each liveness guess opened a window for the rules to
decide wrongly.

Moving the boundary removes the class instead of fencing it:
- only the process that holds the launch's lock writes its outcome, once;
- cancel records a request and signals, and the reaper records crashes only
  while holding the lock;
- everyone else derives status from the outcome and the lock.

A dead runner writes nothing and a live one blocks the next launch, so no late
write exists. The owner's question that found it: "why do we record whether a
launch is running, finished, or failed?"

### Exact liveness beats guessed liveness

The reaper guesses: a PID whose create time is within 30 s of the spawn's start,
plus any activity file touched within 120 s, plus a 15 s startup grace and a 5 s
post-exit grace. Each guard in the reaper lessons above patched one false
answer, and each left a window where a live spawn reads dead or a dead one reads
live. A process-held `flock` has no window: the kernel releases it when the
holder dies, however it dies.

*Who* holds the lock decides whether it tells the truth. The lock probe
(`spawn:p8181`, `spawn:p8182`) tried two placements on real harnesses:
- **Harness-inherited (M1): unreliable.** Whether it is right depends on which
  processes each harness passes fds to. Codex's Node wrapper kept the fd and its
  native re-exec dropped it, which reads as a false crash. A generic descendant
  kept it after the launcher died, which reads as false "running".
- **Runner-only (M2): exact** for the runner, because Meridian controls that
  process's fds.

M2 answers "is the runner alive", not "is everything this launch started gone".
That second question is containment, solved separately
([process containment](process-containment.md)).

### A deleted rule survives in other writers and in fixtures

Main's reaper turned a durable `report.md` into `succeeded`. Identity-v2 drops
that rule: a report counts only through an outcome the lock holder writes. PR 3
deleted it from the reaper and from `derive`, and two more copies surfaced later.

- **A second writer.** Cancel convergence (`_force_cancel_convergence`) applied
  the same rule through `resolve_completion_cancel_precedence`. With the reaper
  now deriving `cancelled` for a dead row with a cancel intent, the two writers
  disagreed, and whichever wrote first won. On main they had agreed only because
  both carried the rule. The fix routed both through one claim
  ([D-spawn-identity-v2, PR 3](../decisions/spawn-identity.md#pr-3-as-built-read_derived-and-one-dead-launch-finalize)).
- **Test fixtures.** Two Pi drain tests staged a lockless `finalizing` child with
  a `report.md` and relied on the reaper overlay reading it as completed to reach
  the publication-pending path. They passed on base and failed only in the
  extended tier of the full preflight. The product was right; the fixtures
  encoded the deleted rule. They now stage a live child as a lock holder and an
  undelivered result as a finished, unconsumed child.

**When deleting a rule, search for every place that applies it, not just the
module that named it:** other writers of the same field, and fixtures whose
setup only reaches the state under test through the rule. A rule that two
writers share looks like agreement until one of them changes.

### "Every reader" sweeps in selectors

PR 3's brief was "every reader moves to `read_derived`". Applied mechanically, it
also moved code that *selects rows for a writer*: descendant cancellation,
archive capture and doctor's warning. Derived status hides a dead launch as
`failed` until someone persists it, so those selectors skipped exactly the rows
they existed to handle. Three reviews found three instances (D80, D82).

The working rule: **selection for convergence or destruction uses stored status;
liveness questions use derived status**
([derived status](../architecture/state-system/derived-status.md#selection-versus-liveness)).
Before migrating a "reader", ask whether its answer feeds a write or a delete.

## Repair the State Before Relaxing the Check

*From `work:resume-after-crash` (2026-10-15). The fix shipped in PR #631
(`8ecd44989`, 2026-10-16).*

**The situation:** native capture refused forever with "Cannot capture an active
native owner: p12669". p12669 was long dead. Its launcher had been killed before
writing a session stop, and the `--continue` that followed took the chat lock
before cleanup could write it. The guard read "this generation has no stop" as
"someone is still writing".

**The tempting fix:** make the guard ask a better question. Check the exact
generation's lease instead of `stopped_at`, and the refusal goes away. The
investigator (`spawn:p8139`) argued against doing only that, and the lead agreed.
A lease-only check makes capture work while the state stays wrong: the chat keeps
an open generation that nothing will ever close. Capture has stopped looking at
that field, but the rest of the system hasn't. Every other reader still sees a
chat with two live generations, and the next check written against `stopped_at`
inherits the same permanent refusal.

**What shipped:** two changes, in this order.
1. **Fix the state.** The generation fold closes an earlier unstopped generation
   when a later start of the same chat is accepted. That inference is safe only
   because every start writer holds the chat lock, so an audit of the writers
   came first.
2. **Then trust real liveness signals.** Once the state no longer leaks, the guard
   can drop `stopped_at` and rely on a non-terminal row, a birth-validated lease,
   a serving scope or an observer.

Doing (2) alone would have hidden the leak. Doing (1) alone would have left a
guard that still refuses whenever a stop is lost some other way, such as a failed
lease write.

**The lesson:** when a safety check refuses because the state says something
false, fix the state first. A check relaxed to step around bad state makes the
symptom disappear and leaves the bad state for the next reader to find. Mechanism:
[abandoned generations](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations);
[decision](../decisions/history-storage.md#d-abandoned-generation-closure).

## Typed State Contracts: Systemic Patterns (PR #423)

Bug patterns that recurred across multiple lanes of the typed-state-contracts
work, worth encoding as durable lessons.

### Operate-Before-Type-Check in `mode="before"` Validators

Pydantic `mode="before"` validators receive raw deserialized input, not typed
model fields. Calling `.strip()` or testing membership (`value in frozenset`)
before checking `isinstance(value, str)` raises `AttributeError` or `TypeError`
on non-string input, bypassing the quarantine seam and crashing the reader.

This pattern bit two separate lanes (#400 and #403) in the same way: a
normalizer called `.strip()` on a value that could be `None`, `int`, or another
non-string type from corrupt persisted JSON. The fix in both cases was to
type-check first, then operate:

```python
if not isinstance(value, str):
    raise ValueError(...)   # routes to quarantine
normalized = value.strip()  # safe — value is str
```

The Gate 2 convergence sweep grepped all eleven `mode="before"` validators
and confirmed the pattern was fully closed.

### Bounded Shape Guards Grow Into Hand Codecs

A pattern observed twice in PR #423: a handwritten runtime shape guard
("just check a few fields") accumulates edge cases until it becomes an
informal codec. The Pi TypeScript `SpawnStateFile` type started as a simple
interface with a runtime `typeof` guard, then grew to validate nested
optional objects, discriminate terminal state, and handle quarantined rows.
Python-side, the `quarantine_unknown_vocabulary` validator started as a
three-field enum-member check and grew to validate nested fact vocabularies,
type-check before string operations, and handle extra-field rejection.

In both cases, the fix was to replace the hand guard with a declarative
schema tool (TypeBox for TypeScript, `extra="forbid"` on Pydantic for
Python). The declarative schema cannot drift from the model it describes.

**Lesson:** When a shape guard crosses two fields, evaluate whether a
schema derivation tool should own the validation. The guard's growth is
predictable — each new field or nested model adds another branch — and the
maintenance cost compounds as the schema evolves across work streams.

### Cross-Language Consumers of Persisted Schema

The "no backwards compatibility" rule is Python-scoped. The Pi
`meridian-spawn-watch` TypeScript extension reads `state.json` directly and
broke when lifecycle facts moved from flat fields to nested sub-models: the
extension rendered `duration` as `-` and misreported quarantined rows as
`succeeded`.

The fix classified completion from the `terminal` discriminant
(`terminal !== null`) and deleted the extension's duplicated status vocabulary.
Wave 4 refined this further: a partial TypeBox schema replaced the handwritten
type and runtime guard stack, deriving `SpawnStateFile` from
`Static<typeof SpawnStateFileSchema>`. Terminal status remains `Type.String()`;
Pi does not duplicate Meridian's terminal vocabulary. A deeper TypeScript
runtime codec was judged unnecessary — the partial schema covers only the
fields the extension consumes.

**Lesson:** When persisted state has cross-language readers, schema changes
require a sweep of those readers. The authority boundary is the discriminant
(`terminal !== null`), not a duplicated vocabulary. Classify from the
authoritative field; don't mirror the Python enum. When the shape guard grows
complex enough to need runtime validation, use a schema derivation tool
(TypeBox) rather than hand-rolling the guard — this pattern bit twice in two
different languages.

### Quarantine Partitions Must Survive All Read Paths (Gate 3, PR #423)

The Gate 3 convergence review probe-confirmed a deletion bug: `ops/pruning`
and `ops/diag` consumed `SpawnScan` envelopes but silently shed the
quarantine partition, operating only on `.records`. A project whose spawn
rows were ALL quarantined (for example, after a schema-breaking upgrade)
would have its entire spawn directory deleted by pruning, and its diagnostic
warnings would report zero spawns. Retention similarly aborted on
`SpawnStateQuarantined` instead of preserving the row.

The fix was fail-closed: pruning skips quarantined rows (quarantine =
uncertain = keep), retention preserves quarantined segments without touching
them, and diagnostics report quarantined rows as warnings. The regression
tests exercise a quarantine-only project through all three paths.

**Lesson:** When a typed read returns an envelope with two partitions (valid
+ quarantine), every consumer must explicitly decide what to do with both.
Silently discarding the quarantine partition is the exact bug class the
quarantine system exists to prevent. This is the same shape as the
`SpawnCollection(list)` problem from wave 3, where list operations shed
quarantines — `SpawnScan` fixed the API, but callers still needed updating.

### Schema-Upgrade Mappings Must Be Derived From Real Disk, Not Old Code (PR #423)

The legacy v2→v3 upgrade codec was first written by deriving the legacy field
inventory from the previous writer's code (`origin/main`'s `spawn_store.py`).
Two of its three initial defects came from disk reality that no living code
generation describes:

- `revision` — an optimistic-concurrency field from a generation OLDER than
  main, present on 6,801 of 7,095 real rows and referenced by zero code
  anywhere. The codec's never-drop-unknown rule quarantined every row that
  carried it.
- `published_at` absent — the field was introduced recently; only 273 of
  7,096 real rows had it. The codec's completeness rule quarantined ~96% of
  real history as `incomplete_terminal`.

Both fixes were driven by a machine-wide inventory script (walk every
`state.json`, count field presence and unknown keys), which took minutes and
turned design debate into arithmetic. Final validation: 7,097 rows, 0
quarantines.

**Lesson:** Persisted history spans every schema generation that ever wrote
it, including generations whose writer code no longer exists. Before writing
an upgrade mapping, inventory the real corpus: which fields actually appear,
at what frequency, and which "required" fields are actually missing. The
previous writer's code tells you the *latest* legacy shape; the disk tells
you all of them.

## Cross-References

- [architecture/state-system/overview.md](../architecture/state-system/overview.md) — full state system architecture
- [architecture/spawn-finalization.md](../architecture/spawn-finalization.md) — discriminated lifecycle facts and quarantine
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — reaper decision logic
- [principles/design-principles.md](../principles/design-principles.md) — crash-only design, files as authority
- [principles/invariants.md](../principles/invariants.md) — projection authority rule, JSONL invariant
- [Abandoned-generation closure](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations) — the fold rule that repairs the leaked generation
- [Portable-history capture](../architecture/state-system/portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops) — the guard that can then trust real liveness signals
- [History-storage decision](../decisions/history-storage.md#d-abandoned-generation-closure) — rationale and rejected alternatives
