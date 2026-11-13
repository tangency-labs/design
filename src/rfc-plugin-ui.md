# Plugin UI and workspaces

| | |
|---|---|
| Status | Draft, open for comment |

This design is already in the SolidCraft repository as RFC 0015, so I am not copying it here. Read and comment on it there:

[RFC 0015: Plugin panels, workspaces and viewport content](https://github.com/tangency-labs/solidcraft/blob/main/docs/rfcs/0015-plugin-ui-workspaces.md)

The one idea behind it: **data in, commands out.** A plugin sends the host data (a panel layout, rows, polylines). The host draws it with its own widgets and theme. When the user acts on it, the host calls one of the plugin's entries as an ordinary command.
