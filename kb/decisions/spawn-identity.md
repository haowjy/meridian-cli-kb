# Decisions: Spawn Identity and Launch Containment

How a spawn is named across launches, who decides its status, and what keeps a
harness from outliving its runner. The superseded run model these replace is
[D-spawn-run-identity](#d-spawn-run-identity).

---

## D-spawn-identity-v2: one ID per spawn; status derived from a runner-held lock (2026-10) {#d-spawn-identity-v2}

**Status:** approved by the owner on 2026-10-16 (`chat:c8003`: "I approve the
design, but make sure u understand it and can explain it back to me again").
The approved design is revision 3.1 (`work:v010-followups`
`design-identity-v2/design.md`, `pr-plan.md`; decisions D45–D53, D55). It is
being built as seven PRs off `main`, each merged with its own approval before
the next starts. PRs 1–3 are merged: the start gate and guardian (#633), the
launch lock and containment sweep (#647), and `read_derived` (#672). **On `main`
today:** every runner holds an exact `launch.lock`; a continue sweeps or refuses
its source before any harness starts
([PR 2 as built](#pr-2-as-built-the-launch-lock-and-the-sweep)); and every
reader derives status from that lock, while dead-launch outcomes are written
only under it
([PR 3 as built](#pr-3-as-built-read_derived-and-one-dead-launch-finalize)).
But every launch, `--continue` included, still mints a new `pN`; status is still
stored in `state.json`, written by the runner, by cancel (outside the lock) and,
for dead launches, by `finalize_dead_launch`; and cancel still targets signals
with the 30 s PID-birth slack.

**Supersedes:**
- [D-spawn-run-identity](#d-spawn-run-identity), #616's `pN`/`pN.K` run
  model. #616 was closed unmerged on 2026-10-16, and #617–#620, filed only
  because of its second identity, were closed as moot.
- [D-launcher-watchdog](primary-terminal-ownership.md#d-launcher-watchdog--one-watchdog-per-launcher-instead-of-pdeathsig),
  the mechanism half, through [D-start-gate-guardian](#d-start-gate-guardian).
- The reaper as a status writer, heartbeat liveness, the 30 s PID-birth slack,
  the startup and post-exit graces, and the `mark_finalizing` CAS. PR 3 deleted
  the liveness guesses and the heartbeat
  ([derived status](../architecture/state-system/derived-status.md#deleted-mechanisms-do-not-bring-back));
  the old rules survive only as history in
  [state decisions](state.md#mark_finalizing-cas-enables-orphan_finalization-distinction).
  PR 4 deletes stored status, the CAS and the reaper's status writes.

### The owner's questions

#616 was ready to merge when the owner stopped it (`chat:c7736`,
2026-10-16 UTC):

- "runID should be a different id from spawn id... right? why is it spawn id.N?"
- "do we even need a runID?"
- "don't they just have a single session id?"
- "why do we record whether a launch is running, finished, or failed?"
- "it would be better to be 'simpler' for less machinery … why can't meridian
  hold the lock"

The last question is the one that unlocked the redesign. Per-launch rows existed
only to fence late writes, and late writes existed only because status had
several writers and nobody knew exactly whether a runner was alive.

### What was wrong with the run model

The requirement stood: `--continue pN` keeps `pN` for primaries and children,
and `--fork`/`--from` mint a new spawn. #616 met it by making every launch an
immutable row with its own key: `pN` for run 1, `pN.2`, `pN.3` after it. The
reason was **status ownership**:
- three parties wrote a mutable `status`: the runner's finalize, the reaper and
  cancel;
- liveness was a guess: a PID with 30 s of create-time slack, plus heartbeat
  file mtimes inside 120 s.

Between them they left windows in which an old launch's late write could land
on a newer one. Separate rows were the fence. The fence cost a second identity
everywhere, and a key format in which a bare `pN` meant both "spawn pN" and
"run 1 of pN". That double meaning generated most of #616's eight review passes
([lesson](../lessons/state-design-lessons.md#a-name-with-two-meanings-generates-bugs)).
The format itself came from an earlier design rule, "no data migration", which
this project's no-backward-compatibility rule never required.

### The decision

**One ID.** A spawn `pN` has one ID in refs, env, parent links, the index and
output. Each run of it is a **launch**, counted inside the record. No user types,
reads or sees the count; it names directories on disk and pins waits and
cancels internally, and is never an identity.

**Status is derived, never stored as a mutable field.**

| Current launch's outcome | `spawns/pN/launch.lock` | Derived status |
|---|---|---|
| written | any | the outcome's status |
| absent | held by the runner | `running` |
| absent, cancel requested | free | `cancelled` |
| absent | free | `failed`, error `crashed` |

- The runner holds an exclusive `flock` on `launch.lock` for its launch's whole
  life. The kernel releases it when the runner dies, however it dies.
- The outcome is written **once**, by whoever holds the lock. Normally that is the
  runner as it exits. A non-runner holder (continue, cancel, maintenance) first
  writes the outcome every reader was already deriving, so no reader sees a change.
- Cancel writes a `cancel_request`, signals the recorded runner of the launch it
  observed, and never writes status.
- The reaper stops writing status. A bounded maintenance pass, which is just
  another lock holder, records crashes.
- Readers use one consistent read: a shared lock try, then the row.

With one writer and exact liveness, a late write cannot happen. A dead runner
writes nothing; a live one holds the lock, so no newer launch exists.

**Containment.** Three layers, in order of when they act:
1. **Start gate:** no harness executes before its process scope, with a verified
   birth time, is durable.
2. **Guardian:** a tiny `sh` process in the harness's group kills that group the
   moment the runner dies ([D-start-gate-guardian](#d-start-gate-guardian)).
3. **Sweep:** before launch K+1 starts, the lock holder kills whatever launch K
   left running, with birth checks. This is the guarantee; the guardian makes it
   rare that the sweep finds anything.

**Continue, parents and artifacts:**
- `--continue pN` adds a launch to `pN`; `--fork`/`--from` create a new spawn.
- Continuing p2 (created under p1) from p9 **moves** p2 under p9. Subtree queries
  follow the current parent, and no run-level history is needed.
- Each launch writes into its own directory, `spawns/pN/launches/K/`, launch 1
  included for new spawns. A reader that skips the path resolver then finds
  nothing and fails its first test, instead of silently returning launch 1's
  files for every later launch.
- No output prints a launch-directory path. `spawn show`, wait and the Pi
  notification point at `meridian spawn report show pN`.
- The runtime artifact store `artifacts/<spawn_id>/` is deleted. It was a second,
  spawn-keyed copy that finalize read first, so it would have published launch
  K's report as launch K+1's.

**Cost bar.** Continue, show, wait, cancel and close stay flat across 240 /
2,400 / 12,000 spawns × 1 to 500 launches, with no scans under global locks
([per-operation cost](../architecture/per-operation-cost.md)). The one exemption is
the pre-existing O(N) live-observer scan on continue, which #418 B3 removes.

### Why a runner-only lock

Two placements were probed on real harnesses before redesigning
(`spawn:p8181` Claude and Codex, `spawn:p8182` OpenCode and Pi; `work:v010-followups`
`probe-lock-liveness-a.md`, `-b.md`, D46).

| Placement | Result |
|---|---|
| **M1:** the harness inherits a copy of the lock fd | **Rejected.** Its truth depends on which processes each harness passes fds to. Codex's Node wrapper kept the fd and its native re-exec dropped it, which makes a false crash. A generic descendant kept it after the launcher died, which makes a false "running" |
| **M2:** only the runner holds the lock | **Exact for the runner.** The existing per-chat session lock was already this shape |

M2 exposed a separate problem: after a runner SIGKILL, Codex's native process
survived with PPID 1, and Claude background spawns had no death linkage at all.
That is containment, and it was on `main` until PR 2 (#625): a `--continue` after a
runner crash ran a second Claude beside the orphan on the same session. Per-launch
keys (the probe's fallback "option 2") don't touch it. So the decision pairs M2
with the gate, guardian and sweep, instead of falling back to per-launch keys.

### Why reopening in place is now safe

Revision 3 of the old design rejected reopening one row because every per-launch
writer would need a run fence, terminal would stop being final, and artifact
rotation had crash windows. Each depended on what this design removes:
- the lock is the fence for lifecycle writes;
- terminal stays final per launch, because an outcome is written once and then
  archived unchanged;
- nothing rotates, because each launch gets a new directory, and no status is
  inferred from an artifact.

The cost is unchanged from the run model: a spawn ID stops being an immutable
receipt for one result. Earlier launches' files stay on disk, and no command
shows them.

### Owner answers on the open questions (D50, `chat:c7736`)

| Question | Answer |
|---|---|
| Move a continued spawn to its new parent? | "yes" (continue-from-another-parent semantics, not a migration) |
| Show a crash as `failed`/`crashed`? | "ok" |
| Keep per-launch files; drop `show --launch K`? | "ok"; "we should just drop launch K". No launch number reaches any user |
| Watchdog? | "i thought we already had a watchdog?" One existed only for macOS backends. The answer generalizes it into the guardian on every path and deletes both old mechanisms |
| Print `meridian spawn report show pN` instead of report paths? | Recommended yes, presented with the approved design; called out in the explain-back so the owner can object |

### The seven PRs

| # | PR | Status (2026-10-28) |
|---|---|---|
| 1 | Start gate + guardian; delete `parent_watchdog.py` and the PDEATHSIG branch | merged, #633 (`e2593004c`, v0.10.13); the owner's macOS smoke is still owed |
| 2 | Launch lock + containment sweep (fixes #625) | merged, #647 (`60863fb78`, v0.10.18) |
| 3 | `read_derived`: exact liveness for every reader; lenient quarantine | merged, #672 (`ea6c8cc3f`, v0.10.24; fixes #663) |
| 4 | Sole-writer outcome; state v4; history index **v11** | not started |
| 5 | Cut the runtime artifact store | implementation complete for #677 at `463519248`; not merged |
| 6 | Per-launch directories through one `LaunchPaths` resolver | not started |
| 7 | Continue keeps the spawn ID | not started |

PR 5 is independent and must land before 6. PRs 4–7 are unbuilt as of
2026-10-28; after #800 ships, the owner wants a session dedicated to them: "PRs
4-7 are unbuilt oh... after this... we shoul start another session dedicated to
getting that back on track...", carrying #800's context "because its releated"
([ref classification](ref-classification.md),
[D-owner-on-start](chat-spawn-ownership.md#d-owner-on-start)). PR 2's residuals are listed
[below](#pr-2-as-built-the-launch-lock-and-the-sweep), and PR 3's
[after them](#pr-3-as-built-read_derived-and-one-dead-launch-finalize).

**Index versions across lanes:** #631 took v10, identity PR 4 takes v11, and
#418 B2 takes v12. Whoever merges second bumps
([D-session-per-chat-journals](state.md#d-session-per-chat-journals)). The
session-browse redesign (PR #714, merged) has since taken v12 out of turn, so both
lanes need numbers above 12; see the flag there.

### PR 2 as built: the launch lock and the sweep

PR 2 adds the lock and uses it in one place, continue. Status authority is
unchanged until PR 4.

**The lock is exact liveness.** `spawns/pN/launch.lock` is held exclusively by the
runner from the row's staged publication to runner exit. The publishing rename
keeps the inode, so the row is never visible unlocked. "Locked" means exactly "the
runner is alive", with no PID guess and no heartbeat. No harness holds the
descriptor. A `--bg` parent `dup`s the locked descriptor into its worker
(`_MERIDIAN_LAUNCH_LOCK_FD`), which restores close-on-exec and records itself as
the runner. The lock stayed held in 658 of 658 samples with the parent SIGKILLed
mid-hand-off. `LaunchSlot` (unbound) binds to a frozen `LaunchHandle`; the slot is
required on the lifecycle and service layers, so a new row creator can't publish a
lockless row.

**The sweep runs before any harness starts.** Every continue path (primary
`--continue`, `spawn --continue` and the session browser) calls one seam,
`launch/continue_sweep.py`:
- **Lock held:** refuse at once, before any row is written: "pM is still running;
  use spawn message pM or --fork pM".
- **Lock free:** terminate the source's unreleased, birth-verified scopes, mark them
  released, then continue. A scope whose birth is unknown is never signalled. A
  reused root PID proves the root dead, so that scope is released without a signal.
- `--dry-run` touches no process and refuses nothing.
- Lockless rows from older builds are judged by `legacy_runner_alive` and never
  swept while it holds. A dead one gets its lock file with `O_EXCL` under the
  mutation lock.

**Continue resolves an old spawn ID to the chat's current owner.** With p1 crashed
and continued as p2, a second `--continue p1` used to lock p1, find it free, and
start p3 on p2's live session. `--continue c1` was already safe, because chat c1's
record points at p2. The sweep now reads `chat_record(chat).spawn_id` (one keyed
read, no scans) and locks that owner first:
- owner live → refuse, naming it: "p1 was continued as p2, which is still running;
  use spawn message p2, --fork p2, or spawn cancel p2";
- owner dead → sweep the owner's scopes and the named spawn's, then launch;
- a pruned owner (no row) is ignored, rather than failing every later continue.

The live successor is **never killed**. Killing it would silently stop work in
another pane and break the rule that continue only sweeps dead launches (owner:
"fix it"; the kill was rejected).

**Two continues of one chat cannot both run a harness.** PR 2 had the `--bg`
parent publish the new spawn as the chat's owner before the hand-off, so that a
second continue in that window would see the chat busy (`spawn:p8373`, reproduced
with SIGSTOP); two truly concurrent continues of a dead source could still both
start. On branch `refactor/ref-resolution` (#800, not merged as of 2026-10-28)
that publish is deleted: `start_session` takes the chat's lifetime lock with a
two-second bound and refuses the loser, which fails in its worker. One harness
runs, racing or back to back, and `test_concurrent_continue_of_dead_source` now
passes instead of being an expected failure. Why the publish went:
[D-owner-on-start](chat-spawn-ownership.md#d-owner-on-start). PR 7's reopen
contract, where continue keeps the spawn ID, still replaces this interim design.

**Accepted residuals:**
- A cancel during the `--bg` hand-off window can SIGTERM the parent instead of the
  worker (`spawn:p8283` finding 4). Main's window was racy too; PR 4 moves cancel
  onto the launch.
- A descendant in its own session (`setsid`, `nohup … &`) survives the sweep, as
  the residuals below say.
- macOS was not exercised live.

Verified live on real Claude haiku and Codex: an orphan group swept before the new
harness starts, a live source refused in about 1.6 s with no new row; on a native
tmux Codex primary (post-#646) the old group was gone 493 ms before the new backend
appeared (`spawn:p8364`). Lessons:
[process containment](../lessons/process-containment.md#continue-must-lock-the-chats-current-owner-not-the-named-spawn).

### PR 3 as built: `read_derived` and one dead-launch finalize

PR 3 makes the lock the only liveness fact for every reader and for the writers
that persist a dead launch's outcome. Stored status stays in state v3 until
PR 4. The mechanism, the tables and the list of deleted guesses are in
[derived status](../architecture/state-system/derived-status.md); this section
records what was decided while building it and why.

**Liveness lives in the state layer.** `read_derived` and its helpers are in
`state/spawn_status.py`, and `legacy_runner_alive` moved there from
`launch/continue_sweep.py`. Leaving it in the launch layer would have made every
reader import launch code.

**The shared lock is held across the row read.** PR 2's `shared_try` released
the lock before returning, so the design's read protocol couldn't be built on
it. PR 3 adds `shared_hold` and re-expresses `shared_try` through it. Rejected:
probe-then-read, which the design review had already ruled out as non-atomic.

**A held lock keeps the stored active status.** The plan said a held lock means
`running`; PR 3 reads that as "active" and returns `queued`/`running`/`finalizing`
as stored. PR 4 deletes stored status, and with it that vocabulary, the stats
buckets and the filters. Collapsing to `running` now would churn all three twice.

**Cancel convergence and the reaper share one finalize** (D79). Building the
reaper's write path found that `_force_cancel_convergence` wrote `succeeded`
from a durable report whenever it saw no live owner, while the lock-holding
reaper derived `cancelled` for the same row. Equal origins meant the first write
won, so ordering decided the outcome. Before PR 3 both used "report ⇒ succeeded"
and agreed by accident. Decided: one state-layer `claim_dead_launch` that both use,
writing the derived outcome under the lock, behind one service method,
`finalize_dead_launch`. Rejected:
- making `derive` honour reports, which contradicts the design's
  report-through-outcome rule and the killed-while-finalizing change the owner
  approved;
- having the reaper skip rows with a cancel intent: reads would still say
  `cancelled` while cancel persisted `succeeded`, so the status would flip.

**Selection for convergence or destruction uses stored status; liveness
questions use derived status** (D80). The final review found descendant
cancellation and archive selecting by derived status, which skipped or archived
dead rows that still needed a writer; the round-2 probe found the same in
doctor's warning (D82). The rule and the three cases are in
[derived status](../architecture/state-system/derived-status.md#selection-versus-liveness).

**The reaper never finalizes a lockless row.** A build before PR 2
may still be running it, and only its own reaper can know. Explicit cancel keeps
the pre-PR-3 lattice for lockless rows; cleanup of a lockless row that is already
terminal is allowed once its old runner is gone, because cleanup is not
finalization.

**The heartbeat and the launch-boundary journal were deleted, not left
unread.** Once readers used the lock, both became write-only. The structure
review asked for the journal's module, call sites, lock class and docs to go in
the same PR.

**A primary `--continue` skips the background orphan repair** (D84). CI caught
the race: with repairers now claiming dead launches' locks, a primary
`--continue` of a crashed spawn scheduled the repair, the repair held the
source's lock while persisting the crash, and the continue's sweep refused with
"still running". The continue's own sweep owns the source, so the orphan pass is
skipped for that launch; the next launch or `doctor --kill-orphans` persists
outcomes. When a dead source's lock is held by someone else, the refusal says
"held by a repair or another continue; retry in a moment". Rejected: waiting
out a non-runner holder, which let two concurrent continues both start
successors; and a holder-marker file, which goes stale on a crash and doesn't
serialize continues.

**Owner rulings** (2026-10-17, on D78's open questions):
- the 30 s PID-birth slack: "Readers only". PR 3 removes it from every status
  reader and the reaper; cancel's signal targeting keeps it until PR 4 rewrites
  cancel;
- a managed primary crash showing `failed`/`crashed` instead of the earlier
  `cancelled`/130: "Accept, state it". Since #658, SIGTERM and SIGHUP primaries
  finalize themselves, so only SIGKILL and OOM reach it.

The other two behaviour changes, killed-while-finalizing → `crashed` and
cancel-of-a-dead-spawn-with-a-report → `cancelled`, follow from the same rule.
They, and `cancel-all` now selecting every stored-active row, were stated in
#672's body when the owner approved the merge (D86).

**Residuals:**
- **Not claimed:** consistency against racing writes. Cancel's request, and the
  `finalizing` transition cancel writes for a managed primary, are still written
  outside the lock; PR 4 moves
  every lifecycle write under it.
- Handed to PR 4: cancel's
  `_has_live_execution_owner` slack gate can veto the lock; cancel of an
  already-dead launch waits out a 5 s grace; a derived terminal's `finished_at`
  is `""`; `managed_primary.release_managed_primary_scopes` still decides
  runner death by a PID-plus-birth-time check
  ([process scope](../architecture/process-scope.md)) instead of the lock.
- **#671:** a parent's finish or cancel never converges its descendants;
  `cancel_descendants` runs only from resident-drain cleanup. It predates PR 3
  and needs a policy decision.
- Readers don't persist. A dead launch stays stored-active until
  `doctor --kill-orphans`, the background repair, or cancel writes it; plain
  `doctor` warns `dead_spawns_unfinalized`.

Lessons: [state design](../lessons/state-design-lessons.md#a-deleted-rule-survives-in-other-writers-and-in-fixtures),
[test determinism](../codebase/test-determinism.md#fixtures-that-mean-live-must-hold-the-lock),
[verification](../lessons/verification-and-review-discipline.md#compare-against-the-base-before-calling-a-probe-failure-a-regression).

**Provenance:** `work:v010-followups` D77–D82 (`pr-body-read-derived.md`,
`design-identity-v2/pr3-dag.md`, `lane-b-terminal-writer-blocker.md`,
`reviews/pr3-*.md`); reader map `spawn:p8418`; implementers `spawn:p8421`,
`spawn:p8426`, `spawn:p8433`, `spawn:p8427`, `spawn:p8428`, `spawn:p8429`,
`spawn:p8451`, `spawn:p8452`, `spawn:p8459`; reviewers `spawn:p8444`,
`spawn:p8445`, `spawn:p8446`, `spawn:p8447`, `spawn:p8453`, `spawn:p8454`;
probers `spawn:p8448`, `spawn:p8455`; investigator `spawn:p8456`.

### Rejected

| Alternative | Why rejected |
|---|---|
| Ship #616's `pN`/`pN.K` run model | A second identity everywhere for a fence that exact liveness makes unnecessary; the bare-`pN` double meaning kept generating bugs |
| Option 2: unambiguous per-launch keys | Fixes the naming but neither status ownership nor containment |
| M1: the harness inherits the lock | Inaccurate in both directions, depending on each harness's fd handling |
| Keep PDEATHSIG plus a Python watchdog | PDEATHSIG reaches only the direct child; see [D-start-gate-guardian](#d-start-gate-guardian) |
| `show --launch K`, a launch-count column | Reintroduces a user-facing second number (owner, D50) |
| Launch 1 at the top level of `spawns/pN/` for new spawns | Readers that skip the resolver would return launch 1 for every later launch, plausibly and silently |

### Residuals

- A descendant the agent moved into its own session (`nohup … &`, Claude's
  Bash tool, Pi's detached bash) survives the sweep once its parent is gone, as on
  `main`. It can't start a second harness, write status, or write into another
  launch's directory.
- Builds older than PR 3 can't resume existing chats once PR 4 writes a v4 row.
  PR 4's upgrade notes say so.
- Every Meridian lock assumes a local filesystem; this design adds no new
  dependency on that.

**Sequencing with the Rust rewrite** is the tech lead's recommendation and still
awaits the owner: finish identity-v2 in Python first, then port strangler-style,
runner first, because flock plus a once-written outcome is a language-agnostic
contract ([Rust rewrite](rust-rewrite.md)).

**Revisit when:** a harness needs to show or act on an earlier launch, or the
NULL-outcome backlog that maintenance drains turns out to grow without bound.

**Provenance:** `work:v010-followups` D45–D63, PR 2 in D65–D75 (`pr-body-launch-lock.md`;
design §5.1, §5.5; implementer `spawn:p8279`, `spawn:p8290`, `spawn:p8367`,
`spawn:p8380`; reviewers `spawn:p8283`, `spawn:p8284`, `spawn:p8292`,
`spawn:p8293`, `spawn:p8373`); `design-identity-v2/`
(`redesign-brief.md`, `design.md` rev 3.1, `pr-plan.md`, reviews r1–r3); design
lead `spawn:p8186`, `spawn:p8230`; reviewers `spawn:p8216`, `spawn:p8226`,
`spawn:p8234`; main probes `spawn:p8231`, `spawn:p8233`; `chat:c7736`,
`chat:c8003`.

---

## D-start-gate-guardian: one `sh` guardian per harness instead of PDEATHSIG and watchdogs (2026-10) {#d-start-gate-guardian}

**Status:** decided as part of [D-spawn-identity-v2](#d-spawn-identity-v2)
(owner answer D50 Q4, approved 2026-10-16). Merged in PR #633 (`e2593004c`,
v0.10.13); the native-terminal primaries were ported onto it in #646 (v0.10.15).
The owner's macOS smoke is still owed. Mechanism:
[process scope](../architecture/process-scope.md#parent-death-linkage).

**Supersedes** the mechanism of
[D-launcher-watchdog](primary-terminal-ownership.md#d-launcher-watchdog--one-watchdog-per-launcher-instead-of-pdeathsig)
(#614, closed unmerged). Its diagnosis stands: PDEATHSIG reaches only the direct child, and
unknown birth is never "probably fine". Its Python watchdog per launcher does not.

### Problem

Two partial mechanisms covered some paths each, and neither closed the
exec-to-registration window:
- **PDEATHSIG kills only the direct child.** Codex's command is a Node wrapper
  that re-execs the native binary as its own child. After a runner SIGKILL on
  `main`, the native process kept running with PPID 1 in the backend's group, on a
  scope recorded as `parent_death_linked: true` (`spawn:p8231`).
- **The Python watchdog** ran only for macOS backends, polled the process table,
  and cost about 17 MB.
- **Nothing covered** Claude, Pi or Cursor stdio children or the primary
  launchers. A Claude runner SIGKILL left Claude running; a continue then ran a
  second Claude on the same session (`spawn:p8233`, #625).

### Decision

Every harness on Linux and macOS starts as a POSIX `sh` stub behind a pipe
handshake:
1. The runner creates the stub with the pipe's read end in `pass_fds`.
2. It reads the stub's birth time. The stub is an unreaped child, so its PID
   can't be reused. Unknown birth kills the child and fails the launch.
3. It records the process scope durably, then writes `go`. `exec` keeps the PID
   and birth time, so the scope already describes the harness.
4. Before `exec`, the stub double-forks a **guardian** that blocks reading the
   pipe. The runner holds the only write end, which is close-on-exec and in no
   `pass_fds`, so EOF means the runner died.
5. After the harness exits, the runner writes `stop`, and the guardian exits
   quietly.

If the runner dies before `go`, the stub reads EOF and exits 125 without
running the harness. If it dies after, the guardian acts:

| Mode | Paths | On runner death |
|---|---|---|
| **Group** | managed stdio (Claude, Pi, Cursor), managed backends (Codex, OpenCode), the PTY primary, non-TTY primaries | SIGTERM to the group, the adapter's grace, SIGKILL. The guardian is a group member, so the group ID can't be reused while it acts |
| **Root only** | a non-TTY primary whose stdin is a TTY (stdout piped), which stays in the runner's group so Ctrl-C reaches it | one SIGTERM to the harness PID, nothing later |

Root-only mode never sends a delayed signal. Nothing pins a bare PID, so a
SIGKILL after the grace period could hit a process that reused it. The next
birth-checked sweep finishes the job.

**Properties:**
- The guardian is re-parented to init, not the harness's child, so a harness
  that waits for all its children doesn't hang on it.
- It ignores SIGHUP while it waits, so a PTY hang-up at runner death doesn't kill
  it first, and it ignores SIGTERM only after deciding to kill.
- Measured on Linux with dash, 20 concurrent launches: 1.1 MB RSS each, 0 CPU while
  blocked, about 0.86 ms added per launch (`spawn:p8254`).

**Rules settled while building PR 1:**
- **The gate fd moves to 9 through `/dev/fd`** (D55). dash accepts only
  single-digit fds in `n<&m`.
- **The guardian's first act detaches it from harness stdio** (D55, D58). Every
  fd it holds references the gate pipe or `/dev/null`.
- **The PTY child closes every inherited fd except the gate's** before exec, as
  `Popen(close_fds=True)` does for the other launchers (`spawn:p8254`).
- **Env names are internal:** `_MERIDIAN_GATE_FD`, `_MERIDIAN_GATE_SCOPE`,
  `_MERIDIAN_GATE_GRACE`. The stub unsets all three before exec (D59, D62).
- **One gate seam** (D62): `LaunchGate.spawn_kwargs()`, `adopt()`/`adopt_async()`
  and one `scope_from_identity()` builder. Every launcher's record-then-go lives in
  one place, so PR 2's lock hooks land once instead of in four drifting copies.
- **Primary scopes are real and killable now**, so the reaper may terminate one
  only when the runner is **proven dead** by the birth check (D62). Unknown
  identity and access errors are not proof (D63). The proof gates scope
  termination only; status finalization still proceeds, because a forever-active
  ghost row is worse than killing nothing.
- **"Proven dead" uses the alive check's ±1 s tolerance** (D63). psutil's
  `create_time` moves with wall-clock steps.
- **Async adoption failures** use a caller-supplied kill-and-reap, so asyncio's
  watcher is the only reaper (D63).

**Deleted:** `platform/parent_watchdog.py`; `platform/detached_process.py`
(PDEATHSIG pre-exec, `prctl`, watchdog start and handshake, `ParentDeathLink` on
POSIX); the `parent_death_linked` scope field, which is now ignored on read. The
Windows Job Object handle stays on `ScopedProcessHandle`, an existing untested
branch.

### Rejected

| Alternative | Why rejected |
|---|---|
| PDEATHSIG, here or on more children | Reaches one PID; Codex's re-exec escapes it |
| A Python watchdog, per launcher (#614) or per child | About 17 MB, polls or needs a registration protocol, and still leaves a registration window |
| The harness inheriting a lock fd as the death signal | Same fd-inheritance problem as M1 above |
| `preexec_fn` dup2 onto fd 9 | `_posixsubprocess` orders `pass_fds`/`close_fds` so fd 9 is either closed or required in the parent |
| `posix_spawn` file actions | Rewrites every Popen launcher |
| Require bash | Not guaranteed on Linux; macOS `/bin/sh` may be dash |
| A named FIFO for the gate | Filesystem cleanup on crash |
| Leave the primary scope unrecorded | PR 2's sweep needs it |

**Residuals:**
- A harness on the root-only path that ignores SIGTERM keeps running until the
  next sweep.
- A guardian that is itself killed (OOM, by hand) leaves the group to the sweep.
- None of the POCs ran on macOS, and the owner's manual Mac smoke is still owed.

Implementation lessons: [process containment](../lessons/process-containment.md).

**Provenance:** `work:v010-followups` D50, D51, D55, D58, D59, D62, D63;
`design-identity-v2/design.md` §5.6; implementer `spawn:p8254`, `spawn:p8258`,
`spawn:p8262`, `spawn:p8267`; reviewers `spawn:p8259`, `spawn:p8260`,
`spawn:p8265`, `spawn:p8266`.

### D-spawn-run-identity: a spawn id names the conversation; each launch is a run (2026-10) {#d-spawn-run-identity}

**Status: superseded, never shipped.** Replaced by [D-spawn-identity-v2](spawn-identity.md#d-spawn-identity-v2) (owner-approved 2026-10-16): one ID per spawn, with launches counted inside the record and never shown, and status derived from a runner-held lock plus an outcome written once. The combined PR #616 (`feat/spawn-identity`) was closed unmerged on 2026-10-16, and #617–#620 were closed as moot. The user's requirement below, that `--continue` keeps the spawn id, carries over. `main` still mints a new bare `pN` per launch. This record stays for the requirement and for why runs were separate rows. That reason, a fence against late writes from several status writers and guessed liveness, is exactly what the replacement removes. The run-row mechanism, reopen contract and type guard below describe #616, not current or planned code.

**Original status:** decided by the user on 2026-10-14 (`work:primary-spawn-identity`, design revision 3). Built, not merged. On 2026-10-15 the owner agreed to combine #587 (phases 1–2: run identity in storage, surfaces that resolve spawns to runs) and #590 (phases 3–4: reopen, telemetry, hooks, vocabulary docs) into one branch, `feat/spawn-identity` (`work:v010-followups` D26). It will open as one PR that replaces both. The reasons for combining:
- #587 alone had no visible value, but it carried the v8 index rebuild, a bare-`pN` directory scan and an `@latest` regression;
- CI releases every merge;
- the cost fixes straddled the two PRs (the latest-run pointer's writer was in #590, its reader in #587);
- it is one design.

The branch adds fixes so it meets the [per-operation cost bar](../architecture/per-operation-cost.md): a latest-run head, a keyed owner lookup, `show pN` reading only its own runs, and session and parent reads moved out of `spawns_flock`. The correctness review found three blockers in those fixes ([lessons](../lessons/cost-cuts-and-safety-checks.md)), and fix pass 2 is in progress. On `main`, **every primary and child launch, `--continue` included, still mints a new bare `pN`**. Architecture, vocabulary and native-session-binding pages describe `main`, so they still show one spawn per launch and change when the combined PR merges.

**Decision.** `--continue` reopens the spawn; `--fork` and `--from` create a new one. The rule is the same for primaries and children.

- A spawn `pN` is the stable name of a conversation thread. It has no row of its own; it is derived from its runs.
- Each launch is a **run**: an immutable row with one terminal status, report and artifact directory. Run 1 is stored under the bare key `pN`, later runs under `pN.2`, `pN.3`, and so on. `pN.1` is an alias of `pN`.
- A spawn has at most one active run. A second `--continue pN` is refused and points at `spawn message` or `--fork`.
- `MERIDIAN_SPAWN_ID` is the stable spawn, and children record it as `parent_id`. `MERIDIAN_RUN_ID` is the exact run, and self-targeting commands act on it.
- `spawn list` shows one row per spawn (the user's choice); `spawn show pN` shows the latest run plus a run table.
- Chat `cN` stays as the native-segment id that native binding relies on. It leaves the default vocabulary: the quit hint will say `pN`.

**The requirement, in the user's words.** They first asked "why would it NOT keep its spawn id?" about resumed primaries. Asked to confirm the design, they answered: "i thought that was already the case for spawns... changing spawn ids even with continue on subagent spawns... thats bad...". Before this, every launch form, resume included, minted a new spawn row; only the chat id persisted. [D-idle-reopen-return](idle-notifications.md#d-idle-reopen-return--a-meridian-reopen-is-a-user-return) had already dropped a design that relied on per-launch spawn ids, because the user did not want that as a contract.

**Why runs are separate rows.** Finalization, the reaper, heartbeats, process scopes, inject and control sockets, locks and telemetry all key on the row id and carry no run token. Reopening a row in place would need a run fence on every one of those writers. It would also break "terminal is final" and bring back the stale-artifact crash windows that retired `attempt-N/` rotation ([D-one-launch-attempt](launch.md#d-one-launch-attempt-harness-turn-failures-finalize-without-replay)). With one row per run, every per-launch mechanism stays as it is. The identity boundary is what changes: references, the public environment, parent links, descendant traversal and presentation use `pN` and resolve it to a run when they need one.

**Settled specifics:**

| Question | Decision |
|---|---|
| Existing rows | No data migration: each is run 1 of its own spawn. The history index gets `spawn_id`/`run` columns through a schema bump, rebuilt rather than migrated in place |
| Type guard (D-guard) | No mass `SpawnId`→`RunId` rename across ~84 files. The guard is narrow: row read/write entry points and path/lock helpers accept only `RunId`; only `parse_ref`, `latest_run` and `runs_of` produce one; one core env accessor replaces raw `MERIDIAN_SPAWN_ID` reads. A CI gate enforces this, and every exact row-key conversion in `ops/`/`cli/` is listed in a reviewed manifest |
| Sibling-run rules | One shared module owns grouping, continuation-target selection and retention protection, so no surface derives "latest" by itself |
| Retention | Each run ages on its own. A spawn's numeric latest run, its latest run that bound a session, and a distinct continuation source are kept until the whole spawn can retire. Run numbers are never re-minted; older runs of a long-lived spawn can still age out. Prune decides from the siblings it re-reads under the allocation lock, never from its scan: a reopen can publish a new run between scan and delete. If any sibling row is unreadable, that run is kept and the rest of the batch continues |
| Spawn counter | Mints stable spawns only. A missing counter is recovered from the spawn prefix of every surviving row, dotted ones included, so retiring run 1 cannot cause `pN` to be re-minted (an explicit correction during PR-A) |
| Run reference grammar | `pN` is a spawn, `pN.K` (K ≥ 2) a run, `pN.1` normalizes to `pN`. `--continue pN.K` is refused in favour of continuing `pN` or forking the run. A `pN` with no run is "not found"; it never falls back to an untracked reference. A spawn whose loose runs were all archived looks the same as one that never existed, so the refusal names both possibilities rather than guessing "archived" |

**Rejected alternatives.**

| Alternative | Why rejected |
|---|---|
| Reopen in place, rotating earlier runs into `runs/K/` | Every per-launch writer would need a run fence; artifact rotation has crash windows |
| Nested layout `spawns/pN/runs/K/` | Same semantics, but needs a live migration of ~8k directories and gains no behavior. Could come later as a pure storage refactor |
| Merge chat into spawn (delete `cN`) | Rewrites the native-identity core, and `/clear` segments would still need a name |
| Keep spawn = run and present `cN` as the conversation | Contradicts the user: ids still change in env, parent links and trees |
| A new `pM` per run, with `pN` as an alias | Two names for one thing, plus alias chains to maintain |
| Store an explicit owner on the chat | The session `spawn_id` field is the generation→run link that archive, retention and index aliases read. The owner is derived from the newest linked generation instead |
| The 84-file mechanical rename (design revision 3 §1) | The reviewer's narrower guard catches the same bug class (a surface that forgets to resolve `p2` and silently reads run 1) at the persistence entry points, with far less churn |

**The cost we accepted.** A spawn id stops being an immutable receipt. After someone continues p5, `spawn show p5`, `wait p5` and `report p5` show the newer run. The exact historical result is `p5.1`. Existing `spawn:pN` provenance citations, including the ones in this KB, will still resolve, but to the spawn. Whether KB provenance should start citing `pN.K` is still the user's call.

**The reopen contract (PR-B).** Primaries and children follow one path:

1. **Busy check.** A read-only pre-flight refuses early with a useful message. The authority is the recheck inside `reserve_run`, under the spawn allocation lock: the pre-flight is advisory and the recheck is never skipped.
2. **Reserve before the chat.** `reserve_run` publishes run K+1 as a queued row, stamped with the reserving process so the reaper can finalize it after a crash. It copies no session generation from the previous run. Reopening an ancestor of the caller is refused, using the union of parents across all runs.
3. **Acquire the chat after releasing the allocation lock.** The chat lock is a try-lock. A live lease owner means `chat_busy`. Contention without a live owner is treated as transient and retried for about 10 s, then fails as `chat_lock_timeout`.
4. **Bind the generation once.** Session scope writes the acquired generation onto the new run, once, for both kinds. Session start never assigns a generation on reopen, so a failed reopen can't steal the previous run's generation.

A reopen that fails before acquiring its session finalizes only the new run. That run stays unbound, keeping the target chat as its chat and the caller's chat as its owner chat. The continue target is the newest run that bound a generation, so an unbound failure can't redirect the next `--continue`. The predecessor's task-dir scope, including a cleared one, carries into the new run. Reservations translate into the stored row through one function shared with fresh launches, so the two paths can't drift.

On every surface, `pN` labels the spawn and `pN.K` the run. `spawn show` heads with the spawn and names the exact run on its own line. The quit hint says `pN`. Telemetry and hook payloads carry the stable `spawn_id` and the exact `run_id`. `MERIDIAN_CHAT_ID` is the run's entry chat.

**Revisit when:** the user reverses "continue keeps the id" before the combined PR merges, or provenance drift from reopened spawns causes real confusion.

---
