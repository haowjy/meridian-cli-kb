# meridian-base

meridian-base is the root prompt package: the shared workers (explorer,
session miner, generic subagent, knowledge agents, cold reader), the
coordination and writing skills that agents in every other package load, the
model-alias catalog, a few hooks, and bootstrap docs. It has no lead of its
own. Leads live in the packages that need them, such as
[meridian-dev-workflow](meridian-dev-workflow.md).

**Source:** `../prompts/meridian-base/`. The README lists the agents, skills
and model aliases, and `CHANGELOG.md` dates each change.

**Published dependencies:** none. meridian-prompter is a local dependency
only ([why](overview.md#prompter-is-local-not-a-foundation)).

Checked against the package source on 2026-10-23.

## Why It Has No Lead

Up to 0.7.19 the package shipped `meridian-default-orchestrator` and
`meridian-subagent`. 0.7.20 removed both: "Real work routes through
specialized agents… the orchestrator carried no `subagents` roster, so under
capability-gated spawn injection it would ship without an agent inventory
anyway." A lead is only useful with a roster of agents it can spawn, and
choosing that roster is a domain decision, so leads belong to domain
packages.

A generic `@subagent` returned in 0.7.37 for tasks no specialist fits. Its
body is one line; the caller's brief carries the work.

## Why Spawning and Work Items Aren't Skills

The package once shipped `meridian-spawn` and `meridian-work-coordination`
skills. 0.7.22 deleted `meridian-spawn`: "Spawn doctrine now lives in
meridian itself." Meridian injects the spawn contract into each launch, and
`meridian spawn -h` is the on-demand reference. Work-item handling folded
into `/work-artifacts`, whose lifecycle detail sits in a resource rather than
the body, because only the agent that started a work item manages its state.
CLI mechanics stay with the CLI, so the package never drifts from the CLI
version it runs on.

## Domain-Neutral Since 0.11.0

From 0.11.0, base skills speak of "the project's files" and "the same
change" rather than code and branches. Rules for code projects (branches,
merges, code as ground truth) moved to `knowledge-layers/resources/code.md`,
which `@kb-lead` loads in a code project. Non-code packages such as
creative-writing-skills build on base, so base can't assume git or a test
suite.

## Knowledge Agents

- **`@kb-lead`** writes durable knowledge itself, into `.context/`, the KB,
  and `docs/`, and removes what has gone stale. Writing used to be split off:
  `@kb-writer` was removed in 0.7.3 ("KB writing folded into `@kb-lead`"),
  and meridian-dev-workflow's `@tech-writer` folded into it in 0.11.4. One
  writer means one agent judges where each fact belongs.
- **`@kb-maintainer`** handles structure: links, moves, splits and the
  graph. It does not author content.
- **`@cold-reader`** (0.11.3) reads text that will outlive the work as a
  reader who wasn't there, and reports where that reader stalls. It exists
  because writers can't see their own conversational bleed.

## Model Aliases

Aliases live here so every package resolves `opus` or `luna` the same way.
They change as the model market changes, and the README table is
regenerated from `mars.toml` by `scripts/models_table.py`, so read the
README instead of copying it. Shifts that break old prompts and notes:

- Aliases are family matchers that resolve to the newest model in the
  family. The pinned-version aliases (`opus46`, `opus48`, `gpt55`, `gpt54`)
  were removed between 0.7.40 and 0.10.18, and `gpt`/`gptmini` were removed
  in 0.11.5. The GPT family is now `sol`, `terra`, `luna` and `astra`.
- `deepseek` has meant V4 Flash since 0.10.9, and `deepseekflash` is gone.
  Pro is `deepseekpro`.
- `sonnet` was removed in 0.7.26 and later restored.
- `fable` is for escalation only and is never a default (0.11.8).

An alias name copied from an older note or prompt may no longer resolve.

## Tool Allowlists

Meridian treats a profile whose bash permissions are all scoped allows
(`bash(meridian spawn *)`, `bash(rg *)`) as an allowlist and denies every
other command. In 0.11.11–0.11.14 this left explorers and session miners
refused `ls`, `pwd` and `git log`, and a dead-code explorer had every bash
call denied. A read-only profile with scoped allows has to list each
ordinary read command it needs.

## Hooks

Mars-compiled hooks using the D91 native fragment schema. Each hook has a
`hook.toml` (identity and routing) and a per-target JSON fragment
(`claude.json`, `codex.json`) carrying the harness's native config shape
verbatim. Mars validates event keys, substitutes `${MARS_HOOK_DIR}` with
the installed path, and merges entries into target config files.

| Hook | Fragment | Native events | Target | Purpose |
|---|---|---|---|---|
| `deny-generic-agent` | `claude.json` | `PreToolUse` (matcher `"Agent"`) | `.claude` | Blocks Claude's bundled generic agents (`Explore`, `Plan`, etc.) |
| `deny-interactive-prompts` | `codex.json` | `PreToolUse` (matcher `"Bash"`) | `.codex` | Blocks interactive-prone auth commands in Codex unless a non-interactive guard is present |
| `context-autosync` | `claude.json` | `SubagentStop`, `SessionEnd` | `.claude` | Pushes stranded context-repo commits at session end and after subagent work |

**`context-autosync`** closes a gap where meridian's builtin `git-autosync`
fires only on meridian lifecycle events (by default `spawn.finished`,
`work.activated` and `work.done`). Commits created after the last such event,
typically knowledge capture by an Agent-tool subagent as a session's final
act, sit stranded locally until the next meridian event. This hook fires at
Claude `SessionEnd` (with `timeout: 30`, because Claude's default SessionEnd
timeout could kill it mid-run) and `SubagentStop`, discovers each enabled
`git-autosync:*` instance via `meridian hooks list` (the single runtime
authority, with no parallel config parsing), and runs them. It is a no-op
without autosync config.

The fragment schema needs mars-agents 0.12.0 or later. The package migrated
to it from the D90 per-target event schema in 0.10.0.

## Bootstrap Docs

`bootstrap/` holds setup guides: `context-setup`, `harness-setup`,
`meridian-guide`, `project-knowledge` and `workspace-setup`.

## Related

- [overview.md](overview.md) — the package graph and its traps
- [meridian-dev-workflow.md](meridian-dev-workflow.md) — the development leads built on this package
- [meridian-prompter.md](meridian-prompter.md) — prompt engineering, which depends on this package
- [../../concepts/hooks-and-plugins.md](../../concepts/hooks-and-plugins.md) — the hook system these hooks plug into

## Provenance

- Audit of the package source against this page: `spawn:p9065`
- Rewrite: `chat:c8796`
