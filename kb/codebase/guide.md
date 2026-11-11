# Codebase guide

The source checkout's `src/meridian/AGENTS.md` and `.context/CONTEXT.md` describe the package layers and entry points, subdirectory guidance owns the rules for each area, and `tests/AGENTS.md` owns testing. This page keeps reasons that span the tree. Remaining cleanup work is tracked in [#569](https://github.com/haowjy/meridian-cli/issues/569); deferred items sit in each directory's `.context/TODO` and `FUTURE`.

## Static typing

Pyright runs strict with warnings fatal, so uncertainty stays at the boundary where data enters the program: validate external data there rather than casting an unchecked shape into a trusted type.

- **One rule, one line, one reason.** Pyright treats `# type: ignore[code]` as a blanket ignore, so suppressions use a rule-specific `# pyright: ignore[...]` with a reason, and project types get fixed rather than suppressed.
- **A name imported by another module is public.** Private access into a third-party library lives in one adapter, so an upstream upgrade has one place to recheck; Cyclopts help rendering uses `cli/cyclopts_internals.py` for this.
- **Nested command handlers register explicitly.** Pyright's unused-function check doesn't count their decorators as references; module-level handlers don't have this problem.

## CLI startup

A lightweight command catalog and lazy imports keep trivial commands from paying for operation and harness dependencies. Read-only resolution never creates project state; only `ensure` operations mutate (source checkout `src/meridian/cli/AGENTS.md` and `.context/CONTEXT.md`). Classification uses longest-prefix matching, so a command with no descriptor silently inherits its parent's bootstrap requirements: some bare command groups classify as a primary launch and can initialize state though they look read-only. Telemetry installation is split between CLI startup and operation-owned background paths, with no single process seam yet, so a change must account for both.

## Removing and merging code

- **Removing a seam is a hypothesis about its contract.** Check the surviving boundary behavior; where coverage is missing, add a focused contract test rather than restoring a broad suite that pins internal wiring. Decide which layer owns the remaining logic, or it scatters across callers.
- **Derive parallel maps from one registry.** Two maps describing one fact diverge; keep state facts separate from rendering, and prefer a projection of an existing authority over a mirror with its own precedence rules. Harness-specific phrasing belongs at the harness boundary.
- **A shared helper must keep the distinctions callers rely on.** A generic JSON helper would erase the difference between an in-memory SQLite value and a file read that tells missing from unreadable; a replacement config parser must keep merging, diagnostics and coercion, not just accept the same syntax.
- **A correctness-bearing parameter has no do-nothing default.** An optional default hides missed callers; make each caller supply the value or explicitly choose absence.
- **Function-local imports hide dependency cycles** from import-time checks, so boundaries are enforced across all import sites.
- **An environment variable can carry a parameter** at a bounded CLI entry point, but not a value that needs local scope: it mutates global state and propagates to children.

## Config warnings come from the validated model

Unknown-key warnings walk the validated model's values rather than a second traversal of type annotations, which drifted on dict-root models and produced false warnings. The warning pass validates with extras temporarily allowed; making extras permanent would expose them as effective config. The caller must follow with the authoritative validation, because a hard error suppresses the warnings.

## Source lints

A source-tree lint may use a text prefilter only for identifier tokens: string literals and dotted names can be spelled differently with the same parsed meaning. Parsing every module for those rules costs about 0.75 s per gate.

## Flaky tests: fix or delete, after checking

Before deleting a flaky test, break the behavior it protects and run the rest of the suite. If nothing else fails, make its coverage deterministic instead. A flake can expose a product bug, so investigate the product as well as the test ([testing](testing.md)).

## Documentation checks

`meridian kg check` validates document links and `meridian mermaid check` validates diagrams; both share one Markdown extraction library. Usage and parser limits are in the Markdown validation skill.
