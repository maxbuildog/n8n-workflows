# Low stock alerts with thresholds computed from real sales (Shopify + Telegram)

Two n8n workflows that compute per-product stock thresholds from the last 30 days of Shopify sales and a supplier lead time, then send a daily Telegram alert when a product's stock falls below them.

## The problem

Fixed reorder thresholds go stale as soon as sales speed changes. A product that sells 2 units a day and one that sells 20 need very different safety margins, and the margin also depends on how long a resupply takes. Merchants either check a spreadsheet by hand or find out about a stockout from customers. Here the thresholds are recomputed every week from real orders, and the daily check stays silent when nothing needs attention.

## How it works

Workflow 1, `workflow-thresholds.json` (every Monday 8am):

1. Sets the shop domain and the Shopify API version in one `Shopify config` node, then requests a Shopify Admin API access token with the client credentials grant.
2. Reads the rows of an n8n Data Table whose `mode` is `dynamic` (one row per tracked product, with its Shopify product ids and lead time).
3. Pages through Shopify orders of the last 30 days (250 per page, following the `Link` header, capped at 100 pages), skipping canceled orders, and accumulates line item quantities in workflow static data.
4. Maps product ids to SKUs and computes, per SKU: average daily sales, red threshold = ceil(avg daily x lead time x 1.2), orange threshold = ceil(avg daily x lead time x 2.0).
5. Upserts the thresholds into the Data Table, matching on `sku`, then sends a recap to Telegram.

Workflow 2, `workflow-daily-check.json` (every day 9am):

1. Reads a Google Sheet that holds one row per product and one column per date (stock counts, optionally followed by text such as `(100 pcs in production)`).
2. Takes the latest non-empty column of each row, parses current stock and the quantity in production, and sums them into an effective stock.
3. Joins those rows with the thresholds in the Data Table.
4. Classifies each product: red if effective stock is at or below the red threshold, orange if at or below the orange threshold. Rows with `mode = excluded` or zero thresholds are ignored.
5. If there is at least one alert, sends a Telegram message (critical first, then watch list with an estimate of days left before the red zone). Otherwise it does nothing.

## Design choices

- Thresholds are derived from sales speed and lead time, not typed in. The 1.2 and 2.0 multipliers are the only constants, and they sit in one code node.
- Stock in production counts toward effective stock, so a product that is already being resupplied does not trigger a false alarm.
- The two workflows share only a Data Table. The daily check never calls Shopify, so it keeps working if the Shopify token request fails. If the table has no rows, nothing can be classified and no alert is sent: `Classify Alerts` returns an error message in its output, but the run then ends on the silent branch, so check the table after the first Monday run.
- Pagination follows the `Link` header and a page cap, and the accumulator is reset after each run so a failed run cannot leak line items into the next one.
- Silence by default: no alert means no message, which keeps the channel useful.

## Setup

Credentials and environment:

- Shopify Admin API: the token request node reads `SHOPIFY_CLIENT_ID` and `SHOPIFY_CLIENT_SECRET` from n8n environment variables (no secret is stored in the workflow). Your n8n instance must allow `$env` access in nodes (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`). The app needs the `read_orders` scope. If you prefer, replace the body of the `Get Shopify token` node with a reference to your own credential or secret store.
- `telegramApi` credential (Telegram bot token), selected in both Telegram nodes. Messages use Telegram's legacy `Markdown` mode; product names are escaped (`_`, `*`, `[` and backticks) so a name such as `Item_2 [new]` cannot break the message.
- `googleSheetsOAuth2Api` credential, selected in the `Read Google Sheet` node.

Placeholders to replace:

- `your-store.myshopify.com` in the `Shopify config` node (`shop_domain`). The token request and the orders URL both read it from there.
- `api_version` in the same `Shopify config` node: the only place the Shopify Admin API version is set. The default is only an example, use a currently supported version and bump it before Shopify retires it.
- `YOUR_DATATABLEID` (in `Read DataTable (dynamic)`, `Upsert thresholds` and `Read DataTable (all)`): the id of your Data Table.
- `YOUR_TELEGRAM_CHAT_ID` (both Telegram nodes).
- `YOUR_DOCUMENTID` and `YOUR_SHEETNAME` in `Read Google Sheet`.

Data Table columns: `sku`, `product_name_sheet`, `shopify_product_ids` (comma separated), `mode` (`dynamic` or `excluded`), `lead_time_days`, `avg_daily_sales_30d`, `red_threshold`, `orange_threshold`, `last_calculated_at`. See `examples/data-table.csv`.

Sheet layout: fixed columns `Variant`, `Product Name`, `SKU`, then one column per date. See `examples/stock-sheet.csv`.

Testing without risk: open each workflow in the n8n editor and use "Execute workflow" (there are no test webhooks, so nothing is reachable from the internet). Run workflow 1 first and check the recap and the table, then run workflow 2. Point the Telegram nodes at a private test chat while you check the numbers.

## Limits

- Sales are counted over a fixed 30-day window; there is no seasonality or trend adjustment.
- Thresholds are per SKU, so a SKU sold under several product ids relies on the comma separated mapping being correct.
- The stock source is a spreadsheet you maintain; the workflow does not read inventory from Shopify.
- An empty Data Table is not reported by Telegram: the daily check stays silent (see above). Run the thresholds workflow before relying on it.
- No automatic purchase order, only an alert.

Derived from a workflow that runs on a real store, generalized so you can adapt it: thresholds, lead times and multipliers are values to set for your own catalog.
