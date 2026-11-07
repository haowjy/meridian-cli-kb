# Harness abstraction

A harness is an external AI runtime integrated through a validated `HarnessBundle`. Launch policy resolves intent; the harness layer translates it to native mechanisms and normalizes events. It must not re-run policy precedence or model routing. See the source checkout's `src/meridian/lib/harness/AGENTS.md` and [launch architecture](../architecture/launch-system.md).

Connections own live communication; extractors recover session and report data. Event semantics distinguish parent activity from nested child activity so a child's events cannot complete or supply the parent's report. A connection's delivery result distinguishes harness acknowledgement from a completed write: neither proves the model read the message. See [primary message delivery](../decisions/primary-inject.md).
