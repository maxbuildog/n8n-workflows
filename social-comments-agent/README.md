# AI comment moderator for Facebook and Instagram

An n8n workflow that watches the comments on a Facebook Page and an Instagram account (organic posts and ads) for an online store. For each new comment it checks the Meta signature, removes duplicates, reads the context of the post, then decides one of four things: `reply`, `hide`, `ignore` or `escalate`. It performs the action through the Graph API, writes a log row and sends a Telegram alert for anything a person should see.

## The problem

Comments arrive all day. Some are questions that deserve a quick, accurate answer. Some are abuse, spam or scam links that should not stay visible. Many are criticism, and criticism needs care: a bot that argues in public makes it worse, and a bot that deletes every unfavorable comment creates a different problem. This workflow answers questions from your own knowledge base, hides only a short list of clearly unacceptable content (and the authors you block yourself), keeps criticism visible and passes it to a human.

## Default policy

The same policy is written in the model prompt and enforced again by deterministic rules in the `Parse agent decision` node. When they disagree, the rules win.

| Situation | Decision | Public reply |
| --- | --- | --- |
| Insults or harassment | `hide` | none |
| Spam or unsolicited promotion | `hide` | none |
| Link that looks like fraud or phishing | `hide` | none |
| Competitor promoting itself with a link | `hide` | none |
| Personal data posted publicly (phone, address, order number, email) | `hide`, then alert | none |
| Author you listed in `BLOCKED_AUTHOR_IDS` | `hide` (hidden: authors you list in `BLOCKED_AUTHOR_IDS`, whatever they write) | none |
| Criticism, even a harsh one ("doesn't work", "misleading", "waste of money", "fake reviews") | `escalate`, comment stays visible | one short, polite message inviting the person to contact support (turn off with `REPLY_TO_CRITICISM`) |
| Safety or health concern | `escalate`, never hidden, never ignored | one neutral message pointing to support |
| Question (price, delivery, availability, how it works) | `reply` | answer taken from the knowledge base only |
| Support request that needs order data | `reply` | points to support and asks for the order number |
| Neutral comment, emoji, tag, simple thanks | `ignore` | none |

There is never a public debate. A model reply that contains a placeholder, a link outside `ALLOWED_REPLY_HOSTS` or no text is rejected and replaced by a rule-based answer or an escalation.

### STRICT_MODE

`STRICT_MODE` is a constant at the top of `Parse agent decision`. It is `false` by default. When set to `true`, criticism that contains no question is hidden instead of escalated. Hiding critical comments from consumers can be treated as a misleading commercial practice or as suppression of reviews in several jurisdictions, and it damages trust when customers notice: get advice before enabling it.

### Transparency

Every public reply ends with a configurable mention, `Reply from our AI assistant` (`AI_DISCLOSURE_TEXT`), enabled by default (`AI_DISCLOSURE_ENABLED`). The prompt also tells the model to say it is an AI assistant if someone asks. Nothing in the workflow tries to pass the assistant off as a person, and nothing asks the model to withhold information: facts come from the knowledge base as written, including guarantees and statutory rights.

## How it works

1. `Meta Webhook (POST)` receives comment events with the Raw Body option on. `Meta Webhook Verification (GET)` and `Respond to Meta challenge` answer Meta's setup handshake and only echo `hub.challenge` when `hub.verify_token` matches your token.
2. `Verify Meta signature` recomputes the HMAC SHA-256 of the raw body with your app secret and compares it with `X-Hub-Signature-256` in constant time. A missing or wrong signature stops the run.
3. `Extract comment` normalizes Facebook and Instagram payloads, drops your own comments, hide, like and remove events, cuts long comments and keeps a first deduplication cache in workflow static data. The Graph API version is set here, once.
4. `Dedup comment_id` inserts the ID into Postgres (`INSERT ... ON CONFLICT DO NOTHING`). `Restore comment data` stops duplicates. If the database fails, the comment goes through and is flagged `dedup_fallback`, so a real comment is never lost.
5. After a 60 second wait, `Fetch post context` reads the first 300 characters of the post text. If the call fails, the run continues without context and the node output carries the reason in `context_error`.
6. `Comment moderation agent` (chat model, Postgres memory, knowledge base as a tool) returns one JSON object: `action`, `message`, `reason`, `category`.
7. `Parse agent decision` extracts the JSON and applies the policy in this order: safety, hard hide rules, then the model decision with rule corrections. It appends the AI mention to public replies.
8. `Router action` posts a public reply when there is one (`POST reply to Meta`). Otherwise `Check hide` hides the comment (`POST hide to Meta`) or passes it straight to the log.
9. `Log to Google Sheets` appends a row. `Alert check` decides whether a person should be told and `Telegram alert` sends the message.
10. `Daily retention purge` and `Purge old rows` delete old technical rows (see Privacy).

## Setup

### Credentials in n8n

- Postgres (`postgres`): chat memory, dedup table, retention purge and the pgvector knowledge base.
- Ollama (`ollamaApi`): embeddings with `nomic-embed-text:latest`. Use the same model to embed your documents.
- OpenAI-compatible API (`openAiApi`): the chat model node uses `gpt-4.1-mini`. Change the model, or add a base URL in the node options, to use another provider.
- Google Sheets OAuth2 (`googleSheetsOAuth2Api`).
- Telegram (`telegramApi`).

### Environment variables on the n8n instance

