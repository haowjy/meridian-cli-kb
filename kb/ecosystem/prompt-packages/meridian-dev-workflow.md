# meridian-dev-workflow

meridian-dev-workflow is the software-development package: the leads a
human works with in a session (tech-lead, product-lead, design-lead,
solo-dev, ux-lead), the specialists they spawn (coders, reviewer, prober,
investigator, researchers), and the lifecycle skills that carry a change
from first edit to PR (`/pre-dev`, `/parallel-execution`, `/post-dev`,
`/dev-workflow`). tech-lead is the default primary.

**Source:** `../prompts/meridian-dev-workflow/`. The README holds the lead
roles, which lead hands off to which, the end-to-end lifecycle, and the agent and
skill tables with default models. Review lanes and stop conditions live in
`skills/agent-staffing/resources/reviewers.md`.

**Dependencies:** meridian-base (`>=0.11.4, <0.13.0`), which supplies the
shared workers including `@kb-lead` and `@cold-reader`. It requires mars
0.18.0 and Meridian 0.14.5 or later. meridian-prompter is a local dependency only.

Checked against the package source on 2026-10-23.

## From Pipeline to Leads Who Decide

The package began as a fixed pipeline: product-lead → design-lead → planner
→ tech-lead, with requirements written in EARS, a tech-lead that never wrote
code, a mandatory four-reviewer final gate, and a parallel swarm of QA and
documentation agents after implementation. Every step ran on every change.
Over five months the package removed that ceremony one piece at a time:

| Version (date) | Change | Stated reason |
|---|---|---|
| 0.4.0 (2026-05-08) | `@verifier` and the per-subphase reviewer removed from the implementation loop | Coders verify and review their own work; three spawns per subphase became one. |
| 0.5.0 (2026-05-09) | `@refactor-coder` and `@refactor-reviewer` absorbed into `@coder` and `@reviewer`; `/dev-principles` rewritten | Structural refactoring is part of coding and reviewing; principles moved from SOLID to simplicity and entanglement. |
| 0.7.0–0.7.24 (2026-05-14 to 05-27) | Planner, plan packages and EARS deleted | "Planning is deprecated; tech-lead decomposes work inline from the design package." |
| 0.10.0 (2026-05-30) | qa-lead, the tester agents and the per-tier test skills removed; `/pre-dev`, `/post-dev` and `/dev-workflow` added | Testers "consolidated into testing skill resources"; the lifecycle moved into checkpoints every lead runs. |
| 0.11.4 (2026-05-31) | `@kb-lead` moved to meridian-base; `@tech-writer` folded into it | One generic agent captures knowledge inline. |
| 0.14.0 (2026-10-15) | tech-lead became the default; product-lead settles direction and hands off; ux-lead builds the UI itself; knowledge capture runs once, from `/post-dev` | product-lead "no longer spawns builders, design leads, or `@kb-lead`." |
| 0.15.0 (2026-10-17) | `@gpt-dev` renamed `@solo-dev` (breaking); review no longer required on every change; new tests kept only when the user asks | A lead can ship a small fix unreviewed; tests written to check the work are deleted afterwards. |
| 0.15.2–0.15.6 (2026-10) | Reviewers run in their own session; staffing follows configured model routing; a model is escalated only after review finds substantial defects | "Task complexity and launch failures no longer justify overrides." |
| 0.15.8 (2026-10-23) | Plan review and the thermo lane fire only when a change adds structure or grows code beyond a local edit | A 7-line edit inside an existing hook had triggered a plan review, a dead-code scan and a serial thermo review, and none found anything. |

The direction throughout is that leads make decisions and do small work
themselves, and each check runs only when the change carries the risk it
checks. The pipeline's goals stayed: independent review, runtime evidence,
captured knowledge. What went was running every check on every change.

## Why the Leads Are Shaped This Way

- **product-lead hands off rather than spawning.** The next lead (design,
  UX, tech or solo) starts as a new session the human launches from the
  handoff brief. A spawned lead would run without the human, and direction
  and design are the phases where the human most needs to steer.
- **tech-lead makes small fixes itself.** Delegating a two-line fix pays for
  a briefing and a cold worker to save nothing. tech-lead delegates work that
  would spend its decision context, work that can run in parallel, and
  independent checks.
- **solo-dev exists for changes one agent can carry.** It implements and
  verifies alone and staffs reviewers only as the change needs. It is a lead
  in its own right, not a tech-lead worker.
- **Coders never push.** The lead integrates, and merges go to
  `@coder --skills merge-prep` (0.15.5).

## Traps

- **Don't revive the retired pieces.** Planner, EARS, `qa-lead`,
  `@verifier`, tester agents, `@tech-writer`, `@alignment-reviewer` (now
  `@reviewer --skills review-alignment`) and the fixed four-lane final gate
  are gone. Older KB pages, notes and session logs still mention them.
- **`@gpt-dev` is now `@solo-dev`.** Profiles and notes that name the old
  agent fail to resolve it.
- **Model names in old notes are stale.** Profiles use meridian-base aliases
  (`opus`, `luna`, `sol`, `astra`), and the interactive leads mark Fable
  `no-fallback`, so automatic fallback never picks it; it runs only when
  chosen explicitly.

## Related

- [overview.md](overview.md) — the package graph and its traps
- [meridian-base.md](meridian-base.md) — shared workers and model aliases this package builds on
- [prompt-principles.md](prompt-principles.md) — the doctrine behind "leads spend context on decisions"

## Provenance

- Audit of the package source against this page: `spawn:p9064`
- The 0.15.8 trigger change and its evidence: `chat:c8779` (the slow solo-dev run), `chat:c8796` (diagnosis and rewrite)
