# meridian-prompter

meridian-prompter is the prompt-engineering package: agents and skills for
writing, reviewing and testing the text agents read (agent profiles, skills,
AGENTS.md). The package has no README, so this page is its main
description. Each agent's own file states its model, tools and method.

**Source:** `../prompts/meridian-prompter/` (github.com/haowjy/meridian-prompter).
`CHANGELOG.md` dates each change, and `AGENTS.md` covers releasing and
editing.

**Dependencies:** meridian-base (`>=0.11.0, <0.13.0`), filtered to the
skills its agents load (`explore-and-engage`, `grill`,
`grill-with-evidence`, `intent-modeling`, `llm-writing`). It requires mars
0.18.0 and Meridian 0.14.6 or later. Other packages
reference prompter only as a local dependency, so a project gets it only by
declaring it, as meridian-cli does
([why](overview.md#prompter-is-local-not-a-foundation)).

Checked against the package source on 2026-10-23.

## What It Contains

- **`prompt-dev`** (primary) drafts and revises prompts with the human.
- **`prompt-reviewer`** reviews a draft adversarially and returns ranked
  findings with a verdict: approve, approve with notes, or request changes.
- **`prompt-tester`** runs behavioral experiments on a prompt, only with a
  question and scope the user has approved.
- **`web-prompt-researcher`** finds external evidence for a prompting
  decision.
- **`python-tool-writer`** builds a deterministic tool when a library can do
  a job more reliably than an agent.
- **Skills:** `/prompt-principles` (the doctrine; see
  [prompt-principles.md](prompt-principles.md)), `/prompt-review`, and
  `/agent-artifacts` and `/skill-artifacts`. Since 0.5.12 the last two are
  short pointers to Meridian's bundled `prompt-authoring__meridian`, which
  holds the agent and skill file formats and versions with the CLI. They stay
  as pointers until package agents can list a bundled skill directly; prompt
  craft (`/prompt-principles`, `/prompt-review`) stays here.

prompt-dev also staffs meridian-base's `@cold-reader`, `@explorer` and
`@session-miner`.

## Verification Scales With Risk

The package started with a fixed cycle: draft, review, then a behavioral
test as "the final arbiter" on every change. That cycle is gone. Since
0.5.6, checks follow `/prompt-principles`' Verify Proportionally: small,
reversible edits get the author's own read of the diff and are watched in
normal work; substantial changes get independent review; a dedicated
experiment runs only when the user approves its question and scope. 0.5.3
had already dropped the rule that the reviewer use a different model family
("a fresh session on the worker's model is independent review too").

The changelog's reason: checks should scale "with uncertainty and
consequences," and real-task runs count as behavioral evidence, so the
evaluation guide no longer demands a run for every edit.

## Shape Shifts Worth Knowing

| Version (date) | Change | Stated reason |
|---|---|---|
| 0.0.7 (2026-04-28) | Dependency on meridian-dev-workflow dropped; the "3–5 function limit" removed | The dependency "was only used for internet-researcher"; the limit was a practitioner heuristic presented as empirical. |
| 0.1.0 (2026-04-30) | `prompter-orchestrator` renamed `prompt-dev` | A real-world title "primes for collaborative iteration over mechanical orchestration." |
| 0.4.0 (2026-07-09) | Keep the Core Harness-Agnostic | Load-bearing prompts state intent; CLI mechanics belong to the harness's system prompt or the tool's `-h`. |
| 0.5.0 (2026-10-15) | Leads may implement; prompt-level principles folded into the skill body | Delegate what spends decision context or needs an independent check, and do small fixes directly. The prompt-level principles govern every line, "so it loads instead of waiting to be opened." |
| 0.5.1 (2026-10-17) | Each meaning gets one home; `resources/meridian.md` deleted | Meridian specifics moved into `/agent-artifacts` and `/skill-artifacts`, so the loading-field list exists once. |
| 0.5.6 (2026-10-22) | Verify Proportionally; the tester is no longer a default gate | Checks "scale with uncertainty and consequences." |
| 0.5.12 (2026-10-25) | `/agent-artifacts` and `/skill-artifacts` point to `prompt-authoring__meridian` | Format details about Meridian and Mars version with the binary that implements them. |

## Related

- [prompt-principles.md](prompt-principles.md) — the doctrine this package ships
- [overview.md](overview.md) — the package graph and why prompter is a local dependency
- [meridian-base.md](meridian-base.md) — the package prompter depends on

## Provenance

- Audit of the package source against this page: `spawn:p9066`
- Rewrite: `chat:c8796`
