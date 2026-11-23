# Antigravity (`agy`)

Meridian hosts `agy` through a plugin and never runs it. The user starts Google Antigravity's own TUI, by hand or with the command `meridian agy` prints, and the plugin Mars installs into `.agy/` (hooks, an MCP server, Meridian's skills) registers the session, carries Meridian's guidance and messages in, and lets the session spawn agents on other harnesses. Antigravity's terms forbid running `agy acp` as a backend worker under a personal Google sign-in, which is the sign-in in use, so spawning `agy` waits until the terms change ([#1016](https://github.com/haowjy/meridian-cli/issues/1016)). Mars carries the same fact: routing never picks `agy`, and Mars writes `.agy` agent files and a host's provided content even when Meridian manages routing ([targets](../mars/package-management.md#targets)).

What the session can and can't do: the source checkout's `docs/antigravity.md`. How the adapter is built (plugin-hosted harnesses, the hook, the server, the watcher): `src/meridian/lib/harness/AGENTS.md`, "`agy/` is hosted by its plugin". How `.agy` is laid out and how `agy` treats hooks and MCP servers: mars-agents `docs/config/mcp-and-hooks.md`, "Antigravity".

## Waking an idle session

`agy` offers no way for an outside process to put input into a running conversation, and Meridian doesn't type into the user's terminal. The one thing that starts a model call with nothing typed is a background command the agent started: when it exits, `agy` wakes the agent with its exit code and stdout. So the plugin's `Stop` hook asks the model to keep one such command running, `meridian harness-watch agy`, which exits when a message arrives for an idle session. A message to a session mid-turn is not a reason to wake: it reaches the model at the turn's next model call through `PreInvocation`.

The watcher is the one wake mechanism: it also exits for a `wake/<conversation>` file, so anything else that must wake the session (keepwarm pings, if `agy` gets them) uses the same path instead of a second thing the model must keep running. Claude's idle mod likewise injects every prompt through one mechanism.

Its costs: every session shows `1 task(s)` for its life, the first turn and each wake end with one extra continuation to restart the watcher, and starting it is a shell command that `agy` may ask to approve unless the user allows `command(meridian)`. Esc leaves the session marked busy (the plugin's `PreInvocation` guard sets the mark, `Stop` clears it, and Esc fires no `Stop`) until the next typed turn, so a message sent after a cancel waits. Idle notices for `agy` would read the same mark and inherit the gap, so finding its cause comes before them.

## Idle notifications and compaction: direction

Not built. Idle notices for `agy` (push, cache warning) come next, on per-harness `[harness.agy.idle]` config like Claude's. The owner chose to bring the state reminder and its `compact-now` reply to `agy` too, `compact-now` above all ([state reminder](../spawns/idle-notifications.md#state-reminder-claude)); today both run on Claude only. Whether `agy` gets keepwarm pings is undecided. Two facts shape this:

- **Nothing can compact a running conversation for the session.** Only a `/compact` the user types compacts, so a reminder could update the work item but nothing could compact afterwards. `compact-now` needs a compact hook reply or a compact tool from Google ([#1049](https://github.com/haowjy/meridian-cli/issues/1049)).
- **The prompt cache has no window to aim at.** Google publishes no TTL, its server changes alone have moved hit rates sharply, and hits differ by effort level ([Antigravity cache](../research/prompt-cache-retention.md#antigravity)), so any built-in TTL would be wrong. And the cache warning goes out `idle.warn_minutes` (global, 10 by default) before the TTL ends, so a TTL of 10 minutes or less gets no warning at all.

## What `agy` does that Meridian builds on

Probed on the 1.5.15 TUI unless a version is given; recheck after an upgrade. Hook-contract surprises Mars renders around (the `{}` reply, `SessionStart` timing) are under [portable hooks](../mars/package-hooks.md#harness-behavior-to-recheck-after-upgrades).

- **A background command's exit wakes the idle agent** about 35 ms later, after idle gaps from 9 seconds to 62 minutes. The command is a `run_command` with `WaitMsBeforeAsync: 500` and `IsDaemon: false`; the agent sees a `SYSTEM_MESSAGE` with the exit code and stdout. The user can type turns while it runs; the command keeps running and still wakes afterwards. `agy` neither capped nor reaped a watcher at 62 minutes; longer is untested. A command killed from outside does not wake the agent. Esc and `/new` leave background commands running, quitting kills them, and a resumed conversation is told its background tasks stopped. `IsDaemon: true` is untested.
- **`PreInvocation` can't tell a typed prompt from a wake.** Both carry `invocationNum: 0`, the last typed `lastUserInput`, and no source field. The transcript can: a typed prompt appends `USER_INPUT`, a wake `SYSTEM_MESSAGE`.
- **`Stop.fullyIdle` is false while any background command runs**, so with a watcher up it is always false and says nothing about the turn.
- **A `Stop` reply of `decision: continue` works.** The model sees `Stop hook blocked termination: <reason>` as a `SYSTEM_MESSAGE`, which the TUI does not show the user, and acts on it. `executionNum` is 1 on the `Stop` ending that continuation; `continuationActive` stays false.
- **Esc fires no `Stop`, and `PostInvocation` doesn't fire on a cancelled call.** The transcript marks a cancel only by a missing step number. No `USER_CANCELED` termination reason has been seen.
- **The terminal sandbox (`agy --mode auto`) hides `~/.meridian` and has its own PID namespace**: `~/.meridian` reads as missing, `~/.gemini` is read-only, `/tmp` is visible, and `agy` is PID 1 inside it. A `run_command` with `BypassSandbox: true` runs outside it, which is why the watcher is started that way. The plugin's MCP server already runs outside it.
- **The plugin's MCP tool is unusable on a conversation's whole first turn**, even after a shell call in that turn, though the server process is up within 2 seconds of launch; from the second prompt the model can call it. So the watcher starts through the shell tool, which works on turn one.
- **A native subagent's hooks carry its own conversation ID**, in both `ANTIGRAVITY_CONVERSATION_ID` and the payload.
- **Nothing but a typed `/compact` compacts.** A hook reply injecting `userMessage: "/compact"` (from `PreInvocation` or `PostInvocation`, with any `terminationBehavior`) reaches the model as ordinary text, because slash commands expand only for input the user typed. No model tool compacts. `AGY_CLI_AUTO_COMPACT_OVERRIDE` sets a threshold at launch only, and one below the prompt's base size breaks the turn. A real compaction shows `Compacting conversation…` and adds a `CHECKPOINT` step.
- **Plugin sidecars are read-only UI panes** in the Remote Control view and can't drive a conversation.

## Probing `agy`

- Never run `agy acp` or `agy -p` on Jimmy's personal sign-in. Probe through the TUI in a private tmux server (`tmux -L <spawn id>`), with few, tiny typed prompts.
- Run every model turn on `gemini-3.8-flash-low`: launch with `agy --model gemini-3.8-flash-low` and check the TUI shows it. This rule makes the cache of Neon (`gemini-4-neon`, the model in real use) unmeasurable by a probe; measuring it needs Jimmy's go-ahead.
- Write nothing into `~/.gemini/config/`, `~/.gemini/antigravity-cli/settings.json` or a real repository. A TUI launch adds its own `trustedWorkspaces` entry. Jimmy's `settings.json` allows `command(meridian)`, and `--settings` rules merge with it rather than replace it, so a probe can't observe the approval prompt a user without that rule gets.
- A scratch project's `.agy/` loads only once `.agents/plugins.json` points at it, and the scratch tree needed `git init`.
- No sign-ins, no `--remote-control`, and never print tokens.
- Most Mars checks need no prompt: `agy plugin validate --strict` on `.agy`, and the files a sync writes.
