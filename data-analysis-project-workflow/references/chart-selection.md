# Chart Selection

## Choose charts by question

| Question | Preferred chart | Avoid |
|---|---|---|
| How does a metric change over time? | line chart or column-line combination | pie chart |
| Which categories or regions lead? | sorted horizontal bar chart | unsorted bars |
| How are a few categories distributed? | bar chart, pie or doughnut for 5 or fewer categories | pie chart with many categories |
| How do volume and rate move together? | column-line combination with secondary axis | putting incompatible scales on one axis |
| What is the cumulative contribution of ranked entities? | Pareto chart: bars plus cumulative line | a line without sorted bars |
| How does an outcome differ by grouped ranges? | columns plus line if one measure is a score/rate | dense scatter of labels |

## Formatting rules

1. Title the chart with the business subject, not a generic word such as "Summary".
2. Sort ranking charts from high to low unless order itself has business meaning.
3. Show units in axis titles or data labels when necessary.
4. Use clear category labels and abbreviate only when the full label is inaccessible.
5. Do not use more than two main colors unless color encodes a business grouping.
6. Highlight one important risk or priority category with a contrasting color.
7. Keep the source result table in the workbook; do not paste chart screenshots as the only output.

## One-sentence insight format

```text
The [segment] has [metric], which is [comparison]; therefore [business implication].
```

Example:

```text
Orders delivered after 21 days have the lowest average review score, suggesting that long fulfillment time is a key service-risk area.
```
