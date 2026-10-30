# Records

Records are the CRM's companies, people, deals and activities. Every record operation takes a `type` of `company`, `person`, `deal` or `activity`.

`record get`, `record update` and `record delete` take `ids`, a list of one or more record IDs, each listed once. `record create` takes none. The `company`, `person` and `deal` fields point to related records, such as the company a deal belongs to.

## Find records

```text
record search {"type":"person","filter":{"company":"k7m2p9qx4dz1"},"limit":200,"offset":0}
```

Matches come back in `items`, most recently updated first. For the filter syntax, see [Filters and pipeline queries](filters.md). Each page has its own `read_id` and reads current data. To get the next page, repeat the same search with `offset` set to `next.offset`. A write based on a search needs every target on one page, so set `limit` to 200 when you search before a write.

To get just the number of matches, set `count` to `true`. Counts have no limit. Search leaves out deleted records by default. Set `include` to `deleted` to get active and deleted records together. A deleted record has a `deleted_at`.

An activity is one entry in a record's history. To read a record's history with the most recent event first, search activities filtered by that record, and order them by `occurred_at`, which only activities accept:

```text
record search {"type":"activity","filter":{"deal":"k7m2p9qx4dz1","occurred_at":{"gte":"2026-09-01T00:00:00Z","lte":"2026-09-30T23:59:59Z"}},"order_by":"occurred_at","limit":200}
```

If a record has more than 200 activities, narrow the `occurred_at` range.

## Read records by ID

```text
record get {"type":"company","ids":["k7m2p9qx4dz1","b3n8r2wq7yt5"]}
```

The records come back under `records` in the order you gave the IDs, with a single `read_id` for all of them. `record get` returns no activity history; use the activity search above.

If an ID doesn't exist or belongs to a deleted record, the read fails with `not_found`. Set `include` to `deleted` to read deleted records. They carry `deleted_at`, and their `change_id` names the deletion. To restore one, see [Reverse a change](changes.md#reverse-a-change).

Each record includes its `id`, its built-in fields, its custom `fields`, `created_at` and `updated_at`. It also includes `change_id` while that change is still in history. Values that were never set are left out, while `false`, `0` and empty strings are present. Archived custom fields never appear. Deals show `value` as a number and `stage` as a name.

## Create records

