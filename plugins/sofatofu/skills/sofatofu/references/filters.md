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

| Operator | Value | Meaning |
| --- | --- | --- |
| `equals` | One value | The field equals this value. |
| `in` | A non-empty list | The field equals at least one listed value. |
| `not_in` | A non-empty list | The field equals none of the listed values. Unset fields also match. |
| `gte` | A number, date or date-time | The field is greater than or equal to this value. |
| `lte` | A number, date or date-time | The field is less than or equal to this value. |

When you put several operators on one field, all of them must match. For example, `{"value":{"gte":1000,"lte":50000}}` selects values from 1000 through 50000, including both limits.

To find an unset value, use `null` with `equals`. For example, `{"owner":null}` finds deals without an owner. The `in` and `not_in` lists can't contain `null`. Unset values never match `gte` or `lte`.

Text matching ignores case and extra whitespace. Name matching also ignores punctuation. IDs have to match exactly. Domain and email filters require valid domains or email addresses, not arbitrary text; use the same formats as [record writes](records.md#fields-and-matching). A domain filter compares domains the same way as [record writes](records.md#edge-cases-and-formatting-details), so `acme.example.com` also finds `www.acme.example.com`. Stage filters take stage names, archived stages included. An unknown stage name returns `invalid_input` along with the stage names.

## Any of several filters

Use `or` to find records that match at least one filter in a list.

Each filter must contain at least one condition. All conditions within that filter must match.

```text
record search {"type":"company","filter":{"or":[{"domain":{"in":["acme.example.com","bolt.example.com"]}},{"name":{"in":["Cobalt"]}}]},"include":"deleted","limit":200}
```

- The list can contain 1 to 200 filters.
- `or` must be the only top-level key. To apply a condition to every filter in the list, repeat it in each filter.
- Filters in the list cannot contain `or`.
- `include` applies to the whole search.
- Each matching record appears once in the results, even if it matches several filters. `total` counts it once.

## Built-in fields

Text, domain, email, ID, stage and activity-kind fields accept `equals`, `in` and `not_in`. Number, date and date-time fields also accept `gte` and `lte`. `last_activity` has an operator of its own.

| Record type | Field | Value type | Accepts null |
| --- | --- | --- | --- |
| company | `name` | Text | No |
| company | `domain` | Valid domain | Yes |
| person | `name` | Text | No |
| person | `email` | Valid email address | Yes |
| person | `company` | Record ID | Yes |
| deal | `name` | Text | No |
| deal | `company` | Record ID | No |
| deal | `stage` | Stage name | No |
| deal | `value` | Number of 0 or more | Yes |
| deal | `owner` | User ID | Yes |
| deal | `expected_close_date` | Date, such as `2026-10-31` | Yes |
| deal | `last_activity` | `{"older_than_days":N}`, with an integer from 1 to 3650 | No |
| activity | `kind` | `call`, `meeting`, `email` or `note`, in any case | No |
| activity | `occurred_at` | ISO 8601 date-time with `Z` or an offset, such as `2026-09-22T10:00:00Z` | No |
| activity | `person`, `company`, `deal` | Record ID | Yes |
| activity | `subject`, `body` | Text | Yes |

`last_activity` matches deals that have no active activity linked directly to the deal in the last N days, including deals with no activities at all. Deleted activities don't count, nor do activities linked only to the deal's company or a person.

## Custom fields

Put custom filters under `fields`, keyed by the active field name. Every custom field accepts null with `equals`.

| Field type | Operators | Value type |
| --- | --- | --- |
| `text` | `equals`, `in`, `not_in` | Text of 1 to 255 characters with no hidden or direction-control character, NUL, or half of a character pair |
| `number`, `currency` | `equals`, `in`, `not_in`, `gte`, `lte` | Number |
| `date` | `equals`, `in`, `not_in`, `gte`, `lte` | Date, such as `2026-10-31` |
| `checkbox` | `equals`, `in`, `not_in` | `true` or `false` |
| `select` | `equals`, `in`, `not_in` | Option name |

For example, find companies in either region with at least 50 employees:

```text
record search {"type":"company","filter":{"fields":{"Region":{"in":["EMEA","APAC"]},"Employees":{"gte":50}}},"limit":100}
```

## When a filter is rejected

An unknown field, an unsupported operator or a wrong value returns `invalid_input`. The error message lists the accepted field names, operators or value format, so use it to correct the filter.

## Edge cases

`last_activity` depends on the current time, so its matches change as time passes, and that can make a read stale. If a write fails with `read_stale` after a `last_activity` search, repeat the search before you write again.
