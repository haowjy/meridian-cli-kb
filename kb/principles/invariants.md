# principles/invariants — System Invariants

Invariants are properties the system guarantees unconditionally. They're checked on every PR touching the relevant subsystems. Violating them creates subtle bugs that compound across time.

## Launch Composition Invariants (I-1 through I-13)

The full invariant specification lives at `.meridian/invariants/launch-composition-invariant.md`. Summarized here for orientation.

### Core Composition Guarantees

**I-1 — Single composition seam.** `build_launch_context()` in `launch/context.py` is the sole place where `SpawnRequest + LaunchRuntime → LaunchContext`. No driving adapter constructs argv, env, permissions, or prompt assembly independently. If a driving adapter needs a launch context, it calls the factory.

**I-2 — No bypass.** Driving adapters (primary CLI path, spawn subprocess path, REST app path, CLI streaming-serve path) call `build_launch_context()`. They may not call the composition pipeline stages (`resolve_policies`, `resolve_permission_pipeline`, `compose_run_prompt_text`, etc.) directly.

**I-3 — DTO discipline.** `SpawnRequest` and `LaunchRuntime` are frozen Pydantic with JSON-safe field types. `LaunchContext` is a frozen dataclass. No `arbitrary_types_allowed`, no `Path` on `SpawnRequest`, no pre-composed intermediate DTOs that cache derived state. DTOs carry inputs; the factory produces outputs.

**I-4 — Session ID observation is post-execution only.** `harness_adapter.observe_session_id()` is called once, after process exit, in the primary path. No mid-execution session ID extraction.

**I-5 — Complete at construction.** `LaunchContext` is complete and immutable when the factory returns. No fields populated after construction by the caller.

**I-6 — Resolve before persist (REST and streaming-serve paths).** `build_launch_context()` is called before the spawn row is created. A resolution failure (bad model, missing profile) produces no spawn row — no phantom active spawns.

**I-7 — Resolved values in the row.** The spawn row stores real resolved model/agent/harness, never placeholder strings like `"unknown"`. Empty string (`""`) is a valid resolved model value (model-optional profile; harness uses its own default). `"unknown"` is not a valid resolved value — it is a placeholder and an I-7 violation. See [decisions/model-resolution.md](../decisions/model-resolution.md) for the model-optionality decision.

**I-8 — Environment overrides from context.** `ConnectionConfig.env_overrides` is populated from `LaunchContext.env_overrides`. Harness subprocesses inherit the correct `MERIDIAN_*` env from the composed context, not from ambient environment.

**I-9 — No surface-specific fields in adapters.** Adapters implement `project_content()` for surface-level content routing. Surface-conditional logic belongs in the composition factory, not in adapter implementations.

**I-10 — Fork materializes after row exists.** `fork.py:materialize_fork()` is called only after the spawn row is created. Fork creates a new harness session from the parent — that side effect must be auditable.

**I-11 — Strategy map completeness.** Every `SpawnParams` field must be in an adapter's `STRATEGIES` dict or in `_SKIP_FIELDS`. Missing entries raise `ValueError` at build time. This prevents adapter drift when `SpawnParams` gains new fields.

**I-12 — Heartbeat covers full active window.** The heartbeat file is touched every 30 seconds for the full active window: both `running` and `finalizing` states. The reaper uses heartbeat recency as the primary liveness signal for `finalizing` spawns (PID probes are not consulted for `finalizing`).

**I-13 — Warnings channel.** `LaunchContext.warnings` is the sole channel for composition warnings. Adapters that silently transform inputs (dropping a skill, ignoring a reference file) must surface the drop as a `CompositionWarning` in `LaunchContext.warnings`.

---

## State Invariants

### Durable File State

Session events are append-only JSONL: events are never deleted or rewritten in place, and truncated lines are skipped on read. Spawn state uses per-spawn `state.json` files: each replacement is atomic, and external writers coordinate through the per-spawn `state.lock`. Both formats preserve the crash-only property, but only session state is currently replayed from JSONL.

### Atomic Writes

Every file replacement uses `atomic_write_text()`: write to a temp file, then `os.replace()`. This guarantees readers see either the old complete file or the new complete file — never a partial write.

### Projection Authority Rule

Terminal spawn status is determined by event projection following these rules, in order:

1. **First authoritative wins.** Once an authoritative origin (`runner`, `launcher`, `launch_failure`, `cancel`) sets terminal status, a later authoritative finalize cannot overwrite it.
2. **Authoritative overrides reconciler.** A `reconciler`-origin terminal status can be overridden by a later `runner`-origin event. The runner is strictly more informed — it was actually executing the harness.
3. **Reconciler-vs-reconciler: first wins.** Two concurrent reconciler calls cannot revise each other.
4. **Metadata merges from all events.** Cost, tokens, and duration accumulate from every finalize event regardless of authority.

**Why:** The reaper may stamp an orphan before the actual runner reports back. The authority rule ensures the runner's ground truth wins over the reaper's inference.

### Status Transition Graph

```
queued    → running | succeeded | failed | cancelled
running   → finalizing | succeeded | failed | cancelled
finalizing → succeeded | failed | cancelled
```

`queued → finalizing` is NOT allowed. `mark_finalizing()` is a CAS operation — it appends `status=finalizing` only when current status is exactly `running`. This prevents races between the runner and the reaper.

---

## Harness Strategy Map Invariant

Every `SpawnParams` field must be either in the adapter's `STRATEGIES` dict or in `_SKIP_FIELDS`. Missing fields raise `ValueError` at command-build time, not at call time. This is an early-detection invariant — it catches adapter drift at test time or first invocation, not in production.

**Effect:** Adding a new field to `SpawnParams` requires updating every adapter's strategy map. The error message names the missing field, so the fix is mechanical.

---

## Depth Gating Invariant

Reaper side effects (`reconcile_spawns`) run only when `MERIDIAN_DEPTH` is absent, empty, or `0`. Non-empty, non-zero depth values (including malformed values like `"garbage"`) skip reconciliation. The gate is inside `reconcile_active_spawn`, so every call site inherits it from one place.

**Effect:** Nested spawns never trigger spawn state mutations in the parent's event store. The parent process owns reconciliation.

---

## Op Layer Identity Invariant

Op API functions in `src/meridian/lib/ops/` do not read from `os.environ` to determine caller identity. Caller identity (`chat_id`, `spawn_id`) is supplied through `RuntimeContext` (the `ctx` parameter) by the caller — CLI callers via `RuntimeContext.from_environment()`, MCP callers via request context.

**Why:** Op functions are shared across CLI, MCP, and REST surfaces. An op that calls `os.getenv("MERIDIAN_CHAT_ID")` directly works only when a harness-session environment is present, silently producing wrong results for non-CLI callers.

**Check:** Any `os.getenv("MERIDIAN_*")` call inside `src/meridian/lib/ops/` is an invariant violation. Move the env-read to the CLI caller and pass the result through `RuntimeContext`.

---

## Plugin API Boundary Invariant

Built-in commands import only from `src/meridian/lib/plugin_api/`. They do not import from `src/meridian/lib/ops/` or `src/meridian/lib/launch/` directly. This enforces the plugin boundary — external plugins that follow the same import rule will behave identically to built-ins.

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
- [architecture/state-system.md](../architecture/state-system.md) — state model where JSONL session state, per-spawn `state.json`, and projection authority invariants apply
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — spawn status transitions and reaper logic
