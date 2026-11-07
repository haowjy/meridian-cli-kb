# Future work and known limitations

This is an inventory of limitations and follow-ups, organized by domain. Follow the issue links for current status and discussion; entries without a link are untracked and should be verified before work starts. Inclusion does not imply priority.

---

## Managed Primary

### Primary message routes: session moves and remaining edges (#916)

**Where:** the Codex and OpenCode primary route, and the Pi `meridian-message` extension.

**State:** every primary harness has a route
([primary delivery](../decisions/primary-inject.md#d-unified-delivery)). What remains is
in [#916](https://github.com/haowjy/meridian-cli/issues/916):

- **Following the TUI.** A sender is refused after a session move where a Claude or Pi primary
  would take the message. Codex could follow `/new` and `/fork`; `/resume` to a loaded
  thread gives no signal, OpenCode gives none until the human types in the new or other
  session, and `harness_session_id`, the idle sensor and
  `--continue` would have to move too
  ([session moves](../decisions/primary-inject.md#d-primary-session-move));
  following the TUI is not planned.
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

- [#418](https://github.com/haowjy/meridian-cli/issues/418): the global session log
  makes session operations grow with project history. Per-chat journals need design
  decisions before implementation: whether to use event journals; absorb older-build
  writes at command entry; how to sequence index revisions and migration; drop the
  cross-chat archive witness check; fail on a read-only runtime; allow corpus search
  to migrate other projects; exclude pruning and archive scans; preserve the accepted
  generation when absorbing an older start; and declare older builds' reads of a
  migrated runtime unsupported. Automatic retirement was approved as a rename after
  14 days without legacy writes, never deletion. Reassess sequencing against current
  code. Continue also scans live
  observers and capture scans owners; the measured primary-close residual was about
  230 MiB. See [per-operation cost](../architecture/per-operation-cost.md).
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
- [#615](https://github.com/haowjy/meridian-cli/issues/615): the automatic archiver folds the whole session log once per
  candidate, increasing work with history; see the
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
two flags merge. Merging or removing a
flag is a public change and goes through the
[deprecation window](../decisions/compatibility-versioning.md#d-compat-policy--one-minor-deprecation-new-builds-read-old-data-old-builds-refuse-new-data).

### Nested doctor warning

At nested depth, `doctor --kill-orphans` skips the reaper but the
`live_active_spawns_remain` warning says “after orphan-kill reconciliation”. This
can imply a kill was attempted. Untracked.

### Legacy runtime-state migration

Legacy repo-local spawn and session files are not on current read paths. The
`migrations/v001_uuid_state_split/` helper can copy them, but the registry marks it
as a stub and startup does not run it. Decide whether to support and validate that
manual migration or remove the dormant path. Untracked.

### Spawn preparation paths

The CLI subprocess path in `ops/spawn/execute.py` prepares the launch and reserves
its row separately from `SpawnApplicationService.prepare_spawn()`, used by streaming
serve. The background worker composes the launch again. Separate paths duplicate
policy and can drift; [#700](https://github.com/haowjy/meridian-cli/issues/700)
tracks the repeated composition. The [cleanup program](../codebase/cleanup-program.md)
tracks broader consolidation.

## Streaming & Completion

### Background work and completion

A resident child can finish a turn while background work remains. The hold protocol
keeps a candidate result until visible work clears; where a harness cannot report its
work, Meridian asks the child at expiry before closing. The child can rearm to keep
waiting; a nudge is the message that asks it to report or continue. Native visibility
of background terminals differs by harness and needs probing before removing that
confirmation. Untracked. See the [completion contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/streaming/.context/completion-and-entry-points.md).

### Test `_sessions` seeding

`tests/integration/launch/test_spawn_manager_lifecycle.py` seeds the manager's
private `_sessions` field, coupling the test to the manager's implementation. A manager-owned setup path could remove that coupling. Untracked.

### Typed Terminal Provenance (#438)

**Where:** `src/meridian/lib/streaming/pi_completion_profile.py`, `src/meridian/lib/harness/connections/pi_rpc.py`

**The issue:** `TerminalEventOutcome` loses provenance when
`error/connectionClosed` is normalized. Precedence currently recognizes the canonical
`PI_SUBPROCESS_EXIT_ERROR_PREFIX` string in precedence logic. A typed terminal
origin (`pi_subprocess_exit`, transport failure, provider failure, timeout)
would make precedence explicit and delete string-prefix policy.

A typed provenance taxonomy is a larger contract revision. Tracked in [#438](https://github.com/haowjy/meridian-cli/issues/438).

---

### Cleanup Telemetry Categories (#439)

**Where:** `src/meridian/lib/streaming/pi_drain.py`, lifecycle phase events

**The issue:** `cleanup_running`/`cleanup_completed` lifecycle phases do not
distinguish connection teardown from tracked-work cleanup. The ambiguity prevents a reader from distinguishing those causes in telemetry.
A reason/category field on cleanup phases would make the distinction
observable.

Tracked in [#439](https://github.com/haowjy/meridian-cli/issues/439).

---

---

## Routing & Model Resolution

### Agent timeout overlays

Mars `[agents.<name>]` has no `timeout` field, so an agent-specific timeout
cannot be set there. Supporting it also requires Meridian to read it into launch
execution policy. Untracked. See [agent overlays](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface).

### Routing diagnostics and overrides

- Plain `MERIDIAN_HARNESS` is not read as a harness override, limiting environment-based routing. Untracked.
- Dry-run shows final provenance but not every resolution layer; unexpected routing is harder to trace. Untracked.
- Invalid model or harness errors can lack the available choices and source layer. Untracked.
- `meridian list` does not offer a mode filter, so users must scan grouped results. Untracked.

Verify these gaps against the current implementation before changing routing.

## Pi Result Delivery Limits {#pi-result-delivery-limits}

These limitations are tracked in [#561](https://github.com/haowjy/meridian-cli/issues/561) where noted; the other items are untracked:

- **Unadmitted queue claim.** A watcher message queued for native steer or
  follow-up that Pi never admits (for example a cleared queue) keeps its IDs
  reserved, and because queueing closes the current opportunity it can also pause
  unrelated automatic publication until a real native lifecycle callback, reload or
  restart; timer and watcher wakeups do not reopen it. Confirm a real reproduction before designing a release path.
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
  mixed-version authority constraints. Establish that data is reclaimable and preserve mixed-version authority before cleanup.
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
  opportunities, one message each; there is no constant-total-inference guarantee for an unbounded backlog.

See the [delivery contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/pi_runtime/.context/delivery-contract.md).

## Telemetry and code structure

Telemetry lifecycle observers and bootstrap cross process boundaries, and event reads
and tails duplicate truncation-tolerant JSONL parsing. These make changes harder to
keep consistent. See [telemetry architecture](../architecture/telemetry/overview.md).
Untracked.

Session-reference handling and bootstrap carriers overlap; `CodexConnection` also
spans several concerns. These increase the surface of changes. Before collapsing
`session_target.py`, establish the state read and repair contract. Untracked.

## Ecosystem: Jupyter Workbench

### Jupyter Workbench limitations

**Where:** `jupyter-workbench:src/jupyter_workbench/core/`, `jupyter-workbench:src/jupyter_workbench/adapters/`, `microct-analysis:src/microct_analysis/`

These follow-ups are untracked; verify them against the current Jupyter Workbench source before acting.

1. **Session manifest policy duplication.** Execution, snapshot, and lineage services repeat session-id resolution and manifest reads. Future cleanup should extract a `SessionStore` / `SessionRepository` boundary instead of letting every service carry manifest policy.

2. **Raw dict DTO fields.** `visualization_delta`, event payloads, and some summary fields use raw mutable dictionaries. Future cleanup should introduce typed DTOs for scene summaries, cell summaries, and event records while preserving CLI/MCP field semantics.

3. **Notebook writes are not atomic.** The per-session mutation lock serializes writes, but `active.ipynb` updates do not yet use tmp+rename.; should be made crash-only before heavier concurrent use.

4. **`LineageService.replay()` carries execution mechanics.** Replay currently invokes execution mechanics directly. A `NotebookOutputSerializer` or similar collaborator would keep lineage policy separate from execution details.

5. **Replay does not reconstruct live visualization state.**: browser/trame state is live process state. Replay should leave visualization `absent`; agents re-execute scene setup cells after replay.

**Decision context:** [ecosystem/jupyter-workbench/architecture.md](../ecosystem/jupyter-workbench/architecture.md) and [ecosystem/jupyter-workbench/runtime-model.md](../ecosystem/jupyter-workbench/runtime-model.md)

---

## Ecosystem: MicroCT Analysis

### PCA Duplication Across Tool and Processing Layers

**Where:** `microct-analysis:src/microct_analysis/tools/orientation_tools.py`, `microct-analysis:src/microct_analysis/processing/orientation.py`

**The issue:** `orientation_tools.py` wraps `processing.orientation.pca_orient`, but both layers carry overlapping PCA logic. The tool layer was added to give agents a clean tool-API surface without importing directly from `processing/`. Overlapping algorithms can drift; their equivalence has not been established.

**What's needed:** Define a single PCA implementation in `processing/`, with `orientation_tools.py` as a thin adapter. The adapter should own frame tracking (`CoordinateFrame`) while processing owns the numerical algorithm.

**Tracking:** Untracked; verify the duplication against current code before acting.

---

### Slice Inspection Shared Primitive Extraction

**Where:** `microct-analysis:src/microct_analysis/tools/slice_renderer.py`, `src/microct_analysis/tools/slice_inspector.py`, `src/microct_analysis/tools/_plane_helpers.py`

**The issue:** `slice_renderer.py` and `slice_inspector.py` share slice-extraction logic through `_plane_helpers.py`, but the two functions differ enough that further abstraction risks over-engineering. Currently `_plane_helpers.py` is an internal module; it's not a stable public API.

**What's needed:** If a third consumer of plane-extraction logic appears, consolidate into a proper `plane_primitives` module with a defined interface.

**Tracking:** Untracked; two consumers do not yet establish a stable public interface.

---

### Prompt-Level Executable Tests for `slice-examination-loop`

**Where:** `microct-analysis:skills/slice-examination-loop/SKILL.md`

**The issue:** The `slice-examination-loop` skill has documented behavior contracts but no prompt-level tests that run the agent against a real session. Prompt testing for skill behavior requires an actual jupyter-workbench kernel session with real scan data — a workbench-integrated test harness that doesn't exist yet.

**What's needed:** A prompt testing framework that can launch a workbench session, load a scan, and drive the skill through its paces. The `meridian-prompter` test toolchain covers stateless skills; this one requires stateful kernel context.

**Tracking:** Untracked; the required stateful test infrastructure does not exist.

---

## Cross-References

- [open-questions/mars-feature-gaps.md](mars-feature-gaps.md) — Mars package manager gaps
- [open-questions/process-scope.md](process-scope.md) — process-scope cleanup follow-ups
- [lessons/state-design-lessons.md](../lessons/state-design-lessons.md) — why dual-root was introduced (v001 context)
- [architecture/state-system/overview.md](../architecture/state-system/overview.md) — migration framework design
- [architecture/process-scope.md](../architecture/process-scope.md) — process-scope ownership design
- [operations/health-checks.md](../operations/health-checks.md) — doctor flow
