# Idle Notification Channels

## D-idle-leaf-packages — `lib/notify` and `lib/idle` are leaf services

Notification delivery is independent of idle policy, so an agent can notify with idle automation disabled. Harness adapters supply observations without depending on idle policy, avoiding a dependency cycle. Native hook delegation must still run when Meridian cannot parse or apply an event, so an existing user callback is not consumed.

## D-idle-notice-context — Reason-first titles and send-time cache countdown

A notice identifies its reason and session context. A cache countdown is computed when sent and marks estimates as such, so a provider heuristic is not presented as a guarantee. See [TTL detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd).

## D-notify-hook-events — Notifications are hook events delivered by routes

Notifications use the existing hook routing and command extension seam rather than a second channel and routing configuration system. A failed route is reported but does not stop the lifecycle. Invalid hook rows are dropped individually: falling back to defaults alone could undo an explicitly disabled route. A partially delivered notification reports a warning rather than failure so a caller does not resend already delivered text. See [hooks and plugins](../concepts/hooks-and-plugins.md) and `docs/hooks.md`.

Inside a spawn, `hooks` commands resolve the project from cwd while `notify` follows the inherited project context ([#178](https://github.com/haowjy/meridian-cli/issues/178)).

## D-notify-smtp-password-file — SMTP password from a file, env as leaky fallback

An SMTP password file is preferred to the environment: environment values are inherited by spawns and can end up in transcripts. `MERIDIAN_SECRET_*` cannot be used for in-session notification because launch strips it from harness children. See [child environment boundaries](../architecture/launch-system.md#child-env-boundaries).

## D-notify-from-spawns — Spawns may notify; idle automation stays primary-only

Any agent may deliberately notify; automatic idle notices run only for primaries.
