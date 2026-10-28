# Store Chat: AI agent with order lookup

A chat assistant for an online store built on Shopify. It answers customer questions from a knowledge base, looks up an order when the customer gives both its number and the email used for it, keeps per-session memory in Postgres, and logs knowledge gaps to Notion. It ships as three linked n8n workflows.

## The problem

Many support requests in an online store are the same few questions: where is my order, what does this product cost, what are the delivery and return terms. A free-form chatbot that improvises prices, guarantees or promises is a liability. This setup answers only from the store's own knowledge base and live order data, never commits the store to a refund, a delay or a code, tells customers they are talking to an AI, and sends real gaps to a review list.

## How it works

Files: `workflow-agent.json` (main agent), `workflow-order-lookup.json` (order lookup sub-workflow), `workflow-session-cleanup.json` (scheduled cleanup).

Main agent:
1. `Chat Webhook` (path `store-chat`) accepts a POST from the chat widget with `message`, `sessionId` and `isFirstMessage`.
2. `Validate Bearer Token` compares the `Authorization` header to the expected value. A mismatch goes to `Respond 401 Unauthorized`. See "The widget token is not a secret" below.
3. `Validate Input` rejects an empty message, a message longer than 1000 characters (constant `MAX_MESSAGE_LENGTH` in the node) and a missing or malformed session id. `Input Valid?` routes failures to `Respond 400 Bad Request`.
4. `AI Agent` (tools agent, max 5 iterations) runs an OpenAI-compatible chat model (`Chat Model`) with three attachments: `Postgres Chat Memory` (session history), `kb_store` (pgvector retrieval, top 6, embeddings from Ollama `nomic-embed-text`) and `order_lookup` (HTTP call to the order lookup workflow).
5. `Check Unanswered` strips Markdown and list formatting, keeps at most 3 paragraphs and about 700 characters, flags replies that contain the exact gap sentence from the prompt, and puts the AI disclosure sentence in front of the first reply of a session.
6. `Respond to Widget` returns `{ reply, sessionId }`. If the reply was flagged, `Unanswered?` sends the question and the reply to `Log to Notion`. If the agent errors, `Respond Fallback` returns a short apology that says it comes from an AI assistant and gives the support address.

Order lookup (`workflow-order-lookup.json`):
1. `Orders Webhook` (path `order-status-lookup`, Header Auth) receives `email` and `order_number`.
2. `Require Both Fields` checks that both are present. If one is missing, `No Info` asks the agent to request it.
3. `Settings` holds the shop domain and the Shopify Admin API version (the one place to change them).
4. `Shopify Orders by Number` queries the Admin API by order number, requesting only the fields needed.
5. `Verify and Format` keeps the order only if its number and its email both match the request, then returns a minimal summary: number, date, payment and shipping status, carrier, tracking numbers and URLs, items and quantities. Otherwise it returns the same neutral "no matching order" message.

Session cleanup (`workflow-session-cleanup.json`): `Daily at 3am` runs `Retention Settings` (retention period and row cap) then `Delete Expired Sessions`, then `Log Result`.

### How the agent reaches the order lookup workflow

`order_lookup` is an HTTP Request Tool that POSTs `email` and `order_number` to the webhook of the lookup workflow, authenticated with a Header Auth credential. Import `workflow-order-lookup.json`, activate it, copy the production URL of `Orders Webhook` and put it in the `url` field of `order_lookup` (placeholder `https://YOUR_N8N_HOST/webhook/order-status-lookup`). Create one Header Auth credential and select it on both `Orders Webhook` and `order_lookup`.

## Behavior rules built in

