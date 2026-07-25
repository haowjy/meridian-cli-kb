# Decisions: Launch Waiting, Goals, and Session Initiation

These decisions govern how callers wait for delegated work, attach completion
goals, and start, continue, fork, or seed sessions.

## Decision Map

| Concern | Status | Concept |
|---|---|---|
| Wait barrier and yield | Current | [Spawn wait barrier](../concepts/spawn-wait-barrier.md) |
| Spawn goal composition | Current | [Composition pipeline](../concepts/composition-pipeline.md) |
| Continue/fork/fork-fresh/from | Current | [Session initiation](../concepts/session-initiation.md) |

## Spawn Wait Barrier

### `spawn wait` no-arg discovers pending spawns by MERIDIAN_CHAT_ID lineage

**Decision:** `meridian spawn wait` with no positional IDs collects all active spawns (`queued`, `running`, `finalizing`) whose `chat_id` matches `MERIDIAN_CHAT_ID`. The wait set is frozen at discovery time; spawns launched after discovery are not added.

**Why:** Orchestrators frequently forgot to run `spawn wait` or lost track of individual spawn IDs. No-arg wait removes the bookkeeping burden — the orchestrator launches all background spawns, then runs one barrier command with no arguments.

**Why chat lineage as scope:** All spawns in a tree share the same root `MERIDIAN_CHAT_ID` by inheritance at any depth. Chat lineage is the natural scope for "all the work I started."

**Alternatives rejected:** Work-item scope — considered but rejected because not all spawns are attached to a work item, and the user's stated preference was chat lineage as primary scope.

See [concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md).

---

### `--yield-after-secs` not `--timeout-secs`; checkpoint exits 0

**Decision:** The wait-yield interval flag is named `--yield-after-secs`. Yield (checkpoint expiry while spawns are still pending) exits with code 0 and prints continuation guidance. "Timeout" semantics (hard failure) are reserved for the existing `--timeout` flag. When `--timeout` is not explicit, `hard_deadline = None` and only the checkpoint governs.

**Why:** Naming the flag `--timeout-secs` would imply failure semantics, causing agents to treat a cache-preserving continuation as an error requiring retry logic. Keeping "timeout" reserved for failures makes the distinction unambiguous. Disabling the hard deadline in checkpoint mode prevents config-level `wait_timeout_minutes` from silently preempting checkpoint-mode waits.

---

### Spawn wait yield interval is harness-aware, not a global constant

**Decision:** The default wait-yield interval is harness-aware: all harnesses default to 3000 seconds (50 minutes), with a 30s minimum clamp. `--yield-after-secs` overrides per invocation.

**Why:** All supported harnesses (Claude Code with `ENABLE_PROMPT_CACHING_1H=1`, Codex extended cache, OpenCode) support cache TTLs of roughly 1 hour or more. A unified 50-minute default avoids unnecessary re-invocations for long-running sessions without requiring the user to tune the interval manually. Users who cannot extend harness cache TTL can lower the yield interval via `--yield-after-secs` or `MERIDIAN_WAIT_YIELD_AFTER_SECONDS`.

**Alternatives rejected:** A single configurable global — doesn't adapt to harness differences. TTL-minus-buffer modeling — too implementation-specific and brittle when TTLs change. Per-harness split defaults (240s/900s) — obsolete now that all harnesses support comparable extended cache TTLs.

> **Implementation updated (2026-05-05).** Values unified to 3000s for all harnesses. Previous split: unknown/default 240s, claude/codex 900s. Harness detection still uses `os.getenv("MERIDIAN_HARNESS")` (D33, 2026-04-30).

---

### Background spawn output uses MUST-run barrier language

**Decision:** The CLI output for a successful `--bg` spawn submission uses explicit obligation language ("you MUST run: `meridian spawn wait`") rather than weak suggestion language.

**Why:** Agents were reliably not running the barrier before reporting. Weak suggestion language was insufficient. Explicit MUST-run text on every background spawn submission is intended to drive habit formation in orchestrators. The no-arg wait command is shown as the primary form; a spawn-specific fallback is shown for non-session callers.

---

### D-headless-deny-at-prepare: Headless deny policy moved from bind to prepare (PR #460, 2026-07)

