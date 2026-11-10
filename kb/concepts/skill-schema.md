# Skill Schema and Variants

Mars lowers a portable skill schema to each harness; some native surfaces cannot represent every field. The independent model and user invocability axes express discovery and human invocation separately. Neither controls whether a profile explicitly loads a skill. See [skill loading](skill-loading.md) and `docs/config/skill-compilation.md` in mars-agents.

## Per-Harness Lowering

**Lossiness model:** When a harness has no native equivalent for a field (e.g.,
Codex has no `user-invocable`), the lowering records a lossiness entry. `mars check`
and `mars init` list these entries, and a sync on current `main` warns about each
one. mars-agents #324, in review, would move skill entries behind
`mars sync --verbose`: losing `user-invocable` changes who can start a skill, and
a skill carries no tool or network restriction, so no target gains any access
([Warnings](../decisions/package-management.md#warnings)).
[Skill compilation](https://github.com/haowjy/mars-agents/blob/main/docs/config/skill-compilation.md)
in mars-agents owns what prints where.

## Skill Variants

A package may supply a harness-specific body while retaining the base skill metadata. Selection is by harness, not model, so variants do not multiply with model aliases. Mars selects the body for both native projection and the launch bundle; Meridian does not select it again. See [package management decisions](../decisions/package-management.md).
