# open-questions/future-work — Deferred Items and Known Gaps

Tracked follow-ups deferred from implementation or design sessions. Each entry names the issue, the context, and why it was deferred. This is not a backlog — it's a record of known state so future work doesn't re-derive the same conclusions.

Items are organized by domain.

---

## Managed Primary

### Primary message routes: session moves and remaining edges (#916)

**Where:** the Codex and OpenCode primary route, and the Pi `meridian-message` extension.

**State:** every primary harness has a route
([D-unified-delivery](../decisions/primary-inject.md#d-unified-delivery)). What remains is
in [#916](https://github.com/haowjy/meridian-cli/issues/916):

- **Following the TUI.** A sender is refused after a session move where a Claude or Pi primary
  would take the message. Codex could follow `/new` and `/fork`; `/resume` to a loaded
  thread gives no signal, OpenCode gives none until the human types in the new or other
  session, and `harness_session_id`, the idle sensor and
  `--continue` would have to move too
  ([D-primary-session-move](../decisions/primary-inject.md#d-primary-session-move), which the
  owner has not yet answered).
- **A false "delivered".** An OpenCode message sent after the new-session or switch-session
  command, before the human types there, goes to the first session.
- **A possible false refusal.** A Codex subagent reopened by `resume_agent` in a continued
  session would load without a `thread/started`, so its status event would read as a move.
  Inferred from a guardian thread loaded that way; not reproduced.
- **Untested.** OpenCode V1 detection; no V1 server was available.
- **Stale Pi listener.** An old Pi outliving its launcher during a `--continue` could unlink the
  newer session's socket. It should not happen: a dead launcher closes the PTY and hangs up
  Pi, and a second live TUI gets `Busy` from the launch lock.

---

## State & Spawn

### Restore metadata hash still depends on the reader's models (#537)

**Status:** Open, [#537](https://github.com/haowjy/meridian-cli/issues/537).
`restored-from.json` stores `session_sha256` as a hash of the folded
`SessionRecord`, so a runtime that 0.6.7 restored into fails a repeat restore under
the new build ("Restored metadata changed"). It is the same model coupling that the
[portable-digest decision](../decisions/history-storage.md#d-history-portable-digest-stored-json)
removed for record digests. Fix: hash the stored `historical_import` session event
JSON. Reads and a first restore are unaffected.

### Per-operation cost program

**Status:** Open. The bar (a per-spawn operation's cost must not grow with project
history), the measured violators and the measurement method are on
[per-operation cost](../architecture/per-operation-cost.md). This entry only lists
the open issues and how they depend on each other:

- [#418](https://github.com/haowjy/meridian-cli/issues/418): sessions move to per-chat
  journals ([D-session-per-chat-journals](../decisions/state.md#d-session-per-chat-journals)).
  B1, the read/append seam, shipped as #651 (v0.10.16;
  [D-session-read-seam](../decisions/state.md#d-session-read-seam)). B2 needs the
  owner's answers to design questions 1–4 and 6–11 (Q5, automatic retirement, is
  yes). B3 removes continue's O(spawns)
  live-observer scan and capture's owner scan, the ~230 MiB primary-close residual.
  #591 was closed as its duplicate.
- [#612](https://github.com/haowjy/meridian-cli/issues/612): session teardown reads
  every spawn's `state.json` on every launch. Since #634 it is one streamed pass for
  all cleaned chats, so memory no longer grows, but the read count does. The fix is
  a chat-filtered index query plus authoritative re-reads, after #594.
- [#594](https://github.com/haowjy/meridian-cli/issues/594): no-change catch-up still
  takes the exclusive gate and re-reads every active spawn.
- [#600](https://github.com/haowjy/meridian-cli/issues/600): indexed reads materialize
  every row; missing order indexes; work listings load everything before `limit`.
- [#601](https://github.com/haowjy/meridian-cli/issues/601): `doctor` and lock GC scan
  everything.
- [#592](https://github.com/haowjy/meridian-cli/issues/592): a cold index build holds
  the catch-up gate for 2.4–5.7 s.
- [#595](https://github.com/haowjy/meridian-cli/issues/595) and
  [#527](https://github.com/haowjy/meridian-cli/issues/527): the history-independent
  floor (launch-bundle subprocess, background worker interpreter, cancel grace, CLI
  imports).
- **Untracked:** the automatic archiver folds the whole session log once per
  candidate. #418 leaves it out of scope, and no issue exists yet; see the
  [per-operation cost inventory](../architecture/per-operation-cost.md).
- [#700](https://github.com/haowjy/meridian-cli/issues/700): a background spawn
  composes its launch twice, once in the launcher and again in the worker; carry
  the prepared launch across instead.
- [#702](https://github.com/haowjy/meridian-cli/issues/702): `spawn wait` rewrites
  Pi wait-observation state on every poll, and does so even for parents that are
  not Pi.

### Should `--fork` and `--fork-fresh` merge? (#701)

**Status:** Open. [#701](https://github.com/haowjy/meridian-cli/issues/701): first
measure whether `--fork` actually keeps the prompt cache, then decide whether the
two flags merge. The owner chose to keep forking (2026-10-19). Merging or removing a
flag is a public change and goes through the
[deprecation window](../decisions/compatibility-versioning.md#d-compat-policy--one-minor-deprecation-new-builds-read-old-data-old-builds-refuse-new-data).

### D-003: Nested-Depth "After Reconciliation" Wording

**Where:** `meridian doctor` warning message for `live_active_spawns_remain`

**The issue:** When `is_root_side_effect_process()` is `False` (i.e., `_MERIDIAN_DEPTH > 0`), `doctor --kill-orphans` skips the reaper entirely, yet the `live_active_spawns_remain` warning still says "after orphan-kill reconciliation". A reader could infer that the reaper ran and left these spawns alive, when it wasn't attempted.

**The fix:** At nested depth, the warning should say the orphan kill was skipped.

**Why deferred:** Out of scope for the initial doctor bugfix. Small wording issue, no functional impact.

---

### D-005: Double Read in `_repair_orphan_runs` (stale framing)

**Where:** `src/meridian/lib/ops/diag.py`

**The issue:** `_repair_orphan_runs()` reads the spawn store to perform reconciliation, then `doctor_sync` re-reads to get the post-reconcile snapshot. In v2, each read is O(1) per spawn file rather than an O(n) JSONL replay, so the performance concern is diminished. The structural concern (two reads where one suffices) remains a minor cleanup target.

**Why deferred:** Minor refactor, no correctness issue. The v2 per-spawn reads are fast.

---

### v001 `uuid_state_split` Migration — Manual Helper, Not an Active Fallback

**Where:** `migrations/v001_uuid_state_split/`

**The issue:** Repo-local `.meridian/` is no longer an active runtime-state root.
`migrations/v001_uuid_state_split/migrate.py` contains a manual copy helper for moving
legacy runtime state to the user project root, but `migrations/registry.toml` still
marks v001 as `stub` and production startup does not discover or run this registry.

**Impact:** Legacy repo-local spawn/session files do not appear on current read paths
until copied into the user runtime root. Nothing reads `.meridian/id` any more, so a
directory holding only a legacy `.meridian/id` is an ad-hoc place
([D-two-tier-identity](../decisions/state.md#d-two-tier-identity)).

**What's needed:** Decide whether to validate and expose v001 as a supported manual
migration or remove the dormant registry path. If retained, align the helper with the
current project-ID model and atomic-write policy before marking it ready.

**Why deferred:** Migration tooling design was deprioritized relative to active state
work; there is no transparent legacy runtime fallback.

---

### CLI Subprocess Path: Resolve-Before-Persist Unification

**Where:** `ops/spawn/execute.py`

**The issue:** The streaming-serve path uses `SpawnApplicationService.prepare_spawn()`, which composes before creating the spawn row (resolve-before-persist, SEAM-1/2/3).

The CLI subprocess path (`ops/spawn/execute.py`) also composes before the row now (`build_create_payload()`), so it has no phantom-row risk, but it reserves the row with its own `lifecycle_service.start()` call instead of going through the shared policy seam. The background worker recomposes after the row exists.

Phase 8.6 removed `SpawnOperationServices` and inlined cancel/cancel-all root/service resolution, leaving this CLI subprocess path as the remaining spawn-pipeline unification target.

**What's needed:** Unify the CLI subprocess path to use `SpawnApplicationService.prepare_spawn()`, removing the direct `lifecycle_service.start()` call.

**Why deferred:** The current behavior is correct. Unification is architectural hygiene, and it is subsumed by the planned shared turn host (launch#1, phase 4 of the [cleanup program](../codebase/cleanup-program.md)).

---

## Streaming & Completion

### One Hold Rule Beyond Claude

**Where:** `src/meridian/lib/streaming/hold_state.py`, `src/meridian/lib/harness/`, Pi's completion profile

**The issue:** A close with no ask needs a harness that reports its own background work, and only Claude does.

- **OpenCode:** it marks `shell.created` and `shell.exited` as work, so the same events can feed a native task source. Then the ask goes there too.
- **Codex:** what the app server reports about background terminals is unknown. Probe it first. Until then a background shell is invisible, and each rearm costs one nudge per 270 s.
- **Pi:** its 270 s nudge asks more than once and overwrites the report each time. It should take the same ask at the idle limit, and it needs a `rearm` to stay.

**Why deferred:** Each one needs a harness probe, and all three work today with the ask.

---

### Store-Level Publication Ordering (#431)

**Where:** `src/meridian/lib/streaming/spawn_manager.py`, `src/meridian/lib/launch/streaming_runner.py`

**The issue:** Store-level `published_at` can trail drain cleanup rows. The probe-fix
cycle measured 56s between death and store publication on real Pi (p5390: cleanup
completed at 14:20:20.545Z, `published_at` at 14:20:21.810Z). The sub-second
`published_at` timestamp (added in #435) makes this ordering observable. Moving
durable publication ahead of cleanup is a separate store-pipeline concern.

**Remaining:**
- `streaming_runner.py` remains large; timeout state spans arbitration, drain
  normalization, failure projection, and lifecycle finalization. Extraction
  of turn-result normalization would prevent another ad-hoc boolean branch.

**Why deferred:** The probe-fix cycle resolved the structural conflation.
Store-level ordering is a pipeline concern tracked as #431 evidence.

---

### Wait Fail-Fast Reporting (#458)

**Where:** `src/meridian/lib/ops/spawn/api.py` — `spawn_wait_sync()`

**The issue:** `spawn wait` blocks until all pending spawns reach a terminal
state. When a denied or failed spawn finalizes quickly (e.g. after the
headless-deny fix in PR #460), the healthy spawns in the wait set continue
running. The caller sees the failure only after the entire wait set drains.
A fail-fast mode would report the first terminal failure immediately without
waiting for healthy spawns to complete.

**What's needed:** An eager/fail-fast option on `spawn wait` (and
`spawn_wait_sync`) that returns as soon as any spawn in the wait set reaches a
terminal failure status. Checkpoint mode already returns partial state, but it is
timer-driven, not event-driven. Fail-fast is conceptually distinct: it reacts to
a specific failure, not to a timer expiry.

**Why deferred:** Scoped out of PR #460 (#442 fix) as a separate concern.
The deny-at-prepare fix eliminates the most common trigger (denied spawns no
longer reach the wait set), reducing urgency. Filed as issue #458.

**Decision context:** `decisions-bugfix-sprint.md` in the workstream-roadmap work
directory.

---

### Typed Terminal Provenance (#438)

**Where:** `src/meridian/lib/streaming/pi_completion_profile.py`, `src/meridian/lib/harness/connections/pi_rpc.py`

**The issue:** `TerminalEventOutcome` loses provenance when
`error/connectionClosed` is normalized. The probe-fix cycle narrowly addressed
the Pi exit-classification defect by recognizing the canonical
`PI_SUBPROCESS_EXIT_ERROR_PREFIX` string in precedence logic. A typed terminal
origin (`pi_subprocess_exit`, transport failure, provider failure, timeout)
would make precedence explicit and delete string-prefix policy.

**Why deferred:** String-prefix recognition is correct and scoped. A typed
provenance taxonomy is a larger contract revision. Filed as #438.

---

### Cleanup Telemetry Categories (#439)

**Where:** `src/meridian/lib/streaming/pi_drain.py`, lifecycle phase events

**The issue:** `cleanup_running`/`cleanup_completed` lifecycle phases do not
distinguish connection teardown from tracked-work cleanup. The probe-fix root
cause analysis showed this ambiguity prevents over-reading cleanup telemetry.
A reason/category field on cleanup phases would make the distinction
observable.

**Why deferred:** Contract revision, not a correctness fix. Filed as #439.

---

---

## Routing & Model Resolution

### Agent Overlay Follow-Ups (2026-05-05, after agent-model-overrides shipped)

**Where:** overlays live in `mars.toml` / `mars.local.toml` and Mars resolves them
([D72](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface));
the remaining Meridian-side item is in `src/meridian/lib/launch/`.

1. **`timeout` is not an overlay field.** Mars's `[agents.<name>]` has no `timeout`, so a
   per-agent timeout comes only from CLI, env, profile or Meridian config. Adding it means a
   Mars overlay field plus Meridian reading it from the launch bundle into
   `execution_policy.timeout` (minutes).


---

### Known Gaps After MR1-MR6 (2026-05-02)

**Where:** `lib/launch/policies.py`, `lib/launch/resolve.py`, `lib/catalog/agent.py`, `cli/main.py`

These items were identified in the mars-capability-packaging final gate (alignment reviewer, p2857) as "partially delivered" — accepted as follow-up work, not blocking.

1. **`MERIDIAN_HARNESS` env var not a policy override (S1.2).** The spec describes `MERIDIAN_HARNESS` as a user-facing harness override. The spawn-local write has been renamed to the internal `_MERIDIAN_HARNESS` (PR #388), so the name is free, but nothing reads a plain `MERIDIAN_HARNESS` yet. Wiring it into `RuntimeOverrides.from_env()` is the remaining work. See [decisions.md](../decisions.md) D57.

2. **Layer-by-layer debug routing logging not implemented (S6.2/S6.3).** `--dry-run` output shows final provenance but not the full layer-by-layer resolution trace. Adding a debug log per layer would help diagnose unexpected routing behavior.

3. **Rich error messages for invalid model/harness not implemented (S6.2/S6.3).** Errors like "unknown harness" show a bare message. A richer format (showing what was requested, what was available, and which layer set the value) would reduce debugging time.

4. **Harness fallback informational log and error message not implemented (S9.5/S9.6).** When harness-availability fallback activates, there's no user-visible notification. The spec calls for an INFO log "falling back from X to Y" and a richer error if no fallback is found.

5. **List override fields in `model-policies` accepted but not functional (S3.3/S3.12).** `skills`, `tools`, `disallowed-tools`, `mcp-tools` in the `override:` block are parsed and emit a "not yet supported" warning. Implementing them requires threading list overrides through policy merge — a separate design effort.

6. **`--mode <value>` filter flag for agent listing not implemented (S8.6).** `meridian list` groups agents by mode but doesn't support filtering by mode via flag.

7. **Model routing logic spans multiple modules — extraction recommended.** The refactor reviewer (p2857 final gate) noted that model routing logic is spread across `policies.py`, `resolve.py`, `catalog/models.py`, and `catalog/model_aliases.py`. A dedicated routing module would improve locality and testability.

**Decision context:** [decisions.md](../decisions.md) — D50–D57

---

## Pi Result Delivery Limits {#pi-result-delivery-limits}

None of these limits is a reproduced defect or a must-do:

- **Unadmitted queue claim.** A watcher message queued for native steer or
  follow-up that Pi never admits (for example a cleared queue) keeps its IDs
  reserved, and because queueing closes the current opportunity it can also pause
  unrelated automatic publication until a real native lifecycle callback, reload or
  restart; timer and watcher wakeups do not reopen it. This predates boundary steer; confirm a real
  reproduction before designing a release path.
- **All-history scan per boundary.** Fresh canonical discovery at each tool
  boundary reads every project spawn row. Revisit only if boundary latency shows up
  in real parent/child workflows. Removal trigger: a measured boundary cost or a
  safe incremental index that does not become authority.
- **Cross-cutting efficiency audit.** Retained runner streams, doubled Bash log
  writes, per-drain history polling, cold CLI process cost and old projection
  namespaces are tracked in
  [issue #561](https://github.com/haowjy/meridian-cli/issues/561) alongside #414,
  #496, #500 and #473. Constraints carry over: an inventory is not proof of
  reclaimability, a foreign negative cache needs proof, and old-schema files remain
  mixed-version authority constraints. No production cleanup is authorized.
- **Foreground attachment ownership (managed-Bash).** Foreground model and human
  execution paths still duplicate settlement, abort/start-stop hooks and
  foreground-count UI bookkeeping. Consider one attachment owner only if it
  deletes that bookkeeping while preserving distinct model/human output APIs,
  physical stop versus publication, and reload/background handoff. Launch already
  separates the owned process from its creation observation, so this is only an
  optional structural cleanup, not needed for any correctness behavior.
- **Post-shell polling cost (managed-Bash).** Groups whose descendants outlive the
  shell are polled on a short cadence. #561 has only exploratory backoff
  measurements, not full-runtime or macOS qualification; qualify output drain,
  cancellation, ps failure, publication delay and ownership retirement before
  changing cadence, and never turn observed host lag into a delivery bound.
- **Large backlogs.** A backlog beyond one packet takes several genuine native
  opportunities, one message each; fair allocation fixed the avoidable per-result
  amplification, so only the unbounded-backlog case remains, with no
  constant-total-inference promise.

See the [delivery contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/pi_runtime/.context/delivery-contract.md).

## Ecosystem: Jupyter Workbench

### Structural Follow-Ups from Final Gate

**Where:** `jupyter-workbench:src/jupyter_workbench/core/`, `jupyter-workbench:src/jupyter_workbench/adapters/`, `microct-analysis:src/microct_analysis/`

These items were accepted as non-blocking after tech-lead spawn p3005 shipped the initial `jupyter-workbench` / `microct-analysis` implementation. They are structural cleanup candidates, not known user-facing failures.

1. **Session manifest policy duplication.** Execution, snapshot, and lineage services repeat session-id resolution and manifest reads. Future cleanup should extract a `SessionStore` / `SessionRepository` boundary instead of letting every service carry manifest policy.

2. **Raw dict DTO fields.** `visualization_delta`, event payloads, and some summary fields use raw mutable dictionaries. Future cleanup should introduce typed DTOs for scene summaries, cell summaries, and event records while preserving CLI/MCP field semantics.

3. **Notebook writes are not atomic.** The per-session mutation lock serializes writes, but `active.ipynb` updates do not yet use tmp+rename. Low risk in current single-session mutation flow; should be made crash-only before heavier concurrent use.

4. **`LineageService.replay()` carries execution mechanics.** Replay currently invokes execution mechanics directly. A `NotebookOutputSerializer` or similar collaborator would keep lineage policy separate from execution details.

5. **Replay does not reconstruct live visualization state.** This is by design: browser/trame state is live process state. Replay should leave visualization `absent`; agents re-execute scene setup cells after replay.

**Decision context:** [ecosystem/jupyter-workbench/architecture.md](../ecosystem/jupyter-workbench/architecture.md) and [ecosystem/jupyter-workbench/runtime-model.md](../ecosystem/jupyter-workbench/runtime-model.md)

---

## Ecosystem: MicroCT Analysis

### PCA Duplication Across Tool and Processing Layers

**Where:** `microct-analysis:src/microct_analysis/tools/orientation_tools.py`, `microct-analysis:src/microct_analysis/processing/orientation.py`

**The issue:** `orientation_tools.py` wraps `processing.orientation.pca_orient`, but both layers carry overlapping PCA logic. The tool layer was added to give agents a clean tool-API surface without importing directly from `processing/`. The duplication is structural, not a correctness issue — both implementations produce the same results.

**What's needed:** Define a single PCA implementation in `processing/`, with `orientation_tools.py` as a thin adapter. The adapter should own frame tracking (`CoordinateFrame`) while processing owns the numerical algorithm.

**Why deferred:** Both work correctly. Unification is structural hygiene. Revisit when a third PCA caller appears in the codebase.

---

### Slice Inspection Shared Primitive Extraction

**Where:** `microct-analysis:src/microct_analysis/tools/slice_renderer.py`, `src/microct_analysis/tools/slice_inspector.py`, `src/microct_analysis/tools/_plane_helpers.py`

**The issue:** `slice_renderer.py` and `slice_inspector.py` share slice-extraction logic through `_plane_helpers.py`, but the two functions differ enough that further abstraction risks over-engineering. Currently `_plane_helpers.py` is an internal module; it's not a stable public API.

**What's needed:** If a third consumer of plane-extraction logic appears, consolidate into a proper `plane_primitives` module with a defined interface.

**Why deferred:** Two consumers don't clearly establish the seam. Premature extraction would create an abstraction for its own sake.

---

### Prompt-Level Executable Tests for `slice-examination-loop`

**Where:** `microct-analysis:skills/slice-examination-loop/SKILL.md`

**The issue:** The `slice-examination-loop` skill has documented behavior contracts but no prompt-level tests that run the agent against a real session. Prompt testing for skill behavior requires an actual jupyter-workbench kernel session with real scan data — a workbench-integrated test harness that doesn't exist yet.

**What's needed:** A prompt testing framework that can launch a workbench session, load a scan, and drive the skill through its paces. The `meridian-prompter` test toolchain covers stateless skills; this one requires stateful kernel context.

**Why deferred:** Infrastructure doesn't exist. Tracked as a capability gap in the prompt testing toolchain.

---

## Cross-References

- [open-questions/mars-feature-gaps.md](mars-feature-gaps.md) — Mars package manager gaps
- [open-questions/process-scope.md](process-scope.md) — process-scope cleanup follow-ups (PROC-004, PROC-007)
- [lessons/state-design-lessons.md](../lessons/state-design-lessons.md) — why dual-root was introduced (v001 context)
- [architecture/state-system/overview.md](../architecture/state-system/overview.md) — migration framework design
- [architecture/process-scope.md](../architecture/process-scope.md) — process-scope ownership design
- [operations/health-checks.md](../operations/health-checks.md) — doctor flow (D-003, D-005 context)
- [decisions.md](../decisions.md) — D27, D29