`record create` makes records of one type in a single change. Pass `records` one JSON array, where each object holds fields from the [table below](#fields-and-matching) and no ID. Either every record is created or none is.

Search before creating records, with the filter from [The search before a create](../SKILL.md#the-search-before-a-create). Create only records that are missing.

```text
record search {"type":"company","filter":{"domain":{"in":["acme.example.com","bolt.example.com"]}},"include":"deleted","limit":200}
record create {"type":"company","records":[{"name":"Acme","domain":"acme.example.com"},{"name":"Bolt","domain":"bolt.example.com","fields":{"Region":"EMEA"}}],"reason":"Add the leads from the fair","read_id":"<read_id>"}
```

The result returns `ids` in the order you gave the records, plus `effect.created`. If the write is refused, the error tells you what to fix:

- **`duplicate`** means an item has the strict key of another record or of an earlier item. The error gives the item's `index`. For a stored record it also gives `existing_id`: update that record instead of creating the item. For an earlier item it gives `existing_index`: send each key once. The read stays usable.
- **`possible_duplicate`** means items have a [loose match](#fields-and-matching). `items` lists every failing item in input order, with its `index` and `count`, the number of matches. `existing_ids` lists up to 20 matching records, most recently updated first. `existing_indexes` lists other items of this create with the same match. On an update, an item names a target by `id`, and `existing_ids` ends with the other targets with the same match. Follow the `possible_duplicate` rule in [When an operation fails](../SKILL.md#when-an-operation-fails). `force` applies to every item of the write. Change history does not show it, so the `reason` is the only record of your decision.
- **`deleted`** means a deleted record holds the strict key of an item. The error gives the item's `index`, the deleted record's `id`, and the deletion's `change_id`. Restore that record instead of creating a replacement, as [Reverse a change](changes.md#reverse-a-change) describes.
- **`invalid_input`** means something in the request is wrong. Fix what the message names. If the error includes an `index`, it points to an item in `records`. Another error can show up once you've fixed the first.

## Update records

`record update` applies one patch to records of one type in a single change. Pass the IDs in `ids`, and pass `patch` one JSON object with at least one field from the [table below](#fields-and-matching). Either every target is updated or none is.

```text
record search {"type":"deal","filter":{"company":"h6j3k8l2m9n4","stage":"in_progress"},"limit":200}
record update {"type":"deal","ids":["k7m2p9qx4dz1","b3n8r2wq7yt5"],"patch":{"stage":"won","owner":null,"fields":{"Priority":"high"}},"reason":"Contracts signed","read_id":"<read_id>"}
```

The result returns `effect.updated`, the number of records updated. Every update is recorded as a change, even when no value changes, so don't repeat an update to check that it worked.

Omitted fields keep their values, and only JSON `null` clears one. Inside `patch.fields`, a named key replaces that custom value, `null` clears it, and every key you leave out keeps its value on that record. That includes values stored under archived fields.

These patches are rejected: an empty object, an unknown key, or `null` on a required field such as `name`. Only some fields can be cleared:

| Type | Fields you can clear |
| --- | --- |
| `company` | `domain` |
| `person` | `email`, `company` |
| `deal` | `value`, `owner`, `expected_close_date` |
| `activity` | `person`, `company`, `deal`, `subject`, `body` |

An activity must keep at least one person, company or deal link.

Updates cannot give two records the same [strict key](#fields-and-matching). For a [loose match](#fields-and-matching), see `possible_duplicate` in [Create records](#create-records).

- **`duplicate`**: `id` names a target whose key would change. `existing_id` names the other record, which can also be a target of this update. Inspect both before retrying.
- **`deleted`**: a deleted record holds the requested key. `id` names that deleted record, and `change_id` names its deletion.

## Delete records

```text
record search {"type":"person","filter":{"company":"k7m2p9qx4dz1"},"limit":200}
record delete {"type":"person","ids":["b3n8r2wq7yt5","c4d8e2f6g0h3"],"reason":"Remove confirmed duplicates","read_id":"<read_id>"}
```

Choose the IDs that fit the request's deletion criteria. When the criteria identify the targets, don't ask for approval of each ID.

Deleting marks records as deleted. You can [reverse the change](changes.md#reverse-a-change) to restore them within 30 days. When you report a delete, give the count and the `change_id`.

A `linked` error means active records still refer to a target. It names up to 20 of them. If the request covers those changes, relink each blocking record with `record update` and a patch that names another record, or delete it. Read each blocking record before you write to it, then read the targets again. If clearing the error would need changes beyond the request, ask about those changes only.

## Log an activity

Use a `kind` of `call`, `meeting`, `email` or `note`, and link the activity to at least one person, company or deal.

Search for the event before you log it, with the activity filter from [The search before a create](../SKILL.md#the-search-before-a-create):

```text
record search {"type":"activity","filter":{"kind":"call","occurred_at":"2026-09-22T10:00:00Z","person":null,"company":null,"deal":"k7m2p9qx4dz1"},"limit":200}
record create {"type":"activity","records":[{"kind":"call","occurred_at":"2026-09-22T10:00:00Z","deal":"k7m2p9qx4dz1","subject":"Pricing call"}],"reason":"Record the pricing discussion","read_id":"<read_id>"}
```

If you don't know whether a create worked, search again before retrying. If you find the activity, use its ID instead of creating it again. If the first create succeeded, repeating it with the same `kind`, `occurred_at` and links gets `possible_duplicate`, even if `subject` or `body` differs. A retry with another kind, time or link gets no `possible_duplicate` and creates a second activity.

## Fields and matching

A strict key can belong to at most one record in a workspace. A deleted record keeps its key until permanent removal after 30 days. A record without a value for its strict-key field has no strict key. A loose match is another active record of the same type with equal values in every loose-match field. Names compare without case, extra spaces, and punctuation. An empty loose-match field prevents a loose match, but a missing `person`, `company` or `deal` of an activity matches the same missing field. A deleted record never counts. A create, or an update that changes a loose-match field, gets `possible_duplicate` unless `force` is `true`. A change of only case, spaces, or punctuation does not count. `force` never skips a strict key. Other fields can repeat.

| Type | Required on create | Also accepts | Strict key | Loose match |
| --- | --- | --- | --- | --- |
| `company` | `name` | `domain` | `domain`, without one [leading `www.`](#edge-cases-and-formatting-details) | `name` |
| `person` | `name` | `email`, `company` | `email` | `name` and `company` |
| `deal` | `name`, `company` | `stage`, `value`, `owner`, `expected_close_date` | None | `name` and `company` |
| `activity` | `kind`, `occurred_at`, and at least one of `person`, `company`, `deal` | `subject`, `body`, and more than one of `person`, `company`, `deal` | None | `kind`, `occurred_at`, `person`, `company` and `deal` |

Every type also accepts `fields`, a JSON object whose keys are active field names from `workspace get`. A select field takes an option name. How to fill them in:

- **Links.** Pass record IDs to `company`, `person` and `deal`. Pass a member `id` from `member list` to `owner`. A deal keeps its owner after that member leaves the workspace.
- **Stage.** Pass an active stage name. The defaults are `lead`, `in_progress`, `won` and `lost`. A new deal without a `stage` gets the first active stage. Check `workspace get` for the workspace's own stages.
- **Value.** Pass a JSON number of 0 or more, such as `1200.5`. Unrealistically large values lose exactness. Value filters take the same format. Set the workspace currency before you write a deal value.
- **Dates.** Use `YYYY-MM-DD` for `expected_close_date`. Use an ISO 8601 date-time such as `2026-09-22T10:00:00Z` for `occurred_at`, ending in `Z` or an offset. A date-time with no offset, a bare date, or a phrase like `tomorrow` is rejected.
- **Domain and email.** Send a `domain` as a bare name such as `acme.example.com`, with no `https://`, port or trailing dot. Send an `email` as a plain address such as `ann@example.com`.

## Edge cases and formatting details

- **Text.** Text keeps its case. Sofatofu trims leading and trailing whitespace and replaces each run of whitespace, including tabs and newlines, with one space. Activity bodies lose their line breaks. Matching on text ignores case.
- **Length limits.** A `name`, `stage`, `subject` or custom text value holds 1 to 255 characters. A `body` holds 1 to 4000. Emoji are fine.
- **Hidden characters.** Text with a hidden or direction-control character, such as U+200B or U+202E, returns `invalid_input`. So does NUL (U+0000) or half of a character pair anywhere in a request. The error names the field and any hidden character.
- **Email case.** Reads return an email in lowercase.
- **Domains in other alphabets.** Reads return these in their ASCII form, so `пример.рф` comes back as `xn--e1afmkfd.xn--p1ai`. A search finds the domain by either form. A domain label (a part between dots) that mixes Latin letters with letters from another alphabet is rejected. This also applies to the domain part of an email.
- **Leading `www.`.** Reads return a domain with its leading `www.`. Sofatofu ignores one leading `www.` only to compare domains, and only when two or more labels remain, so `www.com` matches only `www.com`.
- **Activity kind.** Case and surrounding whitespace don't matter. Reads return the kind in lowercase.
