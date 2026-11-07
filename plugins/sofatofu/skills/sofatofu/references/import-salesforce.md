# Import from Salesforce

Follow [Import and export](import-export.md) for the import method. This page covers Salesforce objects, exports and field maps.

## Get the data

- **API:** use access to the Salesforce org that the user approved. Check that the org allows API access and the signed-in user has **API Enabled**. Read each object's describe for its available fields and types. Record and field permissions limit what you can read. For REST queries, follow `nextRecordsUrl` until `done` is `true`. Use Bulk API 2.0 query jobs for large objects.
- **Data Export:** in **Setup**, search for **Data Export**, select **Export Now**, choose the file encoding and the objects, then select **Start Export**. Ask the user to download every ZIP within 48 hours of the email. A new export removes the files. The ZIPs contain CSV files. Export availability depends on the edition and the time since the last export. See Salesforce's [export guide](https://trailhead.salesforce.com/content/learn/modules/lex_implementation_data_management/lex_implementation_data_export).
- **Data Loader:** use **Export All** for each object, with **Write all CSVs with UTF-8 encoding** in its settings. This includes archived activities and deleted records. Exclude rows with `IsDeleted = true` from the import.

Read the CSV files as [Spreadsheets and CSV files](import-spreadsheet.md) describes.

Get `Account`, `Contact`, `Lead`, `Opportunity`, `Task`, `Event`, `Note`, `ContentNote` and `ContentDocumentLink`. Get `User`, `OpportunityStage` and `RecordType` for owner and stage maps. Include the relation objects below when those links matter, plus `EmailMessage` for email bodies and `OpportunityLineItem` for deal products. Check which objects and fields each export contains. A report export contains only the report's selected data and may lack the IDs and links you need.

## How the objects link

| Salesforce object                     | Link fields                                                                                                                                                          |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Contact`                             | `AccountId` links to its account.                                                                                                                                    |
| `Opportunity`                         | `AccountId` links to its account. `OwnerId` links to `User`. `RecordTypeId` links to `RecordType`.                                                                   |
| `OpportunityContactRole`              | `OpportunityId` and `ContactId`, with `Role` and `IsPrimary`.                                                                                                        |
| `OpportunityLineItem`                 | `OpportunityId` links each product row to its opportunity.                                                                                                           |
| `Lead`                                | `Company` and `Website` are text, not account links. A converted lead has `ConvertedContactId`, `ConvertedAccountId` and, when applicable, `ConvertedOpportunityId`. |
| `Task`, `Event`                       | `WhoId` names a contact or lead. `WhatId` can name an account, opportunity or another object. `AccountId` is the account Salesforce derives from those links.        |
| `TaskWhoRelation`, `EventWhoRelation` | `TaskId` or `EventId`, with `RelationId`, gives the people linked through Shared Activities.                                                                         |
| `EventRelation`                       | `EventId` and `RelationId`, with `IsInvitee`, identify invitees. These can include Salesforce users.                                                                 |
| `EmailMessage`                        | `ActivityId` links the email to its task. Do not import both as separate copies of one email.                                                                        |
| `Note`                                | `ParentId` links a classic note to its record.                                                                                                                       |
| `ContentDocumentLink`                 | `ContentDocumentId` identifies the document or enhanced note. `LinkedEntityId` identifies a record, user or other sharing target. One note can have several links.   |

Use the object's `keyPrefix` from its describe to identify the type of a linked ID. Do not assume every `WhatId` names an opportunity.

A person account has `IsPersonAccount = true` and a related contact in `PersonContactId`. Treat the account and that contact as one person, not two people or a business company. Map both IDs to that person for activity and note links. An opportunity on a person account has no direct company mapping. Include it in the [import questions](import-export.md#before-you-import), together with opportunities that lack `AccountId`.

## Map to Sofatofu

Use [Data with no field of its own](import-export.md#data-with-no-field-of-its-own) for the common mappings. Salesforce custom fields end in `__c`. Read their types from the describe, not their names. A picklist maps to `select`, a multi-select picklist maps to `multi_select`, and a long text area maps to `long_text`. Query address parts such as `BillingStreet` and `BillingCity`, not the compound `BillingAddress`, for Bulk API exports.

### Company

| Salesforce value                                                                                                    | Sofatofu field or use                                                                                                                             |
| ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Business `Account.Name`                                                                                             | `name`                                                                                                                                            |
| `Account.Website`                                                                                                   | `domain`, using only the host                                                                                                                     |
| `Industry`, `Type`, `NumberOfEmployees`, `AnnualRevenue`, `AccountSource`, `Rating`, `Phone`, billing address parts | Custom fields                                                                                                                                     |
| `OwnerId`                                                                                                           | Resolve through `User` for a custom field. A company has no built-in owner.                                                                       |
| `Description`                                                                                                       | A `long_text` custom field. For a value over 10000 characters, follow the [multi-line text rule](import-export.md#data-with-no-field-of-its-own). |

### Person

| Salesforce value                                                                                                 | Sofatofu field or use                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Contact.Name` or `Lead.Name`                                                                                    | `name`                                                                                                                                                            |
| `Email`, `Phone`, `Title`                                                                                        | `email`, `phone`, `job_title`                                                                                                                                     |
| `Contact.AccountId`                                                                                              | `company`, when it names a business account                                                                                                                       |
| `Lead.Company`, `Lead.Website`                                                                                   | Data for a company linked to the person. Group by website host, then compare names where no website exists. Match groups to imported business accounts by domain. |
| `Lead.Status`                                                                                                    | Custom select field for the lead state                                                                                                                            |
| Person account `Name`, `PersonEmail`, `Phone`, `PersonTitle`                                                     | `name`, `email`, `phone`, `job_title`                                                                                                                             |
| `MobilePhone`, `HomePhone`, `OtherPhone`, `PersonMobilePhone`, `Department`, `LeadSource`, mailing address parts | Custom fields                                                                                                                                                     |

