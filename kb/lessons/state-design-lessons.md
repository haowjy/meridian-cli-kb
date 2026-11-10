# State Design Lessons

Committed identity travels with the repository; high-churn runtime state stays user-local. Ad-hoc identity is path-derived and does not survive a move. See [state decisions](../decisions/state.md).

A runner-held lock provides evidence of runner liveness without PID or timestamp guesses. The harness must not inherit it: a descendant can keep it after the runner dies or drop it during re-exec. That lock does not prove all descendants have exited; containment is a separate question. See [spawn identity](../decisions/spawn-identity.md) and [process containment](process-containment.md).

Selection for convergence or destruction uses stored status; a liveness query can use derived status. Otherwise a reader can make a dead but not yet finalized row invisible to the writer responsible for repairing it. See [derived status](../architecture/state-system/derived-status.md#selection-versus-liveness).

## Repair the State Before Relaxing the Check

When a safety check refuses because persisted state is wrong, repair the state before changing the check. A relaxed check may hide the problem while other readers still depend on the false state. See [abandoned-generation closure](../decisions/history-storage.md#a-later-start-of-the-same-chat-closes-earlier-generations-capture-defers-to-real-owners).

## Cross-Language Consumers of Persisted Schema

Persisted state can have readers in another language. Sweep those consumers when changing its shape, and derive meaning from the authoritative discriminant rather than duplicating the writer's status vocabulary. Unknown or quarantined rows must be explicitly preserved by destructive readers; dropping that partition can turn uncertainty into deletion.

A migration must be checked against real historical state, not just the previous writer's code: persisted rows can predate that writer and lack fields later treated as required.

## A Link That Spans Two Stores

A projection joining two stores makes each store's writers responsible for the other's state, inviting deadlocks, partial updates and expensive rewrites. Give each store its own projection and join them without holding both locks. A derived link or environment value must not be read back as authoritative input. See [session work links](../decisions/session-work-links.md#one-projection-per-owner).
