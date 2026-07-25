# Decisions: Launch Harness Compatibility History

This page preserves harness- and platform-specific launch decisions. Native
Windows work is historical and superseded by the current POSIX-first stance;
the Claude trampoline reconciliation decision remains current compatibility
behavior.

## Decision Map

| Concern | Status | Current authority |
|---|---|---|
| Native Windows launch work | Superseded / legacy best effort | Root `AGENTS.md` POSIX-first rule |
| 2026-05 structural simplification | Historical rationale | [Launch architecture](../architecture/launch-system.md) |
| Claude TUI trampoline ID repair | Current | Claude harness implementation |

## Windows Compatibility (2026-05, PR #200 / PR #184)

### D-win-signal: Portable signal handlers via `signal.signal()` + `call_soon_threadsafe()`

**Decision:** Signal handlers in async spawn runners use `signal.signal()` +
`loop.call_soon_threadsafe(event.set)` instead of `loop.add_signal_handler()`.

**Why:** `loop.add_signal_handler()` is not implemented on Windows
`ProactorEventLoop` — it silently no-ops or raises `NotImplementedError`. The
`signal.signal()` + `call_soon_threadsafe()` pattern is cross-platform and
produces identical behavior on POSIX. The cleanup callable pattern (store
previous handlers, restore on exit) is preserved.

**Constraint:** `signal.signal()` must be called from the main thread.
`_install_signal_handlers()` returns `None` when called from a worker thread
to signal "no cleanup needed." Callers handle `None` by skipping cleanup.

**Alternatives rejected:** Platform-specific branches (`if IS_WINDOWS: ...`) —
rejected because the portable pattern works on both platforms without branching.

---

### D-win-cancel: Claude cancel uses `terminate()` on Windows, `send_signal(SIGINT)` on POSIX

**Decision:** The Claude harness adapter's cancel path branches on platform:
POSIX sends `SIGINT` (graceful interrupt, allows shutdown handlers to run);
Windows calls `process.terminate()` (maps to `TerminateProcess`, immediate).

**Why:** `send_signal(SIGINT)` on Windows maps to `GenerateConsoleCtrlEvent`.
This only works for processes in the same console group. Harness subprocesses
launched with `CREATE_NEW_PROCESS_GROUP` — the isolation default — never
receive the event. The subprocess appears to ignore the cancel, continuing
to run until externally killed. On POSIX, `SIGINT` is reliable and allows
Claude to handle it gracefully (e.g., emit a final report before exiting).

**Why not `terminate()` on both platforms:** `terminate()` is `TerminateProcess`
on Windows but `SIGTERM` on POSIX. `SIGTERM` on POSIX gives Claude no
opportunity to run its SIGINT shutdown handler. The POSIX preference is SIGINT
for graceful shutdown; the Windows fallback is terminate because graceful
interrupt delivery is not available.

---

### D-win-output-log: `output_log_path` must be passed through; no hardcoded `None`

**Decision:** `WindowsConsoleLauncher` passes `output_log_path` from its
`bind_launch_context()` call through to the launch spec. Hardcoding `None`
was a one-line omission that caused spawn output to be discarded on Windows.

**Why:** `output_log_path` is how the streaming runner knows where to write
the session history artifact. A hardcoded `None` was a silent bug — the spawn
succeeded but no history was captured, and no error was raised.

---

### D-win-ci: Windows CI gate (superseded)

**Original decision:** Windows CI is a required check on all PRs that touch
platform behavior.

