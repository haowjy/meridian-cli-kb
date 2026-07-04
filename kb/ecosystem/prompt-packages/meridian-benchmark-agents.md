# meridian-benchmark-agents

`meridian-benchmark-agents` is the benchmark-runner prompt package: it exists to run externally specified coding-agent eval tasks unattended, optimize for verifier-backed acceptance, and report comparable evidence across configured harnesses and models.

**Source:** `../prompts/meridian-benchmark-agents/`  
**Role:** benchmark orchestration and verifier-driven implementation attempts  
**Dependencies:** `meridian-base`, `meridian-dev-workflow`

## Why It Is Separate

Benchmark runs need a different success function from product-development work. `product-lead` and `tech-lead` are designed to preserve product quality: they capture intent, design carefully, maintain docs, manage architecture taste, and route post-implementation QA/KB work. Those priorities are actively wrong for fixed benchmark tasks where the evaluator has already specified the requirement.

The benchmark package therefore uses benchmark-specific agents:

| Agent | Role |
|---|---|
| `benchmark-lead` | Primary orchestrator for one externally specified benchmark task. Chooses configured benchmark profiles, delegates bounded attempts, verifies, and reports comparable evidence. |
| `benchmark-runner` | Generic bounded implementation attempt. Optimizes for verifier pass with a minimal task-scoped diff. |
| `benchmark-claude-runner` | Claude-qualified bounded implementation attempt. Exists so Claude Code can discover a native benchmark subagent when native Claude delegation is useful. |

The key boundary is not "less rigor". It is a different optimization target: verifier-accepted behavior first, then minimal task-scoped diff, then cost/time traceability and honest failure evidence.

## Run Contract

Benchmark tasks are already specified by an external evaluator, task statement, or harness. Agents must not ask the human to restate requirements. Missing task statement, working directory, verifier, or other required inputs are blocker conditions to report, not clarification prompts.

Invalid shortcuts are part of the contract. Benchmark agents must not edit benchmark tests, verifiers, hidden-data access paths, grader config, or task metadata to make a run pass. They must not leak held-out data, hide failed attempts, cherry-pick only successful evidence, or reinterpret the metric after seeing results.

## Model And Routing Decisions

The package inherits model aliases and descriptions from `meridian-base`. It intentionally does not define local `[models]` entries or `[settings.model_visibility]` overrides. Benchmark routing should select among configured agent profiles using active model descriptions and task evidence; it should not invent ad hoc model aliases during a run.

This keeps benchmark comparisons tied to named, inspectable profiles instead of one-off model strings embedded in prompts. A report should say which profiles and aliases ran, not merely which provider family was used.

## Claude-Native Materialization

The package uses `[settings.meridian.agent_copy]` with `harnesses = ["claude"]` so Claude-qualified benchmark subagents are emitted into downstream `.claude/agents/` while Meridian still uses `.mars/agents/` for normal spawn routing.

This is deliberate. `.mars/agents/` remains Meridian's compiled read surface, but Claude Code's native subagent discovery only sees `.claude/agents/`. The benchmark package needs both surfaces when evaluating composed harness behavior involving Claude-native delegation.

Validated behavior from the initial scaffold: source-package `mars check`, `mars validate`, and `mars sync` were clean; a downstream smoke consumer materialized the `.mars` benchmark agents and `.claude/agents/benchmark-claude-runner.md`.

## Reporting And README Claims

Benchmark README claims should stay runner-agnostic until real benchmark runs exist. It is valid to say the package supports DeepSWE, Terminal-Bench, or local verifiers as externally specified task sources. It is not valid to claim improved pass rate, lower cost, or superior harness composition without actual run data.

Every benchmark report should include these fields when available: task identifier and source, working directory or benchmark environment, verifier command or source, profiles/model aliases used, attempts including failures, final verifier result, changed files and diff summary, elapsed time, token/cost fields, invalid shortcuts avoided or suspected risks, failure reason when not accepted, and artifact paths.

The likely headline comparison metric is cost per accepted task: total run cost divided by verifier-accepted benchmark tasks. Treat it as a future evaluation metric, not a current result claim.

## Vocabulary

| Term | Meaning |
|---|---|
| Benchmark task | An externally specified coding task with an evaluator or verifier. The agent should not reinterpret the requirement or ask the human to restate it. |
| Verifier | The benchmark-provided mechanism that decides whether a task is solved. Passing it is the primary success signal, but the agent must not edit or game it. |
| Benchmark orchestrator | The primary agent that owns one benchmark task from prompt to final report. |
| Benchmark subagent | A bounded implementation profile optimized for benchmark execution, optionally emitted as a Claude-native subagent when harness-qualified. |
| Configured agent profile | A named profile with configured model, harness, tools, and prompt. Runtime routing selects among profiles; it does not create new aliases. |
| Bounded attempt | One coherent solve pass ending in a verifier run, a blocker, or a recorded stop decision. |
| Minimal task-scoped diff | The smallest safe patch that satisfies observable benchmark behavior, excluding broad refactors, docs churn, style-only cleanup, and unrelated maintainability work. |
| Cost/time traceability | Enough run metadata to compare benchmark policies: profiles, models, elapsed time, token/cost fields, attempts, and accepted/failed status. |
| Cost per accepted task | Total run cost divided by the number of verifier-accepted benchmark tasks. |

## Related

- [overview.md](overview.md) — prompt package model and composition
- [meridian-base.md](meridian-base.md) — model-alias authority and foundation agents
- [meridian-dev-workflow.md](meridian-dev-workflow.md) — product-development workflow that benchmark agents intentionally do not reuse as the benchmark lead path
- [../../concepts/package-management/targeting.md](../../concepts/package-management/targeting.md) — `.mars/` read surface and native harness directory projection
- [../../decisions/package-management.md](../../decisions/package-management.md) — package-management decisions behind targeting and source discovery

## Provenance

- Work item: `work:mellow-hollow-brood`
- Prior session: `chat:c4841`, primary spawn `p4901`
