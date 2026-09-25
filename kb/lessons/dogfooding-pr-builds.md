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

**Why it is worse than it looks.** Any build rewrites the shared
`history-index/history.sqlite3` on read commands. A read-only probe is therefore
not read-only. After a PR-build command, the installed CLI stays broken until
someone rebuilds the index with it.

**What we changed:**
- **Bump the history-index `SCHEMA_VERSION` whenever a serialized record shape
  changes.** PR 1 went 3 → 4 → 5 across its phases. An older build then refuses a
  newer index with a typed "incompatible, run `session index rebuild
  --metadata-only`" error instead of a hydration crash. The bump makes the failure
  legible; it does not make sharing safe.
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
- `design/pr1-foundation-restructure.md` risk 5.
