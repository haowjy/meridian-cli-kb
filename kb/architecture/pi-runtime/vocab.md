# Pi runtime vocabulary

**Tracked Bash** is managed execution that can block quiescence; **detached Bash** no longer blocks it but remains owned by the live runtime until it exits or the runtime shuts down. A cold runtime cannot reclaim that ownership from a PID.

A **Bash origin link** associates a spawn with its launching Bash task and can transfer the launcher's result obligation. It does not establish child membership: canonical parent relationships do that.

An **implicit-wait notification** carries unattended child or Bash results through Pi's native queue. Queueing, native admission and Python's observation are distinct facts. See [coordination](coordination.md) and the source checkout's `src/meridian/pi_runtime/.context/delivery-contract.md`.
