# Import and export

This page covers moving records into Sofatofu from a spreadsheet or another CRM, and writing records out to a file. There's no single import or export operation. You import with `record search`, `record create` and `record update`, and you export with `record search`.

Each source has a page on how to get its data, how its objects link together, and how its values map to Sofatofu:

- [Spreadsheets and CSV files](import-spreadsheet.md)
- [HubSpot](import-hubspot.md)
- [Pipedrive](import-pipedrive.md)
- [Salesforce](import-salesforce.md)

## Send the rows

Use code for the rows. Read the source, convert the values and build each batch with a script. Where a script can run, it writes each batch to a JSON file and passes it with `--records @<path>` (see [Use the CLI](cli.md)), so the rows stay out of your context. Through a connector, you send each batch inline, so every row passes through your context.

You don't need to read thousands of rows yourself. Counts, errors and a few sample rows are enough to check the work. If your code can't read the operation results, copy each result into the code in full, not selected values.

## Source IDs

Give every imported record a `source_id`. It links records to each other and lets a retry or a later run find what already came in. [Fields and matching](records.md#fields-and-matching) gives its format, such as `hubspot:123`, and why it must be unique for each record type. For an import:

1. Use the stable ID of the source record. Never use a name, an email or a row number.
2. When two source objects or two accounts can give the same ID to records of one Sofatofu type, add the object or account name, joined with `:`, as in `hubspot:note:123`. Ask in the import questions whether another account of the same source can come into the workspace. If so, include the account part from the first run.
3. Choose one source ID format and explain it when you ask the import questions. Use that format for every run so later imports can find the same records.
4. A record that you build from other data, such as a note of deal line items, takes the source ID of the record it came from plus a fixed label, such as `hubspot:123:line-items`. If there's no source record to use, omit `source_id` and warn the user that a later run may create a duplicate.

## Before you import

1. **Access.** Read the source with your own tools, or ask the user for an export. Sofatofu never connects to another CRM.
2. **Scope.** By default, import all companies and people, open deals, deals closed in the last 12 months, and activities from the last 12 months. Count the rows for this default and for the full data, so the user can choose.
3. **Map.** Run `workspace get` and `member list` to see the stages, custom fields, currency and members. Map each source column to an existing field from [Fields and matching](records.md#fields-and-matching) or a new custom field, converting values as needed. List any columns you will leave out and explain why. Match each source stage to a workspace stage and each source owner to a member. The [table below](#data-with-no-field-of-its-own) covers the usual cases.
4. **Questions.** Ask the user one batch of questions. Include the two scope counts, the source ID form, each loss of data, each stage, owner or currency with no match, and each new custom field.
5. **Setup.** Before you import any records, have an owner add the missing stages, fields and currency, as [Workspace configuration](workspace.md) shows.

Then clean the rows:

- **Formats.** Convert values to the formats in [Fields and matching](records.md#fields-and-matching), such as bare domains and dates with a time zone. If you can't convert a value, list the row instead of guessing.
- **Repeated rows.** Merge or remove rows with the same [strict key](records.md#fields-and-matching) before import. A repeated key rejects the whole batch. Use the [formatting rules](records.md#edge-cases-and-formatting-details) when comparing keys. Rows with the same [loose match](records.md#fields-and-matching) reject the batch with `possible_duplicate`. Merge them when they are one record. When they are different records, follow the `possible_duplicate` error message.

## Import in batches

Create companies first, then people, then deals, then activities, since each type links to the ones before it. `record create` returns `ids` in the same order as your rows, so you can record which new ID belongs to which source ID.

Import up to 200 rows at a time. For each batch, search with a filter that every row in the batch matches, then create the rows that aren't already there:

```text
record search {"type":"company","filter":{"or":[{"source_id":{"in":["hubspot:101","hubspot:102"]}},{"domain":{"in":["acme.example.com","bolt.example.com"]}}]},"limit":200}
record create {"type":"company","records":[{"name":"Acme","domain":"acme.example.com","source_id":"hubspot:101"},{"name":"Bolt","domain":"bolt.example.com","source_id":"hubspot:102"}],"reason":"HubSpot import of 2026-09-22, companies batch 3 of 12","read_id":"<read_id>"}
```

- **Choosing the filter.** Use one search for each batch, with the `or` filter from [The search before a create](../SKILL.md#the-search-before-a-create).
- **Checking `total`.** Split the batch as [The search before a create](../SKILL.md#the-search-before-a-create) describes.
- **Skipping existing records.** A stored record with the row's source ID came in before, so don't create it again. Otherwise, use the other strict keys, and compare domains as [Leading `www.`](records.md#edge-cases-and-formatting-details) describes. Without a strict key, compare the fields of the [loose match](records.md#fields-and-matching). For activities, use the activity filter in [The search before a create](../SKILL.md#the-search-before-a-create) to find possible duplicates. Check that a match describes the same event before skipping the row. Different `subject` or `body` values do not rule out a duplicate. A company with the same name but a different domain can be a different company. Its create gets `possible_duplicate`, and the error message says what to do. Use the ID of a matching record instead of creating a new one.
- **Naming the change.** The `reason` names the import and the batch, so anyone reading the change history can find all of the import's changes.
- **Saving the result.** After each batch, save its `change_id` and `ids`. You need them to report the import or undo it.

Send batches one at a time. Batches sent in parallel to the same workspace wait on each other and can fail with `write_conflict`.

A batch writes all its rows or none. After a refused batch, fix the rows and send it again. After a failure partway through an import or an unclear result, run the batch step again: its search by source ID shows which rows came in. A row with no strict key, such as a deal with no source ID, can't be matched reliably, so use the saved `ids` for it and don't send it again. When a fix also changes rows that earlier batches wrote, such as a wrong column map, undo the import, fix the map, and run it again.

## Run the import again

Match records with the same rules as the first import. Create the rows that have no match. Update only the records whose stored values differ from the mapped source values, because an update records a change even when no value changes.

Keep changes made outside the import unless the user asks to replace them. A record's `change_id` identifies its latest change. Check the reasons in `change list` to see whether that change belongs to an import run. A record with no `change_id` had its last change more than 30 days ago, so compare its `updated_at` with the time of the last run. If you can't tell which values to keep, ask the user.

## Check and report

- **Verify.** Compare the count of each type with the source, such as `record search {"type":"person","filter":{"source_id":{"contains":"hubspot:"}},"count":true}`. Show the user a few records next to their source records.
- **Report.** Tell the user how many rows you read, created, updated, skipped and refused, with the reason for each refusal. Name the data that didn't come in, and include every `change_id`. Mention anything they still need to decide. If the import stopped early, say which batches finished.
- **Keep the source.** Tell the user to keep their source export, because Sofatofu doesn't keep the data that didn't come in.

## Undo an import

Each batch is a separate change. Reverse the import's changes newest first, as [Reverse a change](changes.md#reverse-a-change) describes. Within one run, that's activities, then deals, people and companies.

If a later run updated a record, the change that created it can't be reversed. Reversing the later update restores the earlier values. To remove the record as well, use `record delete` only if the user's request includes removing it. When a change has `reversibility: "some"`, the items with `reversible: false` were changed later, by a person or by a later run. Name every record you couldn't remove in the report.

## Data with no field of its own

| Source data                                                          | In Sofatofu                                                                                                                                                                                |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| LinkedIn, industry, size, address, probability, lost reason, channel | Custom fields                                                                                                                                                                              |
| More than one email address for a person                             | The primary in `email`, the others in custom fields                                                                                                                                        |
| More than one phone number for a person                              | The first in `phone`, the others in custom fields                                                                                                                                          |
| Notes, calls, meetings, emails                                       | Activities                                                                                                                                                                                 |
| Meeting attendees, email recipients and other people on an activity  | Link up to 50 people through the activity's `people` field. If there are more, ask the user during the import questions to approve leaving the remaining people unlinked to that activity. |
| Completed tasks                                                      | Activities dated when the tasks were completed                                                                                                                                             |
| Leads                                                                | A person (and a company), with a custom field for the lead state                                                                                                                           |
| Line items on a deal                                                 | Keep the deal's own `value`, not the sum of its items. Put the items in a note, label it as an imported summary, and set its `occurred_at` to the deal's creation time.                    |
| Won and lost                                                         | Stages                                                                                                                                                                                     |
| The date the source created a record                                 | A custom date field                                                                                                                                                                        |
| Tags and lists                                                       | A `multi_select` field                                                                                                                                                                     |
| A deal owner who isn't a member                                      | A member the user chooses, with the source owner's name in a custom field                                                                                                                  |
| A link back to the source record                                     | A custom field with the source URL                                                                                                                                                         |
| An activity body over 4000 characters                                | The body without quoted replies and signatures. If it's still too long, the user chooses once for the whole import: cut it and add the link to the source, or leave the activity out.      |

## When data doesn't fit

Map source data to Sofatofu's supported record types and fields without changing its meaning. If there is no suitable mapping, explain what would be lost or changed and agree with the user before importing.

Did anything go wrong, or not come in? Let us know with `feedback send`.

## Export records

Page through `record search` with `limit` set to 200, following `next.offset` until there's no `next`, and write each page to a JSONL or CSV file. Pass the user's filter if they want only part of the data.

Each page is a separate read, so records can change while the export runs. Check that the number of rows written matches the last page's `total`, and tell the user when the export ran. Give them the file path and the row count rather than the rows themselves. If you can't write a file, say so. Give the rows in your reply only when the set is small.
