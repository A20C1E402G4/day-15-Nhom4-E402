# Worksheet 5 — Skills Map & Track Phase 2
## Logistics Operations Agent · Nhóm 4 · Zone E402 · 2026-04-22

---

## 1. Skills recap — what we used in this project

Reviewing the Logistics Ops Agent work against the three competency pillars:

| Skill used | Pillar | Where in this project |
|---|---|---|
| Problem statement + use-case framing | CP1 Business | W0: scenario selection, user identification, impact analysis |
| Cost analysis + ROI framing | CP1 Business | W2: full cost anatomy, hidden-cost multiplier, tier mapping |
| Risk assessment (PDPA, SLA) | CP1 Business | W1: enterprise constraint analysis, security checklist |
| Cloud deployment design (GKE, Docker) | CP2 Infrastructure | W1: GKE Autopilot, namespaces, Helm, Terraform |
| Data pipeline (vector DB, embeddings) | CP2 Infrastructure | W1: Qdrant + text-embedding-3-small, SOP indexing |
| Monitoring + structured logging | CP2 Infrastructure | W1: ELK stack, Prometheus/Grafana, Alertmanager |
| LLM API + model routing | CP3 AI Engineering | W1: LiteLLM Proxy, Haiku/Sonnet/Opus routing |
| RAG pipeline | CP3 AI Engineering | W1: Qdrant retrieval, top-k chunk injection |
| Prompt engineering + caching | CP3 AI Engineering | W3: Anthropic Prompt Caching, compression |
| Tool calling + agent orchestration | CP3 AI Engineering | W1: LangGraph ReAct, Integration Adapter tool calls |
| Guardrails + safety | CP3 AI Engineering | W1: PII scrubber, guardrail Haiku check |
| Evaluation + benchmarking | CP3 AI Engineering | W4: RAGAS gates, LLM-as-Judge, failure log |

---

## 2. Team self-rating (1–5 per pillar)

| Member | CP1 Business/Product | CP2 Infra/Data/Ops | CP3 AI Engineering |
|---|:---:|:---:|:---:|
| Hoàng Đinh Duy Anh | 3 | 3 | 4 |
| Nguyễn Tiến Huy Hoàng | 3 | 4 | 4 |
| Trần Nhật Vĩ | 4 | 3 | 4 |
| Trần Thanh Phong | 3 | 3 | 3 |
| **Team average** | **3.25** | **3.25** | **3.75** |

> Team strengths from W0: requirement analysis and planning (CP1/CP3), RAG (CP3), tool calling (CP3). CP3 is the clear highest pillar. CP1 and CP2 are equal and moderate.

---

## 3. Track decision

**Chosen: Track 3 — AI Application**

### Decision path (from day-15 framework)

```
Q: Thích business hay technical?
→ Technical (team's top strengths are RAG + tool calling + agent orchestration)

Q: Thích infra hay app?
→ App (the work we found most engaging was building the ReAct loop,
   LiteLLM routing logic, and RAGAS evaluation — not the ELK stack setup)

→ Track 3: AI Application
```

### Why Track 3 specifically for this project

1. **The remaining high-value work is in agent reasoning, not platform.** The GKE cluster, ELK stack, and Prometheus setup from W1 are done. What's left to make the Logistics Ops Agent genuinely better is: smarter multi-step ticket reasoning, better RAG recall on edge-case SOP queries, and a robust evaluation loop. All of that is Track 3 territory.

2. **Our W3 optimisations point toward Track 3 skills.** Prompt caching, prompt compression, and the proposed fine-tuning of vLLM on company SOP data (W3 Strategy 3) require AI engineering depth — model customisation, eval-driven iteration, production eval pipelines — not platform skills.

3. **Team average CP3 = 3.75, highest pillar.** Building on the strongest foundation is lower risk than pivoting to CP2 infrastructure to shore up the weakest pillar.

---

## 4. What Track 3 covers (and how it maps to this project)

| Track 3 module | How it applies to Logistics Ops Agent |
|---|---|
| Advanced agent patterns | Improve the LangGraph ReAct loop — currently single-pass; needs multi-step order investigation (e.g., query TMS → check OMS → cross-reference driver app → synthesise) |
| Memory & long-term context | CS agents re-open tickets days later. Agent currently has no session memory across conversations. Persistent memory → faster resolution, fewer repeat questions |
| GraphRAG & knowledge graphs | Orders, routes, drivers, and customers form a graph. GraphRAG can answer "which driver has the most delayed deliveries this week?" — not possible with flat SOP embeddings |
| Fine-tuning & model customisation | W3 Strategy 3: fine-tune Llama 3.1 8B on company SOP + resolved-ticket pairs to replace Sonnet on the on-prem A100 |
| Production evaluation systems | W4 §7 eval gates need a proper pipeline: golden dataset management, automated RAGAS runs on each PR, online sampling with drift detection |

---

## 5. Skill gaps to close for this project in Phase 2

| Gap | Why it matters for Logistics Ops Agent | Priority |
|---|---|---|
| **GraphRAG / knowledge graph integration** | Order ↔ route ↔ driver ↔ customer relationships are not captured by flat vector search. Status queries that require multi-hop reasoning (e.g. "why is this order delayed? check driver history") fail today. | High |
| **Production eval pipeline (automated + online)** | W4 eval gates are designed but not yet built. Without them, we cannot safely deploy W3 optimisations (tighter Opus routing, prompt compression) because we have no automated quality check to catch regressions. | High |
| **Fine-tuning workflow (SFT + PEFT)** | W3 Strategy 3 depends on fine-tuning Llama 3.1 8B on company data. Team has no fine-tuning experience (not listed in W0 strengths). This is the key cost-reduction lever at Growth tier. | Medium |
| **Long-term agent memory** | Sessions currently stateless. For ticket follow-up and multi-day CS conversations, memory is essential. LangGraph checkpointing + PostgreSQL-backed memory store needed. | Medium |

---

## 6. First week of Phase 2 — concrete next step

**Goal:** Build and validate the semantic cache POC with a production eval gate.

**Week 1 tasks:**
1. Deploy RAGAS eval pipeline as a GitHub Actions job triggered on every PR to `main`. Golden dataset: 20 Q&A pairs sampled from the 5 request classes in W2 §2.2.
2. Add `cache_control: ephemeral` to LiteLLM Proxy config for all Sonnet/Opus calls (W3 Strategy 1). Measure `cache_read_input_tokens` in Langfuse — confirm ≥ 60% cached after 3 days.
3. Baseline the eval: run the RAGAS suite before any optimisation, record faithfulness/answer_relevancy/context_recall/context_precision as the gate thresholds for all future work.

**Success criteria:** eval pipeline running on CI, prompt caching confirmed in Langfuse, all 4 RAGAS scores ≥ 0.80, cost delta measurable.

This unblocks W3 Strategies 2 and 3 — we cannot safely roll those out without the eval gate in place.

---

## 7. Career paths mapped to team members

| Member | Pillar strengths | Suggested Phase 2 focus within Track 3 |
|---|---|---|
| Duy Anh | CP3 4 · CP1 3 · CP2 3 | Agent orchestration + eval pipeline — leads the LangGraph memory + RAGAS CI work |
| Huy Hoàng | CP3 4 · CP2 4 | Production eval + fine-tuning pipeline — best infra background to set up SFT workflow |
| Nhật Vĩ | CP3 4 · CP1 4 | GraphRAG design + business mapping — strongest business + AI combo for graph schema design |
| Thanh Phong | CP3 3 · evenly distributed | Build up in one area — recommend fine-tuning data curation (pairs well with Track 3 fine-tune module) |
