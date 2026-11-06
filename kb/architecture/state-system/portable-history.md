# Portable history

Readable files and verified immutable ZIPs are history authority; SQLite indexes are rebuildable projections. A history UUID identifies the transcript across local alias changes, while PIDs, leases and scopes describe runtime ownership. Restore keeps those facts as provenance and never turns them into authority to signal or resume a process. See [history storage decisions](../../decisions/history-storage.md) and the source checkout's `src/meridian/lib/state/.context/history.md`.

An offline selected snapshot cannot silently fall back to older content under the same identity. Only independently verified copies of the same digest are interchangeable. Publication preserves loose sources until verification; restore refuses conflicting content and leaves the ZIP intact.

### A caller with a deadline builds the index before starting its clock

A cold build needs a separate budget: otherwise it can consume the query deadline and make a live child appear unknown. On a 1,093-record / 6,077-session corpus, metadata construction took 5.68 seconds. Corrupt or incompatible state still needs explicit repair; contention is not an empty result. See [test determinism](../../codebase/test-determinism.md#cold-index-builds-inside-a-test-deadline-a-flake-class).

## Archive capture is native-only

Capture selects the exact run generation's bound native source, not runner output or a newer session. Post-stop capture observes current bytes and can include later continuation; it is not a reconstruction of the instant the run ended. Known incomplete transcripts cannot authorize reclaim. See [native-only history](../../decisions/native-only-history.md).

### Capture defers to a live owner and retries when it stops

A same-runtime live owner, observer, exact-generation lease or unreleased serving scope can still write the transcript, so capture defers and rechecks before publication. Stop maintenance retries terminal records bound to that native key. An unstopped generation alone is not proof of a live writer; a crash can leave it unstopped. See [history storage](../../decisions/history-storage.md).

Retirement follows verified ZIP publication and a short exclusive recheck of protection and source witnesses; witnesses detect changes but do not replace checksums. This avoids holding a global writer gate while hashing or recursively deleting. See the source checkout's `docs/history.md` and `src/meridian/lib/state/.context/history.md`.
