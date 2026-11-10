# Codebase Cleanup

Remaining cleanup work is tracked in [#569](https://github.com/haowjy/meridian-cli/issues/569).

## Spawn wait contention

Wait discovery must not answer from a stale index that could omit a new spawn. The contention fix shortens the session writer's lock hold; see [history index readers](history-storage.md#shorten-the-holder-dont-widen-the-wait).

## Pydantic does the traversal for unknown-key warnings

Walk validated model values to detect unknown keys instead of maintaining a second traversal of type annotations. A parallel traversal drifted on dict-root models and produced false warnings. The warning pass uses a temporary allow-extra validation; making extras permanent would expose them as effective config. Its caller must follow with authoritative validation because a hard error suppresses warnings.

## Fsync once per batch, not per envelope

A batch is the durability unit. Per-envelope fsync multiplies I/O without improving the torn-tail contract; see [telemetry persistence](../architecture/telemetry/local-persistence.md).
