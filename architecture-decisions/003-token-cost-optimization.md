# ADR-003: Token Cost Optimization Patterns

**Status**: Accepted  
**Date**: 2025–2026  
**Context**: Pattern applied across AI Marketing Platform ($180-225/month total operating cost), AI Legal Document Analyzer (~60% cost reduction), and AI Trader Psychology Platform (dual-model routing)

---

## Context

LLM API costs scale linearly with token usage. Across three production platforms, I implemented different optimization strategies — each driven by the specific cost bottleneck of that system.

### The Real Cost Problems

| Platform | Original Bottleneck | Root Cause | Monthly Cost Before |
|----------|-------------------|------------|-------------------|
| AI Marketing Platform | Image matching: $2-4 per blog post | 552 images × 5-8 markers = 2,700-4,400 LLM calls per content piece | Unsustainable at scale |
| AI Legal Document Analyzer | Repeat clause analysis | Same clause types appear across hundreds of LPAs — identical analysis generated each time | ~60% wasted on duplicate work |
| AI Trader Psychology Platform | Consultation token volume | 50+ exchange sessions with full conversation history in context | ~$400/month with single model |

---

## Decision

**Apply targeted optimization at the specific cost bottleneck of each system, rather than generic optimization across the board.**

### Optimization 1: Pre-Computed Tags (Marketing Platform)

The original image matching design called the LLM for every image-to-content comparison:

```
BEFORE: Per-Request LLM Matching
─────────────────────────────────
Blog post with 6 [Insert Image] markers
  × 552 artwork images to compare
  = 3,312 LLM API calls per blog post
  
Cost: $2–4 per blog post
Latency: 12–16 seconds per marker
Total: Unsustainable

AFTER: Pre-Computed Tag Matching
─────────────────────────────────
One-time: Each image analyzed at upload (Gemini Flash)
  → Themes, colours, mood, style, subjects stored as tags
  → Vector embedding stored in pgvector

At query time:
  Fast path: keyword extraction → match against stored tags
  → Zero API cost, sub-1-second
  
  Slow path (only if tags insufficient):
  → Generate embedding → pgvector cosine similarity
  
Cost: $0 per blog post (fast path)
Latency: < 1 second
Savings: ~100% on the image matching bottleneck
```

The architectural insight: **move AI computation from query time to ingest time** whenever the input data (images) changes less frequently than the queries (blog posts).

### Optimization 2: Semantic Caching (Legal Document Analyzer)

Law firms review hundreds of LPAs. The same clause types (Key Person, GP Removal, Waterfall) appear in every agreement with similar language:

```
BEFORE: Every Analysis Fresh
─────────────────────────────────
Lawyer clicks "Analyze" on GP Removal clause
  → Call Dify workflow → RAG retrieval → GPT-4o generation
  → 30-60 seconds, full API cost

Same clause type analyzed yesterday in different LPA
  → Identical API call, identical cost

AFTER: SHA-256 Semantic Cache
─────────────────────────────────
Lawyer clicks "Analyze"
  → SHA-256 hash of clause text
  → Cache lookup (7-day TTL)
  
Cache HIT:  Return analysis instantly, $0
Cache MISS: Full pipeline → cache result for 7 days

Result: ~60% of clause analyses served from cache
  → 60% cost reduction on repeat queries across the firm
```

The 7-day TTL balances freshness with cost savings. Identical clause text across different LPAs returns the same analysis instantly. Different clause text (even for the same clause type) gets fresh analysis.

### Optimization 3: Task-Specific Model Routing (Trader Psychology Platform)

Two AI workloads with very different requirements sharing one platform:

