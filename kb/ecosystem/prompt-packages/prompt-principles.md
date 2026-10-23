# Prompt Principles

`/prompt-principles` is the prompt-engineering doctrine shipped by
[meridian-prompter](meridian-prompter.md): how to write text that agents
read so it steers their behavior reliably. The skill is the doctrine. This
page explains how it is organized and why, and lists claims it has
deliberately dropped. Read the skill itself for the principles.

**Source:** `../prompts/meridian-prompter/skills/prompt-principles/`:
`SKILL.md`, plus `resources/agent-level.md`, `skill-level.md`,
`system-level.md`, `evaluation.md` and `research.md`.

Checked against the package source on 2026-10-23.

## How It Is Organized

The body holds the principles that apply to every line of every prompt:
writing for the reader of each layer, progressive disclosure, placement,
one source of truth, flat instructions, harness-agnostic cores, and
proportional verification among them. The body loads whenever the skill
does. The level resources load only when an author opens one:

- `agent-level.md`: writing an agent (role, boundary, description,
  cognitive mode).
- `skill-level.md`: writing a skill (its reader, what to extract, how it
  loads).
- `system-level.md`: leads and multi-agent flow (handoffs, verification,
  loop control, model staffing).
- `evaluation.md`: methods for an approved behavioral experiment.
- `research.md`: the evidence behind each principle.

The levels are a reference split, not a pipeline: an author opens the one
that matches what they are writing.

Until 0.5.0, the prompt-level principles were a resource too. They moved
into the body because "it governs every line, so it loads instead of
waiting to be opened." Meridian-specific mechanics (frontmatter fields, what
Meridian injects, how context reaches a spawn) live in `/agent-artifacts`
and `/skill-artifacts`, not here, so a harness change touches those skills
and leaves the doctrine alone. `/llm-writing` is the companion skill for the
prose itself.

## Dropped Claims

These were in the skill or in older KB text and are now gone, most because
the evidence didn't hold. Don't reintroduce them from old notes:

- **"Agents should have 3–5 functions."** Removed in 0.0.7 as a
  practitioner heuristic presented as empirical.
- **"Loop guards of 15–25 iterations."** Replaced in 0.5.0 by limits per
  failure class: a concern that needs a third round of fixes ends patching.
- **"~2% context lost per handoff."** Removed from `research.md` in 0.5.1
  as unsupported.
- **"Leads coordinate and never implement."** Reversed in 0.5.0. Delegating
  a two-line fix pays for a briefing and a cold worker and saves nothing.
- **"Personas harm performance."** The evidence is mixed: persona prompts
  help some tasks (alignment, style) and hurt others (reasoning accuracy).
- **"Every prompt change gets review and a behavioral test."** Replaced in
  0.5.6 by Verify Proportionally.
- **"Skill bodies under 500 lines"** and other fixed size limits. Dropped;
  the test is whether each line changes behavior.

## Related

- [meridian-prompter.md](meridian-prompter.md) — the package and agents that apply this doctrine
- [overview.md](overview.md) — which projects receive this skill

## Provenance

- Audit of the skill against this page: `spawn:p9066`
- Rewrite: `chat:c8796`
