# lessons/state-design-lessons — Why the State System Is Built This Way

These are the choices that shaped Meridian's state layer, with the reasoning that drove each one. The "we tried X" context is what makes these valuable — without it, the decisions look arbitrary.

## Why Dual-Root (Repo + User)

**The problem we were solving:** Projects get moved, renamed, and copied. A state root tied to the project path breaks when the directory changes. But committing high-churn runtime state (spawn records, artifacts, session transcripts) to git creates commit noise, merge conflicts, and bloated repositories.

**The solution:** Split state into two roots by volatility:

- **Repo `.meridian/`** — committed scaffolding: KB, work items, agent profiles. Moves with the repo. Goes into version control.
- **User `~/.meridian/projects/<uuid>/`** — runtime state: per-spawn `state.json`, session events, artifact directories. Keyed by UUID, not path. Never committed.

**The UUID binding:** `~/.meridian/projects/<uuid>/` uses a project UUID (stored in `.meridian/id`). When you rename or move the repo, the UUID stays the same, and the user-level state is still there. This was the key insight: project identity is the UUID, not the path.

**The migration moment:** This split was introduced in `v001 uuid_state_split` (0.0.34). Before that, all state lived under repo `.meridian/`. The legacy path still worked for reading; new writes went to the user root.

**What we'd do differently:** The split is right, but the migration tooling (currently a stub) should have been implemented before the split was released. Running the old structure and the new structure in parallel required read-fallback code that complicated path resolution.

---

## Why JSONL Over SQLite

**The temptation:** SQLite would give us transactions, queries, indexes, and a schema. Running `SELECT * FROM spawns WHERE status='running'` is cleaner than replaying a JSONL stream.

**Why we didn't:**

1. **Crash tolerance.** JSONL appends are atomic at the line level — a crash mid-write leaves an incomplete line that gets skipped on read. SQLite transactions require a write-ahead log that can get corrupted under hard kills. Crash-only design favors the dumbest possible write that can be safely recovered.

2. **Inspectability.** JSONL and per-spawn JSON files work with standard tools (`cat`, `jq`) without any Meridian tooling. This matters for debugging in CI, on bare machines, in restricted environments. SQLite requires `sqlite3` and schema knowledge.

3. **No service dependency.** SQLite files are exclusive-locked when open on some platforms. Multiple concurrent Meridian processes (nested spawns, parallel CLI invocations) need to read state concurrently. JSONL with `flock` sidecar scales to N readers + 1 writer without a connection manager.

4. **Simpler crash recovery.** A truncated JSONL file is recoverable by skipping the last partial line. A corrupt SQLite database requires `PRAGMA integrity_check` and may need manual reconstruction.

**The cost:** Querying requires reading and replaying the entire file. For spawn lists this is fast (hundreds of events, not millions). If spawns scale to tens of thousands, we'd need an index. That's a future problem.

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

**The fix:** The recency check applies to any of: `{heartbeat, history.jsonl, output.jsonl, stderr.log, report.md}`. A write to any of these resets the recency window.

**The lesson:** Use the broadest safe set of liveness signals. The heartbeat is the primary signal (written every 30s unconditionally), but last-gasp activity on other artifacts should prevent premature reaping.

### Why PID Probes Are Skipped for `finalizing`

**The `finalizing` status** means the harness process has already exited — the runner is in controlled post-exit drain (report extraction, finalization). The runner PID is still alive (it's the runner, not the harness), but the worker (harness) is done.

**The problem:** PID-probing the runner during `finalizing` gives "alive" (runner hasn't exited yet), which would suppress reaping forever for a runner that crashed mid-drain.

**The fix:** For `finalizing` rows, the reaper uses only heartbeat recency, not PID liveness. A stale heartbeat during `finalizing` means the runner crashed mid-drain → `orphan_finalization`.

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

- [architecture/state-system.md](../architecture/state-system.md) — full state system architecture
- [architecture/spawn-finalization.md](../architecture/spawn-finalization.md) — discriminated lifecycle facts and quarantine
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — reaper decision logic
- [principles/design-principles.md](../principles/design-principles.md) — crash-only design, files as authority
- [principles/invariants.md](../principles/invariants.md) — projection authority rule, JSONL invariant
