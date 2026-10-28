# Expected n8n Data Tables (fictional sample rows)

## Table 1: `tracking_index` (written by workflow-tracking-index.json, read by workflow-alerts.json)

| tracking_number | order_number | order_name | email | carrier |
|---|---|---|---|---|
| TRK000000001 | 1000000000001 | #1001 | customer@example.com | Example Carrier |
| TRK000000002 | 1000000000002 | #1002 | buyer@example.com | Example Express |

All columns are strings. `tracking_number` is the upsert key. `order_number` holds the Shopify order id (used in the Admin API URL), `order_name` the human readable order name.

## Table 2: `alerts_sent` (read and written by workflow-alerts.json)

| tracking_number | problem_status | sub_status | order_name | email | sent_at |
|---|---|---|---|---|---|
| TRK000000001 | Exception | Exception_Other | #1001 | customer@example.com | 2026-01-15T10:30:00.000Z |
| TRK000000002 | shopify_delivered | shopify_delivered | #1002 | buyer@example.com | 2026-01-16T08:05:00.000Z |

String columns, except `sent_at` (date). The pair (`tracking_number`, `problem_status`) is the dedup key: a customer alert uses the 17TRACK status as `problem_status`, a Shopify push uses `shopify_delivered` or `shopify_out_for_delivery`.

The alerts workflow has six Data Table nodes. Point the lookup node (`DT - Look up order`, `DT - Look up order (delivery)`) at Table 1 and the four others (`DT - Not yet alerted?`, `DT - Record alert`, `DT - Not yet pushed to Shopify?`, `DT - Record Shopify push`) at Table 2. In the tracking-index workflow, `Upsert DataTable` points at Table 1.
