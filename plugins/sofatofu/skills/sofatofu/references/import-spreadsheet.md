# Import from spreadsheets and CSV files

Use [Import and export](import-export.md) for the import method and [Send the rows](import-export.md#send-the-rows) for processing the files.

## Get the data

Prefer a workbook in `.xlsx` format when the data spans several tabs. It keeps the tabs, cell formats and formulas available for inspection.

- **Excel.** Use **File** > **Save As** or **Save a Copy** to save an `.xlsx` workbook. For CSV, save each needed tab separately. Excel's [CSV export](https://support.microsoft.com/en-us/excel/excel-formatting-and-features-that-are-not-transferred-to-other-file-formats) saves only the active sheet, with values as displayed. Select **CSV UTF-8** if the app offers it.
- **Google Sheets download.** Use **File** > **Download** and choose the Excel format for a workbook. If the user gives you CSV files, check which tab each file contains and whether any needed tabs are missing. The [Drive API export](https://developers.google.com/workspace/drive/api/guides/ref-export-formats) supports `.xlsx`, but its CSV and TSV exports contain only the first sheet.
- **Google Sheets API.** If you can read the user's Google Sheets, read each needed tab through `spreadsheets.values.get`, naming the tab in the range. Use [`UNFORMATTED_VALUE`](https://developers.google.com/workspace/sheets/api/reference/rest/v4/ValueRenderOption) for numbers without display formats. This option returns dates as serial numbers by default. Also read with `FORMATTED_VALUE` to see which values display as dates. Read with `FORMULA` to identify calculated cells.

Read an original CSV directly instead of opening and saving it in Excel first. Excel can change IDs and dates when it opens a CSV. Use a CSV parser: [quoted fields](https://www.rfc-editor.org/rfc/rfc4180#section-2) can contain commas, quotes and line breaks.

## How the rows link

Inspect the headers and sample rows to find what one row represents. A tab name alone does not prove its record type.

| Layout                                            | How to read it                                                                                                                                                                                                 |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Separate company, contact, deal and activity tabs | Find the columns that refer to other tabs. Join by a shared ID where available, then by company domain or person email. Use names only after checking the possible matches.                                    |
| Contact rows with company columns                 | Each contact row describes a person. Several rows can describe the same company. Group the company data by company ID or domain, then compare names where neither exists. Do not make one company per contact. |
| Deal columns on contact rows                      | Several contacts can refer to one deal. Check the deal ID and company before treating each row as a separate deal.                                                                                             |
| Notes in cells on a contact, company or deal row  | A filled notes cell can become one note linked to that row's record. Identify which date belongs to the note. The row's creation date does not prove when the note was written.                                |
| Activity log                                      | Each event row links through its contact, company or deal columns.                                                                                                                                             |

A person email domain does not prove the person's company. Use the company columns. Keep missing or ambiguous links, conflicting company values and notes with no known date for the [import questions](import-export.md#before-you-import).

## Map to Sofatofu

These are common column names, not a fixed sheet format. Check each column's meaning. Use [Fields and matching](records.md#fields-and-matching) for accepted values and [Data with no field of its own](import-export.md#data-with-no-field-of-its-own) for other columns.

### Company

| Column                                       | Sofatofu field | Mapping                                                                               |
| -------------------------------------------- | -------------- | ------------------------------------------------------------------------------------- |
| Company, Company name, Account, Organization | `name`         | Use the company name, not the contact name.                                           |
| Website, Domain, Company URL                 | `domain`       | Take the host of the company's website, not a social profile or shared directory URL. |

### Person

| Column                                    | Sofatofu field | Mapping                                                                                  |
| ----------------------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| Name, Full name, First name and Last name | `name`         | Use the full name, or join the name parts in the order the person's name uses.           |
| Email, E-mail                             | `email`        | A cell can hold several addresses. Split them as **Lists inside cells** below describes. |
| Phone, Mobile, Tel                        | `phone`        | Keep leading zeros and `+`.                                                              |
| Title, Job title, Position                | `job_title`    | Check that Title means a job, not a salutation.                                          |
| Contact ID, Person ID, Customer number    | `source_id`    | Check that the ID identifies a person, then apply [Source IDs](#source-ids).             |

### Deal

| Column                       | Sofatofu field        | Mapping                                                                             |
| ---------------------------- | --------------------- | ----------------------------------------------------------------------------------- |
| Deal, Deal name, Opportunity | `name`                | Do not use the contact name unless it is also the deal's name.                      |
| Company, Account, Company ID | `company`             | Resolve the company reference. A contact alone does not identify the company.       |
| Stage, Status                | `stage`               | Check that the column describes the deal, not the contact's lead state.             |
| Amount, Value, Deal size     | `value`               | Check the decimal mark and currency. A `$` sign alone does not identify a currency. |
| Expected close, Close date   | `expected_close_date` | Use only an expected close date, not an actual one.                                 |

### Activity

| Column                    | Sofatofu field | Mapping                                                           |
| ------------------------- | -------------- | ----------------------------------------------------------------- |
| Type, Kind                | `kind`         | Identify whether the event is a call, meeting, email or note.     |
| Date, Time, Date and time | `occurred_at`  | Use the event time, not a follow-up date or the file export time. |

## Source IDs

Use an ID column under the [general source ID rules](import-export.md#source-ids). Check what each ID column identifies before using it.

A stable ID stays with the same record after edits, sorting, row insertion, deletion and export. It does not change when the record's name changes, and the sheet never assigns it to a different record. A row position or a formula such as `ROW()` is not stable. A column called ID is not proof of stability. One file cannot prove stability. If you cannot tell whether an ID is stable, ask the user in the import questions. Import a row with a blank ID without `source_id`. Do not use a column that gives one ID to different records.

An ID that the sheet assigns itself is unique only within that sheet, and a sheet the user imports later can reuse it. Include a name that identifies the sheet and tab in every such source ID under the general rules. Before the first run of a new sheet, check each Sofatofu type you will import for earlier use of that name. For people, use `record search {"type":"person","filter":{"source_id":{"contains":"<name>:"}},"count":true}`. Choose a different name if another source already uses it. Keep the name the same between runs. Copies or filtered tabs of the same source records are not separate ID sources. Do not give one record different source IDs because it appears on two tabs.

If the ID came from a CRM that has its own page in [Import and export](import-export.md), use that page to identify the source object and its ID. The spreadsheet is the export format, not a different source.

A deal or note built from one row's columns takes that row's source ID plus a fixed label under the general rules. A deal that several rows describe has no single source row. Use a deal ID column or import it without `source_id`.

When no stable ID exists, omit `source_id` under rule 4 of the [general source ID rules](import-export.md#source-ids). Match those rows as [Import in batches](import-export.md#import-in-batches) describes.

## Traps

- **Headers and merged cells.** Locate the header row, including any grouped headings. Separate blank rows, subtotals and summaries from record rows. Inspect merged ranges instead of treating every empty cell as missing data. Confirm what a merged value means before filling it into other rows.
- **Encoding and separators.** Check the CSV encoding and delimiter, then check that parsed rows match the headers. Excel's separator can depend on its region settings. Do not assume a Google Sheets CSV uses a particular encoding or separator. Remove a byte order mark at the start of the file. Do not drop or replace other characters to fix the encoding. If text is damaged, request the original workbook or a fresh UTF-8 export.
- **Display formats.** Excel CSV exports use displayed values, so rounding can remove decimals. Use workbook values when exact amounts matter. For text amounts, check the column's decimal and thousands separators. `1,200` can mean 1200 or 1.2. Keep unclear values for the import questions.
- **Dates and time zones.** Check the sheet's date order and time zone. `03/04/2026` does not identify a date order. Google Sheets keeps locale and time zone in **File** > **Settings**. For numeric dates, read the workbook's [1900 or 1904 date system](https://support.microsoft.com/en-us/excel/date-systems-in-excel). Sheets API [serial dates](https://developers.google.com/workspace/sheets/api/reference/rest/v4/DateTimeRenderOption) count days from 30 December 1899. `occurred_at` needs a time and `Z` or an offset. Do not guess a time or time zone. List activities with only a date for the import questions.
- **IDs and phone numbers.** Excel can [remove leading zeros and lose digits after the fifteenth significant digit](https://support.microsoft.com/en-us/excel/keeping-leading-zeros-and-large-numbers). Read these columns as text. Text conversion cannot restore lost digits. Request the original values instead of guessing them, and do not use a damaged ID as a source ID.
- **Lists inside cells.** The separator between attendees, tags or email addresses can differ from the CSV delimiter. Check it before splitting values. A name can contain a comma, such as `Smith, Ann`.
- **Formulas and sheet features.** Inspect formulas in the workbook or through the Sheets API. A CSV of calculated values does not show which cells were formulas. In the import questions, ask which columns are calculated and what each color means. Do not import summary or pivot tabs. Add each color's meaning as a column in your script's data, not the user's sheet. Handle charts, images and other data with no suitable mapping through [When data doesn't fit](import-export.md#when-data-doesnt-fit).
