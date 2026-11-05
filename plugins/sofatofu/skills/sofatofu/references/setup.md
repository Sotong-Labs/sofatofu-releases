# Setup and sign-in

Each Sofatofu connection gives access to one workspace, which the user chooses when approving access. If a command or tool fails, follow the instructions in its `message`.

If neither Sofatofu tools nor the `sofatofu` command are available, ask the user to set up Sofatofu at [sofatofu.com](https://sofatofu.com).

## Check the connection

- If you have the sofatofu tools, call `auth_whoami`.
- If you have the `sofatofu` command, run `sofatofu auth whoami`.

A network or service error does not mean the user is signed out, so do not connect again or install anything because of it.

When the check returns the user and the workspace, make sure they are the ones the task needs. Then read [Sofatofu](../SKILL.md) before the first operation.

## Sign in

If the check tells you to sign in, connect with what you have, then check again. A new connection needs the user to sign in, approve access and choose the workspace in the browser. If the task names a workspace, tell the user which one to choose.

- **Connector:** if the app lists the official Sofatofu connector in its connector directory or connection controls, offer it there. If not, ask the user to add a custom connector with the URL `https://sofatofu.com/mcp`. In Claude, offer the [official Sofatofu connector](https://claude.ai/directory/sofatofu).
- **CLI:** run `sofatofu auth login`. Until the user approves access, it returns a link (`verification_uri_complete`) and a code (`user_code`). Give the user the link and code once. Ask them to open the link, sign in, choose a workspace, and approve access. Run the same command again to check. If you get the same link again, approval is still pending, so do not ask the user again. You are signed in when the command returns the user and the workspace.

## Switch workspace

- **Connector:** ask the user to disconnect Sofatofu in the app, connect again, and choose the other workspace.
- **CLI:** run `sofatofu auth logout`, then `sofatofu auth login`, and ask the user to choose the other workspace.

## Update

- **Connector:** there's nothing to update. The connector always serves the current operations.
- **CLI:** ask the user to run `sofatofu update`. It updates the CLI and the installed copies of this skill. Then read the updated `SKILL.md`.
- **Skill only:** ask the user to update the skill. Then read the updated `SKILL.md`.

## Sign out

- **Connector:** ask the user to revoke the connection in their account settings at `https://sofatofu.com/portal/account`.
- **CLI:** `sofatofu auth logout` asks the server to end the connection, then removes the local credential and any saved approval request. It succeeds even if there was no credential. Ask the user before you use `--force`.

## Uninstall

Uninstall only when the user asks you to remove Sofatofu.

- **Connector:** ask the user to remove the connector in the app and revoke the connection in their account settings.
- **CLI:** `sofatofu uninstall` signs out and removes the CLI, its local configuration, and skill copies from the home folder. It returns the removed paths as JSON. Show the user any `warning`. Project skill copies remain. Remove those only if the user asks.
- **Skill only:** ask the user to remove the skill.
