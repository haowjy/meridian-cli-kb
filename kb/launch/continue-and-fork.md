# Continue, fork and `--from`

`--continue` resumes the same transcript; `--fork` branches it with identity-shaping inputs locked, so the branch keeps the source's prompt-cache prefix; `--fork-fresh` allows identity changes; `--from` starts a fresh session with prior output as reference material in the user turn. The flag contract is in meridian-cli `docs/commands.md`; the override rules in the source checkout's `src/meridian/lib/launch/.context/CONTEXT.md`. Why per-run material sits in the user turn: [launch system](launch-system.md#the-system-prompt-depends-only-on-what-defines-the-conversation). Whether `--fork` actually preserves the cache, and so whether it and `--fork-fresh` should merge, is unmeasured ([#701](https://github.com/haowjy/meridian-cli/issues/701)).

## Continue replays the recorded launch

An exact continue starts from the recorded launch contract (task directory, work attachment, session identity and policy snapshot) instead of recomputing it from ambient configuration, and accepts only the overrides its entry point allows. It requires a new prompt: replaying the original task would resend completed work instead of saying what to do next. The conversation's settings and native identity come from the chat it continues ([native session binding](../state/native-session-binding.md#settings-belong-to-the-conversation)).

## Model on continue

A continue is constrained by its target, not frozen to its old model. An explicit `--model` is accepted with a warning about cache and cost; Mars revalidates it against the recorded harness's current targets and exclusions, and a literal canonical or provider pin keeps an alias from drifting. Non-routing policy and the original launch snapshot are unchanged, except that project-wide tool denies are refreshed from Mars on every continue and fork ([tool rule precedence](../permissions/tool-rule-precedence.md#continue-and-fork-pick-up-the-denies-in-force-now)).

A plain continue (no fork, no `--model`) runs the model the session last executed, read through the single model-observation writer, and falls back to the selection Meridian accepted at startup. A tracked session with neither must be continued with an explicit `--model`: guessing could silently change models mid-conversation. A recording failure is a coordination error and never switches the runtime model or replays the turn. Startup records the accepted selection against the same generation and attempt as the native session, and binds to the real native ID, not a provisional one.

OpenCode takes different model shapes for session creation and for prompts, and named-model resume is unsupported: it fails before startup rather than silently using the harness default. A failed creation or prompt finalizes the single launch attempt ([launch system](launch-system.md#a-harness-turn-runs-once)).

Re-entry from the session picker is in [session browse](session-browse.md).
