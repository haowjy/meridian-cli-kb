# Decision Index

The reasons behind current choices, by topic. Each is a current choice, open to change when its reason stops holding or the owner asks for something else.

## Domain Records

| Domain | What its records decide | Page |
|---|---|---|
| State layer | Identity, stores, atomicity, locks, reconciliation, typed rows | [decisions/state.md](architecture/state-system/overview.md) |
| History storage | Transcript authority, disposable indexes, retention, restore, and native capture | [decisions/history-storage.md](architecture/state-system/portable-history.md) |
| Native session identity | Immutable chat→native key, exact-target entry evidence, one binding and drift rule, run-boundary outcomes, no discovery, paths move only through `relocate` | [decisions/native-session-identity.md](architecture/native-session-binding.md) |
| Native-only history | Native transcripts as the only conversation source, old-data policy, search projection, run facts, retention | [decisions/native-only-history.md](architecture/native-transcript-reads.md) |
| Native source argument admission | Identity-overriding passthrough refusal | [decisions/native-source-argument-admission.md](architecture/native-session-binding.md) |
| Launch composition | Prepare/bind, policy replay, inventory, capability-to-command guidance, capability gates | [decisions/launch.md](architecture/launch-system.md) |
| Managed processes | Managed primaries, process scope, cancellation and cleanup | [decisions/launch-process-ownership.md](architecture/process-scope.md) |
| Idle notifications | Push, cache warning, running work, delivery, TTL detection | [architecture/idle-notifications.md](architecture/idle-notifications.md) |
| Completion | When a child closes, the report rule, one idle limit, no work limit, close cancels descendants, the hold as one table | [decisions/completion-drain.md](decisions/completion-drain.md) |
| Waiting and sessions | Wait barriers, goals, output, continue/fork/from, session browse re-entry | [decisions/launch-session-initiation.md](concepts/session-initiation.md) |
| Session-browse TUI | Picker-specific framework choice | [decisions/tui-framework.md](architecture/session-browse.md) |
| Session-browse picker | Columns, transcript on demand, staleness, search, lineage view | [decisions/session-browse.md](architecture/session-browse.md) |
| Static typing | Pyright strictness, suppression syntax, boundary narrowing, private-name policy | [decisions/static-typing.md](decisions/static-typing.md) |
| Codebase cleanup | Cleanup program scope, deferrals, and what phase 1 deleted or left open | [decisions/codebase-cleanup.md](decisions/codebase-cleanup.md) |
| Harness compatibility | Harness-specific launch constraints | [decisions/launch-harness-compatibility.md](architecture/claude-native-sessions.md#tui-trampoline) |
| Startup / health / sandbox | Startup descriptors, doctor, bootstrap and projection policy | [decisions/startup-health-sandbox.md](architecture/startup-pipeline.md) |
| Package management | Mars compilation, targeting, ownership, hooks, recovery, engine constraints | [decisions/package-management.md](decisions/package-management.md) |
| Model resolution | Alias routing, candidate policy, prompting ownership | [decisions/model-resolution.md](decisions/model-resolution.md) |
| Workspace | Schema, path resolution, permission projection and migration | [decisions/workspace.md](architecture/workspace.md) |
| Tool permissions | How profile `tools`/`sandbox`/`approval` reach each harness: Mars-owned names, native enforcement, the Codex shell hook, `approval: auto`, reported gaps, allowlists, list form deprecated, headless `ask`, platform rules, permission sets that add across surfaces | [decisions/tool-permissions.md](architecture/tool-permissions.md) |
| Bundle field consumption | Every launch-bundle field applied, reported or deliberately unread; `harness-overrides` on Codex, compaction projection, `tools.mcp` as permission names | [decisions/bundle-field-consumption.md](architecture/mars-launch-bundle.md) |
| Sandbox platform access | What Meridian grants a sandboxed launch: exact project lifetime lock, Meridian commands, Claude and Codex child-spawn escapes, PID-namespace compound-launch refusal, the network field, and the sandbox/spawn/projected-network child guardrail | [decisions/sandbox-platform-access.md](architecture/sandbox-projection.md) |
| Telemetry | Envelopes, sinks, segments, querying and retention | [decisions/telemetry.md](decisions/telemetry.md) |
| Testing | Behavior-owned tiers and evidence gates | [decisions/testing.md](decisions/testing.md) |
| Primary inject | How `spawn message` reaches every live session: one control socket, Claude's inbox, the sender frame and address, posted or delivered, no resend after a write | [decisions/primary-inject.md](decisions/primary-inject.md) |
| Harness connection seams | Surviving seams: connection ownership, HITL handler protocol, passthrough boundary, spawn depth cap | [decisions/connection-seams.md](concepts/harness-abstraction.md#seams) |
