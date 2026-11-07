# Import from Pipedrive

Follow [Import and export](import-export.md) for the import method. This page covers Pipedrive's data and field maps.

## Get the data

Use the [Pipedrive API](https://developers.pipedrive.com/docs/api/v1) with access the user provides, or ask for [export files](https://support.pipedrive.com/en/article/exporting-data-from-pipedrive). Both paths return only data visible to that Pipedrive user. Pipedrive's [MCP server](https://support.pipedrive.com/en/article/mcp-tools) has no tools for archived deals or leads. If you read through it, get archived items from an export.

For files, a global admin opens **Tools and apps > Export data** and exports Organizations, People, Deals, Leads, Activities and Notes as Excel or CSV. Select all archive states for deals and leads. Export activities and notes separately, even when they link to deals or contacts. For line items, check that the files hold each deal's products, not only the product catalog.

Check that each file has an ID column for its own items and each linked item. Read the files as [Spreadsheets and CSV files](import-spreadsheet.md) describes.

For API reads, use v2 where available. These endpoints provide the records and the data needed to map them:

| Data                             | Read endpoint                                                                                                        |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Organizations and people         | `GET /api/v2/organizations`, `GET /api/v2/persons`                                                                   |
| Deals                            | `GET /api/v2/deals` and `GET /api/v2/deals/archived`                                                                 |
| Leads                            | `GET /api/v1/leads` and `GET /api/v1/leads/archived`                                                                 |
| Activities                       | `GET /api/v2/activities?include_fields=attendees`                                                                    |
| Notes                            | `GET /api/v1/notes`                                                                                                  |
| Pipelines and stages             | `GET /api/v2/pipelines`, `GET /api/v2/stages`                                                                        |
| Users                            | `GET /api/v1/users`                                                                                                  |
| Field definitions                | `GET /api/v2/organizationFields`, `GET /api/v2/personFields`, `GET /api/v2/dealFields`, `GET /api/v2/activityFields` |
| Lead labels and activity types   | `GET /api/v1/leadLabels`, `GET /api/v1/activityTypes`                                                                |
| Products on deals                | `GET /api/v2/deals/products?deal_ids=` with up to 100 comma-separated deal IDs                                       |
| Email messages linked to records | `GET /api/v1/persons/{id}/mailMessages`, `GET /api/v1/deals/{id}/mailMessages`                                       |

The regular deal and lead lists exclude archived items. Read both lists before applying the [import scope](import-export.md#before-you-import). Leave out deals with `status: "deleted"`. Archive state is separate from a deal's open, won or lost status.

Follow Pipedrive's [pagination](https://pipedrive.readme.io/docs/core-api-concepts-pagination): v2 lists return `additional_data.next_cursor` for the next `cursor`. Stop at `null`. Offset-based lists return `additional_data.pagination.next_start` for the next `start` while `more_items_in_collection` is true. Use `limit=500` for paged reads. All integrations of a Pipedrive company share its [daily API token budget](https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting). When it is spent, requests get `429` until the budget resets at midnight in the server's time zone, not necessarily the user's. Estimate the request count and token cost before a large read.

Read email messages through the API. For a body missing from a list result, use `GET /api/v1/mailbox/mailMessages/{id}?include_body=1`. Check a message's fields for its send time, sender and recipients. The same message can link to several records, so combine those links under one message ID.

## How the objects link

Read users, pipelines, stages and field definitions before mapping record values. The field names below follow the [v2 migration guide](https://pipedrive.readme.io/docs/pipedrive-api-v2-migration-guide), except for notes and leads, which use v1.

| Link                       | Pipedrive fields                                                                                                                 |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Deal to stage and pipeline | Deal `stage_id` and `pipeline_id`. Each stage also has `pipeline_id`.                                                            |
| Lead to contacts           | `person_id`, `organization_id`, or both                                                                                          |
| Owner to user              | `owner_id`. Resolve it through the users list, including users with `active_flag: false`, then use the email for the member map. |

A deal without `org_id` may have a company through its contact's `org_id`. Check that this is the company for the deal. If neither link gives a company, include the deal in the [import questions](import-export.md#before-you-import).

A lead points to contacts that also appear in the person and organization lists. Apply its lead fields to those contacts, not to a second copy. Use the person when present, or the organization for an organization-only lead. For notes and activities linked only to a lead, resolve the lead's contact links.

Map `is_archived` to the lead state and resolve `label_ids` through the lead labels. Other lead values include `title`, `source_name`, `expected_close_date` and `value` (an `amount` and `currency`). Map these to custom fields where needed. If several leads share a contact, ask how to keep their different values.

## Map to Sofatofu

Use [Fields and matching](records.md#fields-and-matching) for Sofatofu's value formats and [Source IDs](#source-ids) for each record's identity. Use [Data with no field of its own](import-export.md#data-with-no-field-of-its-own) for source creation dates, labels, extra contact values and other fields outside these maps.

### Company

| Pipedrive organization    | Sofatofu field                                                                          |
| ------------------------- | --------------------------------------------------------------------------------------- |
| `name`                    | `name`                                                                                  |
| Website value, if present | `domain`. Check whether it is `website` or a custom field, then extract the host.       |
| `address`                 | Custom text field from `formatted_address`, or `value` when no formatted address exists |
| `owner_id`                | Custom field with the source user's name, not a member owner link                       |

### Person

| Pipedrive person        | Sofatofu field                                                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `name`                  | `name`                                                                                                                       |
| `emails`                | `email` from the non-empty entry with `primary: true`, or the first non-empty entry. Use the shared map for other addresses. |
| `phones`                | `phone` from the first non-empty entry. Use the shared map for other numbers.                                                |
| `job_title`, if present | `job_title`. Otherwise check for a custom field with that meaning.                                                           |
| `org_id`                | `company` through the organization map                                                                                       |

If `emails` or `phones` has no non-empty `value`, omit the corresponding Sofatofu field.

### Deal

| Pipedrive deal                      | Sofatofu field                                                                                                                                                                                                                                                 |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `title`                             | `name`                                                                                                                                                                                                                                                         |
| `org_id`                            | `company`, with the missing-organization check above                                                                                                                                                                                                           |
| `status`, `pipeline_id`, `stage_id` | `stage`. Read status first: won and lost deals can keep an earlier pipeline stage. Map each open deal by its pipeline and stage together, not by stage name alone.                                                                                             |
| `value`, `currency`                 | `value` only in the workspace currency. Include other currencies in the import questions.                                                                                                                                                                      |
| `owner_id`                          | `owner` through the user-to-member map                                                                                                                                                                                                                         |
| `expected_close_date`               | `expected_close_date`                                                                                                                                                                                                                                          |
| `pipeline_id`                       | Custom select field with the pipeline name when several pipelines share the workspace's stage list                                                                                                                                                             |
| Deal product rows                   | A deal line-item note linked through `deal`, per [Data with no field of its own](import-export.md#data-with-no-field-of-its-own). List each item's `name`, `quantity`, `item_price`, `discount`, `sum` and `currency`. The deal's creation time is `add_time`. |

### Activity

| Pipedrive data                        | Sofatofu field                                                                                                                                                                                                     |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Activity `type`                       | `kind`: `call`, `meeting` and `email` map directly. `lunch` maps to `meeting`. Completed `task` and `deadline` items map to `note`. Resolve custom types through the activity types list.                          |
| Activity `due_date`, `due_time`       | `occurred_at` for a call, meeting or email with `done: true`. Check the time zone before joining these values. For completed tasks and deadlines, use `marked_as_done_time`.                                       |
| Activity `subject`                    | `subject`                                                                                                                                                                                                          |
| Activity `note`, `public_description` | `body` from `note`. Append `public_description` to the body after the label `Description:`. Check for HTML and convert it to text.                                                                                 |
| Activity `participants`, `attendees`  | `people`. `person_id` names only the primary participant. Combine `participants[].person_id` and `attendees[].person_id`. An attendee can have an email but no person ID, so check for a matching imported person. |
| Activity `org_id`, `deal_id`          | `company`, `deal`                                                                                                                                                                                                  |
| Note `content`                        | `body` with `kind: "note"`. Convert the HTML to text before applying the shared long-body rule.                                                                                                                    |
| Note `add_time`                       | `occurred_at`. Convert the UTC value from `YYYY-MM-DD HH:MM:SS` to `YYYY-MM-DDTHH:MM:SSZ`.                                                                                                                         |
| Note `person_id`, `org_id`, `deal_id` | `people`, `company`, `deal`                                                                                                                                                                                        |
| Email message                         | `kind: "email"`, with its subject, body, send time and linked contacts or deal. Verify the returned field names.                                                                                                   |

An overdue activity with `done: false` does not prove that the event happened. Ask whether past calls, meetings and emails in that state happened before mapping their event times. Include missing dates, times, unresolved custom types and items linked only to a project in the import questions.

### Custom fields

In v2, values are in `custom_fields` under field codes. Field definitions give each code's `field_name`, `field_type` and options. Leads inherit deal custom fields, but use the v1 response shape. Inspect that shape rather than applying the v2 parser to it.

| Pipedrive `field_type`             | Sofatofu custom field                                                                               |
| ---------------------------------- | --------------------------------------------------------------------------------------------------- |
| `varchar`, `varchar_auto`, `phone` | `text`                                                                                              |
| `text`                             | `long_text`                                                                                         |
| `double`                           | `number`                                                                                            |
| `monetary`                         | `currency`. The v2 value contains both `value` and `currency`, so group by currency before mapping. |
| `enum`                             | `select`, with the option label                                                                     |
| `set`                              | `multi_select`, with the option labels                                                              |
| `date`                             | `date`                                                                                              |
| `daterange`                        | Two `date` fields from `value` and `until`                                                          |
| `time`, `timerange`                | `text` with the time or range and its `timezone_name`, when present                                 |
| `address`                          | `text` from `formatted_address` or `value`                                                          |
| `user`, `org`, `people`            | `text` with the linked item's name, not a record link                                               |

For organizations, people and deals, `include_option_labels=true` resolves custom option IDs. `include_labels=true` adds label names next to `label_ids`. Otherwise resolve IDs through the field definitions. A Pipedrive custom currency is not necessarily an ISO 4217 code. Check it before mapping to a currency field.

## Source IDs

Apply the shared [Source IDs](import-export.md#source-ids) rules with `pipedrive` as the source. Organizations, people, deals, activities, notes and mail messages each have a numeric `id`. A lead has a UUID `id`.

- Activities, notes and mail messages all become Sofatofu activities. Pipedrive does not document that their IDs differ, so treat them as objects that can give the same ID.
- A deal line-item note is a record you build from its deal, not a Pipedrive note.
- A contact that receives lead fields keeps its person or organization identity. The lead UUID does not replace the contact's ID.
- Check that each ID column holds Pipedrive IDs. If a file has none, read the API or ask for a new export. Without IDs, follow [Source IDs](import-spreadsheet.md#source-ids) for rows with no stable ID.

## What does not come in

Apply [When data doesn't fit](import-export.md#when-data-doesnt-fit) to these Pipedrive items:

| Pipedrive data                                                                         | Limit                                                                                                           |
| -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Email messages, when you read only export files                                        | These exports do not contain the messages. Read them through the API, or list the loss in the import questions. |
| Future activities and activities with `done: false` that the user does not confirm     | Planned work, not completed activity                                                                            |
| Activity counts, weighted value and rotten time                                        | Pipedrive computes these values. Sofatofu does not.                                                             |
| Deal `flow`, field changelogs and stage history                                        | These do not become Sofatofu changes.                                                                           |
| Product catalog, Projects, Campaigns, subscriptions, installments and goals            | These have no matching record type.                                                                             |
| Files, Google Drive files and call recordings                                          | These do not become attachments in Sofatofu.                                                                    |
| Deal `person_id`, deal participants, organization `parent` and `related` relationships | These have no matching record link.                                                                             |
| Filters, Insights reports, dashboards, automations and web forms                       | These do not become workspace configuration.                                                                    |
