# Per-operation cost

Launch, close, wait, show, cancel and continue concern one spawn and chat; their work should not grow with project history. Global scans belong to explicit global commands or bounded maintenance. Measure physical reads and lock holds across corpus sizes, not just wall time on a shared machine: a filtered pass still reads the bytes. See [contention diagnosis](../lessons/contention-diagnosis.md).

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

## Measurements and remaining work

Measurements on historical copies are not current guarantees:

- Session appends held the lock for about 16 ms after #588, down from 1,463 ms at 46.6 MB; reader P95 under two writers fell from 2,001 to 101 ms.
- No-argument wait discovery fell from 672 to 95 ms warm at 3,951 indexed rows (#599).
- A no-op foreground launch took 3.59, 6.02 and 9.12 seconds on 1.05, 18.9 and 46.6 MB session logs; scans and lock time grew with history (#418).
- On a 4,226-spawn, 54.5 MB runtime, full object materialization reached 444 MB for a spawn scan and 525 MB for a session replay. Python retained arenas after the scan, so one full materialization could set the process's lifetime RSS peak (#627). Streaming startup repair reduced measured RSS from 226 to 101 MB (#634).
- Warm indexed list operations took about 2 seconds at 12,000 spawns (#600); a cold index build took 2.4–5.7 seconds (#592). A caller with a deadline builds before starting its query clock, so a slow cold build does not turn valid evidence into a timeout ([history](state-system/portable-history.md#a-caller-with-a-deadline-builds-the-index-before-starting-its-clock)).
- A small-runtime no-op foreground launch took about 3.6 seconds; interpreter and subprocess startup are a separate fixed floor (#595, #527).

Session teardown's all-spawn scan (#612), index catch-up and query costs (#594, #600, #592), and explicit doctor scans (#601) remain tracked. A per-chat candidate index must be followed by authoritative reads; a stale or incomplete index cannot authorize a lifecycle decision. See [cost and safety](../lessons/cost-cuts-and-safety-checks.md).