Check `IsConverted`. When it is `true`, import the contact in `ConvertedContactId`, not the lead. Link the lead's activities and notes to that person. If an earlier run imported the lead as a person, update that person's `source_id` to use the contact's ID under [Source IDs](#source-ids). Sending both people fails when they share an email. Map both Salesforce IDs to the same person and follow [Run the import again](import-export.md#run-the-import-again).

### Deal

| Salesforce value                                                              | Sofatofu field or use                                                                                                                             |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Opportunity.Name`                                                            | `name`                                                                                                                                            |
| `AccountId`                                                                   | `company`                                                                                                                                         |
| `StageName`                                                                   | `stage`, using `OpportunityStage` to identify its meaning                                                                                         |
| `Amount`                                                                      | `value`, only when the record's currency matches the workspace currency                                                                           |
| `CloseDate`                                                                   | For open opportunities (`IsClosed = false`), `expected_close_date`. For closed ones, a custom date field for the close date.                      |
| `OwnerId`                                                                     | `owner`: look up the `User` by `Id` and match its `Email` to a member. The Salesforce user can be inactive.                                       |
| `Probability`, `Type`, `LeadSource`, `ForecastCategoryName`, record type name | Custom fields                                                                                                                                     |
| `OpportunityLineItem`                                                         | Product rows for the deal's note: `Name`, `Quantity`, `UnitPrice`, `TotalPrice`. Do not add their total to `Amount`.                              |
| `Description`                                                                 | A `long_text` custom field. For a value over 10000 characters, follow the [multi-line text rule](import-export.md#data-with-no-field-of-its-own). |

`OpportunityStage` gives `IsActive`, `IsClosed`, `IsWon` and `SortOrder`. More than one stage can mean won or closed. Use these flags rather than guesses from stage names. `RecordType.BusinessProcessId` identifies the sales process, which restricts the available stages. Picklist API values can differ from displayed labels.

In a multi-currency org, read each record's `CurrencyIsoCode`. Otherwise, confirm the org currency with the user. Map `AnnualRevenue` and other money values to custom currency fields only when each value's currency matches the field's currency. Include unresolved stage, owner and currency mappings in [Before you import](import-export.md#before-you-import).

### Activity

| Salesforce value                                     | Sofatofu field or use                                                                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Closed `Task`, with `IsClosed = true`                | `kind`: `Call` in `TaskSubtype` means `call`, and `Email` or `ListEmail` means `email`. Check other subtypes and `Type` before choosing `note`. |
| `Task.CompletedDateTime`                             | `occurred_at`. `ActivityDate` is the due date, not proof of when the task finished.                                                             |
| Past `Event`                                         | `kind`: usually `meeting`. Check `Type` for calls or other events. Use `StartDateTime` for `occurred_at`.                                       |
| `Note`, `ContentNote`                                | `kind`: `note`, with the note's `CreatedDate` for `occurred_at`                                                                                 |
| `Subject`, note `Title`                              | `subject`                                                                                                                                       |
| `Task.Description`, `Event.Description`, `Note.Body` | `body`                                                                                                                                          |
| `EmailMessage.TextBody`                              | The linked email task's `body`                                                                                                                  |
| `ContentNote.Content`                                | Full note `body` in HTML, which can be base64-encoded. Decode when needed and convert to text. `TextPreview` is not the full note.              |
| `WhoId`, shared activity relations, event invitees   | `people`, using the imported contacts and leads. A Salesforce `User` ID is not a CRM person ID.                                                 |
| `AccountId`, account `WhatId`                        | `company` for a business account, or a person in `people` for a person account                                                                  |
| Opportunity `WhatId`                                 | `deal`                                                                                                                                          |
| `Note.ParentId`, enhanced note `LinkedEntityId`      | Resolve each target to its company, person or deal.                                                                                             |

If `WhatId` names an excluded object, such as a `Case`, link the activity through usable `WhoId` and `AccountId` values. An activity with no usable link cannot come in as it is. Check missing completion times and all-day events separately. They do not establish an exact event time. A note linked to several business accounts or opportunities cannot keep all those links in one activity. Handle these cases through [When data doesn't fit](import-export.md#when-data-doesnt-fit).

## Source IDs

Apply [Source IDs](import-export.md#source-ids) with `salesforce` as the source.

Use the 18-character ID with its original case. Sofatofu treats the 15- and 18-character forms of one ID as different values. Mixing them makes duplicates or breaks links. Convert record IDs and link IDs to 18 characters before matching. Never lowercase an ID.

The API, Data Loader and Data Export return 18-character IDs. Reports can give 15 characters. Salesforce provides `CASESAFEID` for conversion. See [Salesforce ID types](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/field_types.htm) and [ID conversion](https://help.salesforce.com/s/articleView?id=000385066&language=en_US&type=1).

| Source record               | Stable ID                                                    |
| --------------------------- | ------------------------------------------------------------ |
| Business account            | `Account.Id`                                                 |
| Contact or unconverted lead | `Contact.Id` or `Lead.Id`                                    |
| Person account              | `PersonContactId`, which identifies its contact              |
| Opportunity                 | `Opportunity.Id`                                             |
| Task, event or classic note | The object's `Id`                                            |
| Enhanced note               | Its `ContentDocumentId`, not a link row's ID or a version ID |

Salesforce IDs are unique within an org, across these objects. A sandbox or another org can repeat IDs. An org is an account under rule 2 of [Source IDs](import-export.md#source-ids).

A new company built from one lead comes from that lead under the derived-record rule in [Source IDs](import-export.md#source-ids). A new company that several leads describe has no single source record, so omit `source_id`, as [Source IDs for spreadsheets](import-spreadsheet.md#source-ids) describes.

A line-item note comes from its opportunity. Give it a source ID under the derived-record rule in [Source IDs](import-export.md#source-ids).

## Data without a direct mapping

Use [When data doesn't fit](import-export.md#when-data-doesnt-fit) for these Salesforce cases.

| Salesforce data                                                                                                           | What a direct copy would lose or change                                                           |
| ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Formula fields (`calculated`), roll-up summaries, `ExpectedRevenue`, scores                                               | A copied result no longer recalculates. Data Export excludes formula and roll-up summary fields.  |
| `AccountHistory`, `ContactHistory`, `OpportunityFieldHistory`, `OpportunityHistory`                                       | Source history is not Sofatofu change history.                                                    |
| Open tasks, future events, reminders, `NextStep`                                                                          | Planned work is not a completed activity.                                                         |
| `Case`, `Campaign`, `Product2`, price books, quotes, contracts, assets, custom objects                                    | No matching record type                                                                           |
| `Attachment`, non-note `ContentVersion`, email attachments, recordings                                                    | No file storage. An enhanced note's text is separate from these files.                            |
| Indirect `AccountContactRelation`, `Account.ParentId`, `Contact.ReportsToId`, `OpportunityContactRole`, opportunity teams | No matching relationship. A contact role does not become an activity participant by itself.       |
| Reports, dashboards, list views, flows, assignment rules, Sales Engagement cadences                                       | Source configuration, not CRM records                                                             |
| Multiple sales processes                                                                                                  | One stage list cannot keep separate processes. The record type name can remain in a custom field. |

## Traps

- **Archived activities:** ordinary queries leave them out. Use `queryAll` for `Task` and `Event`, and exclude `IsDeleted = true`. Check that exported relation rows also cover the archived activities you need.
- **Child events:** `IsChild = true` can mark an invitee's copy of a recurring event. Exclude child copies so one meeting does not become several activities.
- **Enhanced notes:** query `ContentDocumentLink` for the imported `LinkedEntityId` values. `ContentNote.FileType` is `SNOTE`. Do not treat every linked file as a note. Check that an export contains both note bodies and their link rows. Missing links do not mean a note was unrelated to a record.
- **Visibility:** check access to private notes and files. An empty result does not prove that none exist. Do not change source sharing settings to get them.
- **Dates:** compare exported date-times with known records in Salesforce before choosing a time zone. A date-only field has no time zone.
- **Repeated keys:** Salesforce can contain contacts and leads with the same email, or accounts with the same website. Check these groups under [Before you import](import-export.md#before-you-import).
- **Merged records:** deleted account, contact or lead rows can hold `MasterRecordId`, which identifies the survivor. Use these links to resolve old source IDs. Do not recreate the deleted records.
