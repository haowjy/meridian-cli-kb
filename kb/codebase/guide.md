# Codebase Guide

The source checkout's `src/meridian/AGENTS.md` and `.context/CONTEXT.md` describe the package layers and entry points. Subdirectory guidance owns the rules for changes there.

## Testing Strategy

See the source checkout's `tests/AGENTS.md` and [testing decisions](../decisions/testing.md). A source-tree lint may use a text prefilter only for identifier tokens: string literals and dotted names can have different source spellings with the same parsed meaning. Parsing every module for those rules costs about 0.75 seconds per gate.

