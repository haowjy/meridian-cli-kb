# Lessons: Running Unreleased Meridian Builds on a Developer Machine

Meridian's state is shared. Every project root under `~/.meridian/projects/`, and
every build that touches it, reads and writes the same files. A PR build that
changes a record shape can therefore break the installed CLI for every agent working
in that project. This page collects what the native-session-identity PRs (#520 and
its stacked PR 2) taught about running such builds safely.

## A PR build must never touch a real runtime root

**What happened (2026-09-25).** Three separate actions ran PR 1 code against the real
meridian-cli root:
- a "read-only" dogfood probe;
- a lane's checks, which used an `_MERIDIAN_RUNTIME_DIR` copy that the build
  silently ignored (next section);
- a lane's PR-build `spawn` calls, whose children wrote PR-shaped records.

The result:
- 10 `native_store` events in `sessions.jsonl`;
- 5 PR-shaped `spawns/*/state.json`;
- 23 PR-shaped rows in the disposable history index.

The installed 0.6.7 then crashed in `spawn list`, `session log`, `work` and search
with pydantic hydration errors (`SpawnRecord extra_forbidden`, `SessionRecord
harness_session_ids Field required`).

**Why it is worse than it looks.** Read commands write: they run the legacy import,
catch up the history index and repair state. A read-only probe is therefore not
read-only. At the time, every build also shared one `history-index/history.sqlite3`,
so after a PR-build command the installed CLI stayed broken until someone rebuilt
the index with it.

