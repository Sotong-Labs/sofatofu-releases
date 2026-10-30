# Use the CLI

This page explains how to call sofatofu operations through the `sofatofu` CLI. To install the CLI, sign in, update or uninstall, see [Setup and sign-in](setup.md).

## Command form

The input keys of an operation become arguments:

- **`id` and `ids`** go after the verb, separated by spaces. They are not flags.
- **Every other key** is a flag with the key name in kebab case: `read_id` is `--read-id`, `record_ids` is `--record-ids`, and `order_by` is `--order-by`.
- **Text and numbers** follow the flag, such as `--type company` or `--limit 200`. Quote text that holds spaces.
- **`true`** is the flag alone, such as `--count`. Leave a flag out for `false`.
- **A list of IDs**, such as `record_ids`, repeats the flag once for each item.
- **An object or a list of objects**, such as `filter`, `patch` and `records`, takes one JSON value. Write it inline in single quotes, put `@` before a file path to read the value from that file, or use `@-` to read it from stdin. If the value holds a single quote, or you run in PowerShell, use `@<path>` or `@-`. A file is useful when a script builds the value, such as an import batch.
- **`read_id`** goes in as `--read-id="<read_id>"`. Use the equals sign so an ID that starts with `-` isn't parsed as a flag.

For example, this call:

```text
record update {"type":"deal","ids":["k7m2p9qx4dz1","b3n8r2wq7yt5"],"patch":{"stage":"won"},"reason":"Contracts signed","read_id":"<read_id>"}
```

is this command:

```bash
sofatofu record update k7m2p9qx4dz1 b3n8r2wq7yt5 --type deal --patch '{"stage":"won"}' --reason "Contracts signed" --read-id="<read_id>"
```

These commands show a repeated flag and a value read from a file:

```bash
sofatofu change reverse m5n1p8q2r7s4 --record-ids c4d8e2f6g0h3 --record-ids d5e9f3g7h1j4 --reason "Restore the contacts removed by mistake" --read-id="<read_id>"
sofatofu record create --type company --records @batch-03.json --reason "Import companies, batch 3 of 12" --read-id="<read_id>"
```

## Help

Run `sofatofu --help` to list the commands, and `sofatofu <noun> <verb> --help` to see the flags of one command. The help of a command also states its effect: whether it only reads, adds data, can change or delete data, or sends data to a recipient you name.

## Results and exit codes

A command that succeeds prints its JSON result on stdout and exits with code 0. A command that fails prints the JSON error body on stderr. Its exit code depends on the `error`:

| Exit code | Error |
| --- | --- |
| 2 | `invalid_input` |
| 3 | `not_signed_in` |
| 4 | No answer or server failure: `connection_failed`, `internal`, `server_busy`, `unexpected_response`, `login_unavailable`, `account_unavailable`, `update_unavailable` |
| 1 | Every other error |

For `not_signed_in`, sign in as [Setup and sign-in](setup.md) describes.

## Warnings and notices

A command can print a line that starts with `warning:` or `notice:` on stderr, after its result. These lines don't change the result or the exit code.

- **A skill version warning:** follow it once. Carry on with the user's task even if the warning stays. Whether the task succeeds or fails, mention the warning in your final response and ask the user to start a new agent session.
- **A newer release notice:** tell the user they can install it with `sofatofu update`.
