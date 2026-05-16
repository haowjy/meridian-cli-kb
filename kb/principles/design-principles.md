# principles/design-principles — Meridian Design Principles

These principles govern how Meridian is built and how it evolves. They're extracted from accumulated practice, not handed down from theory. Each has a reason and a manifestation — understanding both prevents the next person from undoing the decision unknowingly.

## 1. Separate Policy from Mechanism

*Raymond, Rule of Separation: separate policy from mechanism; separate interfaces from engines.*

**The principle:** Harness adapters are mechanism — how to launch Claude, Codex, or OpenCode. CLI commands are policy — what to do, which model, what output. Policy changes fast; mechanism stays stable.

**Why it matters:** When policy and mechanism are tangled, a business logic change requires touching the adapter, and an adapter change requires touching the CLI. Decoupling lets each evolve independently.

**In Meridian:** `src/meridian/lib/harness/` contains adapters with no awareness of user intent. `src/meridian/lib/ops/spawn/` and `src/meridian/cli/` contain policy. The composition factory (`launch/context.py`) is the seam between them.

---

## 2. Extend, Don't Modify

*Open/Closed Principle.*

**The principle:** New harness = one adapter file + registration. New package source = one mars config entry. New CLI command = one module. If a feature requires editing 10 files, the abstraction is wrong.

**Why it matters:** Every file touched in a feature is a file that can conflict, diverge, or be missed. Narrow extension points make changes auditable.

**In Meridian:** The `HarnessRegistry` maps IDs to adapter instances — adding a harness is `register()`. The strategy map invariant (`STRATEGIES` dict in each adapter) enforces that new `SpawnParams` fields are consciously handled everywhere without a grep across adapters.

---

## 3. Knowledge in Data, Not Code

*Raymond, Rule of Representation: fold knowledge into data so program logic can be stupid and robust.*

**The principle:** Agent capabilities live in YAML profiles, not procedural code. State lives in durable file formats, not in-memory objects. This keeps the system inspectable and harness-agnostic.

**Why it matters:** Logic encoded in code is invisible to external tools and requires re-reading source to understand. Data is inspectable, editable, and survives code changes.

**In Meridian:** Agent profiles are markdown with YAML frontmatter — loaded at launch, not compiled in. Session state remains append-only JSONL; spawn state now lives in per-spawn `state.json` files for O(1) reads. Both are readable with standard file tools and survive process death. Config lives in TOML files, not environment-variable patchworks.

---

## 4. Crash-Only Design

*Candea & Fox: crash-only software.*

**The principle:** Every write is atomic (tmp+rename). Every read tolerates truncation. There is no "graceful shutdown." If meridian is killed mid-spawn, the next `meridian status` detects and reports the orphaned state. Recovery IS startup.

**Why it matters:** Graceful shutdown is fragile — it requires the process to get enough time to clean up. Crash-only systems are correct under arbitrary interruption and make failure modes explicit.

**In Meridian:**
- Writes use `atomic_write_text()` (tmp + `os.replace()`)
- JSONL reads skip malformed lines silently; `state.json` writes use atomic replacement
- The reaper runs on every read path — no separate GC process needed
- Work item renames write an intent file first; leftover intent is replayed on startup

---

## 5. Progressive Disclosure

*clig.dev, Lengstorf.*

**The principle:** `meridian spawn "do the thing"` works with smart defaults. Power users override with `--model`, `--harness`, `--skills`. Don't force all-or-nothing configuration.

**Why it matters:** New users need to get started without reading docs. Expert users need access to every lever. These are different users and both matter.

**In Meridian:** `meridian spawn` without flags uses defaults from config and the default agent profile. `-m`, `-a`, `--skills`, `--approval`, and `--from` are progressive additions. Agent profiles encode expert defaults so team conventions don't require flags on every invocation.

---

## 6. Simplest Orchestration That Works

*Google Cloud AI patterns.*

**The principle:** Stay a thin coordination layer. Centralized spawn-and-report is enough. Don't build complex agent choreography until the simple model breaks. "Simplest" means least total complexity owned — lines maintained, platforms tested, failure modes debugged, contributors onboarded — not fewest imports.

**Why it matters:** Complex choreography is expensive to test, debug, and understand. The simplest model that solves the problem leaves room for evolution without accumulated debt.

