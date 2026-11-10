# Pi runtime

A spawned Pi session stays open for follow-up turns while tracked work or delivery is outstanding. Pi's managed Bash, spawn watcher and Python observer keep separate evidence of execution, native message admission and public observation; spawn rows stay the authority for child membership. Contracts: the source checkout's `src/meridian/pi_runtime/.context/CONTEXT.md` and `delivery-contract.md`; Pi drain: `src/meridian/lib/streaming/.context/pi-drain.md`.

## Quiescence

A turn end alone can't establish that a Pi spawn is done: a native retry, a compaction or a delivered result can still need another turn. Completion requires native settlement, no open compaction, and readable evidence that descendants, managed Bash and owed deliveries have cleared. Missing or malformed evidence is unknown, never proof of completion ([completion hold](completion-drain.md)).

An idle Pi spawn with known running execution can get an advisory done nudge. A done directive can override known running execution, but never a result whose publication, admission or observation is still owed, and it waits for an active parent turn and for unknown evidence to resolve.

## Bash ownership

**Tracked Bash** can block quiescence; **detached Bash** no longer blocks it but stays owned by the live runtime until it exits or the runtime shuts down. A managed-Bash owner keeps process handles across extension reload. After a cold restart a recorded PID can't prove ownership, since it may have been reused, so recovery records the ownership as lost and never signals it. A failed advisory ping releases its claim without killing the shell; a failed execution or durable publication is a task failure.

## Delivery evidence

A terminal result stays owed until explicitly consumed or admitted to Pi's native queue. Queueing, native admission and Python's observation of the public event are separate facts in separate stores, and a crash can fall between them, so exactly-once delivery isn't guaranteed. A receipt without its matching observation is unknown: it blocks completion with a diagnostic, and a cold restart can retry work without a receipt but never invents the missing observation. Admission doesn't prove the model acted on the message.

Only canonical direct-child rows establish membership. A **Bash origin link** ties a spawn to the Bash task that launched it and can transfer that task's result obligation to its children, but it never creates a child relationship.

Automatic notices are bounded in aggregate as well as per item: a large backlog could otherwise exhaust file descriptors, or one large report starve the rest. Each included item keeps its identity, outcome and a way to read the full result; anything not included stays owed. A successful tool boundary can deliver completed work while the parent is active; idle delivery covers work that finishes later. An explicit wait can consume a result first. Pi delivers bounded report excerpts where Claude delivers headers ([child result delivery](../concepts/child-result-delivery.md)).

A queued message Pi never admits may keep IDs reserved and pause unrelated publication until a native lifecycle callback, reload or restart. This is a known limit, not a reproduced defect.
