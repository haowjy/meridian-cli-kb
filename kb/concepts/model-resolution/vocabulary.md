# Model Resolution Vocabulary

Terms covering model alias resolution, agent profile loading, skill routing,
approval policies, and fallback-candidate semantics. For the full vocabulary
index see [../../vocabulary.md](../../vocabulary.md).

| Term | Definition | See also |
|---|---|---|
| **Alias entry** | A Mars model catalog entry mapping a human-readable alias (e.g., `sonnet`) to a concrete model ID with resolved harness, default effort, and default autocompact. Defined in `mars.toml` or loaded from `mars models list --json`. | [aliases-and-routing.md](aliases-and-routing.md) |
| **Approval mode** | A per-spawn policy controlling how tool calls are approved. Four values: `default` (harness decides), `confirm` (user approves each call), `auto` (auto-approve safe operations), `yolo` (approve everything). Resolved through the precedence chain. | [../../architecture/sandbox-projection.md](../../architecture/sandbox-projection.md) |
| **Model alias** | A human-readable model name (e.g., `sonnet`, `opus`) resolved at launch time to a concrete model string by Mars. Aliases are defined in the Mars model catalog, not hardcoded in Meridian. | [aliases-and-routing.md](aliases-and-routing.md) |
| **Fallback candidate chain** | The ordered candidate list Meridian tries when harness-availability fallback is active: the demoted base candidate first, then eligible `model-policies` rules in list order. | [aliases-and-routing.md](aliases-and-routing.md) |
| **Model policies** | An agent profile field (`model-policies:`) overriding model-specific settings and contributing eligible fallback candidates through rule order. | [model-policies.md](model-policies.md) |
| **Resolved policies** | The output of the launch policy resolution stage (`ResolvedPolicies`). Contains the selected model, harness, agent profile, and skill policy outcome. Produced by `resolve_policies()` from the `RuntimeOverrides` stack. | [../../architecture/launch-system.md](../../architecture/launch-system.md) |
| **Runtime overrides** | The shared override struct (`RuntimeOverrides`) for model, harness, agent, effort, sandbox, approval, autocompact, and timeout. Resolved by first-non-None precedence across layers: CLI > env > profile > config > alias defaults > harness fallback. | [../../architecture/launch-system.md](../../architecture/launch-system.md) |
| **Skill ref** | An unresolved skill reference in an agent profile's `skills:` frontmatter field. Mars resolves each ref by searching the requester's package, then the dependency closure, then the full resolved graph. Unresolved refs emit `SkillNotFound`. | [../../concepts/package-management/resolution-algorithm.md](../../concepts/package-management/resolution-algorithm.md) |
| **Skill registry** | The catalog of available skills (`SkillRegistry`). Indexes skills from `.mars/skills/` and provides variant-aware lookup by (harness, model token), (harness, canonical model ID), (harness, None), then base skill. | [agent-profiles.md](agent-profiles.md) |
| **Skill variant** | A harness- and model-specific version of a skill at `.mars/skills/<name>/variants/<harness>/<model>/SKILL.md`. Selected over the base skill when the harness and model match. Exact-match only; no globbing. | [../../concepts/skill-schema.md](../../concepts/skill-schema.md) |
