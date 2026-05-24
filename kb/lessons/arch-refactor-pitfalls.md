# Lessons: Implementation Pitfalls from the Architecture Refactor (PR #184)

Pitfalls discovered during the `architecture-refactor-roadmap` work item (2026-05,
PR #184 — `wt/arch/p1a-seam`). Each surfaced during implementation or review and
cost real debugging time.

---

## `GIT_DIR` Leaks from Git Hooks into Subprocess Trees

**The problem:** A git hook (pre-push, post-commit, etc.) runs with `GIT_DIR` set
in its environment. Any subprocess the hook spawns inherits `GIT_DIR`. If that
subprocess is a meridian harness launch, the harness process — and everything
it subsequently forks — inherits `GIT_DIR` from the original hook invocation.
This breaks harness processes that invoke git inside the spawn tree: they think
they're inside the hook's repository at the hook's `GIT_DIR` path, not inside
whatever project the agent is actually working on.

**The symptom:** Harness agents running inside git hooks encounter confusing git
errors or operate on the wrong repository. The spawn may succeed but its git
operations silently target the wrong tree.

**The fix:** Any git hook that execs subprocesses must explicitly unset `GIT_DIR`
(and `GIT_WORK_TREE` for completeness) before exec'ing into meridian or any other
tool that might do git work.

```bash
# In a git hook before launching meridian:
unset GIT_DIR
unset GIT_WORK_TREE
exec meridian spawn ...
```

**Why this is hard to catch:** The hook environment is not visible in the subprocess's
logs or error output. The subprocess just sees a valid `GIT_DIR` and uses it
without knowing it inherited the value from a hook ancestor.

---

## structlog `cache_logger_on_first_use=True` Breaks `capture_logs()` in Tests

**The problem:** structlog's `cache_logger_on_first_use=True` processor (commonly
configured in production logging setup) caches the bound processor chain on
the first log call. Module-level loggers created with `structlog.get_logger()`
are bound at import time. If the module's logger fires once before a test's
`capture_logs()` context manager installs a test processor chain, subsequent
log calls within the test go through the cached production chain — bypassing
the test capture entirely.

**The symptom:** Tests that assert on log output using `structlog.testing.capture_logs()`
produce empty results even when the code under test clearly executes log
statements. The code runs, the logs fire, they just don't appear in the captured
list.

**The fix:** Two approaches:

1. **Rebind the logger in test setup:** Call `structlog.reset_defaults()` or
   rebuild the logging config before each test that needs to capture logs. This
   clears the cache and forces recomputation.

2. **Avoid module-level eager binding:** Create loggers lazily or reset them
   explicitly in the test fixture before the tested module fires its first log call.

**Rule of thumb:** If `capture_logs()` returns empty but you can see the code
running, check whether `cache_logger_on_first_use=True` is in the processor
chain and whether the logger was cached before the test ran.

---

## `CheckpointService` `git add -A` Commits the Entire Working Tree on Misconfiguration

**The problem:** `CheckpointService` (the git autosync hook) runs `git add -A`
to stage changes before committing. When `project_root` is misconfigured — for
example, pointing at the wrong directory or at a parent directory that contains
multiple unrelated repos — `git add -A` stages everything under that root.
The subsequent commit then includes files from unrelated projects or large
untracked artifacts that were never meant to be committed.

**The symptom:** Autosync commits contain unexpected files, sometimes from
entirely unrelated projects. The commit cannot easily be undone if it was pushed.

**Why this matters:** The checkpoint service runs automatically as a lifecycle
hook. Users may not notice the oversized commit until after it has been pushed.

**The fix / guard:**

1. Always validate `project_root` points to the intended git working tree before
   running `CheckpointService`.
2. Prefer explicit `git add <specific-paths>` patterns when staging checkpoint
   changes, or use `--pathspec-from-file` to limit staging scope.
3. In test environments, always use isolated temporary git repos as `project_root`
   to prevent inadvertent staging of the real repo during tests.

**Relationship to `NEVER delete untracked files`:** The same "shared workspace"
concern applies in the commit direction — staging with `-A` in a misconfigured
root is the commit-time analog of `git clean -f` on the wrong directory.

---

## Process-Group Kill Is Required for Spawn Cleanup

**The problem:** Sending `SIGTERM` to a worker PID (`os.kill(pid, SIGTERM)`) is
insufficient for spawn cleanup when the harness has spawned child processes that
have reparented to init (PID 1). Browser-backed harnesses (Playwright/Chrome) and
some harness backends fork child processes that detach from the process group.
A PID-only kill leaves these children running after the worker exits.

