# lessons/pi-rpc-quiescence-impl — Pi RPC Quiescence Implementation Lessons

Hard-won cross-platform and test-design patterns from implementing Pi RPC quiescence
and getting CI green on all platforms. Current quiescence architecture lives in
[architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md); this page keeps the
durable implementation lessons that still apply.

## Windows Path Escaping in Test Fixtures

**The bug:** Tests wrote session JSONL files using f-strings: `f'{{"cwd":"{child_cwd}"}}'`. On Windows, `child_cwd` is `C:\Users\...` — the backslashes produce invalid JSON (`Invalid \escape`).

**Why it persisted:** Tests passed locally on POSIX where paths use `/`. The bug only surfaced in CI's Windows gate.

**The fix:** Use `json.dumps()` for any test fixture content that embeds filesystem paths:
```python
# Bad — breaks on Windows
f'{{"type":"session","cwd":"{some_path}"}}\n'

# Good — json.dumps escapes backslashes
json.dumps({"type": "session", "cwd": str(some_path)}) + "\n"
```

**The lesson:** Never embed `Path` objects or `str(path)` directly into hand-built JSON strings. Always use `json.dumps`. This applies to any test that creates JSONL fixtures, not just Pi tests.

---

## Node ESM Loader Requires file:// URLs on Windows

**The bug:** Python tests passed Windows paths like `D:\a\...\index.js` to Node's ESM `import()` via environment variables. Node's ESM loader rejects bare Windows paths — `D:` is parsed as a URL protocol, giving `ERR_UNSUPPORTED_ESM_URL_SCHEME`.

**The fix:** Convert paths to `file://` URLs with `Path.as_uri()`:
```python
# Bad — fails on Windows
env["EXTENSION_PATH"] = str(extension_dist_path)

# Good — works everywhere
env["EXTENSION_PATH"] = extension_dist_path.as_uri()
```

**The lesson:** When passing filesystem paths to Node.js for ESM `import()`, always use `file://` URLs. POSIX paths happen to work without `file://` but Windows paths don't. `Path.as_uri()` handles both.

---

## pnpm Version Mismatch: Local vs CI

**The bug chain:**
1. Local dev had pnpm 10.x. CI corepack resolved pnpm 11.x (latest).
2. pnpm 11 moved settings out of `package.json` into `pnpm-workspace.yaml`.
3. `pnpm.onlyBuiltDependencies` in `package.json` was silently ignored by pnpm 11.
4. esbuild's postinstall script was blocked → `ERR_PNPM_IGNORED_BUILDS` → CI exit 1.

**The fix:** Dual config — `pnpm.onlyBuiltDependencies` in `package.json` (pnpm 10) AND `allowBuilds` + `strictDepBuilds: false` in `pnpm-workspace.yaml` (pnpm 11). Use `packages: []` in the workspace yaml to prevent workspace scanning.

**The lesson:** When using corepack without a pinned `packageManager` field, CI may pick a different major version. Config that works locally can be silently ignored. Either pin the version or provide config for both majors.

---

## Commit the Lockfile

**The bug:** `pnpm-lock.yaml` was gitignored. CI had to use `--no-frozen-lockfile`, making builds non-reproducible. Dependency resolution happened fresh on every CI run.

**The fix:** Remove `pnpm-lock.yaml` from `.gitignore`, commit it, switch CI to `--frozen-lockfile`.

**The lesson:** Lockfiles exist for reproducibility. Gitignoring them in a package that CI needs to build is wrong. If the lockfile was gitignored to reduce noise, the cost is non-deterministic CI.

---

## Flaky Tests Hide Design Flaws

**The bug:** `test_pi_connection_launches_in_task_cwd_when_provided` intermittently failed with `ConnectionNotReady`. The test shim printed a session event and immediately exited. Sometimes the connection couldn't write the initial prompt before the process died.

**Why it was "flaky":** The test was racing the subprocess lifecycle. It passed most of the time because the connection usually wrote fast enough. On slow CI runners, it lost the race.

**The fix:** Make the shim wait for input (`read _prompt_line`) before exiting. This isn't a timing hack — it models real Pi behavior where the process stays alive to receive prompts.

**The lesson:** A flaky test is a test with a hidden precondition. "Retry it" is never the fix. Either the test's environment setup is incomplete (missing a synchronization point) or the test is asserting something that isn't deterministic. Both are design flaws.

---

## Batch Windows Failures Instead of Iterating

**What happened:** CI Windows gate failed. Fixed one test. Pushed. Windows gate failed with a different test. Fixed that. Pushed. Repeated 3 times.

**What should have happened:** Dump all Windows failures from the first CI run. Read every failing assertion. Fix all of them in one commit. Push once.

**The lesson:** When CI fails on a platform you can't run locally, always read ALL failures from the log before fixing any of them. The cost of one extra `gh run view --log-failed` read is near zero. The cost of an extra CI round-trip is 3-5 minutes × number of iterations.

---

## Test Path Assertions Must Be Platform-Aware

**The bug pattern:** Tests asserted exact path strings like `"Unable to execute Pi at /bad/pi:"` or `resolved.binary_path == "/tmp/pi"`. On Windows, `Path("/bad/pi")` stringifies to `\bad\pi`.

**The subtlety:** Not all paths get normalized. `MERIDIAN_PI_BINARY` values go through `Path()` in the resolver (producing `\bad\pi` on Windows). But `shutil.which()` return values are kept as raw strings (staying as `/tmp/pi` even on Windows). You have to know which normalization the code under test applies.

**The fix:** Match the code's behavior:
```python
# For paths the code normalizes through Path:
assert result == str(Path("/tmp/pi"))

# For paths the code keeps as raw strings:
assert result == "/tmp/pi"  # or use the variable directly
```

**The lesson:** Never hardcode POSIX paths in assertions unless the test is explicitly POSIX-only (`skipif win32`). Always trace how the path flows through the code under test to know whether it gets normalized.

---

## Opening a Directory Fails Differently on Windows

**The bug:** Test set an extension-path environment variable to a directory path, expecting Node's `fs.open()` to fail (`EISDIR` on POSIX). On Windows, some Node/OS combos handle directory opens differently and don't throw.

**The fix:** Use a path through a non-existent parent directory instead. `ENOENT` is universal.

**The lesson:** Filesystem error behavior is platform-specific. "Open a directory for writing fails" is a POSIX assumption. When testing error paths, pick stimuli that fail universally: non-existent paths, permission-denied (with care), or explicitly invalid inputs.

---

## Cross-References

- [harness-integration.md](harness-integration.md) — Claude, Codex, and OpenCode integration patterns
- [architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) — quiescence architecture, state machine, and extension model
- [arch-refactor-pitfalls.md](arch-refactor-pitfalls.md) — Earlier architecture refactor pitfalls including Windows asyncio and process-group cleanup
