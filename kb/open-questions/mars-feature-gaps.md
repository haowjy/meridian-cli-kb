# open-questions/mars-feature-gaps — Mars Package Manager Feature Gaps

Mars (`mars-agents`) handles content sync well: resolving package sources, syncing agent and skill markdown into a managed root, preserving local edits, handling merge/conflict flows. What it does not yet handle is the environment and capability side of packages.

This page tracks the open questions. As of April 2026, these are gaps between Mars-as-content-sync and Mars-as-capability-package-manager.

---

## 1. Permission Sync

**The gap:** Packages can install agent and skill content, but cannot declaratively define the runtime permission model those assets expect.

**What's missing:**
- Approval mode defaults (should this agent run in `auto` or `confirm`?)
- Sandbox tier expectations
- Tool allowlists or denylists
- Runtime-specific permission settings

**Why it matters:** Without permission sync, a package ships content that assumes capabilities the runtime never grants. The agent doesn't work as intended, and the user has no signal about what permissions are expected. Setup becomes manual, non-reproducible.

**Potential direction:** A declarative `[permissions]` section in `mars.toml` that Mars materializes into supported tool/runtime configs on sync.

**Draft issue title:** `Feature: sync package-defined permissions into tool/runtime environments`

---

## 2. Tool Definition Distribution

**Status (2026-05): Partially addressed** — The universal skill frontmatter
schema now includes an `allowed-tools` field. When Meridian loads a skill, it
can inject the declared tool allowlist. This is a per-skill declaration, not
a full tool distribution system.

**Remaining gap:**
- Packaged tool specs independent of skill files
- Shared tool aliases distributed from packages
- Runtime-specific tool registrations derived from one package schema
- Validation that expected tools are available at install time

**Why it matters:** Skills and agents are only part of the execution contract. If tools are configured separately (or not at all), packages are incomplete. Sharing an agent without sharing its expected tools makes the package underspecified.

**Potential direction:** Support package-managed tool definitions with a declarative intermediate schema and per-runtime materialization. Similar to how skills are declared in the package and materialized as files.

**Draft issue title:** `Feature: distribute tool definitions as first-class Mars package artifacts`

---

## 3. Hook Distribution

**Status (2026-05): Partially resolved** — Mars now distributes hook entries as
first-class config-entry artifacts, with provenance tracking, collision
resolution, and stale cleanup on sync. See [architecture/mars-compiler.md](../architecture/mars-compiler.md).

**Remaining gap:** Lifecycle hooks (post-spawn, pre-exit, runtime events)
and post-sync hooks are still not a first-class concern. What shipped
covers harness config hooks (e.g., `settings.json` hook entries) — not
dynamic lifecycle bindings.

**Risk (still applies):** Prematurely adding lifecycle hook distribution
could pull runtime-specific imperative logic into Mars. The design needs
a clear abstraction boundary before expanding further.

---

## 4. MCP Integration

**Status (2026-05): Partially resolved** — Mars now manages MCP server
registrations as first-class config-entry artifacts: packages declare MCP
servers, Mars materializes them into `.mcp.json` / `codex_mcp.json`, tracks
provenance in `mars.lock`, and removes stale entries when packages change.
Collision resolution follows the agent precedence model (local wins,
declaration order). See [architecture/mars-compiler.md](../architecture/mars-compiler.md).

**Remaining gap:**
- Resource or template exposure
- Capability bundles depending on external MCP services
- Validation of expected capability presence at install time

**Draft issue title:** `Feature: MCP capability validation at mars install time`

---

## 5. Distribution Model

**The gap:** Mars can fetch from git and local paths. That's not a package distribution model.

**What's missing:**
- Package discovery (how do you find packages?)
- Publisher identity and trust
- Registry or index support
- Cached metadata and search
- Install policy by source trust level

**Why it matters:** Git URLs work for a developer tool. They don't work for a broader ecosystem where packages are published by third parties and consumed by teams who didn't write them. Trust and provenance are unaddressed.

**Priority note:** This is likely last in the priority order. The core capability gaps (1–4) must be solved first; a distribution model without a clear capability schema is premature.

**Potential direction:** A minimal registry or index model with explicit publisher and provenance handling. Even a GitHub-hosted index would be a step forward.

**Draft issue title:** `Feature: add a real package distribution model beyond git/path sources`

---

## 6. Agent Profile Visibility Fields: `model-invocable` / `user-invocable`

**The gap (as of 2026-05, PR #208):** Mars parses and handles `model-invocable` /
`user-invocable` fields for **skill** SKILL.md files during compilation, but does
not parse or enforce these fields for **agent profile** `*.md` files.

**What this means for Meridian:** Meridian reads `model-invocable` from materialized
agent profiles in `.mars/agents/`. When prompt package source agents declare
`model-invocable: false`, Mars must preserve that field in the compiled output for
Meridian's inventory filter to work. Currently Mars preserves frontmatter verbatim
during compilation, so this works in practice — but the behavior is incidental, not
formally specified.

**Tracked in:** `haowjy/mars-agents#40`

**Risk:** If Mars gains a frontmatter normalization or stripping step for agent files
without first formally specifying that visibility fields are preserved, Meridian's
filter silently loses the suppression. Agents that should be hidden from model
inventories would reappear.

**Required Mars behavior:** The Mars compiler must preserve `model-invocable` and
`user-invocable` fields verbatim in compiled agent profile output. When Mars gains
formal handling for these fields (analogous to skill compilation), it should produce
the same effect rather than stripping them.

**Mitigation until resolved:** Agent package authors must manually verify that
compiled `.mars/agents/` output retains the `model-invocable: false` line after
running `meridian mars sync`.

---

## Priority Order

If resources allow only a few of these to be built, the likely priority:

1. Permission sync — most direct gap between packaged content and usable runtime behavior
2. Tool definition distribution — tools are half the execution contract
3. ~~MCP integration~~ — **partially resolved** (2026-05): config-entry sync and collision resolution shipped; remaining: capability validation
4. ~~Hook distribution~~ — **partially resolved** (2026-05): harness config hooks ship with provenance; remaining: lifecycle/event hooks
5. Distribution model — important eventually, premature without the capability schema

## Cross-References

- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — what Mars currently does
- [open-questions/future-work.md](future-work.md) — other deferred items
