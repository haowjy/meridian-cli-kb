# Spawn Output Contract

When a foreground spawn completes or `spawn wait` resolves a single spawn, the
agent's **report is the primary result**. Metadata (tokens, cost, timestamps,
model, paths) is secondary — accessible on demand, not the default view.

This page documents the output contract: what each mode shows, how flags
control the detail level, and why these choices were made.

---

## Default: Report-First Compact Text

For both foreground `spawn` and single-spawn `spawn wait`, the default text
output is compact:

```
p636 succeeded (4.1s)

<report body>

Transcript: meridian session log p636
```

For failed spawns:

```
p636 failed (4.1s)

Error: <error text>

<report body if available>

Transcript: meridian session log p636
```

If no report file was written (harness crash before finalization), the body
shows `(no report)` for succeeded spawns, or the error text for failed ones.

### What the Transcript Pointer Is

`Transcript: meridian session log <spawn_id>` is part of **default output**,
not metadata. When an extracted report is incomplete or truncated, this line
is the user's (or agent's) escape hatch to the full source conversation.
Hiding it with verbose flags would make the CLI hostile when the report is
imperfect.

---

## Progressive Disclosure Hierarchy

Three levels of detail, from compact to comprehensive:

| Level | Invocation | Shows |
|---|---|---|
| **Compact** (default) | `meridian spawn ...` | Status line, report body, transcript pointer |
| **Metadata** | `--metadata` | Everything in compact, plus model/harness, exit code, cost/tokens, duration, report path |
| **Inspection** | `meridian spawn show <id>` | Curated per-command projection (JSON); full record (text) |

`--metadata` adds detail to the compact view; it does not replace or hide the
primary result. The report body and transcript pointer still appear.

