# principles/invariants — System Invariants

Invariants are properties the system guarantees unconditionally. They're checked on every PR touching the relevant subsystems. Violating them creates subtle bugs that compound across time.

## Launch Composition Invariants (I-1 through I-13)

This page is the canonical statement of the launch composition invariants (I-1 through I-13); colocated `lib/launch/AGENTS.md` and `.context/CONTEXT.md` restate the ones that bind code in that directory.

### Core Composition Guarantees

**I-1 — Single composition seam.** The prepare/bind pipeline in `launch/context.py` — `prepare_launch_surface()` then `bind_launch_context()` — is the sole place where `SpawnRequest + LaunchRuntime → LaunchContext`. No driving adapter constructs argv, env, permissions, or prompt assembly independently. There is no one-call production wrapper; `build_launch_context()` survives only as a test facade.

**I-2 — No bypass.** Driving adapters (primary CLI path, spawn subprocess path, CLI streaming-serve path) enter through the prepare/bind seam or its spawn wrappers (`compose_spawn_launch_surface()`, `bind_spawn_launch_context()`, `SpawnApplicationService.prepare_spawn()`). They may not call the composition pipeline stages (`resolve_policies`, the permission-policy build, `compose_run_prompt_text`, etc.) directly.

**I-3 — DTO discipline.** `SpawnRequest` and `LaunchRuntime` are frozen Pydantic with JSON-safe field types. `LaunchContext` is a frozen dataclass. No `arbitrary_types_allowed`, no `Path` on `SpawnRequest`, no pre-composed intermediate DTOs that cache derived state. DTOs carry inputs; the factory produces outputs.

**I-4 — One identity pipeline, concluded after teardown.** Every runner binds the finalized entry key once before exec (`bind_entry`). It feeds owned live session IDs to `NativeRun.observe`, and runs `conclude_native_run()` once per turn, after the child exited and its teardown joined. The conclusion runs in a fixed order: post-exit IDs → adapter `observe_after_exit` → entry check → exit chat → one `run_boundary` write → failure record or attribution. Only the turn's first owned signal can contradict a pre-exec fact; every later ID is diagnostic. Runners carry no identity code of their own ([native session binding](../architecture/native-session-binding.md#runner-pipeline)).

**I-5 — Complete at construction.** `LaunchContext` is complete and immutable when the factory returns. No fields populated after construction by the caller.

**I-6 — Resolve before persist.** Composition runs before the spawn row is created: streaming-serve through `prepare_spawn()`, the spawn subprocess path through `build_create_payload()`. A resolution failure (bad model, missing profile) produces no spawn row — no phantom active spawns.

