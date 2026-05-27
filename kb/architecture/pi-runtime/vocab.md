# Pi Runtime Vocabulary

> **Status:** Canonical. Pi-bg-redesign shipped. This page defines the current
> vocabulary for the pi-runtime background-work surface.

Domain vocabulary for the pi-runtime background-work redesign. This page defines the authoritative terms for the redesigned bash tool, background-task lifecycle, cross-extension coordination, and quiescence rule. This page stays current-only.

See also: [../pi-lifecycle.md](../pi-lifecycle.md) for the pi spawn lifecycle architecture; [../../concepts/harness-abstraction.md](../../concepts/harness-abstraction.md) for harness and extension concepts.

---

## Identifiers

| Term | Definition | See also |
|---|---|---|
| **`bash_id`** | The identifier the agent uses to reference a tracked bash record. Prefix `b-`. Returned by the `bash` tool when a command is backgrounded or a fg→bg timeout occurs; accepted by all `bash_manage` actions. Replaces the old tracked-job identifiers used before the redesign. Format: `b-<hex>` (e.g. `b-aXXX`). | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`b-*`** | ID prefix for tracked bash records. All bash record IDs begin with `b-`. Owned and assigned by the `managed-bash` extension. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`p-*`** | ID prefix for spawn records. Unchanged from existing Meridian convention. Owned by meridian-cli. | [../../concepts/spawn-lifecycle.md](../../concepts/spawn-lifecycle.md) |
| **`spawn_id`** | The identifier of a spawn record. Unchanged from existing Meridian convention. Used by `meridian spawn` subcommands, `/spawn`, and as the value of `parent_id` on child spawns. | [../../concepts/spawn-lifecycle.md](../../concepts/spawn-lifecycle.md) |

---

## Tools (Agent-Facing)

| Term | Definition | See also |
|---|---|---|
| **`bash`** | The unified bash tool. Replaces pi's built-in `bash`. Required parameter: `command: string`. Optional parameters: `timeout_min?: number` (1–59, default 55, foreground budget only) and `background?: boolean` (default `false`; if true, detach immediately). Returns a foreground success object `{stdout, stderr, exit_code, …}` on synchronous completion, or `{bash_id, status: "backgrounded" \| "started", …}` when backgrounded. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`bash_manage`** | Single discriminated-action ops tool covering all background-bash operations. Required parameter: `action: "list" \| "output" \| "kill" \| "wait" \| "detach"`. Optional parameters: `bash_id?` (required for `output`/`kill`/`wait`/`detach`), `include_completed?` (`list` only, default `false`), `timeout_min?` (`wait` only, default 10, max 59). | [../pi-lifecycle.md](../pi-lifecycle.md) |

---

## Extensions

| Term | Definition | See also |
|---|---|---|
| **`managed-bash`** | The mechanism extension. Owns: `bash` tool registration, `bash_manage` tool registration, the `b-*` bash registry, and env-var injection of `MERIDIAN_PI_BASH_ID` into child processes. Slash commands: `/ps` (with combined/stdout/stderr stream filters), `/ps:b` (alias `/ps:background`), `/ps:kill`, `/ps:logs`, `/ps:clear`. Writes `pi-bash/<spawn-id>/bash-records.json`. | [../../concepts/harness-abstraction.md](../../concepts/harness-abstraction.md) |
| **`meridian-spawn-watch`** | The policy extension. Owns: spawn-record disk watcher (`PiDiskWatcher`), env-var correlation filter, implicit-wait notification dispatch, and ping timer. Slash commands: `/spawn`, `/spawn:wait`, `/spawn:cancel`, `/spawn:show`, `/spawn:log`, `/spawn:clear`. Registers no tools. Successor to the earlier Pi lifecycle policy extension. **`/mspawn` was renamed to `/spawn` — no compatibility alias.** | [../../concepts/harness-abstraction.md](../../concepts/harness-abstraction.md) |

---

## Slash Commands

| Command | Owner | Description |
|---|---|---|
| **`/ps`** | `managed-bash` | List bash records (mechanism view). Shows only `b-*` records for the current session. Supports combined/stdout/stderr stream filters. |
| **`/ps:b <id>`** | `managed-bash` | Background a running foreground bash mid-flight. Alias: `/ps:background <id>`. |
| **`/ps:kill <id>`** | `managed-bash` | Terminate a tracked bash. |
| **`/ps:logs <id>`** | `managed-bash` | Show log tail for a bash record. Uses centered log overlay. |
| **`/ps:clear`** | `managed-bash` | Hide finished bash rows for the current Pi session. |
| **`/spawn`** | `meridian-spawn-watch` | List spawn records (policy view). Shows only `p-*` records correlated to this session via `originating_bash_id`. Renamed from `/mspawn` — no compatibility alias. |
| **`/spawn:wait <id>`** | `meridian-spawn-watch` | Block on a spawn completion. |
| **`/spawn:cancel <id>`** | `meridian-spawn-watch` | Cancel a spawn (proxies `meridian spawn cancel`). |
| **`/spawn:show <id>`** | `meridian-spawn-watch` | Full-screen task-panel view with lifecycle + report + log tail (`meridian session log <id> -n 20` default). |
| **`/spawn:log <id>`** | `meridian-spawn-watch` | Log dock overlay for just the session log. |
| **`/spawn:clear`** | `meridian-spawn-watch` | Hide finished spawn rows for the current Pi session. |

