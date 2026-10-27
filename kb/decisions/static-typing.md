# Decision: Static Typing Gate

Pyright strict mode is the type gate for meridian-cli, with warnings treated as
failures. Type uncertainty gets resolved where data enters the program, not
suppressed or cast away where it is used.

## D-pyright-strict: strict with no demotions, warnings fatal (2026-10-13)

**Decision:** `pyproject.toml` runs `typeCheckingMode = "strict"` with no rule
demoted, and the gate is `pyright --warnings` at 0 errors, 0 warnings. The only
config exception is `reportMissingModuleSource = false`, which covers stub-only
packages. User: "we are in strict mode for a reason"; "warnings are also
warnings for a reason."

Both gates enforce it: the fast gate in `src/meridian/dev/preflight.py` (pre-push
hook and the PR CI job) and the full preflight run `pyright --warnings`. The fast
gate originally ran plain `pyright`, which is how the warnings piled up unseen;
`--warnings` was added before PR #570.

**Why strict had been weakened.** On 2026-04-19, two commits (`13d596c13`,
`6f8ae96bf`) demoted seven strict rules to `warning`, for two different reasons:

| Rules | Stated reason in April | Why it no longer holds |
|---|---|---|
| `reportUnusedFunction` | CLI commands registered by decorator look unused | Registration inside a function is now an explicit call (see below) |
| `reportPrivateUsage` | Cross-module internal helpers are intentional | A name another module imports is public; we renamed ours |
| `reportUnnecessaryCast` | Casts may become redundant after narrowing improves | A redundant cast is dead code and should be deleted |
| `reportAttributeAccessIssue`, `reportUnknownMemberType`, `reportUnknownVariableType`, `reportUnknownArgumentType` | POSIX-only APIs (`SIGWINCH`, `os.fork`, unix sockets) failed pyright on Windows CI | Meridian is POSIX-only and CI is ubuntu-only (meridian-cli root `AGENTS.md`, "POSIX-first") |

Pyright exits 0 when it reports only warnings, so the demoted rules stopped
gating anything. By October, 85 diagnostics had piled up without failing preflight
or CI; the user noticed them in pyright output.

