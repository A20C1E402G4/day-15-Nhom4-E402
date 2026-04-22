# Worksheet 3 — Cost Optimization Debate
## Logistics Operations Agent · Nhóm 4 · Zone E402 · 2026-04-22

---

## 1. Baseline recap (from Worksheet 2)

| Item | Value |
|---|---|
| Total cost MVP (with hidden) | **$1 112/month** |
| Total cost Growth 5× | **$3 273/month** |
| LLM API cost MVP | $157/month |
| Biggest LLM waste | Opus 4.7 = 5% of requests, 62% of LLM spend ($98/mo) |
| Cache already saving | ~$63/mo vs. no-cache baseline |
| Architecture already in place | Semantic cache (Redis) · Model routing (LiteLLM) · on-prem vLLM fallback |

> Note: Semantic caching and model routing (Haiku/Sonnet/Opus) are **already designed into W1's architecture**. The three strategies below build on top of those — they are optimisations the team adds, not replacements.

---

## 2. Strategy evaluation shortlist

| Strategy | Applicable to our system? | Estimated saving | Complexity | When |
|---|---|---|---|---|
| Anthropic Prompt Caching (prefix cache) | Yes — SOP context repeats across many Sonnet/Opus calls | 30–40% on Sonnet/Opus input tokens | Low — API flag only | Now |
| Tighten Opus routing threshold | Yes — borderline complex queries currently escalate to Opus | ~$40–60/mo at MVP | Low — classifier threshold | Now |
| Prompt compression (RAG context) | Yes — SOP chunks are 800/1 400 tokens, compressible | 15–25% input tokens for SOP class | Medium — pre-processing step | Growth |
| Fine-tune vLLM on company data | Yes — on-prem A100 already exists (W1) | 50–70% on SOP class vs. Sonnet | High — data + training pipeline | Scale |
| User tiering (CS vs. supervisor) | Possible — supervisors need Opus, CS agents rarely do | 10–20% LLM cost | Medium — auth + routing logic | Growth |

---

## 3. Three chosen strategies

### Strategy 1 — Do now: Anthropic Prompt Caching

**What it saves:**
The Logistics Operations Agent sends an identical system prompt (~400 tokens) and near-identical SOP context chunks (~800 tokens) on every Sonnet/Opus call. Anthropic's prompt caching API (`cache_control: ephemeral`) caches this prefix server-side for 5 minutes. Subsequent calls within the TTL pay ~10% of normal input price on the cached portion.

**Calculation:**
- Sonnet calls: 200/day × 1 400 in-tok. Cached portion ≈ 1 000 tok (system prompt + SOP context).
  Saving per call = 1 000 × ($3.00 − $0.30)/1M = $0.0027. Monthly = 200 × 30 × $0.0027 = **$16.20/mo**
- Opus calls: 40/day × 2 200 in-tok. Cached portion ≈ 1 400 tok.
  Saving per call = 1 400 × ($15.00 − $1.50)/1M = $0.0189. Monthly = 40 × 30 × $0.0189 = **$22.68/mo**
- **Total saving: ~$39/mo (25% of LLM API cost)**

**Benefit:** Zero infrastructure change — add one API parameter to the LiteLLM proxy config. No latency impact.

**Trade-off:** Cache TTL is 5 minutes. Rush-hour bursts (800 req/h) will benefit most; off-peak single queries may not hit cache. Must pin prompt version — any system prompt change invalidates cache and costs one full-price call.

**Metric to confirm it's working:** Langfuse cost dashboard — `cache_read_input_tokens` should represent ≥ 60% of Sonnet/Opus input tokens within first week.

---

### Strategy 2 — Do now: Tighten Opus routing threshold

**What it saves:**
The Query Classifier (W1) currently labels ~5% of requests as `escalate → Opus`. W2 showed this 5% costs $98/mo (62% of LLM spend). Many of these are "medium-hard" tickets that Sonnet 4.6 can handle with a slightly more detailed prompt. Raising the classifier confidence threshold for Opus from the current implicit level to **cosine similarity ≥ 0.85 against known hard-escalation patterns** reduces Opus volume to ~2–3% of requests.

**Calculation:**
- Reduce Opus from 40 req/day to 18 req/day (−55%).
- Saving = (40 − 18) × 30 × (2 200 × $15 + 600 × $75)/1M = 22 × 30 × ($33 + $45)/1M
  = 660 × $0.078 = **$51.48/mo**
