# Pi Runtime Vocabulary

> **Status:** Canonical current vocabulary for Pi runtime background work and
> extension coordination. Release status belongs in the CLI changelog.

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
| **`managed-bash`** | The mechanism extension. Owns: `bash` tool registration, `bash_manage` tool registration, the `b-*` bash registry, launch-scoped live process ownership, and injection of `_MERIDIAN_PI_BASH_ID` into child processes. Slash commands: `/ps` (with combined/stdout/stderr stream filters), `/ps:b` (alias `/ps:background`), `/ps:kill`, `/ps:logs`, `/ps:clear`. Writes `pi-bash/<spawn-id>/bash-records.json`. | [../../concepts/harness-abstraction.md](../../concepts/harness-abstraction.md) |
| **`meridian-spawn-watch`** | The policy extension. Owns: canonical direct-child observation, idle-only implicit-wait publication, exact native-admission receipts, delivery faults, and `/spawn*` UI. Slash commands: `/spawn`, `/spawn:wait`, `/spawn:cancel`, `/spawn:show`, `/spawn:log`, `/spawn:clear`. Registers no tools. **No `/mspawn` compatibility alias.** | [coordination.md](coordination.md) |

---

## Slash Commands

| Command | Owner | Description |
|---|---|---|
| **`/ps`** | `managed-bash` | List bash records (mechanism view). Shows only `b-*` records for the current session. Supports combined/stdout/stderr stream filters. |
| **`/ps:b <id>`** | `managed-bash` | Background a running foreground bash mid-flight. Alias: `/ps:background <id>`. |
| **`/ps:kill <id>`** | `managed-bash` | Terminate a tracked bash. |
| **`/ps:logs <id>`** | `managed-bash` | Show log tail for a bash record. Uses centered log overlay. |
| **`/ps:clear`** | `managed-bash` | Hide finished bash rows for the current Pi session. |
| **`/spawn`** | `meridian-spawn-watch` | List canonical direct-child `p-*` rows for the current parent. `originating_bash_id` may link a row to its Bash launcher but does not establish membership. Renamed from `/mspawn` — no compatibility alias. |
| **`/spawn:wait <id>`** | `meridian-spawn-watch` | Block on a spawn completion. |
| **`/spawn:cancel <id>`** | `meridian-spawn-watch` | Cancel a spawn (proxies `meridian spawn cancel`). |
| **`/spawn:show <id>`** | `meridian-spawn-watch` | Full-screen task-panel view with lifecycle + report + log tail (`meridian session log <id> -n 20` default). |
| **`/spawn:log <id>`** | `meridian-spawn-watch` | Log dock overlay for just the session log. |
| **`/spawn:clear`** | `meridian-spawn-watch` | Hide finished spawn rows for the current Pi session. |

---

## Environment Variables

| Variable | Definition | See also |
|---|---|---|
| **`_MERIDIAN_PI_BASH_ID`** | Injected by `managed-bash` into each child process with the launching Bash ID. Spawn creation may persist it as `originating_bash_id`; that link transfers a launcher's result obligation but does not define child membership. | [coordination.md](coordination.md) |
| **`_MERIDIAN_PI_TASK_PING_INTERVAL_MS`** | Cadence in milliseconds for tracked background-Bash ping notifications. Default is 55 minutes. Meridian resolves `timeouts.pi_task_ping_interval_seconds` / `MERIDIAN_PI_TASK_PING_INTERVAL_SECONDS` to this extension variable. Pings are advisory and cannot fail task execution. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`_MERIDIAN_PI_TASK_PING_RESET_ON_ACTIVITY`** | Whether tracked background-Bash pings reset on log activity. Default `true`; set to `false` to keep the original ping deadline. | [../pi-lifecycle.md](../pi-lifecycle.md) |

---

## Spawn Record Fields

| Field | Definition | See also |
|---|---|---|
| **`originating_bash_id?: string`** | Field on a spawn record copied from `_MERIDIAN_PI_BASH_ID` when present. It links the child to its launching Bash record and transfers that result obligation. Canonical `parent_id` rows, not this field, define direct-child membership. | [coordination.md](coordination.md) |

---

## Concepts

| Term | Definition | See also |
|---|---|---|
| **Tracked bash** | A bash background record that blocks pi quiescence until it terminates. The default state for all `bash({background: true})` calls and fg→bg-timeout transitions. Opt out by calling `bash_manage({action: "detach", bash_id})`. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **Detached bash** | A bash background record that does NOT block pi quiescence. Created by an explicit `bash_manage({action: "detach", bash_id})` call. Detach releases tracking but leaves the process owned by the live runtime; natural exit or normal Pi shutdown ends it. A cold runtime cannot reclaim or signal it. | [coordination.md](coordination.md) |
| **Implicit-wait notification** | Idle-only `sendMessage` follow-up from `meridian-spawn-watch` for eligible unattended child or tracked Bash results. Queueing is not delivery: native admission and Python's exact public-event observation are separate evidence. Terminal waits can consume results before publication. | [coordination.md](coordination.md) |
| **Bash origin link** | `managed-bash` injects `_MERIDIAN_PI_BASH_ID`; spawn creation may persist that value as `originating_bash_id` on a child row. It transfers a Bash launcher's result obligation to its canonical child; `parent_id`, not this field, defines child membership. No origin sidecar supplies authority. | [coordination.md](coordination.md) |
| **Quiescence** | A Pi spawn is finished only after its semantic turn and when there are no active transitive descendants, live tracked Bash tasks, owed terminal results, or unresolved delivery receipts. Exact notification admission must have its matching public-event observation and a later idle terminal turn; unknown evidence cannot become an empty set. | [../pi-lifecycle.md](../pi-lifecycle.md) |

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
| **`pi-bash/<spawn-id>/delivery-receipts.json`** | Exact native custom-message admission written from `message_start`; maps each delivery identity to its precise work membership. A `sendMessage()` return is not admission proof. | [coordination.md](coordination.md) |
| **`pi-bash/<spawn-id>/delivery-observations.json`** | Python's durable observation of the matching public RPC event, written after marking the parent active. Separate from the native receipt. A receipt without this observation is bounded unknown evidence. | [coordination.md](coordination.md) |
| **`pi-bash/<spawn-id>/delivery-fault.json`** | Bounded scan, admission, or persistence diagnostics separate from Bash execution failures. | [coordination.md](coordination.md) |
| **`pi-bash/<spawn-id>/last-notification.json`** | Retired timestamp marker, ignored by current completion policy. It cannot prove exact message membership, native admission, or public-event observation. Old unconsumed results may repeat after upgrade. | [coordination.md](coordination.md) |
| **`pi-bash/<spawn-id>/cleared-spawns.json`** | Cleared-spawns tracking file. Persists spawn IDs the user has cleared from the `/spawn` view so they stay hidden across Pi session restarts. | [../pi-lifecycle.md](../pi-lifecycle.md) |
| **`pi-bash/<spawn-id>/observed-spawns.json`** | CLI wait's durable exact observations, diagnostic waiting IDs, and process-owned wait reservations. Temporary reservations suppress notices only while caller PID, process-birth identity, and expiry match. | [coordination.md](coordination.md) |
