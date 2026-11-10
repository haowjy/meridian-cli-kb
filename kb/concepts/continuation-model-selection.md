# Continuation model selection

A continue is constrained by its target, not frozen to its old model. An explicit `--model` is accepted with a warning about cache and cost; Mars revalidates it against the recorded harness's current targets and exclusions, and a literal canonical or provider pin keeps an alias from drifting. Non-routing policy and the original launch snapshot are unchanged. The override rules are in the source checkout's `src/meridian/lib/launch/.context/CONTEXT.md`.

A plain continue (no fork, no `--model`) runs the model the session last executed, read through the single model-observation writer, and falls back to the selection Meridian accepted at startup. A tracked session with neither must be continued with an explicit `--model`: guessing could silently change models mid-conversation. A recording failure is a coordination error and never switches the runtime model or replays the turn.

Startup records the accepted selection against the same generation and attempt as the native session, and binds to the real native ID, not a provisional one.

## OpenCode

OpenCode takes different model shapes for session creation and for prompts, and named-model resume is unsupported: it fails before startup rather than silently using the harness default. A failed creation or prompt finalizes the single launch attempt ([launch system](../architecture/launch-system.md#a-harness-turn-runs-once)).
