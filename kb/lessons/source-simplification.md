# Source Simplification

Removing a seam is a hypothesis about its contract. Check the surviving boundary behavior, and when coverage is missing add a focused contract rather than restoring a broad suite that pins internal wiring. See [testing decisions](../decisions/testing.md).

When removing a seam, establish which layer owns the remaining logic; otherwise it can scatter across callers. A documented environment variable can serve as the parameter channel at a bounded CLI entry point, but hidden global mutation or propagation to children makes it unsuitable for values that need local scope. See [state decisions](../decisions/state.md).
