# Mars Feature Gaps

- Package-level permission declarations remain distinct from [per-profile permissions](../decisions/tool-permissions.md). Without them, a package may rely on capabilities its environment does not grant.
- Tool definitions and capability availability are not generally distributed and validated as a package contract. MCP registrations are supported, but external service dependencies and resource exposure remain open.
- Dynamic lifecycle and post-sync hooks remain an open concern; native harness hooks are covered by [package management](../decisions/mars-portable-hooks.md).
- Package discovery, publisher trust, provenance and source-based install policy remain open beyond Git and path sources.
- Cursor fast model variants are filtered from discovery; whether speed belongs in effort or a separate axis remains unresolved ([mars-agents #256](https://github.com/haowjy/mars-agents/issues/256)).
- Meridian's launch bundle path does not expose Mars's refresh controls as spawn flags; offline mode is available. See [model refresh](../architecture/mars-model-refresh.md).
