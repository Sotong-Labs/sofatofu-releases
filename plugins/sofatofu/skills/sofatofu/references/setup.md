# Setup and sign-in

This page covers how to connect an agent to Sofatofu, and how to switch workspace, update, sign out and uninstall. Each connection reaches one workspace, which the user chooses when they approve access. When a command or tool on this page fails, follow its `message`.

## Start

Before you install anything, check for a connection:

- If you have the sofatofu tools, call `auth_whoami`.
- If you can run commands and have the `sofatofu` command, run `sofatofu auth whoami`.

A network or service error does not mean the user is signed out, so do not install or connect again because of it.

If you have no connection, or the check tells you to sign in, connect with what your app supports, then check again:

- **Connector:** if the app lists the official Sofatofu connector in its connector directory or connection controls, offer it there. If not, ask the user to add a custom connector with the URL `https://sofatofu.com/mcp`. In Claude, [this link](https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=Sofatofu&connectorUrl=https%3A%2F%2Fsofatofu.com%2Fmcp) fills in the name and the URL, and the user confirms.
- **CLI:** if you can run commands, [set up the CLI](#set-up-the-cli).

A new connection needs the user to sign in, approve access and choose the workspace in the browser. If the task names a workspace, tell the user which one to choose.

When the check returns the user and the workspace, make sure they are the ones the task needs. Then read [Sofatofu](../SKILL.md) before the first operation.

## Set up the CLI

### Install the CLI

Install the CLI without asking the user first. Download the install script to a temporary file, then run that file. Never pipe a download straight into a shell.

- **macOS, Linux or WSL:** download `https://sofatofu.com/install.sh` and run it with `sh`.
- **Windows PowerShell:** download `https://sofatofu.com/install.ps1` and run it with PowerShell.

Run later commands with the path from the script's `Installed` line. Give the user any PATH advice it prints, and don't act on it yourself.

### Set up the machine

Run `sofatofu init`. It installs this skill for the agents it finds, then signs in as [Sign in](#sign-in) describes. It installs the skill under the home folder by default. Add `--project` in the project directory to install it there instead. This changes the skill's install location, not the sign-in or workspace. Add `--agent <name>` to choose an agent.

When `init` has signed in, ask the user to start a new agent session so the installed skill loads. To call operations, read [Use the CLI](cli.md).

`init` sends setup data to Sofatofu, such as the setup result, the CLI version and the operating system. The [privacy page](https://sofatofu.com/privacy) describes this data. If the user doesn't want it sent, tell them to set `SOFATOFU_NO_TELEMETRY=1`.

### Sign in

`sofatofu init` and `sofatofu auth login` sign in the same way. Until the user approves access, they return a link (`verification_uri_complete`) and a code (`user_code`). Give them to the user once, then run the same command again to check. If you get the same link again, approval is still pending, so do not ask the user again. You are signed in when the command returns the user and the workspace.

## Install only the skill

Run `npx skills add https://sofatofu.com`. That installs this skill and nothing else. To use Sofatofu, the user still adds the connector or you install the CLI, as above.

## Switch workspace

- **Connector:** ask the user to disconnect Sofatofu in the app, connect again, and choose the other workspace.
- **CLI:** run `sofatofu auth logout`, then `sofatofu auth login`, and ask the user to choose the other workspace.

## Update

- **Connector:** there's nothing to update. The connector always serves the current operations.
- **CLI:** `sofatofu update` installs the latest CLI release with the method it was first installed with, and refreshes the installed copies of this skill. It doesn't install an agent, ask for input or sign in. Use it instead of `npx skills update`, then read the updated `SKILL.md`.
- **Skill only:** run `npx skills update sofatofu`, then read the updated `SKILL.md`.

## Sign out

- **Connector:** ask the user to revoke the connection in their account settings at `https://sofatofu.com/portal/account`.
- **CLI:** `sofatofu auth logout` asks the server to end the connection, then removes the local credential and any saved approval request. It succeeds even if there was no credential. Ask the user before you use `--force`.

## Uninstall

Uninstall only when the user asks you to remove Sofatofu.

- **Connector:** ask the user to remove the connector in the app and revoke the connection in their account settings.
- **CLI:** `sofatofu uninstall` signs out, removes the copies of this skill under the home folder, removes its local configuration and removes the binary. It prints the removed paths as JSON. Give the user any `warning` field it returns. It keeps the copies that `init --project` wrote in a project. Delete those from the project if the user wants them gone.
- **Skill only:** run `npx skills remove sofatofu`.
