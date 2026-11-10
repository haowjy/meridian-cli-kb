# Config Precedence

Project-wide operational settings and per-launch policy are separate: the former load from configuration files, while the latter resolve independently for each launch. The source guidance is `src/meridian/lib/config/AGENTS.md`; user-facing settings are in `docs/configuration/`.

## Routing vs Policy Fields

Model and harness selection decide where to run; execution policy decides how. Agent overlays let a project change a profile's policy without editing generated profiles. A matched model-policy rule can refine the profile's generic defaults; an overlay scalar takes precedence over that rule. CLI and environment overrides take precedence over both. See [model resolution](../decisions/model-resolution.md) and `docs/configuration/mars.md` for the complete ladder.

## Declaring config keys

Only fields carrying catalog metadata receive environment overrides and generic config commands. The model owns validation; see `src/meridian/lib/config/AGENTS.md`.

## The `-C` / `--directory` Flag

`-C` selects the project root ahead of inherited `MERIDIAN_PROJECT_DIR` and cwd. Project identity and package discovery are separate; see [state model](../architecture/state-system/overview.md).
