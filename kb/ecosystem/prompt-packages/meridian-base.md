# meridian-base

**meridian-base** is the foundation package that turns a bare Meridian install into a functional multi-agent coordination system. It provides the orchestrator, subagent, KB agents, and the core skills that every other package builds on.

**Package version:** 0.2.2  
**Source:** `../prompts/meridian-base/`  
**Key dependency:** `meridian-prompter` (provides `prompt-principles`)

## Agents

### Coordination

**`meridian-default-orchestrator`** — minimal orchestrator for planning, delegating, and evaluating subagent work. Uses `meridian spawn` for all delegation (never built-in Agent tools). Keeps output at coordination altitude — breaks work into focused subtasks, fans out reviewers for high-risk work, evaluates results before proceeding. Skills: `meridian-spawn`, `meridian-work-coordination`, `meridian-privilege-escalation`, `shared-workspace`.

**`meridian-subagent`** — default execution agent for scoped repo-local tasks. Model: `gpt-5.3-codex`. Executes the described task directly, verifies its own work, avoids touching files outside scope. Skills: `shared-workspace`.

### Knowledge Base

**`kb-writer`** — captures durable knowledge into the KB after work ships, research lands, or decisions are made. Model: `claude-sonnet-4-6`. Spawn with `--from` for conversation context, `-f` for source material. Resolves KB path via `meridian context kb`, reads `index.md` first, integrates into existing pages, mines conversation history for decisions, runs `meridian kg check` and `meridian mermaid check` before committing.  
Skills: `meridian-spawn`, `kb-conventions`, `md-validation`, `session-mining`, `decision-log`, `shared-workspace`, `llm-writing`, `intent-modeling`.

**`kb-maintainer`** — structural health agent for document trees. Model: `gpt-5.5`. Treats docs like code: splits oversized docs, merges thin docs, creates folder structure, fixes cross-references, flags staleness. Defaults to the KB (`meridian context kb`) or accepts an explicit `-f` target.  
Skills: `meridian-spawn`, `kb-conventions`, `md-validation`, `shared-workspace`.

### Exploration and History

**`explorer`** — cheap, fast, read-only codebase exploration. Model: `gpt-5.4-mini`. Uses `rg`, `cat`, `find`, `git show`, `git log`. Reports facts with file/line references. Does not edit. Not for conversation history — use `session-explorer` for that.

**`session-explorer`** — mines conversation transcripts for decisions, rejected alternatives, intent, and constraints. Model: `sonnet`. Passes `--from <spawn-id>` or `--from $MERIDIAN_CHAT_ID` to target a session. Reports substance, not transcript noise. Structures findings by type (decisions, rejected alternatives, constraints, open questions).  
Skills: `session-mining`, `intent-modeling`, `llm-writing`.

## Skills

### Delegation and Coordination

**`meridian-spawn`** — the core delegation skill. Covers `-a` vs `-m`, `-f` for scoped context, `--from` for prior reasoning, `--bg` + `meridian spawn wait` for parallelism, inject-before-cancel for steering, `meridian session log/search` for transcript reading, `meridian spawn files` for staging exact changed files, and prompt variables. Background spawns (`--bg`) must always be drained with `meridian spawn wait` before responding or ending a turn. See `resources/advanced-commands.md` for continue/fork, cancel, stats, dry-run.

**`meridian-work-coordination`** — work-item lifecycle and artifact placement. Orchestrators own work state: create/switch work items before repo work starts, keep status visible, commit artifacts frequently. Rule: this work item → work dir; project-wide → KB.

**`meridian-privilege-escalation`** — how to increase permissions per spawn without changing the agent profile. Least-privilege escalation first; targeted fixes before broad overrides. Covers Codex sandbox tiers, approval modes, model/harness switching.

**`shared-workspace`** — safety rules for concurrent agents and humans. Initial orientation: `meridian context`, `meridian work`, `git status`. Forbids destructive resets and stash flows. Stage only files you changed (`meridian spawn files` for exact lists).

### Knowledge and Writing

**`kb-conventions`** — the five-layer KB model and wiki structural rules. Single concept per document, `index.md` as entry point, overview docs per directory, relative linking, explicit invariants, mermaid for spatial relationships. Flags human-review issues with `> [!FLAG]`. Depends on `/llm-writing`.

**`md-validation`** — markdown and Mermaid validation workflow. Commands: `meridian kg`, `meridian kg check`, `meridian kg graph`, `meridian mermaid check`. Flow: quick stats → broken-link gate → diagram validation → fix/re-run.

**`decision-log`** — captures decisions while reasoning is fresh. Records: human intent, derived goal, choice made, rejected alternatives. Rejected alternatives are usually the most valuable part. Skips boilerplate. Depends on `/intent-modeling`.

**`session-mining`** — recovers decisions and constraints from conversation history. Starts from `$MERIDIAN_CHAT_ID`, delegates bulk reading to `@session-explorer`, lists all sessions for a work item. Skips when artifacts already hold the rationale.

**`llm-writing`** — guards against common LLM writing failure modes: fluent-but-purpose-free prose, label-summaries without explanation, evidence-free conclusions, smoothing uncertainty, conversational register in documents. Depends on `/intent-modeling`.

**`intent-modeling`** — interpret human direction before acting. Prevents literal over-reading of directional corrections. When "stop doing X" means "less X, more Y", reads the context to tell which. Checks for systematic misalignments.

**`agent-management`** — delegation discipline, convergence loops, escalation. Spawns are the unit of substantive action; artifacts are state; in-memory reasoning evaporates. Resources: `convergence.md` (stop conditions, loop patterns, deferral rules), `escalation.md` (redesign brief template, `design-problem` vs `scope-problem`).

## Model Alias Catalog

Defined in `mars.toml`, available to all agents in packages that depend on `meridian-base`:

| Alias | Model | Harness | Character |
|---|---|---|---|
| `gpt55` | gpt-5.5 | codex | Action-oriented, fast; weak at nuanced judgment |
| `opus47` | claude-opus-4-7 | claude | Strongest benchmarks; needs prompt migration from 4.6 |
| `opus` | claude-opus-4-6 | claude | Orchestration, frontend, creative work |
| `sonnet` | claude-sonnet (latest) | claude | Balanced reasoning, good writing |
| `gptmini` | gpt-5.x-mini | codex | Bulk exploration, simple tasks |
| `codex` | gpt-codex | codex | Backend implementation, faithful execution |
| `gpt` | gpt-5.4 | codex | Strongest generalist judgment, review, architecture |

## Package Composition Patterns

The package's composition logic, inferred from skill assignments:

| Agent type | Core skills |
|---|---|
| Orchestrator | `meridian-spawn` + `meridian-work-coordination` + `agent-management` + `shared-workspace` |
| Subagent | `shared-workspace` |
| KB agents | `kb-conventions` + `md-validation` + `decision-log` + `intent-modeling` + `llm-writing` |
| Session explorer | `session-mining` + `intent-modeling` + `llm-writing` |
| Explorer | none (thin shell) |

## Related

- [overview.md](overview.md) — package model and composition
- [meridian-dev-workflow.md](meridian-dev-workflow.md) — dev workflow package built on top of this
- [meridian-prompter.md](meridian-prompter.md) — prompt engineering package (dependency of this one)
- [../../concepts/spawn-lifecycle.md](../../concepts/spawn-lifecycle.md) — spawn state machine
- [../../codebase/guide.md](../../codebase/guide.md) — meridian-cli codebase navigation
