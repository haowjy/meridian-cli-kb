# Model Policies

Model policies refine execution settings for a matched model and may provide prelaunch fallback candidates. Overlay rules precede profile rules for settings matching; an empty overlay does not erase profile rules. See [config precedence](../config-precedence.md#routing-vs-policy-fields) and `src/build/policy/` in mars-agents.

## Fallback Participation

Fallback handles prelaunch harness availability, not failures after launch. Concrete profile rules provide candidates in order; glob rules and `no-fallback` rules do not. Explicit model requests disable model fallback. See [model resolution decisions](../../decisions/model-resolution.md#target-constrained-model-policy-fallback).

## Model Inventory Is a Separate Concern

Mars separates Possible discovery, display-only Curated rules, and Selection
routing. `model-policies` participate in Selection; Curated rules affect only
the human-facing `models list` view and never filter aliases, resolve results,
or launch-bundle routes. See the [model-catalog decision](../../decisions/model-resolution.md#model-catalog-layers-2026-09)
for command responsibilities and cross-repo boundaries.