```
┌─────────────────────────────────────────────────────────────┐
│              DUAL-MODEL COST OPTIMIZATION                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Quiz Analysis (every completion):                           │
│    Input: ~20 answers (small payload)                        │
│    Output: Structured JSON (insights, traits)                │
│    Requirement: < 5 seconds, Zod schema compliance           │
│    → GPT-4o-mini: $0.15/1M input tokens                     │
│                                                              │
│  Consultation Engine (lower volume, higher depth):           │
│    Input: 50+ Q&A exchanges (large context)                  │
│    Output: Adaptive question selection + plan generation     │
│    Requirement: Long-context coherence                        │
│    → Gemini 2.5 Flash: better long-context at lower cost     │
│                                                              │
│  Fallback (both paths):                                      │
│    Quiz: Deterministic centroid scoring (no API call)         │
│    Consultation: Rule-based selection from 100+ question bank │
│    → $0 when LLM providers are unavailable                   │
│                                                              │
│  Combined: fraction of single-model cost                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Optimization 4: Hybrid RAG Over Fine-Tuning (Marketing Platform)

The Marketing Platform needed to replicate a client's writing voice. Fine-tuning would have been the obvious approach:

```
FINE-TUNING:
  Cost: $500+ per training run
  Retraining: Every time voice evolves (monthly?)
  Lock-in: Tied to one model provider
  Annual cost: $3,000–6,000 in training alone

HYBRID RAG (Selected):
  Cost: One-time embedding of 80+ writing samples
  Adaptation: New samples indexed immediately, $0 retrain
  Flexibility: Works with any LLM (Haiku, Sonnet, GPT-4)
  Annual cost: ~$20 in embedding API calls

  Implementation: 70% vector similarity + 30% full-text keyword
  (Reciprocal Rank Fusion combining pgvector + PostgreSQL FTS)
```

---

## The Cost Optimization Framework

After building these systems, the pattern is:

```
IDENTIFY THE BOTTLENECK FIRST

├── Repetitive queries on similar data?
│   └── Semantic caching (hash-based TTL)
│       Example: LPA clause analysis, FAQ responses
│
├── AI computation that could happen at ingest instead of query?
│   └── Pre-compute and store results
│       Example: Image tagging, document classification
│
├── Different tasks with different complexity?
│   └── Route to appropriate model tier
│       Example: Haiku for drafts, Sonnet for final copy
│
└── Training/adaptation costs dominating?
    └── RAG over fine-tuning (if context injection works)
        Example: Voice replication, domain knowledge
```

---

## Consequences

### Positive

1. **Marketing Platform: $180–225/month** total operating cost for 5 AI tools, voice RAG, lead scoring, email intelligence, image matching, and content generation
2. **Legal Analyzer: 60% cost reduction** on repeat clause analyses without any quality degradation
3. **Trader Psychology: significant reduction** vs single-model approach — GPT-4o-mini at $0.15/1M input tokens replaces GPT-4 at $30/1M for quiz workload, with full rule-based fallback paths eliminating API costs entirely during outages
4. **Quality maintained or improved** — Pre-computed image tags are actually more consistent than per-request LLM calls (no variance between runs)

### Negative

1. **Cache staleness risk** — 7-day TTL on legal analysis means a model update won't affect cached results immediately
2. **Pre-computation storage cost** — 552 images × (tags + embeddings) stored in PostgreSQL/pgvector
3. **Complexity of multiple optimization layers** — Each platform uses 2-3 strategies simultaneously

### Trade-offs Accepted

- Accepted 7-day cache staleness for 60% cost savings — legal clause language doesn't change between analyses
- Accepted one-time image processing cost (Gemini Flash per image) for zero ongoing cost per match
- Accepted prompt maintenance overhead across model variants for task-specific routing savings

---

## Results

| Platform | Optimization | Before | After | Savings |
|----------|-------------|--------|-------|---------|
| AI Marketing Platform | Pre-computed image tags | $2–4/blog, 16s/marker | $0/blog, <1s/marker | ~100% on image matching |
| AI Marketing Platform | Model routing (Haiku/Sonnet/Flash) | ~$800/month (all Sonnet) | $180–225/month | ~75% |
| AI Legal Document Analyzer | SHA-256 semantic caching | Full API cost per analysis | 60% served from cache | 60% |
| AI Trader Psychology | Dual-model + rule fallback | ~$400/month (all GPT-4) | GPT-4o-mini ($0.15/1M) + Gemini Flash | 200x cheaper per token on quiz workload |
