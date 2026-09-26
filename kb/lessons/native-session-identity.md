# lessons/native-session-identity — Lessons from Native Session Identity Work

These lessons capture failures and review findings from implementing cross-harness native session identity. The architecture and decision are documented in [native session identity](../decisions/native-session-identity.md).

## Green Suites Did Not Find the Seam Defects

**What happened:** During the native-session-identity work, every lane passed a green
full suite (~1,950 tests, pyright 0). Each lane still had defects that only showed up
at a seam between components:

- A Codex fork lost its recorded store in the fork *request builder*. The adapter-level
  fork tests passed because they called the adapter directly.
- Claude preparation seeded a fork from a same-ID decoy in the ambient root. Reads were
  exact, but nobody had put a decoy next to the recorded file.
- Codex bound a session ID from assistant prose (`codex resume <uuid>` in model
  output). Fixtures never put identity-shaped text in content.
- The Pi session-boundary extension poisoned its record on a real session switch. A
  read-only audit of Pi's installed lifecycle API and a Node fixture replaying the
  documented event order both passed. Only a real zero-turn Pi process showed that
  Pi invalidates the hook context on replacement, and that a shutdown can arrive on
  the invalidated runner when stdin closes mid-replacement.

After the lanes merged, a whole-change review of the integrated head (all lanes
green, ~1,983 tests) found three more P1s, each at a seam no single lane owned:

- The primary Claude runner completed and attributed a run whose first `system/init`
  named B while argv assigned A (exit 0, `succeeded`). The streaming lane had fixed
  this; the primary runner belonged to no lane.
- The merge that wired the Claude trampoline successor into the exit allocator
  promoted a heuristic to "verified". An unrelated concurrent chat in the same cwd
  became the run's exit.
- Claude and Codex "exact" locators checked filenames only, and tests blessed `"{}\n"`
  as a resumable journal. A file named for A holding B's events resumed as A.

