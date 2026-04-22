# Worksheet 0 — Learning Timeline & Topic Lock

**Team:** Nhóm 4 · **Zone:** E402 · **Date:** 2026-04-22

**Members:** Hoàng Đinh Duy Anh · Nguyễn Tiến Huy Hoàng · Trần Nhật Vĩ · Trần Thanh Phong

## Top 2–3 team strengths
1. Requirement analysis and planning
2. RAG
3. Tool calling 

## Topic lock

Logistics Operations Agent — an AI copilot for a Vietnamese mid-size 3PL (third-party logistics) company. It helps dispatch staff and CS agents handle three things in real time: (1) order/shipment status lookups, (2) customer ticket drafting and triage, and (3) SOP/procedure Q&A. Traffic is bursty (3–5× during rush hours), queries split cleanly into cheap repetitive lookups vs. rare complex reasoning, and every order carries PII — which is why the plan lands on a Hybrid deployment with edge PII scrubbing, semantic caching for status lookups, Haiku→Sonnet→Opus routing, and an on-prem vLLM fallback.

- **Source:** ☐ Phase 1 product   ☒ **Scenario card**
- **Card chosen:** **Scenario 5 — Logistics Operations Agent**
- **Why this card:** cost + reliability + cache + fallback tension is rich → matches rubric priorities (cost 20, reliability 15, architecture 20).

## Scenario-5 answer block

| Field | Answer |
|---|---|
| Project / agent name | Logistics operations agent |
| Primary user | Điều phối viên · CS team · Quản lý vận hành |
| Data touched | Đơn hàng, trạng thái giao vận, ticket CS, lịch giao hàng, SOP |
| Impact of wrong/slow answer | Sai status → khiếu nại; CS reply chậm → SLA miss |
| Realistic deployment context | SaaS cho mid-size 3PL ở VN, tích hợp TMS + ticketing |

