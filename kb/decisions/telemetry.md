# Telemetry

Telemetry supplements authoritative state rather than duplicating it: two durable copies of lifecycle and session data would drift. The mechanism is in [telemetry architecture](../architecture/telemetry/).

## Reader pipelines

Future error reporting and feature tracking read local segments rather than adding sinks to the write path. This keeps their backpressure and consent boundaries out of event emission.

## Existing stores

Keep full lifecycle and conversation records in their authoritative stores; telemetry supplies diagnostic events and correlation.

## Flat segments

Per-process segments let retention and readers discover files without coordinating domain-specific directories.

## Segment identity

A logical owner accompanies the PID because the OS can reuse PIDs. A spawn's segment remains protected while its spawn is live, not merely while a process with the same PID exists.

## Project storage

CLI and spawn events share a project telemetry directory so project queries see both.

## Early buffering

Early events are buffered before the project root is known. Upgrading the sink in place avoids losing events from threads still holding the old router.

## Event concerns

A registry maps event names to concerns; repeating fixed tags on every event wastes space.

## Envelope

The local envelope omits data derivable at read time and fields needed only by a future remote pipeline.

## Retention

Protect live spawn segments through finalization; harness exit alone does not mean the runner has stopped writing.

## Global discovery

Cross-project queries take a point-in-time directory snapshot. They are a debugging path, so a registry and its stale-registration failure mode are unnecessary.

## Byte offsets {#d-tel-binary-mode}

Tail cursors use byte offsets, avoiding text-mode translation and multibyte seek ambiguity.
