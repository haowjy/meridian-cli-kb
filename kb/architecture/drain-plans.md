# Drain plans

Drain plans separate generic streaming from completion policy. A plain plan uses terminal frames; resident and Pi plans add their own evidence and completion policies. The source checkout's `src/meridian/lib/streaming/.context/completion-and-entry-points.md` describes the resident hold; [completion decisions](../decisions/completion-drain.md) give its reasons.

## Resident completion

A completed turn supplies a candidate, not yet a published outcome. The hold combines descendant evidence with any native work evidence. A parent with unread child results is woken before it closes, including when a child finished during the parent's turn and never appeared as a blocker. A successful result cannot publish over an active turn or unreadable evidence.

The hold sends a confirmation where the harness cannot report its own background work. `done` can override known blockers after the active turn ends, but not unknown evidence; `rearm` keeps the child open. A reply to the expiry ask closes through cleanup, which stops work named by the ask. A child without a writable completion-signal channel receives no confirmation ask. See [completion coordination](completion-drain-coordination.md).

### Claude native holds

Claude's background-task snapshots add native work evidence to persisted descendants. An empty snapshot is not by itself a completed follow-up turn: Claude may still owe the result of the turn triggered by the completed task. A malformed snapshot holds completion. A harness mod cannot replace the native stream because the timeout and finish decision must survive a wedged child.
