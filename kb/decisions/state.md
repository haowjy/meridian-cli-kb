# Decisions: State Layer

State-layer decisions cover project identity, runtime state, crash recovery, concurrency,
and typed state contracts. Spawn identity and launch containment decisions live in
[spawn-identity.md](spawn-identity.md). History-storage decisions live in
[history-storage.md](history-storage.md).

## State Layer

### Crash recovery

Durable state must survive a process being killed at any time; recovery does not
depend on shutdown handlers running. Normal termination can still perform cleanup,
but SIGKILL bypasses it. See [the system overview](../overview.md#design-properties).

---

### JSONL append-only event stores over SQLite or mutable JSON

**Decision (original):** Spawn and session state is stored as append-only JSONL event logs, not a database or mutable JSON files.

**Session state** still uses one global JSONL log, read through a keyed seam since #651 ([D-session-read-seam](#d-session-read-seam)). A per-chat split is accepted but not yet built: see [D-session-per-chat-journals](#d-session-per-chat-journals). **Spawn state** was subsequently migrated to per-spawn `state.json` files for performance reasons — see the spawn-state-v2 decision below. The original JSONL rationale remains active for session state.

**Constraints that drove the original JSONL design:**
1. **Crash tolerance** — Writers serialize appends and repair a torn tail before the next append. Readers treat the newline as the commit marker, so an incomplete final line is withheld.
2. **Inspectability** — `cat sessions.jsonl | jq` works with no tooling.
3. **Simplicity** — no service dependency, no connection management, no schema versioning in the hot path.

**Alternatives rejected:**
- SQLite as the authoritative session/spawn store — richer query capability,
  but adds database locking and schema migration to the authoritative write
  path and breaks the "files as authority" principle. Disposable derived
  indexes remain compatible with that principle; see
  [D-history-file-authority](history-storage.md#d-history-file-authority).
- Mutable JSON per-spawn — reconsidered and adopted for spawn state in 2026-05 once the performance problem was measured (see spawn-state-v2 below)

---

### Project identity in `meridian.toml`; no repo-local state

**Decision:** The committed, machine-managed `[project] id` in `meridian.toml` is the precedence-exempt project identity. A directory is a project when it has `meridian.toml` or `mars.toml`; zero-TOML directories remain runnable. Read-only commands create nothing.

The ID keys `~/.meridian/projects/<id>/` for runtime and `~/.meridian/context/<id>/` for default context. No state, lock, cache, generated `.gitignore`, or fallback context directory lives under repo-local `.meridian/`. Clones and worktrees retain shared history because identity remains committed and immutable.

**Alternatives rejected:** repo-local identity/state created a second root; local-only identity split clones and worktrees; path or VCS-derived identity broke moves or the no-VCS contract; precedence overrides could silently split one project across state roots.

**Status:** current for committed projects. **Superseded in part (2026-10)** by [D-two-tier-identity](#d-two-tier-identity): the clause "the first durable write creates identity", which wrote `meridian.toml` with a fresh id into any directory that received a write, no longer holds. A marker-less directory now gets path-derived state under `~/.meridian/` and no file. The rest of this record stands.

---

### Committed and ad-hoc project identity {#d-two-tier-identity}

A root containing `meridian.toml` or `mars.toml` is a committed project. Its
`[project] id` is minted on first write and lets clones and worktrees share
state. A root with neither marker, or the user Mars root itself, is ad hoc: its
ID derives from its canonical path and its state stays under the user home.
Read-only commands create nothing. Identity does not walk ancestors or use VCS;
Mars package discovery retains its own walk-up behavior. See
[project identity architecture](../architecture/state-system/project-identity.md).

A bare directory can launch using user-scope configuration and packages without
receiving a `meridian.toml` marker. Project-authoring mutations require an
initialized project. When no project Mars root exists, content comes from the
[user Mars root](package-management.md#user-scope-packages-live-in-an-ordinary-mars-root-2026-10).
A project layers over user scope.

Ad-hoc IDs use the reserved `adhoc.` prefix, a basename-derived slug and a hash
of the canonical path. `origin.json` records the path and activity. Recreating a
path can recover unpruned state; moving it cannot. Initializing a project starts
new committed state rather than adopting its old ad-hoc state. Absorbing that
state was rejected because live sessions would require pending and recovery
states, additional locks and continuation handling for scratch history.

`doctor --global --prune` reclaims eligible roots; pruning is not automatic.
The default ad-hoc retention is 30 days of inactivity. Gone or newly marked
paths use the shorter of that value and seven days, with a one-hour floor to
protect operations that hold no lease after creation. Retention is user-scoped,
and pruning removes both runtime and context roots. The doctor lists roots
before they are eligible, making the pre-init state discoverable.

The two tiers retain shared, movable identity for committed projects without
leaving a marker in every scratch directory. An ancestor identity search was
rejected because it can silently join the wrong project; path-derived identity
for every project would split clones and break moves.

---

### Per-spawn state

Per-spawn state avoids replaying all spawn history for an individual read. See [spawn state](../architecture/state-system/spawn-state.md) and [spawn identity](spawn-identity.md).

### Proposed per-chat session journals {#d-session-per-chat-journals}

**Status:** design accepted; storage migration not built. Session state still uses
one `sessions.jsonl` file. Selective reads have shipped; see [session read seam](#d-session-read-seam).
The migration is tracked in [#418](https://github.com/haowjy/meridian-cli/issues/418).

The proposed design gives each chat an append-only journal and stores cross-chat
native-session claims and model observations by native ID. Writers take a per-chat
lock and at most one native lock; readers take no lock. A one-time, lossless
split migrates the global file. At command entry, new builds absorb subsequent
old-build appends, tagging each row by its legacy offset to prevent duplication
across crashes. Once the old file has not grown for 14 days, it is renamed, not
deleted. The cutover refuses a read-only runtime; older builds cannot read the
new format, although their appends are absorbed. Corpus search also cuts over
other projects' runtimes when initializing them.

The proposal addresses repeated global-log scans and shared-writer contention.
The prototype split 12,363 chats without fold mismatches in 2.1–2.3 seconds;
absorption survived 55 forced kills with a concurrent old writer. A journal was
chosen over per-chat `state.json` because consumers read events and a state file
would add a derived schema and a read-modify-write for each update. A global
log with per-key offsets would retain the global lock and require rebuilding its
index. Ingesting old writes on every read would add a global lock and a deadlock
risk; ignoring them would lose resume and model-selection facts. Waiting for
all old worktrees to go quiet could prevent cutover indefinitely.

Open design questions remain before implementation. Per-chat pruning and the
automatic archive's full scan are outside this change. Observer rows are a
separate proposed follow-up to avoid scanning every spawn for live observers.

---

### Session read seam {#d-session-read-seam}

Keyed session reads avoid materializing an entire global journal for each operation. A measured runner fell from roughly 325 MB to about 100 MB after selective reads. A prefilter may nominate extra rows but must never discard a row the decoded predicate would select. A multi-chat read must scan once rather than once per chat; a per-chat loop previously turned a 0.2 s repair into a 57–120 s operation.

### D-lineage-conversation-identity: fork and `--from` ancestry belong to the chat, not a generation (2026-10-21) {#d-lineage-conversation-identity}

**Status:** shipped in meridian-cli PR #714.

**Decision.** A chat's parents are part of its identity, the way its native session
binding is:
- **Two edges.** `forked_from_chat_id`/`forked_from_history_id` (from `--fork`) and
  `context_from_chat_id`/`context_from_history_id` (from a primary's `--from`) are
  recorded on the start event. Each pair moves together.
- **Carried across continuation.** The session fold keeps the pair from the first
  start event that set it, and every later generation's start inherits it, along
  with the chat's original creation time. The fold names these chat-owned fields
  once (`_CHAT_LINEAGE_FIELDS`, `_CHAT_OWNED_FIELDS` in `state/session_fold.py`).
- **Resolved once.** A primary's `--from` is resolved to chat id and history id at
  the CLI by the same resolver that builds the prompt context, and persisted from
  that result ([D-from-request-boundary](launch.md#d-from-request-boundary)).
- **Indexed, never authoritative.** The history index projects parent edges into a
  table (schema 12) so finding a chat's children is a query. The journal stays the
  only copy; the index rebuilds from it.
- **Generation metadata stays per generation.** The tmux target captured at launch
  describes one launch and is not carried forward.

Sessions recorded before this change have no `context_from_*`, and lost
`forked_from_*` on any continuation. They read as "no edge", never as a guess.

**Why.** Continuing a forked chat wrote a new start event without its parent, so the
fork link vanished after the first `--continue` (repro: parent `c1` before, `None`
after). Fixing the lineage view's reads alone would have left every other reader of
the fold with the same hole; the class is "chat-owned facts treated as generation
facts", and the fold is where it is fixed.

**Rejected:** lineage on `SessionSettings`, the per-launch settings, because
continuation replaces those by design; deriving lineage from SQLite alone, which
would make a disposable projection the only copy of a fact.

Spawn `--from` is deliberately not an edge: lineage covers primary sessions only
([D-browse-lineage-view](session-browse.md#d-browse-lineage-view)).


---

### Presentation vs storage field naming: harness-agnostic wire names over harness-specific storage names

**Decision (2026-05):** Storage fields may retain harness-specific names for historical reasons (e.g. `claude_config_dir`); wire/presentation fields use harness-agnostic names (e.g. `session_config_dir`). Mapping lives in the query layer. Storage fields are not renamed to avoid migration cost.

---

### Directory flag and inherited project scope {#directory-env-scope-eliminates-meridian_directory_explicit}

When `-C` selects a different project from the inherited one,
`_directory_env_scope` removes inherited project-scoped variables, including the
runtime override and caller's chat, spawn, work and task directory. The caller
identity is retained separately for message attribution. Environment values are
restored on exit. A flag saying to ignore individual inherited values would
require each resolver to implement the same boundary. See
[config precedence](../concepts/config-precedence.md#the--c----directory-flag).

---

### D-WorkScope: Explicit WorkScope model over implicit path-or-None (PR #328, 2026-06)

**Decision:** Work scope is modeled as an explicit `WorkScope` frozen dataclass with two kinds — `work_item` (named, durable, repo scratch dir) and `ambient_spawn` (ephemeral, child spawn dir). Resolution is explicit (`resolve_bound_work_scope()` at bind time, `resolve_work_scope_from_parts()` from already-derived context). The raw-`Path` overload (fake-scope fabrication from a bare directory) is removed.

**Why:** Before this, work scope was implicit — a `Path | None` that could be a named work item directory, an ambient spawn directory, or a bare path with no semantic tag. Code that needed to know *what kind* of scope it was had to infer from directory structure or env vars. The explicit model makes:
- Leave-scope warnings properly worded by kind ("left work item directory" vs "left spawn working directory")
- Dashboard grouping by kind unambiguous
- Artifact counting bounded (excluded top-level names per kind)
- Bind-time scope resolution a single function call, not scattered path-construction

**WorkScope is a pure value object:** Warning wording and display logic live in the ops layer, not in `WorkScope` itself. The state layer provides the canonical scope facts; the ops layer decides what to say.

Work switches write the caller's session row, which outranks launch environment
for managed executions. See [execution-owned links](session-work-links.md#d-execution-owns-links).

See [concepts/../codebase/work-items.md](../codebase/work-items.md#workscope-named-vs-ambient).

---

### Concurrency by construction: mutate-under-lock seams over convention-enforced write tiers (PR #422, 2026-07)

**Decision:** The prior two-tier write model (owner writes without lock / external writes with lock) was collapsed into a single locked mutation seam: every mutation of published state calls a `write_state_locked`-shaped function that acquires a stable lock, re-reads current state, applies a pure mutator, and writes atomically. The public unlocked `write_state()` was deleted. This applies across all stores: spawn records, archived spawns, work items, hook intervals, scope projections, and autosync transactions. (The archived-spawns store was deleted in 2026-10 with the rest of the unreachable archive overlay; see [Codebase cleanup](codebase-cleanup.md#d-spawn-archive-deleted-delete-the-orphaned-overlay-do-not-keep-it).)

**Why:** The two-tier split was the root cause of every lost-update bug the thermo-nuclear audit reproduced. The "convention" that owner-writes skip the lock was unenforceable: any process calling the public `write_state()` could race the runner's unlocked write path, and five stores instantiated the same absent abstraction independently. Collapsing to one locked seam makes the lost-update shape structurally unwritable.

**Seam shape as a durable contract (user decision):** these seams are behavior-preserving contracts that a planned future store rewrite (#423 typed-state, #376 store scaling) inherits. The shape (lock-acquire, re-read, pure-mutate, atomic-write) is the invariant; the implementation details may change.

---

### Stable lock inodes with validated-exclusive GC seam (PR #422 + PR #444, 2026-07)

**Decision (PR #422):** All coordination locks live under `locks/<domain>/` (e.g. `locks/spawns/`, `locks/process-scopes/`, `locks/reaper-cleanup/`, `locks/launch-boundary/`, `locks/hooks/`) outside the directories they protect. (`locks/launch-boundary/` went with the launch-boundary journal in identity-v2 PR 3.)

**Why:** POSIX `flock` is per-open-file-description, not per-path. If one process unlinks a lock file and creates a new one while another holds the old inode, both processes believe they hold "the lock" on different inodes — split-brain. The audit reproduced this failure.

**Amendment (PR #444, closes #427):** Lock inodes are never unlinked except through `unlink_validated_lock()`, which unlinks only while holding a fresh, non-reentrant exclusive flock whose `fstat` matches the current `stat` of the path, immediately before release. The existing acquire-side revalidation loop makes this provably split-brain-free: a blocked acquirer that finds the inode gone (or changed) retries onto the recreated inode.

Two call sites use this primitive:
- `lock_gc.py::gc_orphaned_locks()` sweeps orphaned per-spawn locks (four classes) when the corresponding spawn directory no longer exists. Wired into doctor background repairs and post-prune. A non-blocking meta-lock (`locks/gc.lock`) serializes sweeps for churn control; safety is per-inode, not meta-lock dependent.
- `cleanup_stale_sessions()` unlinks cleaned session locks before release, bounding the session-lock rescan with zero new state.

**The forbidden pattern:** unlinking while the lock remains held afterwards. Concrete example: `delete_published_spawn` uses `reentrant=True`; an inline unlink would strand the outer frame on an orphaned inode while a new acquirer validates against a recreated path — split-brain. No unlink at ordinary release time, no unlink inside reentrant contexts.

**Striping rejected (PR #444 design doc):** A bounded lock-stripe namespace (hash spawn IDs onto a fixed set of lock files) was evaluated and rejected on three grounds: (1) reentrant holds would silently lose exclusion between stripe-colliding spawns; (2) session locks are lifetime-held liveness beacons whose acquirability is the staleness signal — striping collapses this; (3) larger surface than the validated-EX sweep for the same correctness goal. Other rejected alternatives: opportunistic unlink at release time (forbidden pattern above), unlink without holding (meta-lock alone does not exclude holders), epoch/generation lock directories (moves the problem without solving it).

**Alternatives rejected (PR #422, original decision):**
- Unlink-while-holding: still allows a third process to create a new inode after unlink, producing split-brain between holder and newcomer.
- Revision-CAS without re-read-and-reapply: requires the caller to hold the complete previous state, which is fragile across process boundaries and doesn't compose with pure mutators.

---

### Project-lifetime shared/exclusive gate (PR #422, 2026-07)

**Decision:** `~/.meridian/projects/.locks/<uuid>.lock` is a project-lifetime coordination lock outside the deletable project root. Sessions hold a shared lock for their lifetime; global pruning acquires exclusive + revalidates before removal.

**Why:** The audit's gate-1 review reproduced global pruning destroying a runtime root while sessions held spawn locks inside it. The shared/exclusive gate prevents this without requiring pruning to enumerate all active sessions.

---

### Permission policy split: preserve-mode for user files, strict 0600 for runtime state (PR #422, 2026-07)

**Decision:** `atomic_replace()` in `lib/platform/atomic.py` accepts `permissions="preserve"` (default, keeps existing file mode) or `permissions=0o600` (strict mode for runtime state). State-facing writes enforce 0600; user-owned project files and context work-item metadata preserve the original mode.

**Why:** The initial atomic-replace primitive unconditionally set 0600, which flipped user-owned files (like `mars.toml`) to unreadable modes. The split policy was driven by the gate-2 review reproducing Codex rollout permission broadening.

---

### AST conformance guard over ruff TID251 for raw-write ban (PR #422, 2026-07)

**Decision:** Raw writes to authoritative state are rejected by `src/meridian/dev/lints.py`, a source-tree AST lint with a documented single-entry allowlist. Stale allowlist entries are detected. The failure message names the offending call site and guides toward the correct primitive.

**Why ruff TID251 was rejected:** TID251 matches import paths, not method calls on inferred types. It cannot flag `some_path.write_text()` because it does not know `some_path` is a `Path`. The AST-based test walks call nodes and matches method names against a banned set, which is the correct granularity.

---

### Autosync AGENTS.md notice removal (PR #422, 2026-07)

**Decision:** Per-conflict AGENTS.md rewrites removed; conflict JSON is the sole durable signal. The notices modified user-owned files without a consuming contract.

---

### Torn-tail-proof JSONL appends via parse-before-discard and atomic inode replacement (PR #444, 2026-07)

**Decision:** Durable JSONL writers, including session events, permission journals and control-action journals, use `append_durable_jsonl_line`, which repairs any torn tail before appending. A complete row missing only its trailing newline is preserved and given its newline; a genuinely torn partial row is dropped. Repair writes the corrected content via atomic inode replacement (`atomic_replace` with `permissions="preserve"`), so unlocked buffered readers can never splice a fabricated hybrid event from a partial overwrite.

**Why:** A SIGKILL mid-append leaves a partial last line. The next append previously concatenated its content onto the torn tail, making both the torn row and the new row unreadable. For session events, this made the session permanently unstopped after its lock was unlinked (the stop event was the new row). For permission/control journals, torn tails allowed `transition_seq` reuse.

**Design choices:**
- *Parse-before-discard:* the tail is parsed as JSON before deciding whether to keep or drop it. A complete object row missing only `\n` is common (valid write, missing only the delimiter) and must not be discarded.
- *Atomic inode replacement over in-place truncation:* `os.truncate` would mutate the file in place; an unlocked reader with a buffered file descriptor could read bytes from the old content followed by bytes from a new append, producing a fabricated hybrid event. Atomic replacement creates a new inode, so the reader sees either the old file (complete) or the new file (repaired), never a splice.
- *O(1) fast path:* the clean-tail check reads only the last byte (`seek(-1, SEEK_END)`); no whole-file read per append when the tail is intact.

---

### One JSONL commit rule: a row exists when its newline does (cleanup phase 2, 2026-10)

**Decision (user, 2026-10-14):** Every JSONL file is read through `state/jsonl.iter_jsonl()` and appended through `append_durable_jsonl_line(s)` in the same module, under one rule. A row is committed when its newline is written; a final row without one is withheld until the next append repairs it. A malformed complete row is skipped (with a debug-level diagnostic), except where a reader must fail closed: archive receipts raise, because a missing receipt would change archive decisions. The single-row writer owns the newline and rejects embedded CR/LF, so one call can never commit two rows.

**What it replaced.** Four readers, each with a hand-written loop and its own policy:

| Reader | Torn last line | Malformed middle line | Empty line |
|---|---|---|---|
| Event store | shown if it parsed as JSON | silently skipped | skipped |
| History index | ignored until complete | silently skipped | skipped |
| Archive receipts | ignored | crash | crash |
| Telemetry | shown if it parsed | silently skipped | skipped |

**Options put to the user:**

- **A. The newline is the commit marker** (chosen). Cost: after a rare crash that splits a write exactly before its newline, one valid record stays invisible until something appends again, which for a finished session may be never.
- **B. One reader, but a valid JSON object on the last line counts without its newline**, as the event store and telemetry did. No record is ever withheld, but the reader needs an exception (do not advance its offset past that line), and it shows a record the writer never finished committing.
- **C. Keep the four loops.** The archive catalog keeps crashing on an empty line, silent skips keep hiding corruption, and the next JSONL file gets a fifth loop.

**Why A.** The writer already repairs the tail before every append, so nothing is ever concatenated or lost, only delayed. A runtime probe confirmed no authoritative reader depends on the tail: terminal status, liveness and `spawn wait` read `state.json`, and the history index already withheld torn tails on `main`. That made B's extra availability unnecessary, and A is the simpler invariant to state. The probe also found a pre-existing bug: the telemetry sink wrote JSON and newline separately with no repair, so a torn tail swallowed the next record. The sink now appends through the same durable primitive.

---

### Spawn deletion composition owner: spawn_aggregate.py (PR #444, 2026-07)

**Decision:** `delete_published_spawn()` and its precondition type live in `spawn_aggregate.py`, which composes the spawn repository and process-scope projection persistence leaves. The dependency is one-way: `spawn_aggregate` imports from both leaves; neither leaf imports the other or the aggregate.

**Why:** `spawn_store.py` had grown to 988 lines by hosting both per-spawn mutation and cross-leaf deletion. The thermo-nuclear consistency review identified the extraction as a structural fix: the deletion seam composes two leaves (spawn lock then projection lock) and should not live inside either leaf.

---

### Published-row lifetime owns spawn artifacts (issue #437, 2026-07)

**Decision:** A spawn-owned artifact may be created or changed only while that
spawn has a published `state.json` row. Parent-creating or otherwise late
mutations use `mutate_published_spawn_artifact()` in `spawn_aggregate.py`. The
seam acquires the stable external per-spawn lock, re-reads the row while locked,
optionally checks a current-row predicate, and runs the artifact mutation only
if the row is still eligible. `delete_published_spawn()` uses the same outer
lock, so deletion and a late artifact write have a total order.

Simple writers that do not need to create parents, such as heartbeats (deleted in identity-v2 PR 3), may fail
closed on `FileNotFoundError` instead of taking the aggregate seam. Connection
startup and child-process adoption treat the published spawn directory as a
precondition; they never manufacture it. Late diagnostics are best-effort and
may be dropped rather than outlive their authoritative row.

**Why:** The spawn directory was both the aggregate being deleted and the place
where low-level helpers recreated parents. Retention could remove a terminal
spawn, then an awaited callback could append a journal, write metadata, or
touch a heartbeat and reconstruct `spawns/<id>/` without `state.json`. The
result was a ghost directory: invisible to authoritative listing, confusing to
cleanup, and able to delay external lock GC. Checking existence before the
write was insufficient because deletion could win between the check and the
mutation.

**Rejected:**

- Guard only the launch-boundary writer. Harness callbacks, history drains,
  signals, scope registration, metadata, reaper evidence, and other
  asynchronous writers share the same lifetime race.
- Teach atomic-write and JSONL helpers about spawn publication. Those helpers
  are deliberately domain-blind and serve state outside the spawn aggregate.
- Preserve direct connection startup without a published row by recreating the
  directory. Missing publication is a lifecycle violation, not a compatibility
  case.

The resulting lock order keeps the per-spawn lifetime lock outside specialized
artifact locks: spawn lock before launch-boundary append lock (that journal was
deleted in identity-v2 PR 3), and spawn lock before process-scope projection lock.

See issue #437.

---

### Mutate-under-lock seams stay store-specific; generic mutation framework rejected (PR #444, 2026-07)

**Decision:** The six mutate-under-lock seams (spawn records, archived spawns, work items, hook intervals, scope projections, autosync — archived spawns since deleted) stay store-specific. A generic mutation framework (e.g. a `LockedStore[T]` base class or protocol) was evaluated in the thermo-nuclear consistency review and rejected.

**Why:** The seams share a shape (lock, re-read, pure-mutate, atomic-write) but diverge in lock-path computation, read/write codecs, file layout, and error handling. A generic abstraction would need enough parameters to reconstruct each store's specifics, providing the cost of an abstraction without reducing the decision space. The shared mechanism is already `lock_file` + the atomic-write canon; the seam shape is a documented behavioral contract, not a code-level inheritance hierarchy.

**Still holds after cleanup phase 2.** Phase 2 added `state/json_doc.py` (`read_model` / `write_model` / `mutate_model`) under eight small JSON sidecars. It is a set of functions over a Pydantic model and a caller-supplied lock, not a store base class: each store still computes its own lock and path. `mutate_model()` takes the lock as a required argument so an unlocked read-modify-write cannot be written.

---

## Typed State Contracts (PR #423, 2026-07)

Six decisions made invalid states unparseable or unconstructible, eliminating bug classes that existed only because the types permitted them.

### Single `SpawnStatus` StrEnum authority with member-derived lifecycle sets

**Decision:** `SpawnStatus` is a `StrEnum` in `core/domain.py`. Lifecycle sets (`ALL_SPAWN_STATUSES`, `ACTIVE_SPAWN_STATUSES`, `TERMINAL_SPAWN_STATUSES`) are derived from a `dict[SpawnStatus, SpawnLifecycleClass]` map. `TerminalSpawnStatus` is a `Literal` alias verified against the terminal set at import time.

**Rejected:** Positional ordering with magic count (the sets were previously hand-curated frozensets of strings whose membership could drift from the Literal duplicate). Keeping the Literal as an independent authority (it still exists for static type-checking, but the import-time guard ties it to the StrEnum).

---

### Quarantine, not coercion, for out-of-vocab persisted rows

**Decision:** `StoredSpawnState` validates vocabulary fields in a `model_validator(mode="before")`. Rows with unknown `status`, `kind`, `launch_mode`, or nested fact vocabularies are quarantined: single reads raise `SpawnStateQuarantined`; collection reads partition into `SpawnScan` (valid rows + quarantine reports). Migration and retention fail closed on quarantine — an unreadable row may contain active work.

**Rejected:** Silently omitting unknown rows (loses data). Coercing to `failed` (misrepresents a parse failure as a lifecycle outcome). The quarantine seam validates type before string operations so a non-string value routes to quarantine rather than crashing with `AttributeError`.

---

### Single status authority with non-duplicating terminal facts


**Decision (updated):** Runner-exit evidence and finalized-outcome evidence are frozen sub-models (`RunnerExitFacts`, `TerminalFacts`) nested under `runner_exit` and `terminal`. (Since v0.18.0 nothing writes `runner_exit`; it is a read-only legacy field.) Top-level `status` is the sole status authority. `TerminalFacts` carries no status field. When status is terminal, `terminal` must not be `None`; when status is active or `unknown`, `terminal` must be `None`. `extra="forbid"` on the persisted model quarantines rows with extra fields, including any legacy `terminal.status`.

**Rejected (earlier approach):** Enforced-equivalence discriminant (`status == terminal.status`) with `_RevalidatedFrozenModel.model_copy` revalidation. This worked but duplicated status authority across two fields, requiring a validator, a custom base class, and 17 compatibility properties. The thermo-nuclear audit identified the duplication as unnecessary complexity; the decision removed the second field entirely.

**Rejected (earlier alternative):** Sole-carrier (a single discriminated union carrying both status and facts). The non-duplicating approach achieves the same structural guarantee without changing the persisted record shape beyond deleting one field.

---

### Applied/Declined/Missing discriminated mutation result


**Decision (updated):** Locked mutators return `SpawnRecord | Decline(reason)`. `write_state_locked()` returns `LockedMutationResult`, which is exactly one of `Applied(before, after)`, `Declined(snapshot, reason)`, or `Missing`. Callers pattern-match on the variant. The lifecycle layer derives `transitioned` from comparing `Applied.before.status` and `Applied.after.status`.

**Rejected:** Four ad-hoc exception classes and two `nonlocal` smuggles that encoded mutation outcomes as control flow. The typed outcome makes the wrote/declined distinction part of the return type, not exception handling. The intermediate `MutationOutcome(wrote, snapshot, reason)` was itself replaced because boolean `wrote` left callers guessing what the `None` snapshot meant and required separate None-checks for missing-row vs declined paths.

---

### Normalize-once EventSemantics descriptor per raw event


**Decision (updated):** Raw harness events stay open (unknown event types pass through). Each adapter's `HarnessBundle` registers a `HarnessSemantics` port with an event-name to `EventSemantics` descriptor table and optional payload resolver. `normalize_event()` dispatches by `HarnessId` before `event_type` and returns `NormalizedHarnessEvent` carrying the raw event and its single semantic descriptor. Shared `semantics.py` contains no harness event names. Connections own primary scope; scope construction was moved from shared semantics into each connection.

**Rejected:** Strict rejection at raw parsing — harness CLIs are unpinned; new event types arrive across minor releases. The open envelope / typed descriptor preserves forward compatibility without ignoring classification responsibility.

**Rejected (intermediate):** `SemanticEvent` discriminated union with `SemanticClass` flag-bag. This created an IR that no consumer used directly: every downstream path extracted scalar properties, re-dispatching by union variant. The descriptor-over-union choice eliminates the intermediate type and the per-consumer dispatch.

---

### Directory location as sole archived-ness authority for work items

**Decision:** A work item's `archived_at` timestamp is stored but never decides whether the item is archived. Directory location (`work/<slug>/` vs `archive/work/<slug>/`) is the sole authority. Archive and reopen operations move the directory first; `__status.json` is updated inside the moved directory. `StoredWorkItemState` is the typed codec for `__status.json`; work-item `status` is an open string vocabulary (not a closed enum) because custom labels exist.

**Rejected:** Closed `WorkStatus` enum — would delete custom labels. Relocating the reconciliation heuristic (the 117-line heuristic that previously reconciled `archived_at` with directory location was deleted).

---

### Three-way work store split: work_state / work_store / work_repository (PR #423, 2026-07)

**Decision:** The monolithic `work_store.py` was split into three modules with distinct roles: `work_state.py` (models, metadata codec, slug normalization, shared directory-location primitives), `work_store.py` (pure read projections), and `work_repository.py` (all mutations serialized behind `work-store.flock`). Nine mutation pass-through wrappers and 13 private cross-imports were deleted.

**Why:** Review identified a circular facade: `work_store.py` and `work_repository.py` had 13 private cross-imports and 9 lazy wrappers, with 4 "unused" helpers that were actually misplaced. The split eliminates the cycle by making the dependency one-way: `work_store` and `work_repository` both import from the neutral `work_state`, but never from each other.

**Rejected:** `WorkLocation` discriminated model (the "judo" suggested by the reviewer). The boundary between work store and repository could be made honest without changing the location callback contract. Retaining the two-optionals shape minimizes behavioral risk in crash-sensitive transitions that move directories between active and archive roots.

---

### One-shot versioned legacy upgrade codec over quarantining history (PR #423, 2026-07)

**Decision:** Spawn `state.json` writes use v4 for rows with launch data and v3 for rows without it. Legacy v2 (and versionless) rows upgrade in memory at the parse boundary through `state/spawn/legacy.py` before the strict `extra="forbid"` model validates. Reads never rewrite legacy files; a row’s next locked mutation persists the appropriate format. The module is documented as deletable once pre-v3 rows are gone.

The never-guess posture holds, with two curated, evidence-based exceptions derived from a real-disk inventory (7,096 rows across all projects on a dev machine):

- `RETIRED_LEGACY_FIELDS = {"revision"}` — a pre-main optimistic-concurrency field present on 6,801 rows and referenced by zero living code; deliberately dropped during upgrade. Fields outside this set still quarantine as `unknown_fields`.
- `published_at := finished_at` backfill — `published_at` is a recent field (273 of 7,096 rows had it); for pre-field history, finish time is defined as the publication moment. A total deterministic mapping of a metadata timestamp, not value guessing.

Meaning-carrying conflicts still quarantine: nested/top-level status disagreement, partial runner-exit or terminal facts, terminal facts on an active status.

**Validation:** machine-wide scan of 7,097 real rows → 0 quarantines.

**Alternatives rejected:**
- Tolerant reader inside the model — a permanent legacy-reading branch is a second schema whose meaning is decided by reader fallbacks; it rebuilds the drift disease this PR deletes.
- Making `TerminalFacts.published_at` optional — weakens the new contract for all future rows to accommodate old ones.
- Quarantining all legacy rows (the initial no-backcompat posture) — every machine's entire spawn history goes dark on upgrade; the practical cost was never weighed as its own decision until a user asked "why can't we read legacy?"

---

## Related

- [../architecture/spawn-finalization.md](../architecture/spawn-finalization.md) — full subsystem architecture for the 2026-05 refactor and typed state contracts
- [../architecture/state-system/overview.md](../architecture/state-system/overview.md) — state-system mechanism
- [../architecture/state-system/portable-history.md](../architecture/state-system/portable-history.md) — archive publication, verification, and reclaim behavior
- [../concepts/state-model.md](../concepts/state-model.md) — state mental model
