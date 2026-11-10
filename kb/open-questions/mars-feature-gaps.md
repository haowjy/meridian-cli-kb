# Mars Feature Gaps

- Package-level permission declarations remain distinct from [per-profile permissions](../architecture/tool-permissions.md). Without them, a package may rely on capabilities its environment does not grant.
- Tool definitions and capability availability are not generally distributed and validated as a package contract. MCP registrations are supported, but external service dependencies and resource exposure remain open.
- Dynamic lifecycle and post-sync hooks remain an open concern; native harness hooks are covered by [package management](../concepts/package-hooks.md).
- Package discovery, publisher trust, provenance and source-based install policy remain open beyond Git and path sources.
- Cursor fast model variants are filtered from discovery; whether speed belongs in effort or a separate axis is unresolved.
