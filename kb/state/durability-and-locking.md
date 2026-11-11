# Durability and locking

Write order and lock order are in the source checkout's `src/meridian/lib/state/AGENTS.md` and `.context/CONTEXT.md`, and JSONL tail repair in `src/meridian/lib/state/jsonl.py`; this page holds the reasons.

## A newline commits a JSONL row

A valid final object without its newline stays invisible until a later append repairs the tail: showing it sooner would expose a write that was never acknowledged. Repair keeps a complete object and discards a torn one. It replaces the file's inode rather than truncating in place, because an unlocked buffered reader could otherwise splice old and new bytes into an event nobody wrote.

## Locks are inodes, not paths

Unlinking a held lock and recreating it lets two processes each hold "the" lock on different inodes. The validated removal contract is in `src/meridian/lib/state/AGENTS.md`. A project-lifetime gate that lives outside the deletable root lets pruning exclude active sessions without enumerating their locks.

- **No fixed lock stripes.** Collisions break reentrant exclusion, and a shared stripe would stop a session lock's availability from meaning that session is gone.
- **No generic store framework.** Stores share lock, re-read, mutate and atomic-write semantics, but a framework would have to parameterize their different paths, codecs and failure handling.
- **A work-store lock and the session lock pair are never held together.** Holding both deadlocked work completion against session restore. Each store keeps its own projection instead ([work items](work-items.md)).
