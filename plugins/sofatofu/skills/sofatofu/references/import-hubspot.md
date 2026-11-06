# Import from HubSpot

Follow [Import and export](import-export.md) for the import method. This page gives the HubSpot objects, links and field maps.

## Get the data

For API reads, use an existing HubSpot private app token or a [service key](https://developers.hubspot.com/docs/apps/developer-platform/build-apps/authentication/account-service-keys). Give it only the read access needed for the objects you will import. The [API reference](https://developers.hubspot.com/docs/api-reference/legacy/crm/understanding-the-crm) lists the required scopes for each endpoint.

Read these endpoints at `https://api.hubapi.com`:

| Data                    | Endpoint and use                                                                                                                                                                                                                                                              |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Account settings        | [`GET /account-info/v3/details`](https://developers.hubspot.com/docs/api-reference/legacy/account/account-information/get-account-details). Read `timeZone` for date conversion and `companyCurrency` for the home currency.                                                  |
| Property definitions    | `GET /crm/v3/properties/{object}`. Read the internal names, labels, types and options before mapping values.                                                                                                                                                                  |
| Records                 | `GET /crm/v3/objects/{object}` for `companies`, `contacts`, `deals`, `notes`, `calls`, `meetings`, `emails`, `tasks`, `line_items` and, if used, `leads`. Request the properties on your map, use `limit=100`, and pass `paging.next.after` as `after` until no page remains. |
| Associations            | `POST /crm/v4/associations/{from}/{to}/batch/read`, with up to 1,000 source IDs. Each source record can have its own `paging.next.after`. Send that record's `id` with `after` to read its remaining links.                                                                   |
| Deal and lead pipelines | `GET /crm/v3/pipelines/{objectType}`. Use the [object type](https://developers.hubspot.com/docs/api-reference/legacy/crm/understanding-the-crm#object-type-ids) for each. Keep pipeline and stage IDs with their labels and stage metadata.                                   |
| Owners                  | `GET /crm/v3/owners`. Read both active owners and `archived=true` for owners who left. Follow paging for both.                                                                                                                                                                |

HubSpot search stops at 10,000 results per query. Use the record list endpoints for a full extract. If the import includes lists, read their [memberships](https://developers.hubspot.com/docs/api-reference/legacy/crm/lists/guide), not only their names.

For files, follow HubSpot's [record export guide](https://knowledge.hubspot.com/import-and-export/export-records). Open the all-records list view of Contacts, Companies and Deals, and export each separately. Choose CSV, English column headers, **All properties and associations on records**, and **All associated records**. The default association setting stops at 1,000 IDs per column. Include all email addresses and all company domains. Read every file if the export arrives as a ZIP. Read the CSV files with the rules in [Get the data](import-spreadsheet.md#get-the-data) and [Traps](import-spreadsheet.md#traps).

Calls and [tasks](https://knowledge.hubspot.com/tasks/export-tasks) also have exports. For notes, meetings and emails, use the activity APIs or check which activity reports the account can export. Check that the files contain bodies, dates, record IDs and association IDs. A CRM backup is not a substitute: it omits associations and activity data.

Check the header row and sample cells for association column names, separators, primary-company labels, and whether values are labels or internal values.

For a file import, also read account settings, property definitions, pipelines and owners through the API when possible. Without API access, read the account's time zone and home currency from its settings and request a [property export](https://knowledge.hubspot.com/properties/organize-and-export-properties). In the import questions, resolve missing property definitions and confirm the account's time zone and home currency, which pipeline each stage label belongs to, which stages mean won or lost, and which member each owner is. Use that confirmed map when the files have labels instead of IDs.

## How the objects link

HubSpot stores links as [associations](https://developers.hubspot.com/docs/api-reference/legacy/crm/associations/associate-records/guide). Use their IDs, not record names.

| HubSpot association                       | Sofatofu link                                                                                                                                  |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Contact to company                        | Person `company`. HubSpot type `1` marks the primary company. Type `279` is an unlabeled company link.                                         |
| Deal to company                           | Deal `company`. Type `5` marks the primary company. Type `341` is an unlabeled company link.                                                   |
| Activity to contacts, companies and deals | Activity `people`, `company` and `deal`. Keep the direct associations rather than inferring links from the timeline where an activity appears. |
| Deal to line items                        | The items for a summary note on the deal.                                                                                                      |
| Lead to contact or company                | The record whose lead state you map. Types `578` and `580` mark the primary contact and primary company.                                       |

Use the primary company when present, or the sole company association. For multiple companies without a primary, a deal without a company, or an activity with multiple deals or companies, resolve the link choice under [Before you import](import-export.md#before-you-import). A contact's `company` property is free text, not a company association.

## Map to Sofatofu

These tables use API property names. Use the property definitions to match export labels. [Fields and matching](records.md#fields-and-matching) gives the target formats. Apply [Data with no field of its own](import-export.md#data-with-no-field-of-its-own) to lists, source dates and additional properties such as `hs_additional_emails`. The API's `createdAt` is the source creation time, and `url`, when present, is the record link. For company, contact and activity `hubspot_owner_id`, keep the resolved owner name in a custom field.

### Company

| HubSpot value                                                                                                            | Sofatofu field       | Mapping                                |
| ------------------------------------------------------------------------------------------------------------------------ | -------------------- | -------------------------------------- |
| `name`                                                                                                                   | `name`               |                                        |
| `domain`                                                                                                                 | `domain`             |                                        |
| `hs_additional_domains`, `website`                                                                                       | Custom fields        | Keep separate from the primary domain. |
| `industry`, `numberofemployees`, `annualrevenue`, `phone`, `address`, `city`, `country`, `linkedin_company_page`, `type` | Custom fields        | Use the matching type.                 |
| `lifecyclestage`, `hs_lead_status`                                                                                       | Custom select fields | Use the property's option labels.      |

### Person

| HubSpot value                                      | Sofatofu field              | Mapping                                                                            |
| -------------------------------------------------- | --------------------------- | ---------------------------------------------------------------------------------- |
| `firstname`, `lastname`                            | `name`                      | Join the non-empty parts. List contacts with neither part in the import questions. |
| `email`                                            | `email`                     |                                                                                    |
| `phone`, `mobilephone`                             | `phone`                     | Use `phone`, or `mobilephone` when `phone` is empty.                               |
| `jobtitle`                                         | `job_title`                 |                                                                                    |
| Primary company association                        | `company`                   |                                                                                    |
| `lifecyclestage`, `hs_lead_status`                 | Custom select fields        | Use the property's option labels.                                                  |
| `hs_linkedin_url`, `city`, `country`, `hs_persona` | Custom fields               |                                                                                    |
| `hs_buying_role`                                   | Custom `multi_select` field |                                                                                    |

HubSpot's `leads` object is separate from `lifecyclestage` and `hs_lead_status`. Read each lead's primary associations and map `hs_pipeline_stage` through the pipeline in `hs_pipeline`. Keep the lead state on the linked person or company, not a second copy of that record. Resolve multiple leads for one record, or a lead with no usable association, under [Before you import](import-export.md#before-you-import).

### Deal

| HubSpot value                                                        | Sofatofu field                        | Mapping                                                                                                                                                                 |
| -------------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dealname`                                                           | `name`                                |                                                                                                                                                                         |
| Primary company association                                          | `company`                             |                                                                                                                                                                         |
| `pipeline` and `dealstage`                                           | `stage`                               | Map the pair of internal IDs, or the confirmed pipeline and stage labels. Keep the pipeline label in a custom select field when the account has more than one pipeline. |
| `amount`                                                             | `value`                               | Check `deal_currency_code` against the workspace currency.                                                                                                              |
| `closedate`                                                          | `expected_close_date` or custom field | Use `expected_close_date` for open deals and a custom date field for the actual close date of closed deals. See [Traps](#traps) for the time zone.                      |
| `hubspot_owner_id`                                                   | `owner`                               | Resolve an ID through the HubSpot owner's `id`, not `userId`, then match the owner's email to a member. For file labels, use the confirmed owner map.                   |
| `closed_lost_reason`, `closed_won_reason`, `dealtype`, `hs_priority` | Custom fields                         |                                                                                                                                                                         |

Use stage metadata when available, not stage labels, to identify outcomes. HubSpot defines stage probability `0.0` as lost and `1.0` as won. Two pipelines can have stages with the same label, so keep each pipeline and stage pair separate while mapping.

For the line-item note in [Data with no field of its own](import-export.md#data-with-no-field-of-its-own), read the deal's associated `line_items`: `name`, `quantity`, `price`, `amount` and `hs_line_item_currency_code`. The deal's creation time is `createdAt`.

### Activity

| HubSpot object                               | `kind`                                                                                     | `occurred_at`                                                                                                                                         | `subject` and `body`                                                                                                     |
| -------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `notes`                                      | `note`                                                                                     | `hs_timestamp`                                                                                                                                        | No subject property. Body from `hs_note_body`.                                                                           |
| `calls`                                      | `call`                                                                                     | `hs_timestamp`                                                                                                                                        | `hs_call_title`, `hs_call_body`                                                                                          |
| `meetings`                                   | `meeting`                                                                                  | `hs_meeting_start_time`                                                                                                                               | `hs_meeting_title`, `hs_meeting_body`. Append `hs_internal_meeting_notes` to the body after the label `Internal notes:`. |
| `emails`                                     | `email`                                                                                    | `hs_timestamp`                                                                                                                                        | `hs_email_subject`, `hs_email_text`. If the text is empty, convert `hs_email_html` to text.                              |
| `tasks` with `hs_task_status` of `COMPLETED` | From `hs_task_type`: `CALL` becomes `call`, `EMAIL` becomes `email`, `TODO` becomes `note` | The **Completed at** property. Find its internal name in the account's property definitions. `hs_timestamp` is the due date, not the completion time. | `hs_task_subject`, `hs_task_body`                                                                                        |

Check `hs_call_status`, `hs_meeting_outcome` and `hs_email_status` so a failed call, canceled meeting or failed send does not look completed. Keep the outcome in a custom field or in the activity text. `hs_call_duration` is in milliseconds. `hs_call_disposition` holds a GUID. Resolve it to a label through HubSpot's [call outcome documentation](https://developers.hubspot.com/docs/api-reference/legacy/crm/activities/calls/guide).

## Source IDs

Apply the general [Source IDs](import-export.md#source-ids) rules with these facts:

- HubSpot returns the stable record ID as `id`, also held in `hs_object_id`. CSV exports call it **Record ID**. Keep it as text for companies, contacts, deals and each activity object.
- HubSpot record IDs are unique only within an object. Notes, calls, meetings, emails and completed tasks all become Sofatofu activities, so add the object name to their source IDs under the general rules.
- A line-item summary note comes from a deal, not from a HubSpot note. Its source record is that deal. A lead state copied onto a person or company does not create a separate record or replace that record's source ID.

## What does not come in

The data below does not come in. List it in the import questions and the report, as [When data doesn't fit](import-export.md#when-data-doesnt-fit) requires.

| HubSpot data                                                                                          | What it represents                                                                      |
| ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Calculated properties (`calculation_equation`, `calculation_rollup`), scores and time-in-stage values | Formula results, totals and time-in-stage figures, not the rules that keep them current |
| `propertiesWithHistory`, **Date entered** and **Date exited** stage properties, pipeline audit        | HubSpot's history, not Sofatofu changes                                                 |
| Incomplete tasks, `hs_task_reminders`, future meetings, scheduled emails                              | Planned work, not completed activity                                                    |
| `hs_attachment_ids`, `hs_call_recording_url`                                                          | Files and recordings, not activity text                                                 |
| Parent-company associations, secondary company links, deal-to-contact associations                    | Links with no matching Sofatofu field                                                   |

## Traps

- **Property types.** API values can be strings even for numbers and booleans. Use the property definition to convert them. HubSpot's `checkbox` field type means multiple checkboxes, with semicolon-separated internal option names. Map each name to its label in a Sofatofu `multi_select` field, not a `checkbox` field.
- **Currency.** `amount` is in the deal's currency. If `deal_currency_code` is empty or absent, check `companyCurrency` (or the confirmed home currency) and source data rather than assuming a currency. `amount_in_home_currency` uses HubSpot's exchange rates. It is not interchangeable with `amount`.
- **Dates.** HubSpot [`date` values](https://developers.hubspot.com/docs/api-reference/legacy/crm/properties/guide#add-values-to-date-and-datetime-properties) represent a calendar day. Do not shift them by a time zone. `datetime` values are UTC. When a date-time such as `closedate` or `createdAt` becomes a Sofatofu date, convert it to the HubSpot account's time zone before removing the time. Compare a few `closedate` values with the HubSpot app. HubSpot can [set `closedate` when a deal closes](https://knowledge.hubspot.com/properties/hubspots-default-deal-properties).
- **Rich text.** Check bodies for HTML, convert it to text and decode entities. Then apply the body rules in [Data with no field of its own](import-export.md#data-with-no-field-of-its-own).
- **Merges.** A [HubSpot merge](https://knowledge.hubspot.com/records/merge-records) can change the record ID. `hs_merged_object_ids` lists earlier IDs. Check those IDs for existing import matches before treating the result as new. If one imported record holds an earlier ID, update that record and set its `source_id` to the new ID under the general rules. If several imported records match, resolve them with the user.
