# CLI startup

A lightweight command catalog and lazy imports keep trivial commands from paying for operation and harness dependencies. Read-only resolution never creates project state; only `ensure` operations mutate. Contract: the source checkout's `src/meridian/cli/AGENTS.md` and `.context/CONTEXT.md`.

Classification uses longest-prefix matching, so a command with no descriptor silently inherits its parent's bootstrap requirements. Bare command groups hit this: some classify as a primary launch and can initialize state though they look read-only.

Telemetry installation is split between CLI startup and operation-owned background paths, with no single process seam yet, so a change must account for both ([telemetry](telemetry.md)).
