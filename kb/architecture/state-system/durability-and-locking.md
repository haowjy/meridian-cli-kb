# Durability and Locking


Every file write goes through one of four patterns:

**JSONL append** (`state/jsonl.py`, the only JSONL read/repair/append module; `event_store.py` calls it): acquire `lock_file()` on `.flock` sidecar → repair any torn tail → append line → release. One rule governs every JSONL file: **a row exists when its newline does.** `iter_jsonl()` yields only newline-terminated rows, so a torn final row stays invisible until the next append commits it; malformed complete rows are skipped with a debug diagnostic, and readers that must fail closed (archive receipts) pass `on_malformed="raise"`. `append_durable_jsonl_line()` takes one row, owns its newline and rejects embedded CR/LF; `append_durable_jsonl_lines()` is the batch form (one repair, one write, one `fsync`), used by the telemetry sink and the session writer. If the process dies mid-append, the next locked append repairs the torn tail before writing: a complete row missing only its delimiter gets its newline; a genuinely torn partial row is dropped via atomic inode replacement (so unlocked readers never splice a fabricated hybrid event). The O(1) fast path (check last byte for newline) avoids a full-file read on clean tails. Launch-boundary events, permission journals and control-action journals use `append_durable_jsonl_line`. Session rows go only through the append seam, `session_store.append_chat_rows` / `append_native_rows` (#651, [D-session-read-seam](../../decisions/state.md#d-session-read-seam)): it takes the history mutation lock (shared) and the session lock, refuses to mutate inert historical records (by the event's reserved `historical-` generation prefix in O(1), since #588 in v0.10.4), marks the `sessions` source dirty for the metadata index, and appends the whole batch at once. Selection and observation dedup run inside that lock epoch and still read the log while holding it, see [session state](session-state.md#sessionsjsonl-is-one-global-log). Spawn state uses atomic overwrite.

**JSON sidecar document** (`state/json_doc.py`): small JSON sidecars (session leases, primary metadata, process scopes, cleanup claims, spawn scopes, idle state, Pi lifecycle, autosync state and conflicts) are Pydantic models persisted only through `read_model()` / `write_model()` / `mutate_model()`. A missing, truncated or invalid file reads as `None` and the next write repairs it; writes are atomic with mode `0600`; `mutate_model()` requires the caller's lock and re-reads under it. A new sidecar uses this seam, not a hand-rolled read → validate → lock → replace.

**Atomic file replacement** (`lib/platform/atomic.py:atomic_replace()`): the dependency-neutral platform primitive that `state/atomic.py`, `plugin_api/fs.py`, autosync, and the Codex streaming rewriter all delegate to. Writes to a same-directory temp, optionally fsyncs, then `os.replace()`. Permission policy: `permissions="preserve"` (default) keeps existing file mode; `permissions=0o600` enforces strict mode for runtime state. `AtomicReplaceDurabilityError` surfaces post-commit fsync failures so callers know the write is committed but not yet durable.

State-facing writes use `state/atomic.py:atomic_write_text()` which sets mode `0600` for runtime state. User-owned project files and context work-item metadata use the preserve-mode platform atomic writer.

**Atomic directory publication** (`state/atomic.py:atomic_publish_dir()`): rename a complete same-volume staging directory into a destination that must not exist, then fsync the publication parent.

**Work item renames:** `work-items.rename.intent.json` is written before any rename begins. Leftover intent is replayed on startup/reconciliation — crash-safe two-phase rename.

### Conformance Guard

`tests/contract/test_state_write_conformance.py` is a repo-wide AST test rejecting raw file writes (`Path.write_text`, `Path.write_bytes`, `open(..., "w")`) to authoritative state. It enforces that all state mutations route through the atomic primitives. A documented single-entry allowlist covers the one justified exception (telemetry cooldown marker). Stale allowlist entries are detected. The failure message names the offending call site and guides toward the correct primitive.

## Platform Locking

`platform.locking.lock_file(path, mode, timeout, reentrant)` is the single cross-process locking primitive:

- **POSIX:** `fcntl.flock(LOCK_EX | LOCK_SH)` — advisory, kernel-backed
- **Windows:** `msvcrt.locking(LK_NBLCK, 1)` with retry loop (50 ms sleep)

**Modes:** `exclusive` (default) or `shared`. Shared mode uses `LOCK_SH`; a held shared lock cannot be upgraded to exclusive in place.

**Reentrancy:** thread-local by default. A thread that already holds the lock re-enters safely; the OS lock releases only on outermost exit. Non-reentrant mode (`reentrant=False`) is used for mutation seams where nesting would let an inner run invalidate the outer's state snapshot.

**Fork safety:** acquired handles are tracked in a process-wide registry. On `fork()`, the child closes every inherited descriptor (releasing the parent's open-file-description lock without explicit unlock) and clears the reentrancy state. Release-window descriptors are also registered so a fork during the gap between OS release and handle close does not leak.

**Stable lock inodes with GC seam:** all coordination locks live under `locks/<domain>/` outside the directories they protect. This prevents the split-brain failure where one process unlinks a lock file and creates a new inode while another still holds the old one (POSIX `flock` is per-open-file-description, not per-path). Lock inodes are never unlinked except through a validated GC seam: `unlink_validated_lock()` unlinks only while holding a fresh, non-reentrant exclusive flock on the inode currently linked at that path, immediately before release. The acquire-side revalidation loop (`open → flock → compare fstat(fd) vs stat(path) → retry on mismatch`) makes this provably split-brain-free. Two GC call sites use this primitive: `lock_gc.py` sweeps orphaned per-spawn locks (four classes under `locks/`) when the corresponding spawn directory no longer exists; `cleanup_stale_sessions()` unlinks cleaned session locks before release. Both run on episodic paths (doctor, prune, cleanup), never on hot paths. The forbidden pattern — unlinking while a lock remains held afterwards (e.g. inside a reentrant context) — is never used.

`spawn_aggregate.py` owns published-row lifetime coordination. Artifact writers
use `mutate_published_spawn_artifact()` to check publication under the spawn
lock; deletion uses `delete_published_spawn()` to acquire the spawn lock then
the scope-projection lock, check for pending cleanup claims, and remove the
spawn directory. Cross-leaf spawn operations belong in the aggregate, not in
either persistence leaf.

### Lock-Order Invariants

When multiple locks are needed, acquire in this order to prevent deadlocks:

1. `spawns_flock` (global spawn-ID allocation and publication)
2. Per-spawn lock (`locks/spawns/<id>.lock`)
3. Scope-projection lock (`locks/process-scopes/<id>.lock`)

`delete_published_spawn()` (in `spawn_aggregate.py`) acquires the per-spawn lock then the scope-projection lock, and checks for pending cleanup claims before deletion. Pruning acquires `spawns_flock` first.

### Project-Lifetime Gate

`~/.meridian/projects/.locks/<project-id>.lock` sits outside the deletable project root. Sessions hold a **shared** lock for their lifetime; global pruning acquires **exclusive** + revalidates the target before removal. This prevents pruning from destroying a runtime root while sessions hold spawn locks inside it.

See `lib/platform/locking.py` for implementation details.


## Related Pages

- [State system overview](overview.md) — state roots and subsystem map
- [Spawn state](spawn-state.md) — mutation and deletion seams using these primitives
- [Reconciliation](reconciliation.md) — repair operations coordinated by spawn locks
- [State decisions](../../decisions/state.md) — rationale for locking and write policies
