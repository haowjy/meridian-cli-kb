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

### Continuation model selection

Exact continuation can request a model within the recorded harness's current target and exclusion constraints. A later plain continuation uses Meridian's accepted selection, rather than guessing the last model executed by the harness. Identity must bind to the actual native session, not a provisional seed. See [continuation model selection](../concepts/continuation-model-selection.md).

OpenCode uses different model shapes for native session creation and prompt invocation. Readiness polling must be idempotent and precede the turn; retrying the turn can repeat side effects.
