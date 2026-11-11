# Records

Records are the CRM's companies, people, deals and activities. Every record operation takes a `type` of `company`, `person`, `deal` or `activity`.

For `record get` and `record delete`, provide one or more record IDs in `ids`.

For `record create` and `record update`, provide a list of objects in `records`.

Use `company` and `deal` to link related records, such as a deal and its company. An activity can also link to people through its `people` list.

## Find records

```text
record search {"type":"person","filter":{"company":"k7m2p9qx4dz1"},"limit":200}
```

By default, `items` lists the matches newest first by `created_at`. For the filter syntax, see [Filters and pipeline queries](filters.md). To read the next page, repeat the same search with `cursor` set to `next.cursor`. A write based on a search needs every target on one page, so set `limit` to 200 when you search before a write.

To get just the number of matches, set `count` to `true`. Counts have no limit.

An activity is one entry in a record's history. To read a record's history with the most recent event first, search activities filtered by that record, and order them by `occurred_at`, which only activities accept:

```text
record search {"type":"activity","filter":{"deal":"k7m2p9qx4dz1","occurred_at":{"gte":"2026-09-01T00:00:00Z","lte":"2026-09-30T23:59:59Z"}},"order_by":"occurred_at","limit":200}
```

If a record has more than 200 activities, narrow the `occurred_at` range. A write that changes `occurred_at` during this walk can make an activity missing or repeated.

## Read records by ID

```text
record get {"type":"company","ids":["k7m2p9qx4dz1","b3n8r2wq7yt5"]}
```

The records come back under `records` in the order you gave the IDs, with a single `read_id` for all of them. `record get` returns no activity history; use the activity search above.

If an ID doesn't exist, the read fails with `not_found`. To restore a deleted record, see [Reverse a change](changes.md#reverse-a-change).

Each record includes its `id`, its built-in fields, its custom `fields`, `created_at` and `updated_at`. It also includes `change_id` while that change is still in history. Values that were never set are left out, while `false`, `0` and empty strings are present. Archived custom fields never appear. Deals show `value` as a number and `stage` as a name.

## Create records

