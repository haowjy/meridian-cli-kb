# Compiler Pipeline

Mars compiles selected package content into canonical and harness-native outputs. Selection belongs to target construction: preflight and lowering consume the same selected hooks and MCP servers, so they cannot disagree about which source owns an output. Lock evidence keeps failed removals retryable instead of silently surrendering ownership.

See `src/compiler/AGENTS.md` in mars-agents for the pipeline and [package management decisions](../../decisions/package-management.md) for the rationale.