- **AI disclosure.** The first reply of a session starts with a configurable sentence, added by code in `Check Unanswered` so it does not depend on the model. The prompt only tells the model never to present itself as a person. Set the sentence through the `YOUR_AI_DISCLOSURE` placeholder, for example: `Hi, I am the store's AI assistant.` The widget tells the workflow which message is the first one with the `isFirstMessage` field (see Session identifiers). Keep it enabled.
- **Knowledge base only.** No price, figure, duration or claim that `kb_store` did not return. When retrieval finds nothing, the assistant says "I don't have that information." and gives the support address; that exact sentence is what `Check Unanswered` uses to log a gap.
- **No commitments.** It never promises a refund, replacement, delay, delivery date or code. Warranty and return terms are quoted as written in the knowledge base, including statutory rights.
- **Safety and wellbeing.** A message that reports a safety problem is never answered on the substance or ignored: the assistant points to human support and asks for the order number.
- **Order lookup needs two matching identifiers.** Number and email must match the same order, otherwise the answer is the same neutral message whether or not the order exists. No search by email alone, no hint about the real email.
- **Data minimization.** The lookup returns no name, address, phone or email to the model.
- **Authenticated lookup endpoint.** `Orders Webhook` requires Header Auth and the agent's tool sends the same header.
- **Output guardrail in code.** Widget formatting and length are enforced after the model answers, not only in the prompt.

## Session identifiers

The widget must generate the session id itself: a random UUID (for example `crypto.randomUUID()`), created once per visitor session and sent as `sessionId` in every request. It is required. `Validate Input` rejects anything that is not a UUID with a 400. The widget also sends `isFirstMessage`: `true` with the first message of a session and `false` with every later one. The workflow cannot read the chat memory itself, so it relies on this field to add the AI disclosure once. If the field is missing, the disclosure is added to every reply (the safe default). There is no fallback to the email address or to the IP address, and no IP address is read, logged or stored by these workflows. The id is returned in the response and used as the memory key.

## The widget token is not a secret

The bearer value checked by `Validate Bearer Token` ships inside the widget code, so anyone who opens the page can read it. It only filters background noise (random scanners and bots hitting the URL). Do not treat it as authentication of a customer. Real protection comes from:
- Restricting `allowedOrigins` on `Chat Webhook` to your store domain. The workflow ships with the placeholder `YOUR_STORE_ORIGIN` (for example `https://www.your-store.example`); do not use `*`. Browsers enforce this, other clients can ignore it.
- Rate limiting in front of n8n, for example in a reverse proxy (per-IP request limit on `/webhook/store-chat`). n8n does not rate limit by itself.
- The message length guard in `Validate Input`, and the iteration and token limits on the agent and model.
- The fact that the only sensitive tool (order lookup) needs both an order number and its email.

## Setup

Credentials to create in n8n:
- OpenAI API credential (`openAiApi`) for `Chat Model`. The node's default model is `gpt-4.1-mini`; change the model, or add a base URL in the node options, to use another OpenAI-compatible provider.
- Postgres credential (`postgres`) for `Postgres Chat Memory`, `kb_store` and the cleanup workflow.
- Ollama credential (`ollamaApi`) for `Ollama Embeddings (KB)`, with `nomic-embed-text:latest` pulled.
- HTTP Header Auth credential (`httpHeaderAuth`) for the Shopify node: header `X-Shopify-Access-Token`, value an Admin API token with read access to orders.
- HTTP Header Auth credential for the lookup webhook: any header name, a long random value. Select it on `Orders Webhook` and on `order_lookup`.
- HTTP Header Auth credential for `Log to Notion`: header `Authorization`, value `Bearer <your Notion integration token>`.

Placeholders to replace:
- `YOUR_WIDGET_TOKEN` in `Validate Bearer Token` (the widget sends `Authorization: Bearer <value>`).
- `YOUR_STORE_ORIGIN` in `allowedOrigins` of `Chat Webhook`.
- `YOUR_AGENT_NAME`, `YOUR_STORE_NAME` and `YOUR_SUBSCRIPTION_PORTAL_URL` (drop the subscriptions section of the prompt if the store has none) in the agent system prompt. `YOUR_SUPPORT_EMAIL` in the agent system prompt, in `Respond Fallback` and in `Check Unanswered`. `YOUR_AI_DISCLOSURE` in `Check Unanswered`.
- `YOUR_KB_TABLE`: the pgvector table used by `kb_store`.
- `YOUR_NOTION_DATABASE_ID`: the Notion database receiving knowledge gaps, with properties `Question` (title), `Status` (select with an option `To review`), `Agent reply` (text) and `SessionId` (text).
- `https://YOUR_N8N_HOST/webhook/order-status-lookup` in `order_lookup`.
- `shop_domain` in the `Settings` node of the lookup workflow.

