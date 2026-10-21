# Decisions: State Layer

State-layer decisions cover project identity, runtime state, crash recovery, concurrency,
and typed state contracts. Spawn identity and launch containment decisions live in
[spawn-identity.md](spawn-identity.md). History-storage decisions live in
[history-storage.md](history-storage.md).

## State Layer

### Crash-only design: no graceful shutdown path

**Decision:** Meridian has no graceful shutdown protocol. The design assumption is that any process can be killed at any time, and the system must reach consistent state on the next startup.

**Constraints:** Graceful shutdown adds complexity (shutdown hooks, drain timeouts, cleanup responsibilities) and is unreliable in practice — SIGKILL bypasses all shutdown hooks. The alternative is "crash-only" design: instead of "we always clean up before exiting," the guarantee becomes "the next startup always reconciles correctly."

**Why this works:** The reconciliation path is exercised on every startup regardless of whether a crash occurred. This keeps the recovery path warm and tested, not a rarely-exercised special case.

**Alternatives rejected:**
- Graceful shutdown with SIGTERM handler — works unless SIGKILL is used; SIGKILL is common (OOM killer, user force-quit, container restart)
- Separate crash-recovery daemon — unnecessary; the reaper running on every read path achieves the same result without an additional process

See [principles/design-principles.md](../principles/design-principles.md) for full rationale.

---

### JSONL append-only event stores over SQLite or mutable JSON

**Decision (original):** Spawn and session state is stored as append-only JSONL event logs, not a database or mutable JSON files.

**Session state** still uses one global JSONL log, read through a keyed seam since #651 ([D-session-read-seam](#d-session-read-seam)). The per-chat split is the accepted design; its storage change (B2) waits on the owner's open questions: see [D-session-per-chat-journals](#d-session-per-chat-journals). **Spawn state** was subsequently migrated to per-spawn `state.json` files for performance reasons — see the spawn-state-v2 decision below. The original JSONL rationale remains active for session state.

**Constraints that drove the original JSONL design:**
1. **Crash tolerance** — JSONL appends are atomic at the OS level: a partial write corrupts only the partial line, not any previous lines.
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

### Project identity in `meridian.toml`; no repo-local state (Decision 3, 2026-07)

**Decision:** The committed, machine-managed `[project] id` in `meridian.toml` is the precedence-exempt project identity. A directory is a project when it has `meridian.toml` or `mars.toml`; zero-TOML directories remain runnable. Read-only commands create nothing.

The ID keys `~/.meridian/projects/<id>/` for runtime and `~/.meridian/context/<id>/` for default context. No state, lock, cache, generated `.gitignore`, or fallback context directory lives under repo-local `.meridian/`. Clones and worktrees retain shared history because identity remains committed and immutable.

**Alternatives rejected:** repo-local identity/state created a second root; local-only identity split clones and worktrees; path or VCS-derived identity broke moves or the no-VCS contract; precedence overrides could silently split one project across state roots.

