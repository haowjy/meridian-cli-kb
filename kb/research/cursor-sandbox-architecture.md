# Cursor Sandbox and Approval

A Cursor probe found that sandboxed workspace writes could succeed while deletion was approval-gated. Cursor's binary sandbox cannot precisely represent every Meridian sandbox policy, and its default approval behavior has no explicit confirm flag. Meridian reports approximations; see [tool permissions](../architecture/tool-permissions.md).

Mars does not materialize Cursor's undocumented `.cursor/sandbox.json`: doing so would conflict with user configuration, couple it to an internal format, and require cleanup ownership. See [mars-agents #91](https://github.com/haowjy/mars-agents/issues/91).
