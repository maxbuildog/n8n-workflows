# n8n workflows

n8n workflows for an online store, published as importable templates. Each one
solves a problem a small business actually has: customer emails, shop chat,
social comments, late parcels, stock-outs, a cluttered task board.

Each one is derived from a workflow that runs unattended on a real store,
generalized so you can adapt it: prompts and business rules are templates to
fill in. The design choices in each README come from what broke along the way.

| Workflow | What it does | Main services |
|---|---|---|
| [ai-email-support-agent](ai-email-support-agent/) | Reads the support inbox every 3 minutes and drafts answers grounded in your knowledge base and order data (auto-send for safe cases is off by default) | Gmail, Shopify, Postgres + pgvector, any OpenAI-compatible model |
| [ecommerce-chat-agent](ecommerce-chat-agent/) | Website chat agent that answers product questions and looks up an order only when the customer gives both its number and its email | Shopify, Postgres, OpenAI-compatible model |
| [social-comments-agent](social-comments-agent/) | Answers questions under Facebook and Instagram posts and ads, hides only abuse, spam, scams, personal data and the authors you block yourself, and sends criticism and safety concerns to a human instead of hiding them | Meta Graph API, Telegram, OpenAI-compatible model |
| [shipping-exception-alerts](shipping-exception-alerts/) | Catches delivery problems from carrier webhooks before the customer writes in | 17TRACK, Shopify |
| [low-stock-alerts](low-stock-alerts/) | Warns before a product runs out, with thresholds computed from real sales | Shopify, Telegram |
| [notion-nightly-archive](notion-nightly-archive/) | Moves finished tasks out of a Notion TODO database every night | Notion, Telegram |

## How to use one

1. In n8n, **Workflows → Import from file**, pick the `workflow*.json` of the folder.
2. Create the credentials listed in that folder's README (every credential in the
   file is set to `REPLACE_ME`).
3. Replace each `YOUR_…` placeholder. The README lists all of them.
4. Run it by hand first. The email agent ships in draft mode; the chat and
   comment agents answer live, so test them on a staging site or a private Page
   before you point them at customers.

## Principles behind them

- **Deterministic first, AI second.** Sorting, deduplication and the decision to
  send are rules you can read. The model drafts; it does not decide alone.
- **Ground every answer.** Agents look up the knowledge base and the order before
  they write, and say so when they do not know.
- **A human stays on what engages the brand.** Doubtful cases, criticism and
  anything about safety become drafts or alerts for a person, never a silent hide.
- **Say it is an AI.** Messages written for customers carry a short, configurable
  AI disclosure, on by default.
- **Webhooks that trigger an action check who is calling.** They verify a signature
  or a header token before doing anything. The public chat widget cannot keep a
  secret: it filters the origin and the message length, and must sit behind a
  proxy that limits the request rate.
- **Collect less, keep it shorter.** Models only see what they need, and memory
  tables have a retention period you set.
- **Leave a trail.** Every run is logged, and questions the agent could not
  answer land in a review queue, so a human can grow the knowledge base.

## Need one adapted to your business?

I build and run automations and AI agents like these.
[LinkedIn](https://www.linkedin.com/in/maxbuildog) ·
[Malt](https://www.malt.fr/profile/maximepons)

MIT licensed.