**Superseded (2026-07-17, PR #375):** The windows-gate CI job was demoted to
non-blocking (`continue-on-error: true`) as part of the POSIX-first platform
stance. The job remains as an informational signal but no longer gates merges.
Rationale: native Windows was never made to work and is not planned; maintaining
a blocking gate for untested, best-effort platform branches costs more than its
informational value. See the POSIX-first foundational decision in
[decisions.md](../decisions.md).

---

### D-win-stdout: `sys.stdout.reconfigure(encoding='utf-8')` at CLI entry on Windows

**Decision:** The CLI entry point calls `sys.stdout.reconfigure(encoding='utf-8')` on
Windows before any command runs. The call is Windows-only; POSIX is unchanged.

**Why:** On Windows, `sys.stdout` defaults to the active code page (often cp1252 or
cp850). Any Unicode output — agent names, file paths with non-ASCII characters, emoji
in status messages — raises `UnicodeEncodeError` or silently corrupts the output.
Reconfiguring stdout to UTF-8 at the entry point normalizes encoding for all downstream
code without requiring per-call `encode`/`errors='replace'` guards.

**Why not `PYTHONUTF8=1`:** The env var would require users to set it before running
Meridian. A startup call inside the CLI owns the fix unconditionally.

**Why Windows-only:** On POSIX, stdout is already UTF-8 (locale-determined in practice).
Reconfiguring on POSIX would change behavior for terminals that legitimately use
non-UTF-8 locales; the risk is asymmetric.

---

### D-win-spawn-wait-scope: No-arg `spawn wait` in nested spawns scoped to descendants only

**Decision:** When `MERIDIAN_SPAWN_ID` is set (the caller is a nested spawn,
not the primary session), `_resolve_wait_targets()` passes
`only_descendants_of=self_spawn_id` to `_discover_pending_spawns()`. The wait
set is restricted to the caller's subtree via BFS. Primary session behavior
(no `MERIDIAN_SPAWN_ID`) is unchanged — it still waits on all chat-scoped spawns.

**Why:** Without descendant scoping, no-arg `spawn wait` from a tech-lead or
orchestrator spawn would include the primary session in its wait set. The
primary session is perpetually active while spawns run — it never terminates
during the orchestrator's lifetime. This caused tech-lead and orchestrator
agents to hang indefinitely on `spawn wait`.

**Why BFS (not parent-only filter):** An orchestrator may spawn nested
orchestrators, which themselves spawn workers. A flat parent-only filter would
leave grandchild spawns in the active-but-invisible category — the orchestrator
waits for its direct children, but a direct child waiting for its own children
fails to drain because the grandparent's wait released too early. BFS through
`_collect_descendants()` captures the full subtree regardless of depth.

**Alternatives rejected:**
- Work-item scope filter — not all spawns have work items; primary session
  spawns are not work-item-scoped
- Explicit spawn ID tracking — the existing background-spawn output already
  gives explicit IDs; no-arg wait exists specifically to avoid tracking IDs
- Filter to direct children only — misses nested orchestration; still incomplete

See [../concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) —
descendant-scoping semantics and implementation detail.

---

---

## Structural Simplification (2026-05, PR #184)

### Wave 1: Indirection removal

**Decision:** Four proxy/wrapper indirections were deleted:
- `decision.py` proxy — callers now call target functions directly
- `prepare.py` wrapper pairs for the dry-run prepare path — inlined to direct calls
- `process/__init__.py` compat shim — removed
- `autocompact` validation — centralized

**Why:** Each removed indirection was a pure maintenance surface with no semantic value — it routed calls without adding logic, policy, or error handling. Extra layers obscured the actual call chain, made grep-based navigation harder, and created dead weight in reviews. Removing them makes the data flow explicit and auditable at a glance.

**Principle:** Indirections earn their keep by expressing a boundary, enabling testing, or isolating change. Proxy modules that just re-export the target do none of these.

---

### Wave 2: Session carrier consolidation

**Decision:** `session_scope()` and `lifecycle.start()` accept typed carriers (`SessionRequest` + `PrimarySessionMetadata`) instead of individual keyword arguments.

**Why:** `**kwargs` spreading at call sites made it impossible to see at a glance what was being passed, deferred type errors to runtime, and meant new fields silently vanished if a caller forgot to thread them. Typed carrier dataclasses make the call site explicit and auditable — what goes in is visible from the type, not inferred from reading the function body.

This is the same rationale as `SpawnStartMetadata` in the spawn-goal work: typed carriers over `**kwargs` is the consistent pattern for session and spawn boundary calls.

---

### Wave 3: LaunchSpec flattening

**Decision:**
- `build_child_env_overrides` wrapper deleted — call sites use the target directly
- `LaunchRequest` compat bridges removed
- Harness launch specs flattened into a single `ResolvedLaunchSpec`

**Why:** The compat bridges existed to ease migration from an older struct shape. Once the migration was complete, the bridges were pure ceremony — they copied fields without transforming them. Flattening into `ResolvedLaunchSpec` gives a single canonical struct that all downstream paths read, eliminating the question of which shape is authoritative.

## Claude TUI Trampoline Session-ID Reconciliation (2026-06)

**Decision:** Repair Claude TUI trampoline session IDs at finalization time via
file-based `history.jsonl` evidence rather than an interactive `claude
--resume` probe.

Claude's new TUI creates a transient trampoline session when entering `/tui
fullscreen`, then writes the durable transcript under a different session ID.
Meridian may record the trampoline ID during launch. The fix checks
`~/.claude/history.jsonl` at finalization: find the `/tui fullscreen` marker,
identify the successor same-project prompt, verify the successor transcript's
first user message matches, and update records to the durable ID.

**Why file-based over probe:** A `claude --resume <real-id> --print
--output-format json 'Reply: RESUME_PROBE_OK'` was tested and rejected. It
performs model work that hits the configured budget cap, making it
non-deterministic and inappropriate for finalization-time reconciliation.

**Why adapter override over generic finalization:** The trampoline pattern is
specific to Claude's new TUI and depends on Claude's `history.jsonl` format and
`projects/<slug>/` transcript layout. Embedding this in `ClaudeAdapter` keeps
trampoline-specific helpers alongside the existing transcript-path utilities
they depend on. Codex, OpenCode, and Pi do not have this pattern and need no
changes.

**Conservative fallback:** When evidence is insufficient (no trampoline marker,
no unique successor, transcript mismatch), the recorded ID is preserved
unchanged. This means existing behavior is preserved — the system fails or
succeeds on the recorded ID rather than guessing.

**Implementation:** `reconcile_tui_trampoline_session_id()` in
`src/meridian/lib/harness/claude.py`, wired into
`ClaudeAdapter.observe_session_id()`. Called by `runner.py` during primary
finalization. See [../architecture/claude-session-isolation.md#tui-trampoline-session-id-reconciliation](../architecture/claude-session-isolation.md#tui-trampoline-session-id-reconciliation).

## Related

- [Launch composition decisions](launch.md)
- [Harness abstraction](../concepts/harness-abstraction.md)