`spawn show <id>` is the canonical inspection command. Its text output shows
all fields; its JSON output is a curated sparse projection via `to_cli_wire()`
(see [Sparse JSON Projections](#sparse-json-projections) below).

---

## `--metadata` vs `--verbose`

These are separate concerns:

- **`--verbose`** — runtime/debug verbosity. Does not affect the
  report/metadata toggle.
- **`--metadata`** — inline detailed spawn record. Shows the fields that
  previously cluttered default output: model/harness (where available), exit
  code, duration, cost/tokens, report path, transcript command, and other
  inspection fields.

`--metadata` still includes the report body and transcript command — it adds
accounting context, it does not demote the primary result.

**Rejected alternative:** Using `--verbose` as the metadata/accounting flag.
Verbose is debug verbosity; metadata is structured accounting. Conflating them
would mean "verbose" has two unrelated meanings depending on context.

---

## Output Mode Behavior

| Invocation | Output |
|---|---|
| `meridian spawn ...` (human or agent, no flags) | Compact text: status + report + transcript pointer |
| `meridian spawn ... --format text` | Same as default |
| `meridian spawn ... --format json` | Structured equivalent: `spawn_id`, `status`, `duration_secs`, `exit_code`, `report_body`, `transcript` field |
| `meridian spawn ... --metadata` | Compact + inline spawn record (model, cost, tokens, paths) |
| `meridian spawn ... --verbose` | Debug-level detail; report body and transcript still included |
| `meridian spawn ... --no-full` | Status line only — no report body, no transcript pointer |

### Agent Mode: Same Default as Human Mode

Agent mode (`MERIDIAN_DEPTH > 0`, no explicit `--format`) uses compact text
output, matching human mode. Previously, agent mode defaulted to JSON via the
`default_output_mode` in `CommandDescriptor`.

**Why:** Agents consuming spawn results need the report content, not a JSON
metadata object. Agents that need structured data can request it explicitly
with `--format json`.

**Change:** `default_output_mode` for `("spawn",)` and `("spawn", "wait")`
changed from `"json"` to `"text"` in `startup/catalog.py`.

### Explicit JSON Content

When `--format json` is used, `to_cli_wire()` on the output model produces a
curated sparse projection. See [Sparse JSON Projections](#sparse-json-projections)
for the full contract.

---

## Multi-Spawn and Background Behavior

| Surface | Behavior |
|---|---|
| Multi-spawn `spawn wait` | Table + report sections (existing behavior) |
| Background spawn submission (`--bg`) | Spawn ID + MUST-run obligation notice |
| Multi-ID `spawn show p1 p2` | JSON array of per-spawn sparse projections |

---

## Sparse JSON Projections

The codebase uses curated per-command default projections via
`CLIOutputProtocol` / `to_cli_wire()`. Each output model defines a
`to_cli_wire()` method that returns the JSON-default shape — a sparse
contract carrying the fields a coordinating agent actually needs. Heavy fields
stay reachable via explicit flags. No `--fields` flag or query language exists;
the noisy *default* was the bug, and curated projections are the fix.

This matches the same pattern used by gh, kubectl, and docker: a stable
default shape with selection layered on top when needed.

### `spawn show`

The JSON default is a sparse projection: spawn identity, status, timing,
model, agent, description, `report_path`, and a bounded 500-char
`report_summary`. Two fields are deliberately excluded:

- **`report_body`**: opt-in via `--full`. Text mode keeps
  report-by-default UX; JSON default carries only the bounded summary.
  `--full` is tri-state: omitted (format-dependent default), `--full`
  (include), `--no-full` (exclude). (Renamed from `--report`/`--no-report`
  in PR #469 to align with `session log --full`.)
- **`harness_session_id`**: dropped from the default JSON contract entirely.
  It is an internal lookup key; `session log <chat-id>` covers the use case.
  This is a deliberate, human-approved divergence from the original issue
  text (#113), which asked to keep it.

Multi-ID invocations (`spawn show p1 p2`) emit a JSON array where each
element is the same sparse projection. Previously the list path bypassed
projection and dumped full models.

`to_agent_wire()` on `SpawnActionOutput` is a separate projection for
managed-session ergonomics (implicit agent-mode JSON for background spawns).
It solves a different problem and is not unified with `to_cli_wire()`.

### `spawn wait`

Uses the same JSON report gating as `spawn show`. Agent-mode `spawn wait`
defaults to text and retains the full report — the sparse JSON path does not
affect agent workflow.

### `work show`

Sparse `to_cli_wire()`: work identity, status, timing, directories, plus
summary-only nested `spawns` (id, status, model, desc) and `sessions`
(chat_id, harness, status, model, agent). Worktree internals and
`harness_session_id` excluded. Deliberately no verbose/full-model escape
hatch — no consumer of the full shape exists, and a global `--json=full` can
be added later if a real need appears.

### `hooks run`

Outcome projection: hook, event, outcome, success, skipped, skip_reason,
error, exit_code, duration_ms. Raw `stdout`/`stderr` gated behind
`include_output` on `HookRunInput` + `--verbose` CLI flag (mirrors how
`include_report_body` flows through `SpawnWaitInput`).

### Contract Tests

Real-CLI JSON tests for the six projected commands (including multi-ID
arrays) assert required keys present and noisy keys recursively absent.
These also pin the four previously projected commands, which nothing
protected before.

---

## Implementation Notes

The compact text path is implemented in two places:

- **`SpawnActionOutput.format_text()`** — called for foreground spawn results.
  A new `_format_terminal_text()` branch handles `succeeded`/`failed`/`cancelled`
  statuses when the spawn is not background.
- **`SpawnDetailOutput.format_text()`** — called for single-spawn `spawn wait`
  results. A new `_format_compact_text()` branch handles `ctx.verbosity == 0`.
  The existing full `kv_block` output is extracted to `_format_verbose_text()`
  for `--verbose` mode.

Report body is populated in `execute_spawn_blocking()` via `read_report()` after
completion. The `--full` flag default in `_spawn_wait()` flipped from `False`
to `True`.

---

## Related Pages

- [Spawn Lifecycle](spawn-lifecycle.md) — status machine, report.md artifact,
  finalization
- [Spawn Wait Barrier](spawn-wait-barrier.md) — multi-spawn wait semantics,
  yield mechanics, no-arg discovery
- [architecture/spawn-finalization.md](../architecture/spawn-finalization.md) —
  how and when report.md is written
- [codebase/session-operations.md](../codebase/session-operations.md) — `meridian
  session log` and transcript extraction