**The symptom:** Orphaned browser/backend processes accumulate after spawn
cleanup. `meridian doctor` detects orphaned spawn rows but the underlying
processes remain alive and consuming resources.

**The fix:** Use `os.killpg(os.getpgid(worker_pid), SIGTERM)` on POSIX to kill
the entire process group. The reaper now does this in
`state/reaper.py:_terminate_process_group()`:

```python
process_group_id = os.getpgid(worker_pid)
os.killpg(process_group_id, signal.SIGTERM)
```

**Windows fallback:** `os.killpg` is not available on Windows. The reaper falls
back to `os.kill(worker_pid, SIGTERM)` there. Windows process group semantics
differ; the equivalent cleanup may need `taskkill /T` for subtrees.

**Why this matters for managed primaries:** Managed Codex/OpenCode primaries have
backend and TUI processes that may reparent. The reaper's managed-primary
finalization path calls `terminate_managed_primary_processes()` explicitly when
finalizing a failed managed primary, targeting runtime children by their
recorded PIDs with PID-reuse guards.

See [architecture/managed-primary-lifecycle.md](../architecture/managed-primary-lifecycle.md)
for the managed-primary termination model.

---

## Windows: `asyncio.ProactorEventLoop` Silently Drops Signal Handlers

**The problem:** `loop.add_signal_handler()` does not work on Windows.
`ProactorEventLoop` (the Windows default) either raises `NotImplementedError`
or silently no-ops. Signal handlers registered this way are never called.

**The symptom:** Cancellation via Ctrl-C or SIGTERM has no effect inside async
spawn runners on Windows. The process ignores the signal and keeps running.

**The fix:** Use `signal.signal()` + `loop.call_soon_threadsafe()` instead:

```python
def _handle(signum: int, frame: object) -> None:
    loop.call_soon_threadsafe(shutdown_event.set)

signal.signal(signal.SIGINT, _handle)
signal.signal(signal.SIGTERM, _handle)
```

`signal.signal()` works cross-platform. The thread-safety note is real:
`signal.signal()` must be called from the main thread. Check
`threading.current_thread() is threading.main_thread()` before installing
and skip (returning `None`) when in a worker thread — callers must handle the
`None` return meaning "no cleanup to call."

See `lib/launch/streaming_runner.py:_install_signal_handlers()` for the
canonical implementation.

---

## Windows: Claude Cancel Used `SIGINT` — Silent No-Op on Non-Console Processes

**The problem:** The Claude harness adapter cancelled spawns via
`process.send_signal(SIGINT)`. On Windows, `SIGINT` maps to
`GenerateConsoleCtrlEvent(CTRL_C_EVENT)`, which only reaches processes sharing
the same console process group. Harness subprocesses launched with
`CREATE_NEW_PROCESS_GROUP` (isolation default) never receive the event. The
call silently succeeds from Python's perspective but the subprocess is
unaffected.

**The symptom:** `meridian spawn cancel` appeared to succeed but the Claude
process kept running on Windows.

**The fix:** Branch on platform:
- POSIX: `process.send_signal(SIGINT)` — graceful interrupt with signal handler opportunity
- Windows: `process.terminate()` — maps to `TerminateProcess`, immediate kill

Do not claim `os.kill(pid, SIGTERM)` fails on Windows — it works (maps to
`TerminateProcess`). The problem is `SIGINT`, not `SIGTERM`.

---

## Windows Audit Lesson: Validate Claims Against CPython Source Before Filing Bugs

**The problem:** The initial Windows compatibility audit identified 9 critical
bugs. Design-lead validation reduced this to 3 genuine criticals + 1 CI gap.
Several audit claims were incorrect:

- `os.kill(pid, SIGTERM)` was claimed to "silently fail" on Windows — it maps
  to `TerminateProcess` and works. The concern was ungraceful termination, not
  failure.
- Other claims assumed POSIX-only stdlib implementations without checking
  CPython's Windows-specific branches.

**The lesson:** Before filing a Windows compatibility bug, verify the claim
against CPython source or a quick test. Many stdlib functions have platform
branches that aren't obvious from the API docs. False positives waste
engineering time and create incorrect KB entries.

Reliable sources: CPython `Modules/posixmodule.c` for `os.*` signal/process
calls; `Lib/asyncio/` for event loop signal-handler behavior.

---

## No-arg `spawn wait` Waited on the Full Chat Tree, Not Just Descendants

**The problem:** `meridian spawn wait` (no-arg form) discovered all active
spawns sharing `MERIDIAN_CHAT_ID`. When called from a nested spawn (e.g.,
tech-lead), this included the primary session itself and any sibling spawns.
The primary session is always active while spawns run — it never left the wait
set — so nested `spawn wait` calls blocked forever.

