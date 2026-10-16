# Decisions: The Rust Rewrite

The owner wants Meridian rewritten in Rust. The two reasons given are that a
running `meridian` crashes when it is updated underneath itself, and memory use.
This page records what has been deferred to the rewrite and the requirements it
inherits. The rewrite itself has no design yet.

---

## D-update-crash-waits-for-rust: no Python fix for #628 (2026-10) {#d-update-crash-waits-for-rust}

**Status:** decided by the owner on 2026-10-16 ("wait for Rust";
`work:update-crash` D2). #628 stays open as the record. No fix PR.

**Problem (confirmed, `spawn:p8243`, `spawn:p8247`).** `uv tool install --force`
overwrites the single tool virtualenv in place. A long-lived process keeps its
already-imported modules from version N, then lazily imports a module from
version N+1 next to them and fails at the first incompatible boundary. The
captured failure, between v0.10.4 and v0.10.5:
`TypeError: IdleFacts.__init__() missing 1 required positional argument: 'draft'`.
A same-version control passed.

**Options offered:**
- a self-snapshot into `releases/<content-hash>/` (the recommendation);
- a `meridian upgrade` launcher;
- eager imports, which mitigate only part of it;
- wait for the Rust rewrite (chosen).

**Workaround until then:** don't upgrade while long-lived Meridian processes are
running.

**Requirements the rewrite carries.** A static binary alone does not satisfy
them:
- Replace the binary by rename, so running processes keep their image.
- No re-exec through a mutable path mid-workflow. Today `cli/session_cmd.py`
  calls `os.execv(sys.executable, …)`.
- Embed or version assets.
- Explicit state-schema versions, because old and new processes share
  `~/.meridian` during an upgrade.

---

## Sequencing (recommendation, pending the owner)

The tech lead recommended finishing [identity-v2](spawn-identity.md#d-spawn-identity-v2)
in Python first (`work:v010-followups` D54):
- It fixes #625 now.
- It deletes the reaper, heartbeats, watchdog, artifact store and three-writer
  finalize, leaving less to port.
- After it, flock plus a once-written outcome is a language-agnostic contract
  that a Rust runner can honour beside Python readers.

The port would then go strangler-style, runner first.

**Rejected in that recommendation:**
- **Rewrite before identity-v2.** It ports machinery that is about to be deleted,
  and leaves #625 open for months.
- **Rewrite during identity-v2.** Two moving targets.

**Memory, measured 2026-10-16:**
- 17 Python processes held 5.8 GB RSS.
- About 90 MB per process is import cost, which a port removes. `meridian --help`
  is 16 MB; importing `execute_bg` is 92 MB.
- Runner heap above that came from `get_session_active_work_id` replaying all of
  `sessions.jsonl` (#418 B1). A port of the same algorithm would carry that cost
  over, so it was fixed in Python: #634 and #651 (v0.10.16) brought a real runner
  from 329 to 103 MiB, the import floor
  ([D-session-read-seam](state.md#d-session-read-seam)). Fixed Python processes sit
  near 100 MiB; the audit's estimate for Rust was 5–30 MB.
