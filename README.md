# Newsbot — Telegram News & Market Sentiment Bot

A news and market-sentiment bot for a private Telegram group, built on
self-hosted n8n. Two behaviors from one shared sentiment engine:

- **Proactive** — a scheduled morning digest posted to the group, covering
  configured focus topics plus a sentiment read on tracked tickers.
- **Reactive** — a group-chat agent for on-demand questions ("latest on X",
  "sentiment on NVDA"), with conversation memory and live web search.

All LLM outputs are validated by deterministic QA checks, logged to a central
QA sheet, and reviewed weekly by an AI analyst that diagnoses failures against
a knowledge base of known signatures.

## Architecture

```
                    ┌───────────────────────────────────┐
                    │   sentiment_lookup (sub-workflow) │
                    │   Input normalize → Alpha Vantage │
                    │   → (throttled?) Marketaux        │
                    │   → Qwen dimensional scoring      │
                    │   → merge + net read → QA → log   │
                    └───────────────┬───────────────────┘
                        called by   │   called by
          ┌─────────────────────────┴──────────────────────────┐
          │                                                    │
┌─────────▼───────────┐                          ┌─────────────▼──────────────┐
│ News Bot A          │                          │ News Bot B                 │
│ Proactive Daily Push│                          │ Reactive On-Demand         │
│ cron 07:00          │                          │ Telegram Trigger           │
│ Tavily (per topic)  │                          │ → Has text (event filter)  │
│ → sentiment_lookup  │                          │ → Auth gate (chat IDs)     │
│ → Qwen compose      │                          │ → AI Agent (Claude)        │
│ → QA → log          │                          │    tools: tavily_search,   │
│ → sanitize → send   │                          │           sentiment_lookup │
│   (to group)        │                          │ → sanitize → reply         │
└─────────┬───────────┘                          └──────────────┬─────────────┘
          │                                                     │
          └────────────────► llm_qa_logger ◄────────────────────┘
                     (shared: normalize → Google Sheet append)
                                    │
                          ┌─────────▼───────────┐
                          │ QA Analyst (weekly) │
                          │ read sheet → stats  │
                          │ → Claude diagnosis  │
                          │ → Telegram DM report│
                          └─────────────────────┘
```

## Components

| Workflow | Trigger | Purpose |
|---|---|---|
| `sentiment_lookup (sub-workflow)` | Called by A & B | Ticker news → dimensional sentiment scoring → per-ticker net read |
| `News Bot A - Proactive Daily Push` | Cron `0 7 * * *` | Tavily news per focus topic + sentiment block → HTML digest → Telegram group |
| `News Bot B - Reactive On-Demand` | Telegram webhook | Agent (Claude Sonnet) with search + sentiment tools, windowed memory, non-text event filter, chat allow-list |
| `llm_qa_logger (shared sub-workflow)` | Called by A & sub-workflow | Appends standardized QA records to a Google Sheet |
| `QA Analyst - Weekly Pipeline Health` | Cron `0 8 * * 1` | Reads QA sheet → deterministic stats → Claude diagnosis against failure-signature KB → plain-text DM report |

## Models & data sources

| Role | Model / API | Notes |
|---|---|---|
| Dimensional scoring | `qwen3.5:9b` via Ollama (`/api/chat`, `format: json`, `think: false`) | Local, free, high-volume |
| Digest compose | `qwen3.5:9b` via Ollama | Prose HTML, `think: false` |
| Reactive agent | `claude-sonnet-5` (Anthropic) | Tool routing + synthesis |
| QA analyst | `claude-sonnet-5` (Anthropic) | Interprets pre-computed stats only |
| News discovery | Tavily (`topic=news/finance`, `time_range`) | Free tier 1,000 credits/mo |
| Ticker sentiment | Alpha Vantage `NEWS_SENTIMENT` | Free tier 25 req/day — one batched call per run |
| Sentiment fallback | Marketaux | Kicks in on AV throttle (detected via `Note`/`Information`) |

## Scoring methodology

