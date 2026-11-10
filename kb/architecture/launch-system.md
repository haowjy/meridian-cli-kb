# Launch system

The launch pipeline separates preparation from binding so expensive resolution can be reused for preview and execution without separate composition paths drifting. See the source checkout's `src/meridian/lib/launch/AGENTS.md` and `.context/CONTEXT.md`.

## Continuation selection and attempt identity

Primary and spawn continuation share target-constrained selection. An explicit
`--model` is accepted and warned on every request; Mars revalidates current
targets and exclusions under the same harness, while a literal
canonical/provider pin prevents alias drift. The rule is not a model freeze or
prohibition, and it does not change non-routing policy or original launch
snapshots.

Plain continuation uses the Meridian selection recorded at `accepted-running`,
not a model observed from the native harness's last execution. `SessionAttempt`
binds that selection to the same generation and attempt. Native-ID callbacks are
authoritative; a derived fresh ID remains only a provisional row hint. Append
failure is a coordination error and never triggers runtime model switching.

The legacy increment (baseline `c3087ceb`) performs original-generation lookup
read-only. An original snapshot or partial historical model is carried through
`SessionRequest` and seeded once at session scope under the store lock; failed
continuation is never used as the initial value. New-protocol continuation with
missing acceptance or absent tracked original history requires an explicit model
instead of guessing. Native raw IDs remain literal, and ambiguous
tracked/untracked/mixed ownership requires explicit `--harness`; known p/c
references retain their recorded harness rather than being silently replaced
by file detection. Native reference `833dc1f2` covers these rules.

The spawn native-ID reference form is now resolved for tracked provenance:
coordinated final gates/PR/release.

## control_root / task_cwd Split

The authority root selects project configuration and context; the task directory selects the checkout and relative reference anchor. This lets work happen in a worktree without changing which project controls the launch. See [worktree anchoring](../decisions/spawn-cwd-worktree-anchor.md) and [reference resolution](../concepts/reference-resolution.md).

## One Launch-Level Attempt

A harness turn executes once: replay after a terminal failure could duplicate model work or side effects. Bounded idempotent readiness polling before submission is separate. See [launch decisions](../decisions/launch.md#harness-turn-failures-finalize-without-replay).

## Child Env Boundaries

`MERIDIAN_SECRET_*` values are stripped from harness children, including primaries. A child cannot use them for in-session notifications; ordinary inherited environment values can leak into transcripts. See [notification credentials](idle-notifications.md#smtp-password-from-a-file).

## _MERIDIAN_HARNESS Child Env Injection

Each launch sets its own harness identity rather than inheriting an ancestor's. Wait uses the calling harness's yield interval, not its child's. See [spawn wait](../concepts/spawn-wait-barrier.md).

## Accepted startup selection and OpenCode transport (2026-09)

Continuation model selection and its current audit boundaries are summarized in
[the continuation model-selection concept](../concepts/continuation-model-selection.md).

All three driving paths record **model/policy** acceptance at startup through the shared
`SessionAttempt`. This is model/policy acceptance, separate from the chat's native
key ([native session binding](native-session-binding.md)). Streaming-serve binds its pending selection through the
identity callback when available, otherwise through existing post-run artifact
extraction. Native-spawn fork child isolation prevents the parent from being
recorded as the child.

OpenCode creation sends `{providerID, id}`. Every invocation's initial prompt—an
explicit prompt or a plain recorded continuation—sends `{providerID, modelID}`;
only subsequent resident/injected messages omit the model. Invalid or timed-out
creation does not downgrade to `{}`. Bounded transport-local session readiness
polling is separate from launch orchestration and must remain idempotent and
pre-turn. A creation or prompt failure finalizes the one launch attempt. Named-model
resume remains unsupported, with no UI replacement.

Tracked raw native IDs passed to `spawn --continue` now resolve against their
recorded session provenance. An explicitly supplied older native ID is retained
when it identifies the matched session; otherwise continuation uses that
session's recorded spawn ID, with exact chat/harness checks. Untracked or pruned
provenance fails honestly. `--harness` disambiguates ownership only; it does not
invent history or override recorded identity. Primary, spawn, and c-prefixed
native sequences preserve their recorded models. Coordinated T1–T19 audit, full
suites, readiness, final reviews, and PR/release gates remain outstanding. No
model-observation or index work is implied by this increment.
