# Skills

`skills.load` gives an agent a skill from its first turn; `skills.available` points it at a skill it can load later. Mars lowers a portable skill schema to each harness ([skill compilation](https://github.com/haowjy/mars-agents/blob/main/docs/config/skill-compilation.md)); Meridian composes the loaded skills into the prompt on every harness. Delivery rules: the source checkout's `src/meridian/lib/launch/.context/CONTEXT.md`.

## Discovery and loading are separate axes

`model-invocable` controls whether a model can discover a skill, and `user-invocable` whether a human can start it; neither decides whether a profile loads it. A profile can therefore load a role-specific skill without advertising it to every model. A non-invocable skill listed as available can be named but may not be loadable through the harness's own discovery, and a harness's native skill list can omit a loaded skill, which is why loaded skills are delivered through composition.

## Variants are chosen by harness, once

A package can supply a harness-specific body while keeping the base skill's metadata. Selection is by harness, not model, so variants don't multiply with model aliases; model-specific selection wasn't built because no package needed it, and would belong in Mars rather than a Meridian resolver. Mars picks the body once for both native projection and the launch bundle, Meridian never picks again, and an exact continue reuses the recorded snapshot. Mars still indexes model-variant directories and warns about their layout without selecting them, which can suggest support that doesn't exist.

## Lost fields

Where a harness has no native equivalent for a field (Codex has no `user-invocable`), lowering records the loss. Skill-field losses print only under `--verbose`: a skill carries no tool or network restriction, so losing one changes who can start it but gives no target any access. This follows the lost-restriction invariant in meridian-cli's root `AGENTS.md`.