Each article is scored by the local model on three anchored dimensions, then
composited:

```
score = polarity (−1..1) × materiality (0..1) × confidence (0..1)
```

Rumors and trivial mentions decay toward neutral structurally (low confidence
or materiality) rather than by rule. Per-ticker `net` pools provider scores
and model scores; labels at ±0.15. The QA validator independently recomputes
the arithmetic, checks ranges and label bands, and measures model-vs-provider
divergence per ticker (divergence is surfaced, not smoothed — it's signal).

Note: sentiment coverage for ETFs and thinly-covered symbols is often sparse;
the digest flags thin reads (`n` of 1–2) rather than hiding them.

## QA & monitoring

Three layers:

**1. Inline validators** (Code nodes) run deterministically, never block, and
attach a `qa` record to the data:

- **Validator A** (`sentiment_lookup`): JSON parse/coverage, range checks,
  `polarity×materiality×confidence == score` recompute, label-band check,
  silent-failure (all-nulls) detector, per-ticker divergence.
- **Validator B** (`News Bot A`): length bounds, banned strings (preamble,
  advice, markdown leakage), **hallucinated-link detector** (every href must
  exist in the payload), unknown-ticker detector, HTML balance.

The `Telegram chunk` nodes in both bots additionally **sanitize outbound
HTML** (escape stray angle brackets, balance tags per chunk) so malformed
markup degrades to visible text instead of a Telegram 400.

**2. Central log.** Validators feed `llm_qa_logger` (fire-and-forget, Wait
for Sub-Workflow OFF; Sheets node mapping mode must be **Map Automatically**),
which appends to a Google Sheet:

```
ts | project | workflow | node | model | status | flags | metrics
```

**3. Weekly AI analyst.** `QA Analyst` reads the sheet, computes stats
deterministically in a Code node (fail rates, flag frequencies, early-vs-late
trend, mean divergence per ticker), and has Claude diagnose against the
failure-signature knowledge base — with instructions to hedge on thin samples
and never invent numbers. Report arrives as a plain-text Telegram DM (no
parse mode, so the diagnostician can't fail on formatting itself).

**Known blind spot:** the analyst sees only rows that reached the Sheet. A
failure that prevents the log write (e.g. a broken Sheets node) is invisible
to it — the global n8n error workflow is the net for that layer. Planned fix:
give the analyst the n8n API as a tool to cross-reference failed executions.

## Setup

### Prerequisites
- Self-hosted n8n (built on 2.29.x) reachable via public HTTPS for Telegram
  webhooks (e.g. Tailscale Funnel), `WEBHOOK_URL` set.
- Ollama on the LAN with the scoring model pulled; must bind `0.0.0.0` if n8n
  runs in Docker (`OLLAMA_HOST=0.0.0.0`). Endpoint used: `http://<host>:11434/api/chat`.
- A Telegram bot (via @BotFather) added to the target group (see Telegram
  group setup below).

### Credentials (n8n → Credentials)
- Telegram API (trigger + send nodes across bots and analyst)
- Anthropic API (agent + analyst chat models)
- Tavily (predefined credential type)
- Alpha Vantage + Marketaux (Query Auth credentials — **never inline keys**)
- Google Sheets OAuth (QA logger + analyst)

### Telegram group setup
1. Create the group, add the bot.
2. **Group Privacy:** @BotFather → `/mybots` → Bot Settings → Group Privacy →
   **Disable** so the bot receives plain group messages (with privacy ON,
   bots only reliably receive /commands, @mentions, and replies). After
   changing this, the bot **must be removed from the group and re-added**
   for the setting to take effect.
3. **Get the group chat ID** (a negative number): post in the group, then
   read `message.chat.id` from the Telegram Trigger input in n8n's
   Executions list. `getUpdates` returns 409 while a webhook is registered,
   so use the execution log instead.
4. Put the group ID in **News Bot A → CONFIG → `chat_id`** and in
   **News Bot B → Auth gate** (OR'd with any personal chat IDs that should
   also be allowed, e.g. for DM access).
5. **Supergroup caveat:** Telegram silently upgrades groups to supergroups
   when certain settings change or members are added — the chat ID changes
   to a new `-100…` value and the old one goes dead. If the bot goes silent
   in the group, re-grab the ID from a fresh execution and update both spots.

### Import order
1. `workflows/sentiment_lookup.json` — save, note its workflow ID
2. `workflows/llm_qa_logger.json` — save, note its ID; create the QA Sheet
   with the header row above; set the Sheets node to Map Automatically
3. `workflows/news_bot_a_proactive.json` — point its `sentiment_lookup` and
   logger Execute Workflow nodes at the IDs from steps 1–2
4. `workflows/news_bot_b_reactive.json` — same re-pointing for the
   `sentiment_lookup` tool; set the Auth gate chat IDs; activate to register
   the webhook
5. `workflows/qa_analyst.json` — point Read QA log at the same Sheet;
   runs weekly, or execute manually anytime for an on-demand health report

### Configuration
All reader-facing config lives in **News Bot A → CONFIG** node:
`focus_topics` (drives Tavily searches), `tracked_tickers`, `chat_id`
(the group). The agent's focus areas are also stated in **News Bot B →
AI Agent** system message — keep the two aligned. Model tags are set inside
the two Ollama HTTP nodes and the QA validators' `model` field.

**Access control (Bot B):** the `Has text` IF node drops non-text group
events (joins, photos, pins) before they reach the agent; the `Auth gate`
allow-lists chat IDs. Gating is per-chat, not per-person — anyone in an
allowed group can use the bot and spend API credits. For person-level
control, gate on `message.from.id` instead.

## Testing

See `docs/test-plan.md`. Short version: prove `sentiment_lookup` in isolation
(including forcing the Marketaux fallback once), then Bot A manually, then
Bot B live. Known failure signatures:

- **All `model_score` null** → scoring call broken (model tag, Ollama
  unreachable, or reasoning leaked into JSON). The merge fails silently by
  design; Validator A flags it.
- **Telegram 400 "can't parse entities"** → broken HTML from the model; the
  chunk-node sanitizer now absorbs this (degrades to visible text); Validator
  B logs the underlying flag either way.
- **Tool schema rejection from Anthropic** (`input_schema.properties` pattern)
  → a tool is exposing a malformed schema; the `sentiment_lookup` tool uses an
  explicit manual schema (`tickers`: string) for this reason.
- **"No ticker received"** → the agent called sentiment without a symbol; the
  error text instructs it to resolve the ticker via search or fall back to
  news-only (private companies).
- **Bot silent in the group, no executions** → Group Privacy still ON, or the
  bot wasn't removed/re-added after changing it, or the group upgraded to a
  supergroup and the chat ID changed (see Telegram group setup).
- **Executions stop at `Has text` or `Auth gate`** → non-text event filtered
  (by design), or the incoming `chat.id` doesn't match the allow-list — read
  the actual ID from the execution input.
- **QA rows missing despite incidents** → logger itself failed (check
  `llm_qa_logger` executions; historically: Sheets mapping mode left on
  Define Below).

## Security

- API keys live in n8n credentials only. Workflow exports must contain
  credential IDs, never raw keys — check before committing:
  `grep -rE "tvly-|apikey|api_token" workflows/` should return nothing
  suspicious.
- Bot B is gated to allow-listed Telegram chats (Auth gate node) so strangers
  can't spend API credits; every member of an allowed group has access.
- If a key ever lands in a chat, log, or commit: rotate it.

## Roadmap

- n8n API as a QA Analyst tool — cross-reference failed executions against
  the Sheet (closes the logger blind spot).
- 👍/👎 inline-keyboard feedback on digests/answers → logged next to QA rows →
  few-shot calibration of the scoring prompt.
- Relevance/recency-weighted net aggregation (fields already flow; merge-node
  change only).
- Sentiment for private/unlisted companies via Tavily-fed scoring branch.
- Golden-set eval harness for model upgrades (9B vs 14B/30B comparison).
