# ADR-001: Multi-Model LLM Selection Strategy

**Status**: Accepted  
**Date**: 2025–2026  
**Context**: Pattern applied across 5 production platforms

---

## Context

Every production AI system I've built has faced the same question: single LLM provider or multi-model orchestration? After shipping 5 platforms with different answers, a clear pattern emerged.

### The Real Decision Across Projects

| Project | AI Workloads | Single Model Would Cost | Multi-Model Actual Cost |
|---------|-------------|------------------------|------------------------|
| AI Marketing Platform | Voice RAG, email drafting, image vision analysis, lead scoring | ~$800/month (all Sonnet) | $180–225/month |
| AI Trader Psychology Platform | Quiz analysis, 50+ exchange consultations, plan generation | ~$400/month (all GPT-4) | Fraction of that (GPT-4o-mini + Gemini Flash) |
| AI Legal Document Analyzer | Clause extraction, 4-part legal analysis, cross-document RAG | ~$500/month (all GPT-4o) | ~$150–300/month |
| DocsFlow | Document chat, hybrid retrieval, summarization | Variable | 3-tier cascade with failover |

The pattern is consistent: **multi-model routing reduces costs 50-75% while maintaining or improving quality**, because most workloads don't need frontier models.

---

## Decision

**Implement task-specific model routing with fallback chains, applied differently per project based on workload characteristics.**

### How This Looks in Practice

```
┌─────────────────────────────────────────────────────────────────┐
│              ACTUAL MODEL ROUTING (PRODUCTION)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AI Marketing Platform (via OpenRouter):                         │
│    Claude Haiku    → Content drafts, email classification        │
│    Claude Sonnet   → High-stakes outreach, voice-critical copy   │
│    Gemini Flash    → Image vision analysis (552 artwork images)  │
│                                                                  │
│  AI Trader Psychology Platform:                                  │
│    GPT-4o-mini     → Quiz analysis (fast, $0.15/1M, Zod output) │
│    Gemini 2.5 Flash → Consultation (long-context, 50+ exchanges) │
│    Rule-based      → Fallback (deterministic scoring, no API)    │
│                                                                  │
│  AI Legal Document Analyzer:                                     │
│    GPT-4o          → Clause analysis (high-stakes legal output)  │
│    OpenAI ada-002  → Embeddings (1536-dim, pgvector)             │
│    Dify routing    → Workflow-level model selection               │
│                                                                  │
│  DocsFlow:                                                       │
│    Tier 1 (Primary)   → Claude / GPT-4o                         │
│    Tier 2 (Fallback)  → Gemini Pro                               │
│    Tier 3 (Emergency) → Lightweight model                        │
│    Circuit breaker triggers cascade on provider failure           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Selection Criteria That Actually Matter

Through building these systems, three factors consistently drove model choice:

1. **Output structure requirements** — If the output needs to conform to a Zod schema (Trading Advisor quiz), GPT-4o-mini excels at structured JSON. If the output is creative prose that must match a specific voice (Marketing Platform), Claude Sonnet is measurably better.

2. **Context window usage** — The Trading Advisor consultation engine maintains 50+ exchange sessions. Gemini 2.5 Flash handles this at lower cost per token than GPT-4. The Marketing Platform's voice RAG injects top-K writing samples into every prompt — model selection must account for the injected context overhead.

3. **Failure consequences** — Quiz personality classification in the Trading Advisor uses deterministic centroid scoring (no LLM), with GPT-4o-mini only adding supplementary insights. If OpenAI is down, the product still works. The LPA Analyzer can't afford hallucinated legal analysis — it uses GPT-4o exclusively for clause analysis, no cheaper fallback.

---

## Implementation Patterns

### Pattern 1: Gateway Routing (Marketing Platform)

OpenRouter as a single billing surface with instant model switching:

```
Request → OpenRouter Gateway → Route by task type
                                  ├── Haiku  (drafts, classification)
                                  ├── Sonnet (voice-critical content)
                                  └── Gemini Flash (vision analysis)
```

Single API key, single billing dashboard, zero vendor lock-in. Switch models by changing a string, not refactoring integrations.

### Pattern 2: Dual-Model with Graceful Degradation (Trading Advisor)

```
Quiz Funnel:
  Primary: GPT-4o-mini (structured output via Zod)
  Fallback: Deterministic scoring only (no AI insights)
  → Product works without any LLM

Consultation Engine:
  Primary: Gemini 2.5 Flash (adaptive questioning)
  Fallback: Rule-based question selection from 100+ question bank
  → Session continues with pre-authored questions
```

The key insight: both AI paths have **non-AI fallbacks**. The product degrades gracefully, never breaks.

### Pattern 3: Circuit Breaker Cascade (DocsFlow)

```
Request → Tier 1 (Primary) ──── Success → Return
              │
          Failure (timeout/error/rate limit)
              │
              ▼
          Tier 2 (Fallback) ──── Success → Return
              │
          Failure
              │
              ▼
          Tier 3 (Emergency) ─── Success → Return
              │
          Failure
              │
              ▼
          Error response with retry guidance
```

Circuit breaker tracks failure rates per provider. After N consecutive failures, the primary is marked unhealthy and bypassed for a cooldown period.

---

## Consequences

### Positive

1. **50-75% cost reduction** across all platforms compared to single-model approach
2. **Zero downtime from provider outages** — Every system has at least one fallback path
3. **Task-appropriate quality** — Frontier models only where output stakes justify the cost
4. **Operational flexibility** — New models (GPT-4.1, Claude 4) can be tested per-task without system-wide migration

### Negative

1. **Prompt tuning per model** — Claude and GPT respond differently to the same prompt; voice RAG prompts needed model-specific variants
2. **Testing matrix** — Each model combination needs validation; the Marketing Platform has 3 models × 5 tools = 15 paths to verify
3. **Monitoring complexity** — Must track cost, latency, and quality per model per task type

### Trade-offs Accepted

- Accepted slightly higher implementation complexity for significantly lower operating costs
- Accepted prompt maintenance overhead (model-specific variants) for quality optimization
- In the Trading Advisor, accepted that AI insights are supplementary — deterministic scoring is the source of truth, not LLM output

---

## Results

| Platform | Monthly Operating Cost | Fallback Coverage | Models Used |
|----------|----------------------|-------------------|-------------|
| AI Marketing Platform | $180–225 (all APIs incl. AI) | OpenRouter auto-failover | 3 (Haiku, Sonnet, Gemini Flash) |
| AI Trader Psychology Platform | Low (GPT-4o-mini + Gemini Flash) | Full rule-based fallback | 2 + rule-based |
| AI Legal Document Analyzer | ~$150–300 | Dify workflow routing | 2 (GPT-4o, ada-002) |
| DocsFlow | Variable | 3-tier circuit breaker | 3+ (configurable) |

---

## Alternatives Considered

### Alternative 1: Single Model (GPT-4 for Everything)
- **Rejected**: 3-4x higher costs. Marketing Platform would be $800+/month instead of $180-225. Overkill for classification and drafts.

### Alternative 2: User-Selected Models
- **Rejected**: Users don't know model characteristics. In the Trading Advisor, asking "which AI model for your quiz?" would be absurd UX.

### Alternative 3: Fine-Tuned Single Model
- **Deferred**: Fine-tuning creates a static model that degrades as requirements evolve. The Marketing Platform's hybrid RAG approach (inject writing samples at inference) adapts instantly when the client's voice changes — no retraining cost.