**Status:** current for committed projects. **Superseded in part (2026-10)** by [D-two-tier-identity](#d-two-tier-identity): the clause "the first durable write creates identity", which wrote `meridian.toml` with a fresh id into any directory that received a write, no longer holds. A marker-less directory now gets path-derived state under `~/.meridian/` and no file. The rest of this record stands.

---

### D-two-tier-identity: two-tier project identity and zero-config launch (2026-10) {#d-two-tier-identity}

**Status:** decided by the user on 2026-10-14 (`work:two-tier-identity`, design revision 2; D32, D38, D40, D44 in `work:pyright-strict`). Merged to `main` in PR #575 on 2026-10-14 (`d160a07d9`, `release:minor`); released in v0.10.0 (`b8da247f9`, 2026-10-14) after #579 fixed the app-lane socket path that failed the first release-on-main preflight. Layering user agents into a project that has its own `mars.toml` uses `--fallback-root` from mars-agents 0.16.0 (PR #173), which meridian-cli pins; see [D97](package-management.md#d97-user-scope-packages-live-in-an-ordinary-mars-root-2026-10).

**Decision.** The tier of a directory depends only on what its literal project root contains:

- **Committed project:** the root holds `meridian.toml` or `mars.toml`. Identity and state behave as in Decision 3: `[project] id`, minted on the first write.
- **Ad-hoc place:** the root holds neither, or is the user Mars root `~/.meridian/mars/` itself. Its id is `adhoc.<slug>.<hash12>`, derived from the canonical path, and its state lives in `~/.meridian/projects/<id>/` and `~/.meridian/context/<id>/`. Meridian never writes a file into the directory.

Where no project Mars root exists, agents, skills, model aliases and launch targets come from **user scope**: `~/.meridian/config.toml` plus the user Mars root ([D97](package-management.md#d97-user-scope-packages-live-in-an-ordinary-mars-root-2026-10)). A project layers over user scope.

**The requirement, in the user's words:** "I just want to be able to do `meridian opencode` and run something … if we don't have meridian.toml, use default values + have all harnesses possible available." A probe (`spawn:p7814`) showed bare launches already started. What blocked `meridian opencode -a reviewer` was package discovery: agents and skills were read only from the project's `.mars/`, and `~/.mars` was never searched. The litter the July rule left behind was a symptom of the same gap.

**Invariants**, each owned by one place in the code:

1. **Tier predicate.** The tier depends only on the resolved project root: no ancestor walk, no VCS lookup, no task dir.
2. **No litter.** Meridian writes into a directory only if it already holds a marker, or through `init`, `config init` or `meridian mars init`.
3. **Runs anywhere.** Everything runs in a bare directory except project authoring: `config set/reset`, `workspace init` and `meridian mars` mutations without `--user` error and point to `meridian init`.
4. **Read purity.** Reads, `doctor` and `--dry-run` included, never mint an id, write a marker or materialize an ad-hoc root.
5. **Two deciders.** `resolve_project_identity` / `ensure_project_identity` decide where state lives; `resolve_mars_roots` decides where package content comes from for every Mars subprocess.
6. **Sharing needs commitment.** Clones and worktrees share state only through a committed `[project] id`.
7. **Disjoint namespaces.** Ad-hoc ids carry the reserved `adhoc.` prefix (a committed id may not use it), and each retention rule owns exactly one namespace.
8. **No carry-over.** Ad-hoc and committed roots never merge or move. After `init` the old ad-hoc root is unreachable and ages out.

#### Why the July model had to give

July's model rested on three premises: committed identity, a marker at the literal CWD, and zero-TOML operation. Together they turned every directory that received a write into a committed project. The record never priced that. A census on 2026-10-14 found 326 `meridian.toml` markers under `/tmp` (260 distinct ids, 3 with state in the real `~/.meridian/projects/`; the rest came from smoke tests with scratch homes) and 97 under `~/gitrepos`. `~/.meridian/projects/` held 163 roots: 6 matched a marker in a real repository, and 158 (65 MB) were UUID roots nothing could attribute to a path.

The revision keeps all three premises and narrows the first: committed identity is how state is *shared*, not the only way to *have* it. The user asked whether ad-hoc roots just move the litter into `~/.meridian/`. They do keep the root count. What they add is attribution (`origin.json` names the path) and a retention rule that can reclaim them.

**What July's rationale still governs.** The id is committed so clones and worktrees share history, and it is lazy so a project survives a move. Both remain true for committed projects. Identity still never walks up (#335: an ancestor walk silently joined the wrong project).

#### Settled specifics

| Question | Decision |
|---|---|
| Does `mars.toml` alone mark a project? | Yes, as ratified in July |
| Ad-hoc key | `adhoc.<slug>.<hash12>`: `hash12` is the first 12 hex digits of sha256 over the canonical path; the slug is the lowercased basename reduced to `[a-z0-9-]`, at most 24 characters. A recreated path gets the same id and inherits any unpruned state. A move orphans state and retention reclaims it |
| Ad-hoc metadata | `origin.json` `{path, created_at}`, written once; its mtime records activity and is touched at most hourly. No registry, no id-keyed lock |
| Legacy `.meridian/id` fallback and `meridian migrate` | Deleted. A directory with only `.meridian/id` is ad-hoc |
| Does ad-hoc state carry into a project at `meridian init`? | No. `init` writes its scaffold and a minted id in one atomic write and starts the project clean, as git, cargo and uv do. If an ad-hoc root existed it prints `note: earlier ad-hoc state at <root> is not carried over` |
| `config set` / `config reset` / `workspace init` in a bare directory | Error `no project config here; run 'meridian init'`. Prior art is nearly unanimous: `uv run` works anywhere while `uv add` needs a project |
| Ad-hoc retention | `state.adhoc_retention_days`, default 30 days quiescent; roots whose path is gone or now holds a marker age at `min(N, 7)` days; a one-hour floor applies to every ad-hoc prune. Read from user config or `MERIDIAN_STATE_ADHOC_RETENTION_DAYS` only. Rationale below |
| Creation notice | Human-mode launches print a one-time notice on materialization naming the nearest marked ancestor; agent mode and MCP get `identity_created` instead |
| Ancestor walk | None for identity. Package content keeps Mars's own walk-up, so a repo subdirectory gets the repo's agents but its own ad-hoc state |

**Retention, and why these numbers.**

- **No automatic pruning** (user, 2026-10-14). `meridian doctor --global --prune` is the only reclaimer, for both tiers. An opportunistic daily sweep (ad-hoc roots only, run from a root primary launch under the existing locks) was offered and declined: "i think its fine to not automatically prune". `doctor --global` keeps surfacing what is prunable instead.
- **`doctor --global` lists every ad-hoc root**, not only prunable ones: id, origin path, reason (`adhoc-idle` / `adhoc-path-gone` / `adhoc-shadowed`), last activity, size and `prunable: yes|no`; `--prune` acts only on the prunable ones. Pre-`init` state is findable the moment it is shadowed, which is what the `init` note promises. The first implementation listed only roots past retention, which contradicted the design smoke (G3 probe).
- **30 days idle** matches committed-root retention and was the user's choice. Last activity is the newest of every signal: `origin.json`'s mtime, the runtime-root and `context/<id>/` tree mtimes, and the spawn and session start and end timestamps. A primary that ran for 40 days and ended yesterday is kept, and so is a root whose `origin.json` is old but whose spawn ran minutes ago. The hourly `origin.json` touch alone left a near-zero protection window at the hour boundary (G3 blocker, `spawn:p7873`).
- **`min(N, 7)` for gone or shadowed roots.** Nothing can reach a root whose path now holds a marker (`adhoc-shadowed`: pre-`init` state), and a recreated path would inherit a gone one (`adhoc-path-gone`). Those roots should age *faster* than idle ones, never slower. A flat 7 would have kept them longer than idle roots when N < 7, which the G2 probe caught.
- **One-hour floor.** `work start` and `spawn done` hold no lease after `ensure`, and `ensure` touches `origin.json` at most hourly. Without a floor, `adhoc_retention_days = 0` let a prune delete a work item a second after it was created (G2 blocker, `spawn:p7868`). Retention 0 therefore means "as soon as quiescent, after one hour".
- **User config only.** The key is never read from the cwd project, so running `doctor` from inside some project cannot change how every ad-hoc root ages.
- **Prune deletes `projects/<id>/` and `context/<id>/` together.** Nothing pruned `context/` before. A recreated path would otherwise inherit an old work tree.

**The cost we accepted** (CHANGELOG wording): running `meridian init` later starts that project with empty history. Spawns, sessions and work items from before `init` remain as files under the old root until pruned. Anyone who relied on automatic creation to share state across worktrees runs `meridian init` once and commits `meridian.toml`.

#### Rejected alternatives

| Alternative | Why rejected |
|---|---|
| **Keep create-on-first-write** | Leaves a marker in every directory that receives a write and makes the root unattributable |
| **Explicit `init` only** | Breaks "runs anywhere" and revives the dead establishment gate across CLI, agent mode and MCP (`spawn:p7782`) |
| **Writes fail in a bare directory** | Same breakage as explicit `init` only, for launches |
| **Ancestor walk for identity** | #335: silently joins the wrong project |
| **Path-keyed identity for everything** | Breaks moves and splits clones and worktrees (`0b6f7e6f3`). The path key is used only where nothing is shared |
| **A path→id registry** | A second store to keep consistent; the path hash plus `origin.json` gives attribution without one |
| **Absorption on promotion** (design revision 1) | See below |
| **Adopting the ad-hoc id on `init`** | Commits a path-derived, reserved-prefix id into a shared file |
| **VCS detection or auto-`init`** | Violates the no-VCS principle; a notice tells the human instead |
| **A daily automatic ad-hoc sweep** | Declined by the user (2026-10-14); manual `doctor --global --prune` stays the rule |

**What revision 2 deleted from revision 1, and why.** Revision 1 moved a path's ad-hoc root into its committed root on the first write after a marker appeared ("absorption"). To do that safely it needed a `pending` state while live sessions held leases, `doctor --fix` to resume a pending absorption, `superseded` roots when another clone had already written under the committed id, an id-keyed lock `locks/project-id/<N>.lock`, and tests for each. It was about a third of the design's blast radius, and it broke continuations that pointed at the old root. All of it served to carry scratch history across `init`, which no prior art carries. The user gave up carry-over and called the simplified shape the long-term shape, not a stepping stone.

**Revisit when:** users report losing pre-`init` history they needed; ad-hoc root counts grow without retention catching up; or a harness needs worktrees of an uninitialized repository to share state.

Current mechanism: [project identity architecture](../architecture/state-system/project-identity.md).

---


### `mark_finalizing` CAS enables `orphan_finalization` distinction

**Status:** superseded in part. Identity-v2 PR 3 removed the reason below: the reaper no longer reads `finalizing`, a dead runner is `crashed` whichever window it died in, and nothing writes `orphan_finalization` or `orphan_run` ([derived status](../architecture/state-system/derived-status.md)). The CAS itself still runs on `main` as a status marker; [D-spawn-identity-v2](spawn-identity.md#d-spawn-identity-v2) PR 4 deletes it, and the runner then writes one outcome instead of `record_runner_exit` → `mark_finalizing` → `finalize`.

**Decision:** After the harness process exits, `streaming_runner.py` atomically transitions the spawn from `running` to `finalizing` (compare-and-swap) before beginning drain and report extraction. The reaper uses this state to distinguish `orphan_finalization` from `orphan_run`.

**Why:** Without the distinction, all crashes look the same to the reaper. `orphan_finalization` (harness completed, output likely present, only metadata persistence failed) is much less severe than `orphan_run` (harness died mid-execution). The distinction enables better diagnostics and recovery guidance.

---

### Terminal write policy as a pure function (terminal_policy.py)

> **Superseded (PR #423 wave 4, 2026-07).** `terminal_policy.py` was deleted.
> The authority lattice is now inlined in `spawn_store.py:finalize_spawn()` as
> a four-line check inside the locked mutator. The v1 reducer that also called
> it no longer exists, so the single-caller justification for a separate module
> evaporated. The authority rule itself is unchanged: authoritative origins
> replace reconciler-origin terminals; all other terminal-to-terminal writes
> are rejected.

**Decision (2026-05, spawn-finalization-refactor):** The finalization authority lattice is encoded as a pure function `decide_terminal_write(current_status, current_terminal_origin, incoming_origin)` in `state/spawn/terminal_policy.py`. The store calls this function under flock; the reducer calls it during event projection.

**Why:** The authority lattice had previously been duplicated between the store write path and the reducer projection path — two copies that could drift. Extracting it as a pure function makes the lattice a single testable artifact. It can be unit-tested without filesystem setup, making the full decision space verifiable (append / replace / reject × all status + origin combinations).

**Alternative rejected:** Inline conditionals in both the store and reducer — resulted in the two paths drifting and the projection authority rule being impossible to audit as a whole.

---

### CR1 fix: complete_spawn() always delegates to store, never short-circuits

**Decision (2026-05, spawn-finalization-refactor):** `SpawnApplicationService.complete_spawn()` no longer short-circuits when a pre-lock snapshot shows the spawn already terminal. It always delegates to `lifecycle.finalize()`, which calls `spawn_store.finalize_spawn()` under flock.

**Why:** The original short-circuit check (`if is_terminal: return early`) was a non-atomic read-then-skip. A concurrent writer could have appended a reconciler event between the read and the early return, allowing the caller to believe the spawn was already-terminal-by-authoritative when it was actually only-terminal-by-reconciler. The fix makes the store the sole decision authority: the read, decision, and write are all under flock.

**Effect:** Callers now receive a `CompleteSpawnOutcome` that accurately reflects whether _this_ call wrote the terminal event, whether it replaced a prior reconciler event, or whether a prior authoritative event already existed. The `snapshot` field carries post-write state regardless of outcome.

---

### Terminal arbitration extracted to a single module (terminal_arbitrator.py)

**Decision (2026-05, spawn-finalization-refactor):** Terminal trigger arbitration for streaming spawns is extracted from `streaming_runner.py` into `launch/streaming/terminal_arbitrator.py`. Both `run_streaming_spawn()` and `_run_streaming_attempt()` delegate to `arbitrate_terminal()`.

**Why:** Both functions previously contained near-identical `asyncio.wait()` races with slightly different priority orderings, making it easy for one path to be updated without the other. A single `arbitrate_terminal()` with documented priority order (terminal frame > budget > timeout > watchdog > completion-with-grace > signal) is the authoritative implementation. The 0.5-second completion grace — brief wait for a late terminal frame after process exit — is now a single named constant rather than an implicit behavior.

---

### StreamingRunConclusion replaces mutable sentinel locals

**Decision (2026-05):** One-turn execution outcome accumulation uses a
`StreamingRunConclusion` dataclass instead of scattered mutable locals. It
centralizes terminal-state resolution across startup, drain, extraction, guardrails,
identity conclusion, and exception handling. The runner has no launch-retry loop.

---

### PreparedExecutionHandoff with ExitStack ownership transfer

**Decision (2026-05):** `_prepare_execution_handoff()` uses ExitStack ownership transfer: a local stack enters context managers, transfers ownership to the returned handoff on success, and closes on failure. Guarantees no session-scope leak on any failure path.

---

### failure_policy.py consolidates launch-failure finalization sites

**Decision (2026-05):** All launch-failure finalization routes through `ops/spawn/failure_policy.py` with a fixed terminal tuple (`status="failed"`, `exit_code=1`, `origin="launch_failure"`). Replaced eight scattered call sites that used inconsistent exit codes.

---

### Spawn state v2: per-spawn state.json over global JSONL (2026-05)

**Decision:** Spawn state migrated from a single global `spawns.jsonl` event log to individual `spawns/<id>/state.json` files. Each file holds the full current state of one spawn; no event replay needed.

**What drove the migration:** replaying the global spawn event log made every
status read O(total history). Per-spawn `state.json` makes an individual read
O(1). The canonical dataset and timing provenance are recorded in
[Architecture: State System](../architecture/state-system/spawn-state.md).

**Two-tier write model:**

- *Tier 1 (owner writes):* The spawn's own runner writes `state.json` directly via atomic tmp+rename — no per-spawn lock. The runner is the sole writer during active execution.
- *Tier 2 (external writes):* Other processes (reaper, cancel, `update_spawn()` callers) acquire `spawns/<id>/state.lock`, read the current state, apply a mutator, write atomically. `update_spawn(claude_config_dir=...)` used by session-bleed-isolation is always tier-2 because any process may call it.

**Migration design (lazy, one-shot, no quiescence gate):**

`ensure_v2_format()` in `state/spawn/migration.py` migrates on first access. A `spawns/v2-format.json` marker file signals completion. Multiple processes race safely via `migration.lock` — the second process reads the marker and skips.

**Quiescence gate rejected:** The migration does not wait for running spawns to finish. This was deliberate: users always have running spawns, so a quiescence gate would never trigger in practice. Safety net: the reconciler (reaper) handles any spawn whose runner died mid-migration by reading v2 state and finalizing via the normal path.

**Integration with spawn-finalization architecture:**
- `decide_terminal_write()` is preserved as a pure function — application point shifted from read-time (v1 reducer) to write-time (v2 `write_state_locked()` mutator). The authority lattice and `CompleteSpawnOutcome` contract are unchanged.
- Per-spawn locking in v2 eliminates the global `spawns.jsonl.flock` contention bottleneck.

**Known remaining gap:** sessions never got the same split; they are still one global log ([D-session-per-chat-journals](#d-session-per-chat-journals) is the accepted fix). Listing now goes through the history index, but two per-spawn paths still read every `state.json` ([spawn state](../architecture/state-system/spawn-state.md), [per-operation cost](../architecture/per-operation-cost.md)).

---

### D-session-per-chat-journals: per-chat session journals replace the global log (#418 design B, 2026-10) {#d-session-per-chat-journals}

**Status:** accepted design; **B1 shipped, the storage change is not built**. #418 is
open. The owner chose option B ("redo PR 2") on 2026-10-15, and the design converged
after two adversarial review passes (`work:v010-followups` D22, D25; design in that
work item's `design-591/design-B.md`, `decision-B.md`, `pr-plan-B.md`).
- **B1** merged as #651 (`e6ae6cf53`, v0.10.16, 2026-10-16), with no format change.
  Its contract and its two deviations from the plan are
  [D-session-read-seam](#d-session-read-seam).
- **Owner answers** (`decision-B.md` "Owner answers"): Q5, automatic retirement of
  `sessions.jsonl` (B4), is **yes** (2026-10-16). Questions 1–4 and 6–11 are still
  open and must be answered before B2.
- **B2** comes after identity-v2 PR 4, which takes index v11; B2 takes v12.
  > [!FLAG] **Needs human review**
  > The session-browse branch (`feat/session-browse-redesign`) bumps the history
  > index from v10 straight to **v12** for lineage edges and model observations,
  > taking B2's reserved number and skipping v11. Once it merges, identity-v2 PR 4
  > ([spawn identity](spawn-identity.md)) and B2 need new numbers above 12.

On `main` sessions are still the one global log. #591 was closed as a duplicate of
#418, which has owned this work since the 2026-07 thermo-nuclear audit.

**Problem.** Session state is one global `sessions.jsonl`, and per-spawn operations
replay it: 9 full-log reads per foreground launch
([session state](../architecture/state-system/session-state.md#sessionsjsonl-is-one-global-log);
measurements and the acceptance bar on [per-operation cost](../architecture/per-operation-cost.md)).
It is the shape spawn state left in May 2026 ([spawn state v2](#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05)).
The owner had believed sessions already lived in per-chat files. They never did.

**Decision.**
- **Per-chat journals.** Each chat gets an append-only journal,
  `session-state/chats/<cN>.jsonl`, in today's row format. It is that chat's only
  authority.
- **Native-ID files.** The only facts that cross chats are keyed by native session
  ID, in `session-state/native/<aa>/<sha256(id)>.jsonl`: `claim` rows naming the
  chats, plus model observations.
- **Locks.** Writers take a per-chat lock and at most one native lock. Readers take
  no lock. This build never writes `sessions.jsonl`.
- **Cutover.** A one-time split of `sessions.jsonl`, published by one rename. It is
  lossless on the orange corpus (0 fold mismatches over 12,363 chats) and took
  2.1–2.3 s.
- **Absorb.** Rows that older builds still append to the legacy file are copied
  into their journals at the next `meridian` command's entry point. The fast path
  is one `stat`. Each copy is tagged `legacy_offset`, so a crash can't copy a row
  twice. The POC was exactly-once under 55 `kill -9`s with a concurrent older writer.
- **One fold rule.** An absorbed start that is older than the chat's accepted start
  adds its generation without replacing it.

Each per-run operation then reads its run row and one journal: 1.7 KB at p50, and
1.2 ms to fold the largest real chat.

**Settled specifics** (owner questions in `decision-B.md`; recommendations adopted
in D25):
- **Journal, not `state.json`.** #418 had literally prescribed
  `sessions/<chat_id>/state.json`. Every consumer of chat state reads events, so a
  state file would need a new derived schema and a read-modify-write on every
  update, to save about 1 ms.
- **Index bumps in sequence:** #631 took v10 (8 and 9 were burned by unreleased
  #616 builds), identity-v2 PR 4 ships v11, B2 ships v12. Whichever merges second
  bumps.
- **Pre-B builds are unsupported for reads** against a cut-over runtime; their
  writes are still absorbed. The cutover fails loudly on a read-only runtime.
- **Corpus search cuts over other projects' runtimes** when it initializes them.
- **Out of scope:** per-chat pruning, and the automatic archive's full scan.
- **B4, automatic retirement: yes** (owner, 2026-10-16). After the B2 upgrade, once
  14 days pass with no growth from older builds, `sessions.jsonl` is renamed to
  `sessions.jsonl.pre-418`. It is never deleted automatically.

**PR slices**, each off `main` with its own merge approval:
- **B1 (shipped, #651):** one read/append seam over today's log, keyed lookups,
  deletion of redundant reads, plus an old-versus-new equivalence harness. No format
  change.
- **B2:** the journals, cutover, absorb and index v12. It also owns the
  accepted-generation index semantics and seam-owned lock epochs that B1 deferred.
- **B3:** observer rows, so continue's live-observer check and capture's owner check
  stop scanning every spawn. That scan is the ~230 MiB primary-close residual.
- **B4:** retire the old file (approved, above).

**Rejected alternatives.**

| Alternative | Why rejected |
|---|---|
| **A: offset sidecar.** Keep `sessions.jsonl` as the only authority and add disposable per-key offset files (round 2's pick) | It avoided an authority migration, but the owner's context removed most of that risk: one user, no backwards compatibility, and upgrade survivors that live hours, not days. A leaves the global log growing forever and needs an O(log) sidecar rebuild after every reboot (1.3 s at orange). Chats can't be pruned one at a time, and writers keep sharing one global lock |
| **Ingest-on-observe** (round 1): every session-store entry ingests legacy growth, under a global ingest lock, with per-chat watermarks | It made session reads write and lock, which brought a deadlock class (review M1) and a global-lock tax on every call. It was about twice the code of absorb at entry points |
| **Ignore older-build writes after the cutover** | It loses facts the user would notice and can't recover: about 2.4 native binds and 2.3 model selections per cutover. A run that bound after the upgrade can't be resumed natively, and `--continue` forgets its model. Pre-B worktrees sharing the live runtime keep writing for weeks, not hours. It was simpler by only about 80 lines |
| **Wait for quiescence before cutting over** | It never fires while any pre-B worktree is in use, so the build would carry two live storage modes indefinitely. Spawn state v2 rejected a quiescence gate for the same reason |
| **Literal `state.json` per chat** | See "Journal, not `state.json`" above |
| **Keep the cross-chat archive witness check** | It never occurred (0 of 12,332 spawns), and the cutover reports any it finds |

**Revisit if:** a chat journal grows large enough to matter (p99 is 11 rows, the max
77), in which case compact that one file. Also revisit if absorb ever has to handle
a legacy-file rewrite rather than appends. It currently absorbs nothing and warns.

**Provenance:** `work:v010-followups` D19, D21, D22, D25; `design-591/` round 1–3
designs and reviews (`review.md`, `review-r2.md`, `review-r3.md`, `review-B.md`,
`review-B-r2.md`); design lead `spawn:p8040`; reviewers `spawn:p8059`,
`spawn:p8066`; `chat:c7736`.

---

### D-session-read-seam: one keyed read/append seam over `sessions.jsonl` (#418 B1, 2026-10-16) {#d-session-read-seam}

**Status:** shipped. #634 (MEM-1, `a7e53cadb`, v0.10.14) added the selected reads,
the byte prefilter and the scoped `SessionBindings`. #651 (#418 B1, `e6ae6cf53`,
v0.10.16) made them one seam with a multi-chat read and a single append path. The
storage behind the seam is still the one global log; B2 replaces the bodies
([D-session-per-chat-journals](#d-session-per-chat-journals)).

**Problem.** Per-operation session reads replayed all of `sessions.jsonl` and built
every row as a Pydantic object. That cost time in proportion to history, and
because CPython keeps freed arenas it also set each process's resident memory at
the peak of its largest replay: a background runner sat at 321–325 MB against
101 MB on an empty runtime, and a primary close peaked at 678 MiB
([per-operation cost §5](../architecture/per-operation-cost.md#5-resident-memory-full-scans-kept-as-pydantic-objects-627)).
B2 also needs one place to swap: a reader that opens the log itself would be
missed by the cutover.

**The seam** (`state/session_store.py`). Reads: `chats_rows`, `chat_rows`,
`chat_record`, `chat_generations`, `native_chats`, `native_observations`.
Appends: `append_chat_rows`, `append_native_rows`. `start_session` returns a
`StartedSession` (chat, generation, record), so launch no longer reads back what it
just wrote. B2 changes these bodies and nothing else. The colocated contract is
`src/meridian/lib/state/.context/history.md`.

**Invariants.** Each one exists because a draft broke it.

1. **One pass per multi-chat read, never a pass per chat.**
   `chats_rows(root, chats)` reads the selected set once; `chat_rows` is its
   single-chat case; `chats_rows(root, None)` is the explicit full snapshot. Native
   scope is `native_chats` then one `chats_rows`: two passes, whatever the alias
   count. B1's first draft offered only a one-chat read, so every multi-chat caller
   looped it, and each call was a filtered pass over the whole log.
   `bind_late_legacy_sessions` went from 0.2 s to 57–120 s, on every CLI start
   while the import marker was stale, holding `legacy-native-import.lock`.
   `record_model_selection` on a 19-alias key went from 0.87 s to 4.4–5.7 s under
   the exclusive sessions flock.
2. **Global log order, never a timestamp sort.** `ChatRows` is the per-chat mapping
   plus `.ordered`, the same rows in log order, for cross-chat decisions. Within one
   chat, row order always wins, in B2 too. The first draft rebuilt cross-chat order
   by sorting on timestamps and diverged from `main` in five tie and clock-step
   cases. **B2 must supply a stable merge of per-chat streams, each in row order**,
   not a global sort; read `pr-plan-B.md`'s `(time, chat number, row)` that way.
   One accepted exception: `SessionBindings.commit` groups its rows by chat, so
   order across chats within one commit batch is not kept. Bind rows never create a
   chat's first row and are compared only within a chat.
3. **One append path.** `SessionBindings.commit` publishes through
   `append_chat_rows` as one grouped, durable batch (one fsync). Selection and
   observation dedup run inside the append's lock epoch, so validation and
   publication are atomic. Lock order: project lifetime → history mutation (shared)
   → `sessions.jsonl.flock`. A second writer path would be a second thing B2 has to
   find.
4. **The byte prefilter only nominates.** `_row_may_select_chat` and
   `_row_may_name_native` let a selected read skip `json.loads` on other chats'
   lines. They must return true for every row the decoded check would keep; the
   decoded check decides. A false positive costs one decode; a false negative drops
   history silently. The first prefilter dropped whitespace-padded ids, which the
   decoded side strips. The fix applies the decoder's own predicate to every
   `chat_id` token, and nominates any row it can't see into (NUL bytes, an escaped
   key letter). A 100k-row fuzz found no false negatives.
5. **`SessionBindings.scope` is explicit.** `None` is a full snapshot; otherwise a
   frozenset of the chats it read. Binding outside the scope raises
   `ChatOutsideBindingScope`, a `RuntimeError` so data-error `except ValueError`
   handlers don't swallow it. Cross-chat uniqueness checks require `scope is None`.
   Without this, a cross-chat check could run inside a truncated snapshot and pass
   because the conflicting chat was never read.
6. **Poison stays in its chat.** A corrupt row in another chat no longer blocks this
   chat's read or bind; reading the corrupt chat, or the full list, still raises.
   Accepted as better crash-only behaviour.
7. **A spawn row without a chat falls back to one full pass.** `session_records_for_spawns`
   and `get_session_record_for_spawn` keep `main`'s `spawn_id` match for that case:
   the crash window between `start_session` and the spawn row learning its chat.
   Named rows stay restricted to their own chat, even in a mixed batch.

**Deviations from `pr-plan-B.md`** (recorded in `decision-B.md` "Owner answers"):
- **The accepted generation moves to B2** (`work:memory-audit` M20). Index consumers
  (`recent_sessions`, preview freshness) keep `main`'s highest-ordinal generation
  bucket, and the index stays at schema 10. The plan claimed that without absorbed
  rows the accepted generation equals the last bucket. That is false when a
  generation key is reused: the key keeps its original ordinal. No current writer
  reuses a key (all use `uuid4`), but joining on the accepted generation needs a
  new index namespace, B1's contract is no format change, and v11 belongs to
  identity-v2 PR 4. B2 carries absorbed rows and its own bump (v12), so "latest =
  accepted" lands there. Non-index `chat_record` readers keep accepted-record
  semantics.
- **Eager close-path imports are a walk plus a named tuple** (M14). A daemon thread
  at session start imports every module under `meridian.lib.state` and
  `meridian.lib.ops`, then `_CLOSE_PATH_MODULES` in `launch/session_scope.py`
  (`harness.capture_qualify`, `harness.transcript_capture`), which capture loads
  lazily at close. The point is that an in-place upgrade can't break a running
  process at close ([D-update-crash-waits-for-rust](rust-rewrite.md#d-update-crash-waits-for-rust)).
  An acceptance test checks that a full close then adds no `meridian.*` module, so
  a new lazy close-path import must join the tuple. Rejected: walking
  `meridian.lib.harness`, which loads every adapter into every runner and undoes
  the memory win; dropping eager imports. The thread suppresses its own exceptions
  and only debug-logs them, because a traceback on a primary's terminal would break
  the rule that nothing reaches a terminal a TUI owns.

**Equivalence oracle.** `tests/support/session_lookup_oracle.py` is a frozen copy of
`main`'s 17 lookup functions (AST-equal at freeze). The extended test compares it
with the seam over generated corpora that include tied and backward clocks,
aliases, late binds, historical imports and legacy arrays, and over real runtime
copies through the real disk seams. 11 real runtimes matched with 0 mismatches. B2
reuses this oracle and the pass/lock probe
(`tests/support/session_lookup_measurement.py`).

**Rejected alternatives.**

| Alternative | Why rejected |
|---|---|
| A standalone native-identity lookup for the capture guard (MEM-2 as first scoped) | Native identity isn't indexed and observers have no per-chat rows on `main`, so it would duplicate B1's native-scope seam and conflict with it |
| Startup repair from history-index candidates | A dirty index makes catch-up project the log into memory, the cost being removed, and a stale disposable projection could hide an orphan. Repair streams the authority (`iter_repair_candidates`); identity-v2 PR 4 swaps only its predicate |
| A dirty marker to skip startup repair | Another state file to keep correct; identity-v2 bounds the pass anyway |
| A per-chat read looped by multi-chat callers | Invariant 1 |
| Rebuilding cross-chat order from timestamps | Invariant 2 |
| A cap on the Codex/OpenCode event queues | Suspected for runner RSS but not reproduced: instrumented queues never passed depth 4 or 1.8 KB. A stalled consumer under a real burst is unmeasured |

**Deferred:** seam-owned lock epochs (`chat_epoch`/`native_epoch`) to B2, which
touches every lock site anyway; deduping `native_identity_candidates` inline in
`_native_session_owners`; appending an observed chat's start and stop as one batch.

**Provenance:** `work:memory-audit` M1–M24 and `evidence/` (`memory-audit.md`,
`mem3-discriminate.md`, `mem2-explore.md`, `mem1-startup-repair.md`,
`b1-one-chat.md`, `b1-recent-generation-conflict.md`); reviews `spawn:p8268`
(MEM-1), `spawn:p8288`/`p8289` and re-reviews `p8300`/`p8301` (B1);
`work:v010-followups` `design-591/decision-B.md` "Owner answers".

### D-lineage-conversation-identity: fork and `--from` ancestry belong to the chat, not a generation (2026-10-21) {#d-lineage-conversation-identity}

**Status:** settled (`work:session-browse-redesign`, plan revision after review
`spawn:p14100`); on the unmerged meridian-cli branch `feat/session-browse-redesign`.

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

**Provenance:** `work:session-browse-redesign` `plan.md` "Revision"; implementation
`spawn:p14101`; re-entry probes `spawn:p14131`, `spawn:p14150`.

---

### Presentation vs storage field naming: harness-agnostic wire names over harness-specific storage names

**Decision (2026-05):** Storage fields may retain harness-specific names for historical reasons (e.g. `claude_config_dir`); wire/presentation fields use harness-agnostic names (e.g. `session_config_dir`). Mapping lives in the query layer. Storage fields are not renamed to avoid migration cost.

---

### `_directory_env_scope` eliminates `MERIDIAN_DIRECTORY_EXPLICIT` {#directory-env-scope-eliminates-meridian_directory_explicit}

**Decision (2026-05, global-directory-flag):** When the `-C` / `--directory` flag is active, `main.py` directly unsets `MERIDIAN_RUNTIME_DIR` via a `_directory_env_scope` context manager, rather than introducing a signal env var (`MERIDIAN_DIRECTORY_EXPLICIT`) that the resolution layer would check to skip `MERIDIAN_RUNTIME_DIR`.

**Alternative eliminated:** A `MERIDIAN_DIRECTORY_EXPLICIT` flag that the resolution layer would check on every project-root read. Problems:
- Two env vars to coordinate (`MERIDIAN_PROJECT_DIR` + the signal flag).
- `MERIDIAN_DIRECTORY_EXPLICIT` itself would need cleanup propagation logic identical to what it was trying to avoid.
- The coalesce function reading both vars would grow complexity with each new override env var added.

**Approach chosen:** `_directory_env_scope` context manager in `main.py`. On entry:
- Sets `MERIDIAN_PROJECT_DIR` to the `-C` path.
- If `-C` is active, unsets `MERIDIAN_RUNTIME_DIR` entirely.

On exit: restores the original values of both env vars.

**Governing principle:** "If you can remove the value that would be misread, you don't need a flag saying 'ignore that value'." The signal var, its coalesce function, and all cleanup sites disappear. Generalizes: before adding a "skip/ignore" flag, ask whether the skippable value can simply be absent.

See [../concepts/state-model.md](../concepts/state-model.md#root-resolution) for the runtime-root derivation context and [../concepts/config-precedence.md](../concepts/config-precedence.md#the--c----directory-flag) for how `-C` works at the CLI boundary.

---

### D-WorkScope: Explicit WorkScope model over implicit path-or-None (PR #328, 2026-06)

**Decision:** Work scope is modeled as an explicit `WorkScope` frozen dataclass with two kinds — `work_item` (named, durable, repo scratch dir) and `ambient_spawn` (ephemeral, child spawn dir). Resolution is explicit (`resolve_bound_work_scope()` at bind time, `resolve_work_scope_from_parts()` from already-derived context). The raw-`Path` overload (fake-scope fabrication from a bare directory) is removed.

**Why:** Before this, work scope was implicit — a `Path | None` that could be a named work item directory, an ambient spawn directory, or a bare path with no semantic tag. Code that needed to know *what kind* of scope it was had to infer from directory structure or env vars. The explicit model makes:
- Leave-scope warnings properly worded by kind ("left work item directory" vs "left spawn working directory")
- Dashboard grouping by kind unambiguous
- Artifact counting bounded (excluded top-level names per kind)
- Bind-time scope resolution a single function call, not scattered path-construction

**WorkScope is a pure value object:** Warning wording and display logic live in the ops layer, not in `WorkScope` itself. The state layer provides the canonical scope facts; the ops layer decides what to say.

**Session work-switch precedence fix (same PR):** A same-process `work start`/`switch` sets `RuntimeContext.work_id` — this takes precedence over the launch-bound `MERIDIAN_ACTIVE_WORK_DIR`. The `work_id` provenance gates which source is trusted: an in-session active work_id wins over a bound-dir that may be stale. Child-spawn binding is preserved.

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

**Decision:** Raw writes to authoritative state are rejected by `tests/contract/test_state_write_conformance.py`, a repo-wide AST test with a documented single-entry allowlist. Stale allowlist entries are detected. The failure message names the offending call site and guides toward the correct primitive.

**Why ruff TID251 was rejected:** TID251 matches import paths, not method calls on inferred types. It cannot flag `some_path.write_text()` because it does not know `some_path` is a `Path`. The AST-based test walks call nodes and matches method names against a banned set, which is the correct granularity.

---

### Autosync AGENTS.md notice removal (PR #422, 2026-07)

**Decision:** Per-conflict AGENTS.md rewrites removed; conflict JSON is the sole durable signal. The notices modified user-owned files without a consuming contract.

---

### Conftest git-config guard deletion (PR #422, 2026-07)

**Decision:** Session-wide git-config snapshot/restore guard in `tests/conftest.py` deleted. No test writes repo-local `.git/config`; the guard falsely attributed VS Code extension writes and restored stale bytes. Test isolation already handled by stripping `GIT_*` state and temp-file global config.

---

### Torn-tail-proof JSONL appends via parse-before-discard and atomic inode replacement (PR #444, 2026-07)

**Decision:** All durable JSONL consumers (session events, launch-boundary events, permission journals, control-action journals) use `append_durable_jsonl_line`, which repairs any torn tail before appending. A complete row missing only its trailing newline is preserved and given its newline; a genuinely torn partial row is dropped. Repair writes the corrected content via atomic inode replacement (`atomic_replace` with `permissions="preserve"`), so unlocked buffered readers can never splice a fabricated hybrid event from a partial overwrite.

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

Provenance: `work:launch-boundary-resurrection`, issue #437.

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

> **Updated (PR #423 wave 4, 2026-07).** Wave 4 chose the non-duplicating
> design that the earlier enforced-equivalence approach approximated: terminal
> status lives ONLY in the top-level `status` field. `TerminalFacts` carries
> exit code, timestamps, metrics, error, and origin — but not status.
> `_RevalidatedFrozenModel` and 17 backward-compatible property accessors were
> deleted. `StoredSpawnState` uses `extra="forbid"`, so persisted rows with a
> nested `terminal.status` are quarantined rather than silently accepted.

**Decision (updated):** Runner-exit evidence and finalized-outcome evidence are frozen sub-models (`RunnerExitFacts`, `TerminalFacts`) nested under `runner_exit` and `terminal`. Top-level `status` is the sole status authority. `TerminalFacts` carries no status field. When status is terminal, `terminal` must not be `None`; when status is active or `unknown`, `terminal` must be `None`. `extra="forbid"` on the persisted model quarantines rows with extra fields, including any legacy `terminal.status`.

**Rejected (original wave 2-3 approach):** Enforced-equivalence discriminant (`status == terminal.status`) with `_RevalidatedFrozenModel.model_copy` revalidation. This worked but duplicated status authority across two fields, requiring a validator, a custom base class, and 17 compatibility properties. The thermo-nuclear audit identified the duplication as unnecessary complexity; wave 4 removed the second field entirely.

**Rejected (original wave 2-3 alternative):** Sole-carrier (a single discriminated union carrying both status and facts). The non-duplicating approach achieves the same structural guarantee without changing the persisted record shape beyond deleting one field.

---

### Applied/Declined/Missing discriminated mutation result

> **Updated (PR #423 wave 4, 2026-07).** `MutationOutcome(wrote, snapshot,
> reason)` was replaced by a discriminated union: `Applied(before, after) |
> Declined(snapshot, reason) | Missing`. Each variant carries exactly the
> evidence its callers need. Duplicate cancel requests return
> `Declined(reason="cancel intent already recorded")`.

**Decision (updated):** Locked mutators return `SpawnRecord | Decline(reason)`. `write_state_locked()` returns `LockedMutationResult`, which is exactly one of `Applied(before, after)`, `Declined(snapshot, reason)`, or `Missing`. Callers pattern-match on the variant. The lifecycle layer derives `transitioned` from comparing `Applied.before.status` and `Applied.after.status`.

**Rejected:** Four ad-hoc exception classes and two `nonlocal` smuggles that encoded mutation outcomes as control flow. The typed outcome makes the wrote/declined distinction part of the return type, not exception handling. The intermediate `MutationOutcome(wrote, snapshot, reason)` was itself replaced because boolean `wrote` left callers guessing what the `None` snapshot meant and required separate None-checks for missing-row vs declined paths.

---

### Normalize-once EventSemantics descriptor per raw event

> **Updated (PR #423 wave 4, 2026-07).** The `SemanticEvent` union
> (`ActivitySemanticEvent | TerminalSemanticEvent | SignalClearedSemanticEvent`)
> and the `SemanticClass` flag-bag were deleted. They were an unused
> intermediate representation: three legacy scalar queries (`terminal_outcome`,
> `activity_transition`, `clears_signal`) re-dispatched per event, the drain
> interpreted twice, and connections re-interpreted. Wave 4 replaced them with
> one typed `EventSemantics` descriptor (fields: `activity`, `clears_signal`,
> `terminal`) attached to `NormalizedHarnessEvent`. Each raw event is
> normalized exactly once; downstream consumers read the descriptor.

**Decision (updated):** Raw harness events stay open (unknown event types pass through). Each adapter's `HarnessBundle` registers a `HarnessSemantics` port with an event-name to `EventSemantics` descriptor table and optional payload resolver. `normalize_event()` dispatches by `HarnessId` before `event_type` and returns `NormalizedHarnessEvent` carrying the raw event and its single semantic descriptor. Shared `semantics.py` contains no harness event names. Connections own primary scope; scope construction was moved from shared semantics into each connection.

**Rejected:** Strict rejection at raw parsing — harness CLIs are unpinned; new event types arrive across minor releases. The open envelope / typed descriptor preserves forward compatibility without ignoring classification responsibility.

**Rejected (wave 2-3 intermediate):** `SemanticEvent` discriminated union with `SemanticClass` flag-bag. This created an IR that no consumer used directly: every downstream path extracted scalar properties, re-dispatching by union variant. The descriptor-over-union choice eliminates the intermediate type and the per-consumer dispatch.

---

### Directory location as sole archived-ness authority for work items

**Decision:** A work item's `archived_at` timestamp is stored but never decides whether the item is archived. Directory location (`work/<slug>/` vs `archive/work/<slug>/`) is the sole authority. Archive and reopen operations move the directory first; `__status.json` is updated inside the moved directory. `StoredWorkItemState` is the typed codec for `__status.json`; work-item `status` is an open string vocabulary (not a closed enum) because custom labels exist.

**Rejected:** Closed `WorkStatus` enum — would delete custom labels. Relocating the reconciliation heuristic (the 117-line heuristic that previously reconciled `archived_at` with directory location was deleted).

---

### Three-way work store split: work_state / work_store / work_repository (PR #423 wave 4, 2026-07)

**Decision:** The monolithic `work_store.py` was split into three modules with distinct roles: `work_state.py` (models, metadata codec, slug normalization, shared directory-location primitives), `work_store.py` (pure read projections), and `work_repository.py` (all mutations serialized behind `work-store.flock`). Nine mutation pass-through wrappers and 13 private cross-imports were deleted.

**Why:** The thermo-nuclear audit (p26 H2) identified a circular facade: `work_store.py` and `work_repository.py` had 13 private cross-imports and 9 lazy wrappers, with 4 "unused" helpers that were actually misplaced. The split eliminates the cycle by making the dependency one-way: `work_store` and `work_repository` both import from the neutral `work_state`, but never from each other.

**Rejected:** `WorkLocation` discriminated model (the "judo" suggested by the reviewer). The boundary between work store and repository could be made honest without changing the location callback contract. Retaining the two-optionals shape minimizes behavioral risk in crash-sensitive transitions that move directories between active and archive roots.

---

### One-shot versioned legacy upgrade codec over quarantining history (PR #423, 2026-07)

**Decision:** Spawn `state.json` is stamped `v: 3` by the writer. Legacy v2 (and versionless) rows upgrade in memory at the parse boundary through `state/spawn/legacy.py` BEFORE the strict `extra="forbid"` model validates. Reads never rewrite legacy files; a row's next locked mutation persists v3 naturally. The module is documented as deletable once pre-v3 rows are gone.

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
