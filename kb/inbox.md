# The machine inbox

A message for no particular session waits in its machine's inbox (`meridian message inbox`, `meridian inbox`) until a session or a person takes it. The commands are in meridian-cli `docs/commands.md`, the `inbox.received` event in `docs/hooks.md`; the files, the write and claim sequences, who may do what and how a command reaches another machine are in the source checkout's `src/meridian/lib/inbox/.context/CONTEXT.md`, with its known limits in `FUTURE` beside it. This page holds the reasons that span those, for whoever changes the design. Why the inbox and the offer are Meridian's while the router is [meridian-comms](ecosystem.md#meridian-comms)'s is on that page.

## One inbox per machine, claimed by whoever it is for

The inbox is files under `~/.meridian/`, and that directory is per machine, so there is one inbox per machine and no merged view: "my inbox" across machines is `meridian inbox list --machine <name>` for each. Slack messages are filed on the machine that runs the meridian-comms bot, which runs no sessions. The owner chose that over a home machine per person, which cost a config value, an SSH call to file every message, and messages stuck in meridian-comms whenever that machine was off. What it gives up: a session starting on another machine isn't told about waiting Slack messages; they show in Slack.

A message leaves the waiting list only when someone takes it. Nothing delivers the text to a session: a router that handed the text over and claimed for a session would leave a wrong pick or a dead session holding the message, and untrusted text would land in a session that never asked for it. The inbox knows waiting and taken, nothing else. Whether a taken message was answered isn't tracked, because the answer goes to the sender through the reply address, so nothing releases a claim when its taker exits: a session that took a message, did the work and exited would have the message handed out a second time. `inbox list --taken` marks a claim whose session is gone, and a terminal releases it by hand. A taker is a chat ID, so a primary that `/clear`s needs a terminal to release what it held.

Nothing starts a session for a message. A start would spend the owner's money in a project the sender chose, on text the sender wrote, and Meridian can't start a primary without a terminal to hold it. A session that takes a message may spawn whatever its own permissions allow.

**Who sees what is scoped by project, by the owner's choice.** A session sees messages of its own project and messages with no project; a person at a terminal sees everything, and a primary may pass `--all`. A session's project comes from the runtime root its spawn record is in, never from a working directory or `-C`: a hook row runs in the home directory, and a daemon's directory would decide who sees every Slack message. A project ID Meridian can't use refuses the command instead of counting as no project, because a message with no project is shown to every session. Only a primary or a terminal may take, release or offer. That stops a child confused by something it read; it doesn't stop one that forges its environment, which nothing in Meridian stops today.

Agents type message IDs, so they are short counted numbers (`m412`), not random or time-based. The counter is a file that only goes up, read and written under the one inbox lock every change already holds. Taking the newest message as the floor instead would have kept the newest message, Slack text included, from ever being deleted, and reusing a number a stale link names could hand a repeated call another sender's message.

## A new message fires one event, from the user's config, detached

Every message enters through a Meridian command, so the command that wrote it fires `inbox.received` and no process watches the directory. Meridian ships no row for it: with nothing configured, a message waits. A push to the owner's phone for every message is one row away; the owner declined it as too many.

- **Rows come from `~/.meridian/config.toml` alone.** A later config layer's row replaces an earlier one of the same name, so a repository's `meridian.toml` could otherwise switch off the owner's router, and a broken project file would stop the event.
- **The rows run in a detached process.** meridian-comms runs `meridian message inbox` and reads its output to end of file while holding a thread's lock. Rows that held those pipes would keep the daemon waiting until the router finished, and the router waits on the daemon. The sender is answered as soon as the message is on disk, and the process records each row's result in a hook log beside the message, since its stderr goes nowhere.
- **The process drops the sender's session variables.** Otherwise the router would act as the session that filed the message.
- **A repeated filing fires the event again.** The first call may have died after writing the message and before starting the process, and nothing else would notice. So a consumer treats a message ID it has already handled as done.

Taken messages are deleted 30 days after they were taken, at most once a day, from that detached process and from any `meridian inbox` command. A primary launch would be the obvious trigger, but the bot's machine launches no primary, so Slack text there would never be deleted. A waiting message is never deleted.

## The waiting line appears only at a fresh launch

A primary's fresh launch says how many messages from the last 7 days wait for its project (`3 messages are waiting in this machine's inbox`). A continue or fork leaves the line out, so every continue composes the same prompt bytes and keeps the [prompt cache](research/prompt-cache-retention.md). Replaying the line from the launch would mean storing it in the launch snapshot, a new field an older build reading that row would refuse. The count is a hint: a stale or missing one is accepted, and a failure to count leaves the line out instead of failing the launch. Why it is a block of its own is in `src/meridian/lib/launch/.context/CONTEXT.md`.

## A party is its machine's ID

A message's sender and a claim's taker each name their machine by its [machine ID](machines.md#a-machines-id-and-its-name) as well as its name, and "same party" compares IDs. Compared by name, a session lost its own claim when its machine was renamed. The name stays beside the ID for display and for the reply address. Comparing a name against this machine's current and former names would have covered this machine, but the inbox also stores parties from other machines, whose former names this machine doesn't know. A party stored without an ID is this machine's when its name is this machine's current or a former name.

## An offer grants nothing and runs where the message is

`meridian inbox offer m412 <session>` sends one session a fixed notice:

```text
A message is waiting in this machine's inbox and looks like yours: m412, from <sender> (project <id>).
To claim it and print it, run: meridian inbox take m412
If it isn't yours, do nothing and it stays for someone else.
```

The notice is written in one place in Meridian and carries none of the message's text, so a router passes only a message ID and a session, and nothing a sender wrote reaches a command line or another session's prompt. The session claims with its own `take`; a wrong pick costs one turn on a warm session and leaves the message for someone else. The offer record exists for `inbox list` and grants nothing. The command sits on a line of its own, unquoted, because Claude's delivery drops backticks.

An offer is a guess, so it refuses what a guess shouldn't reach: a cold session, because the turn would re-read its whole context; a spawned child, because the notice says "claim it" and only a primary may take; and a session in another project, because a model's guess shouldn't carry a message past the scope the owner set. The notice is sent before the offer is recorded: recording first left "offered to" on file for a session that was never told when the send failed.

`offer` runs on the machine that holds the message, and the session may be on any saved machine: the notice then goes as an ordinary cross-machine `meridian message` and names the holder (`meridian inbox take m412 --machine <holder>`). `offer --machine <holder>` aimed at a session on a third machine is refused, because the holder would have to call that machine on the caller's behalf, and a saved machine never passes a request on. meridian-comms runs `offer` where the message is.

## Taking across machines

`take`, `release` and `offer` change files on the machine that holds the message. They cross although the [crossing list](machines.md#reading-messaging-and-taking-from-the-inbox-cross) is otherwise reading and messaging, because with the bot on a machine that runs no sessions, a session elsewhere has to take what is filed there; the owner approved it. A call to one of the three that was sent and got no whole answer fails with `maybe_delivered`, as `meridian message` does ([messaging](spawns/messaging.md)): it may have acted. Repeating it is safe because the taker is the caller's machine ID and chat ID, the same on every retry, a release of an unclaimed message succeeds, and a repeated offer only sends the notice again. A message ID is always bare and travels with `--machine`; `<machine>/m412` is not a reference, because that form names sessions.
