# Results and waiting

A lead is owed the unconsumed launches of its direct background children; a continuation can owe another launch under the same spawn ID. A nested lead owns its own children, never its siblings or its parent's backlog, and a primary owns its chat's background children across generations. Interactive Claude primaries and Pi sessions get automatic delivery at idle. Spawned Claude, Codex and OpenCode leads don't: they end their turn and the [completion hold](completion-hold.md#waiting-has-one-number) wakes them with finished children's report paths, or they run `spawn wait`. Contracts: the source checkout's [`ops/spawn/.context/CONTEXT.md`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/.context/CONTEXT.md) and [`pi_runtime/.context/delivery-contract.md`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/pi_runtime/.context/delivery-contract.md). Per-launch keys are explained in [spawn launches](../state/spawn-launches.md#results-are-owed-per-launch).

## Core decides what is owed; a Claude mod only senses idle

On a Claude primary, a mod senses idle and submits; core decides what is owed and when it is consumed. A mod fits here because delivery is wanted at idle, unlike mid-turn messaging. Submission proves only that the prompt was queued: Claude's `$.prompt.submit` also resolves for a prompt queued behind a running turn, which is lost if the user clears the queue or Claude dies, and Pi's `sendMessage()` says nothing about admission. So consumption is recorded from evidence that the model's turn began with the text, never from the send call's return: the Claude mod puts a nonce line in the submitted text and acknowledges only on the main-loop `turn.start` that contains it, and Pi records the admission event ([Pi](../harnesses/pi.md#delivery-evidence)). The order is claim in memory, send, observe the turn, then write the durable record, so a crash between steps can cause a duplicate but never a loss. A failed acknowledgment excludes those IDs until reload, to avoid repeated delivery; one duplicate after a restart is possible. Foreground results have already reached the caller, and grandchildren reach it through their parent's report.

Claude delivery carries lookup headers, not report bodies: that bounds repeated context and avoids report I/O. Pi delivers bounded report excerpts ([Pi](../harnesses/pi.md#delivery-evidence)).

> [!FLAG] **Needs human review**: whether Pi delivery should also become header-only is undecided; Pi keeps its excerpts until then. Flagged 2026-11-07.

## What consumes a result

A CLI surface that prints a terminal report consumes it for the caller. Internal reads, summaries, logs and context references don't: reading a result for another purpose must not erase a delivery the lead hasn't seen. A cancel that reports `cancelled` consumes that outcome, since the caller just saw it; a cancel that finds success or failure leaves the unseen report owed. An unrelated caller's observation can't suppress the owner's delivery.

Consumption is keyed by the consuming spawn (a primary keeps its spawn ID across continues) and stored in that spawn's directory, where published-spawn locking stops a concurrent prune from recreating a deleted directory. Readers also count command observations, Pi admission receipts and cleared launches, including the legacy files, because processes from older builds still write them until they restart.

## `spawn wait` blocks on a fixed set

`meridian spawn wait` blocks on a fixed set of launches and yields a checkpoint when work remains. A fixed set lets the caller choose when to start another barrier, instead of letting new work extend the wait forever. A yield is not a failure; `--timeout` is a separate hard limit. Nested callers exclude siblings and ancestors, so a child never waits on its own parent. A waiting process pins the launch it awaits across checkpoints, so continuing the same spawn can't replace the result ([spawn launches](../state/spawn-launches.md#results-are-owed-per-launch)). A busy history index can yield a discovery checkpoint; retry rather than treating it as an empty set. Spawned leads normally end their turn instead of waiting ([completion hold](completion-hold.md#waiting-has-one-number)).

## No-argument wait includes owed results

A no-argument `spawn wait` waits on a fixed barrier of active descendants in the caller's scope, which stays uncapped, and adds up to the 32 oldest finished owed children as headers. Without them a child that finished just before discovery would vanish from view. Owed rows don't affect counts, fail-fast or the exit code, so an old failure can't fail a new wait, and headers rather than full reports keep a backlog from flooding context. Capping the combined set would leave live descendants outside the barrier.

Children launched after the wait starts are reported in a trailer and never join the barrier: otherwise a producer could extend the wait indefinitely. The trailer is advisory: it doesn't lease, consume or count those children, and discovery failure just omits it.

## Accepted duplicates and losses

Delivery is best effort, and these windows were accepted instead of a renewable claim protocol, which was disproportionate to them:

- A wait records consumption before printing, so a kill between the two loses the output.
- Collect's last consumption check and a wait's lease are separate, so a race can deliver one duplicate header; a queued mod delivery can also duplicate.
- Recording an observation happens after output; contention or a read-only runtime causes a duplicate rather than a lost report.
- Older builds may redeliver a result a newer build consumed.
