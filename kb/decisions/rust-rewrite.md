# The Rust Rewrite

The rewrite is motivated by memory use and crashes when a running Python installation is updated underneath itself. It has no settled design yet.

## D-update-crash-waits-for-rust: no Python fix for #628 {#d-update-crash-waits-for-rust}

`uv tool install --force` can overwrite modules in a live process's virtualenv; a later lazy import can mix incompatible versions. Until the rewrite addresses [#628](https://github.com/haowjy/meridian-cli/issues/628), avoid upgrading while long-lived Meridian processes run.

A static binary alone does not solve upgrades: replace it by rename, avoid re-exec through a mutable path, embed or version assets, and version shared state schemas for old and new processes running together.

The Python import floor was measured near 90 MB per process; algorithmic heap growth is separate and would survive a direct port of the same algorithm.