---

## Environment Variables

| Variable | Definition | See also |
|---|---|---|
| **`MERIDIAN_PI_BASH_ID`** | Injected by `managed-bash` into every child process's environment with the value of the launching bash's `bash_id`. Read by meridian-cli's spawn-store at spawn-record creation time and persisted to the spawn record as `originating_bash_id`. The cross-extension correlation bridge. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`MERIDIAN_PI_TASK_PING_INTERVAL_MIN`** | Cadence in minutes for the 1-shot bg-bash nag notification. Default `20`. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`MERIDIAN_PI_TASK_PING_TRIGGERS_TURN`** | `0`/`1` flag. Whether the ping notification uses `triggerTurn: true` (interrupting) or `false` (display-only). Default `0` (display-only). | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`MERIDIAN_PI_TASK_PING_INCLUDES_SPAWNS`** | `0`/`1` flag. Whether spawn rows (`p-*`) also receive pings. Default `0` (no — spawns have their own completion notification path). | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`MERIDIAN_PI_TASK_MAX_BG_LIFETIME_MIN`** | Optional hard cap on background bash lifetime in minutes. Default unset (no cap). Tasks exceeding the cap are killed and reported as `timed_out`. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`MERIDIAN_PI_MSPAWN_NOTIFY_ON_COMPLETION`** | `0`/`1` flag. Whether the `meridian-spawn-watch` policy extension fires implicit-wait completion notifications. Default `1`. Disable only for testing. | [../pi-lifecycle.md](../pi-lifecycle.md) |

---

## Spawn Record Fields (New)

| Field | Definition | See also |
|---|---|---|
| **`originating_bash_id?: string`** | Field on spawn records. Populated at creation time from `MERIDIAN_PI_BASH_ID` if set in the environment. Used by `meridian-spawn-watch` to filter `/spawn` rows to spawns originating from the current session's bash invocations. The detection signal is disk state + env, never argv parsing. | [../pi-lifecycle.md](../pi-lifecycle.md) |

---

## Concepts

| Term | Definition | See also |
|---|---|---|
| **Tracked bash** | A bash background record that blocks pi quiescence until it terminates. The default state for all `bash({background: true})` calls and fg→bg-timeout transitions. Opt out by calling `bash_manage({action: "detach", bash_id})`. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **Detached bash** | A bash background record that does NOT block pi quiescence. Created by an explicit `bash_manage({action: "detach", bash_id})` call. The process continues until natural exit; pi shutdown sends a cleanup signal on exit. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **Implicit-wait notification** | A `sendMessage` issued by `meridian-spawn-watch` when a tracked spawn or tracked bash background record terminates, regardless of whether the agent called an explicit wait. The safety net for "agent backgrounded work then forgot to wait." Wave-batched: multiple terminations close together produce one notification. Default `triggerTurn: true` for spawns and tracked bash; configurable. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **Env-var correlation** | The cross-extension bridge between `managed-bash` and `meridian-spawn-watch`. `managed-bash` injects `MERIDIAN_PI_BASH_ID` into commands; meridian-cli writes `originating_bash_id` into spawn records at creation time; `meridian-spawn-watch` reads that field to scope `/spawn` to this session's spawns. Detection signal is disk state + env, never argv parsing. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **Quiescence (revised rule)** | A pi spawn is considered finished when, AFTER the most recent `agent_end` event: (1) no spawn records with `parent_id == current_spawn_id` in non-terminal state, AND (2) no tracked bash background records for this session in non-terminal state, AND (3) no pending implicit-wait notifications (delivered but no follow-up `agent_end` yet). All three conditions must hold simultaneously. | [../pi-lifecycle.md](../pi-lifecycle.md) |

---

## State Vocabulary

Applies to both bash records and spawn records.

| State | Definition |
|---|---|
| **`running`** | Currently executing (foreground or background). Sub-flag `is_background: boolean` distinguishes the two sub-states; `is_background` becomes `true` after a fg→bg transition or for tasks started with `background: true`. |
| **`exited`** | Finished normally with an `exit_code` (bash records) or `succeeded` (spawn records). |
| **`failed`** | Finished abnormally. Spawn-specific; bash records use `exited` with a non-zero `exit_code`. |
| **`killed`** | Terminated externally via `bash_manage kill`, `/ps:kill`, `meridian spawn cancel`, or `/spawn:cancel`. |
| **`timed_out`** | Exceeded `MERIDIAN_PI_TASK_MAX_BG_LIFETIME_MIN` (if set). Task was killed and recorded as timed out. |

**Verb for fg→bg transition:** "backgrounded." Used in tool result messages and log output.

---

## Disk Artifacts

| Artifact | Definition | See also |
|---|---|---|
| **`pi-bash/<spawn-id>/bash-records.json`** | Per-spawn aggregate file for bash records. Written by `managed-bash`, watched by `PiDiskWatcher` / `PiQuiescenceTracker` to evaluate condition (2) of the revised quiescence rule. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`pi-bash/<spawn-id>/last-notification.json`** | Notification marker file. Written by `meridian-spawn-watch` when it calls `sendMessage({triggerTurn: true})`. Python quiescence checker requires `agent_end_ts > last_notification_ts` (condition 3) before declaring quiescence. | [../pi-lifecycle.md](../pi-lifecycle.md) |
