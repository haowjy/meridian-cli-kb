# Model Resolution

Mars owns aliases and runtime routing. Package authors can evolve model vocabulary without a Meridian release; model-name prefixes do not prove harness support or account readiness. See [aliases and routing](../concepts/model-resolution/aliases-and-routing.md).

### Model catalog layers (2026-09) {#model-catalog-layers-2026-09}

**Decision:** Keep three concerns separate across Mars and Meridian:

- **Possible** is a derived view of discovered harness–model evidence, not a
  second persisted catalog.
- **Curated** applies authored display choices to that view. It affects only
  the human `models list` presentation; it does not affect alias resolution,
  sync/materialization, or Selection.
- **Selection** is runtime route choice from support/auth evidence and authored
  policy. It does not read Curated. A supported authored harness preference
  remains meaningful when auth is unchecked if model support is confirmed or
  constrained (mere installed-harness passthrough is insufficient), subject to
  the latest failed credential-gated listing. A successful credential-gated
  listing can provide auth evidence; a public model listing alone cannot.

The command boundary follows the same separation: `models list` renders the
human-facing inventory; `models aliases` supplies alias definitions;
`models catalog` supplies raw provider data; `models resolve` explains alias
resolution, and the launch bundle makes the final runtime routing decision. A
list entry, alias preference, or catalog record alone does not establish a
runnable route.

Mars and Meridian must release in dependency order: land and release the Mars
producer command/schema change first, then update Meridian's exact Mars pin and
release the migrated consumer. Do not add a dual-command compatibility path to
hide a mismatched pin. The current implementation details (curation files and
matching, cache/probe semantics, route evidence, and Meridian subprocess
handling) belong in each source tree's colocated docs, not here.

### Model is optional — harness alone is sufficient for launch execution {#model-optional-empty-model}

An absent managed model lets the harness choose its default. A stronger harness override can also clear an incompatible model. Older snapshots represent absence as an empty string; a fake model token would instead be sent as an override.

### Project-scoped agent overlays {#d72-agentsname-overlay-config-as-the-project-scoped-override-surface}

Project-specific agent policy belongs in Mars overlays, not generated agent files that sync overwrites. The overlay rules and precedence are in [model policies](../concepts/model-resolution/model-policies.md) and [config precedence](../concepts/config-precedence.md#routing-vs-policy-fields).

## Target-constrained model-policy fallback

Enabled targets bound routing before availability and authentication checks. Profile model-policy entries provide backup candidates independently of which settings rule matches; otherwise changing the primary can hide earlier backups. See [fallback participation](../concepts/model-resolution/model-policies.md#fallback-participation).

### Harness-specific model IDs via RunnablePath {#d76-harness-specific-model-ids-via-runnablepath}

A canonical model can need different native IDs, such as a provider-qualified ID for OpenCode and a bare one for Codex. Mars supplies the selected harness-specific value; every native model selection boundary must use it. An explicit harness narrows routing but does not grant permission or establish support or authentication.

### Agent-first prompting guidance

Prompting guidance resolves an agent through the same policy as launch before looking up its model guidance. A shallow duplicate resolver can give guidance for a model that routing later replaces. Agent names are tried before aliases because callers normally ask about the agent they plan to spawn. Guidance is on demand rather than added to every launch prompt.

### A matching model-policy rule beats the profile default

A matching profile model-policy rule beats the profile's top-level value; otherwise a top-level effort would disable the profile's own per-model rules. Overlay scalars have higher precedence. See [config precedence](../concepts/config-precedence.md#routing-vs-policy-fields).

### Each harness lowers effort to what it supports

Each harness lowers resolved effort to its own capability and reports unsupported effects. Current adaptive-thinking Claude models did not turn reasoning off in the tested configurations, so projecting switches as if they did would mislead users. See `docs/agent-profiles.md` and `src/meridian/lib/harness/projections/.context/CONTEXT.md`.
