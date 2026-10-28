# Proactive shipping exception alerts (17TRACK webhooks + Shopify)

Two linked n8n workflows that listen to carrier tracking events from 17TRACK, warn customers by email (through Klaviyo) when a parcel has a delivery problem, and write delivered / out-for-delivery events back to the Shopify fulfillment.

## The problem

A merchant usually learns about a lost or blocked parcel when the customer writes in angry. Carrier tracking pages show the problem days earlier, but nobody watches them order by order. Shopify also often keeps showing "in transit" after the parcel has been delivered, because the tracking number is attached by hand or by an app that never reports back.

## How it works

Workflow 1, `workflow-tracking-index.json` (index tracking number to order):
1. A Shopify fulfillment webhook receives the fulfillment payload. A Code node first verifies the `X-Shopify-Hmac-Sha256` signature on the raw body; an invalid or missing signature gets a `401` and the run stops.
2. A Code node extracts every tracking number with its order id, order name, customer email (lowercased) and carrier. If there is none, the run stops with a skip marker.
3. The row is upserted in a Data Table, keyed on the tracking number.
4. The tracking number is registered at 17TRACK so that pushes start arriving. The register call never fails the run.
5. The webhook is answered with `{"status": "ok"}`.

Workflow 2, `workflow-alerts.json` (receive 17TRACK pushes):
1. A webhook receives the 17TRACK push. A Code node checks the shared secret token in the URL (`?token=`); a wrong or missing token gets a `401` and the run stops. A valid push is answered `ok` immediately.
2. A Code node extracts only the tracking numbers from the push. The status in the push body is ignored.
3. 17TRACK `gettrackinfo` is called for each number to read the current status and the latest event description. This is the only source of the status used for decisions.
4. Branch A, problems: if the status is `Exception`, `Undelivered` or `Expired`, the order is looked up in the Data Table, merged with the status, and skipped if there is no email. A Data Table check makes sure this tracking number and status were not alerted before. A Klaviyo event is sent, then the alert is recorded.
5. Branch B, delivery: if the status is `Delivered` or `OutForDelivery`, the order is looked up, skipped if there is no order id, and a Data Table check makes sure this status was not already pushed. Then a Shopify token is requested, the order fulfillments are read, the matching fulfillment is chosen by tracking number, and a fulfillment event (`delivered` or `out_for_delivery`) is posted. The push is recorded.

## Design choices

- **Dedup on tracking number plus status.** Carriers push the same status many times. A Data Table row per (tracking number, status) means a customer gets one alert per problem type and Shopify gets one event per state.
- **Answer the webhook first.** The 17TRACK push is acknowledged before any API call, so slow Shopify or Klaviyo calls cannot trigger carrier-side retries and duplicate pushes.
- **Re-read the status instead of trusting the push.** The push is only a signal that a tracking number changed. The workflow calls `gettrackinfo` and decides on the status it gets back, so a forged, stale or out-of-order push cannot trigger a customer email or a Shopify write on its own. The latest event description (translated when available) comes from the same call and goes into the customer message.
- **Authenticate both webhooks.** The Shopify webhook is verified with its HMAC signature (constant-time comparison, fails closed if the secret is missing). 17TRACK pushes are protected by a shared secret token in the URL, because the push is only used to pick which number to re-read.
- **Fail closed on missing data.** No email means no alert, no order id means no Shopify call, no matching fulfillment means a skip marker rather than a write on the wrong one.
- **Index decoupled from alerts.** Tracking number to order mapping lives in its own workflow and table, so the alert flow needs no Shopify lookup on the problem path.

## Setup

Credentials and environment variables (nothing secret is stored in the JSON):

- n8n credential, type `Header Auth`, name `17TRACK token (header 17token)`: header name `17token`, value is your 17TRACK API key. Used by `17TRACK gettrackinfo` and `17TRACK - Register`.
- Environment variables on the n8n instance (read with `$env`; if your instance sets `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`, unblock it or replace the expressions with a credential of your choice):
  - `NODE_FUNCTION_ALLOW_BUILTIN=crypto`: required, the two verification Code nodes use `require('crypto')`. n8n blocks Node built-ins in Code nodes unless they are allowed this way.
  - `SHOPIFY_WEBHOOK_SECRET`: the signing secret of the Shopify webhook (shown in the Shopify admin next to the webhook, or the app client secret for webhooks created by an app). Used by `Verify Shopify HMAC`.
  - `TRACK17_WEBHOOK_TOKEN`: a long random string you generate (for example 32 random bytes in hex). Used by `Verify push token`.
  - `SHOPIFY_CLIENT_ID` and `SHOPIFY_CLIENT_SECRET`: client credentials of your Shopify app, used by `Shopify Get Token` (client credentials grant).
  - `KLAVIYO_PUBLIC_API_KEY`: your Klaviyo public API key (company id), used in the `Klaviyo - Event` URL.
