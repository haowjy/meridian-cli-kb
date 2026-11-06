# Package Management Vocabulary

**Self source:** current-project content selected from `.mars-src` and, for a declared package, package content; `.mars-src` wins matching definitions. See [self-source selection](self-source-selection.md).

**Occupied layer:** the shallowest logical package layer containing convention items; deeper layers are ineligible while it is occupied. See [self-source selection](self-source-selection.md).

**Canonical write intent:** evidence recorded before a new canonical output is written, used to recover ownership after interruption. It is not an installed claim or a replacement for the lock. See [sync model](sync-model.md#pending-canonical-writes).
