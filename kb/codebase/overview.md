# codebase/overview — Codebase Domain

Codebase pages orient contributors to where implementation concerns live and which modules own which responsibilities.

## Pages

- [guide.md](guide.md) — Where to start for common change types and subsystem ownership.
- [harness-adapters.md](harness-adapters.md) — Harness capability matrix, primary event scope, and cross-harness comparison.
- [observability.md](observability.md) — Telemetry spine framing and relationship to durable state log.
- [tools.md](tools.md) — KG analysis, Mermaid validation, and Markdown extraction.
- [work-items.md](work-items.md) — Cross-module work attachment, scratch directories, rename propagation, hook coordination.
- [session-operations.md](session-operations.md) — `session log / search / export`: compaction segments, transcript extraction workflow, user-facing command guidance.
- [session-log-rendering.md](session-log-rendering.md) — Internal rendering pipeline: ToolCall normalization, clean vs raw output, `--raw`/`--no-truncate` flag design, content pipeline order.
- [test-determinism.md](test-determinism.md) — Test determinism infrastructure: `AsyncDeterminism` + `FakeClock`, `process_race.py` cross-process harness, advance-past-boundary principle, behavioral vs safety-bound timeouts.