- Sonnet absorbs displaced queries (22 req/day extra):
  Extra Sonnet cost = 22 × 30 × (1 400 × $3 + 350 × $15)/1M = 660 × ($4.20 + $5.25)/1M
  = 660 × $0.009 = $5.99/mo
- **Net saving: ~$45/mo**

**Benefit:** Straightforward classifier tuning — no new infrastructure. Combined with Strategy 1, LLM API cost drops from $157 → ~$73/mo (−53%).

**Trade-off:** Risk of misrouting genuinely hard tickets to Sonnet → lower response quality → higher human escalation rate. Must monitor: if human review rate rises above 20%, loosen threshold back.

**Metric to confirm:** Opus call rate (Langfuse) · Human review escalation rate (PostgreSQL audit table). Gate: if human review > 20%, revert threshold.

---

### Strategy 3 — Later @ Growth (≥ 2 000 req/day): Prompt compression for SOP class

**What it saves:**
At Growth tier (4 000 req/day), SOP questions grow to ~1 000/day. Each call sends top-5 SOP chunks (~800 tokens of RAG context). A lightweight Haiku 4.5 summarisation step compresses those 5 chunks to a single ~350-token dense summary before the Sonnet call — reducing input tokens from 1 400 → ~900.

**Calculation at Growth (1 000 SOP req/day):**
- Saved tokens per call: 500 input tokens.
- Saving = 1 000 × 30 × 500 × $3/1M = **$45/mo** on Sonnet input.
- Cost of compression call: 1 000 × 30 × (800 in × $1 + 350 out × $5)/1M = 30 000 × $1.55/1M = **$46.50/mo**
- Net at Growth: approximately **break-even on token cost**, but the real saving is **latency** — Sonnet processes 35% fewer tokens, reducing p95 from ~4.2s → ~2.8s, which directly reduces peak-hour SLA breach risk.

**Benefit:** At scale the benefit shifts from cost to latency — aligns with the p95 ≤ 5s SLO (W1 §8). Also reduces context noise: compressed summary removes boilerplate from SOP chunks, improving faithfulness score.

**Trade-off:** Extra Haiku call adds ~200ms latency before the Sonnet call. Risk of summary losing critical detail from long SOPs. Must validate: run RAGAS on compressed vs. full-context and block rollout if faithfulness drops > 0.03.

**Threshold to trigger rollout:** SOP query volume ≥ 800/day AND p95 latency > 3.5s on SOP class (i.e., approaching SLO headroom).

**Metric to confirm:** p95 latency for SOP class (Prometheus) · RAGAS faithfulness delta (Langfuse) · Sonnet input token count per call (Langfuse).

---

## 4. Expected combined saving

| Scenario | Baseline | After S1 (prompt cache) | After S1 + S2 (tighten Opus) | After S1 + S2 + S3 (at Growth) |
|---|---:|---:|---:|---:|
| LLM API monthly | $157 | $118 (−25%) | $73 (−54%) | $220* (−30% vs. Growth baseline) |
| Total with hidden | $1 112 | $1 073 | $1 028 | $2 480* |

*Growth baseline without optimisations is ~$3 273/mo. After all three strategies at Growth: ~$2 480/mo (−24% vs. Growth baseline).

---

## 5. Strategies we explicitly rejected and why

| Strategy | Reason rejected |
|---|---|
| Full self-hosted (replace Sonnet/Opus entirely) | On-prem A100 already committed to fallback. Adding a second GPU for serving primary traffic is $15K+ capex — not justified until token volume > 1M/day (~5× Growth). |
| Smaller model for SOP questions (Haiku replacing Sonnet) | RAGAS faithfulness tests showed Haiku scores 0.61 on SOP retrieval vs. Sonnet 0.84. Below the 0.80 guardrail threshold from W1. Quality gate prevents this. |
| Aggressive cache TTL increase (5min → 30min) | Order status changes in real time from TMS. Stale status responses are a direct SLA violation — the exact failure mode that drives CS complaints (W0 risk statement). |
| User tiering (free/premium model split) | All 85 users are internal staff on the same plan. No pricing structure to enforce tiering without a new access-control layer — deferred to Phase 2. |
