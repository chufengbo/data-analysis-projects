# Metric Catalog

Use the definitions that fit the dataset. State any deviations in the report.

## E-commerce operations

| Metric | Formula | Note |
|---|---|---|
| GMV | `SUM(product_amount + shipping_amount)` | Confirm whether shipping is included. |
| Order count | `COUNT(DISTINCT order_id)` | Use distinct order IDs when rows are item-level. |
| Customer count | `COUNT(DISTINCT customer_id)` | Prefer a stable customer identifier if the order customer ID changes per order. |
| Average order value | `GMV / order_count` | Use the same filter for numerator and denominator. |
| Delivery days | delivery timestamp minus purchase timestamp | Exclude null or invalid timestamps. |
| On-time delivery rate | on-time delivered orders / delivered orders | Define on-time against promised date. |

## User behavior

| Metric | Formula | Note |
|---|---|---|
| PV | count of page-view events | Event-level metric. |
| UV | `COUNT(DISTINCT user_id)` | User-level metric. |
| Funnel rate | next-stage count / prior-stage count | State whether it is event-level or user-level. |
| Daily conversion | daily buyers / daily visitors or PV | Use the appropriate denominator for the business question. |
| Retention | users active on future day / cohort users | Do not interpret incomplete cohorts as low retention. |
| Repurchase rate | buyers with at least 2 orders / buyers | Count orders at the customer grain. |

## Customer value

| Component | Meaning | Typical direction |
|---|---|---|
| R | Days since last purchase | lower is better |
| F | Number of distinct purchase orders | higher is better |
| M | Total valid purchase amount | higher is better |

For discrete and heavily tied frequency values, prefer explicit `CASE WHEN` scoring over `NTILE()`. `NTILE()` can assign different scores to customers with the same purchase count.

## Data-quality notes

- Define valid order statuses before all revenue and customer metrics.
- Exclude cancellations, returns, tests, or abnormal quantities only when the business rule is documented.
- Preserve raw values in a separate layer; create analysis fields instead of overwriting original data.
