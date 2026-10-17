# lessons/process-containment — Keeping Harnesses From Outliving Their Runner

What building the start gate and guardian taught (identity-v2 PR 1, #633), the
launch lock and continue sweep (PR 2, #647), and stopping a black-box primary
through the gate on a signal (#623, PR #658). The
decision and mechanism are
[D-start-gate-guardian](../decisions/spawn-identity.md#d-start-gate-guardian). These
are the traps, most of which a passing POC missed.

---

## PDEATHSIG reaches one process, and wrappers hide the real one

`PR_SET_PDEATHSIG` signals the direct child when its parent dies, and nothing
else. Harness commands are often wrappers: Codex's is a Node script that re-execs
the native binary as its own child. After a runner SIGKILL, the wrapper died and
the native process kept running with PPID 1. Its scope said
`parent_death_linked: true` the whole time (`spawn:p8231`). The flag recorded that
a link had been set up, not that it covered anything.

The same is true of any per-PID death link: a pre-exec hook, a watchdog that
tracks one PID, a lock fd the harness inherits (Codex's re-exec dropped that
too). The unit to contain is the **process group**, and the thing that kills it
has to be inside the group or able to pin it.

## The killer should be in the group it kills

A group ID can be reused once every member is gone. A killer that sits outside
the group and sends SIGTERM, waits, then SIGKILL can hit a new group that took
the ID during the grace period. Putting the guardian *in* the harness's group
closes that window, because the group can't empty while the guardian lives. It
also means neither signal needs a birth check.

It must not be the harness's child, though. A harness that ends with a bare
`wait` would hang on it. The stub double-forks it (`( ( … ) & )`), so init
adopts it while it stays in the group and session.

## Without a pin, send one signal and stop

The one path that can't use a group is a primary whose stdin is a TTY and
whose stdout is piped. It must stay in the runner's group so Ctrl-C reaches it,
and that group can hold a shell pipeline's other members, such as `tee`. So the
scope records no pgid, and the guardian targets the harness PID.

A bare PID has no pin. The first design sent SIGTERM, slept the grace period,
and sent SIGKILL. During that sleep the harness can exit and its PID can be
reused, so the SIGKILL can reach an unrelated process (design review r3 F1). The
rule: **with no pin, send one SIGTERM at the moment of death and nothing later**.
The next sweep finishes the job with birth checks. The single SIGTERM could hit a
reused PID only if the harness had already exited, the runner had died before
writing `stop`, and the PID was reused before the guardian woke. That window is
microseconds.

## dash takes single-digit fds only

Linux `/bin/sh` is usually dash. In dash, `n<&m` accepts only single-digit fd
numbers, so the r3 stub's `eval "exec 9<&$fd $fd<&-"` failed with "Bad fd number"
as soon as the gate pipe landed on fd 10 or above. In a real runner it usually
does (reproduced on fd 13).

The fix reopens the pipe through the fd filesystem:
`[ "$fd" = 9 ] || exec 9</dev/fd/"$fd"`.
- On Linux, `/dev/fd` is `/proc/self/fd`, so this opens a new description of the
  same pipe.
- On macOS, opening `/dev/fd/N` is `dup(N)`.

Either way EOF still depends only on the runner's write end. dash can't close the
original multi-digit fd, so the harness inherits a read end of the gate pipe.
That is harmless, because EOF depends on writers and only the guardian reads.
The guard against fd 9 itself matters too: moving fd 9 onto itself closes it
(r3 F3).

Rejected on the way:
- `preexec_fn` dup2: `_posixsubprocess` orders `pass_fds` and `close_fds` so that
  fd 9 is either closed or must already be open in the parent;
- `posix_spawn` file actions, which rewrite every launcher;
- requiring bash;
- a named FIFO, which needs filesystem cleanup on crash.

## A guardian that holds harness stdio deadlocks the runner

The guardian inherits the stub's stdio, which is the PTY slave or the harness's
stdout pipe. While it holds them, the runner never sees EOF after the harness
exits. The PTY launcher waits for EOF before `waitpid`, and `stop` is written after
`waitpid`, so the runner blocks forever. Any stdio consumer that reads to EOF
does the same (`claude_ws.py`).

Fix: the guardian's first act is `exec </dev/null >/dev/null 2>&1`, before it
blocks on the gate. The runner's exit order stays EOF → `waitpid` → `stop`.

Check fd **targets** when you assert on this, not an exact fd table. While dash
blocks in `read <&9`, it dups the gate onto fd 0 and saves its own stdin on
fd 10. The invariant is: every guardian fd references the gate pipe or
`/dev/null`; none references harness stdio, a PTY slave, or a launcher `pass_fds`
entry (D58). An acceptance check that listed exact fds failed on a correct stub.

## A forked PTY child inherits whatever the runner had open

`subprocess.Popen(close_fds=True, pass_fds=…)` closes every other descriptor
in the child. The PTY launcher forks by hand, so it got no such guarantee. The
runner is multithreaded, and any of its threads can hold an inheritable
descriptor at fork time. The full gate test surfaced a real inherited Python
executable fd in the guardian, which a deliberately inheritable descriptor then
reproduced.

The fix: the PTY child closes every non-stdio fd except the gate's read end
before exec, matching `Popen`'s behavior. A hand-written fork/exec path must
reproduce what `Popen` does for you, and a test should start with a planted
inheritable fd.

## "Proven dead" must share the alive check's tolerance

PR 1 made primary scopes real and killable. Until PR 3 replaces guessed
liveness, a false `orphan_run` would kill a live TUI, so the reaper may terminate
a primary's scope only when the runner is proven dead. The first version of that
proof compared the recorded birth time with exact float equality. psutil derives
`create_time` from boot time, which moves when the wall clock steps, so a live
runner could read as dead and the live-TUI kill came back (D63 N1).

The rule: dead means the process is gone, is a zombie, or
`abs(create_time - birth) > 1.0`. That is the exact inverse of the alive check.
AccessDenied and unknown are neither alive nor dead. Two predicates over the same
observation must be defined from one probe and one tolerance, or the gap between
them becomes a decision nobody made.

The same review narrowed where the proof applies. Requiring it before *status
finalization* would leave an unprovable primary active forever, a ghost row the
user sees. It applies to **scope termination** only. Killing nothing is already
guaranteed there, and status can still settle.

## POCs must use production fd allocation and I/O topology

The r3 POCs passed. Both D55 defects were still in the stub:
- the POCs started with the gate pipe already at fd 9, so dash's fd limit never
  came up;
- none ran a PTY or a stdout consumer that reads to EOF, so the stdio deadlock
  couldn't happen.

The production runner gives the pipe whatever fd number is free (13 in the
repro) and always has one of those consumers. The PTY fd leak was the same shape:
the POC had no other threads holding descriptors.

A POC answers the question only under the conditions it ran in. When the
mechanism touches fds, reproduce how production allocates them and what reads
the other end. A POC for an fd-sensitive mechanism should run through the real
launcher, not a stand-in. PR 1's regression tests now cover both: fd 9 and fd 13,
and pipe and real `fork`/`login_tty` PTY topologies.

## Moving a blocking wait onto a thread moves its cleanup too

#623's fix (PR #658) moved the black-box `running.wait()` onto a daemon thread
so the main thread could poll a signal latch. Review (`spawn:p8399`, F1) found
what that cost. Before, a `KeyboardInterrupt` on the main thread unwound through
the PTY relay's `finally` blocks, restoring termios and closing the gate. After
the move it unwound only the poll loop. The relay thread kept the outer tty raw,
the spawn finalized as failed with the harness still alive, and the harness
died only when process exit closed the gate's fd.

When a wait moves off the main thread, the main thread's exception path has to
do explicitly what unwinding used to do implicitly: stop the child through the
gate, release the wait, join with a bound, then re-raise. Reproduce it with the
exception the main thread can actually get (SIGINT is unshielded in PTY mode),
not only with the signal the change was written for.

The same fix also showed that **`termios.error` is not an `OSError`**. The first
live pane-kill probe left a traceback that a fake-launcher test could not have
shown ([mechanism](../architecture/primary-terminal-ownership.md#a-dead-terminal-raises-termioserror-which-is-not-an-oserror)).

## Continue must lock the chat's current owner, not the named spawn

PR 2's first sweep locked the spawn ID the user typed. Both reviews passed it. An
edge-case probe the owner asked for afterwards found the hole: with p1 crashed and
continued as p2, a second `--continue p1` found p1's lock free and started p3 on
p2's live session. `--continue c1` was refused correctly, because chat c1's record
pointed at p2. The bug is sequential and easy to hit: an agent reusing an old ID,
or a user retyping one.

The unit that must not run twice is the **native session**, and the chat record
names its current owner. So the sweep resolves the named spawn to
`chat_record(chat).spawn_id` and locks that first. A live owner is refused, never
killed: killing it would silently stop work in another pane, and continue only
ever sweeps dead launches. A pruned owner is ignored, or a missing row would fail
every later continue.

The fix's re-check found the same hole one level down. A `--bg` continue moved
the chat's owner pointer only in the worker's `start_session`, after the parent
had returned, so a second continue in that gap still saw the old owner
(reproduced with SIGSTOP). The parent now publishes the pointer while it holds the
new spawn's lock, before the hand-off. **A guard is only as exact as the record it
reads: publish the record before the lock-holder lets anyone else proceed.**

## A lock fd survives the hand-off; a PID record doesn't

A `--bg` continue hands its runner role to a detached worker. The parent `dup`s the
locked descriptor into the worker through `pass_fds`, and the worker's first act is
to restore close-on-exec and record its own PID and birth time. Because the lock
lives on the open-file description, it stays held across the hand-off: 658 of 658
samples with the parent SIGKILLed at once. A liveness record keyed on the parent's
PID would have shown a dead runner in that window. Two rules made it work:
- don't use the reentrant `platform/locking.lock_file` for runner liveness: it is
  thread-reentrant and closes inherited handles on fork;
- set close-on-exec again immediately, so the harness never inherits the lock
  (the [M1 probe](../decisions/spawn-identity.md#why-a-runner-only-lock) showed
  why a harness-held lock lies).

## Related

- [D-spawn-identity-v2](../decisions/spawn-identity.md#d-spawn-identity-v2):
  why containment is the gate, guardian and sweep together.
- [Process scope](../architecture/process-scope.md): durable scopes and cleanup.
- [State design lessons](state-design-lessons.md#exact-liveness-beats-guessed-liveness):
  why liveness moved from PID guesses to a lock.
