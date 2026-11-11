# Filters and pipeline queries

Pass the `filter` of `record search` one JSON object. Each key names a field. A plain value means equality, so `{"stage":"lead"}` and `{"stage":{"equals":"lead"}}` mean the same thing.

Every condition in the object must match. To find records that match at least one of several filters, use [`or`](#any-of-several-filters).

## Pipeline queries

These examples use the default stage names. If the workspace has different stages, use the stage names from `workspace get`. Swap in the dates the user asked about. When you report the results, give `total` and say whether more remain.

Deals in one stage:

```text
record search {"type":"deal","filter":{"stage":"in_progress"},"limit":100}
```

Open deals expected to close in September 2026:

```text
record search {"type":"deal","filter":{"expected_close_date":{"gte":"2026-09-01","lte":"2026-09-30"},"stage":{"not_in":["won","lost"]}},"limit":100}
```

Open deals without an owner:

```text
record search {"type":"deal","filter":{"owner":null,"stage":{"not_in":["won","lost"]}},"limit":100}
```

Open deals with no activity in the last 30 days:

```text
record search {"type":"deal","filter":{"last_activity":{"older_than_days":30},"stage":{"not_in":["won","lost"]}},"limit":100}
```

## Operators

| Operator   | Value                       | Meaning                                                              |
| ---------- | --------------------------- | -------------------------------------------------------------------- |
| `equals`   | One value                   | The field equals this value.                                         |
| `in`       | A non-empty list            | The field equals at least one listed value.                          |
| `not_in`   | A non-empty list            | The field equals none of the listed values. Unset fields also match. |
| `gte`      | A number, date or date-time | The field is greater than or equal to this value.                    |
| `lte`      | A number, date or date-time | The field is less than or equal to this value.                       |
| `contains` | One text value              | The field holds this text.                                           |

When you put several operators on one field, all of them must match. For example, `{"value":{"gte":1000,"lte":50000}}` selects values from 1000 through 50000, including both limits.

To find an unset value, use `null` with `equals`. For example, `{"owner":null}` finds deals without an owner. The `in` and `not_in` lists can't contain `null`. Unset values never match `gte` or `lte`.

Text matching ignores extra whitespace and, except for `source_id`, ignores case. Name matching also ignores punctuation. `contains` follows these same rules. Record IDs must match exactly. All `source_id` comparisons are case-sensitive: `equals`, `in`, `not_in` and `contains`.

- **`contains`** finds any part of the text. A name text that is only punctuation fails. To match any of several texts, use [`or`](#any-of-several-filters). When you know the full name, use `equals`: results come back by last update, not by how well they match.
- **Domains and emails:** `equals` needs a valid domain or address, in the formats of [record writes](records.md#fields-and-matching). `contains` takes any part, such as `@acme` or `.de`, and ignores full-width forms. A domain compares without one leading `www.`, so `acme.example.com` also finds `www.acme.example.com`, and `www` alone does not find `www.acme.org`.
- **Phones:** a phone compares as text, with no change to its digits. `{"phone":{"contains":"010-2030"}}` finds `+1 (555) 010-2030`, and `0102030` does not.
- **Stages:** filters take stage names, archived stages included. An unknown stage name returns `invalid_input` with the stage names.

Deals with "pilot" in the name, such as "Q3 Pilot":

```text
record search {"type":"deal","filter":{"name":{"contains":"pilot"}}}
```

## Any of several filters

Use `or` to find records that match at least one filter in a list.

Each filter must contain at least one condition. All conditions within that filter must match.

```text
record search {"type":"company","filter":{"or":[{"domain":{"in":["acme.example.com","bolt.example.com"]}},{"name":{"in":["Cobalt"]}}]},"limit":200}
```

- The list can contain 1 to 200 filters.
- `or` must be the only top-level key. To apply a condition to every filter in the list, repeat it in each filter.
- Filters in the list cannot contain `or`.
- Each matching record appears once in the results, even if it matches several filters. `total` counts it once.

## Built-in fields

Text, domain, email, ID, stage and activity-kind fields accept `equals`, `in` and `not_in`. Number, date and date-time fields also accept `gte` and `lte`. `name`, `subject`, `email`, `domain`, `phone`, `job_title` and `source_id` also accept `contains`, and `body` does not. `last_activity` has an operator of its own.

| Record type | Field                 | Value type                                                               | Accepts null |
| ----------- | --------------------- | ------------------------------------------------------------------------ | ------------ |
| company     | `name`                | Text                                                                     | No           |
| company     | `domain`              | Valid domain                                                             | Yes          |
| person      | `name`                | Text                                                                     | No           |
| person      | `email`               | Valid email address                                                      | Yes          |
| person      | `company`             | Record ID                                                                | Yes          |
| person      | `phone`, `job_title`  | Text                                                                     | Yes          |
| deal        | `name`                | Text                                                                     | No           |
| deal        | `company`             | Record ID                                                                | No           |
| deal        | `stage`               | Stage name                                                               | No           |
| deal        | `value`               | Number of 0 or more                                                      | Yes          |
| deal        | `owner`               | User ID                                                                  | Yes          |
| deal        | `expected_close_date` | Date, such as `2026-10-31`                                               | Yes          |
| deal        | `last_activity`       | `{"older_than_days":N}`, with an integer from 1 to 3650                  | No           |
| activity    | `kind`                | `call`, `meeting`, `email` or `note`, in any case                        | No           |
| activity    | `occurred_at`         | ISO 8601 date-time with `Z` or an offset, such as `2026-09-22T10:00:00Z` | No           |
| activity    | `people`              | Person ID                                                                | Yes          |
| activity    | `company`, `deal`     | Record ID                                                                | Yes          |
| activity    | `subject`, `body`     | Text                                                                     | Yes          |
| every type  | `source_id`           | Text                                                                     | Yes          |

The `people` filter finds activities linked to any of the given people, including activities linked to other people too. `not_in` finds activities linked to none of the given people. Use `null` to find activities with no linked people.

`last_activity` finds deals with no activity in the last N days, including deals with no activity at all. It counts only activities linked directly to the deal, not those linked only to its company or a person.

## Custom fields

Put custom filters under `fields`, keyed by the active field name. Every custom field accepts null with `equals`.

| Field type           | Operators                              | Value type                                                                                                    |
| -------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `text`               | `equals`, `in`, `not_in`, `contains`   | Text of 1 to 255 characters with no hidden or direction-control character, NUL, or half of a character pair   |
| `long_text`          | `equals`, `in`, `not_in`               | Text of 1 to 10000 characters with no hidden or direction-control character, NUL, or half of a character pair |
| `number`, `currency` | `equals`, `in`, `not_in`, `gte`, `lte` | Number                                                                                                        |
| `date`               | `equals`, `in`, `not_in`, `gte`, `lte` | Date, such as `2026-10-31`                                                                                    |
| `checkbox`           | `equals`, `in`, `not_in`               | `true` or `false`                                                                                             |
| `select`             | `equals`, `in`, `not_in`               | Option name                                                                                                   |
| `multi_select`       | `equals`, `in`, `not_in`               | Option name                                                                                                   |

A `long_text` filter compares the full value in lower case, after it trims the start and end and changes each run of whitespace to one space. It does not accept `contains`.

A `select` or `multi_select` filter accepts the names of archived options too. A `multi_select` filter compares each option on its own. `equals` and `in` find records that hold at least one of the given options, also when they hold other options. `not_in` finds records that hold none of them.

For example, find companies in either region with at least 50 employees:

```text
record search {"type":"company","filter":{"fields":{"Region":{"in":["EMEA","APAC"]},"Employees":{"gte":50}}},"limit":100}
```

## When a filter is rejected

An unknown field, an unsupported operator or a wrong value returns `invalid_input`. The error message lists the accepted field names, operators or value format, so use it to correct the filter.
