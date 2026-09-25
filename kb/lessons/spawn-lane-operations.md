# Spawn Lane Operations

Lessons from running implementation and measurement lanes as `meridian spawn` children.
They concern how a lane can fail without the failure being visible in its result.

## Declared Backups Do Not Cover a Mid-Run Auth Failure

**What happened (2026-09-25):** two Codex-routed spawns, a `gpt-dev` fix lane and a
`prober` measurement lane, failed within 35 s. The error was `401 Unauthorized:
Incorrect API key provided: sk-svcac…`, from the Codex responses endpoint. Neither made
changes. Both profiles declared Claude backups, but no fallback happened.

**Why:** declared backups are prelaunch candidates. Mars picks one when the primary
harness or model is unavailable at resolution time
([model policies](../concepts/model-resolution/model-policies.md#fallback-participation)).
A credential rejected after launch is a run failure, not an unavailable harness.

**What worked:** relaunch each lane with an explicit `-m` naming the profile's declared
Claude backup (`-a gpt-dev -m opus`, `-a prober -m sonnet`), resuming with `--from`. The
key was the user's; the tech lead did not touch credentials and told the user instead.

**The lesson:** when several spawns on one harness fail fast with an auth error, stop
relaunching on that route. Pin a declared backup explicitly, and report the credential
problem to the user. Every spawn on that route fails until the key is fixed.

## A Claude-Routed Lane Can End Its Turn Before Its Gate Finishes

**What happened (2026-09-25):** fix lane D ran on the Claude harness (`gpt-dev -m
opus`). It committed its fixes, started the full preflight as a background command,
then ended its turn "waiting" for it. The spawn finished without the preflight result
and without a report. The tech lead reviewed the commits, ran the gates and wrote the
report.

**Why:** in this run, the background command finishing did not bring the headless lane
back for another turn. The spawn's result was whatever had been written before the turn
ended. The exact harness behavior was not investigated further.

**The lesson:** briefs for Claude-routed spawns should say to run gates and long tests
in the foreground, with a timeout that fits the gate, and to write the report before
ending the turn. When a spawn ends without a report, check its commits and rerun the
gates before merging.

**Provenance:** `work:native-harness-session-identity` (`decision.md` "Codex harness
auth failure → Claude backups" and "Recheck PASS WITH FIXES; V2 partial; lane D";
`evidence/pr2-fix-d-report.md`; spawns `p7149`, `p7150`, `p7151`).
