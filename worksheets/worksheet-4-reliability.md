# Worksheet 4 — Scaling & Reliability Tabletop
## Logistics Operations Agent · Nhóm 4 · Zone E402 · 2026-04-22

---

## 1. System context recap (from Worksheets 1–2)

| Item | Value |
|---|---|
| Normal load | 800 req/day · ~33 req/h avg |
| Peak load | 08–10h and 16–19h · 4× avg = ~133 req/h |
| Peak spike scenario | Sudden 10× = ~330 req/h |
| SLO: p95 response time | ≤ 5 s |
| SLO: availability | 99.5% / month (≤ 3.6h downtime) |
| Fallback chain | Claude API → GPT-4o-mini → on-prem vLLM → rule-based + auto-ticket |
| Architecture | GKE Autopilot · HPA 3→10 replicas · Redis queue · Cloud VPN HA |
| Critical data | PII in every order — scrubber must never skip even under load |

---

## 2. Failure scenario matrix

### Scenario A — Traffic spike 10× (330 req/h sustained 30 min)

| Dimension | Detail |
|---|---|
| **Trigger** | Flash sale announcement by a partner e-commerce platform; all CS agents open agent simultaneously at 16:00 |
| **User impact** | Response time climbs from p95 ~2s → ~12s; some requests timeout (default Nginx timeout 60s); CS agents see blank or spinner |
| **Short-term reaction (< 5 min)** | HPA scales from 3 → 10 pods (GKE Autopilot ~90s to provision). Redis queue absorbs burst — requests enqueue, users see "đang xử lý…" instead of 5xx. LiteLLM Proxy rate-limits at 200 req/min to Claude API to avoid 429 errors. |
| **Long-term fix** | Pre-scale on schedule: Kubernetes CronJob boosts HPA minimum replicas to 7 at 07:45 and 15:45 daily. Integrate TMS webhook — if shipment volume spikes > 2× in 30 min, trigger pre-scale alert. |
| **Metrics to watch** | GKE pod count · Redis queue depth · Nginx active connections · p95 latency (Prometheus) · Claude API 429 rate |
| **Acceptable degradation** | If queue depth > 500: drop non-urgent SOP queries (return "please retry in 2 min") and keep only status-lookup and ticket-action classes live. |

---

### Scenario B — Claude API provider outage (5xx for > 2 min)

| Dimension | Detail |
|---|---|
| **Trigger** | Anthropic API returns 5xx or connection timeout; no warning. Happens mid-rush-hour. |
| **User impact** | All Sonnet/Opus responses fail. Status-lookup cache hits still serve fine (42% of requests). Non-cached queries fail silently unless fallback fires. |
| **Short-term reaction (< 30 s)** | LiteLLM Proxy: retry × 2 with exponential jitter (500ms, 1.5s). After 2 failures → route to OpenAI GPT-4o-mini (secondary). GPT-4o-mini serves status-lookup (cache miss) and SOP class at reduced quality. |
| **If GPT-4o-mini also fails (< 2 min)** | LiteLLM routes to on-prem vLLM over Cloud VPN HA. Llama 3.1 8B handles simple requests. Complex escalation class is suspended — auto-ticket created in Freshdesk, supervisor notified via Slack. |
| **Circuit breaker parameters** | OPEN if: ≥ 3 failures in 60s window. HALF-OPEN after 30s: probe with 1 test request. CLOSE if probe succeeds. (Implementation: LiteLLM built-in circuit breaker config.) |
| **Long-term fix** | Quarterly chaos drill: deliberately cut Claude API in staging and verify fallback chain fires within SLO. Add Anthropic status page webhook to Alertmanager. |
| **Metrics to watch** | Claude API error rate · LiteLLM fallback counter · on-prem vLLM request rate · Circuit breaker state (Prometheus custom metric) |
| **PII constraint during fallback** | GPT-4o-mini and on-prem vLLM must receive only PII-masked payloads — same edge scrubber path, no bypass under fallback. |

---

### Scenario C — Response latency creep (p95 drifts from 2s → 6s over 3 days)

| Dimension | Detail |
|---|---|
| **Trigger** | SOP vector index grows as new runbooks are added; Qdrant top-k retrieval slows. Or: Elasticsearch log queries slow down the same node. Not an acute outage — a slow drift. |
| **User impact** | CS agents notice slowness; p95 breaches 5s SLO; Alertmanager fires P2 Slack alert. No hard errors — requests complete but slowly. |
| **Short-term reaction** | Grafana shows which class is slowest. If SOP class: switch from top-k=5 → top-k=3 (reduce retrieval scope). If all classes: check Redis cache miss rate — if cache hit dropped below 40%, investigate TTL or cosine threshold drift. |
| **Root cause investigation checklist** | 1. Check Qdrant query time (p95 retrieval). 2. Check LLM TTFT (time-to-first-token) from Langfuse. 3. Check Nginx → Orchestrator → LiteLLM latency breakdown. 4. Check GKE pod CPU/memory saturation. 5. Check Cloud VPN HA bandwidth (if on-prem is in the path). |
| **Long-term fix** | Index optimisation: enable Qdrant HNSW payload indexing on `category` field to pre-filter by query class before vector search. Add Logstash index lifecycle policy to archive logs older than 30d (prevents Elasticsearch node slowdown). Set Prometheus alert on p95 at 3.5s (before SLO breach at 5s). |
| **Metrics to watch** | p95 per request class (Prometheus) · Qdrant query latency · LLM TTFT (Langfuse) · Redis cache hit rate · Elasticsearch node heap usage |

