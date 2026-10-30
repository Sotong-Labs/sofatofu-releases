# Workspace configuration

A workspace defines three things: the currency for deal values, the deal stages, and the custom fields. Every member can read these settings, but only owners can change them.

## Read the settings

```text
workspace get {}
```

`workspace get` returns the workspace `name`, your `role`, the deal `currency`, and every stage and field in the `stages` and `fields` arrays. `currency` is `null` until an owner sets one. The arrays are always complete, because this operation has no list limits.

Each stage item has `id`, `name`, `position` and `archived`. Each field item has `id`, `record_type`, `name`, `type`, `currency`, `options` and `archived`. Items also carry their creation and update times, and a `change_id` while that change is still in history.

## Change one setting

Pass the `read_id` from `workspace get` to `workspace configure`. Each call takes exactly one of `currency`, `name`, `stage` or `field`, along with `reason` and `read_id`.

Any configuration change makes an earlier configuration read stale, so read again before each write. The examples below are separate writes. They aren't a sequence that can share one `read_id`.

Creating a stage or field returns its new ID in `ids`. An update returns only `change_id` and `effect`.

When you report a change, say what changed.

## Set the deal currency

```text
workspace configure {"currency":"EUR","reason":"Company reports deal values in euro","read_id":"<read_id>"}
```

Use an ISO 4217 code such as `EUR` or `USD`. Set the currency before you write any deal value.

Once a currency is set, `workspace configure` can't replace it. You can reverse the change that set it, but only if no active deal has a value and the reversal passes the checks in [Resolve a rejected reversal](changes.md#resolve-a-rejected-reversal).

## Rename the workspace

```text
workspace configure {"name":"Acme Sales","reason":"Match the team's name","read_id":"<read_id>"}
```

Only an owner can rename the workspace. The change history records the rename, and you can reverse it like any other configuration change.

## Create or change a stage

Pass `stage` a JSON object. To create a stage, give it a `name` and leave out `id`. To update one, pass its `id` from `workspace get` along with the fields to change.

Create a stage at position 1:

```text
workspace configure {"stage":{"name":"qualified","position":1},"reason":"Separate qualified leads from new enquiries","read_id":"<read_id>"}
```

Rename a stage:

```text
workspace configure {"stage":{"id":"s2t6u0v4w8x1","name":"negotiation"},"reason":"Match the sales team's stage name","read_id":"<read_id>"}
```

Archive a stage:

```text
workspace configure {"stage":{"id":"s2t6u0v4w8x1","archived":true},"reason":"Stop using this sales stage","read_id":"<read_id>"}
```

What to know about stages:

- Names hold 1 to 255 characters and must be unique, archived names included. Comparison ignores case and extra whitespace.
- `position` is an integer starting at 0. A new stage with no position goes last.
- Deal operations use stage names, not stage IDs. Deal writes accept only active stages. Stage filters also accept archived stages.
- Archived stages stay in `workspace get`. Set `archived` to `false` to make one active again.
- If active deals use a stage, archiving it returns `linked` along with the count. Search deals by that stage and move them to another stage first. Deleted deals don't block it, and they keep the archived stage.
- Reversing a stage's creation archives the stage, and it keeps its name. Like any reversal that would archive a stage, it returns `linked` while active deals use the stage.

## Create or change a custom field

Pass `field` a JSON object. To create a field, leave out `id` and supply `record_type`, `name` and `type`. The types are `text`, `number`, `currency`, `date`, `checkbox` and `select`.

Create a select field with three options:

```text
workspace configure {"field":{"record_type":"company","name":"Region","type":"select","options":[{"name":"EMEA"},{"name":"AMER"},{"name":"APAC"}]},"reason":"Assign companies to sales regions","read_id":"<read_id>"}
```

A currency field needs its own ISO 4217 currency code:

```text
workspace configure {"field":{"record_type":"deal","name":"Budget","type":"currency","currency":"USD"},"reason":"Record budgets quoted in US dollars","read_id":"<read_id>"}
```

To update a field, pass its `id` and one or more of `name`, `options` and `archived`. For example, add an option to a select field:

```text
workspace configure {"field":{"id":"f9g3h7j1k5l8","options":[{"name":"LATAM"}]},"reason":"Assign Latin America its own sales region","read_id":"<read_id>"}
```

What to know about fields:

- Field and option names hold 1 to 255 characters. Field names must be unique within a record type, archived names included. Comparison ignores case and extra whitespace.
- `record_type`, `type` and `currency` can't change. If one is wrong, archive the field and create a new one with a different name.
- Only a currency field accepts `currency`, and only a select field accepts `options`.
- An option without an `id` adds a choice, and an option with an `id` renames that choice. Options you leave out stay as they are. You can't delete or archive options, and their names must be unique within the field.
- To stop using a field, set `archived` to `true`. Archived fields stay in `workspace get`, but record writes and filters reject them, and record reads leave out their values. The stored values are kept, so they come back if you set `archived` to `false`.
- Reversing a field's creation archives the field. The field keeps its name, and records keep their stored values for it.