**What restoring it took.** Branch `fix/pyright-strict` (25 commits; shipped in
v0.9.10 through PR #570, merged 2026-10-14) fixed every
diagnostic at its source. It also changed a few behaviors where a cast had been
hiding malformed input. Legacy native-import rows with a non-string `chat_id` or a
non-list `harness_session_ids` are now skipped; before, a string could be split
into fake IDs. Process-scope sidecar rows are validated against a strict schema
that keeps the existing malformed-row rejection.

## D-pyright-suppression: one rule, one line, one reason (2026-10-13)

**Decision:** The only allowed suppression is `# pyright: ignore[rule]  # reason`.
`enableTypeIgnoreComments = false` disables `# type: ignore`, and
`reportUnnecessaryTypeIgnoreComment = "error"` fails the gate on stale
suppressions. File-level `# pyright: reportX=false` pragmas are not used.

**Why:** Pyright does not understand mypy error codes. To pyright,
`# type: ignore[attr-defined]` means `# type: ignore`, which hides every
diagnostic on that line, including ones introduced later. Suppressions are for third-party code that is untyped or
private. For our own code, fix the type.

## D-typed-boundaries: narrow at the boundary, not with casts (2026-10-13)

**Decision:** Untyped data is validated once where it enters the program. Past
that point, code is fully typed. A `cast` is allowed only after a real runtime
check, or for a live object whose protocol really is typed. It must never assert
the shape of input nobody checked. The reviewer's version: "validate once at the
boundary, keep code past the boundary typed" (`spawn:p7727`).

The boundary tool depends on how fixed the input's shape is:

| Input | Tool | Examples |
|---|---|---|
| JSON of loose or partial shape (harness transcripts, event payloads) | `TypeIs` helpers in `lib/utils/json_shapes.py`: `json_object`, `json_list`, `loads_json_object` | capture qualification, OpenCode transcript/idle, retention archive and digest |
| A record or envelope with a fixed shape | Pydantic model | Codex rollout and Claude session headers, the Mars error envelope, the session-archive header |
| A persisted row whose exact key set is the contract | Strict `TypedDict` (`extra="forbid"`, strict types) validated through a module-level `TypeAdapter` | `ScopeRow` in the process-scope projection rejects string or bool PIDs, extra keys, and string epochs |

`json_shapes` lives in `lib/utils/` rather than `lib/core/` because it is a
dependency-free helper with no domain meaning. `loads_json_object` catches
`ValueError`, not just `JSONDecodeError`, so an oversized integer reads as "not
an object" and does not raise.

**Scope of the sweep:** The branch converted only the sites it touched, plus
seven private duplicates of the same helper. That cut casts in those files from
178 to 127. About 300 older casts elsewhere were left on purpose. The planned
state-layer restructures (typed JSON documents, collapsing the settings
normalizers into Pydantic) will delete many of them. Converting them first would
be wasted work. For new code, use the table above instead of copying a nearby
cast.

## D-private-usage: cross-module names are public (2026-10-13)

**Decision:** If another module imports a name, it is public, so drop the
underscore. When we reach into a third party's private API, all of that access
goes through one adapter module, with a per-line suppression and a reason. For
Cyclopts this is `cli/cyclopts_internals.py`. Help rendering needs
`App._assemble_help_panels` and the raw `Group._show`. The module records which
Cyclopts version it was checked against, so an upgrade has a single place to
recheck.

**Exception kept:** Harness connection classes declare a class-private
`_CAPABILITIES`, which is the convention for new transports. The one place that
needs another class's capabilities reads a module-level constant
(`OPENCODE_V1_CAPABILITIES`). The class attribute keeps its underscore. Renaming
it everywhere would have broken a documented convention to satisfy a single
access site.

**Deferred:** Some OpenCode V2 helpers are public only so the V2 transport can
reuse V1 internals. The fix is template hooks, and it waits for the planned
harness dialect restructure. Until then those names stay public.

## D-explicit-handler-registration (2026-10-13)

**Decision:** A command handler defined inside a `register_*_commands()` function
is registered by a call: `sub_app.command(fn, name=...)`, `sub_app.default(fn)`,
or `bindings.add(key)(fn)` for prompt_toolkit key bindings. Decorated handlers at
module level (`kg_cmd.py`, `qi_cmd.py`) stay as they are, because pyright counts
a module-level public function as used.

**Why:** `reportUnusedFunction` flags a local function whose only reference is a
decorator. Pyright does not count decoration as use.

**Rejected alternatives:**

| Alternative | Why rejected |
|---|---|
| Keep the April rule demotions | Warnings don't fail pyright, so a demoted rule stops being checked. That is exactly how the 85 diagnostics piled up. |
| `# type: ignore[code]` comments | Pyright reads them as blanket ignores for the whole line |
| File-level `reportPrivateUsage=false` | Turns the rule off for every line in the file, not just the one access that needs it |
| Keep decorators on local handlers and rename them without the underscore | Tried: pyright still flags them, because it checks for references, not names |
| Inline `cast(...)` over decoded JSON | Asserts a shape without checking it; malformed input then fails far from where it entered |
| Several private `isinstance` helpers, or a `TypeAdapter` built on every call | Duplicated narrowing logic. Consolidated into `json_shapes` and module-level adapters. |
| Reflection (`object.__getattribute__`) to reach Cyclopts internals | Hides the private access from the type checker without making it any safer |
| Put `json_shapes` in `lib/core/` | `core` holds domain types; this helper has no domain meaning |

## Revisit when

- Cyclopts exposes public APIs for help-panel assembly or raw group visibility:
  delete `cyclopts_internals.py`.
- Pyright starts counting decoration as use: decorators can come back for nested
  handlers.
- The state-layer restructures land: sweep the remaining casts against the
  boundary table above.

## Sources

`work:pyright-strict` (decisions D1–D7), `chat:c7499`, `spawn:p7727`
(thermo-nuclear review of the branch), `spawn:p7739` (behavior review).
