# Pi

A spawned Pi session stays open for follow-up turns while tracked work or delivery is outstanding. Pi's managed Bash, spawn watcher and Python observer keep separate evidence of execution, native message admission and public observation; spawn rows stay the authority for child membership. Contracts: the source checkout's `src/meridian/pi_runtime/.context/CONTEXT.md` and `delivery-contract.md`; Pi drain: `src/meridian/lib/streaming/.context/pi-drain.md`.

## Quiescence

A turn end alone can't establish that a Pi spawn is done: a native retry, a compaction or a delivered result can still need another turn. Completion requires native settlement, no open compaction, and readable evidence that descendants, managed Bash and owed deliveries have cleared. Missing or malformed evidence is unknown, never proof of completion ([completion hold](../spawns/completion-hold.md)).

An idle Pi spawn with known running execution can get an advisory done nudge. A done directive can override known running execution, but never a result whose publication, admission or observation is still owed, and it waits for an active parent turn and for unknown evidence to resolve.

## Delivery evidence

A terminal result stays owed until explicitly consumed or admitted to Pi's native queue. Queueing, native admission and Python's observation of the public event are separate facts in separate stores, and a crash can fall between them, so exactly-once delivery isn't guaranteed. A receipt without its matching observation is unknown: it blocks completion with a diagnostic, and a cold restart can retry work without a receipt but never invents the missing observation. Admission doesn't prove the model acted on the message.

Spawn rows are the only authority for which results a session owes; the extension mirrors core's rule rather than keeping its own. Membership, origin links, notice bounds and Bash ownership: the source checkout's `src/meridian/pi_runtime/.context/`. Pi delivers bounded report excerpts where Claude delivers headers ([results and waiting](../spawns/results-and-waiting.md)).

A queued message Pi never admits may keep IDs reserved and pause unrelated publication until a native lifecycle callback, reload or restart. This is a known limit, not a reproduced defect.

## Session identity

Pi can create a new identity when a requested session file is missing or empty, so Meridian validates the exact recorded source before resume or fork. Pi also discovers existing IDs by scanning headers rather than filenames; collision checks must use the same evidence. Create persistence can be lazy, so an assigned but unwritten session is not yet resumable. See the source checkout's `src/meridian/lib/harness/pi_identity.py` and [native session binding](../state/native-session-binding.md).

### Exit observation

The session-boundary extension supplies launch-correlated observations; the launcher reads its final record after process teardown, because Pi's shutdown hook can run after the RPC turn ends. A missing or invalid record leaves exit unresolved rather than inventing identity. Pi invalidates an extension context on session replacement; the known stale-context error clears exit proof without poisoning identity. A Pi upgrade that changes the error text needs requalification. See the source checkout's `src/meridian/pi_runtime/README.md`.

Even after preflight, external replacement of the source can make Pi open a different session before the mismatch is detected. Detection fails the attempt but cannot retract input already sent.

### Journal topology and readback

Pi reopens the root-to-leaf ancestry ending at the last physical journal entry. Meridian uses that view for tracked sessions, omitting abandoned branches and marking incomplete graphs partial; an explicit file read may show physical order. See [native transcript reads](../state/native-transcript-reads.md#native-transcripts-are-the-only-source).