**What we changed:**
- **Bump the history-index `SCHEMA_VERSION` whenever a serialized record shape
  changes.** PR 1 went 3 → 4 → 5 across its phases. Since 2026-09-26 each schema
  has its own index file (`history-v<N>.sqlite3`), so builds no longer share the
  index at all ([decision](../decisions/history-storage.md#d-history-index-schema-namespace)).
  Authoritative records are still shared, and a PR-only field in them still crashes
  an older build.
- **Keep a tested rollback script** that strips PR-only fields from authoritative
  records and rebuilds the index with the older build. Test it on a copy before
  installing the PR build (the work item's `rollback-to-0.6.7.sh`).
- **Hard rule in every brief:** the PR build runs only against a copy. Run it as
  `env -i HOME=$HOME PATH=… TERM=dumb _MERIDIAN_RUNTIME_DIR=/tmp/<copy> <build>
  …`, with no PR-build `spawn` (its children would inherit the real root). Finish
  with a `find ~/.meridian ~/.claude ~/.codex … -newer <marker>` check, and
  attribute any change by content, because live sessions also write.

## A nested Meridian environment pins every command to the parent project

Commands run from inside a Meridian session inherit `MERIDIAN_PROJECT_DIR` and
`_MERIDIAN_DEPTH`. Two consequences bit during the incident:
- **The runtime override is ignored.** `_MERIDIAN_RUNTIME_DIR` applies only at the
  primary Meridian root (`ops/runtime.py`, `_runtime_dir_env_override_applies`). A
  process that inherited `_MERIDIAN_DEPTH=1` counts as nested, so its "copy" runs
  hit the real root.
- **Commands meant for another project run against the parent's.** An index rebuild
  meant for meridian-flow ran against meridian-cli's root instead.

**The rule:** any cross-project or copy-based check from inside a session starts
from a clean environment.
- **Another project:** `env -i HOME=$HOME PATH=$PATH TERM=dumb meridian -C
  <project> …`.
- **A copy:** `env -i … _MERIDIAN_RUNTIME_DIR=<copy> meridian …`, run from the
  project directory. Do not add `-C` here: `-C` unsets `_MERIDIAN_RUNTIME_DIR`, so
  the command would derive the real root again.

The [config precedence](../concepts/config-precedence.md) page explains how `-C`
works.

## Installing a test build can replace the user's `meridian`

**What happened.** A lane installed its test wheel with `UV_TOOL_DIR` set to a
throwaway directory but without `UV_TOOL_BIN_DIR`. uv relinked `~/.local/bin/meridian`
to the PR build. For 3.5 minutes every project's `meridian` ran PR code. In that
window:
- meridian-flow ran the one-time legacy import;
- a spawn in meridian-flow started under the PR runner and kept rewriting its
  record until it finished.

That spawn had to be repaired after it ended.

**The rule:** never `uv tool install` a test build. Unpack the wheel into scratch and
run it with its own interpreter, or set **both** `UV_TOOL_DIR` and `UV_TOOL_BIN_DIR`.

## Scratch data does not go in the synced docs repo

A probe brief said "your scratch" without a location. The probe put a 5.3 GB runtime
copy and a 205 MB search database under the work item's directory, which lives in
the auto-synced docs repo. The copy held private conversation data. It was caught
before an autosync commit.

**The rule:** runtime copies and probe databases live under `/tmp`. The work
directory keeps only scripts and timing tables. The docs checkout now also ignores
`work/*/scratch/` locally.

## Probing the upgrade path needs a genuinely old build and old state

The round-3 probe of PR #534 (2026-09-26) found two upgrade defects that every
earlier probe missed. A 0.6.7 background runner wedged after its last turn once the
new build upgraded the shared history index in place. Old Pi chats stayed unbound.
The first attempt at that probe was invalid, and the brief caused it.

- **`uvx --from meridian-cli==X` can run the PR build.** uvx reused the installed
  PR tool environment, because it had the same name and version (both said
  0.6.7). "OLD" was the new code: it wrote no `history.jsonl`, and its prune was
  vacuous. `uvx --isolated --from meridian-cli==X meridian` gets the real old build.
- **Prove "old" is old before trusting a result.** A version string is not proof,
  because an unreleased build keeps the last version. Check a behavior only the old
  build has: 0.6.7 writes `spawns/pN/history.jsonl` and rejects
  `--prune-runner-history`. The re-run made this its sanity gate.
- **Start from state the old build wrote.** An upgrade probe that creates its
  "old" data with the new build tests nothing. The same holds for fixtures: the
  overlap regression test seeds tables captured from a real 0.6.7 index.
- **Test the overlap, not just the before and after.** Users upgrade while old
  processes run. Start an old `--bg` spawn, run the new build while it is still
  running, then check that the old run finalizes and old commands still work. The
  in-place index upgrade only failed in that window.
- **Never edit a worktree that running lanes execute from.** The lead edited
  `pr3-base` while probe lanes ran `$NEW` from it. For a moment the file had a
  duplicate keyword argument, and one lane hit the `SyntaxError`. Make direct edits
  in a separate worktree and merge them.

Narrow probe briefs with enumerated checks completed; broad ones stopped short.
The upgrade failure mechanism and decision are recorded in
[native session identity lessons](native-session-identity.md#an-in-place-projection-upgrade-breaks-the-build-still-running).

## Pushing through a long pre-push gate

**Masked exit codes.** One push "succeeded" only because `git push … | tail`
reported `tail`'s exit code. The pre-push preflight took 5.5 minutes under
concurrent spawn load, and GitHub closed the idle SSH connection ("Connection to
github.com closed by remote host", exit 141).

**What works:**
- Push with a keepalive:
  `GIT_SSH_COMMAND="ssh -o ServerAliveInterval=20 -o ServerAliveCountMax=60"`.
- Check the exit code with `set -o pipefail`, or don't pipe at all.
- Confirm the remote head with `git ls-remote`.
- Never skip the gate with `--no-verify`.

The general rule, "record exit status", is in
[verification discipline](verification-and-review-discipline.md#record-exit-status-and-working-directory).

**Provenance:** `work:native-harness-session-identity`:
- `decision.md` entries of 2026-09-25: "INCIDENT", "INCIDENT (cont.)", "INCIDENT
  RESOLVED", "Push note", and "PR 2 kicked off" (the scratch near-miss);
- `design/pr1-foundation-restructure.md` risk 5;
- the entry of 2026-09-26 "Round-3 probe of every command" (invalid uvx lane p7201,
  re-run p7210, worktree-edit error), `evidence/probe3-upgrade.md` and
  `evidence/probe3-upgrade-rerun.md`.
