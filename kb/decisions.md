# Decision Index

Decision pages own rationale, rejected alternatives and revisit conditions. Architecture pages own current mechanism. [Decision Records](decisions/overview.md) lists every record by topic and says how to add one.

## Awaiting review

| Decision | Record |
|---|---|
| A dropped field is reported unless its lowering site says why not (`Harmless { why }`). A lost restriction prints one line per target on every `add`, `sync` and `upgrade`, because a lock change does not prove the user was told. Unknown `[settings]` keys warn in `mars validate` only: `sync` stays silent because an older Mars must accept a newer config. A relative `PATH` for `mars adopt` and `mars build plugin` resolves against the working directory when it exists there, else against an explicit `--root`, because Meridian always appends `--root`. Rejected: a report-once gate on the lock, a digest in the lock, warning on every sync, root-only and cwd-only path rules | [Package management](decisions/package-management.md#warnings) |
| `spawn message` says posted or acknowledged, stated by the connection; a missing field reads as posted. Resend only on `DeliveryRefused`. A `meridian` shim for the launching build lives under the user home, because writable roots would run agent-written code outside the sandbox; `-P` covers the implicit working-directory entry and not `PYTHONPATH` (#876). `spawn --continue` requires a prompt | [Primary inject](decisions/primary-inject.md#d-delivery-kind), [Sandbox platform access](decisions/sandbox-platform-access.md#d-escape-loads), [Launch](decisions/launch.md#d-continue-needs-prompt) |
| No-arg wait adds at most the oldest 32 terminal rows from collect's shared owed scope while leaving blocked-on targets uncapped. Owed rows resolve once outside the poll loop, appear separately under `owed_spawns`, and do not affect wait counts, fail-fast, aggregate failure or exit code. The consumed set unions CLI observations, Pi delivery receipts and Pi clears. Existing observation leases remain the only temporary coordination; an occasional duplicate header is accepted instead of a new claim subsystem. Wait consumes before printing, so a kill in between loses output. | [Child result delivery](decisions/child-result-delivery.md#d-no-arg-wait-owed-bounded) |
| Proposed, not yet built: per-chat append-only journals become each chat's authority; cross-chat facts live in per-native-ID files. A one-time cutover splits `sessions.jsonl`, and later old-build appends are absorbed at command entry using offsets to prevent duplication. Alternatives include a global log with an offset sidecar, ingest on read, waiting for quiescence, and per-chat `state.json`. | [State](decisions/state.md#d-session-per-chat-journals) |
| Claude TTL detection uses the chat's launch-recorded native store, never the current cwd; Pi maps provider and retention mode to known, estimated, or unknown TTLs. | [Idle TTL detection](decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd) |
| Possible is derived evidence, Curated is display-only, and Selection is independent runtime routing. Separate list, aliases, catalog, and resolve APIs; release Mars before moving Meridian's exact pin. | [Model resolution](decisions/model-resolution.md#model-catalog-layers-2026-09) |

## Domain Records

| Domain | What its records decide | Page |
|---|---|---|
| State layer | Identity, stores, atomicity, locks, reconciliation, typed rows | [decisions/state.md](decisions/state.md) |
| History storage | Transcript authority, disposable indexes, retention, restore, and native capture | [decisions/history-storage.md](decisions/history-storage.md) |
| Native session identity | Immutable chat→native key, exact-target entry evidence, one binding and drift rule, run-boundary outcomes, no discovery, paths move only through `relocate` | [decisions/native-session-identity.md](decisions/native-session-identity.md) |
| Native-only history | Native transcripts as the only conversation source, old-data policy, search projection, run facts, retention | [decisions/native-only-history.md](decisions/native-only-history.md) |
| Native source argument admission | Superseded; identity-overriding passthrough refusal is what remains | [decisions/native-source-argument-admission.md](decisions/native-source-argument-admission.md) |
| Launch composition | Prepare/bind, policy replay, inventory, capability-to-command guidance, capability gates | [decisions/launch.md](decisions/launch.md) |
| Managed processes | Managed primaries, process scope, cancellation and cleanup | [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md) |
| Idle notifications | Idle push and cache-expiry warning, Claude's stateless permission-prompt notice, running work and the push, session role, notifications as hook events (D-notify-hook-events), notify channels and secrets; Meridian idle compaction removed (D-idle-compaction-removed) | [decisions/idle-notifications.md](decisions/idle-notifications.md), [decisions/idle-notify-channels.md](decisions/idle-notify-channels.md); current mechanism: [architecture/idle-adapters.md](architecture/idle-adapters.md) |
| Idle TTL detection | Launch-recorded native-store lookup and provider TTL mapping for idle scheduling | [decisions/idle-ttl-detection.md](decisions/idle-ttl-detection.md) |
| Completion | When a child closes, the report rule, one idle limit, no work limit, close cancels descendants, the hold as one table | [decisions/completion-drain.md](decisions/completion-drain.md) |
| Waiting and sessions | Wait barriers, goals, output, continue/fork/from, session browse re-entry | [decisions/launch-session-initiation.md](decisions/launch-session-initiation.md) |
| Session-browse TUI | Picker-specific framework choice | [decisions/tui-framework.md](decisions/tui-framework.md) |
| Session-browse picker | Columns, transcript on demand, staleness, search, lineage view | [decisions/session-browse.md](decisions/session-browse.md) |
| Static typing | Pyright strictness, suppression syntax, boundary narrowing, private-name policy | [decisions/static-typing.md](decisions/static-typing.md) |
| Codebase cleanup | Cleanup program scope, deferrals, and what phase 1 deleted or left open | [decisions/codebase-cleanup.md](decisions/codebase-cleanup.md) |
| Harness compatibility | Harness-specific launch history and superseded platform work | [decisions/launch-harness-compatibility.md](decisions/launch-harness-compatibility.md) |
| Startup / health / sandbox | Startup descriptors, doctor, bootstrap and projection policy | [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) |
| Package management | Mars compilation, targeting, ownership, hooks, recovery, engine constraints | [decisions/package-management.md](decisions/package-management.md) |
| Model resolution | Alias routing, candidate policy, prompting ownership | [decisions/model-resolution.md](decisions/model-resolution.md) |
| Workspace | Schema, path resolution, permission projection and migration | [decisions/workspace.md](decisions/workspace.md) |
| Tool permissions | How profile `tools`/`sandbox`/`approval` reach each harness: Mars-owned names, native enforcement, the Codex shell hook, `approval: auto`, reported gaps, allowlists, list form deprecated, headless `ask`, platform rules, permission sets that add across surfaces | [decisions/tool-permissions.md](decisions/tool-permissions.md) |
| Bundle field consumption | Every launch-bundle field applied, reported or deliberately unread; `harness-overrides` on Codex, compaction projection, `tools.mcp` as permission names | [decisions/bundle-field-consumption.md](decisions/bundle-field-consumption.md) |
| Sandbox platform access | What Meridian grants a sandboxed launch: exact project lifetime lock, Meridian commands, Claude and Codex child-spawn escapes, PID-namespace compound-launch refusal, the network field, and the sandbox/spawn/projected-network child guardrail | [decisions/sandbox-platform-access.md](decisions/sandbox-platform-access.md) |
| Telemetry | Envelopes, sinks, segments, querying and retention | [decisions/telemetry.md](decisions/telemetry.md) |
| Testing | Behavior-owned tiers and evidence gates | [decisions/testing.md](decisions/testing.md) |
| Primary inject | How `spawn message` reaches every live session: one control socket, Claude's inbox, the sender frame and address, posted or delivered, no resend after a write | [decisions/primary-inject.md](decisions/primary-inject.md) |
| Harness connection seams | Surviving seams: connection ownership, HITL handler protocol, passthrough boundary, spawn depth cap | [decisions/connection-seams.md](decisions/connection-seams.md) |
