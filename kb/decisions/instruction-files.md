# Agent Instruction Files

Shared per-directory instructions live in `AGENTS.md`; `CLAUDE.md` is reserved for Claude-specific instructions and begins with `@AGENTS.md`. Claude reads `CLAUDE.md` instead of `AGENTS.md` when both exist, so omitting the import drops shared instructions. See the `qi-layer` skill for the full convention.
