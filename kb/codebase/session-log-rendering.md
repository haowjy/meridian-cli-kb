# Session log rendering

Tool calls are normalized before rendering so the display does not need harness-specific parsing. Cleaning content before truncation makes the preview limit apply to useful text rather than harness wrappers.

Readable versus raw output and truncated versus full content are independent choices: a reader can expand content without receiving debug formatting. See [session operations](session-operations.md) and [session browse decisions](../decisions/session-browse.md#d-browse-transcript-on-demand).