---

## 3. Fallback chain — full specification

```
Request arrives at LiteLLM Proxy
│
├─ [1] Redis semantic cache check
│       HIT (cosine ≥ 0.92, TTL valid) ──► return cached response immediately (0 LLM cost)
│       MISS ──► continue
│
├─ [2] Route to Claude API (primary)
│       Success ──► return response
│       5xx / timeout > 8s ──► retry × 2 (jitter 500ms, 1.5s)
│       Retry exhausted ──►
│
├─ [3] Route to OpenAI GPT-4o-mini (secondary)
│       Success ──► return response (append disclaimer: "powered by backup model")
│       5xx / timeout ──►
│
├─ [4] Route to on-prem vLLM · Llama 3.1 8B (tertiary)
│       Success for simple/SOP class ──► return response
│       Complex class (escalate) ──► skip vLLM, jump to [5]
│       VPN down / unreachable ──►
│
└─ [5] Circuit breaker OPEN
        ├─ Status-lookup class:
        │       serve from Redis cache (stale-ok mode, extend TTL to 5min)
        │       append: "Thông tin có thể chưa được cập nhật mới nhất"
        ├─ All other classes:
        │       return: "Hệ thống đang bảo trì. CS sẽ phản hồi trong 10 phút."
        │       auto-create Freshdesk ticket with full user query
        │       push to Human Review Queue (Redis Streams)
        └─ Alerts:
                PagerDuty P1 (if Claude + OpenAI both down)
                Slack #ops-alerts P2 (if only one provider down)
```

---

## 4. Real-time vs. async classification

| Request class | Mode | Reason |
|---|---|---|
| Status lookup | **Real-time** | CS needs answer within seconds; stale data causes direct SLA violation |
| Ticket reply draft | **Real-time** | CS agent is waiting to send; > 10s is unusable |
| SOP question | **Real-time** | Dispatch coordinator is mid-call or mid-decision |
| Escalation (complex) | **Real-time** with queue fallback | Best-effort real-time; if unavailable, queue + human notify |
| SOP index re-embedding | **Async (overnight)** | New runbooks can be indexed in the 02–05h maintenance window |
| Batch ticket triage | **Async (overnight)** | Low-priority historical tickets sent to on-prem vLLM 22–06h |
| RAGAS eval pipeline | **Async (weekly)** | Runs Sunday 03:00 when load is lowest |
| Freshdesk ticket digest | **Async (hourly)** | Supervisors receive hourly summary, not real-time stream |

---

## 5. Scaling mechanics

| Mechanism | Tool | Trigger | Action |
|---|---|---|---|
| Horizontal pod autoscale | GKE HPA | CPU > 65% or queue depth > 100 | Scale Orchestrator pods 3 → 10 |
| Pre-scale schedule | Kubernetes CronJob | 07:45 and 15:45 daily | Set HPA min replicas = 7 |
| Request queue | Redis Streams | Any burst > 200 req/min | Buffer requests, process FIFO with backpressure |
| Rate limiting | LiteLLM Proxy | > 200 req/min to Claude API | Hold excess in queue, return 202 Accepted |
| Load shedding | Nginx | Queue depth > 500 | Drop SOP class (503), keep status + ticket live |
| Cache hit boost | Redis TTL extension | Circuit breaker OPEN | Extend status cache TTL 60s → 5min (stale-ok) |

---

## 6. SLO targets & alert thresholds

| Metric | SLO target | Warning threshold (P2 Slack) | Critical threshold (P1 PagerDuty) |
|---|---|---|---|
| Availability | ≥ 99.5% / month | < 99.8% in last 24h | < 99.5% in last 1h |
| p95 response time | ≤ 5 s | > 3.5 s (10 min avg) | > 5 s (5 min avg) |
| Claude API error rate | ≤ 1% | > 0.5% (5 min window) | > 2% (2 min window) |
| Redis cache hit rate | ≥ 55% | < 45% (15 min avg) | < 30% (5 min avg) |
| Queue depth | ≤ 200 | > 150 | > 400 |
| Fallback trigger rate | ≤ 2% / day | > 1% | > 3% |
| PII scrub failure | 0% | Any failure | Any failure (immediate P1) |
| LLM faithfulness (RAGAS) | ≥ 0.80 | < 0.83 (weekly eval) | < 0.78 (weekly eval, block release) |

---

## 7. Evaluation tie-in (from Day 14)

Reliability is not just uptime — it includes response quality under load. The following eval gates are part of the reliability plan:

| Gate | Frequency | Tool | Action on fail |
|---|---|---|---|
| RAGAS full suite (20 cases) | Every release | Langfuse + RAGAS | Block deploy if faithfulness < 0.80 |
| RAGAS regression (targeted) | Every prompt change | Langfuse | Block merge if score drops > 0.03 |
| Online sampling (1% of production) | Continuous | Langfuse streaming eval | Alert if faithfulness drops > 0.05 vs baseline |
| LLM-as-Judge (10 sampled responses) | Weekly | Claude Opus 4.7 judge | Flag to supervisor if avg score < 3.5/5 |
| Failure log review | Weekly | PostgreSQL audit + Langfuse | Top-3 worst cases → added to golden dataset |
| Safety / adversarial suite | Monthly | Custom test suite | Alert on any jailbreak pass or PII leak |

Statistical gate: if two weekly RAGAS runs show faithfulness drop, run paired t-test (scipy `ttest_rel`). Block release if p < 0.05.