The one-turn real-Pi run then found a fourth: the streaming runner read Pi's exit
record before Pi had written it (see
[Terminal Status Is Not Process Exit](#terminal-status-is-not-process-exit)).

**What found them:** an independent per-lane review, then a *recheck* of the fix pass
on a frozen head, then a review of the whole change. Each cycle found *new* seams
rather than the same seam failing again, which is what convergence looks like (see
[Review Convergence Gate](review-convergence-gate.md#lane-reviews-do-not-review-the-composition)).
Reviewers reproduced each defect in an isolated installed wheel using POSIX `sh`
harness shims, decoy fixtures, and an unrelated concurrent session, not source
imports. The two Pi defects needed the real binary.

**The lesson:** For identity and lifecycle code, a green suite shows that the paths
you wrote down work. It says nothing about composition paths you did not enumerate.
Budget one review plus one recheck per lane, then one review of the integrated whole
before release. Qualify through the installed artifact
with adversarial fixtures: decoys, prose, nested keys, a changed ambient root. A fake
of an external lifecycle must reproduce its *invalidation* semantics (what stops
working after a transition), not just its event order. Before trusting an extension
against a host, run one real-process probe of the host lifecycle.

**Where this lives:** `tests/integration/launch/test_owned_identity_shims.py`,
`test_recorded_source_store.py`, `test_fork_source_store.py`,
`test_claude_recorded_prelaunch.py`, `test_pi_run_boundary.py`;
`pi_runtime/extensions/session-boundary/lifecycle.fixture.mjs`,
`test_launch_process_claude_session.py`. Provenance:
`work:native-harness-session-identity` (`spawn:p7056`, `spawn:p7062`, `spawn:p7065`,
whole-change review `spawn:p7072`, real-Pi run `spawn:p7075`).

---

## Terminal Status Is Not Process Exit

**What happened:** The first Meridian spawn against real Pi 0.87.1 created the right
native session, and the session-boundary record on disk held a valid final `quit`.
`spawn show` still said `exit unresolved`. Runner lifecycle timestamps showed why:
`finalizing` at 06:25:36.8, boundary file written at 06:25:40.9, connection cleanup
done at 06:25:41.0. Pi runs its shutdown hook only when the RPC connection's cleanup
shuts the process down. The streaming runner published terminal status when the turn
completed, which hid the connection, and a `get_connection(...) is not None` guard
then skipped `stop_spawn`. The boundary was read about four seconds early.

**The tempting fix:** poll the record until `quit` appears, or add a timeout.

**What worked:** move the existing teardown join ahead of the read and drop the guard.
`stop_spawn` already joins sessions that are terminal. No polling, no timer. The
regression shim publishes `quit` only from its SIGTERM trap, so it fails whenever the
read happens before teardown.

**The lesson:** Evidence a child writes while exiting can be read only after the child
has exited. "The turn finished", "terminal status published", and "the connection is
gone" are not process exit. Order the read after the join instead of waiting for the
file. Shims for exit evidence must write it late, during termination, or they cannot
catch an early read.

**Where this lives:** `launch/streaming_runner.py` (teardown join before
`conclude_native_run`), `tests/integration/launch/test_pi_run_boundary.py`.
Provenance: `work:native-harness-session-identity` (`spawn:p7075`,
`evidence/lane-q3-report.md`; fix `spawn:p7076`, commit `9eba09a2`).

---

## When a Fix Pass Is Not Converging, Remove Carriers

**What happened:** After the first Lane C fix pass, the recheck still blocked, with one
old P1 partly open and one new P1. Both came from the same thing: three request fields
described "the recorded source" (`source_native_store`, a Claude config-root field, a
Pi session-dir field). Each had its own producer and consumers. One consumer (the fork
builder) forgot one field. Another (Claude preparation) gave a field a different
meaning than its producer did.

**The tempting fix:** patch the fork builder, patch Claude preparation, and add tests
for both. That would have been a third round of the same shape.

**What worked:** treat both P1s as one consolidation. Delete the two extra fields, make
`source_native_store` the only carrier, and push each harness's interpretation of the
store into its adapter. Every source read became exact, and the two P1s closed
without adding any mechanism. The check that the design was still converging was
simple: each cycle *removed* carriers rather than adding them.

**The lesson:** When review rounds keep finding "this consumer dropped/misread that
field," the fields are the defect. Count the carriers of one concept, reduce them to
one, and move per-consumer interpretation to the seam that owns the consumer.

---

## A Once-Only Marker Turns Transient Failures Into Permanent Ones

**What happened:** The one-time legacy import writes a completion marker, and after
that the import never runs again. Two review rounds found three ways the marker made a
temporary problem permanent, or a local problem global:

- The first cut ran inside runtime-root resolution and wrote the marker only on
  success. One quarantined `state.json` or a locked OpenCode DB raised on every
  command. The import never finished, so it failed the CLI the same way each run.
- OpenCode's lookup swallowed `sqlite3.Error` and reported "absent". A snapshot torn
  by a checkpoint during the copy made 201 of 251 real sessions look absent in a
  reproduction. The marker would have recorded them as `missing` forever.
- Each bind did three fsyncs while holding the sessions lock. 3,000 imports took
  181 s, and every live launch waited behind them.

**What worked:** separate "could not look" from "looked and found nothing". Damaged
or changing sources raise a typed error. The trigger catches it, writes no marker,
and lets the command continue. Lookups on the import path are strict. The snapshot
is fingerprinted before and after the copy. Slow native I/O runs outside the sessions
lock, the lock-held part only rechecks identity, and all binds share one durable
append (0.5 s for 3,000).

**The lesson:** Before writing a marker that ends a migration, check each negative
outcome it will record permanently. Every such negative must come from a successful
read, not from an error that was swallowed. A migration that runs at a common startup
point has to fail soft, and its per-item cost has to be measured at real scale while
the relevant locks are held.

**A related trade-off: `mode=ro` is not "untouched".** A read-only SQLite
connection to a WAL database can still write shared-memory read marks. The import's
first fix copied the OpenCode DB and WAL and fingerprinted the copy. That cost about
6 GB of scratch and 17–26 s per attempt on a 6.3 GB live database. The copy also ran
again on every deferred attempt while OpenCode kept writing. The shipped import reads
in place through the same strict `mode=ro` reader that live reads use:
- read marks are what every reader of that database writes, OpenCode included;
- strict queries still raise instead of reporting "absent";
- a failed attempt still defers, now with a 15-minute backoff note.

Copy only when a promise of zero writes matters more than the copy's cost.

**Where this lives:** `ops/legacy_native_import.py`, `harness/legacy_native_stores.py`,
`state/session_binding.py`, `tests/integration/ops/test_legacy_native_import.py`.
Provenance: `work:native-harness-session-identity` (review `spawn:p7091`, recheck
`spawn:p7093`, `evidence/pr1-legacy-recheck-wal-race.py`; in-place read and backoff:
`evidence/pr1-install-readiness-report.md`).

---

## Restructure Before Stacking

**What happened:** PR 1 passed its whole-change review, and the next two PRs were
planned to stack on it. A thermo-nuclear review then ran three independent lanes:
state, runners and harness. All three said **restructure before stacking**, and
they converged on the same defects:
- **Three drifted copies of the runner identity pipeline:** the process runner, the
  streaming runner and `streaming serve`. The drift was real: a post-exit observed-ID
  contradiction failed the process runner but only warned in streaming, and serve had
  no entry verification at all.
- **The binding rule written three times** in the session store, with conflict
  logging inside the pure replay fold. That was the source of 153 warnings per
  command on real data.
- **Stringly identity errors,** and an overloaded `locator` field.
- **Three single-harness post-exit hooks,** plus dead hooks.
- **Write-only copies** of entry and exit facts on the spawn row and in primary
  metadata.

**The tempting path:** stack PR 2 now and clean up later. PR 2's readers would then
have been written against names the cleanup renames, and against a pipeline whose
three copies disagreed.

**What worked:** one reconciled design, then five phases in parallel worktrees:
- P0: types and errors.
- P1, P2 and P4 in parallel: store binding, harness pre-exec, spawn row.
- P3: the runner pipeline.
- P5: verification.

Each phase had to leave the files over 1,000 lines smaller. Each was gated on
behavior equivalence against real data:
- the session fold over a copied 7,000-chat journal serialized byte-equal;
- 1,000 generated histories matched;
- a launch-golden matrix pinned argv, env and refusals across harness × operation.

The behavior changes were few, named, and each had a red-first test. The
restructure also produced the seams PR 2 needed: `by_native_key`,
`SpawnRecord.continue_chat_id`, and a single artifact identity read for PR 2 to
delete. PR 2's design, measurement and first slices ran in parallel with P3.

**The lesson:** when independent reviewers converge on duplicated policy under a PR
that others will build on, consolidate first. Gate the consolidation on real-data
equivalence, not only on the suite. Let the next PR's seams be the restructure's
outputs.

Provenance: `work:native-harness-session-identity`:
- `review/thermo-state.md`, `review/thermo-runners.md`, `review/thermo-harness.md`
  (`spawn:p7083`–`spawn:p7085`);
- `design/pr1-foundation-restructure.md`;
- phase lanes `spawn:p7100`, `spawn:p7102`, `spawn:p7114`, `spawn:p7104`,
  `spawn:p7116`;
- recheck `spawn:p7120`, alignment `spawn:p7121`, probe `spawn:p7122`, fix pass
  `spawn:p7126`.

---

## A Projection That Re-derives the Authority's Rule Drifts From It

**What happened:** PR 2's design took search bindings from the metadata index's
`sessions` projection, to avoid folding the 10.8 MB `sessions.jsonl` on every search.
The precondition was that the projection "folds keys the way the authority does". On
the real journal, all 7,025 chats agreed, so the gap looked structural only.

R3 reproduced the gap on synthetic roots before writing any code. The index started
each generation from an empty record:
- a key-less resume generation read as unbound;
- a conflicting start created a generation that the authority rejects.

Real data had agreed only because neither shape had happened yet.

**What worked:** R3 stopped at its API gate instead of patching the index. The fold's
per-event generation step became one public pure function in `state/session_fold.py`.
The authoritative fold and the index's incremental catch-up both call it, and the
existing 1,000-seed equivalence test guards it.

**The lesson:** a projection that needs an authority's rule must call the
authority's code, not restate it. "All real rows agree" is not evidence of
equivalence; build the divergent shapes on purpose.

Provenance: `work:native-harness-session-identity` (`evidence/pr2-r3-report.md`,
`spawn:p7128`; `decision.md` "R3 escalation → fold extraction authorized").

---

## `pytest -x` in a Slice Gate Hides the Next Stale Test

**What happened:** PR 2 slice F2 intentionally removed two behaviors:
- the reaper's history-mtime sign of life;
- the transcript hint for spawns with only runner history.

Its gate ran `pytest -x`, stopped at the first stale test, and that test was fixed.
Two more stale tests further down the suite never ran in the slice gate. They
surfaced only in the integration branch's full run, after the merge.

**The lesson:** a slice that deliberately changes behavior should run the full suite
without `-x` once. That enumerates every test encoding the old behavior, so they are
fixed in one sweep. CI can keep `-x`, because there a first failure is enough to
block.

Provenance: `work:native-harness-session-identity` (`decision.md` "P5 fix pass
merged into PR 1; PR 2 gate red on two F2-stale tests";
`evidence/pr2-stale-tests-report.md`).

---

## An argv Gate in `sitecustomize` Misses `python -m` Children

**What happened:** PR 2's history-blind test mode (`pytest --runner-history=off`) must
reach subprocesses. It does this through a test-only `sitecustomize`. To avoid
importing Meridian into every Python child, fix lane C gated the writer patches on
`sys.argv[0]` looking like the Meridian entry point.

Under `python -m meridian`, `argv[0]` is `'-m'` while site runs; `runpy` rewrites it
later. So CLI subprocess tests ran with **real writers** in blind mode. The failure
count even improved, from 15 to 3, which looked like progress. The tech lead caught it
by reading the gate, not the count.

**What worked:** decide by import, not by argv. `sitecustomize` installs a meta-path
hook, and the hook patches the writer modules right after they load. This covers
`python -m`, console scripts and `runpy` alike, and costs about 17 ms per child. A test
now asserts that a `python -m meridian` child sees the patched writers.

PR 3 later deleted the writers and retired the patches; the read trap still reaches
children through the same `sitecustomize`.

**The lesson:** a test mode that silently stops applying reports *better* numbers.
Prove that the mode reached the process under test with a positive assertion, not with
a lower failure count. `sys.argv` is not reliable at site time.

Provenance: `work:native-harness-session-identity` (`decision.md` "Fix pass: A merged;
B authorized; C's blind-mode change sent back"; `evidence/pr2-fix-c-report.md`;
`review/pr2-recheck.md` N11).

---

## A Diagnostic Flag Must Not Discard Authoritative Data

**What happened:** the thermo review found that `AttemptFacts.incomplete` was set but
never read (F2). Fix lane A made finalization persist `usage=None` whenever facts were
incomplete, so a partial generic sum could not pass as a total.

But `incomplete` was also set by any unparseable stdout line. In the recheck probe, one
`Warning:` line before a Claude `--print` result dropped its `total_cost_usd`. Budget
enforcement read the same value, so it went blind too (NF1). The 200-run replay could
not catch it: none of those runs had a malformed line.

**What worked:** move the rule into the fold, where the cause is known. A failed fold
step drops only generic fallback usage. A harness-specific total survives, and a
malformed line only marks the diagnostic. The probe became a red-first test.

**The lesson:** when a fix makes a flag consequential, list every place that sets the
flag. Each one must justify the new consequence. A replay corpus proves agreement only
for the inputs it contains.

Provenance: `work:native-harness-session-identity` (`review/pr2-recheck.md` NF1;
`evidence/pr2-fix-d-report.md`).

---

## A Test Helper That Reads Runner Bytes Trips the Blind Trap

**What happened:** PR 3's prune lane added a test that snapshots the whole runtime
tree before and after a dry run, to prove nothing changed. Its `tree()` helper read
every file's bytes, including the runner `history.jsonl` fixtures. Under
`--runner-history=off` the read trap fired in test code: "runner history read is
disabled: …/p6/history.jsonl". The default suite was green, and the blind suite went
from 0 failures back to 1.

**What worked:** compare runner fixtures by `lstat` (size, `mtime_ns`, inode), not by
bytes. Production code was never at fault.

**The lesson:** a read trap guards test helpers as strictly as production code. A test
that proves a file is untouched should compare metadata, not contents, whenever the
contents are off-limits. Run both suite modes before calling a lane green.

Provenance: `work:native-harness-session-identity` (`review/pr3-review.md` finding 2;
`evidence/pr3-prune-test-blind-report.md`; `spawn:p7160`).

---

## A Repair That Fixes the Cause Must Also Clear the Latched Failure

**What happened:** PR 3's schema needed an index reproject. On the upgraded runtime
copy, the reproject met the quarantined dogfood rows and latched an `authority`
initialization failure for that index generation. `meridian doctor` then migrated all
63 rows, but the latch stayed: the migration marked the sources dirty without changing
the generation. Archive and search kept failing until someone ran `session index
rebuild --metadata-only`, and running the rebuild before doctor failed again.

**What worked:** the repair re-arms what it fixed. When the migration rewrites a row,
it clears only `authority` markers, under the catch-up lock, so an in-flight
initializer is ordered first. The error text names the quarantined row and `meridian
doctor`. The copy probe then went: archive fails naming doctor → `doctor` → archive
exits 0, with no manual rebuild.

**The lesson:** when a failure is latched to stop retry loops, the latch outlives its
cause. Whatever repairs the cause must also clear the latch, or the repair looks
successful and changes nothing. Probe the upgrade path on a copy of real data, in the
order a user would run it.

Provenance: `work:native-harness-session-identity` (`evidence/pr3-fix-e-report.md`
"upgrade hazard"; `evidence/pr3-fix-f-report.md`; `decision.md` "PR 3 lanes E/F
landed"; `spawn:p7162`, `spawn:p7163`).

---

## Probe the Harness Contract, Not the Probe Brief

The first live probe of every harness found real defects, but it also reported a
Claude `output.jsonl` absence as a failure. That expectation was wrong: Claude writes
that file only through the process runner, while the headless path under test uses the
streaming runner. Likewise, tmux showed TUI prompt text in the composer before Enter;
the message was not submitted until the prober sent a separate Enter. Treating either
observation as a product failure would have produced a code change without evidence.

**The lesson:** make the probe's expected observable behavior match the actual launch
path and the TUI's input protocol. When evidence contradicts a brief, inspect the
implementation and reproduce at the relevant seam before changing code.

**Reinstall risk:** reinstalling the uv tool environment while an old-build headless
spawn was running killed that runner and its Codex child; interactive primaries
survived. Avoid replacing the running tool environment during a headless spawn, and
do not assume the interactive-primary observation generalizes to headless children.

Provenance: `work:native-harness-session-identity`, `evidence/probe-final-claude.md`,
`evidence/probe-final-codex.md`, `evidence/probefix-opencode-report.md`, and `decision.md`
(p7171 reinstall observation).

---

## Moving One Option Let a Variadic Flag Swallow the Prompt

PR #534 moved Claude's `--session-id` to the front of argv so the identity was bound
early. The change looked local, and every test passed. But in 0.6.7 that option had
also been the only thing ending Claude's variadic `--add-dir <directories...>`.
Without it, `--add-dir` consumed the trailing starting prompt as one more directory,
and every interactive Claude primary with `-p`, `--prompt-file` or `--from` opened
with an empty composer. A user's composed handoff sat in `starting-prompt.md` for
11 minutes until they typed something else. The tests checked that the prompt was
in argv, not where. The same probe found that Pi and OpenCode had never delivered
the prompt on any build; nobody had checked the first native user turn.

**The lesson:** argv order is part of a CLI contract when the target has variadic
options. End option parsing explicitly (`--`), and assert position in tests. To
verify prompt delivery, read the first native user turn, not Meridian's own
`starting-prompt.md`. Mechanism: [starting prompt delivery](../architecture/launch-system.md#starting-prompt-delivery).

Provenance: `work:native-harness-session-identity`, `evidence/probe-primary-prompt.md`,
investigation `spawn:p7218`, fix commit `b6e7cd0b`.

---

## An In-Place Projection Upgrade Breaks the Build Still Running

The history index was "disposable", so upgrading it in place looked free. It was
not free for a 0.6.7 background runner still running during the upgrade. That
runner read the same file, hit a schema it did not know, and never finalized. A
projection that is disposable for the new build is still live input for the old
one. The fix gave each schema its own files
([D-history-index-schema-namespace](../decisions/history-storage.md#d-history-index-schema-namespace)).
Only a probe that kept an old runner alive across the upgrade found it
([probing the upgrade path](dogfooding-pr-builds.md#probing-the-upgrade-path-needs-a-genuinely-old-build-and-old-state)).

---

## A Lead's Direct Edit Skips the Gate Its Lanes Run

The lead made a one-line JSON change (`f6eb3984`) and ran only the targeted test. It
broke five sparse-JSON contract tests, fixed in `27a57934`. The same afternoon a lead
edit in a worktree broke probe lanes running from it
([rule](dogfooding-pr-builds.md#probing-the-upgrade-path-needs-a-genuinely-old-build-and-old-state)).
**Rule:** a direct edit gets the full suite, like any lane commit.

---

## Cross-References

- [Native session identity decision](../decisions/native-session-identity.md)
- [Native session binding](../architecture/native-session-binding.md)
- [Claude native sessions](../architecture/claude-native-sessions.md)
- [Pi native sessions](../architecture/pi-native-sessions.md)
- [Native-only history decision](../decisions/native-only-history.md)
- [Native transcript reads](../architecture/native-transcript-reads.md)
- [Attempt facts and delivery](../architecture/attempt-facts-and-delivery.md)
- [Review convergence gate](review-convergence-gate.md)
- [Spawn lane operations](spawn-lane-operations.md)
