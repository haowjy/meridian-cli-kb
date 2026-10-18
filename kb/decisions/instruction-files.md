# Decisions: Agent Instruction Files

Decision records for which files hold per-directory agent instructions. See
[../decisions.md](../decisions.md) for the chronological index.

---

## D-agents-md-only: AGENTS.md is the instruction file; CLAUDE.md only for Claude-specific lines

**Decision (2026-10-18):** Every directory's agent instructions live in
`AGENTS.md`. Repos keep no `@AGENTS.md` `CLAUDE.md` mirrors, and
`meridian qi claude-md-fix` is deleted. A `CLAUDE.md` exists only for lines that
apply to Claude Code alone, and its first line is `@AGENTS.md`.

**Why:** Claude Code reads `AGENTS.md` itself. On Claude Code 2.1.302, a root
`AGENTS.md` loads at startup and a nested one loads when Claude reads a file in
that directory, with no `CLAUDE.md` present. When a directory has a `CLAUDE.md`,
Claude reads it *instead of* `AGENTS.md`, so a Claude-specific `CLAUDE.md` that
omits the import silently drops the shared instructions. Pi also reads
`CLAUDE.md` as an alternative context file, so mirrors added nothing there.

**Rejected:** keeping mirrors for older Claude Code versions. Meridian has no
users to protect, and the mirrors had already drifted (mars-agents mirrored 1 of
17 `AGENTS.md` files; meridian-dev-workflow ignored `CLAUDE.md`).

**Where it lives:** `/qi-layer` in meridian-base (v0.11.6) and its
project-knowledge bootstrap; creative-writing-skills' `project-setup` writes
project conventions to `AGENTS.md` (v0.5.13).

**Revisit if:** a harness Meridian supports reads only `CLAUDE.md`.
