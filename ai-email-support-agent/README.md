# AI email support agent (Gmail + Shopify + knowledge base)

An n8n workflow that reads a store's support inbox every 3 minutes, sorts incoming email with deterministic rules, lets an AI agent look up the customer's order, tracking and knowledge base, and leaves a ready-to-review draft reply in Gmail for a human.

## The problem

A small online store receives a steady flow of repetitive support email: where is my parcel, how does this work, can I return it. A human has to open each thread, look up the order in Shopify, check the carrier, find the right policy and write a polite answer. Most of the day goes into lookups, not decisions. The rest of the inbox is noise (out-of-office replies, platform notifications, newsletters, cold outreach) that hides the few messages that really are urgent: a legal threat, a safety or health report, a personal data request.

## How it works

1. A schedule (every 3 minutes) or an authenticated manual webhook (`support-agent-run`) starts a run. Gmail returns up to 10 unread messages that do not already carry one of the support or non-support labels.
2. Each email is flattened to a small record (sender, subject, body capped at 8000 characters, attachment types). The store configuration (name, signature, own domains, default language, confidence threshold, memory retention) is copied onto each record.
3. Duplicate lock: every `message_id` is inserted into a Postgres table with `ON CONFLICT DO NOTHING ... RETURNING`. Only the rows really inserted continue, so two overlapping runs never process the same email. If the database call fails, the workflow lets everything through rather than losing email. Old claims are purged by the same statement, and the query is parameterized (the ids are passed as parameters, never concatenated into the SQL).
4. A JavaScript classifier (no AI) sorts the email. Out-of-office replies, the store's own sends, registrar and no-reply senders, file-transfer links, social, marketing and review platforms and cold outreach are labeled, marked read, archived and logged. Every list is a constant in a configuration block at the top of the node, with generic `.example` values to replace. Strong customer signals (order words, `#1234` style numbers) stop a real customer from being filed as admin by the weaker rules.
5. Two kinds of non-support email are escalated back into the support flow: an alert from your live chat tool saying a conversation needs a human (the trigger phrases are a configurable list), and a 1 to 3 star review notification.
6. The support emails and the escalated notifications from step 5 are first joined by a Merge node, so each email goes through the next steps exactly once. For each of them the workflow loads the last 5 messages of the Gmail thread and the customer's memory row (first contact, interaction count, last summary, sentiment trail, last order). Memory older than the retention period is ignored and deleted.
7. The agent (an OpenAI-compatible chat model, temperature 0.3, at most 18 iterations) receives the sender's name, subject, body, thread and memory. It can call four tools: a vector store of facts (KB1), a vector store holding the playbook and reply templates (KB2), a read-only Shopify order lookup and a 17TRACK tracking lookup. It answers with one strict JSON object: category, emotional level, language, confidence, label, draft subject and body, an optional note for the reviewer, the order number and a one-line summary.
8. A code node parses the JSON and applies the guardrails (see Escalations and guardrails). Unparseable output becomes a yellow item with the raw text in a note; hitting the iteration cap becomes a red item. A line stating that an AI assistant was involved is appended to every customer-facing draft.
9. With the shipped setting (`AUTO_SEND_ENABLED = false`) every reply is a draft: old support labels are removed, a Gmail draft is created in the customer's thread (reviewer note appended below a separator), one color label is applied, and the thread is marked read and archived. Partnership email gets a label only, with no draft.
10. Every processed email is appended to a master log sheet. Blue (to validate) items are also appended to an "unanswered questions" sheet, and the customer's memory row is updated.

## Escalations and guardrails

Four kinds of message always end on the red label and are never auto-sent: a legal threat, a safety or health concern, a request about personal data (GDPR style), and a very unhappy customer (emotional level 4). Chat tool alerts and low-rating review notifications are forced to red as well. The agent is told how to draft for each case, and code enforces the label independently of the model: the model's category or emotional level, or a match of the patterns in `LEGAL_PATTERNS`, `SAFETY_HEALTH_PATTERNS` and `PRIVACY_PATTERNS` on the customer's own words, forces red. A safety or health message is never answered on the merits and never ignored: it goes to a person.