`record create` makes records of one type in a single change, with the fields from the [table below](#fields-and-matching). Either every record is created or none is.

Search before creating records, with the filter from [The search before a create](../SKILL.md#the-search-before-a-create). Create only records that are missing.

```text
record search {"type":"company","filter":{"domain":{"in":["acme.example.com","bolt.example.com"]}},"limit":200}
record create {"type":"company","records":[{"name":"Acme","domain":"acme.example.com"},{"name":"Bolt","domain":"bolt.example.com","fields":{"Region":"EMEA"}}],"reason":"Add the leads from the fair","read_id":"<read_id>"}
```

The result returns `ids` in the order you gave the records, plus `effect.created`. A refused create can return these entries in `errors`:

- **`duplicate`**: an item has the strict key of another record or of an earlier item. `items` gives the item's position in `records`. `existing_id` names the stored record, and `existing_index` names the earlier item.
- **`possible_duplicate`**: an item has a [loose match](#fields-and-matching). The entries follow request order. Each entry gives the item's position in `items` and the number of matches in `count`. `existing_ids` lists up to 20 matching records, most recently updated first. `existing_indexes` lists up to 20 matching items in the same request. On an update, `existing_ids` ends with the other targets with the same match.

## Update records

`record update` changes records of one type in a single change, with the fields from the [table below](#fields-and-matching). You can update different fields and values for each record. The entire update succeeds or nothing changes.

```text
record search {"type":"deal","filter":{"company":"h6j3k8l2m9n4","stage":"in_progress"},"limit":200}
record update {"type":"deal","records":[{"id":"k7m2p9qx4dz1","stage":"won","owner":null},{"id":"b3n8r2wq7yt5","value":12000,"fields":{"Priority":"high"}}],"reason":"The sales review confirmed the win and revised estimate","read_id":"<read_id>"}
```

The result returns `effect.updated`, the number of records with at least one changed value. Sofatofu skips each record whose values all equal the stored values. If no values change, the update returns no `change_id` and does not affect whether you can undo earlier changes.

For custom fields, include the names and values you want to change in `fields`. Other custom values stay unchanged, including values in archived fields. An empty list also clears a multi-select field.

You can clear these built-in fields with `null`:

| Type       | Fields you can clear                                        |
| ---------- | ----------------------------------------------------------- |
| `company`  | `domain`, `source_id`                                       |
| `person`   | `email`, `company`, `phone`, `job_title`, `source_id`       |
| `deal`     | `value`, `owner`, `expected_close_date`, `source_id`        |
| `activity` | `people`, `company`, `deal`, `subject`, `body`, `source_id` |

An activity must keep at least one link: a person in `people`, a `company`, or a `deal`.

Updates follow the same [strict key](#fields-and-matching) and [loose match](#fields-and-matching) rules as [creates](#create-records). In a `duplicate` entry, `existing_id` can name another record in the same update.

## Delete records

```text
record search {"type":"person","filter":{"company":"k7m2p9qx4dz1"},"limit":200}
record delete {"type":"person","ids":["b3n8r2wq7yt5","c4d8e2f6g0h3"],"reason":"Remove confirmed duplicates","read_id":"<read_id>"}
```

Choose the IDs that fit the request's deletion criteria. When the criteria identify the targets, don't ask for approval of each ID.

Deleting removes records from search and direct reads. Change history keeps a copy for 30 days, so you can [reverse the deletion](changes.md#reverse-a-change) to restore them. Once Sofatofu removes the change, you can no longer reverse that deletion. Report the number of deleted records and the `change_id`.

A `linked` error means active records still link to records you want to delete. Each `errors` entry describes one blocked record. `items` holds its position in `ids`, `count` gives the number of linking records, and `records` lists them. The response lists at most 20 linking records in entry order. Later lists may be empty.

Change those links only when the user's request covers it. Ask for approval of changes outside the request, and only those.

## Log an activity

Use a `kind` of `call`, `meeting`, `email` or `note`, and link the activity to at least one person, company or deal.

Search for the event before you log it, with the activity filter from [The search before a create](../SKILL.md#the-search-before-a-create):

```text
record search {"type":"activity","filter":{"kind":"call","occurred_at":"2026-09-22T10:00:00Z","people":null,"company":null,"deal":"k7m2p9qx4dz1"},"limit":200}
record create {"type":"activity","records":[{"kind":"call","occurred_at":"2026-09-22T10:00:00Z","deal":"k7m2p9qx4dz1","subject":"Pricing call"}],"reason":"Record the pricing discussion","read_id":"<read_id>"}
```

If you don't know whether a create worked, search again before retrying. If you find the activity, use its ID instead of creating it again. If the first create succeeded, repeating it with the same `kind`, `occurred_at` and links gets `possible_duplicate`, even if `subject` or `body` differs. A retry with another kind, time or link gets no `possible_duplicate` and creates a second activity.

## Fields and matching

Each strict-key field must have a unique value among records of the same type in a workspace. Sofatofu checks each field separately. Unset fields do not count as strict keys. A loose match is another record of the same type with equal values in every loose-match field. Names compare without case, extra spaces, and punctuation. An empty loose-match field prevents a loose match, except for an activity's `people`, `company` and `deal` fields. For these fields, an empty value matches another empty value. Two activities have equal `people` when they list the same people, in any order. A create, or an update that changes a loose-match field, gets `possible_duplicate` unless `force` is `true`. A change of only case, spaces, or punctuation does not count. `force` never skips a strict key. Other fields can repeat.

| Type       | Required on create                                                                               | Also accepts                                                                     | Strict keys (each unique)                                                                   | Loose match                                           |
| ---------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `company`  | `name`                                                                                           | `domain`, `source_id`                                                            | `domain`, without one [leading `www.`](#edge-cases-and-formatting-details), and `source_id` | `name`                                                |
| `person`   | `name`                                                                                           | `email`, `company`, `phone`, `job_title`, `source_id`                            | `email` and `source_id`                                                                     | `name` and `company`                                  |
| `deal`     | `name`, `company`                                                                                | `stage`, `value`, `owner`, `expected_close_date`, `source_id`                    | `source_id`                                                                                 | `name` and `company`                                  |
| `activity` | `kind`, `occurred_at`, and at least one link (a person ID in `people`, a `company`, or a `deal`) | `subject`, `body`, `source_id`, and more than one of `people`, `company`, `deal` | `source_id`                                                                                 | `kind`, `occurred_at`, `people`, `company` and `deal` |

Every type also accepts `fields`, a JSON object whose keys are active field names from `workspace get`. A select field takes an option name. A multi-select field takes a list of option names, each name once, and the list replaces the stored list. An empty list or `null` clears it. A write can't add an archived option to a record. A record keeps the archived options that it holds, and a multi-select write can keep them in the list. How to fill them in:

- **Links.** Pass record IDs to `company` and `deal`. Set `people` to a list of up to 50 person IDs, such as the IDs of meeting attendees. This replaces the activity's entire people list. Use `[]` or `null` to clear it. Pass a member `id` from `member list` to `owner`. A deal keeps its owner after that member leaves the workspace.
- **Stage.** Pass an active stage name from `workspace get`. A new deal without a `stage` gets the first active stage.
- **Value.** Pass a JSON number of 0 or more, such as `1200.5`. Unrealistically large values lose exactness. Value filters take the same format. Set the workspace currency before you write a deal value.
- **Dates.** Use `YYYY-MM-DD` for `expected_close_date`. Use an ISO 8601 date-time such as `2026-09-22T10:00:00Z` for `occurred_at`, ending in `Z` or an offset. A date-time with no offset, a bare date, or a phrase like `tomorrow` is rejected.
- **Domain and email.** Send a `domain` as a bare name such as `acme.example.com`, with no `https://`, port, path, query, fragment, or trailing dot. Send an `email` as a plain address such as `ann@example.com`.
- **Phone.** Send a `phone` in any format, such as `+1 (555) 010-2030 ext. 4`. Sofatofu stores it as sent, after the [text rules](#edge-cases-and-formatting-details), and does not check its format. A phone is not a strict key or a loose-match field, so people can share one.
- **Source ID.** Use the record's ID from its source system, prefixed with the source name: `<source>:<id>`, for example `hubspot:123`. A company and a person can share a source ID. Source IDs are case-sensitive, so `Hubspot:1` and `hubspot:1` are different values.

## Edge cases and formatting details

- **Text.** Text keeps its case. Sofatofu trims leading and trailing whitespace. An activity `body` and a `long_text` field value keep their line breaks, tabs and indentation, and each `\r\n` or `\r` line ending becomes `\n`. Other text replaces each run of whitespace, including tabs and newlines, with one space. Text matching ignores case except for `source_id`. A `body` or `long_text` filter also trims the start and end and treats each run of whitespace as one space.
- **Length limits.** A `name`, `stage`, `subject`, `phone`, `job_title`, `source_id` or custom `text` value holds 1 to 255 characters. A `body` or `long_text` value holds 1 to 10000 characters. Both limits count the stored text, after NFC, soft-hyphen removal, and the whitespace rules above. Emoji are fine.
- **Hidden characters.** Sofatofu removes soft hyphens (U+00AD) from text, emails, and domains in writes and filters. Text that contains only soft hyphens returns `invalid_input` because it is empty. Disallowed hidden or direction-control characters, such as U+200B or U+202E, return `invalid_input`. So does NUL (U+0000) or half of a character pair anywhere in a request. The error names the field and any hidden character.
- **Email case.** Reads return an email in lowercase.
- **Leading `www.`.** Reads return a domain with its leading `www.`. Sofatofu ignores one leading `www.` only to compare domains, and only when two or more labels remain, so `www.com` matches only `www.com`.
- **Activity kind.** Case and surrounding whitespace don't matter. Reads return the kind in lowercase.
