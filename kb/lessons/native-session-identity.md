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
`finalize_run_boundary`), `tests/integration/launch/test_pi_run_boundary.py`.
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

A related trap: SQLite `mode=ro` does not mean the source files stay untouched. A
read-only connection to a WAL database can still write shared-memory read marks. When
an import promises not to write native stores, copy the files and open the copy.

**Where this lives:** `ops/legacy_native_import.py`, `harness/legacy_native_stores.py`,
`state/session_binding.py`, `tests/integration/ops/test_legacy_native_import.py`.
Provenance: `work:native-harness-session-identity` (review `spawn:p7091`, recheck
`spawn:p7093`, `evidence/pr1-legacy-recheck-wal-race.py`).

---

## Cross-References

- [Native session identity decision](../decisions/native-session-identity.md)
- [Native session binding](../architecture/native-session-binding.md)
- [Claude native sessions](../architecture/claude-native-sessions.md)
- [Pi native sessions](../architecture/pi-native-sessions.md)
- [Review convergence gate](review-convergence-gate.md)