**Decision:** The `deny_headless_harnesses` policy check moved from
`bind_launch_context()` to `prepare_launch_surface()`. The bind-time check is
retained as defense in depth for persisted worker requests that may have
bypassed preparation.

**Why:** The deny check lived in `bind_launch_context()` -- the cheap
materialization phase. Background spawns call `prepare_launch_surface()` before
row reservation, then persist a worker request. The detached worker later calls
`bind_launch_context()` in a separate process. Between submission and worker
bind, the spawn row exists and `spawn wait` blocks on it. The denial surfaces
only seconds later as a generic `launch_failure` from the worker.

Moving the check to preparation rejects denied spawns synchronously at
submission, before any row, log directory, or worker artifacts are created. The
caller receives the actionable policy error immediately.

**Structural principle:** Policy decisions belong in preparation; bind purely
materializes. The prepare/bind split separates "should this spawn run?"
(preparation) from "build the process arguments" (binding). Deny-headless is a
policy decision, not a materialization step. This principle applies to any
future policy gate: if it can be evaluated before the spawn ID exists, it
belongs in `prepare_launch_surface()`.

**Guard scope:** The enforcement is guarded to `SPAWN_PREPARE` surface only --
primary launches are unaffected. The bind-time check remains because the
background worker binds from a persisted request that was not necessarily
prepared in the same process.

**Alternatives rejected:**
- Add an ops-level preflight check before `execute_spawn_background()` --
  duplicates the policy outside the composition seam; `prepare_launch_surface()`
  is the natural home because all required inputs (resolved harness, config
  snapshot) are already available there.
- Remove the bind-time check entirely -- unsafe because persisted worker
  requests skip preparation; defense in depth is cheap.

See [architecture/launch-system.md](../architecture/launch-system.md) for the
prepare/bind split.

---

## Spawn Goal (`--goal`)

### Spawn-level goal authority: not work-level, not prompt sugar

**Decision (2026-05, spawn-goal):** `meridian spawn --goal TEXT` stores goal as a first-class flat field on the spawn record, not as a work-level property and not as additional prompt body text.

**Why spawn-level:** Work-level goal would create inheritance ambiguity across the multiple spawns (reviewer, tester, etc.) that belong to one work item, each with a different purpose. A future work-level default can materialize into `SpawnCreateInput.goal` at create time without changing v1's spawn-level authority.

**Why flat field, not sidecar:** `state.json.goal` is the simplest structured field and is enough for future hooks. A `completion-contract.json` sidecar becomes valuable only when the contract gains subfields (success criteria, required artifacts, hook state). Adding it before those needs exist creates a second persistence authority prematurely.

**Why not prompt sugar:** Goal as prompt body text would require future hooks to parse prompt artifacts — violating files-as-authority. A structured field remains queryable without prompt parsing.

**Alternatives rejected:**
- Work-level goal — inheritance ambiguity; couples the feature to work-item semantics prematurely
- `completion-contract.json` sidecar in v1 — no subfields yet; two persistence authorities before hooks exist
- Prompt-body injection only — future hooks would need to parse prompt artifacts

---

### Goal does not affect policy; it only affects content composition

**Decision:** `compile_prepared_policy_surface()` (in `launch/policies.py`) is goal-agnostic. Goal does not affect model, harness, approval mode, or routing. It affects only the semantic IR (`completion_contract` block) inside content composition.

**Why:** Policy decisions (model selection, harness routing) happen before content assembly. Goal is a bounded completion contract for the agent, not a routing signal. Keeping policy goal-agnostic preserves the policy/mechanism separation and makes goal a pure composition concern.

---

### `completion_contract` placed after `context_prompt` in `SYSTEM_INSTRUCTION_BLOCK_ORDER`

**Decision:** The new `completion_contract` field on `ComposedLaunchContent` is positioned after `context_prompt` in `SYSTEM_INSTRUCTION_BLOCK_ORDER`:

```python
SYSTEM_INSTRUCTION_BLOCK_ORDER = (
    "supplemental_documents",
    "agent_profile_body",
    "report_instruction",
    "inventory_prompt",
    "context_prompt",
    "completion_contract",   # ← goal injected here
)
```

