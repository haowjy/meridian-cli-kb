# Completion drain coordination

The shared completion coordinator separates evidence from the policy that decides whether a spawn can finish. Resident and Pi profiles supply different policy and evidence; neither may publish success over an active turn. See the source checkout's `src/meridian/lib/streaming/.context/completion-and-entry-points.md` and [drain plans](drain-plans.md).

## The coordinator accepts evidence; it does not discover work

Persisted descendant assessment distinguishes ready, blocked and unknown. Unknown evidence cannot establish that no work remains. A fresh assessment is required before ordinary success because a child may have appeared since the previous read. Pi also needs private Bash and delivery evidence; its done nudge cannot bypass owed delivery ([Pi lifecycle](pi-lifecycle.md#idle-done-nudge)).

Terminal publication precedes asynchronous cleanup. A stuck cleanup cannot hold the result forever, and cleanup failure cannot replace the already published outcome. Startup recovery reconciles incomplete cleanup after a crash.
