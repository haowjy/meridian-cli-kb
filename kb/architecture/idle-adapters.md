# Idle adapter architecture

Harness adapters report idle and background-work facts; core owns the notification decision so policies do not drift among hosts. The source checkout's `src/meridian/lib/idle/.context/CONTEXT.md` describes the contract, and [idle notifications](../decisions/idle-notifications.md) explains the policy and hosting choices.

## Running work and the push

Only work that will wake the session can hold a push. Treating uncertain work as certain can suppress a needed notification indefinitely. A known wake before cache expiry also makes a cache warning unnecessary.

## Known fragility

Adapters depend on upstream event names and payloads. If those change, idle transitions or background work can go unseen without an error; a busy signal that stops matching can instead cause a premature notice. Claude's permission notification is a separate upstream signal, not a reconstruction of prompt state. See [harness integration](../lessons/harness-integration.md).

- Codex detects busy from the text `Working (`; a change can make notices fire during a turn.
- Claude forwards `classic.Stop` background tasks at `turn.complete`. A missing hook or renamed field silently loses the hold; a renamed task type can turn a hold into a delay. If `classic.Notification` stops reporting `permission_prompt`, the separate permission notice also disappears silently.
- Return-event drift can leave an idle stretch open and suppress later notices; turn-end drift can prevent the sensor from arming at all.
- Reopen must close the old stretch before the harness starts, using the same harness identity and home as its idle commands. Otherwise Claude or Pi can recover the old timer and notify on its former schedule without an error.