**Why:** The goal contract is behavior-shaping and should be prominent after the agent has been oriented via context/inventory. Placing it after `context_prompt` ensures the agent reads its context before seeing the completion obligation. `passthrough_system_fragments` (user's explicit `--append-system-prompt`) remains last to allow deliberate user overrides.

---

### Goal text normalization: trim-only; no `--prompt-var` substitution

**Decision:** Goal normalization trims only leading/trailing whitespace. Interior whitespace — spaces, newlines, list formatting — is preserved exactly. `--prompt-var` substitution is not applied to goal text.

**Why:** The same goal text must appear both in `state.json` (persisted) and in the injected completion contract. If template substitution ran after persistence, the persisted text would differ from the injected text, making the state.json record unreliable as a source of truth. For v1, keeping them identical is correct.

**Error contract:** Normalization rejects explicitly provided empty strings with `--goal cannot be empty`. `None` is the only absent-goal value accepted by the prompt renderer; an empty or non-normalized string reaching the renderer indicates an upstream bug and fails fast.

---

### Continue/fork goal inheritance: concrete spawn refs only

**Decision:** `meridian spawn --continue SOURCE` inherits the source spawn's goal when SOURCE is a tracked spawn row; explicit `--goal TEXT` overrides. `meridian spawn --fork p<ID>` inherits goal when the reference matches the concrete spawn-ref shape (`p` followed by digits); chat IDs, raw harness session IDs, and untracked refs do not inherit a goal.

**Implementation:** A `_looks_like_spawn_ref(ref: str)` type-narrowing helper gates the inheritance lookup:

```python
def _looks_like_spawn_ref(ref: str) -> bool:
    normalized = ref.strip()
    return len(normalized) > 1 and normalized[0] == "p" and normalized[1:].isdigit()
```

**Why no chat-ref inheritance:** A chat can contain multiple child spawns (reviewer, tester, coder) with different goals. There is no authoritative single goal owner for a chat ID. Requiring explicit `--goal` for chat/raw refs prevents invented goal inheritance from surprising the user.

**SpawnStartMetadata carrier:** The start call passes goal via a dedicated `SpawnStartMetadata` dataclass (not loosely via `**kwargs`) to keep the call site explicit and auditable. This is the R-05 refactor from the plan.

---

## Session Initiation (2026-05, PR #216)

Same-session continue replay was tightened later; see [D-continue-replays-recorded-launch-contract](#d-continue-replays-recorded-launch-contract-same-session-continue-is-not-live-policy-recomputation).

### D-fork-identity-lock: `--fork` locks identity; `--fork-fresh` allows changes

**Decision:** `--fork` rejects `-a`, `-m`, and `--skills` at the CLI. `--fork-fresh` is the distinct mode that permits identity overrides. Enforcement is CLI-only — the ops layer (`SpawnForkInput`) handles both modes without a separate field.

**Why:** Agent, model, and skills determine the system prompt fingerprint — the dominant factor in prompt cache locality. `--fork` guarantees identity-shaping inputs are unchanged, so the harness cache can warm from the shared prefix. Without the explicit split, callers would silently invalidate the cache by passing `-a` to `--fork`. The two modes capture two distinct user intents: "branch this conversation to explore a task variant" (`--fork`) vs "hand off to a different role" (`--fork-fresh`).

Task-scoped overrides (`--goal`, `-f`, `--prompt-var`, `--work`, `-p`) are allowed by `--fork`. These change task-scoped content within a preserved identity, not the identity itself. `--prompt-var` substitutes into the profile body — this is intentional customization, not an identity change.

**Why CLI-only:** `SpawnForkInput` already handles both fork behaviors. The ops layer does not need to enforce CLI ergonomics. Non-CLI callers (MCP, programmatic) may legitimately fork with identity changes.

**Alternatives rejected:**
- Reject `-a`/`-m`/`--skills` in the ops layer — couples an ergonomic CLI policy to a layer that has no semantic stake in it; breaks programmatic callers.
- Add a mode field to `SpawnForkInput` — adds coupling without correctness benefit; the CLI already validated before constructing the input.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) — full four-mode model.

---

### D-prior-context-user-turn: `--from` prior context goes in user turn, not system prompt

**Decision:** Prior spawn/session context delivered via `--from` is rendered into `UserTurn.context_blocks` (user turn), not into `SystemInstruction` (system prompt / `--append-system-prompt`).

**Why (four independent reasons):**

1. **Prompt injection defense.** Prior spawn reports may contain user-generated content or tool outputs. System prompt placement grants implicit authority status. User-turn placement means the model evaluates the content as evidence, not as instructions to obey. `sanitize_prior_output()` is defense-in-depth; channel placement is the primary trust boundary.
2. **Prompt cache locality.** The system prompt fingerprint determines the cache key. Variable prior-context material in the system prompt destroys cross-session cache sharing for sessions that share the same agent identity. User-turn injection preserves the stable system-prompt prefix.
3. **Harness consistency.** `--append-system-prompt` is reserved for agent identity material. Codex and OpenCode have no separate system-prompt channel; the user-turn channel is the only consistent cross-harness delivery path.
4. **Semantic consistency with `-f`.** File references (`-f`) go in user-turn context blocks. `--from` prior context is structurally the same kind of reference material. Same channel avoids a confusing distinction.

**Exception:** The completion goal (`--goal`) and report contract are authoritative stopping conditions set by Meridian — not prior agent output — and belong in `SystemInstruction`.

**Alternatives rejected:**
- System prompt injection for authority — grants untrusted prior output instruction status; invalidates the prompt cache.
- Harness-specific channels — inconsistent across Claude/Codex/OpenCode; user-turn is the only cross-harness consistent path.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) — full layer model and authority hierarchy.

