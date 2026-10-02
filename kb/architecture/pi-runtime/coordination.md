# Pi Runtime Coordination

Pi spawned sessions coordinate through three owners: the live Bash process
owner inside Pi, `meridian-spawn-watch` inside the same Pi process, and the
Python streaming observer. Spawn rows remain Meridian's authority; Pi-private
files carry task ownership and notification evidence. None of the private
records replaces persisted spawn state.

The design is deliberately evidence-based. A child row, a live process handle,
a native queue admission, and Python's observation of the public event are
different facts. Keeping them separate lets a crash or unreadable store fail
closed without claiming a result was delivered when the system cannot prove it.

## Child and Bash ownership

The watcher discovers children from canonical rows whose `parent_id` is the
current parent spawn. `originating_bash_id` links a child to the Bash command
that launched it and transfers that command's result obligation to the child;
it does not establish child membership by itself. Logs, timestamps, scan
history, and the retired origin sidecar are not child authority.

Python's terminal-child projection accepts canonical status facts alongside
extra Pi metadata; it does not define or validate the full `TerminalFacts`
schema. Scope checks exclude canonical child rows definitively parented to a
different spawn. Missing, malformed, or locally contradictory authority stays
unknown rather than being treated as an empty child set.

`managed-bash` owns each live shell through one launch-scoped owner keyed by
the immutable Bash-record path. Module reload keeps the owner and process
handles alive while current hooks are rebound. A new Node process has no such
owner: a formerly running record is retained as `ownership_lost`, and its old
PID is never signalled. Wait reports the loss; kill explains that Meridian no
longer owns the process; explicit detach releases tracking.

Each live task owns its POSIX process group. Termination escalates from TERM to
KILL with bounded waits; terminal state is published after the group exits and
queued output writes finish. `bash_manage(detach)` only removes the task from
tracking; it does not detach process ownership. Normal Pi shutdown still kills
every group owned by that runtime, including explicitly detached tasks.

Ping notifications are advisory. If a native notification callback fails,
managed-bash releases and persists the ping claim, warns, and leaves the shell
running. It does not spin on the same broken callback. Hook rebinding or later
task activity can schedule another attempt. Task, output, and record-publication
failures remain execution failures.

### Extension UI and RPC output

Pi's `ctx.hasUI` alone does not identify a useful custom UI host: Pi 1 can expose
it as true in RPC while the custom-display API is a no-op. Panel selection must
use the mode-aware capability predicate, including `ctx.mode` when available.
Slash-command output in RPC goes through native `ctx.ui.notify()` so ordinary
text never corrupts JSON-RPC stdout; print-mode commands retain plain text.

## Completion obligations and delivery

Tracked terminal background Bash results remain owed until a terminal wait
successfully persists `notification_consumed_at_ms`, or an exact unattended
completion message is admitted. A wait that fails or times out while the task is
still running does not consume the result. An admitted child message consumes
only its exact `work_ids`; a launcher Bash result is transferred only to its
canonical direct children.

Completion policy separates known running work from causal delivery obligations.
An explicit Pi `done` may retain the override for known running execution or
descendant liveness; it cannot bypass publication, native admission, or public
observation still owed for a terminal result. Delivery blockers suppress the
done nudge and hold the done decision. An active parent reply, non-idle parent,
or unknown evidence also prevents done from completing the spawn.

The watcher publishes only while Pi's native context is idle. It formats spawn
results with `spawn wait --no-observe`: reading details must not consume the
same result whose notice is being prepared. Immediately before sending, it
rechecks explicit consumption and live wait leases and rebuilds a partially
consumed batch. Thus an explicit terminal wait during an active tool turn can
consume its result before any follow-up is queued; an unattended eligible result
is delivered once through the normal idle path.

