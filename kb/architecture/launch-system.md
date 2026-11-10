# Launch system

A launch is prepared once and then bound: preparation resolves policy and composes content, and binding (unrelated to [native session binding](native-session-binding.md)) supplies per-run values and can run for a preview. One composition path serves preview and execution, so the two can't drift. Contracts: the source checkout's `src/meridian/lib/launch/AGENTS.md` and `.context/CONTEXT.md`, which also cover exact continue, prompt placement, skill delivery, task directories and the child environment. Managed Codex approval requests: `src/meridian/lib/harness/.context/CONTEXT.md`.

## Composition by meaning

Content is classified by meaning (system instructions, task context, the user's request) before any harness-specific projection, so each adapter picks its native channels without composition depending on harness identity.

## Authority root and task directory

The authority root selects project configuration and context; the task directory selects the checkout and the anchor for relative references. Keeping them separate lets an agent work in a nested directory or worktree without changing which project's configuration governs it ([file references](../concepts/reference-resolution.md), [work items](state-system/work-items.md#checkout-pins)).

## A harness turn runs once

A terminal error, transport failure, timeout, guardrail or identity failure finalizes the launch; nothing replays the turn. In an audit, 21 of 22 retrying lineages failed and 63 of 64 retried attempts failed; retries also hid the real cause (often a quota error) and could repeat tool calls or native session creation. Readiness polling before the turn is separate and allowed, because it is idempotent.

## The system prompt depends only on what defines the conversation

Per-run values go in the first user turn so sibling runs share the system prefix and its prompt cache, and so prior reports, which can carry untrusted text, never gain the system prompt's authority. A primary's identity and goal stay in system guidance because a continue replays no user turn. Placement rules: `src/meridian/lib/launch/.context/CONTEXT.md`.

## Continue replays the recorded launch

An exact continue starts from the recorded launch contract (task directory, work attachment, session identity and policy snapshot) instead of recomputing it from ambient configuration, and accepts only the overrides its entry point allows. It requires a new prompt: replaying the original task would resend completed work instead of saying what to do next. Model choice on continue is in [continuation model selection](../concepts/continuation-model-selection.md).

## Skills and guidance

- **Delegation guidance follows capability.** Spawn guidance and the agent inventory are injected for profiles that can delegate. An opt-in skill alone was repeatedly not loaded; injecting it everywhere would waste leaf agents' context.
- **Packages name capabilities; Meridian maps them to commands.** Prompt packages are also used by runtimes with app tools and no shell, so Meridian syntax inside them would make their guidance unportable. Meridian's own operations guidance ships with the CLI so commands and advice change together.
- **Pi keeps native skill and context discovery** for available skills and project instructions. Ambient Pi extensions stay separately controlled, because they run code and can conflict with Meridian's managed extensions.

