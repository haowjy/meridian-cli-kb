# Per-Operation Cost: the Constant-Time Bar

**State as of 2026-10-15, `main` at `ea8e37e58` (v0.10.10 plus #608).** Status
and memory findings updated 2026-10-16 (`main` at `0a6b3acb4`, v0.10.16: #634
and #651 shipped).

## The bar

The owner set it after #588 merged: "shouldn't launch and close and wait etc be all
practically constant time?" (`chat:c7736`, D17). Launch, finalize/close, wait,
show, cancel and continue each concern one spawn and one chat. **Their cost must not
grow with how much history the project has**: not with `sessions.jsonl` bytes, not
with the number of spawn directories, and not with the number of chats.

In practice a per-spawn operation reads:
- its own run rows (`spawns/<id>/state.json`);
- its own chat's session state;
- at most a bounded candidate set found by a keyed lookup, each candidate re-read
  from authority.

Whole-project scans belong to explicit global commands (`spawn list`, `doctor`,
`session archive`, `session search`, `session index rebuild`) and to interval-gated
maintenance. They must not run on a per-spawn path. Users also run the global
commands often, so those should be bounded by their result size where possible
(#600, #601).

Cost has two separate parts:
- the **slope**: work proportional to history, which the bar forbids;
- the **floor**: a fixed cost per command (interpreter startup, subprocesses,
  teardown). It doesn't grow, but it is still large: a no-op foreground launch
  took about 3.6 s even with a 1 MB log (#595, #527).

Fix the slope first. A fixed floor is annoying, but a slope makes busy projects
degrade until they fail: #586 was a slope (a full-log fold under a lock) that
surfaced as daily `spawn wait` timeouts.

## Fixed so far

| PR | Release | What became constant | Measured |
|---|---|---|---|
| #588 | v0.10.4 | The session append guard is O(1) (the `historical-` generation prefix), and history-index readers never wait on session writers | `sessions.jsonl.flock` hold per append: 1,463 ms → about 16 ms at 46.6 MB; reader P95 under 2 writers 2,001 → 101 ms ([decision](../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers)) |
| #599 | v0.10.8 | No-arg `spawn wait` discovery filters by owner and descendant in SQL, then re-reads only those rows from authority | 672 → 95 ms warm at 3,951 indexed rows ([spawn wait](../concepts/spawn-wait-barrier.md)) |
| #605 | v0.10.10 | Finalize watchdogs wake on the completion event instead of on their next 1 s poll | exit → finalized 1,610 → 460 ms on the tiny corpus; this is floor, not slope ([finalization](spawn-finalization.md#terminal-arbitration-terminalarbitrator)) |
| #634 (MEM-1) | v0.10.14 | Startup repair streams `state.json`; single-chat session reads decode only that chat's rows; scoped `SessionBindings`. **Memory only**: repair still reads every spawn row, and filtered reads still scan every byte of `sessions.jsonl` | repair RSS 226 → 101 MB, flat with spawn count (4× spawns: 309 → 101 MB); runner RSS while the harness runs 224 → 96 MB, the empty-runtime floor ([§5](#5-resident-memory-full-scans-kept-as-pydantic-objects-627)) |
| #651 (#418 B1) | v0.10.16 | One session read/append seam; every multi-chat read is one pass; native scope is two passes regardless of alias count. Same caveat: passes are filtered, not removed | primary close 6.3 → 1.7 s (mellow), 14.4 → 5.3 s (orange); `resolve_session_ref` 0.88 → 0.17 s; runner peak RSS 329 → 103 MiB, close 678 → 335 MiB ([§1](#1-session-state-is-one-global-replay-log-418), [§5](#5-resident-memory-full-scans-kept-as-pydantic-objects-627)) |

## Known violators on `main`

Grouped by root cause. Numbers come from the probes listed under
[Provenance](#provenance). "Tiny / mellow / orange" are copies of real runtimes
with 1.05 / 18.9 / 46.6 MB session logs. "240 / 2,400 / 12,000" are synthetic
spawn-directory counts.

### 1. Session state is one global replay log (#418)

Every chat appends to one `sessions.jsonl`, and most session reads replay all of
it ([session state](state-system/session-state.md#sessionsjsonl-is-one-global-log)).
A real foreground no-op launch with a fake harness took **3.59 / 6.02 / 9.12 s**
at tiny / mellow / orange, with **9 full-log reads** per launch (12 at tiny). At
orange that was 3.87 s of scanning and 3.13 s holding `sessions.jsonl.flock`. The
largest callers were `session_bindings` (two reads, 1.12 s), `get_session_active_work_id`
(0.60 s), `conclude_native_run` (0.58 s) and the `_session_execution_context`
re-read (0.46 s). Explicit-ID wait, show and cancel also replay the log, 1–11 times
per command depending on the operation and corpus.

The accepted fix is per-chat journals,
[D-session-per-chat-journals](../decisions/state.md#d-session-per-chat-journals).
#591 was closed as a duplicate of #418.

**After #634 and #651 (v0.10.16).** Every per-operation session read now goes
through one keyed seam ([contract](../decisions/state.md#d-session-read-seam)).
A read selects its chats with a byte prefilter and decodes only their rows, and a
multi-chat read is one pass. Unfiltered folds per primary close went 7 → 0 and per
runner 1 → 0. **The slope in bytes is still there**: every filtered pass reads the
whole file, and a primary close makes 10 of them. What shrank is the decode and the
resident memory, plus the wall time that went with them (table above). The #418
gate of 0 full-log reads per launch, close, continue or wait is B2's, when each
chat's rows live in their own journal. Bulk reads still outside the seam:
`list_all_session_records`, `list_session_generations`, and
`legacy_native_import.py`'s raw reads.

### 2. Launch and continue read every spawn's `state.json` (#612, #418 B3)

- **Every launch, at session teardown (#612).** `session_scope` exit calls
  `reclaim_session_owned_scopes_for_chat`, which calls `spawn_store.list_spawns`
  and filters by chat *after* reading every row. A foreground launch read
  **272 / 2,432 / 12,032** `state.json` files at 240 / 2,400 / 12,000 spawn
  directories. That is one read per existing spawn plus about 32 for the target
  run. The background worker hits the same path when its run exits. The cleanup
  must still reclaim every session-owned scope of *every* run on the chat, not just
  the current one. A test pins that, so "current spawn only" is wrong.
  Since #634 the pass is `reclaim_session_owned_scopes_for_chats`: one streamed
  pass for every cleaned chat, keeping only the matching rows. Memory no longer
  grows with spawn count and N stale chats cost one pass, not N (23 chats had taken
  6.2–6.6 s). It still opens every `state.json`, so #612 stays open.
- **Continue into an existing chat (#418 B3).** `start_session` calls
  `live_observer_for_chat`, which also lists every spawn. That makes a second
  O(spawns) pass per continue: 518 reads at 240 directories. `native_run._warn_overlap`
  has the same scan. #418 B3 replaces both scans with observer rows kept in the chat.

The planned fix for #612 is a chat-filtered candidate query against the history
index, then an authoritative re-read of each candidate. It waits for #594. Without
#594 the index query's catch-up can itself re-read every active row, so the O(spawns)
work would just move into the index (`spawn:p8095`).

### 3. History-index reads pay for more than their answer (#594, #600, #592)

- **#594:** every reader takes the exclusive catch-up gate, even when nothing
  changed, and `_reread_unmarked` re-reads every active indexed spawn. With 16
  concurrent readers, a no-change catch-up took 400 ms median and 775 ms P95
  (739 rows). `_reread_unmarked` exists for mixed-version safety: an older
  build's writes don't mark this schema's queue. It can only be deleted once a
  durable per-source freshness witness replaces it.
- **#600:** indexed reads materialize every row and filter in Python.
  `spawn list`, `stats`, `children`, `@latest` and the reaper's scan took about 2 s
  at 12,000 spawns. Work listings load everything before applying `limit` (archived
  work list, `limit=10`: 647 ms). Two order indexes are missing. Adding them sped up
  the planner POCs: `records(started DESC, local_id DESC)` 4.47 → 2.41 ms and
  `session_chats(ordinal DESC)` 1.63 → 0.014 ms.
- **#592:** a cold index build holds the catch-up gate for 2.4–5.7 s. It mostly
  happens at upgrade or after an index bump (v12 in #714; next in identity-v2
  PR 4 and #418 B2).
- **Deadline rule (#656, 2026-10-16).** A caller with a deadline must build the
  index *before* it starts its clock. `query()` keeps an absolute caller deadline
  across a cold build, so the build otherwise uses up the 2 s. Corpus search and
  descendant evidence follow the rule
  ([history index](state-system/portable-history.md#a-caller-with-a-deadline-builds-the-index-before-starting-its-clock)).
  A cold build now costs time, but it no longer produces a wrong answer. #592,
  the gate held for the whole build, stays open.

### 4. Maintenance scans (#601, no issue for the archiver)

- **#601:** `doctor` (unpruned) took 6.23 s at 12,000 spawns, and lock GC 402 ms.
  They are global commands, but users run them interactively.
- **Automatic archive:** one full session-log fold per candidate: 612 folds and 4.7 s
  warm on mellow, more than 20 s cold. It runs on primary stop, gated by an
  interval. #418 deliberately leaves it out (design question 10). **No issue tracks
  it yet.**

### 5. Resident memory: full scans kept as Pydantic objects (#627)

The same full scans also set resident memory (`work:memory-audit` M1, M6;
`spawn:p8175`, `spawn:p8256`). They are materialized as Pydantic objects, and after
they are freed CPython keeps the memory arenas, so RSS stays at the scan's peak for
the life of the process. It isn't SQLite, threads, polling or a leak. Measured in the
owner's project shape (4,226 spawns, 54.5 MB `sessions.jsonl`):
- `list_spawns` over 4,226 rows: 444 MB;
- one full `sessions.jsonl` replay (67,400 lines): 525 MB;
- the native-capture guard (a spawn scan plus two session replays): 537 MB still
  resident after it returned;
- startup repair: 373 MB after it finished;
- a `spawn wait` on explicit ids: flat at 94 MB for 65 s.

**Background runners.** On a read-only copy of real state (1,022 spawns, 37k-line
log), a `reply OK` runner peaked at 321–325 MB, against 101 MB on an empty runtime.
The runner-side jump is `_prepare_execution_handoff` → `get_session_active_work_id`,
which replays every session record to read one chat's work id. Before the fork,
first-write legacy import replays the session log twice and calls an unfiltered
`list_spawns` through telemetry retention. Unbounded Codex/OpenCode event queues
were suspected but not reproduced: instrumented queues never passed depth 4 or 1.8 KB.

Import cost is separate: `meridian --help` is 16 MB, `import execute_bg` 92 MB.

**The fix is to stop materializing, not to free memory afterwards.** Because the
arenas stay at their peak, a scan that builds every row as an object costs its
peak for the rest of the process, even if the caller keeps one row. Both shipped
fixes read the same authority and keep only what they return:

| Workload (copy of real state) | Before | After | Fixed by |
|---|---:|---:|---|
| Startup repair, no orphan primary (1,027 spawns, 25 MB log) | 226 MB, 9.8–10.3 s | 101 MB, 3.5–3.6 s | #634: `iter_repair_candidates` streams `state.json` and yields only active rows and rows with a cleanup claim |
| Startup repair at 4× spawns (4,096 rows) | 309 MB | 101 MB | #634 |
| Startup repair with real orphan-primary capture | 363 MB | 308–311 MB | not fixed: the residual is the capture guard's owner scan |
| `execute_bg` runner, RSS while the harness runs | 224 MB | 96 MB | #634: single-chat `get_session_active_work_id` and `get_session_harness_id`, and a scoped `SessionBindings` |
| `execute_bg` runner peak, largest runtime | 329 MiB | 103 MiB | #651: `record_model_selection`'s cross-chat dedupe reads native scope in two passes |
| Primary close peak, largest runtime | 678 MiB | 335 MiB | #651 |

Converting `get_session_active_work_id` alone did not move the runner:
`SessionBindings.__init__` replayed the whole log 0.5 s later and refilled the same
arenas. Remove every full replay on a path before you measure; one survivor sets the
peak.

**Left:**
- **The close residual, about 230 MiB.** Capture's spawn-side owner check still
  reads every spawn row. That O(spawns) scan was permitted in B1; #418 B3 (observer
  rows in the chat) or identity-v2's derived reads remove it. Close is not yet at
  the roughly 100 MiB import floor.
- **A cold history index.** A runner on a cold index reached 376 MiB. Cold builds
  are outside the bounded-read claim (#592).
- The rarer bind branch of legacy import, `list_all_session_records`, index
  catch-up batching, `list_active_session_records` and `get_last_session`. None is
  measured on a hot path yet.

### 6. The fixed floor (#595, #527)

These don't grow with history, but they are the next-largest cost per launch:
- about 0.8 s of CLI imports and Pydantic model construction per command (#527);
- `mars build launch-bundle`, about 497 ms per launch;
- the detached background worker's second interpreter, 630 ms of imports. That
  interpreter is the crash-only detachment boundary, so removing it needs an
  architecture change;
- cancel of an unresponsive harness: about 11.5 s (5 s terminate grace + 5 s
  canceller wait + 1 s service wait).

### Checked and already flat

- The session append after #588: about 16 ms at every log size.
- Per-spawn terminal writes: one `state.json` under its own lock (22–38 ms holds).
- Exact-ID `spawn show`, `cancel` and `report`, and explicit-ID wait polls, apart
  from the session-log reads in section 1.
- Spawn and chat ID allocation through their counters. Only a missing counter
  triggers a one-time directory scan.
- `context` resolution and hook dispatch.

## In flight

- **Spawn identity: identity-v2**
  ([D-spawn-identity-v2](../decisions/spawn-identity.md#d-spawn-identity-v2)),
  seven PRs; PRs 1–2 merged (#633, #647). Its acceptance bar: continue, show, cancel,
  wait and close have equal counts and equal `state.json` bytes across 240 /
  2,400 / 12,000 spawns × 1 / 10 / 50 / 500 launches. `spawns_flock` is held only
  for allocation, publication and a bounded re-parent write. Background repair is
  capped at 64 lock attempts and 32 recoveries per pass, replacing the reaper over
  every row on each primary launch. Continue's live-observer scan stays O(spawns)
  and is measured as its own exempt line until #418 B3.
  - The bound to carry forward comes from #616, closed unmerged on 2026-10-16:
    direct continue was flat across 240 / 2,400 / 12,000 spawns × 1 / 10 / 50 runs
    (0 scans, 5 state reads, 1 keyed lookup, 1 `spawns_flock` hold).
  - The safety regressions its cost cuts introduced are recorded as
    [lessons](../lessons/cost-cuts-and-safety-checks.md).
- **#418 B1 → B2 → B3 (#418 open):** B1 shipped as #651 (v0.10.16). B2 (the
  journals, cutover, absorb, index v12) needs the owner's answers to design
  questions 1–4 and 6–11, and comes after identity-v2 PR 4 (index v11). B3 removes
  the O(spawns) observer and capture-owner scans, and with them the ~230 MiB close
  residual.

## How to measure

Measure **counts, not wall time**. A shared development box ran at load 15–60
during this work. That invalidated whole batches of wall times, while counts were
unaffected ([lesson](../lessons/contention-diagnosis.md#count-work-dont-time-it-on-a-shared-box)).

The harness lives in `work:v010-followups` under `launch-e2e-probe/`:
- **Real CLI, fake harness.** A `claude` shim answers `--version`, then emits
  `system/init` and a successful `result`. No model or network call is made.
- **Isolated roots.** Every sample uses fresh `HOME`, `MERIDIAN_HOME`,
  `_MERIDIAN_RUNTIME_DIR` and project directory, with a copied corpus. The real
  `~/.meridian` is never touched.
- **`sitecustomize.py` counters.** On `PYTHONPATH`, they wrap:
  - `scandir`/`listdir`/`Path` scans of the spawn directory;
  - `state.json` opens;
  - full `sessions.jsonl` reads (`read_events`, `iter_jsonl`, `iter_jsonl_file`
    at offset 0);
  - lock acquisitions and summed hold time per lock file.

  Stack capture per event names the caller. That is how #612 was attributed to
  session teardown and not to the reaper or the index.
- **Count every pass, filtered or not, per operation.** A gate that counted only
  unfiltered folds passed while B1's first draft looped a filtered pass per chat
  ([lesson](../lessons/contention-diagnosis.md#count-every-pass-per-operation-not-one-kind-of-fold)).
  `tests/support/session_lookup_measurement.py` (with `io_probe.py`) counts physical
  `iter_jsonl_file` passes and lock holds on built wheels; B2's gate reuses it.
- **Count one logical read once.** One keyed fold passes through three nested
  instrumented seams, so it shows up as three events. Report logical reads, or
  state the seam multiplicity. An uncorrected probe reported 30–39 "full-log
  reads" where the true number was 9.
- **Warm the index on the measured root first**: one catch-up, then check that
  coverage is complete. A copied corpus starts cold. Cold-index catch-up made no-arg
  wait return discovery-pending 3 of 3 times, against 0 of 8 warm.
- **Scale axes.** Use the three session-log corpora for slope in history bytes, and
  240 / 2,400 / 12,000 spawn directories for slope in spawn count. Add 1 / 10 / 50
  runs per spawn for anything that resolves runs.
- **Acceptance.** Counts are equal across sizes for per-spawn operations. #418's
  program gate adds:
  - 0 full-log reads per launch, close, continue or wait;
  - a foreground no-op launch spread of at most 10% or 150 ms;
  - lock holds whose orange P95 minus tiny P95 is at most 10 ms.

  Record `uptime` with every batch, and redo any wall-time batch taken at more than
  about 2× the quiet load (about 5 on this box).

## Provenance

`work:v010-followups` `decisions.md` D17–D31:
- `constant-time-audit.md` (`spawn:p7993`);
- `launch-e2e-probe.md` (`spawn:p7997`);
- `startup-audit.md` (`spawn:p8007`);
- `scale-audit.md` (`spawn:p8006`);
- `probe-587-590.md` (`spawn:p8044`);
- `combined-results.md` (`spawn:p8073`);
- `investigate-launch-scan.md` (`spawn:p8095`);
- `watchdog-finalize-report.md`.

Also `chat:c7736`. Memory: `work:memory-audit` `decisions.md` M1, M6, M8–M24
(`evidence/memory-audit.md`, `spawn:p8175`; runner probe `mem3-discriminate.md`,
`spawn:p8256`; MEM-1 `mem1-startup-repair.md`, `spawn:p8261`/`p8270`; B1
`b1-one-chat.md`, `spawn:p8274`/`p8298`/`p8299`/`p8302`).

## Related

- [Session state](state-system/session-state.md): the global log and its readers.
- [Spawn wait barrier](../concepts/spawn-wait-barrier.md): the bounded discovery and
  wait-set semantics that make the bar user-visible.
- [Spawn finalization](spawn-finalization.md): terminal arbitration and the
  watchdog floor.
- [Contention and cost diagnosis](../lessons/contention-diagnosis.md): how these
  costs were found, and the measurement traps.
- [Cost cuts that weakened safety checks](../lessons/cost-cuts-and-safety-checks.md)
- [Spawn state v2](../decisions/state.md#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05):
  the precedent for moving a global log to per-entity files.
- [Future work](../open-questions/future-work.md#per-operation-cost-program)