Other guardrails in Parse agent output:

- A green draft is downgraded to yellow if it contains a promo code that is not in `ALLOWED_PROMO`, a discount above `MAX_AUTO_DISCOUNT_PERCENT` (20 by default), or a commitment (refund, reshipment, replacement, cancellation, change).
- A green draft whose confidence is below the threshold (0.7 by default, one value set in Extract email data) is moved to blue. The same value appears in the agent prompt and in the unanswered-questions log, so the three always agree.
- An unknown label is forced to blue.

## Design choices

**Duplicate lock in the database, not in the inbox.** Gmail "unread" is not a safe queue when a run takes longer than the 3 minute interval. Claiming the `message_id` with a unique key makes processing once per email, and the label exclusion in the Gmail query adds a second layer. The lock fails open on purpose: a database outage weakens dedup, it does not drop customer email.

**Deterministic triage before the model.** Anything that can be decided from the sender domain, an address pattern or a keyword is decided in plain code: cheaper, testable, and it never invents anything. The model only sees email that really is a customer conversation, plus two escalation types that rules detect on their own. Code also runs after the model, so the final label does not depend on the model's judgment alone.

**An agent that checks before it writes.** The prompt tells the agent which tool to consult for which question, and forbids quoting any order number, date, product or tracking status that is not literally in a tool response. If the order lookup finds nothing, or the number in the email matches no order, the draft must ask for confirmation and the email goes to the "to validate" label instead of guessing. Tracking is only queried after a tracking number has been read from the order.

**Send or draft is decided by a rule, not by the model.** The model proposes a label and a category. Whether anything is sent without a human is a constant and a small condition in code: toggle on, green label, non-empty draft, no reviewer note, and a category in a short allow-list. The shipped toggle is off, so every reply is a draft. Turning on auto-send is a one-line change that is visible in a diff, not a prompt tweak.

**A journal that feeds the knowledge base.** Each run appends a row per email. Low-confidence, non-default-language or ambiguous cases also go to an "unanswered questions" sheet with a Gmail link, a status of `To process` and an empty `kb_update_needed` column. A person turns those rows into knowledge base entries. The workflow does not write to the knowledge base by itself.

## Default behaviors

| Topic | Default | Where to change it |
| --- | --- | --- |
| Sending | Drafts only, a human reviews and sends | `AUTO_SEND_ENABLED` in Parse agent output |
| AI transparency | Each draft ends with "This reply was drafted by an AI assistant and reviewed by our team." (a different line is used if you enable auto-send, since nobody reviews then) | `AI_DISCLOSURE_ENABLED`, `AI_DISCLOSURE_DRAFT`, `AI_DISCLOSURE_AUTO` in Parse agent output. Add one entry per language you serve; `en` is the fallback |
| Facts | Quoted from your knowledge base as written, nothing is hidden or invented. Statutory consumer rights of the customer's country are quoted whenever they apply | KB1 content |
| Safety, health, legal, personal data | Always escalated to a human | patterns and categories in Parse agent output |
| Review requests | None | `REVIEW_INVITE_ENABLED` in Parse agent output. If you turn it on, the same neutral line is added to every non-red draft, whatever the customer's mood or the outcome. Never filter on satisfaction |
| Memory retention | 365 days | `MEMORY_RETENTION_DAYS` in Extract email data |
| Claim table retention | 90 days | `CLAIM_RETENTION_DAYS` in Build claim SQL |
| Webhooks | Header Auth required | credential `Support webhook auth` |

## Setup

### Credentials to create in n8n

