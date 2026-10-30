# Import and export

This page covers moving records into Sofatofu from a spreadsheet or another CRM's export, and writing records out to a file. There's no single import or export operation. You import with `record search` and `record create`, and you export with `record search`.

Use code for the rows. Read the file, convert the values and build each batch with a script, then pass the batch as `records`. If your code can't read the operation results, copy each result into the code in full, not selected values. You don't need to read thousands of rows yourself. Counts, errors and a few sample rows are enough to check the work.

## Before you import

Run `workspace get` and `member list` first. They show the stages, custom fields and currency the workspace uses, and the user ID of each member who could own a deal.

Match each column in the file to a field from [Fields and matching](records.md#fields-and-matching), each source stage to a workspace stage, and each source owner to a member. Leave out columns that have no match. If a column's meaning is unclear, or the file needs a stage, field or currency the workspace doesn't have, ask the user. Only an owner can add them (see [Workspace configuration](workspace.md)).

Then clean the rows:

- **Formats.** Convert values to the formats in [Fields and matching](records.md#fields-and-matching), such as bare domains and dates with a time zone. If you can't convert a value, list the row instead of guessing.
- **Repeated rows.** Merge or remove rows with the same [strict key](records.md#fields-and-matching) before import. A repeated key rejects the whole batch. Use the [formatting rules](records.md#edge-cases-and-formatting-details) when comparing keys. Rows with the same [loose match](records.md#fields-and-matching) reject the batch with `possible_duplicate`. Merge them when they are one record. When they are different records, follow the `possible_duplicate` rule in [When an operation fails](../SKILL.md#when-an-operation-fails).

## Import in batches

Create companies first, then people, then deals, then activities, since each type links to the ones before it. `record create` returns `ids` in the same order as your rows, so you can record which new ID belongs to which row in the file.

Import up to 200 rows at a time. For each batch, search with a filter that every row in the batch matches, then create the rows that aren't already there:

```text
record search {"type":"company","filter":{"domain":{"in":["acme.example.com","bolt.example.com"]}},"include":"deleted","limit":200}
record create {"type":"company","records":[{"name":"Acme","domain":"acme.example.com"},{"name":"Bolt","domain":"bolt.example.com"}],"reason":"Import companies from the CRM export, batch 3 of 12","read_id":"<read_id>"}
```

- **Choosing the filter.** Use one search for each batch, with the `or` filter from [The search before a create](../SKILL.md#the-search-before-a-create).
- **Checking `total`.** Split the batch as [The search before a create](../SKILL.md#the-search-before-a-create) describes.
- **Skipping existing records.** For import matching, use the strict key when present, and compare domains as [Leading `www.`](records.md#edge-cases-and-formatting-details) describes. Without a strict key, compare the fields of the [loose match](records.md#fields-and-matching). For activities, use the activity filter in [The search before a create](../SKILL.md#the-search-before-a-create) to find possible duplicates. Check that a match describes the same event before skipping the row. Different `subject` or `body` values do not rule out a duplicate. A company with the same name but a different domain can be a different company. Its create gets `possible_duplicate`, and the rule in [When an operation fails](../SKILL.md#when-an-operation-fails) applies. Use the ID of a matching active record instead of creating a new one. For a matching record with `deleted_at`, follow [The search before a create](../SKILL.md#the-search-before-a-create).
- **Saving the result.** After each batch, save its `change_id` and `ids`. You need them to report the import or undo it.

If an import stops partway, continue from the saved `change_id` and `ids` of the batches that finished. Don't run those batches again, because some rows never match a stored record (see [Fields and matching](records.md#fields-and-matching)). For activities, also follow the retry advice in [Log an activity](records.md#log-an-activity).

## Undo an import

Each batch is a separate change, so to undo an import you reverse each batch's `change_id`. Start with the last type you imported: activities, then deals, people and companies. A company can't be removed while other records still link to it. Before each reversal, read that batch's records with `record get`, as [Reverse a change](changes.md#reverse-a-change) explains.

## Report the import

Tell the user how many rows you read, how many you created, how many already existed, and which rows you couldn't import and why. Include every `change_id`, and mention anything they still need to decide, such as a stage that had no match. If the import stopped early, say which batches finished.

## Export records

Page through `record search` with `limit` set to 200, following `next.offset` until there's no `next`, and write each page to a JSONL or CSV file. Pass the user's filter if they want only part of the data. Set `include` to `deleted` only if they ask for deleted records.

Each page is a separate read, so records can change while the export runs. Check that the number of rows written matches the last page's `total`, and tell the user when the export ran. Give them the file path and the row count rather than the rows themselves. If you can't write a file, say so. Give the rows in your reply only when the set is small.
