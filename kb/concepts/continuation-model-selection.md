# Continuation Model Selection and Startup Binding

This page covers how continuation selects a model and how startup choices are
recorded and transported. The four initiation modes and their re-entry behavior
are in [session initiation](session-initiation.md).

## Continuation model selection and identity

Continuation is target-constrained rather than model-freeze or model-prohibition.
Exact primary and spawn continuation accept explicit `--model` requests and warn
for each explicit request. Mars revalidates current targets and exclusions under
the same harness, and a literal canonical/provider pin prevents alias drift.
Non-routing policy and the original launch snapshots are unchanged.

A later plain continuation reads the Meridian selection recorded at
`accepted-running`; it never uses the native harness's observed last-executed
model as discovery. `SessionAttempt` binds that selection to the same generation
and attempt. A plan-assigned native ID is bound before exec; native-ID callbacks
confirm it or, when no ID was assigned, bind the first observation. Append failure terminates continuation with a
coordination error rather than switching the runtime model.

Legacy original-generation lookup is read-only against baseline `c3087ceb`.
The original snapshot or partial historical model travels through
`SessionRequest` and is seeded at session scope under the store lock; failed
continuation cannot become the initial value. Under the new protocol, missing
acceptance or absent tracked original history requires an explicit model.

Native raw IDs stay literal rather than becoming the latest chat ID. Ambiguous
tracked/untracked/mixed ownership requires explicit `--harness`; known p/c
references keep their recorded harness, and file detection never silently
replaces it. A tracked reference with no recorded harness refuses. These increments are validated against native reference
`833dc1f2`. For spawn continuation, tracked raw native IDs are accepted;
explicit older native IDs remain authoritative when they match the session,
otherwise the matched session's recorded spawn ID is used with exact
chat/harness checks. Untracked or pruned provenance fails honestly, and
`--harness` disambiguates ownership rather than history. Primary, spawn, and
c-prefixed native sequences preserve recorded models. Remaining C7 work is
primary provisional/trampoline identity and fork checks, streaming-serve
recording, OpenCode streaming model transport, and coordinated final gates/PR.

## Accepted startup selection and native transport

Startup selection is recorded by the shared `SessionAttempt` for all three
driving paths. Streaming-serve binds the pending selection through the identity
callback or existing post-run artifact extraction. A native-spawn fork isolates
the child without recording its parent as the child.

For OpenCode, session creation uses `{providerID, id}`. Every invocation's
initial prompt, whether explicit or a plain recorded continuation, uses
`{providerID, modelID}`; only later resident/injected messages omit the model.
Invalid or timed-out creation does not fall back to `{}`. The outer same-model
runtime retry policy is unchanged and remains per startup attempt. Primary
named-model resume remains unsupported and has no UI replacement.

A runtime C7 probe still shows raw native IDs passed to `spawn --continue`
erroring. The uniform reference contract is therefore still an audit item, not
settled behavior. Coordinated audit, suites, readiness, review, and release
work remain open.

> [!FLAG] **Needs human review**: This page says raw native IDs passed to
> `spawn --continue` still error, while [launch-system](../architecture/launch-system.md)
> says tracked raw native IDs now resolve against session provenance. Both claims
> are retained pending reconciliation. Flagged 2026-09-24.

## Related

- [Model resolution](model-resolution/overview.md) — alias/profile resolution and harness model projection
- [Native session identity](../decisions/native-session-identity.md) — immutable native key and exact continuation target
- [Native session binding](../architecture/native-session-binding.md) — cross-harness binding and observation seams