| Credential type | Used by |
| --- | --- |
| Gmail OAuth2 (`gmailOAuth2`) | all Gmail nodes (read, labels, drafts, optional send) |
| Google Sheets OAuth2 (`googleSheetsOAuth2Api`) | the four log nodes |
| Postgres (`postgres`), with the `vector` extension available | claim table, memory table, both vector stores |
| Ollama (`ollamaApi`) with the `nomic-embed-text` model pulled | both embedding nodes (any embedding model works if you change both nodes and the vector tables together) |
| Header Auth (`httpHeaderAuth`) named `Order lookup webhook auth` | `shopify_orders` tool, and the webhook of your order lookup workflow |
| Header Auth (`httpHeaderAuth`) named `Support webhook auth` | the two webhooks of this workflow (`support-agent-run`, `support-memory-init`) |
| Header Auth (`httpHeaderAuth`) named `17TRACK token (header 17token)`: header name `17token`, value is your 17TRACK API key | `tracking17_tool` (same credential as in `shipping-exception-alerts`) |
| OpenAI API (`openAiApi`) | the `Chat model` node; add the Base URL option on the node to use another OpenAI-compatible provider |

All credential ids in the file are `REPLACE_ME`: re-select your credentials in each node after import.

### Configuration

Store-specific values live in one place: the CONFIGURATION block of **Extract email data** (store name, support address, store and tracking URLs, signature, own domains, default language, confidence threshold, memory retention). The agent prompt reads them from the item, so you do not edit the prompt itself. Other constants:

- **Classify Non-Support Email**: chat tool domain and trigger phrases, registrar, file-transfer, social video, messaging, ads, email marketing and review platform domains and sender names, out-of-office patterns, admin and support keywords, consumer mailbox providers. The shipped `.example` values match nothing real.
- **Parse agent output**: allowed promo codes (keep them identical to the codes in your playbook), maximum discount, escalation patterns, auto-send toggle, disclosure and review lines.
- **Build claim SQL**: claim retention.

Consumer mailbox providers: `CONSUMER_DOMAIN_STEMS` matches the first label of the sender domain, so `gmail.com`, `gmail.co.uk`, `hotmail.de`, `yahoo.com.au` and similar national domains all match; `CONSUMER_DOMAINS_EXACT` covers providers with a fixed domain. Senders outside these lists are treated as business-like by the weaker admin rules (cold outreach, no-reply services). If you would rather not use this business-sender check at all, set `BUSINESS_SENDER_RULES_ENABLED = false`: every sender is then treated as a possible customer by those rules.

### Placeholders to replace (`YOUR_...`)

| Placeholder | Where | Meaning |
| --- | --- | --- |
| `YOUR_N8N_HOST` | `shopify_orders` tool | host of the n8n instance that serves the order lookup webhook (`/webhook/orders-by-sender`) |
| `YOUR_LABEL_ID_URGENT`, `_ACTION`, `_TO_VALIDATE`, `_HANDLED`, `_PARTNERSHIPS` | Inject label_id_map, both Reset nodes, Determine Non-Support Label | Gmail label ids for red, yellow, blue, green and partnerships |
| `YOUR_LABEL_ID_AUTO_SENT` | Label auto-sent | label for auto-sent replies |
| `YOUR_LABEL_ID_ADMIN`, `_FILE_TRANSFER`, `_CHAT_TOOL`, `_SOCIAL_VIDEO`, `_MESSAGING`, `_SOCIAL_ADS`, `_EMAIL_MARKETING`, `_REVIEWS`, `_AUTO_ARCHIVE` | Determine Non-Support Label | Gmail label ids for the non-support categories |
| `YOUR_MASTER_LOG_DOCUMENT_ID`, `YOUR_MASTER_LOG_SHEET` | Log Master Sheet (two nodes) | master log |
| `YOUR_UNANSWERED_LOG_DOCUMENT_ID`, `YOUR_UNANSWERED_LOG_SHEET` | Log Unanswered Questions | unanswered questions queue |
| `YOUR_ROUTING_LOG_DOCUMENT_ID`, `YOUR_ROUTING_LOG_SHEET` | Log Non-Support Routing | routing log for non-support email |

