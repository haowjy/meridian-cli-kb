# `meridian init`

Setup preserves an existing primary-agent choice: a package recommendation must not silently override user configuration. Initializing after ad-hoc use creates a separate project identity and leaves the old state intact. See the [command reference](https://github.com/haowjy/meridian-cli/blob/main/docs/commands.md) and [project identity](../architecture/state-system/project-identity.md).

## Idempotency

Setup can be rerun without replacing the existing project configuration. Bootstrap performs setup before starting a new session.
