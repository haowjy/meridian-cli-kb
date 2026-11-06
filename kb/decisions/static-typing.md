# Static Typing

Pyright strict mode with warnings fatal keeps uncertainty at the boundary where data enters the program. Validate external data there rather than casting an unchecked shape into a trusted type.

## D-pyright-suppression: one rule, one line, one reason

Pyright treats `# type: ignore[code]` as a blanket ignore, so use a rule-specific `# pyright: ignore[...]` with a reason. Fix project types rather than suppressing them.

## D-private-usage: cross-module names are public

A name imported by another module is public. Third-party private access belongs in one adapter so an upstream upgrade has one place to recheck; Cyclopts help rendering uses `cli/cyclopts_internals.py` for this reason.

## D-explicit-handler-registration

Register nested command handlers explicitly: Pyright's unused-function check does not count their decorators as references. Module-level public handlers do not have this problem.
