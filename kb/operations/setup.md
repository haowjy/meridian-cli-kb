# `meridian init` and `meridian bootstrap`

`meridian init --add` sets up a project; `meridian bootstrap --add` runs the same setup and then starts a guided primary session. Options are in meridian-cli `docs/commands.md`.

- **Setup keeps an existing primary-agent choice.** A package's recommendation must not silently override the user's configuration.
- **Setup is idempotent:** rerunning it doesn't replace existing project configuration.
- **Initializing after ad-hoc use creates a new project identity** and leaves the ad-hoc state where it was ([project identity](../architecture/state-system/project-identity.md)).
- **Bootstrap resolves its own root,** so first use works with no project state yet.

Package bootstrap documents carry setup guidance that spans skills; they stay in the canonical `.mars/` store instead of riding along with every skill, so setup detail isn't loaded on every invocation. Skill resources travel with their skill, including into native harness directories. Discovery and authoring: mars-agents `docs/config/bootstrap-docs.md`.