### Gmail labels

Create the labels `Support/🔴 Urgent action`, `Support/🟡 Action needed`, `Support/🔵 To validate`, `Support/🟢 Handled by agent`, `Partnerships`, `Admin`, `Auto-archived`, `Chat tool`, `Social video`, `Messaging`, `Social ads`, `Email marketing`, `Reviews`, and `Support/🤖 Auto-sent` for auto-sent email. The Gmail Get Unread query excludes all of these names, so edit that query if you rename the labels. Get the label ids from the Gmail API or from an n8n Gmail "Get many labels" node.

### Tables and files to prepare

Claim table (not created by the workflow):

```sql
CREATE TABLE IF NOT EXISTS support_processed (
  message_id TEXT PRIMARY KEY,
  thread_id TEXT,
  label TEXT,
  processed_at TIMESTAMPTZ DEFAULT now()
);
```

Memory table: call the `support-memory-init` webhook once (POST, with the `Support webhook auth` header); it runs the `CREATE TABLE IF NOT EXISTS support_memory` node.

Vector tables: `kb_facts_embeddings` (KB1) and `kb_playbook_embeddings` (KB2), in the default layout of n8n's PGVector Store node, embedded with `nomic-embed-text` (768 dimensions). Filling them is not part of this workflow; use n8n's PGVector Store in insert mode or any loader. See `examples/kb1-facts.example.md` and `examples/kb2-playbook.example.md` for the expected content.

Sheets: create one header row per sheet, see `examples/sheet-headers.example.md`.

### Order lookup contract

The `shopify_orders` tool calls `https://YOUR_N8N_HOST/webhook/orders-by-sender`, a webhook you must provide (a second workflow, not included).

- **Input** (POST, JSON): `email` and, optionally, `order_number`. The `email` value is **fixed by an expression** on the sender address of the current email (`from_email`). It is never provided by the model, so the model cannot ask for another customer's orders. The model can only add an optional order number it read in the email; validate its format in the webhook workflow (for example `#` followed by digits) and ignore anything else.
- **Output**: JSON with `found`, `count` and `orders[]`, at most 5 most recent orders. Each order contains only: `name` (order number), `created_at_local`, `days_since_order`, `financial_status`, `fulfillment_status`, `items[]`, `tracking_numbers[]`, `tracking_urls[]`. No postal address, no phone number, no customer name, no payment detail.
- **Read-only**: the workflow behind it must not create or change anything in the shop. Give its Shopify credential read scopes only.
- **Authenticated**: the tool sends the Header Auth credential `Order lookup webhook auth`, and the webhook node must require the same header (Webhook node, Authentication: Header Auth). The response contains personal data, so never expose a lookup keyed on an email address without this authentication.

The sender address of an email is not proof of identity (it can be spoofed). That is why the response is limited to the fields above, why a human reviews every draft, and why nothing sensitive is returned. A lookup used by a public web chat, where the visitor is not authenticated, needs a different and stricter contract (for example order number and email both required).

### API versions

External API versions are kept in one field each, and defaults are meant to be updated: the chat model name (`Chat model` node, plus its Base URL option if you use another provider), the 17TRACK endpoint version in the URL of `tracking17_tool` (`/track/v2.4/`), and the Google and Postgres node versions of your n8n release. Check each provider's documentation and use a currently supported version. The 17TRACK request body is an array with one `{"number": "..."}` object, as in the 17TRACK v2 documentation; check it against the version you use.

### Testing without risk

