# Telemetry event catalog

The source checkout's `src/meridian/lib/telemetry/events.py` owns the event registry and envelope. Concerns are derived from event names rather than serialized on each record, keeping the records smaller and emitters consistent. See [telemetry decisions](../../decisions/telemetry.md#event-concerns).

