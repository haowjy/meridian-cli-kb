# Session Initiation Modes

Every Meridian session — spawn or primary — is initiated in one of four modes.
The mode controls whether transcript lineage is inherited from a prior session,
whether the agent identity can change, and how prior context is delivered to the
new session.

Understanding these modes is foundational for writing spawn orchestration and
for understanding why certain combinations of flags are rejected.

---

## The Four Modes

| Mode | Transcript lineage | Identity | Prior context delivery |
|------|-------------------|----------|----------------------|
| `--continue [REF]` | Resume same session in-place | Same (unchanged) | Not applicable — no new turn |
| `--fork [REF]` | Fork prior transcript | Locked (inherited from source) | None — transcript already carries it |
| `--fork-fresh [REF]` | Fork prior transcript | May override (`-a`/`-m`/`--skills`) | None — transcript already carries it |
| `--from [REF]` | None — fresh session | Fully user-controlled | Rendered into user-turn context blocks |

### `--continue [REF]`

Resume the same harness session in-place. The harness picks up exactly where it
left off — same identity, same transcript, same state. No new Meridian-rendered
content. The next turn comes from interactive input or the next spawn task.

Continuation also preserves the recorded Meridian launch contract. For tracked
sessions and spawns, both primary and spawn continue build a launch-owned
`ContinueReplayContract` from the source reference plus its persisted
`LaunchPolicySnapshot`. The contract reuses the source work item, task directory
(`MERIDIAN_TASK_DIR` / task cwd), harness identity, model, agent, skills,
execution policy, env, passthrough args, and snapshot metadata. If the source had
no work item or task directory, that absence is also part of the contract:
continue must not attach the caller's ambient work item or inherited task cwd. It
must not silently recompute system-prompt-shaping or prompt-cache-shaping inputs
from the caller's current CWD, config, or environment.