- Import the workflow inactive. Use a test Gmail account, or a test label, so that real customers' threads are not touched: even in draft mode the workflow labels, marks read and archives threads.
- Run it once by hand with the manual webhook (`support-agent-run`, POST with the auth header) or execute from the editor.
- In the editor, check that the `shopify_orders` and `tracking17_tool` nodes resolve their expressions and credentials in your n8n version (the email field uses `$('Merge Memory').item.json.from_email`).
- Keep `AUTO_SEND_ENABLED = false`. Nothing is then sent; you only get drafts.
- Pin sample email data on Extract email data to replay specific cases (a parcel question, a legal threat, an out-of-office reply, a data deletion request).
- You can start with empty KB tables: the agent is told not to invent facts, and a low-confidence email lands under the blue label and in the unanswered sheet.

## Privacy

The workflow processes customers' email addresses, names, message text and order data. Before using it:

- **Choose the model provider carefully.** The chat model receives the customer's name, subject, body, thread snippets, memory and order data. Use a provider that signs a data processing agreement (DPA) and lets you pick a suitable region, or run a model yourself. Review the provider's retention and training terms. The embedding model runs on your own Ollama by default.
- **Minimization.** The model does not receive the sender's email address (the order lookup is bound to it by expression). The order lookup returns no address or name. The agent is told to write the memory summary without address, phone number, payment or health details. The body is capped at 8000 characters and thread messages at 500.
- **Retention of the customer memory.** `MEMORY_RETENTION_DAYS` (365 by default) in Extract email data. Rows whose last interaction is older are deleted by the `Read Support Memory` query, which runs on each processed email, and are never returned. If a quiet inbox processes no email, nothing is purged until the next one; schedule your own `DELETE FROM support_memory WHERE last_interaction_date < NOW() - INTERVAL '365 days'` if you need a hard limit. No profile is kept without an end date.
- **Retention of the logs.** The claim table is purged after `CLAIM_RETENTION_DAYS` (90 by default). The Google Sheets logs contain sender addresses, subjects and draft text and are NOT purged by the workflow: delete old rows on a schedule (a monthly Apps Script or a manual cleanup) matching your own policy, and restrict who can open the sheets.
- **Gmail drafts and labels** stay in the mailbox under your normal mailbox retention.
- **Data subject requests** (access, deletion): the workflow escalates them to a human and never answers them itself. When a person asks for deletion, also delete their row in `support_memory`, their rows in the sheets, and their thread if required.
- **Transparency.** Customers are told that an AI assistant was involved in each reply (see Default behaviors). Mention the use of AI in your privacy notice as well.

## Limits

- It does not update the knowledge base. The unanswered sheet is a queue for a person; `kb_update_needed` stays empty.
- The auto-send branch exists and is wired (reply, then label), but it ships disabled and has not been validated on its own. Treat it as a starting point. If you enable it, the disclosure line changes to the one that does not claim human review; keep legal and personal-data cases red and consider limiting the allow-list further.
- Replies are drafts for one human reviewer. The prompt is written around a single reviewer who finishes the action (refund, replacement, code creation); the workflow itself performs no business action in Shopify.
- The classifier lists, out-of-office patterns, review-star pattern and guardrail patterns are English only. For another language, translate and extend them. The prompt asks for a blue label when the customer's language differs from the default language. The disclosure line needs one entry per language.
- Guardrails are regular expressions. They catch common phrasings, not every possible wording. The escalation patterns are deliberately broad: a few harmless emails will turn red, which costs a human a quick look.
- Attachments are not read, only their MIME types are passed to the agent. Videos are not analyzed.
- Thread context is the last 5 messages as Gmail snippets (500 characters each), not full bodies.
- The memory table stores the latest one-line summary, a sentiment trail, the interaction count and the last order. The `open_issues` and `flags` columns are read but never written by this workflow.
- At most 10 emails per run. An email the model cannot finish within 18 iterations becomes a red item for manual handling.
- The partnership branch labels and logs, but does not write to the memory table.
- A notification from a platform (chat tool alert, review) also passes through the memory update, keyed on the sender address of the notification.

The prompt is a generic template: fill the knowledge bases (see `examples/`) with your own terms before relying on any draft.
