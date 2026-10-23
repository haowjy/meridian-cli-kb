# Prompt Packages Overview

A prompt package is a mars source repo of agent profiles, skills, hooks and
bootstrap docs. `meridian mars sync` resolves a project's packages and
compiles them into `.mars/`, which Meridian reads at spawn time, and copies
them into whichever harness directories the project lists as targets. How
mars resolves, compiles and targets is documented under
[package management](../../concepts/package-management/overview.md); this
page covers the packages themselves: which exist, how they depend on each
other, and the traps in that graph.

Checked against the package sources on 2026-10-23.

## Where Each Fact Lives

Each package's README lists its agents and skills, and each agent or skill
file states its own model, tools and method. meridian-base's README carries
the model-alias table, generated from its `mars.toml`. Its `CHANGELOG.md`
dates every change. Read those for inventories. The KB pages for each
package hold what no repo file says: why the package has its shape, what it
used to be, and what trips agents up.

The KB used to copy inventories. Within a few months its package pages
described agents, skills and model aliases that had been deleted, and they
misled agents that trusted them. So these pages link instead.

## The Package Graph

Sources live in sibling checkouts under `../prompts/<package>/`.

```mermaid
graph TD
    MB["meridian-base<br/>shared workers, coordination skills, model aliases"]
    MDW["meridian-dev-workflow<br/>dev leads, specialists, lifecycle skills"]
    MPR["meridian-prompter<br/>prompt engineering"]
    MBA["meridian-benchmark-agents<br/>unattended benchmark runs"]
    AR["autoresearch<br/>research loops"]
    CWS["creative-writing-skills<br/>fiction agents"]
    WS["writing-sync<br/>Drive/Docs sync skills"]
    MCA["microct-analysis<br/>micro-CT agents"]
    MCR["microct-arena<br/>consumer workspace"]
    JW["jupyter-workbench<br/>notebook skills"]

    MDW --> MB
    MPR -->|"selected skills only"| MB
    MBA --> MB
    MBA --> MDW
    AR --> MB
    AR --> MDW
    CWS --> MB
    CWS --> WS
    MCA --> MDW
    MCR -.->|"local path"| MCA
    MB -.->|"local only"| MPR
    MDW -.->|"local only"| MPR
    MCA -.->|"local only"| MPR
    CWS -.->|"local only"| MPR
```

Solid arrows are published `[dependencies]`: a project that installs the
package gets them too. Dashed arrows are `[local-dependencies]`, which mars
resolves when someone works inside that package's checkout but does not
export to consumers.

meridian-base is the root. Every other Meridian-flavored package builds on
it, directly or through meridian-dev-workflow. writing-sync and
jupyter-workbench stand alone. Domain packages are covered in
[domain-packages.md](../domain-packages.md).

## Prompter Is Local, Not a Foundation

Several packages list meridian-prompter as a local dependency, so a prompt
author editing meridian-base or meridian-dev-workflow can run `prompt-dev`
and load `/prompt-principles` inside that checkout. Because the edge is
local, nobody who installs those packages receives prompter. A project gets
`/prompt-principles` only by declaring meridian-prompter itself, as
meridian-cli's `mars.toml` does. The published edge runs the other way:
meridian-prompter depends on meridian-base, filtered to the handful of skills
its agents load.

Older KB text called prompter "a dependency of both base and dev-workflow."
That was never true for consumers.

## Traps

- **Stale version pins.** meridian-benchmark-agents and autoresearch pin
  meridian-base `<0.11.0` and meridian-dev-workflow `<0.14.0`, and
  microct-analysis pins meridian-dev-workflow `<0.14.0`. Current releases are
  meridian-base 0.11.x and meridian-dev-workflow 0.15.x, so these packages
  can't be installed alongside them until their pins move.
- **jupyter-workbench's manifest predates the current schema.** It lists
  skills as `[[package.skills]]` path entries, where other packages rely on
  directory discovery.
- **microct-arena is a workspace, not a distributable package.** It points
  at a local microct-analysis worktree so a human can drive those agents by
  hand.
- **meridian-prompter has no README.** Its [KB page](meridian-prompter.md)
  is its main description.

## Releasing and Consuming

Each package's `AGENTS.md` gives the release command. From a Meridian
session, pass the package root explicitly (`meridian -C "$PWD" mars version
patch --push`), because nested `meridian` commands resolve the session's
project root, not the current directory. Consumers pick up a release with
`meridian mars add <package>@<version>` followed by `meridian mars sync`.

## Related

- [meridian-base.md](meridian-base.md) — the root package
- [meridian-dev-workflow.md](meridian-dev-workflow.md) — development leads and lifecycle
- [meridian-prompter.md](meridian-prompter.md) — prompt engineering
- [meridian-benchmark-agents.md](meridian-benchmark-agents.md) — benchmark runs
- [prompt-principles.md](prompt-principles.md) — the prompt-engineering doctrine
- [../domain-packages.md](../domain-packages.md) — domain packages
- [../../concepts/package-management/overview.md](../../concepts/package-management/overview.md) — how mars resolves, compiles and targets packages

## Provenance

- Audit of the package sources against this page: `spawn:p9065`
- Rewrite: `chat:c8779` follow-up, prompt-dev session
