# Launch Composition and Policy

See [launch architecture](../architecture/launch-system.md) and `src/meridian/lib/launch/.context/CONTEXT.md` for the mechanism.

### D-from-request-boundary: `--from` is resolved once at the request boundary; the request prompt stays raw (2026-10-21) {#d-from-request-boundary}

Resolve prior context once at the request boundary and keep the request prompt raw. A composed prompt in a persisted background request was composed again by the worker, duplicating references and prior context.

### D-control-root-task-cwd-split: Split execution_cwd into control_root + task_cwd

The configuration authority root and the task directory can differ. Keeping them separate lets an agent work in a nested directory or worktree without changing the project that owns its configuration.

### D-one-launch-attempt: harness-turn failures finalize without replay

Each harness turn has one launch-level attempt. In an audit, 21 of 22 retrying lineages failed and 63 of 64 subsequent attempts failed; retries also hid causal quota errors and can repeat tool or native-create side effects. Bounded pre-turn idempotent readiness polling remains separate.

### D-primary-approval: Managed-primary Codex approval routing

A managed Codex primary owns app-server approval requests; the attached TUI cannot answer them directly. Meridian must route them to the user. Headless spawned-agent requests are rejected.

### D-bundled-meridian-skills: Meridian-owned operations guidance versions with the CLI

Meridian operations guidance ships with the CLI so its commands and advice change together. Skills provided through Mars share the package skill home without changing committed package state; when provision fails the wheel is the fallback. Bundled guidance must retain judgment, not just flag syntax. See `src/meridian/skills/AGENTS.md`.

### D-continue-replays-recorded-launch-contract: same-session continue is not live policy recomputation

Tracked exact continue starts from the recorded launch contract rather than recomputing it from ambient configuration. The entry point can permit specific overrides; see [session initiation](../concepts/session-initiation.md).

### Continue requires input {#d-continue-needs-prompt}

A continue requires a prompt or prompt file. Replaying the original task would resend completed work rather than tell the agent what to do next.

### D-stable-system-prompt: the system prompt depends only on what defines the conversation (2026-10) {#d-stable-system-prompt}

Per-run values go in a spawn’s first user turn so sibling runs share the system prefix. A primary’s identity and goal remain system guidance because continue has no first task turn. Work goals are read through `meridian work show`, so edits do not stale or invalidate the prefix. An exact continue recomposes from its snapshot instead of replaying stored bytes: replay can misstate the surface or discard a new goal.

### D-skill-single-path: Loaded skills are composition documents on every harness

Loaded skills are composed for every harness. A native skill inventory is not equivalent to loading: it can omit a loaded skill that is not model-invocable. Multiple conditional delivery paths previously lost skills when one path overwrote another.

### D-spawn-capability-gate: Capability-gated spawn knowledge injection (PR #328, 2026-06)

Spawn guidance and agent inventory are injected for profiles with delegation capability. An opt-in skill alone was repeatedly not loaded; unconditional guidance would waste context on leaf agents.

### D-capability-command-mapping: Packages name capabilities; launch guidance maps commands (2026-10, v0.14.3) {#d-capability-command-mapping}

Prompt packages name portable capabilities; Meridian launch guidance maps them to CLI commands. Packages are also used by runtimes with app tools and no shell, so embedding Meridian syntax makes their judgment nonportable.

### Prepare before reserving a spawn

Preparation precedes spawn reservation; setup errors need not create a row. A launch lock protects a published row from being stranded. See [spawn identity](spawn-identity.md#d-continue-reopens).

### D-pi-spawn-native-discovery: Pi spawns keep native skill and context discovery

Pi needs native skill and context discovery for available skills and project instructions. Ambient extensions remain separately controlled because they execute code and can conflict with managed extensions.

### D-continuation-target-constrained-selection: explicit continuation requests remain target-constrained

**Decision:** Exact primary and spawn continuation accept an explicit
`--model` request, and every such request emits a warning. Mars revalidates the
current target and exclusion constraints under the same harness; the literal
canonical/provider pin is retained so an alias cannot drift during continuation.
This is target-constrained selection, not a model-freeze or model-prohibition
rule. Non-routing policy and the original launch snapshots remain unchanged.

After the explicit request is accepted, a later plain continuation uses the
Meridian model selection recorded at `accepted-running`. It does **not** discover
or infer the model from the last model executed by the native harness. A
`SessionAttempt` binds that accepted selection to the same generation and
attempt, while authoritative native-ID callbacks identify the session. A newly
derived ID is only a provisional row hint until the authoritative callback
arrives.

If an append fails, continuation ends as a coordination error; it does not
switch runtime models to recover. Legacy original-generation lookup is
read-only against baseline `c3087ceb`; the original snapshot or partial
historical model is carried through `SessionRequest` and seeded at session
scope under the store lock. Failed continuation never becomes the initial
value. New-protocol missing acceptance or absent tracked original history
requires an explicit model—there is no guessing. Native raw IDs remain
literal, and ambiguous tracked/untracked/mixed ownership requires explicit
`--harness`; known p/c references retain their recorded harness and file
detection does not silently replace it. Native reference is `833dc1f2`.

The remaining work is the spawn native-fork/reference-form audit and
coordinated final gates/PR/release.

### D-primary-identity-delayed-binding: fresh and native-fork primary selection waits for identity

**Decision (2026-09, target-constrained-fallback):** Primary fresh and
native-fork launches do not bind a generated seed or source parent before the
actual harness identity exists. Selection and persistence wait for the
authoritative identity; a generated seed remains provisional and cannot bind
the source parent early. Claude's existing transcript/trampoline identity
detector remains the identity observation mechanism; this increment adds no
model discovery.

If identity binding fails, the failure propagates while adapter cleanup and
session stop still run. This keeps the binding error visible without leaking a
live adapter or masking the original error. The following remain open:
streaming-serve recording, OpenCode streaming named-model transport, spawn
native-fork/reference-form audit, and coordinated final gates/PRs.


### D-streaming-opencode-target-constrained-fallback: accepted startup selection is attempt-bound

**Decision (2026-09, target-constrained-fallback):** All three driving drivers record
accepted startup selection through the shared `SessionAttempt`. Streaming-serve
binds a pending selection using the identity callback when available, or the
existing post-run artifact extraction path. Native-spawn fork child isolation
does not record the parent as the child.

OpenCode preserves the native HTTP model contract: session creation uses
`{providerID, id}`, while every invocation's initial prompt—including an
explicit prompt or a plain recorded continuation—uses `{providerID, modelID}`.
Only subsequent resident/injected messages omit the model. Invalid or timed-out
creation does not downgrade to `{}`. Bounded transport-local session readiness
polling is separate from launch orchestration and must remain idempotent and
pre-turn. Meridian does not replay the harness launch after a creation or prompt
failure: one invocation has one create path.

The primary OpenCode named-model resume remains unsupported and has no UI
replacement. Tracked raw native IDs for `spawn --continue` are now accepted.
An explicitly supplied older native ID is retained when it matches the resolved
session; otherwise the matched session's recorded spawn ID is used, with exact
chat/harness checks. Untracked or pruned provenance fails honestly, and
`--harness` disambiguates ownership rather than history. Native primary/spawn/
c-prefixed sequences preserve recorded models.