| Variable | Purpose |
| --- | --- |
| `META_APP_SECRET` | App secret used to verify `X-Hub-Signature-256`. Without it the workflow refuses every request. |
| `META_VERIFY_TOKEN` | Required. The token you type in the Meta webhook settings, checked on the GET handshake. If it is empty or unset, the handshake is refused with a 403: there is no default value. |
| `META_PAGE_TOKEN` | Page access token, used in three places (reply, hide, post context). It is never written in the workflow. |
| `NODE_FUNCTION_ALLOW_BUILTIN=crypto` | Lets the Code node call `require('crypto')`. |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` | Lets nodes read the variables above. If you cannot allow this, replace the two HTTP Request token fields with an HTTP Query Auth credential (parameter name `access_token`) and keep the secrets elsewhere. |

### Raw body

The POST webhook node has the Raw Body option enabled. The signature is computed on the exact bytes Meta sent, so the parsed JSON cannot be used. If you recreate the node, enable the option again, otherwise `Verify Meta signature` fails with an explicit message.

### Webhook URLs

Both webhook nodes use the path `social-comments-events`. Register the production URL (`https://YOUR_N8N_HOST/webhook/social-comments-events`) in the Meta app, with the same verify token as `META_VERIFY_TOKEN`.

### Placeholders to replace (search for `YOUR_`)

- `Extract comment`: `YOUR_PAGE_ID`, `YOUR_IG_USERNAME`, `OUR_NAMES`, `GRAPH_API_VERSION` (the only place the version is set: use a currently supported version, default `v24.0`) and `DEFAULT_PRODUCT` (empty by default: the product assumed when a comment names none; when it is empty the agent answers in general terms or points to the relevant product page).
- `Parse agent decision`: `SUPPORT_EMAIL`, `STORE_URL`, `ALLOWED_REPLY_HOSTS` (the domain of your store, so replies may link to it), the product URLs, `COMPETITOR_NAMES`, `COMPETITOR_DOMAINS`, `BLOCKED_AUTHOR_IDS`, `SAFETY_KEYWORDS`, `FAQ_RULES`, `STRICT_MODE`, `REPLY_TO_CRITICISM`, `AI_DISCLOSURE_ENABLED`, `AI_DISCLOSURE_TEXT`.
- Agent system prompt: `YOUR_STORE_NAME`, `YOUR_SUPPORT_EMAIL`, `YOUR_STORE_URL`, `YOUR_PRODUCT_A_URL`. Rename Product A, B and C to your catalog.
- `YOUR_DOCUMENTID`, `YOUR_SHEETNAME`: the Google Sheet and its tab. The columns are in `examples/google-sheets-log-example.csv`.
- `YOUR_TELEGRAM_CHAT_ID`: the chat that receives alerts.

### Meta app and permissions

- Webhook subscriptions: `feed` on the Page object and `comments` on the Instagram object.
- Reading comments and post text: `pages_read_engagement`, `pages_read_user_content`, `pages_manage_metadata`.
- Replying to and hiding Facebook comments: `pages_manage_engagement`.
- Instagram comments (read, reply, hide): `instagram_basic`, `instagram_manage_comments`, with the Instagram account linked to the Page. Live use requires Meta App Review.

### Tables and files

- `chat_memory` is created by the memory node. `kb_embeddings` is a pgvector table filled with your own documents (see `examples/knowledge-base-example.md`).
- `dedup_comments` must exist before the first run:

  ```sql
  CREATE TABLE IF NOT EXISTS dedup_comments (
    comment_id TEXT PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW()
  );
  ```

- A Google Sheet with the nine columns of the example CSV.

### Testing without risk

Keep the workflow inactive. Send a signed test payload to the test URL, or run the nodes manually with pasted data, and point `POST reply to Meta` and `POST hide to Meta` at a comment on a private Page. Check the sheet and the Telegram message before activating.

## Privacy

This workflow processes public comments, display names and account IDs, and it writes them to several places.

- **What the model receives:** the platform, the comment text (cut to 1000 characters) and the first 300 characters of the post. It does not receive the author name or any ID.
- **Model provider:** choose one that offers a data processing agreement (DPA), a region that fits your customers and a no-training commitment. Local embeddings keep the knowledge base text on your server.
- **What is stored:**
  - Google Sheet: comment text, display name, comment ID, decision, reply. Decide a retention period (90 days is a reasonable default), delete older rows on a schedule and delete a person's rows on request.
  - `dedup_comments`: comment IDs only. The `Purge old rows` node deletes rows older than the number of days in its query replacement (30 by default, change it there).
  - `chat_memory`: a few turns keyed by platform, post and account ID. The purge node empties it every day.
  - Telegram: alerts contain the comment text and display name. Use a private chat and delete old alerts.
  - n8n execution history: it keeps the payloads. Set `EXECUTIONS_DATA_MAX_AGE` (or the matching settings) to the same retention period.
- **No profiling:** nothing is kept to build a profile of a person across comments.
- **Hidden personal data:** when a comment containing personal data is hidden, an alert tells a human so the person can be contacted privately.
- Tell your users in your privacy policy that comments are processed by an AI assistant.

## Limits

- The rule lists are keyword and pattern based and written in English. For other languages, translate the lists in `Parse agent decision` and `Alert check` and the prompt.
- Only comments are handled, not private messages, and only the first entry and change of each webhook payload.
- Personal data detection relies on patterns (phone numbers, email addresses, street addresses, order numbers). It will miss unusual formats and can flag a customer who posts their own order number: the alert exists so a human follows up.
- Replies are posted automatically. There is no draft mode. Add an approval step before `POST reply to Meta` if you want one.
- A model that proposes `hide` for something outside the five hide categories is overruled: the comment stays visible and a human is alerted.
- The Graph API version is a constant in `Extract comment` and needs a review from time to time.
- The Telegram message uses `parse_mode: HTML`. `Alert check` escapes `<`, `>` and `&` in the comment text. Reply bodies are built with `JSON.stringify`, so quotes and line breaks cannot break the request.
