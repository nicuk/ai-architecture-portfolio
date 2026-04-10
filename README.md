<div align="center">

# 🏗️ AI Architecture Portfolio

### Production-grade AI systems that deliver measurable business outcomes

[![MIT License](https://img.shields.io/badge/License-MIT-green.svg)](https://choosealicense.com/licenses/mit/)
[![Portfolio](https://img.shields.io/badge/🌐_nicchin.com-Portfolio_&_AI_Assistant-00D9FF?style=flat-square)](https://nicchin.com)

---

**Nic Chin** — Lead AI Architect & Fractional CTO
*End-to-end delivery: architecture → development → production*

</div>

---

## 🎯 Impact at a Glance

<table>
<tr>
<td align="center">
<img src="https://img.shields.io/badge/50--75%25-AI_cost_reduction-blue?style=for-the-badge" alt="50-75% AI cost reduction"/>
<br/>
<sub><b>Across 5 Production Platforms</b></sub>
</td>
<td align="center">
<img src="https://img.shields.io/badge/82%25-faster_page_load-green?style=for-the-badge" alt="82% faster page load"/>
<br/>
<sub><b>15.6s → 2.8s Mobile LCP</b></sub>
</td>
<td align="center">
<img src="https://img.shields.io/badge/15--20_hrs/wk-automated-purple?style=for-the-badge" alt="15-20 hrs/week automated"/>
<br/>
<sub><b>AI Marketing Automation</b></sub>
</td>
<td align="center">
<img src="https://img.shields.io/badge/24/24-enterprise_criteria-orange?style=for-the-badge" alt="24/24 enterprise criteria"/>
<br/>
<sub><b>Pharma Pilot Approved</b></sub>
</td>
</tr>
</table>

---

## 🔥 Flagship Project

**[DocsFlow — Multi-Tenant Document Intelligence SaaS](https://github.com/nicuk/docsflow)** · [Live Demo](https://docsflow.app)

Full-stack RAG platform with open source code and 1,188 commits. Key architectural decisions:
- **Hybrid retrieval**: Dense vectors + BM25 sparse vectors with Reciprocal Rank Fusion
- **Hierarchical 2-stage search**: Document-level ranking → chunk-level retrieval for scale
- **4-layer tenant isolation**: RLS + Pinecone namespaces + Clerk auth + subdomain routing
- **Circuit breaker failover**: 3-tier LLM cascade with emergency fallback

→ [View source code & full README](https://github.com/nicuk/docsflow)

---

## 📂 Featured Case Studies

Each includes architecture diagrams, technical decisions with rationale, and measurable results.

| Project | Domain | Key Outcome | Tech Stack |
|:--------|:-------|:------------|:-----------|
| 🔍 [**AI Codebase Audit SaaS**](./case-studies/ai-codebase-audit-saas.md) | DevTools / Due Diligence | 3-min audits replacing 2-week cycles — [Live](https://systemaudit.dev) | `Next.js 16` `Claude 4.6` `Stripe` |
| 🤖 [**AI Marketing Automation Platform**](./case-studies/multi-agent-marketing-platform.md) | Creative Industry / AI | 15-20 hrs/week recovered, voice fidelity validated by client's network — 18-week sole delivery | `Next.js 14` `Supabase` `pgvector` `OpenRouter` |
| ⚖️ [**AI Legal Document Analyzer**](./case-studies/ai-legal-document-analyzer.md) | Legal Tech / Investment Funds | 150-200pg LPAs processed in minutes, Word Add-in, 60% API cost reduction via caching | `Next.js 15` `GPT-4o` `pgvector` `Office.js` |
| 🏗️ [**IoT Utility SaaS Platform**](./case-studies/iot-utility-saas-platform.md) | PropTech / IoT | 15-person team led, 82% mobile LCP improvement, 1,746 commits shipped | `Next.js 16` `Supabase` `MQTT` `Docker` |
| 🔒 [**Production Readiness — Pharma SaaS**](./case-studies/production-readiness-pharma-saas.md) | Pharma / Enterprise | 24/24 acceptance criteria passed, enterprise pilot approved in 7 weeks | `Django` `Kubernetes` `AWS` `CloudWatch` |
| 🧠 [**AI Trader Psychology Platform**](./case-studies/trader-psychology-platform.md) | FinTech / EdTech | Dual-app platform: hybrid deterministic + AI scoring, automated revenue funnel with CRM | `Next.js 14` `GPT-4o-mini` `Gemini Flash` `Supabase` |

## 📁 Additional Case Studies

| Project | Domain | Key Outcome | Tech Stack |
|:--------|:-------|:------------|:-----------|
| 🎮 [**Multi-Agent Game Platform**](./case-studies/multi-agent-game-platform.md) | Game Dev / AI | 70% time reduction, $350K seed raised | `GPT-4` `Unity` `Multi-Agent` |
| 📊 [**Market Intelligence Agent**](./case-studies/market-intelligence-agent.md) | Crypto / Trading | 336% engagement increase, 40% cost savings | `FastAPI` `Gemini` `Redis` |
| 🤖 [**Social Automation System**](./case-studies/social-automation-extension.md) | Marketing / Growth | 3000x engagement improvement | `Chrome Extension` `TypeScript` |
| 🔐 [**User Verification Ecosystem**](./case-studies/user-verification-ecosystem.md) | Web3 / Auth | 15K+ users served | `Supabase` `PostgreSQL` |
| 📈 [**Trading Analytics Dashboard**](./case-studies/trading-analytics-dashboard.md) | FinTech / Crypto | 45% portfolio improvement | `TypeScript` `Multi-Model LLM` |

---

## 🧠 Architecture Decision Records

Real decisions from real projects — not theoretical patterns:

| ADR | Topic | Key Insight |
|:----|:------|:------------|
| 📘 [**ADR-001**](./architecture-decisions/001-multi-model-llm-selection.md) | Multi-Model LLM Selection | 50-75% cost reduction across 5 production platforms via task-specific routing |
| 📗 [**ADR-002**](./architecture-decisions/002-realtime-vs-batch-processing.md) | Real-Time vs Batch Processing | Hybrid patterns from IoT, Marketing, and Legal platforms — route by who's waiting |
| 📙 [**ADR-003**](./architecture-decisions/003-token-cost-optimization.md) | Token Cost Optimization | Pre-computed tags, semantic caching, model routing — real numbers from production |
| 📕 [**ADR-004**](./architecture-decisions/004-immutable-audit-architecture.md) | Immutable Audit Architecture | CloudWatch append-only logs for regulated industries — $0 marginal cost |
| 📒 [**ADR-005**](./architecture-decisions/005-iot-ingestion-pipeline-design.md) | IoT Ingestion Pipeline Design | Protocol-agnostic ingestion — fixed a 480→44 phantom device bug |

---

## 🛠️ Technical Expertise

<details>
<summary><b>🤖 AI/ML Engineering</b></summary>

- **Multi-Model Orchestration**: Production routing across GPT-4o, Gemini Flash, Claude Sonnet/Haiku, with circuit breaker failover — 50-75% cost reduction vs single-model
- **RAG Systems**: Hybrid retrieval (dense vectors + BM25 + Reciprocal Rank Fusion), pgvector, voice replication validated by professional networks
- **Cost Engineering**: Pre-computed analysis, semantic caching (SHA-256 TTL), task-specific model routing — $180-225/month total for 5-tool AI platform
- **Regulated AI**: Zero-retention processing, immutable audit trails, AI guardrails (input/output), prompt injection detection

</details>

<details>
<summary><b>⚙️ Full-Stack Systems</b></summary>

- **Frontend**: Next.js 14-16 (App Router), React 19, TypeScript, Tailwind CSS, Radix UI, Office.js (Word Add-in)
- **Backend**: Supabase (PostgreSQL + Edge Functions + Auth + RLS), Drizzle ORM, Django, FastAPI
- **IoT**: MQTT broker, wMBus protocol parsing, Docker microservices, real-time telemetry ingestion
- **Integrations**: Gmail OAuth + Pub/Sub, Apollo.io, Firecrawl, Stripe, GoHighLevel CRM, WordPress REST API

</details>

<details>
<summary><b>🚀 Delivery & Infrastructure</b></summary>

- **Team Leadership**: Led 15-person engineering team, merged 100+ PRs, managed staging→main release pipeline
- **Deployment**: Vercel, AWS (EKS, RDS, CloudWatch), serverless, edge functions, Docker
- **Security**: Row-Level Security on every table, same-day CVE patching, 5-layer spam protection, GDPR compliance
- **Methodology**: Milestone-based delivery with client feedback loops — 6 milestones, 18 weeks, 100% acceptance rate

</details>

---

## 💼 Work With Me

Currently taking select AI architecture and consulting engagements.

<table>
<tr>
<td>✅</td>
<td><b>12 production AI systems delivered and running</b></td>
</tr>
<tr>
<td>✅</td>
<td><b>End-to-end ownership: architecture → development → production</b></td>
</tr>
<tr>
<td>✅</td>
<td><b>Microsoft, Google & IBM AI Certified</b></td>
</tr>
</table>

### 📬 Get in Touch

<div align="center">

[![Portfolio](https://img.shields.io/badge/Portfolio_&_AI_Assistant-nicchin.com-00D9FF?style=for-the-badge&logo=About.me&logoColor=white)](https://nicchin.com)
&nbsp;
[![Email](https://img.shields.io/badge/Email-nic.chin%40bitto.tech-D14836?style=for-the-badge&logo=gmail)](mailto:nic.chin@bitto.tech)

</div>

---

<div align="center">

*This portfolio showcases architecture patterns and outcomes from production systems.*
*Client-specific details have been generalized to protect confidentiality.*

</div>
