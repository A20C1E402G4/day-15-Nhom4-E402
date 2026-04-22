# Model price anchor — fixed at 2026-04-22

Use these numbers consistently across Worksheets 2–3 and the slide deck so all costs are comparable. Verify against the official pricing page before presenting (see `day15.md §Tài Liệu Tham Khảo`).

## Frontier / mid / cheap tiers — USD per 1M tokens

| Model | Tier | Input ($/1M) | Output ($/1M) | Notes |
|---|---|---:|---:|---|
| Claude Opus 4.7 | Frontier | 15.00 | 75.00 | Use for complex reasoning / hard tickets |
| Claude Sonnet 4.6 | Mid | 3.00 | 15.00 | Default workhorse |
| Claude Haiku 4.5 | Cheap | 1.00 | 5.00 | Simple classify / routing / FAQ |
| GPT-4o | Mid | 2.50 | 10.00 | Alt mid-tier |
| GPT-4o-mini | Cheap | 0.15 | 0.60 | Cheapest capable OpenAI tier |
| Gemini 2.0 Flash | Cheap | 0.10 | 0.40 | Lowest cost, high speed |

## Embedding + vector DB

| Component | Anchor price |
|---|---|
| OpenAI text-embedding-3-small | $0.02 / 1M tokens |
| Cohere embed-v3 | $0.10 / 1M tokens |
| Pinecone serverless | ~$0.33 / GB / month + $8 / 1M reads |
| Qdrant Cloud (1 vCPU, 4 GB) | ~$25 / month |

## Infra reference (Logistics Ops tier)

| Component | MVP | Growth (5–10×) |
|---|---|---|
| App hosting (Railway / Cloud Run) | $20–40/mo | $200–400/mo |
| Redis (cache + queue) | $10/mo | $80/mo |
| Vector DB (Pinecone serverless) | $15/mo | $100/mo |
| Monitoring (Langfuse Cloud / Grafana) | $0–30/mo | $100–300/mo |
| Logging (Logtail / Loki) | $0–20/mo | $50–150/mo |

## Rule-of-thumb multipliers

- **Hidden-cost multiplier:** API cost × 1.5–2× for retries + guardrails LLM calls + eval pipeline. (Source: `day15.md §3`.)
- **Eval run cost:** 20 cases × 4 RAGAS metrics × judge ≈ 80 API calls ≈ $1–4 per run. (Source: `day14.md §1`.)
- **Self-hosted break-even:** ~1M tokens/day. Below → cloud API cheaper when GPU + ops are counted.
