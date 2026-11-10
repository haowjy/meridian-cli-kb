# Config precedence

Project-wide operational settings load from configuration files; per-launch policy resolves separately for each launch. The ladder and source guidance are in meridian-cli `src/meridian/lib/config/AGENTS.md`; settings in `docs/configuration/`.


## Routing vs policy fields

Model and harness selection decide where a launch runs; execution policy decides how. Agent overlays let a project change a profile's policy without editing generated profiles. A matched model-policy rule refines the profile's generic defaults, an overlay scalar beats that rule, and CLI and environment overrides beat both ([model resolution](model-resolution.md#policies-and-fallback); full ladder in `docs/configuration/mars.md`).

`-C` selects the project root ahead of an inherited `MERIDIAN_PROJECT_DIR` and the cwd; project identity and package discovery stay separate ([state system](../architecture/state-system/overview.md)).