**The symptom:** Tech-lead or other orchestrator spawns would run `spawn wait`
after launching subagents and hang indefinitely. The primary session (running
the same chat) was in the wait set but was never going to reach a terminal
status while the orchestrator was still running.

**The fix:** When `MERIDIAN_SPAWN_ID` is set (inside a nested spawn),
`_resolve_wait_targets()` passes `only_descendants_of=self_spawn_id` to
`_discover_pending_spawns()`. The discovery function uses BFS via
`_collect_descendants()` to scope the wait set to the caller's subtree only.
Top-level (primary) `spawn wait` behavior is unchanged.

**Impact:** Any nested agent orchestration pattern that used no-arg `spawn wait`
was silently broken before this fix. The fix was applied in
`src/meridian/lib/ops/spawn/api.py` (PR #199).

See [../concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) for
the full documented semantics including descendant-scoping.

---

---

## `prepare_prompt_payload()` or-Chain Silently Drops `appended_system_prompt` (PR #208)

**The problem:** `prepare_prompt_payload(projected_content=X, appended_system_prompt=Y)`
returns a payload where `appended_system_prompt` is empty when `X` has a non-empty
`system_prompt` field. The implementation uses an or-chain:

```python
PreparedPromptPayload(
    system_prompt=projected_system_prompt or appended_system_prompt,
    ...
)
```

When `projected_system_prompt` is truthy, `appended_system_prompt` is never evaluated —
it disappears silently. No error, no warning.

**The symptom:** After adding skill document suppression (`supports_native_skills=True`
stops populating `supplemental_documents`), skill content that should arrive via
`--append-system-prompt` in Claude spawns was not reaching the harness. The regression
was discovered in smoke testing — skill documents simply weren't present in Claude
spawn system prompts.

**Root cause:** The Claude skill injection path calls `prepare_prompt_payload()` with
both `projected_content` (from content composition) and `appended_system_prompt`
(from `compose_skill_injections()`). The or-chain preferred the projected system prompt
and dropped the appended skills.

**The fix:** Directly construct `PreparedPromptPayload` with both fields populated
independently, bypassing the or-chain:

```python
PreparedPromptPayload(
    system_prompt=projected_system_prompt,
    appended_system_prompt=appended_system_prompt,  # set independently
    user_turn_content=projected_user_turn_content,
)
```

**The rule:** Any callsite that needs *both* a projected system prompt *and* an
independently appended system prompt must construct `PreparedPromptPayload` directly.
`prepare_prompt_payload()` is a convenience helper that assumes at most one source
of system prompt content; it is not safe for the two-channel case.

**Why it wasn't caught earlier:** The two delivery channels coexisted undetected —
before skill document suppression, skill content arrived via *both* channels. After
suppression, only `--append-system-prompt` remained, and the or-chain drop became
the only path. The bug was latent but inactive.

---

## Worktree Repo Detection Uses `cwd`, Not `MERIDIAN_PROJECT_DIR` (PR #266)

**The problem:** When invoking `meridian work worktree --ensure` (or `spawn --worktree`) without an explicit `--repo`, Meridian detects the target git repository from the **current working directory** — not from `MERIDIAN_PROJECT_DIR`. If the shell cwd is a feature worktree, the managed worktree is provisioned in that repo, even when `MERIDIAN_PROJECT_DIR` points to a different project root.

**The symptom:** Running `uv run meridian work worktree --ensure` from a feature worktree (e.g., the spawn-worktree-autoprovision worktree) provisions a worktree under the feature worktree's repo root, not the main checkout the user expected.

**The rule:** When the target repo differs from the current cwd, always pass `--repo <path>` explicitly. The `--repo` flag is only consumed on the **first** managed ensure for a work item — subsequent ensures ignore it and reuse persisted metadata.

**Why cwd wins:** `--repo` is an initial configuration hint, not an ongoing override. Repo detection from cwd is intentional — it matches the "you are working here" mental model. `MERIDIAN_PROJECT_DIR` tracks project identity for CLI context resolution, not for worktree provisioning.

---

## Cross-References

- [architecture/managed-primary-lifecycle.md](../architecture/managed-primary-lifecycle.md) — managed primary reaper safety boundary
- [operations/health-checks.md](../operations/health-checks.md) — doctor `--kill-orphans` flag
- [principles/design-principles.md](../principles/design-principles.md) — crash-only, files-as-authority
- [codebase/observability.md](../codebase/observability.md) — stdlib vs structlog boundary and why it matters
- [concepts/spawn-wait-barrier.md](../concepts/spawn-wait-barrier.md) — spawn wait descendant-scoping semantics