The watcher reserves a batch through formatting and native send so parallel
flushes cannot publish the same IDs. IDs that arrive after reservation wait for
the next batch. A failed send leaves its IDs eligible for the next ordinary
scan; it does not start a retry loop. Successful admission commits only the
delivered or explicitly suppressed IDs, and a final Bash-consumption read
closes the race with a wait completing during formatting. Shutdown prevents new
flushes while retaining undelivered work for normal recovery.

`sendMessage()` queues a native message and returns no admission proof. The
custom message carries one `delivery_id` and its exact `work_ids`. Pi's native
`message_start` hook writes a durable `delivery-receipts.json` receipt for that
membership before the public RPC event is exposed. Python observes that exact
public event only after marking the parent active, then writes the corresponding
`delivery-observations.json` entry. Receipt and observation are separate
cross-store facts; neither substitutes for the other.

One launch-scoped watcher owner reserves a batch through native admission.
Reload rebinds that owner and preserves queued claims. A cold restart has no
old native queue: work without an admission receipt is eligible for one new
delivery attempt, while explicitly consumed and fully admitted-and-observed work
stays consumed. A receipt without its public observation remains unknown. The
deprecated `last-notification.json` timestamp is ignored; it cannot prove exact
membership or delivery.

The stores cannot atomically commit a Pi-native queue admission and Python's
observation across a process crash. A receipt without its matching observation
therefore remains unknown and fails closed at a fixed diagnostic deadline; do
not invent an acknowledgement, delete the receipt, or report success. This is a
bounded failure contract, not atomic exactly-once delivery. An older
unconsumed result whose former admission had only the retired timestamp marker
may be delivered again after upgrade.

Explicit waiters reserve spawn IDs in `observed-spawns.json` under a lease
containing caller identity, owner PID, process-birth epoch, expiry, and IDs. Only
a live matching process lease suppresses a concurrent notice. Expired or
ownership-lost reservations cease suppressing, while durable exact observations
remain consumed. Diagnostic deadlines for unknown evidence anchor at the first
unknown assessment; unrelated activity does not move them.

## RPC transport and usage

Pi's RPC receiver runs independently of event subscribers and starts before the
initial prompt is written. This keeps ACK progress live even when an event
consumer is busy or initial input and output are both large. JSONL reception is
bounded at two levels: 64 MiB per raw frame and 128 MiB for the unread serialized
event inbox. Oversize input fails explicitly; the receiver does not truncate or
silently discard earlier events. ACK deadlines and terminal connection failure
remain owned by the receiver.

Usage is folded from normalized assistant-message increments once. Replayed
cumulative `agent_end.messages` data is not added a second time. Missing usage
fields stay unknown rather than becoming zero, and cost stays null when the
provider price is unknown. Tool-result payloads do not become model usage.

## Authority files

All files live under `pi-bash/<parent>/`; the checked-out source schema is
`src/meridian/pi_runtime/.context/delivery-contract.md`. Their durable roles
are:

| File | Authority |
|---|---|
| `bash-records.json` | Bash task facts, wait-consumption marker, and unresolved execution/storage evidence. |
| `observed-spawns.json` | Durable explicit wait observations plus process-owned temporary wait leases. |
| `delivery-receipts.json` | Exact native message admission and work membership. |
| `delivery-observations.json` | Exact matching public RPC events observed by Python. |
| `delivery-fault.json` | Bounded diagnostics for scan, admission, and persistence failures, separate from task execution. |
| `cleared-spawns.json` | User dismissal from the `/spawn` view; not delivery or child authority. |

Missing authority means empty only when no obligation requires evidence. Present
malformed, wrong-parent, or invalid-schema evidence is unknown. The shared
delivery reader validates receipts before Bash history pruning as well as
completion decisions, so clearing the UI cannot erase an unconsumed obligation.

See [Pi lifecycle and quiescence](../pi-lifecycle.md) for how these private
facts combine with the shared transitive descendant assessment, and the
[Pi result-delivery decision](../../decisions/pi-runtime.md) for the rationale
and failure tradeoffs.
