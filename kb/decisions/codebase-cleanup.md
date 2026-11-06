# Codebase Cleanup

Remaining cleanup work is tracked in [#569](https://github.com/haowjy/meridian-cli/issues/569).

## D-b7-via-546: the spawn wait contention failure is fixed by PR #546

Wait discovery must not answer from a stale index that could omit a new spawn. The contention fix shortens the session writer's lock hold; see [history index readers](history-storage.md#d-history-index-readers-never-wait-on-session-writers).

## D-spawn-archive-deleted: delete the orphaned overlay, do not keep it

The old spawn-archive visibility overlay had no reader or writer and was removed. A future archive feature must be designed against current listing and history behavior.

## D-unknown-keys-from-pydantic: Pydantic does the traversal for unknown-key warnings

Walk validated model values to detect unknown keys instead of maintaining a second traversal of type annotations. A parallel traversal drifted on dict-root models and produced false warnings. The warning pass uses a temporary allow-extra validation; making extras permanent would expose them as effective config. Its caller must follow with authoritative validation because a hard error suppresses warnings.

## D-telemetry-batch-durability: fsync once per batch, not per envelope

A batch is the durability unit. Per-envelope fsync multiplies I/O without improving the torn-tail contract; see [telemetry persistence](../architecture/telemetry/local-persistence.md).
