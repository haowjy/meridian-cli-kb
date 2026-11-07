# CLI startup

Classification uses longest-prefix matching, so a missing command descriptor can silently inherit a parent's bootstrap requirements. Bare command groups currently expose this hazard: some classify as primary launch and can initialize state despite appearing read-only. See the source checkout's `src/meridian/cli/AGENTS.md` and `.context/CONTEXT.md`.

## Telemetry

Telemetry installation is split across CLI startup and operation-owned background paths. A single process seam is not yet implemented, so changes must account for both. See [startup decisions](../decisions/startup-health-sandbox.md).

