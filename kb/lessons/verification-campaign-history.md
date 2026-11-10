# Verification Hazards

An ownership check based only on content can follow a symlink to matching external bytes and wrongly claim the link as managed output. Verify filesystem structure as well as content before asserting ownership. See [package management decisions](../concepts/package-management.md).

A capability token proves a specific operation was checked only if the consumer binds its identity to the checked target. A regression test can also pass at an earlier gate without reaching the failure it claims to exercise; verify the intended path was reached. See [review convergence](review-convergence-gate.md).
