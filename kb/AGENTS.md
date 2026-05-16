# KB Guide

This governs how agents and humans maintain the Meridian KB.

## What the KB Is

The KB is the authoritative description of this system — both the concrete
reality (what exists, how it works, how pieces connect) and the reasoning
behind it (why it's this way, what was rejected, what traps exist). It serves
agents orienting on unfamiliar code, humans onboarding onto the project, and
anyone tracing back why a decision was made.

A page can document a module's structure, walk through an architectural
tradeoff, record a decision's rationale, or do all three. The question isn't
"is this code documentation or knowledge" — it's "would this help someone
understand this part of the system next time they touch it?"

Raw research dumps and conversation transcripts don't belong here — synthesize
them first. Active task plans live in work directories, not the KB. But
documenting how the code works, what the APIs look like, how data flows through
the system — that absolutely belongs here, alongside the reasoning and context
that make those facts meaningful.

## Writing

### Calibrate depth to importance

Not every page needs the same treatment. A vocabulary page is reference — terse
definitions, no narrative needed. An architecture page explaining why the
harness abstraction exists needs to walk through the problem, the constraints,
the alternatives considered, and how the current design addresses them. Match
the depth to how much understanding the reader needs to work effectively in
that area.

Reference-style content (config formats, API surfaces, file locations, CLI
flags) should be direct and scannable. Conceptual content (design reasoning,
tradeoff analysis, failure modes, "how to think about this") should use
narrative prose that builds understanding rather than listing facts.

### Show during decisions, tell during facts

When documenting a decision — why we chose X over Y, why a module is shaped
the way it is, what constraint forced a tradeoff — show the reasoning. Walk
through it. What was the problem, what options existed, what made one better.
The reader should finish understanding not just what was decided but why, well
enough to know when the decision should be revisited.

When documenting factual structure — where files live, what a config key does,
what calls what — state it directly. Don't narrativize reference material.

### Write what you'd want to find

Before writing, imagine you're an agent or engineer about to modify this part
of the system. What would you need to know? What would bite you if nobody told
you? What looks obvious from the code but actually has a non-obvious reason
behind it? Write that.

### LLM failure modes to resist

LLM training pushes toward specific failure patterns when writing documentation.
Recognize and resist them:

- **Summarizing instead of explaining.** "This module handles spawn lifecycle
  management" is a summary. How does it handle it? What are the states, the
  transitions, the edge cases? Summaries are labels, not understanding.

- **Homogenized structure.** Not every page needs the same Header → Bullets →
  Table → Related Pages skeleton. A decision record looks different from an
  architecture walkthrough looks different from an operations guide. Let the
  content dictate the shape.

- **Smoothing over uncertainty.** Some things in the system are ugly
  compromises, open questions, or known-fragile. Say so. "This works but is
  brittle because X" is more useful than presenting everything as clean and
  settled. Flag genuine uncertainty rather than narrating past it.

- **Labeled conclusions instead of shown reasoning.** "This is an important
  architectural boundary" — why? Show the consequence of violating it. Show
  what breaks. The reader should reach the conclusion themselves from the
  evidence you present.

- **Overcorrecting on direction changes.** When recording a change in approach,
  capture the calibration — what we moved toward, what we kept from before, why
  the balance shifted. "We moved from X toward Y" is different from "X is
  forbidden." The goal is usually balance, not a pendulum swing. Don't encode
  directional corrections as absolute prohibitions.

### Cross-reference, don't re-explain

When a concept is owned by another page, link to it. Duplicate explanations
drift apart. Each page is responsible for one coherent concept — if it covers
two unrelated topics, split it.

## Provenance

Record where knowledge came from so future readers can verify claims and
recover full reasoning. Concentrate provenance where it adds value — major
decisions, architecture pages, research synthesis. Leaf pages like vocabulary
entries and operations guides don't need it.

| Reference type | Format | What it identifies |
|---|---|---|
| Work item | `work:kb-llm-wiki-overhaul` | The work item that drove the change |
| Spawn | `spawn:p342` | The spawn that produced a finding |
| Chat session | `chat:c1234` | Conversation containing relevant reasoning |
| Decision entry | `D-001` | A numbered entry in `decisions.md` |

## Structure

- `index.md` is the catalog. Update it when you create or significantly modify a page.
- `decisions.md` is the decision index — what, when, link to rationale.
- `log.md` is the KB change log for structural reorganizations and major additions.
- Domain directories emerge organically. A topic earns a directory when it has
  multiple related pages. Don't create directories with a single page.

## Diagrams

Default to Mermaid for anything spatial — flows, state machines, module
relationships, dependency graphs. Every diagram must pass
`meridian mermaid check`. Prefer simple constructs and clear labels. Don't rely
on color alone as a differentiator.

## Links

- Run `meridian kg check .` before committing.
- Use relative paths for internal links.
- Update all inbound links when renaming or moving a page.

## Validation

Before committing:

```bash
meridian kg check .           # broken links
meridian mermaid check .      # diagram validity
```
