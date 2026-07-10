---
name: data-analysis-project-workflow
description: Guide complete data analysis projects from public data selection through SQL analysis, Excel visualization, and business reporting. Use when a user wants to start, continue, structure, validate, or deliver an end-to-end data analysis project involving CSV, databases, SQL, spreadsheets, charts, or reports.
---

# Data Analysis Project Workflow

Guide a project from a business question to a defensible analysis report. Use the workflow as a checklist, but adapt the analyses to the actual dataset and business domain.

## Workflow

### 1. Define the analysis problem

Before querying data, write down:

```text
Business question:
Analysis object:
Time range:
Core metrics:
Expected deliverables:
```

Classify the project before selecting metrics:

| Project type | Typical questions |
|---|---|
| E-commerce operations | GMV, orders, products, regions, sellers, fulfillment |
| User behavior | PV, UV, funnel, retention, repurchase |
| Customer value | RFM, customer tiers, churn risk, consumption |
| Content or campaign operations | activity, engagement, conversion, content performance |

Do not start visualizing before the business question and metric definitions are clear.

### 2. Select and understand data

Prefer public, documented data with enough fields to answer the question. Check the dataset license, source description, time range, and whether data is sampled.

Inspect before analysis:

1. List tables or files, field names, row counts, and time range.
2. Identify the row grain: one row per order, item, user event, customer, or another entity.
3. Identify primary keys and join keys.
4. Check nulls, duplicates, unexpected categories, negative quantities or prices, and impossible dates.
5. Document filters such as delivered orders only, valid customers only, or excluded cancellations.

Never invent tables or fields. If a required dimension is unavailable, state the limitation and revise the question or use a different public dataset.

### 3. Define business metrics

Read [references/metric-catalog.md](references/metric-catalog.md) before calculating common e-commerce, user, or customer metrics.

For every metric, record:

```text
Metric name:
Formula:
Data grain:
Included records:
Excluded records:
```

Use `COUNT(DISTINCT order_id)` for order counts when an order can contain multiple detail rows. Do not assume that a payment amount, product amount, and shipping amount are interchangeable.

### 4. Build the SQL analysis

Run analyses in this order unless the business question requires a different sequence:

1. Data overview: row counts, entities, time range, status distribution.
2. Trend: day, week, or month metrics and period-over-period change.
3. Structure: region, category, behavior type, channel, or status composition.
4. Ranking: Top products, categories, customers, sellers, or regions.
5. Conversion or lifecycle: funnel, repurchase, retention, or RFM.
6. Experience or risk: delivery, reviews, cancellations, anomalies, or concentration.

Use these SQL patterns deliberately:

| Need | Preferred pattern |
|---|---|
| Connect business tables | `INNER JOIN` or `LEFT JOIN` |
| Break a complex query into steps | `WITH` / CTE |
| Apply business rules | `CASE WHEN` |
| Compare with prior period | `LAG()` |
| Rank records | `ROW_NUMBER()` |
| Rank within each month or group | `PARTITION BY` |
| Segment customers | `NTILE()` or explicit `CASE` rules |
| Compute cumulative contribution | `SUM() OVER()` |

### 5. Protect aggregation correctness

Before summing a joined table, verify join cardinality.

```text
One order -> many items
One order -> many payment rows

Joining items directly to payments can create:
item rows x payment rows

This duplicates amounts.
```

When two one-to-many detail tables must be used together, aggregate each to the same grain first, then join them:

```sql
WITH item_amount AS (
    SELECT order_id, SUM(price + freight_value) AS gmv
    FROM order_items
    GROUP BY order_id
),
payment_amount AS (
    SELECT order_id, SUM(payment_value) AS payment_value
    FROM payments
    GROUP BY order_id
)
SELECT ...
FROM orders o
JOIN item_amount i ON o.order_id = i.order_id
JOIN payment_amount p ON o.order_id = p.order_id;
```

For slow recurring joins, create indexes on join keys after import. Check the query result before optimizing further.

### 6. Interpret before charting

For each completed query, write four short lines:

```text
What does the result show?
What is the important pattern or exception?
What is a plausible business explanation?
What action could the business take?
```

Do not claim causation from a descriptive comparison alone. Use language such as "is associated with" or "suggests" when the dataset does not establish causality.

### 7. Create Excel visualizations

Read [references/chart-selection.md](references/chart-selection.md) before selecting chart types.

Create a separate data sheet for each analysis result. Use a dashboard or summary sheet only after the source tables and individual charts are correct.

Chart rules:

- Use readable titles that state the analysis topic.
- Use one chart to answer one business question.
- Use a secondary axis only for comparable time-series measures with different scales.
- Keep category labels readable; prefer horizontal bars for long names.
- Remove redundant legends and labels.
- Add a one-sentence insight below each chart.

### 8. Write the business report

Read [references/report-outline.md](references/report-outline.md) before drafting.

Use evidence-first writing:

```text
Data result -> business interpretation -> recommended action
```

Include only charts that support a conclusion. Keep metric definitions and data limitations explicit. For document or spreadsheet creation, use the applicable document or spreadsheet workflow and verify visual layout before delivery.

### 9. Final quality checklist

Complete this checklist before calling a project finished:

```text
[ ] Business question and time range are stated.
[ ] Data grain and join keys are documented.
[ ] Metric formulas and filters are clear.
[ ] Multi-table aggregates do not duplicate values.
[ ] Results reconcile with the source tables.
[ ] Charts have titles, readable axes, and useful labels.
[ ] Every chart has a written takeaway.
[ ] Recommendations are supported by analysis results.
[ ] Report numbers match SQL and spreadsheet outputs.
[ ] Original raw data remains unchanged.
```

## Reference routing

- Read [references/metric-catalog.md](references/metric-catalog.md) for metric formulas and scope notes.
- Read [references/chart-selection.md](references/chart-selection.md) when preparing Excel visuals.
- Read [references/report-outline.md](references/report-outline.md) when writing a formal analysis report.
