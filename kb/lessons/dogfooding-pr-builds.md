# Running Unreleased Builds

## A PR build must never touch a real runtime root

A build under test can write state that an installed older build cannot read, even during a nominally read-only operation. Use copied state and an isolated environment from the first command. Inherited project, chat, harness and depth variables can redirect a scratch run to the real project. See [probe runtime placement](probe-runtime-placement.md).

An isolated `uv tool` install needs both tool and executable directories isolated: otherwise it can relink the user's `meridian`. A login shell can rebuild PATH, so invoke the intended build by absolute path and verify its behavior.

## Probing the upgrade path needs a genuinely old build and old state

Create test state with the actual older build, establish its identity by behavior rather than version string, and test overlap while an old process is still running. An unreleased build can report the same version as the release and cached tool environments can select it accidentally.

## Writing over `.venv/bin/*` rewrites uv's cache

uv may hard-link executables to its cache, so writing through a venv executable can change other environments too. Use a scratch venv with copy link mode for a development build. Check inode/link count or content hash rather than relying on its version string.
