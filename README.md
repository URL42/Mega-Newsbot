# Mega Newsbot — Telegram News & Market Sentiment Bot

A personal news and market-sentiment bot for one reader, built on self-hosted
n8n. Two behaviors from one shared sentiment engine:

- **Proactive** — a scheduled morning digest covering configured focus topics
  plus a sentiment read on tracked tickers.
- **Reactive** — a Telegram DM agent for on-demand questions ("latest on X",
  "sentiment on NVDA"), with conversation memory and live web search.

All LLM outputs are validated by deterministic QA checks and logged to a
central QA sheet for drift/consistency monitoring.

## Architecture

```
                    ┌──────────────────────────────────┐
                    │   sentiment_lookup (sub-workflow) │
                    │   Input normalize → Alpha Vantage │
                    │   → (throttled?) Marketaux        │
                    │   → Qwen dimensional scoring      │
                    │   → merge + net read → QA → log   │
                    └───────────────┬──────────────────┘
                        called by   │   called by
          ┌─────────────────────────┴─────────────────────────┐
          │                                                    │
┌─────────▼──────────┐                          ┌──────────────▼─────────────┐
│ News Bot A          │                          │ News Bot B                 │
│ Proactive Daily Push│                          │ Reactive On-Demand         │
│ cron 07:00          │                          │ Telegram Trigger           │
│ Tavily (per topic)  │                          │ → Auth gate (chat ID)      │
│ → sentiment_lookup  │                          │ → AI Agent (Claude)        │
│ → Qwen compose      │                          │    tools: tavily_search,   │
│ → QA → log          │                          │           sentiment_lookup │
│ → Telegram push     │                          │ → Telegram reply           │
└─────────┬───────────┘                          └──────────────┬─────────────┘
          │                                                     │
          └────────────────► llm_qa_logger ◄────────────────────┘
                     (shared: normalize → Google Sheet append)
```

## Components

| Workflow | Trigger | Purpose |
|---|---|---|
| `sentiment_lookup (sub-workflow)` | Called by A & B | Ticker news → dimensional sentiment scoring → per-ticker net read |
| `News Bot A - Proactive Daily Push` | Cron `0 7 * * *` | Tavily news per focus topic + sentiment block → HTML digest → Telegram |
| `News Bot B - Reactive On-Demand` | Telegram webhook | Agent (Claude Sonnet) with search + sentiment tools, windowed memory, chat-ID auth gate |
| `llm_qa_logger (shared sub-workflow)` | Called by A & sub-workflow | Appends standardized QA records to a Google Sheet |

## Models & data sources

| Role | Model / API | Notes |
|---|---|---|
| Dimensional scoring | `qwen3.5:9b` via Ollama (`/api/chat`, `format: json`, `think: false`) | Local, free, high-volume |
| Digest compose | `qwen3.5:9b` via Ollama | Prose HTML, `think: false` |
| Reactive agent | `claude-sonnet-5` (Anthropic) | Tool routing + synthesis |
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

## QA & monitoring

Deterministic validators (Code nodes) run inline, never block, and attach a
`qa` record to the data:

- **Validator A** (`sentiment_lookup`): JSON parse/coverage, range checks,
  `polarity×materiality×confidence == score` recompute, label-band check,
  silent-failure (all-nulls) detector, per-ticker divergence.
- **Validator B** (`News Bot A`): length bounds, banned strings (preamble,
  advice, markdown leakage), **hallucinated-link detector** (every href must
  exist in the payload), unknown-ticker detector, HTML balance (Telegram 400
  preventer).

Both feed `llm_qa_logger` (fire-and-forget, Wait for Sub-Workflow OFF), which
appends to a Google Sheet with columns:

```
ts | project | workflow | node | model | status | flags | metrics
```

`status` ∈ pass / warn / fail. Review weekly for drift; `divergence` and
`math_bad` trends are the early-warning metrics.

## Setup

### Prerequisites
- Self-hosted n8n (built on 2.29.x) reachable via public HTTPS for Telegram
  webhooks (e.g. Tailscale Funnel), `WEBHOOK_URL` set.
- Ollama on the LAN with the scoring model pulled; must bind `0.0.0.0` if n8n
  runs in Docker (`OLLAMA_HOST=0.0.0.0`). Endpoint used: `http://<host>:11434/api/chat`.
- A Telegram bot (via @BotFather) and the reader's numeric chat ID.

### Credentials (n8n → Credentials)
- Telegram API (trigger + both send nodes)
- Anthropic API (agent chat model)
- Tavily (predefined credential type)
- Alpha Vantage + Marketaux (Query Auth credentials — **never inline keys**)
- Google Sheets OAuth (QA logger)

### Import order
1. `workflows/sentiment_lookup.json` — save, note its workflow ID
2. `workflows/llm_qa_logger.json` — save, note its ID; create the QA Sheet
   with the header row above
3. `workflows/news_bot_a_proactive.json` — point its `sentiment_lookup` and
   logger Execute Workflow nodes at the IDs from steps 1–2
4. `workflows/news_bot_b_reactive.json` — same re-pointing for the
   `sentiment_lookup` tool; set the Auth gate chat ID; activate to register
   the webhook

### Configuration
All reader-facing config lives in **News Bot A → CONFIG** node:
`focus_topics` (drives Tavily searches), `tracked_tickers`, `chat_id`.
The agent's focus areas are also stated in **News Bot B → AI Agent** system
message — keep the two aligned. Model tags are set inside the two Ollama HTTP
nodes and the QA validators' `model` field.

## Testing

See `docs/test-plan.md`. Short version: prove `sentiment_lookup` in isolation
(including forcing the Marketaux fallback once), then Bot A manually, then
Bot B live. Known failure signatures:

- **All `model_score` null** → scoring call broken (model tag, Ollama
  unreachable, or reasoning leaked into JSON). The merge fails silently by
  design; Validator A flags it.
- **Telegram 400 "can't parse entities"** → broken HTML from the model;
  Validator B's balance check is the early warning.
- **Tool schema rejection from Anthropic** (`input_schema.properties` pattern)
  → a tool is exposing a malformed schema; the `sentiment_lookup` tool uses an
  explicit manual schema (`tickers`: string) for this reason.
- **"No ticker received"** → the agent called sentiment without a symbol; the
  error text instructs it to resolve the ticker via search or fall back to
  news-only (private companies).

## Security

- API keys live in n8n credentials only. Workflow exports must contain
  credential IDs, never raw keys — check before committing:
  `grep -rE "tvly-|apikey|api_token" workflows/` should return nothing
  suspicious.
- Bot B is gated to allow-listed Telegram chat IDs (the Auth gate node) so
  strangers can't spend API credits.
- If a key ever lands in a chat, log, or commit: rotate it.

## Roadmap

- 👍/👎 inline-keyboard feedback on digests/answers → logged next to QA rows →
  few-shot calibration of the scoring prompt.
- Relevance/recency-weighted net aggregation (fields already flow; merge-node
  change only).
- Sentiment for private/unlisted companies via Tavily-fed scoring branch.
- Golden-set eval harness for model upgrades (9B vs 14B/30B comparison).