- In the `Shopify Fulfillment Webhook` node of the index workflow, the option `Raw Body` must stay enabled: the HMAC is computed over the exact bytes Shopify sent, and a re-serialized JSON body would never match.
- Placeholders to replace in the JSON:
  - `YOUR_DATATABLEID` in every Data Table node (pick the right table in the node UI, see `examples/data-tables.md`).
  - `YOUR_TRACKING_PAGE_URL` in `Build Klaviyo body`, the public tracking page of your store.
  - `your-store.myshopify.com` (`shop_domain`) and `api_version` in the `Shopify config` node of the alerts workflow. It is the only place the Shopify domain and Admin API version are set; the three Shopify HTTP nodes read them from there. The default version is only an example: use a currently supported version and bump it before Shopify retires it.
- Klaviyo: the API revision is the constant `KLAVIYO_REVISION` at the top of `Build Klaviyo body` (default `2025-10-15`), sent in the `revision` header. Check the current revision in the Klaviyo documentation and bump it from time to time. The event metric is named `Delivery Problem`. Build a flow triggered by this metric; the event properties are `order_number`, `tracking_number`, `problem_type`, `problem_description`, `tracking_url`, `carrier`, `sub_status`.
- Webhooks: point a Shopify fulfillment webhook (topic `fulfillments/create`) at `https://<your-n8n>/webhook/shopify-fulfillment-created`, and set the 17TRACK push URL to `https://<your-n8n>/webhook/carrier-tracking-events?token=<TRACK17_WEBHOOK_TOKEN>`. Rotate the token by changing the environment variable and the URL registered at 17TRACK together.
- Data Tables: create the two tables described in `examples/data-tables.md`.

Testing safely: keep both workflows inactive and run them manually with the test webhook URL and a sample payload. The verification nodes will answer `401` unless the sample request carries a valid signature (index) or the `?token=` query parameter (alerts); compute the signature of your sample body with the same secret, or temporarily pin the Verify node's output while you test. Disable `Klaviyo - Event` and `Shopify POST Event` first, or use a test Klaviyo account and a development store.

## Privacy

The workflows handle customers' email addresses and order references.

- **Two Data Tables hold customer emails.** `tracking_index` stores the email next to the tracking number and the order, and `alerts_sent` stores it again next to each alert. n8n Data Tables do not expire rows by themselves. Decide a retention period (for example 90 days after the last carrier event) and purge old rows on a schedule: add a scheduled workflow with a Data Table delete step that filters on a date. `alerts_sent` has `sent_at`; `tracking_index` has no date column, so add one (for example `created_at`) or clear it on the same schedule. Delete a person's rows on request.
- **Klaviyo events are transactional.** The `Delivery Problem` event tells a customer about their own parcel. Sending the event creates or updates a Klaviyo profile for that email, but it does not give marketing consent. Do not subscribe these profiles to marketing lists without the customer's consent, and keep the flow triggered by the event strictly informational.
- **What leaves your server.** 17TRACK receives tracking numbers only. Klaviyo receives the email, the order name, the tracking number and the carrier status. Shopify receives a fulfillment event with a short status message.
- **Execution history.** n8n keeps the payloads of executions, including emails. Set a short execution retention (`EXECUTIONS_DATA_MAX_AGE`).
- Mention these processors (17TRACK, Klaviyo, Shopify) in your privacy policy.

## Limits

- The Shopify calls use REST Admin API endpoints (`orders/{id}/fulfillments.json`, `fulfillments/{id}/events.json`), which Shopify now labels legacy. Check that they are still served for the API version you set; otherwise port these two calls to the GraphQL Admin API.
- The 17TRACK webhook is protected by a shared secret token in the URL, not by a payload signature. The token appears in the URL, so keep it out of logs and proxies that record query strings. Because decisions rely on `gettrackinfo`, a leaked token still cannot inject a status, only trigger extra re-reads.
- No retry or error workflow: a failed Klaviyo or Shopify call is not replayed, and the Data Table row is only written after success.
- Only the first accepted tracking item is used in the status step.
- The Shopify event is only posted when a fulfillment of the order matches the tracking number. Otherwise `Build Shopify event` returns a skip marker, `Fulfillment found?` stops the run and nothing is written.
- Customer email wording lives in Klaviyo, not in n8n.

Derived from a workflow that runs on a real store, generalized so you can adapt it: the Data Table names, the Klaviyo metric name and the alert wording are values to set for your own store.
