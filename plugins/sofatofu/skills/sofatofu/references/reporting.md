# Reporting and analysis

Use this guide for summaries, comparisons, trends and charts. Follow [Talking to the user](../SKILL.md#talking-to-the-user) for tone and general reporting rules.

Retrieve data with the operations in [Records](records.md) and [Filters and pipeline queries](filters.md), then calculate and present the answer. Sofatofu returns records and counts; don't assume it provides grouped statistics or charts.

## Choose the presentation

Lead with the finding, not a dump of the records. Use the simplest format that answers the question:

| Question | Useful result | Presentation |
| --- | --- | --- |
| How many deals match? | The returned count and its scope | A sentence |
| What needs attention? | Records with the relevant issue, such as no owner or no recent activity | A short, prioritised table |
| How is the pipeline distributed? | Deal count and known value by stage or owner | A comparison table or bar chart |
| What changed over time? | Comparable measurements for each period | A table, line chart or column chart |
| Which records explain the result? | The relevant records and their IDs | A detail table |

## Make a chart

Use a chart when it shows a comparison or trend more clearly than a table. Use the same figures in the chart and the text.

- **Use Sofatofu's colours unless the user asks for others.** Lilac `#EEEBFF` for the background, Cream `#FFFDF8` for the chart area, Ink `#2D2C4E` for titles and labels, Muted `#595A78` for supporting text, and Coral `#FF806B` for the main series. Add Purple `#372986` and Mint `#BEE8D6` for more series. Don't rely on colour alone: label each series.
- **Label everything.** Give the chart a title, label the axes, periods and units, and show the value on each bar or point when space allows. Say in the chart when the data is partial or illustrative.
- **Check the result.** If you can view the chart, look at it before presenting it. Fix cropped, overlapping or unreadable labels.
- **Share the chart.** Save it with a descriptive filename, such as `open-deals-by-stage.svg`. Attach the file or give the user its full path. Display it if the interface supports images. If you cannot create the chart or the user cannot open the file, explain why and use a table.

Always include the key figures in text or a table next to the chart.

Keep reports short. Include enough detail to inspect the finding, not every available field. Suggest next actions when useful, but don't change CRM data unless the request covers those actions.

## Calculate from retrieved data

Every figure must be exact and come from the records and counts you retrieved. Use returned counts directly. Use calculation tools for totals, averages, percentages, and other derived figures. Do not rely on memory or mental arithmetic, estimate results, or fill in missing values.

- **Keep figures traceable.** Use only results from operations that succeeded, and work from the full results, not values that you copied by hand. Keep the query scope, the method and the record IDs behind each figure, so you can show where it came from.
- **Handle money precisely.** Deal values arrive as numbers. Round only for presentation. Read the workspace currency with `workspace get`; custom currency fields have their own currency. Don't add amounts in different currencies without an explicit conversion basis.
- **Separate missing from zero.** A missing value is unknown, not zero. Report how many records lack values. For an average over known values, use only those records in the denominator and say so.
- **Define rates.** State the numerator, denominator and period. A zero denominator means the rate isn't available, not that it is zero.
- **Check the output.** Verify row counts and group totals against the source. Deduplicate records by type and ID when combining overlapping searches.

Report only figures you can verify. If you cannot answer the question, give the verified records and counts, explain which figures are missing and why, and say what you need to produce them.

## Define the scope and check completeness

State the workspace or relevant filters, time period, units and currency. For time-based reports, name the date field used and the timezone where it affects the boundaries. Deals expected to close in a month are not the same set as deals actually won in that month.

Use metric definitions from the request or established workspace context. If an unresolved definition would materially change the answer, ask. Don't silently invent which custom stages count as open, won or lost.

Calculate over the full matching set, not just the first page:

- Check `total` and follow `next.offset` while more results remain. `items.length` counts only the returned page.
- If only a count is needed, use `record search` with `count` set to `true`. Counts don't provide the values needed for sums or averages.
- Separate reads return current data, not a shared snapshot. If records change during collection, reconcile the results or disclose the inconsistency. Deduplication alone doesn't prove that no records were missed.
- If you can't retrieve the complete set, label the analysis as partial and state what is missing. Don't extrapolate a partial sum, average or rate to the whole dataset.

## Respect what the data can establish

Distinguish observations from interpretations and estimates. A useful chart doesn't make an unsupported metric valid.

- **Pipeline snapshots aren't conversion funnels.** Current deal counts by stage show today's distribution, not how many deals progressed between stages.
- **Trends and durations need history.** Stage conversion, time in stage, historical pipeline value and period-specific win rates require the relevant events or historical observations. Current records alone aren't enough. Change history has limited retention; see [Change history and reversal](changes.md). Don't assume it covers the requested period.
- **Expected close dates aren't actual outcomes.** Label totals based on `expected_close_date` as scheduled or expected pipeline value, not revenue earned or a probability-weighted forecast. Don't invent stage probabilities.
- **Activity measures need a defined link.** `last_activity` counts only active activities linked directly to the deal. It doesn't include activity linked only to a company or person, and it doesn't prove that no contact happened outside the CRM.

## Example outputs

These examples show the shape of a report. Their values are illustrative, not data from a workspace.

A complete pipeline summary:

> **Open pipeline by stage.** All 42 open deals in `lead` or `in_progress`. Values in EUR.
>
> | Stage | Deals | Known value | Deals with no value |
> | --- | --- | --- | --- |
> | `lead` | 18 | 54,000 | 3 |
> | `in_progress` | 24 | 186,500 | 2 |
> | Total | 42 | 240,500 | 5 |
>
> The total covers 37 deals with recorded values. Five deals have no recorded value.

A partial result:

> **Partial: deals expected to close in September 2026**
>
> I retrieved 200 of 236 open deals with an expected close date in September 2026. Their recorded values total EUR 410,000. The next page failed to load, leaving 36 deals unread, so this is not the full total. I can retry that page.
