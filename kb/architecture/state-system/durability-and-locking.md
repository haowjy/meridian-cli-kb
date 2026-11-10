# Durability and locking

See the source checkout's `src/meridian/lib/state/AGENTS.md` for write and lock order rules and [state decisions](../../decisions/state.md) for their rationale.

Locking is by inode, not path: unlinking and recreating a lock while another process holds the old inode permits two owners. A work-store lock and the session lock pair cannot be held together; doing so deadlocked work completion against session restore. See [work and checkout links](../../decisions/session-work-links.md#one-projection-per-owner).