---

### D-argv-normalization-sentinel: Pre-Cyclopts argv normalization for optional-value flags

**Decision:** `--fork`, `--fork-fresh`, and `--from` accept an optional positional REF. Because Cyclopts requires a value token for `str | None` parameters, bare invocations (e.g., `--fork` without a ref) would cause a parse error. The fix is pre-Cyclopts argv normalization: `normalize_optional_value_flags()` in `cli/argv_normalization.py` inserts a sentinel token (`__SELF__`) before Cyclopts sees the argv. Bootstrap uses `SYNTHETIC_VALUE_TOKENS` to detect and skip synthetic values. `resolve_optional_ref(raw_ref, flag_name)` maps the sentinel to `$MERIDIAN_SPAWN_ID` or raises with a flag-specific error.

**Why pre-Cyclopts:** Cyclopts does not support optional-value parameters natively. Post-Cyclopts transformation would require patching the parsed result after Cyclopts already rejected the bare form. Pre-normalization is transparent to Cyclopts — the sentinel looks like a normal value token.

**Sentinel design:** `__SELF__` is not a valid spawn ref (`pN`), chat ref (`cN`), or UUID. The sentinel is defined as a named constant (`SELF_FORK_REF_SENTINEL`), not a magic string, and checked explicitly in `resolve_optional_ref`.

**Bare ref semantics:** All three flags default to `$MERIDIAN_SPAWN_ID` — the current spawn context. `$MERIDIAN_CHAT_ID` is available as an explicit ref for session-level context. `--continue` was intentionally excluded: bare `--continue` means "resume this session" which is a no-op, and its semantics differ from the optional-ref pattern.

**Applies to both surfaces:** spawn and primary. The normalization runs in `main.py` before bootstrap and Cyclopts dispatch.

**Alternatives rejected:**
- Cyclopts optional-value parameter — not supported without custom type annotation plumbing.
- Post-parse sentinel injection — brittle; Cyclopts rejects before transformation can run.
- Require explicit ref always — adds ceremony for the common case; `$MERIDIAN_SPAWN_ID` is always set inside a Meridian-managed session.

---

### D-from-fork-mutual-exclusion: `--from` + fork rejected in MVP

**Decision:** Combining `--from` with `--fork` or `--fork-fresh` is rejected. If external context is needed on a forked conversation, pass it via `-f`.

**Why:** `--fork` already carries the full prior transcript as Layer 2 (harness-managed, Meridian-invisible). Adding `--from` on top is redundant — the model would see the same context twice (once as lived transcript, once as rendered reference) with ambiguous ordering between a harness-managed layer and a Meridian-rendered layer. `-f` composes cleanly with fork because file references are always Layer 3 content with no semantic overlap with transcript lineage.

**Future path if needed:** Allow `--from` + `--fork` with explicit ordering rule: `--from` blocks rendered into user turn after the forked transcript and before `-p` text. But the redundancy risk makes this opt-in, not default.

---

## Related

- [Launch composition decisions](launch.md)
- [Launch process ownership](launch-process-ownership.md)
