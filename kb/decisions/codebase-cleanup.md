# Decision: Codebase Cleanup Program

Decisions about the scope and edges of the [cleanup program](../codebase/cleanup-program.md)
that followed the 2026-10-13 thermo-nuclear review. The program page owns the
diagnosis, phase plan and status; this page owns why the lines were drawn where
they are.

## D-cleanup-scope: phases 1–2 now, 3–5 tracked; stacked on the pyright branch

**Decision (user, 2026-10-13):** Run phases 1 and 2. Defer phases 3–5, the standalone items,
and bugs B5 and B9 (whose fixes belong to those phases) to issue
[#569](https://github.com/haowjy/meridian-cli/issues/569). The cleanup branches
start from `fix/pyright-strict` (PR #570), not `main`, and ship as stacked PRs:
`chore/cleanup-p1` → `chore/cleanup-p2`, each retargeted as its base merges.
They landed that way on 2026-10-14: #570 (v0.9.10), #571 (phase 1, v0.9.11),
#574 (phase 2, v0.9.12).

**Why this cut** (the program's framing; the user chose "Phase 1 + 2" from the backlog's options): Phases 1 and 2 are low-risk and enabling. Phase 1 is bug fixes
and deletions with proof of no readers. Phase 2 builds the shared config and IO
seams that phases 3–4 would otherwise have to work around. Phases 3–4 carry four
behavior changes that each need sign-off (fork unification, the
`SpawnActionOutput` wire format, agent-mode bootstrap, primary honoring
`request.env`), so they wait until those are decided.

**Why stack on the pyright branch:** Phase 2 restructures the same files that
branch had just typed (`json_shapes`, settings, the process-scope sidecar).
Basing on `main` would have guaranteed a conflict at merge. When `main` moved
(v0.9.8), it was merged into `fix/pyright-strict` rather than rebased, because
six cleanup lanes had already branched from it.

**Rejected:** Taking all five phases now (the backlog's other option) — the
phase-3/4 behavior changes are not yet signed off. Basing on `main` — certain
conflicts with the typing work.

## D-b7-via-546: the spawn wait contention failure is fixed by PR #546

**Superseded 2026-10-15** by
[D-history-index-readers-never-wait-on-session-writers](history-storage.md#d-history-index-readers-never-wait-on-session-writers),
landing in PR #588 (#586). PR #546 stalled. The root cause below was incomplete, and
#546's fix (retry within the wait budget) was rejected: retrying waited out the lock
holder instead of shortening it. The holder was a full session-log fold under
`sessions.jsonl.flock` on every session append. Still current: keeping B7 out of the
cleanup program, and refusing to answer from stale discovery. #546 is to be closed
with credit after #588 merges, which needs the user's OK. The record is kept for the
fail-closed reasoning.

**Decision (2026-10-13):** B7 (`meridian spawn wait` failing with "History index has
unresolved sources (7 files)" while six spawns started; a retry succeeded) is
not fixed in the cleanup program. PR #546 owns the fix; B7 closes when it merges.

**Root cause** (`spawn:p7749`): no-argument wait discovery queries the history
index. Its bounded catch-up treats a dirty marker whose source lock is busy, or
whose turn comes after the two-second deadline, as unresolved, and
`HistoryIndex.query()` then raises `HistoryIndexIncomplete`. The seven "files"
were valid markers held by concurrent startup writes. Failing closed is correct:
answering from a stale SQLite projection could omit a just-created spawn, which
would break files-as-authority. The bug is that the UX boundary turns a
transient, replayable condition into a hard failure.

**The fix in #546:** a typed `HistoryIndexBusy` for pending-source coverage,
retried within the wait budget, plus session-scoped discovery. Corruption,
schema, generation and authority failures are still not retried.

**Rejected:** Accepting stale discovery on contention (violates
files-as-authority). Transplanting the #546 commit into a cleanup lane (broader
than the bug, and already in review on its own PR). The structural follow-up —
separating projection coverage from query policy — is state#12, a standalone
item in #569.

## D-spawn-archive-deleted: delete the orphaned overlay, do not keep it

**Decision (2026-10-14):** The spawn-archive overlay — `lib/spawn/archive.py` (briefly moved
to `lib/state/archived_spawns.py`), `SpawnApplicationService.archive()`, the
`spawn.archived` telemetry event, its lock and its tests — was deleted in phase 1.

**Why:** Phase 1 removed the archived app-server extension specs. One of them,
`ARCHIVE_SPAWN_SPEC`, was the only caller of `archive()`. Afterwards nothing
wrote the archive set, nothing read it, no listing applied its "hidden from
default listings" filter, and the telemetry event could never fire. In the same
phase another lane had mechanically moved the module and written fresh docs
describing it as live behavior — dead code with new documentation is the worst
kind, because agents load the docs and reason from them (`spawn:p7763`).

**Rejected:**

| Alternative | Why rejected |
|---|---|
| Keep the moved module and its docs | Describes a visibility overlay that no command applies |
| Keep it behind a flag until a caller returns | There is no consumer to gate; a future spawn-archive feature should be designed against today's listing and history index, and git history has the old code |

## D-auto-init-deleted: delete the inert flag; the real question is project identity

**Decision (2026-10-14):** The descriptor flag `bootstrap_plan.auto_init_cwd` and its
plumbing (`cli/bootstrap.py`, `cli/startup/policy.py`, `cli/startup/catalog.py`) were
deleted in phase 2. Phase 1 had removed a changelog line claiming B3 fixed
auto-init for `spawn done` / `spawn rearm`.

**Why:** Since #388 (July 2026), `resolve_cli_project_root()` reports
`established=True` for every root, so the bootstrap branch that consumed the flag
was unreachable. In a fresh directory `spawn done`, `spawn rearm`, `spawn cancel`
and `work switch` each wrote `meridian.toml`, because `prepare_for_runtime_write()`
creates the project identity, not auto-init. Deleting the flag is correct under
every identity outcome.

**What it surfaced:** asked to choose between deleting the flag and restoring an
established-project check, the user answered "we shouldn't be creating
meridian.toml everywhere... right?". The flag was never the issue; the July
create-on-first-write model was. That question is recorded with its evidence in
[D-two-tier-identity](state.md#d-two-tier-identity), which replaced create-on-first-write.

## D-state1-accepted: accept the first config collapse at −88 lines

**Decision (2026-10-14):** state#1, the first step of folding the TOML normalizers
into the Pydantic models, was accepted on the reviewer's go even though it removed
5 of 20 normalizers (−88 net lines) rather than the ~1,000 the finding estimated.
The rest moved to state#2.

**Why:** Six catalog-aliased sections were still pre-parsed and then validated
again, so most normalizers could not go until the loader itself changed. The
reviewer (`spawn:p7765`) found zero value, precedence or source differences across
about 85 configs and two error-wording regressions; state#2 fixed both, adopted the
reviewer's layout (`config/models.py`, `loader.py`, `dynamic_sections.py`, a plain
`BaseModel` root, key policy as `ClassVar`s, later reduced to one by
[D-unknown-keys-from-pydantic](#d-unknown-keys-from-pydantic-pydantic-does-the-traversal-for-unknown-key-warnings))
and replaced the re-entrant
`BaseSettings` load with an explicit low-to-high merge.

**Context:** the config chain was split by risk on purpose: state#1 ran alone and
was reviewed before state#2 and state#3 started.

## D-g2-double-validation: the first config pass relocated the ladder

**Decision (2026-10-14):** Phase 2 was not called done at the first integration
gate. The thermo-nuclear review (`spawn:p7795`) found the state IO seams real, but
on the config and hooks side "validation done twice" had been moved, not fixed:

- `hooks/schema.py` hand-validated every field and then built `HookSpec` with
  `model_construct`, so the model's own validators never ran in production
  (a probe showed the two had already drifted: `repo` was type-checked, then
  silently dropped).
- `config/dynamic_sections.py` was the old `settings.py` ladder under a new name,
  next to models that declared the same shapes; `[env]` was validated three times.
- `hidden_env_specs` hand-listed per-harness env vars and missed Pi, while idle
  solved the same problem with a model factory.
- `loader._config_validation_error` kept an error-string compatibility table.

**How the fix pass removed it:** hooks validate through
`HookConfigSection.model_validate`, with field constraints on the models and hand
checks only for multi-field rules; dynamic sections validate through descriptor
models (`dynamic_sections.py` has no casts); one `harness_profile_model()`
factory gives every harness its env names (Pi gained
`MERIDIAN_HARNESS_WAIT_YIELD_SECONDS_PI`); one error format in `config/errors.py`;
`MeridianConfig` is the single root and validates `[env]`; `settings.py` was
deleted and 33 importers migrated. The re-verification (`spawn:p7810`) closed both
blockers; a polish pass removed the residue (duplicate hook warnings, the
two-branch model walker, the phantom builtin hook layer, the implicit newline
contract). The correctness review (`spawn:p7794`) had found zero semantic
differences across 600+ env-var cases beyond the intended wording.

## D-unknown-keys-from-pydantic: Pydantic does the traversal for unknown-key warnings

**Decision (2026-10-14, PR #581, released in v0.10.1):** Meridian finds unknown
config keys by validating each section an extra time with Pydantic's call-time
override, `model.model_validate(raw, extra="allow")`, and then reading
`model_extra` off the validated instance tree. The walk goes over values, not
types:
- a `RootModel` descends into `.root`;
- a model reports its extras, then recurses into each field;
- a list or tuple appends `[i]`, and a dict appends `.key`.

Models with `extra="forbid"`, and `ConfigModel`s that set `config_silences_extras`,
report nothing. The code is `warn_unknown_model_keys` and `_warn_extras` in
`lib/config/errors.py`. It relies on Pydantic ≥ 2.12, the project's floor. From
that version on, the `extra=` override applies to every nested model, including
models that set `extra="ignore"` or `"forbid"` themselves.

**Why:** The walker this replaced read field annotations. Its `_nested_model_type`
unwrapped `Model | None`, `tuple[Model, ...]` and `dict[str, Model]`, and it probed
the raw TOML to guess which shape applied. That made it a second description of the
model tree, kept next to Pydantic's, and the two drifted apart:

- **#580 (v0.10.0).** `02124b70c` ("generalize dynamic model traversal", landed with
  #575) ran the walker over every dynamic section. The walker handled list-rooted
  `RootModel`s (`hooks`) but not dict-rooted ones. `DynamicContextConfig` is
  `RootModel[dict[str, DynamicContextEntry]]`, so each context name was checked
  against the model's only field, `root`. Every project with `[context.*]` tables
  then printed `Ignoring unknown Meridian config key 'context.work'` (and one line
  per context) on every config load, three times per primary launch. The keys were
  honored all along, but the warning told users their context config was being
  dropped.
- **#582 item 3.** A mis-shaped root (`[hooks]` written as a table) fell through
  to the plain-model branch, which warned `hooks.a` before the real
  `expected array[table]` error.
- **Latent gaps.** The walker never descended into `list[Model]` fields or into
  `dict[str, Model]` fields other than a root. No current field has either shape
  (`spawn:p7912`; `spawn:p7915` scanned all 24 reachable models), but the next one
  would have let typos through without a warning.

The first #581 commit added a dict-root branch. Review showed that each new shape
needs another branch, and each missed shape produces a false or missing warning.
Walking validated values removes that class of bug: the walk reaches whatever
Pydantic validated, so it can't fall out of step with the models.

**Rejected:**
- **`extra="allow"` on `ConfigModel` itself.** Extras would be carried through
  `model_dump(exclude_unset=True)` into the merged layer dicts and the recorded
  sources, so `config show` would list `context.kb.bogus` as if it were set.
  Every dump site would then have to strip extras. With the call-time override,
  the allow-mode instance exists only inside the warning pass and is then thrown
  away.
- **A forbid pass that reads `extra_forbidden` error locations** (such as
  `('rows', 0, 'z')`). It finds the keys, but a location doesn't say which model
  owns a key. Honoring `config_silences_extras` would then need the annotation
  walk back, or a location-driven walk of the instance. Either is more machinery
  than the chosen design.
- **Keeping the annotation walker with a dict-root branch** (the first #581
  commit). That fixed #580, but it kept two traversals and left #582 item 3 and
  the latent shapes open.

**Accepted costs:**
- **Double validation.** Each section is validated twice per load: once for
  warnings and once for real. At config sizes this is negligible.
- **No warnings for a section that has a hard error.** The warning pass returns
  quietly on `ValidationError`, and the authoritative pass reports the error. The
  warnings appear on the first run after the user fixes the error. This only works
  because both callers (`loader._normalize_toml_payload_values` and
  `dynamic_sections.normalize_dynamic_sections`) run the authoritative validation
  immediately afterwards. A new caller that skipped that step would lose the error
  silently.
- **`config_ignored_keys` is gone.** `PrimaryConfig`'s before-validator already
  discards the deprecated `model` and `harness` keys, so they never reach
  `model_extra`. `config_silences_extras` is now the only warning-policy `ClassVar`.

**Outside this mechanism:** the flattened catalog sections (`[defaults]`,
`[spawn]`, `[timeouts]`) are alias-mapped before any model sees them, so
`loader._map_catalog_table` warns for their keys itself. Each `load_config` call
also repeats real warnings, three times per primary launch. That is noise, not a
bug, and stays unless someone asks.

**How it was decided:** the tech lead's brief to the strict reviewer said that its
default was not to block #581. The reviewer prototyped the restructure, found it
smaller than the PR's own patch (net −44 lines), and still answered "I agree with
your default". The lead overrode that and folded the restructure into #581. It
fixed the root cause in the same function and closed #582 item 3 as well
([lesson](../lessons/review-convergence-gate.md#a-review-brief-must-not-carry-the-leads-verdict)).
The audit that went with it is in
[verification discipline](../lessons/verification-and-review-discipline.md#a-generalized-diagnostic-path-needs-a-loads-silently-oracle).

## D-telemetry-batch-durability: fsync once per batch, not per envelope

**Decision (2026-10-14):** The telemetry sink appends each `write_batch` through
`append_durable_jsonl_lines`: one tail repair, one write, one `fsync` per batch.

**Why:** The first repair fix made the sink repair and `fsync` per envelope. That
multiplied I/O on the hot path without improving the crash contract, which is at
most one torn tail, repaired by the next append, either way.

**Rejected:** per-envelope durability (the cost buys nothing the contract needs);
keeping the old sink (it had no repair at all, so a torn tail swallowed the next
record on `main`).

## D-jsonl-inventory: delete inbound.jsonl, keep runner-lifecycle and debug

**Decision (2026-10-14):** Asked "is there any JSONL we might not actually need?",
an inventory (`spawn:p7789`) and a coder proof (`spawn:p7793`) settled each file on
evidence:

| File | Verdict | Evidence |
|---|---|---|
| `inbound.jsonl` | Deleted | A write-ahead copy of `control_actions.jsonl` with no production reader; `inbound_seq` now comes from `control_actions.jsonl`'s `submission_seq` |
| `runner-lifecycle.jsonl` | Kept | Nine of its ten event types record facts nothing else holds at that moment: atexit before finalize, raw signals, exception type, identity-binding diagnostics |
| `debug.jsonl` | Kept | Opt-in diagnostics, never a correctness mechanism |
| `history.jsonl` legacy recognition, `sessions.jsonl` growth | Deferred to #569 | Removal needs an archive migration; growth needs a retention design |

**Rejected:** deleting `runner-lifecycle.jsonl` on the assumption that
`launch-boundary.jsonl`, `finalize-evidence.json` and `state.json` already hold its
events; the proof showed they do not.

## The torn-tail JSONL rule

The one phase-2 behavior change, with the three options put to the user and why
the newline became the commit marker, is recorded with the state decisions:
[one JSONL commit rule](state.md#one-jsonl-commit-rule-a-row-exists-when-its-newline-does-cleanup-phase-2-2026-10).

## Revisit when

- #588 merges: B7 is closed; drop it from the program's open list and close #546.
- A phase-3/4 behavior change is signed off: record it here before its phase starts.
- Someone needs spawn archiving again: design it fresh against current listings
  rather than restoring the deleted overlay.
- Pydantic drops or changes the call-time `extra=` override, or a config section
  stops passing through a Pydantic model: unknown-key detection for that section
  needs its own reporter (D-unknown-keys-from-pydantic).

## Sources

`work:pyright-strict` (decisions D8, D10, D13, D16–D18, D22, D24, D29–D41),
`spawn:p7749` (B7 root cause), `spawn:p7763` (phase-1 integration review),
`spawn:p7769` (review fixes), `spawn:p7765` (state#1 review), `spawn:p7771`
(torn-tail probe), `spawn:p7789`, `spawn:p7793` (JSONL inventory),
`spawn:p7794`, `spawn:p7795` (G2 reviews), `spawn:p7799`–`spawn:p7801`,
`spawn:p7810` (fixes and re-verification), `spawn:p7813` (polish), PR #546,
issue #569. D-unknown-keys-from-pydantic: `work:v010-followups` (D1, D4),
`spawn:p7912` (static audit), `spawn:p7913` (runtime differential),
`spawn:p7915` (review and prototype), `spawn:p7917` (implementation), PR #581,
issues #580 and #582.
