# Telemetry

Telemetry supplies diagnostic events and correlation across subsystems; lifecycle state and session records stay in their authoritative stores. Two durable copies of lifecycle data would drift, and emission is best effort, so nothing may treat telemetry as a second state store. Registry, envelope and reader: the source checkout's `src/meridian/lib/telemetry/`.

## Local segments

- **Per-project, per-process segments in one flat directory.** CLI and spawn processes write to the same project telemetry directory, so a project query sees both, and retention and readers find files without coordinating domain-specific directories.
- **Segment names carry a logical owner beside the PID.** The OS reuses PIDs, so a spawn's segment stays protected while its spawn is live, through finalization, not merely while some process with that PID exists. Harness exit alone doesn't mean the runner has stopped writing.
- **A batch is the durability unit:** one tail repair, write and fsync. Fsync per envelope multiplies I/O without improving the torn-tail guarantee; a crash leaves at most a torn final batch.
- **Early events are buffered** before the project root is known, and the buffer upgrades its sink in place so threads still holding the old router don't lose events.
- **Small envelopes.** Concerns are derived from event names through a registry rather than repeated on every record, and the envelope omits anything derivable at read time or needed only by a future remote pipeline.

## Readers

Cross-project queries take a point-in-time directory snapshot. They are a debugging path, so a registry and its stale-registration failure mode aren't worth it. Tail cursors use byte offsets, which avoid text-mode translation and multibyte seek ambiguity.

Error reporting and feature tracking aren't built. They are meant to read local segments rather than join the write path, keeping their backpressure and consent outside event emission.

Installation is split between CLI startup and background operation paths ([CLI startup](codebase/guide.md#cli-startup)).
