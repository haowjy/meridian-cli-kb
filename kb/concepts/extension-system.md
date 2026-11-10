# Extension system

The CLI and the MCP server are Meridian's two command surfaces, and they share one command specification per command, so their validation can't diverge; each command declares which surfaces expose it. The capability and trust checks are the seam for future third-party extensions; first-party commands don't exercise capability grants yet. There is no HTTP server surface. Source: `src/meridian/lib/extensions/.context/CONTEXT.md`; command registration: [CLI startup](../architecture/startup-pipeline.md).
