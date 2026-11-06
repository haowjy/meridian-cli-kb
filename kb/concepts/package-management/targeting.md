# Targeting

Mars maintains a canonical store and projects selected content to native harness surfaces. Native formats require lowering rather than assuming each harness understands Mars profiles. Separate managed targets avoid ambiguity over shared skill directories; agent emission depends on whether Meridian or a native harness owns routing. See [package management decisions](../../decisions/package-management.md) and `src/compiler/AGENTS.md` in mars-agents.

Known harness targets can constrain routing; generic targets control materialization without naming a harness route. See `docs/config/mars-toml.md` in mars-agents for the current configuration.
