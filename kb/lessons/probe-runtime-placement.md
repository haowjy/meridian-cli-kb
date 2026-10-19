# Lessons: Probe Runtimes Live Under /tmp, Never in a Work Directory, and Never Touch Shared Installs or Config

**The rule:** a probe, POC or live check that needs a scratch `HOME`,
`MERIDIAN_HOME`, `_MERIDIAN_RUNTIME_DIR`, `CODEX_HOME`, XDG roots, copied
Meridian state, a Python environment, harness binaries or databases puts all of
it under a short `/tmp/<prefix>-XXXXXX` path and deletes it when done. The work
directory gets only notes, scripts, measurements and result summaries. Write
the `/tmp` path into the evidence file so the run can be traced, and say when
it was removed.

This applies to every agent that writes into `$MERIDIAN_CONTEXT_WORK_DIR`.
That directory is in the docs repo, which git-autosync commits and pushes
continuously.

## Why: what a runtime in a work directory does

**2026-10-15, 24 GB.** Probes for the v0.10 follow-ups copied whole runtimes
into `work/v010-followups/`: scratch homes, copied Meridian state, Python
environments, Claude native binaries, Codex caches, OpenCode databases with WAL
files, and event logs. The biggest were `probe-587-590` (about 14 GB, still
growing while active), `design-591` (2.2 GB), `wait-scope-sql-measure`
(1.8 GB), `scale-non-session-…` (1.5 GB), `session-fold-probe-588` (1.4 GB) and
`constant-time-audit` (1.2 GB). Autosync ran `git add` over the whole tree
every cycle, hit its timeout, was SIGKILLed, and left `.git/index.lock` behind.
After that, every later autosync failed silently: no work directory in the
docs repo synced for about four hours, and the lock came back after it was first
removed by hand. About 59,000 runtime files had already landed in nine unpushed
local commits.

Recovery deleted about 280,000 runtime files (12.7 GB) and kept the notes,
scripts and results. It squashed the nine unpushed commits and pushed one clean
commit. Commits that are already pushed can't be cleaned that cheaply.

**2026-09-25, near miss.** A probe brief said "your scratch" without a
location. The probe put a 5.3 GB runtime copy and a 205 MB search database
under its work item. The copy held private conversation data. It was caught
before an autosync commit.

The same mistake happened twice, a month apart, because nothing in the probe's
path said where scratch goes. A brief that says "scratch" without "under
`/tmp`" invites it.

## A scratch home does not isolate the harness binary

**2026-10-15, the user's global Codex replaced.** A probe for PR #621 ran Codex in
a fully isolated scratch root: its own `HOME`, `CODEX_HOME`, XDG roots and tmux
socket. Codex showed its built-in update prompt, the update was accepted, and it
replaced the user's real global install
(`~/.nvm/versions/node/…/lib/node_modules/@openai/codex`). A scratch `HOME` doesn't
cover that: the binary lives on the shared `PATH`, and the updater writes wherever
the binary was installed. The coder reinstalled 0.162.1 (`codex --version`
confirmed it), and the probe was repeated on a clean TUI.

**The rule:** a scratch Codex run disables update checks in its scratch
`CODEX_HOME`, with `check_for_update_on_startup = false` in `config.toml`
(the key is present in codex-cli 0.162.1). Nothing in a probe ever accepts an
update prompt. The same applies to any harness with a self-updater: a probe brief
that isolates the home but runs the shared binary must say how the updater is
switched off. If a probe does hit an update, check the installed version
afterwards and restore it before reporting.

Source: `work:resume-after-crash` `decisions.md` R9; `evidence/lane-a-live-terminal-ownership.md`
("Review-finding recheck"); `spawn:p8162`.

## A throwaway worktree shares the main repo's config

**2026-10-16, a fake identity for every checkout.** For #652, the smoke script
of `spawn:p8356` built a throwaway *worktree* and ran
`git config user.name/email 'Meridian Verification'` in it. A `git worktree`
has no config of its own: plain `git config` writes to the main repo's
`.git/config`. The identity landed in `~/gitrepos/meridian-cli/.git/config`,
so every worktree of that repo picked it up, and it authored the coder's commit.
The lead unset both keys and re-authored the commit. No other commit had used
the identity.

**The rule:** a smoke that needs a disposable repo with its own settings uses a
`git clone` (into `/tmp`, like any other probe runtime), not a worktree. If it
only needs an identity to commit, pass it per command with
`git -c user.name=… -c user.email=… commit` and leave the config untouched.
A probe brief that asks for a throwaway repo should say which of these to use.

Source: `work:changelog-misfile-check` `decisions.md` C3.

## A scratch home does not isolate the project

**2026-10-18, a probe work item in the shared docs repo.** A review probe for
#687 (`spawn:p8648`) set a temporary home but inherited the spawn's
`MERIDIAN_PROJECT_DIR`, which pointed at the real meridian-cli checkout. Its
`work start probe-item` and `work done probe-item` ran against that project.
The project's `git-autosync` hook fired on both events and pushed two commits
to the shared docs repo, creating and then archiving the item. The probe
deleted it, and the deletion reached the repo with the next autosync, so the
net change was zero. The second probe ran under `env -i` with a temporary
`HOME` and touched nothing real.

**The rule:** a probe that runs `meridian` commands which mutate project state
(`work`, `spawn`, `hooks run`) starts from `env -i`, or unsets every inherited
`MERIDIAN_*` variable, and points `HOME` and `MERIDIAN_HOME` at `/tmp`.
`MERIDIAN_PROJECT_DIR` picks the project; the chat, spawn and depth variables
attach the probe's actions to the parent's session. A scratch home alone leaves
the inherited project root in charge, and that project's hooks fire on
whatever the probe does.

Source: `work:hook-event-names` `decisions.md` (wave 2 notes).

## What is in place, and what isn't

- **`.gitignore` backstops** in the docs repo cover the known runtime shapes:
  `work/**/home/`, `spawns/`, `projects/`, `.meridian/`, `locks/`, `runs/`,
  `xdg-*`, virtualenvs, `*.db*`, Claude `versions/`, and nested git repos.
  They are a safety net. A new layout gets through them.
- **meridian-dev-workflow#7** (open): the `/probe` and `/poc` skills should
  require `/tmp` scratch for homes, copied state, environments, databases, logs
  and artifacts. Once that lands, it is the place probe agents actually read.
  Until then, **a lead writing a probe brief must state the `/tmp` location
  explicitly.**
- **meridian-cli #538** (open): a SIGKILLed autosync leaves `index.lock` and
  sync stops silently. The proposed fixes are to detect a stale lock and to
  refuse an oversized pending tree with a clear diagnostic. Until then, a docs
  repo that stops syncing means checking for `.git/index.lock` and a large
  untracked tree.

Good practice from the native-terminal probes (`work:native-terminal-primaries`
evidence): each run used a fresh `/tmp/ntf-*` root with only auth files copied
in, unset inherited `_MERIDIAN_DEPTH` / `MERIDIAN_SPAWN_ID` / `MERIDIAN_CHAT_ID`
/ `MERIDIAN_RUN_ID`, ran a dedicated tmux socket, and removed the root at the
end. See also
[dogfooding PR builds](dogfooding-pr-builds.md#a-pr-build-must-never-touch-a-real-runtime-root)
for keeping unreleased builds off real runtime roots.

**Provenance:** `chat:c7524` (owner: "yes, clean it up... clean up old probes
from docs repo"); docs-repo `.gitignore` comments; `work:native-harness-session-identity`
`decision.md` 2026-09-25 "PR 2 kicked off" (the near miss).