**I-7 — Resolved values in the row.** The spawn row stores real resolved model/agent/harness, never placeholder strings like `"unknown"`. Empty string (`""`) is a valid resolved model value: it can come from a model-optional profile or from Mars clearing an incompatible model after a stronger harness override won. In both cases, the harness uses its own default. `"unknown"` is not a valid resolved value — it is a placeholder and an I-7 violation. See [model-resolution: model optional](../decisions/model-resolution.md#model-optional-empty-model) for the model-optionality decision.

**I-8 — Environment overrides from context.** `ConnectionConfig.env_overrides` is populated from `LaunchContext.env_overrides`. Harness subprocesses inherit the correct `MERIDIAN_*` env from the composed context, not from ambient environment.

**I-9 — No surface-specific fields in adapters.** Adapters implement `project_content()` for surface-level content routing. Surface-conditional logic belongs in the composition factory, not in adapter implementations.

**I-10 — Fork materializes after row exists.** `fork.py:materialize_fork()` is called only after the spawn row is created. Fork creates a new harness session from the parent — that side effect must be auditable.

**I-11 — Per-adapter `SpawnParams` accounting.** Every adapter lists every `SpawnParams` field in its own `consumed_fields` or `explicitly_ignored_fields`, and no stale names. Harness bootstrap raises `ImportError` on any adapter's drift. The check is per adapter: one adapter consuming a field does not cover the others. This prevents adapter drift when `SpawnParams` gains new fields.

**I-12 — The runner holds `launch.lock` for its whole life.** It takes `spawns/<id>/launch.lock` exclusively before the row is published (a `--bg` launcher hands it to the worker) and holds it through `running` and `finalizing` until it exits. Every status reader derives liveness from that lock alone ([derived status](../architecture/state-system/derived-status.md)). This replaced the heartbeat in identity-v2 PR 3.

**I-13 — Warnings channel.** `LaunchContext.warnings` is the sole channel for composition warnings. Adapters that silently transform inputs (dropping a skill, ignoring a reference file) must surface the drop as a `CompositionWarning` in `LaunchContext.warnings`.

---

## State Invariants

### Durable File State

Session events are append-only JSONL: events are never deleted or rewritten in place, and a row counts only once its newline is written (a torn tail is withheld on read and repaired by the next append). Spawn state uses per-spawn `state.json` files: each replacement is atomic, and all mutations coordinate through a stable per-spawn lock at `locks/spawns/<id>.lock`. Both formats preserve the crash-only property, but only session state is currently replayed from JSONL.

### Mutate-Under-Lock Seam

Every mutation of authoritative state goes through a locked read-modify-write seam: acquire a stable lock, re-read current state, apply a pure mutator, write atomically. This shape applies to spawn records, work items, hook intervals, scope projections, autosync transactions, and published-spawn deletion. The seam is a behavior-preserving contract that planned future store rewrites inherit.

### Spawn Artifact Lifetime

A spawn-owned artifact cannot be created or changed after its published
`state.json` row is gone. Deletion and parent-creating artifact mutations
coordinate through the stable external per-spawn lock; the writer re-reads the
row only after acquiring that lock. Heartbeats and connection startup never
create a missing spawn directory. Late diagnostics fail closed rather than
resurrecting a partial aggregate.

### Stable Lock Inodes

All coordination locks live under `locks/<domain>/` outside the directories they protect. Lock inodes are never unlinked except through `unlink_validated_lock()`: a fresh, non-reentrant exclusive acquisition that validates `fstat` matches `stat(path)`, unlinks, and releases immediately. POSIX `flock` is per-open-file-description, not per-path: unlinking a lock file while another process holds it creates split-brain where two processes hold "the lock" on different inodes. Unlinking while the lock remains held afterwards (the forbidden pattern) is never used. Orphaned per-spawn locks are GC'd by `lock_gc.py`; cleaned session locks are unlinked by `cleanup_stale_sessions`.

### Lock-Order Invariants

When multiple locks are needed: `spawns_flock` before per-spawn lock before scope-projection lock. `delete_published_spawn()` checks for pending cleanup claims before deletion; pruning acquires `spawns_flock` first.

The work store's lock and the session lock pair are never held together, in either order; a command that changes both runs them as sequential steps, work store first ([D-one-projection-per-owner](../decisions/session-work-links.md#d-one-projection-per-owner)).

### Atomic Writes

Every file replacement uses the dependency-neutral `lib/platform/atomic.py:atomic_replace()`, which delegates through `state/atomic.py:atomic_write_text()` for runtime state. Permission policy: `preserve` for user-owned files, `0600` for runtime state. The conformance guard (`tests/contract/test_state_write_conformance.py`) rejects raw writes to authoritative state at CI.

### Conformance Guard

`tests/contract/test_state_write_conformance.py` is a repo-wide AST test that rejects direct `Path.write_text`, `Path.write_bytes`, and `open(..., "w")` calls on authoritative state paths. One documented allowlist entry (telemetry cooldown marker); stale entries are detected. New raw writes to state files are CI-rejected.

### Projection Authority Rule

Terminal spawn status is determined by event projection following these rules, in order:

1. **First authoritative wins.** Once an authoritative origin (`runner`, `launcher`, `launch_failure`, `cancel`) sets terminal status, a later authoritative finalize cannot overwrite it.
2. **Authoritative overrides reconciler.** A `reconciler`-origin terminal status can be overridden by a later `runner`-origin event. The runner is strictly more informed — it was actually executing the harness.
3. **Reconciler-vs-reconciler: first wins.** Two concurrent reconciler calls cannot revise each other.
4. **Metadata merges from all events.** Cost, tokens, and duration accumulate from every finalize event regardless of authority.

**Why:** Before identity-v2 PR 3 the reaper could stamp an orphan before the actual runner reported back, and the authority rule let the runner's ground truth win. Now the reaper writes only after claiming a dead runner's lock, so for lock-backed rows rule 2 no longer arises from the runner; PR 4 deletes the lattice.

### Status Transition Graph

```
queued    → running | succeeded | failed | cancelled
running   → finalizing | succeeded | failed | cancelled
finalizing → succeeded | failed | cancelled
```

`queued → finalizing` is NOT allowed. `mark_finalizing()` is a CAS operation — it appends `status=finalizing` only when current status is exactly `running`. It is a status marker for observers; liveness comes from the lock.

---

## Terminal Event Outcome Invariant

`TerminalEventOutcome` enforces at construction that a `succeeded` outcome
requires `exit_code=0` and `error=None`. Constructing a succeeded outcome with
a nonzero exit code or an error string raises `ValueError`. This makes the
invalid state unrepresentable rather than relying on downstream checks.

**Why:** Before this invariant, Codex `turn/completed` with `turn.status=failed`
was normalized to succeeded/0 because the semantics port did not inspect the
nested turn payload. The lifecycle resolver then persisted `succeeded` with the
upstream error string intact -- a contradictory state that misled callers. The
construction-time check eliminates this class of bug across all harnesses, not
just Codex.

**Effect:** Every call site that constructs a `TerminalEventOutcome` must
classify the outcome correctly before construction. A harness adapter that
attempts to report succeeded with an error will fail immediately at the
semantics layer rather than silently propagating contradictory state.

See [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) for
the per-harness terminal status semantics.

---

## Harness Strategy Map Invariant

Every `SpawnParams` field must be either in the adapter's `STRATEGIES` dict or in `_SKIP_FIELDS`. Missing fields raise `ValueError` at command-build time, not at call time. This is an early-detection invariant — it catches adapter drift at test time or first invocation, not in production.

**Effect:** Adding a new field to `SpawnParams` requires updating every adapter's strategy map. The error message names the missing field, so the fix is mechanical.

---

## Depth Gating Invariant

Reaper side effects (`reconcile_active_spawn`: persisting a dead launch's outcome and cleaning its scopes) run only when `_MERIDIAN_DEPTH` is absent, empty, or `0`. Non-empty, non-zero depth values (including malformed values like `"garbage"`) skip them. The gate is inside `reconcile_active_spawn`, so every call site inherits it from one place. Status reads never write, at any depth.

**Effect:** Nested spawns never trigger spawn state mutations in the parent's event store. The parent process owns reconciliation.

---

## Plugin API Boundary Invariant

Built-in commands import only from `src/meridian/plugin_api/`. They do not import from `src/meridian/lib/ops/` or `src/meridian/lib/launch/` directly. This enforces the plugin boundary — external plugins that follow the same import rule will behave identically to built-ins.

---

## Doc-Health Checks Ignore Fenced Code Block Contents

`meridian kg check` (link analysis) and `meridian mermaid check` (diagram validation) both operate on the markdown document structure, not on the text inside fenced code blocks.

**What this means in practice:**

- Links written inside a fenced block are not resolved or checked for validity.
- Actionable review markers are blockquote callouts whose line starts with `> [!FLAG]` after optional whitespace; inline-code and prose mentions of `[!FLAG]` are examples, not findings.
- Conflict markers, shell snippets, and future doc-health patterns inside fences are silently ignored unless a check explicitly opts into code-content analysis.
- Mermaid validation only processes fences whose language tag is `mermaid`; other fenced blocks are scanned for link extraction and then skipped.

**Why:** Fenced code blocks are literal or example regions — they document what something looks like, not what the document says. A usage example of the FLAG syntax, a markdown snippet showing a broken link, or a conflict marker inside a code block is part of the instructional content, not an error in the document itself. Treating code-block contents as actionable findings would produce false positives on every KB page that demonstrates syntax.

**Opt-in:** A future check that explicitly analyzes code-block content (e.g., validating shell commands, checking example links) may opt in by inspecting the fenced-block payload directly. The default scan path must never treat code-block interior as document surface.

---

## Cross-References

- [principles/design-principles.md](design-principles.md) — the design principles these invariants enforce
- [architecture/launch-system.md](../architecture/launch-system.md) — launch composition architecture where I-1 through I-13 apply
- [architecture/state-system/overview.md](../architecture/state-system/overview.md) — state model where JSONL session state, per-spawn `state.json`, and projection authority invariants apply
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — spawn status transitions and crash recovery
