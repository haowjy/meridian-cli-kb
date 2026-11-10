# The Rust Rewrite

The rewrite is motivated by memory use and crashes when a running Python installation is updated underneath itself. It has no settled design yet.

## Upgrade crashes await the rewrite

`uv tool install --force` can overwrite modules in a live process's virtualenv; a later lazy import can mix incompatible versions. Until the rewrite addresses [#628](https://github.com/haowjy/meridian-cli/issues/628), avoid upgrading while long-lived Meridian processes run.

A static binary alone does not solve upgrades. Replacing it by rename protects running processes from partial replacement; avoiding re-exec through a mutable path prevents switching versions mid-run. Embedded or versioned assets prevent mixed versions, and versioned shared state schemas let old and new processes coexist. These are considerations, not a settled design.

The Python import floor was measured near 90 MB per process, so concurrent Meridian processes multiply the baseline memory cost. This is a baseline measurement, not a comparison with a Rust implementation; algorithmic heap growth is separate and would survive a direct port of the same algorithm.
