# Change history and reversal

Every write is recorded as a change. Change history tells you who made a change, when, and why, and it lets you undo most changes.

Every member can read change history. Change reads have no `read_id`.

Changes and deleted records are removed after 30 days. After that, `change reverse` can't restore them.

## Find a change

List the changes for one record:

```text
change list {"record_type":"deal","record_id":"k7m2p9qx4dz1","limit":20}
```

Or find deal creations, updates and deletions during a period (not reversals):

```text
change list {"since":"2026-09-01T00:00:00Z","until":"2026-09-30T23:59:59Z","operation":"record.deal","limit":100}
```

All filters are optional, and a change must match every one you pass. Results list the newest changes first. Each item has an `id`, `at`, `author`, `reason`, `operation` and `reversibility`. [Reverse a change](#reverse-a-change) explains its values.

| Key | Use |
| --- | --- |
| `record_type` and `record_id` | Find changes to one record or configuration item, including reversals. Pass both keys. Types are `company`, `person`, `deal`, `activity`, `stage`, `field`, `workspace`, `member` and `invitation`. For `member`, `record_id` is the user ID. |
| `operation` | Match an operation name or prefix. `record.` matches record creations, updates and deletions, not reversals. `record.deal.update` matches deal updates. Reversals use `change.reverse`. |
| `author` | Filter by the author's user ID, such as a member `id` from `member list`. |
| `since`, `until` | Set inclusive time limits with ISO 8601 date-times ending in `Z` or an offset. |
| `count` | Set to `true` to return the number of matches without change details. |

Operation names include:

- `record.<type>.create`, `record.<type>.update`, `record.<type>.delete`
- `change.reverse`
- `workspace.create`, `workspace.rename`, `workspace.currency.set`, `workspace.stage.create`, `workspace.stage.update`, `workspace.field.create`, `workspace.field.update`
- `account.invitation.create`, `account.invitation.revoke`, `account.member.create`, `account.member.remove`

## Inspect changed values

```text
change get {"id":"m5n1p8q2r7s4","limit":100,"offset":0}
```

`change get` returns the change's details and the items it affected under `records.items`. Use `records.next.offset` to read another page. Each item has its `type`, `id` and `reversible`, plus `before` and `after` values when they exist. `reversible` tells you whether you can undo that record now. A creation has no `before`, and a record deletion has no `after`. A `member` item uses the user ID as its `id`.

`author.id` is the user ID of whoever made the change. It matches the member `id` in `member list` and `user.id` in `auth whoami`. `author.name` is that user's current name, even after they've left the workspace.

An unknown or removed change returns `not_found`.

## Reverse a change

Reversing restores the `before` values as a new change, with your reason. The original history stays intact. Reversing a creation deletes the records it created, and reversing a deletion restores the deleted records. Reversing the creation of a stage or field archives it.

Inspect the change first and check `reversibility`:

- `all`: you can undo every record of the change now.
- `some`: you can undo only some records. Run `change get` and pass the items with `reversible: true` in `record_ids`.
- `none`: you can't undo any record. The kind of change can't be undone, or later changes touched every record. Member access changes and workspace creation can't be undone.

`all` and `reversible: true` mean that no later change has touched the records. The undo is still a write, so it can fail like any write. For example, undoing a domain change fails as `duplicate` if another company has taken the old domain since. Undoing a creation fails as `linked` while other records link to the new record. The error tells you what to fix. An undo or a restore checks no [loose match](records.md#fields-and-matching), so it can give back a name that another record now holds.

You can undo only the latest change to a record. For example, change 1 renames Acme, and change 2 sets its phone. You can't undo change 1. Undoing change 2 doesn't help, because that reversal is a newer change on Acme. To restore the old value, make a normal update.

No two changes with `reversibility: "all"` share a record, so undoing one never blocks another.

By default a reversal covers every item in the change. To reverse only some items, list them in `record_ids`. This works for record and configuration changes alike. Choose the items the request covers, and no others.

Before you reverse, read the current data and pass that read's `read_id`:

- **Record changes:** use `record search` or `record get`, and check that each selected record matches the request. Set `include` to `deleted` to restore records. If any record the read returns changes before the reversal, the reversal fails with `read_stale`, so keep the read narrow.
- **Currency, stage or field changes:** use `workspace get`. Only owners can reverse these changes; other members receive `forbidden`.

For example, restore one person from a deletion of many and leave the others deleted:

```text
change get {"id":"m5n1p8q2r7s4"}
record search {"type":"person","filter":{"email":"ana@example.com"},"include":"deleted","limit":200}
change reverse {"id":"m5n1p8q2r7s4","record_ids":["c4d8e2f6g0h3"],"reason":"Restore the contact removed by mistake","read_id":"<read_id>"}
```

Either every selected record is reversed or none is. You can reverse the rest later with a fresh read. A selection that includes an already-reversed record returns `reversal_conflict`.

Dependency checks still look at records outside your selection. A restored record needs its link targets to exist. And reversing a creation can't leave active records linked to a record it would delete.

When you report a reversal, give the number of items reversed and the new `change_id`.

## Resolve a rejected reversal

Stay within the request when you recover. Being asked to reverse one change doesn't authorize deleting or relinking other records. Ask only about extra changes that fall outside the request. Report the outcome as it is: which error you hit, what you did, and what's still blocked. See [Talking to the user](../SKILL.md#talking-to-the-user).

| Error | What it means and what to do |
| --- | --- |
| `reversal_expired` | The change is unknown or removed, or its kind can't be undone. Read the current state, and make a new write if the correction is still possible. |
| `reversal_conflict` with `later_change_ids` | Later changes touched the selected records. Make a new write with the values you want. Reversing those later changes won't clear the conflict, because they stay in history. |
| `reversal_conflict` with `type` and `id` | The reversal would clear the workspace currency while an active deal has a value. Resolve the deal values that block the reversal. |
| `invalid_input` naming `record_ids` | The list is wrong. Run `change get` to see the change's items, then correct it. |
| `invalid_input` with `id` | The reversal would put back a value that's no longer valid for the record in `id`. The message names the problem. Make the value valid again, for example by restoring a deleted link target, or make a new write with `record update`. |
| `duplicate` | The reversal would give two records the same [strict key](records.md#fields-and-matching). `id` names the record being changed, and `existing_id` names the conflicting record. Inspect both before retrying. |
| `deleted` | A deleted record holds the key the reversal would restore. `id` names that record, and `change_id` names its deletion. Inspect the record before choosing a correction. Restoring it does not free the key. |
| `linked` | Something still uses what the reversal would remove. If active records link to a record it would delete, relink or delete them, as [Delete records](records.md#delete-records) describes. If active deals use a stage it would archive, move them to another stage. Read each record before you change it, then try the reversal again. |
