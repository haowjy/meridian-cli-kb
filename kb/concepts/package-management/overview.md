# Package Management

Mars compiles package content into a canonical store and native harness projections. Meridian consumes the result instead of maintaining an independent package manager. Package and user scopes are separate; project content takes precedence where fallback is supported. See [Mars user scope](../../architecture/mars-user-scope.md) and [package management decisions](../../decisions/package-management.md).

The subtree covers [resolution](resolution-algorithm.md), [current-package selection](self-source-selection.md), [compilation](compiler-pipeline.md), [targeting](targeting.md), [sync and recovery](sync-model.md), and [vocabulary](vocabulary.md).