**In Meridian:** Meridian does not build task graphs, retry trees, or agent state machines. It launches subagents, tracks their state, and exposes results. Orchestration logic belongs in agent prompts, not in the coordination layer.

---

## 7. Harness-Agnostic

**The principle:** One CLI, many runtimes. Meridian never assumes Claude, Codex, or any specific harness — adapters bridge the gap.

**Why it matters:** Harness lock-in makes migration expensive and prevents the system from routing to the best tool for each task.

**In Meridian:** `HarnessId` is a string. Capability flags (`HarnessCapabilities`) let the launch layer make harness-sensitive decisions without adapter-specific `if harness == "claude"` branches in shared code.

---

## 8. Files as Authority

**The principle:** All state is files — project-level under `.meridian/`, user-level under `~/.meridian/`. No databases, no services, no hidden state. If it's not on disk, it doesn't exist.

**Why it matters:** Files are observable, inspectable, portable, and survive process death. In-memory state is hidden, non-portable, and lost on crash.

**In Meridian:** Spawn state is per-spawn `state.json`; session state is a JSONL event stream. Config is TOML. Agent profiles are markdown. Nothing requires a running service to inspect system state.

---

## 9. Coordination, Not Control

**The principle:** Meridian provides structure (spawns, sessions, skills, sync) but never dictates how agents do their work.

**Why it matters:** Overly directive coordination systems constrain the agents they coordinate, reducing their effectiveness and adaptability.

**In Meridian:** Meridian launches, tracks, and routes. What the agent does inside the session — its reasoning, tool use, decisions — is entirely the agent's domain. Skills inject context; they don't prescribe execution steps.

---

## 10. Observable by Default, Intrusive Only Where Observation Requires It

**The principle:** Meridian reads harness state rather than driving harness behavior. Where observation needs a mechanism the harness doesn't provide, Meridian uses the minimum machinery needed.

**Why it matters:** Intrusive observation creates coupling and fragility. But pure passivity can leave important state unobservable.

**In Meridian:** The canonical example is PTY capture for Claude primary-launch session-ID extraction. The Claude TUI only emits its session ID to a TTY. Meridian uses PTY capture for that specific extraction, not as a general control mechanism. Any intrusive code should be justified against a specific unobservable-otherwise constraint.

---

## 11. Idempotent Operations

**The principle:** `meridian mars sync` twice = same result. Re-running after a crash converges to correct state, never doubles side effects.

**Why it matters:** Idempotency makes systems safe to retry and easy to reason about. Non-idempotent operations create "did it run?" ambiguity.

**In Meridian:** `reconcile_spawns` runs on every read path — safe to call repeatedly. `meridian mars sync` is convergent. `atomic_write_text` uses tmp+rename so partial writes never corrupt the target. Work-item rename intent files ensure the rename completes correctly even if replayed.

---

## 12. Windows Is First-Class

**The principle:** Windows support is a product requirement, not cleanup work. Design cross-platform from the start.

**Why it matters:** Retrofitting cross-platform support after the fact is much more expensive than designing for it upfront. POSIX-only assumptions compound.

**In Meridian:** `src/meridian/lib/platform/` centralizes OS detection. `fcntl.flock` vs `msvcrt.locking` is abstracted behind `platform.locking.lock_file()`. Path logic uses `pathlib.Path` throughout. Process termination uses `psutil` for cross-platform liveness checks.

---

## 13. No VCS Dependency for Core Functionality

**The principle:** Meridian must work in plain directories without git. Git metadata may be used as optional hints, but core operations must not require a git repository.

**Why it matters:** CI environments, temporary work directories, and non-git version control systems are all real use cases.

**In Meridian:** Project identity uses a UUID in `.meridian/id`, not the git remote. Root discovery uses `.mars/` presence as the primary project marker during the post-`.agents/` migration, with legacy `.agents/skills/` and `.git` as fallback heuristics. Workspace init writes `.git/info/exclude` but gracefully degrades when `.git` is absent.

---

## 14. Behavior Tests Over Implementation-Pinning Tests

**The principle:** A unit test is presumptively deletable if its primary failure mode is an internal rename rather than a behavioral regression. Tests that verify observable behavior at a stable seam survive refactoring; tests that verify internal wiring don't.

