# Idle adapter architecture

Harness adapters report idle and background-work facts; core owns the notification decision so policies do not drift among hosts. The source checkout's `src/meridian/lib/idle/.context/CONTEXT.md` describes the contract, and [idle notifications](../decisions/idle-notifications.md) explains the policy and hosting choices.

## Running work and the push

Only work that will wake the session can hold a push. Treating uncertain work as certain can suppress a needed notification indefinitely. A known wake before cache expiry also makes a cache warning unnecessary.

## Known fragility

Adapters depend on upstream event names and payloads. If those change, idle transitions or background work can go unseen without an error; a busy signal that stops matching can instead cause a premature notice. Claude's permission notification is a separate upstream signal, not a reconstruction of prompt state. See [harness integration](../lessons/harness-integration.md).