If the recorded launch-policy snapshot has `model=""`, that empty model is legacy
persisted JSON for the same-session contract. Policy snapshot replay normalizes it
to in-memory `model=None`: Meridian should pass no managed model override and let
the recorded harness use its default, not resolve a replacement from current config
or environment. See [model-resolution: model optional](../decisions/model-resolution.md#model-optional-empty-model).

Changing task location, work attachment, identity, or launch policy is a
divergence, not continuation. Use `--fork`, `--fork-fresh`, `--from`, or a fresh
session for that. Same-session continue rejects overrides such as `--work`,
`--task-dir`, `--model`, `--agent`, `--skills`, execution-policy flags, env
overrides, and passthrough args. Agent opt-out (`--agent ''`) is also a launch
identity mutation: when the source opted out, continue preserves that opt-out and
must not reintroduce a configured default agent; when it did not, continue cannot
opt out during replay.

Use when: resuming interrupted work on the same session.

### `--fork [REF]`

Branch the harness session. The forked session inherits the full prior transcript
(the harness loads it; Meridian does not compose or modify it). Agent, model, and
skills are locked to the source — `-a`, `-m`, and `--skills` are rejected.

Task-scoped overrides are allowed: `--goal`, `-f`, `--prompt-var`, `--work`,
`-p`/`--prompt-file`.

Use when: exploring a different task direction on an existing conversation, or
handing off a task continuation to the same agent type.

### `--fork-fresh [REF]`

Branch the harness session with identity changes allowed. `-a`, `-m`, and
`--skills` may override source values. When none are specified, behavior is
identical to `--fork`.

Use when: handing off mid-conversation to a different role (e.g., coder → reviewer)
while the model should still see the full prior conversation.

**Cache implication:** When identity overrides are present, the system prompt
changes, so the harness prompt cache cannot warm from the forked prefix. The
transcript lineage is still forked (the model sees the prior conversation), but
the cache prefix is invalidated.

### `--from [REF]`

Start a completely new session. No harness transcript lineage. Prior context from
the referenced spawn or session is rendered into the user turn as structured
reference material — evidence the agent evaluates, not instructions it obeys.

Use when: seeding a fresh session with findings or outputs from a prior one, without
conversation continuity.

---

## Decision Tree

```mermaid
graph TD
    Q1{"Model should see\nthe prior transcript?"}
    Q1 -->|Yes| Q2{"Change agent/\nmodel/skills?"}
    Q1 -->|"No — just reference material"| FROM["--from [REF]"]
    Q2 -->|"No — same identity"| FORK["--fork [REF]"]
    Q2 -->|"Yes — new role"| FRESH["--fork-fresh [REF]"]
```

---

## Four-Layer Content Composition

Every session launch assembles content across four layers, ordered from most
stable (best prompt cache hit) to most volatile:

```
Layer 1: SystemInstruction
  Agent identity: profile body, skills, runtime instructions,
  context dirs, passthrough system fragments.
  Completion goal / report contract (spawn runs only).
  Stable across launches with the same agent/model/skills.

Layer 2: Harness transcript lineage
  The harness's own conversation history, loaded via session
  resume or fork. Invisible to Meridian — the harness loads it
  from its own storage.
  Present in: --continue, --fork, --fork-fresh.
  Absent in: --from, fresh sessions.

Layer 3: UserTurn.context_blocks
  Meridian-rendered user-turn material: -f file refs, --from
  prior-context blocks (wrapped in <prior-spawn-context> tags).
  Ordering: -f refs first, then --from blocks.
  Channel: user turn (never system prompt).

Layer 4: UserTurn.current_request
  The prompt text: -p / --prompt-file / interactive first turn.
  Always the final user-turn item.
```

### Why Layer 3 is user-turn, not system prompt

Four reasons, each independently sufficient:

1. **Prompt injection defense.** Prior spawn reports may contain user-generated
   content or tool outputs. System prompt injection grants implicit authority
   status. User-turn placement means the model evaluates the content as evidence,
   not as instructions to obey.

2. **Prompt cache locality.** The system prompt fingerprint determines the prompt
   cache key. Variable prior-context in the system prompt destroys cross-session
   cache sharing for sessions that have the same agent identity. User-turn
   injection preserves the stable system-prompt prefix.

3. **Harness consistency.** `--append-system-prompt` is reserved for agent
   identity material (skills, profile, inventory, report instructions). The
   user-turn channel works consistently across Claude, Codex, and OpenCode;
   the system-prompt channel does not.

4. **Semantic consistency with `-f`.** File references (`-f`) go in user-turn
   context blocks. `--from` prior context is structurally the same kind of
   reference material. Same channel, same semantics.

The completion goal (`--goal`) and report contract are the exception — they are
authoritative stopping conditions set by Meridian (not prior agent output) and
belong in Layer 1 (SystemInstruction).

---

## Identity Lock: `--fork` vs `--fork-fresh`

`--fork` rejects `-a`, `-m`, and `--skills` with the error:
> `--fork preserves launch identity. Use --fork-fresh to change agent, model, or skills.`

**Why the split exists:**
Agent, model, and skills determine the system prompt fingerprint — the dominant
factor in prompt cache locality. `--fork` guarantees that the identity-shaping
inputs are unchanged, so the harness cache can warm from the shared prefix.
`--fork-fresh` allows identity changes but cannot guarantee cache warmth.

**What `--fork` does allow (task-scoped overrides):**
`--goal`, `-f`, `--prompt-var`, `--work`, `-p`/`--prompt-file`. These change
task-scoped content within a preserved identity — the profile name, model, and
skill set remain the same even when `--prompt-var` substitutes into the profile
body.

**Why enforcement is CLI-only, not ops-layer:**
`SpawnForkInput` already handles both identity-preserving and identity-changing
forks. The ops layer doesn't need to enforce which CLI mode was chosen. Non-CLI
callers (MCP, programmatic) may have legitimate reasons to fork with identity
changes.

---

## Bare Flag Inference

Three optional-ref flags default to `$MERIDIAN_SPAWN_ID` when no REF is given.
Bare `--continue` has different semantics: it opens the session browse picker.

| Flag | Bare form | Resolved to |
|------|-----------|-------------|
| `--fork` | `--fork` (no REF) | `$MERIDIAN_SPAWN_ID` |
| `--fork-fresh` | `--fork-fresh` (no REF) | `$MERIDIAN_SPAWN_ID` |
| `--from` | `--from` (no REF) | `$MERIDIAN_SPAWN_ID` |
| `--continue` | `--continue` (no REF) | argv rewritten to `session browse` (see below) |

If `MERIDIAN_SPAWN_ID` is not set for `--fork`/`--fork-fresh`/`--from`, fails with:
> `Cannot infer {flag} target: not inside a Meridian-managed session. Pass {flag} REF explicitly.`

`$MERIDIAN_CHAT_ID` is available as an explicit ref when session-level context is
wanted instead of spawn-level context.

### Bare `--continue` as session browse

Bare `meridian --continue` (no ref) opens the interactive session browse picker
rather than failing. The canonicalization function `canonicalize_argv` rewrites
`["--continue"]` to `["session", "browse", ...]` (preserving `-C`/`--config`
flags) before any classifier call. `--continue <ref>` is untouched and resolves
to `PRIMARY_LAUNCH` as before.

Both entry forms produce the same invocation: bare `meridian --continue` *is*
`meridian session browse` — same `READ_RUNTIME` bootstrap (no telemetry, no
auto-init), same handler, same non-TTY table, same exit 0. There is no origin
flag and no provenance-dependent behavior.

This replaced an earlier design where bare `--continue` was excluded from bare
flag inference because "resume this session" without a ref is a no-op. The
browse picker is the useful resolution: the user wants to continue *something*
but doesn't know the ref.

### Argv normalization: how bare flags work with Cyclopts

Cyclopts requires a value token for `str | None` parameters. Bare `--fork`
(without a ref) would cause a parse error.

**Solution:** Pre-Cyclopts argv canonicalization (`canonicalize_argv()`
in `cli/argv_normalization.py`), which runs before bootstrap parsing and Cyclopts
dispatch. This function combines two rewrites: the existing optional-value-flag
sentinel insertion and the bare-continue-to-browse rewrite. It is idempotent and
runs once at the top of each entry point.

Sentinel insertion for `--fork`/`--fork-fresh`/`--from`:

- `--fork` → `--fork __SELF__`
- `--fork=` → `--fork __SELF__`
- `--fork=p123` → `--fork p123`
- `--fork-fresh -a x` → `--fork-fresh __SELF__ -a x`

Bare-continue rewrite: `--continue` with no value token (last, followed by
another flag, or `--continue=`) and no positional command tokens rewrites argv
to `["session", "browse", *remaining_flags]`.

**Sentinel:** `SELF_FORK_REF_SENTINEL = "__SELF__"`. Not a valid spawn ref (`pN`),
chat ref (`cN`), or UUID. Bootstrap uses `SYNTHETIC_VALUE_TOKENS` to detect and
skip synthetic values. `resolve_optional_ref(raw_ref, flag_name)` maps sentinel
to `MERIDIAN_SPAWN_ID` or raises with the flag-specific error message.

**Applies to both surfaces:** spawn (`meridian spawn --fork`) and primary
(`meridian --fork`). The normalization runs in the CLI entry point before any
parsing.

---

## Mutual Exclusion Matrix

| Combination | Allowed? |
|-------------|----------|
| `--fork` + `--fork-fresh` | No — two fork modes |
| `--fork` + `--continue` | No — fork branches; continue resumes |
| `--fork-fresh` + `--continue` | No |
| `--from` + `--continue` | No — injecting context into a resumed session is incoherent |
| `--from` + `--fork` | No (MVP) — see below |
| `--from` + `--fork-fresh` | No (MVP) — see below |
| `--from` + `-a`/`-m`/`--skills` | **Yes** — `--from` imposes no identity constraints |
| `--from` + `-f` | **Yes** — additive; file refs and prior-context blocks are complementary |

### Why `--from` + fork is rejected

`--fork` already carries the full prior transcript (Layer 2). Adding `--from` on
top means the model sees the same context twice — once as lived transcript, once
as rendered reference. This is redundant and creates ambiguous ordering between
the transcript (harness-managed, Meridian-invisible) and the `--from` blocks
(Meridian-rendered, user-turn).

If you need a forked conversation plus external context from a third spawn, pass
it via `-f`. File references compose cleanly with fork without semantic overlap
with transcript lineage.

---

## Surface Coverage

All four modes apply to both surfaces:

| Surface | `--continue` | `--fork` | `--fork-fresh` | `--from` |
|---------|-------------|----------|----------------|----------|
| `meridian spawn` | Yes | Yes | Yes | Yes (repeatable: multiple `--from` refs allowed) |
| `meridian` (primary) | Yes | Yes | Yes | Yes (single ref) |

---

## Implementation Seam

The shared user-turn context resolution seam lives in `launch/context.py`:

```python
resolve_task_context_inputs(
    context_from: tuple[str, ...],
    reference_files: tuple[str, ...],
    project_root: Path,
) -> TaskContextInputs
```

Both `_resolve_spawn_prepare_projection()` and `_resolve_primary_projection()` call
it. These two functions are **not** unified — they keep their respective surface
differences. Only the user-turn context resolution (file refs + `--from`
prior output + sanitization) is shared.

`sanitize_prior_output()` provides defense-in-depth by escaping content that could
look like Meridian structural markers, but user-turn placement is the primary trust
boundary.

---

## Session Re-entry Model

When a user selects a session to activate (through `session browse` or
programmatically), the system must decide which initiation mode to use. The
re-entry model answers this with a three-variant decision:

```
SessionReentryDecision = Resume(chat_id) | Fork(chat_id) | Blocked(reason)
```

- **Resume:** the session is stopped (no active lease) and has a recorded
  harness session — exec `meridian --continue <c-id>`.
- **Fork:** the session is live (active lease, owner process alive) — exec
  `meridian --fork <c-id>`. Fork allocates a new c-id, so the live session
  is never double-attached by construction.
- **Blocked:** the session has no recorded harness session id and cannot be
  resumed or forked — the picker shows an inline message and stays open.

The decision is ops-owned (`lib/ops/session_reentry.py`) with a pure core
(`decide_reentry`) and a fresh-read resolver (`resolve_session_reentry`).

### Advisory vs authoritative resolution

The listing rows carry an **advisory** re-entry decision, computed at listing
time from data the listing already read. This drives the UI: the footer verb
(`[enter] resume` vs `[enter] fork -> new session (live)`), the row's live
marker, and the blocked hint all render from the advisory decision.

The **authoritative** decision is resolved fresh at Enter time from a re-read
of lease liveness and harness session presence. The two can diverge in the
window between listing and Enter (a session may go live or stop), but the
authoritative decision governs the action. The worst case of advisory/
authoritative divergence is a silent wait (resume on a session that just went
live — the per-chat lifetime lock in the exec'd process blocks until the live
owner exits) or a fork of a just-stopped session (legal and safe — a branch
where a resume was possible; the source remains resumable).

### Fork-on-live: never double-attach by construction

The design deliberately forks live sessions rather than blocking or warning.
Fork allocates a new c-id (`start_session` with `chat_id=None`), so it never
touches `sessions/<source>.lock`. The user gets a branched copy of the
transcript at the current point. The pre-keypress footer verb and row marker
make the fork-not-resume behavior visible before the Enter keypress lands.

The per-chat lifetime lock taken by `start_session` in the exec'd process is
the real guard against double-attachment. The re-entry model is advisory for
verb display and authoritative for action selection, but the lock is the
safety invariant.

### Per-harness fork safety

How the transcript is branched at fork time differs by harness:

| Harness | Fork mechanism | Live-source safety |
|---|---|---|
| Claude | Delegated: `claude --resume <id> --fork-session` | Harness-owned snapshot semantics |
| OpenCode | Delegated: `opencode --session <id> --fork` | Harness-owned (DB transaction) |
| Pi | Delegated: `pi --fork <id>` | Harness-owned |
| Codex | Meridian-materialized: copies rollout JSONL + inserts into Codex's SQLite | Requires `materialize_fork_rollout` fix for live sources (see [lessons](../lessons/harness-integration.md)) |

Delegated harnesses own their concurrent-write semantics. Codex is the only
harness where Meridian copies the transcript file, and the current copy has a
known corruption bug on live sources that must be fixed before fork-on-live
ships.

---

## Related Pages

- [composition-pipeline.md](composition-pipeline.md) — how `ComposedLaunchContent` and `ProjectedContent` work; where the user-turn channel is materialized per harness
- [../decisions/launch.md#d-continue-replays-recorded-launch-contract-same-session-continue-is-not-live-policy-recomputation](../decisions/launch.md#d-continue-replays-recorded-launch-contract-same-session-continue-is-not-live-policy-recomputation) — why continue replays recorded work/task-dir and cache-shaping launch policy
- [../decisions/launch.md](../decisions/launch.md) — identity-lock and argv-normalization decision rationale
- [../architecture/launch-system.md](../architecture/launch-system.md) — `build_launch_context()` as the sole composition seam; `resolve_task_context_inputs` placement
- [../architecture/claude-session-isolation.md](../architecture/claude-session-isolation.md) — how `--continue` and `--fork` work at the Claude harness level
- [../decisions/session-reference-resolution.md](../decisions/session-reference-resolution.md) — how spawn/chat/session IDs are resolved for `--from`/`--fork`/`--continue`
