# Decision: Admit native arguments against the selected route (superseded)

**Superseded 2026-09-24 by the [native session identity decision](native-session-identity.md).**
None of it shipped. It existed only on the comparison branch (PR #519).

This decision had the selected adapter classify every raw passthrough argument. That
meant per-harness option grammars, retained-control carriers, duplicate-scalar
suppression, and typed raw checks ordered before replay (the R1/R2 phases), all
followed by independent final-emission validation. That machinery served a rule the
identity decision later dropped: that a tracked launch needs *observed* entry before
any input. Without that rule, the grammars had no remaining job that justified their
size.

The part that survived is small. A tracked launch refuses raw passthrough flags that
would override Meridian-managed native identity, operation, store, or persistence
(Pi: `--session`, `-c/--continue`, `-r/--resume`, `--session-dir`, `--session-id`,
`--fork`, `--no-session`). This is enforced at the existing adapter projection seam.
Other raw-argument handling stays as it is on `main`, and Mars routing stays at its
existing late point.

**Provenance:** `work:native-harness-session-identity` (`design/b3b-c-late-admission.md`,
`design/b3b-r1-scalar-boundary.md`, `DIVERGENCE/exact-locator-entry.md`).
