# Mars launch bundle

Mars resolves package content, routing and portable tool policy; Meridian applies the returned policy and composes its own launch prompt from the selected profile, loaded skills and per-run context. The bundle's precomposed prompt and scaffold slots are deliberately unread so there is one instruction surface. Per-run prompt files stay with Meridian.

Mars owns capability-to-native tool names; Meridian owns harness permission mechanisms. Native configuration is kept out of model instructions. See [bundle field consumption](../decisions/bundle-field-consumption.md), [tool permissions](../decisions/tool-permissions.md), and the source checkout's `src/meridian/lib/launch/bundle_adapter.py`.
