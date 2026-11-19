# Context sync

Every git context (the KB, the work directory, any `[context.<name>]` with `source = "git"`) resolves to a clone under `~/.meridian/git/`. `meridian-sync`, a Rust daemon run as a user service, keeps each clone in sync with its remote: it commits local edits, fetches, integrates and pushes. It replaced the `git-autosync` hook in v0.24.0. User guidance: meridian-cli `docs/commands.md#context-sync` and `docs/configuration/context.md`. The two halves of the code: `crates/meridian-sync/AGENTS.md` (the daemon) and `src/meridian/lib/autosync/AGENTS.md` (Python's side).

## Why it is a daemon

The hook ran only on `spawn.finished`, `work.activated` and `work.done`, so an edit made in an editor, or a commit pushed from another machine, waited for the next spawn to finish. It also needed a `[[hooks]]` row per remote beside the context table, and a context without a row never synced. A daemon watching each clone syncs edits from anywhere. `autosync` is a context setting, on by default for every managed clone, so a context syncs without extra configuration. A hand edit to `meridian.toml` that adds a context takes effect at the next `meridian` command in that project, the first time anything uses the context.

The service installs itself the first time a human runs `meridian` in a project with a git context, and never from an agent session, a dev install or a non-default `MERIDIAN_HOME`, so tests and checkouts run in place don't take over the user's service. Turning it off for the machine is `meridian sync service uninstall`, which leaves a `service.off` marker so the next launch doesn't reinstall it.

## Every incident was a silent stop

| When | What happened | How long nobody saw it |
|---|---|---|
| [#538](https://github.com/haowjy/meridian-cli/issues/538) | The hook SIGKILLed a slow `git add`, which left `index.lock`; every later sync failed and reported a successful skip | 37 h |
| 2026-10-14 | A nested repository with no commit inside the docs clone made `git add` fail | silent |
| 2026-10-15 | A 24 GB directory of probe output hung `git add`; the timeout kill left `index.lock` | silent |
| 2026-10-17 | A spawn whose task dir was an eval fixture inside the docs clone (a directory without its own `.git`) ran `git checkout -b`; the hook kept committing on that branch and pushed nothing | 16 h |

None was data loss, and none would have been caught by a guard on what gets committed. Each was a sync that stopped where no one looked. So the design keeps three rules and nothing else:

- **Loud failure.** A stopped clone shows on the line `meridian` prints before the harness starts, in `meridian context`, `meridian sync status` and `meridian doctor`, and once through `meridian notify`. "Daemon not running" shows the same way.
- **Stay on the default branch.** A clone on a branch that was never pushed, whose commits the default branch can fast-forward to, is repaired by moving the default branch to it without a checkout. Anything else stops and names the branch. A pushed branch is someone's deliberate work, so it is never folded in.
- **Never SIGKILL git.** A git past its timeout gets SIGTERM, so it removes its own `index.lock`. A lock left by a crash or reboot isn't cleaned up: it stops the clone as `index_locked`, and the fix text names the file to delete.

**The guards that were cut**: mass-deletion and size limits, an empty-index check, a conflict-marker scan, nested-repository exclusion and stale-`index.lock` removal. Each was designed against a failure with no recorded incident, and each failure it would catch ends as a stopped sync, which loud failure shows. A mass deletion or conflict markers get committed as they are; git history keeps the previous state. The docs and KB repos' allowlist `.gitignore` files are what keep nested repositories and probe output out of the index now. Add a guard back when an incident shows a failure that loud failure doesn't surface.

## Integrating off the worktree

A pass merges with `git merge-tree --write-tree` and `commit-tree`, then moves the branch with `merge --ff-only`; only that fast-forward writes the worktree. A merge in the worktree can lose a write an agent makes during it, and a rebase can stop partway through replay and leave the clone wedged; merge-tree costs about the same and has neither problem. A clone that is only behind fast-forwards without a merge commit, or two machines would trade merge commits forever.

A conflict stops the clone with its worktree untouched. The daemon keeps committing locally and stops pushing until a human or agent merges `origin/<branch>`. While that merge is open the daemon stays out of the clone, as it does for any merge, rebase or bisect it finds in progress.

## How agents learn of a conflict

The daemon writes `SYNC-CONFLICT.md` into each context directory of a conflicted clone, and each context's `AGENTS.md` has one permanent line telling an agent that finds it to run `meridian sync now <context>`, resolve, and rerun. The file is never committed: the daemon excludes it by pathspec, because the docs clone's allowlist `.gitignore` re-includes `*.md` over any exclude file. A note about sync state in every spawn's prompt was rejected as a cost every launch pays for a rare event. Other stops (off-branch, a failing fetch) reach only the human surfaces.

## One judge, one file each way

Python and the daemon share two kinds of file under `~/.meridian/autosync/`, each with one writer:

- **Root records** (`roots/<hash>.json`), one per project checkout, written by Python with that checkout's resolved git contexts. Python keeps all config resolution, so Rust never reads TOML and two implementations of config layering can't drift. A record rewritten with a new `written_at` asks the daemon for a pass; that is all `meridian sync now` and a launch do. An earlier design had Python project one registry of every project and the daemon call back into Python to refresh it; that needed input watching, schema negotiation and staleness rules, and one record per checkout needs none.
- **`state.json`**, written only by the daemon's main thread, which judges every clone's state from its workers' events. Python renders it and reads no clock of its own. An earlier design had per-clone status files, a daemon file beside them and timing rules in the Python reader; each review found another place where the three disagreed.

The daemon judges durations on a monotonic clock that stops during suspend, so a laptop that slept overnight doesn't wake to an alarm. It writes each clone's running timers into `state.json`, so a restart neither forgets a stop nor resets the 15-minute rules, and a daemon that crash-loops still reaches a stop. A main thread that hangs while it holds the lock isn't detected: every surface keeps showing the last state it wrote, and `meridian sync status` shows the stale `alive_at`.

## No spawns or projects inside a clone

The 2026-10-17 stray branch came from a spawn working inside the docs clone. So a spawn whose task dir is inside a managed clone is refused, wherever the task dir came from, as is setting such a task dir or creating a project there; a nested repository inside a clone counts as inside it, since nested repositories caused the 2026-10-14 stop. The one exception is the caller's directory: agents `cd` into KB and docs clones routinely, so a task dir inferred from the cwd falls back to the project root instead of refusing the spawn. A human's primary launch inside a clone is allowed, with a warning that its spawns will be refused. A clone at a `[git."<url>"].path` override is the user's own checkout and isn't covered; autosync is off there by default.

## Rust, inside meridian-cli

The daemon is in Rust as the first piece of the planned [Rust rewrite](codebase/rust-rewrite.md). It lives in meridian-cli at `crates/meridian-sync/` rather than its own repository, because every change to the seam would otherwise take two PRs and a release. It ships as its own binary wheel on PyPI, released at meridian-cli's version and pinned exactly, so meridian-cli stays a pure Python wheel; the cost is that a source checkout's `uv sync` needs a Rust toolchain. At the rewrite it becomes part of the one `meridian` binary, and the separate wheel and the root-record seam go away. No existing autosync tool was a base: none integrates off the worktree.

## Known gaps

- `meridian sync status` in a project that hasn't registered yet says stopped instead of telling you to run `meridian sync now` ([#1004](https://github.com/haowjy/meridian-cli/issues/1004)).
- Each clone takes its own inotify instance, against a default limit of 128 per user shared with other programs; past it, clones fall back to polling ([#1005](https://github.com/haowjy/meridian-cli/issues/1005)).
- The launchd service on macOS is untested ([#1006](https://github.com/haowjy/meridian-cli/issues/1006)).
- The service starts outside the login shell, so an ssh agent known only to the shell is missing and fetches fail; the fix is in `docs/configuration/context.md`.
