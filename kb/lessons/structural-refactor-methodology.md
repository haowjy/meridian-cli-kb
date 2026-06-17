# Lessons: Structural Refactor Methodology (PR #328)

Lessons from the thermo-nuclear review follow-through on PR #328 (`feature/spawn-capability-injection`, 2026-06). The review fan-out (7 reviewers) returned request-changes on "the new abstractions are directionally right but didn't go all the way." Rather than defer, all of it was taken in one follow-through wave. This is the methodology that made it possible.

---

## Single-Source Registry Over Parallel Maps

**The problem:** Help content, root-help curation, app lookup, and command registration were spread across five parallel data structures:
- `AGENT_DESCRIPTION_OVERRIDES` — agent-mode root descriptions
- `AGENT_ROOT_COMMANDS` — which groups appear in agent root help
- `_HUMAN_COMMAND_ORDER` — human-mode ordering
- Static app dict — group App objects
- Registration maps — lazy-registration bucket mapping

Adding a new command group required updating 3–5 separate lists. Forgetting one meant the group was invisible in agent mode but listed in human mode, or listed but unorderable.

**The fix:** `CommandGroupSpec` registry in `cli/command_groups.py`. One frozen dataclass per group. All derived maps (`AGENT_ROOT_COMMANDS`, `HUMAN_ROOT_ORDER`, `COMMAND_REGISTRATION`, `AGENT_DESCRIPTION_OVERRIDES`, `AGENT_VISIBLE_SUBCOMMANDS`, `AGENT_HELP_SUPPLEMENTS`) are auto-generated from the single source:

```python
AGENT_ROOT_COMMANDS = tuple(
    name for name, spec in COMMAND_GROUP_SPECS.items() if spec.agent_root
)
```

**The rule:** Any time you have N parallel lists that must stay consistent, replace them with one registry and auto-derive the lists. The registry is the write surface; the lists are read-only projections.

**Verification tactic:** Rendered help matrix was byte-identical before and after. This is the standard for structural-only refactors — diff the output, not the code.

---

## Typed Carriers Over `**kwargs`

**The problem:** The spawn lifecycle start call had ~20 keyword arguments. Adding a field meant updating every call site but getting no help from the type checker if one was missed. `announce_started` was an alias, not a distinct concept.

**The fix:** `SpawnReservation` — a single typed dataclass. `start(reservation)` maps fields in one write site. The ~20-kwarg signature is gone.

**The rule:** When a function signature grows past 5 params, replace with a typed carrier dataclass. The carrier is the contract; the function body is the implementation. The type checker enforces the contract at every call site.

---

## Pure Value Objects + External Rendering

**The problem:** `WorkScope` (before the refactor) had warning-label wording and display logic embedded in the state layer. The raw-`Path` overload allowed a bare directory to masquerade as a scope — fake-scope fabrication.

**The fix:** `WorkScope` is a frozen dataclass with `kind`, `identifier`, `root`. No display logic, no warnings. The ops layer reads the scope and decides what to say. The raw-`Path` overload is deleted — callers must use `resolve_bound_work_scope()` or `resolve_work_scope_from_parts()`.

**The rule:** State objects answer "what is true?" Rendering objects answer "what do we say?" Never mix them.

---

## One Prompt IR, No Mirrors

**The problem:** Bind refresh (re-projection after composition) had a `ContextEnvRefreshPlan` mirror struct. `ChildEnvContext` had its own scope-precedence engine, separate from `ResolvedContext`. Two paths could diverge.

**The fix:** One prompt IR — `PreparedLaunchContent` carries the `ComposedLaunchContent` and bind reprojection is a single `dataclasses.replace`. `ChildEnvContext` is a projection of `ResolvedContext` (no second scope-precedence engine). Deleted the mirror struct.

**The rule:** Before creating a mirror struct for a different use case, ask: can the original struct serve both? If yes, project (don't duplicate). If no, make the projection explicit (`from_resolved_context()`) so readers can trace the relationship.

---

## Adapter-Owned Phrasing, No HarnessId Conditional

**The problem:** `build_guidance_blocks()` had a `HarnessId` conditional — "if Claude, phrase the spawn contract this way; else, that way." Adding a harness required editing guidance code.

**The fix:** `RunPromptPolicy.spawn_usage_contract_variants()` returns a `SpawnUsageContractVariants`. `build_guidance_blocks()` receives a pre-built contract string and composes it unconditionally. The adapter owns the phrasing; the guidance system owns the composition.

**The rule:** When phrasing varies by harness, make the harness provide the phrasing (add to its protocol), don't branch on harness identity in shared code.

---

## Test Hygiene: Behavior-Level Contracts Over Prose-Pinned Tests

**The problem:** Help tests were pinned to prose strings — changing help wording broke tests. Help tests poked private state (`service._hooks`). The 870-line context-env test file had no internal structure. Scope-resolution tests duplicated scenarios across files.

**The fix:**

1. **Behavior-level contract suites:** Help tests assert structure (groups present, subcommands visible) and agent-mode filtering, not exact prose. The contract is "these groups are visible in agent mode with these subcommands," not "the help text says exactly X."

2. **Public seams for tests:** `register_hook()` added as a public method on `SpawnLifecycleService`. Tests stop poking `service._hooks`. When the internal structure changes, tests don't break.

3. **Split by contract:** The context-env test was split into `test_child_env_context.py`, `test_launch_context_env_runtime_overrides.py`, `test_launch_context_env_task_cwd.py`, and `test_launch_context_env_work_scope.py`. Each file tests one contract.

4. **Table-driven consolidation:** Duplicated scope-resolution scenarios merged into one table-driven suite. The table is the contract; one test function iterates it.

**The rule:** Tests assert behavior contracts, not implementation details. If a refactor changes prose but not behavior, tests should not need updating. If a test needs `_private` member access, add a public seam instead.

---

## Verification Discipline for Structural Refactors

The PR used a two-wave verification pattern:

1. **First wave (feature work):** 1542 passed, 2 skipped; ruff clean; pyright 0 errors.
2. **Second wave (structural follow-through):** Fan-out to correctness reviewer (76 targeted tests, no regressions) + prober (full runtime: help renders, every command invocable, spawn lifecycle, capability gate, named/ambient scope).

**The rule:** A structural refactor that touches the help system, spawn lifecycle, and context projection needs two independent verification paths: a correctness reviewer scanning for regressions, and a prober running the actual CLI surface end-to-end. Neither alone is sufficient.

---

## Cross-References

- [arch-refactor-pitfalls.md](arch-refactor-pitfalls.md) — implementation pitfalls from earlier architecture refactor (different PR, complementary lessons)
- [source-simplification.md](source-simplification.md) — deletion-first simplification tactics
- [state-design-lessons.md](state-design-lessons.md) — state-design tradeoffs