### API versions

Versions are kept in one place each. Use a currently supported version and review it at least once a year.
- Shopify Admin API: field `shopify_api_version` in the `Settings` node of the lookup workflow (default `2026-07`). Shopify supports each version for about a year; check the current list before going live.
- Notion API: header `Notion-Version` in `Log to Notion` (default `2022-06-28`). If you move to a newer version, check how it handles database parents before changing it.

### Tables

`chatbot_memory` is created by the Postgres chat memory node on first use. The cleanup workflow needs a creation date, so add one once:

```sql
ALTER TABLE chatbot_memory
  ADD COLUMN IF NOT EXISTS created_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
```

Rows that exist when you add the column get the date of the migration. `YOUR_KB_TABLE` is created by the pgvector node when you ingest documents. See `examples/kb-example.md` for the kind of passages to ingest.

## Cleanup and retention

`Retention Settings` holds two constants:
- `retention_days` (default 30): conversation rows older than this are deleted every night.
- `max_rows` (default 10000): safety cap. Rows are ranked by id, newest first, and everything after the first `max_rows` is deleted, so the most recent rows are the ones kept.

`Log Result` outputs `rows_remaining` (rows left after cleanup) and `cleanup_date`. It does not report how many rows were deleted. Adjust `retention_days` to what your privacy notice states.

## Privacy

What is processed and where:
- The text of each message and reply goes to the language model provider, and to the embeddings host for knowledge base queries. For an order lookup, the order number and email typed by the customer go to your lookup webhook and then to Shopify; only the minimal summary described above goes back to the model.
- Conversation memory is stored in the `chatbot_memory` table, keyed by the random session id. Customers can type personal data in free text, so treat the table as personal data.
- Knowledge gaps are written to Notion: the question (first 1000 characters), the reply and the session id. Review and delete these rows regularly; Notion rows are not covered by the cleanup workflow.
- No IP address and no cookie is stored by these workflows. Your reverse proxy and n8n execution logs may keep IP addresses; set their retention too.

Recommendations:
- Choose a model provider that offers a data processing agreement (DPA), with a region suited to your customers, and a no-training setting on API data. Do the same for Notion and your hosting.
- Set a retention period and publish it in your privacy notice. The default is 30 days (`retention_days`). Also limit how long n8n keeps execution data (execution pruning), since executions contain messages.
- Do not build long-term customer profiles from chat history. If you want analytics, aggregate first and keep only counts.
- Tell customers they are talking to an AI (built in) and how to reach a person (the support address in the prompt).

## Customer reviews

This workflow never asks for reviews. If you add an invitation, keep it neutral and identical for every customer, independent of how the conversation went. Do not filter who is asked based on sentiment.

## Testing safely

Run each workflow manually first. Send the agent webhook a test POST with the bearer header, a UUID `sessionId` and a message such as `How long does delivery take?`. Test the lookup with a test order. Leave `Log to Notion` pointing at a scratch database, and run the cleanup workflow only after checking the row count of `chatbot_memory`.

## Limits

- Anyone holding both an order number and its email can read that order's status, items and tracking. Orders with no email on record cannot be looked up.
- Only one order per request is checked, by number.
- The bearer check is a plain string comparison and is not a secret (see above).
- The first-message disclosure relies on the widget sending `isFirstMessage`. A client that omits it gets the sentence on every reply; a client that sends `false` on a first message gets none.
- The gap detector matches one English sentence; if you change the sentence in the prompt, change `GAP_PHRASE` in `Check Unanswered` to match.
- No automatic handoff to a human (the assistant gives the support address), no refunds or order changes, one language model per run.