**Why it matters:** Mock-heavy implementation-pinning tests break on every internal rename without catching real regressions. They create the appearance of coverage while adding friction to structural improvement. With no real users and no backward-compatibility constraints, accumulated structural debt costs more than fixing it — and test suites that resist structural work are a form of structural debt.

**The positive rule:** Retain tests at stable seams with explicit contracts (policy precedence, fail-closed behavior, pure helper logic, cross-boundary behavior). Delete tests that verify collaborator call sequences, internal function invocations, or mock return plumbing. When reviewers find over-collapse, restore behavior contracts, not full suites.

**Smoke tests as markdown guides, not pytest:** `tests/smoke/` uses `.md` files, not pytest functions. Smoke tests for CLI behavior require real harness execution that pytest cannot automate usefully. Markdown guides are human-readable scenarios that can be followed interactively or delegated to `@smoke-tester`. Pytest-based smoke tests add maintenance burden without automation benefit — they break on every CLI shape change but don't actually validate end-to-end behavior. New smoke scenarios go in `tests/smoke/<topic>.md`, not in `test_*.py` files.

**The framework-testing deletion criterion:** A second, independent reason to delete a unit test: it tests framework behavior rather than project logic. Tests that parse a known TOML/JSON string and assert that `serde::Deserialize` or a stdlib function deserialized it correctly are testing the framework's correctness, not the project's. The framework is already tested by its own maintainers. Delete these tests without replacing them — they add maintenance burden with no coverage value. The concrete criterion: if the test would pass or fail identically in an unrelated project that happens to use the same library, it is a framework test. Applied during init-ux-overhaul (2026-05): 4 serde-parse unit tests deleted from mars-agents that verified `toml::from_str()` deserialized known struct shapes correctly.

**In Meridian:** Validated in Phase 8.6 (2026-05-08): −3,005 LOC, 7 mock-heavy test suites collapsed, all four gates passing. Over-collapse in primary_launch/compiler was corrected with targeted contract tests, not suite restoration. `test_work_items.py` and `test_workspace.py` converted to `work-items.md` and `workspace.md` in the same session. See [lessons/source-simplification.md](../lessons/source-simplification.md).

---

## 15. Op Layer Purity — Caller Identity via RuntimeContext

**The principle:** Op API functions in `lib/ops/` must never read environment variables directly (e.g., `os.getenv("MERIDIAN_CHAT_ID")`). Caller identity — which chat, which spawn — comes through `RuntimeContext`, passed as the `ctx` parameter.

**Why it matters:** `lib/ops/` is a shared layer consumed by CLI callers, MCP callers, and REST callers. A function that reads `MERIDIAN_CHAT_ID` from the environment is silently coupling itself to CLI semantics: it only works correctly when a harness session env is present. MCP callers pass identity from the request; REST callers pass it from the HTTP context. Leaking `os.getenv()` into op functions makes them untestable and incorrect for non-CLI callers.

**The pattern:** CLI callers construct `RuntimeContext.from_environment()` before the first op call. The `RuntimeContext` object carries the resolved `chat_id` and `spawn_id`. Op functions call `runtime_context(ctx)` to unpack caller identity — they never touch `os.environ` themselves.

**In Meridian:** The fix to `cancel-all` (`SpawnCancelAllInput` previously read `os.getenv("MERIDIAN_CHAT_ID")`) established this pattern by mirroring `spawn_wait_sync`, which already used `runtime_context(ctx)`. Any op function that needs to know which chat or spawn it's operating in behalf of should receive that identity through `RuntimeContext`, not through env-reading in the op model.

**See also:** Principle #1 (Separate Policy from Mechanism) — env-reading in op models is the op layer encoding CLI policy.

---

## Cross-References

- [principles/invariants.md](invariants.md) — collected system invariants
- [architecture/system-overview.md](../architecture/system-overview.md) — how principles manifest structurally
- [lessons/state-design-lessons.md](../lessons/state-design-lessons.md) — why the state system is built the way it is
- [lessons/harness-integration.md](../lessons/harness-integration.md) — harness-agnostic lessons from practice
- [lessons/source-simplification.md](../lessons/source-simplification.md) — Phase 8.6 test collapse and source seam lessons
