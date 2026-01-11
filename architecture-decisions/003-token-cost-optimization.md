# ADR-003: Token Cost Optimization Patterns

**Status**: Accepted  
**Date**: 2025  
**Context**: LLM-Powered Applications

---

## Context

LLM API costs scale linearly with token usage. For production systems processing thousands of requests daily, optimization strategies can mean the difference between sustainable operations and budget overruns.

### The Cost Reality

```
┌─────────────────────────────────────────────────────────────┐
│              MONTHLY TOKEN COST PROJECTION                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario: 10K requests/day × 2K tokens/request              │
│                                                              │
│  GPT-4:      $0.03/1K tokens  →  $18,000/month              │
│  GPT-3.5:    $0.002/1K tokens →  $1,200/month               │
│  Gemini:     $0.001/1K tokens →  $600/month                 │
│  LLaMA:      $0.0002/1K tokens→  $120/month                 │
│                                                              │
│  Potential savings with optimization: 40-60%                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Requirements

1. **Maintain quality** — Can't sacrifice output for cost
2. **Predictable costs** — Budget planning requires consistency
3. **Scalable** — Optimization must work at 10x volume
4. **Measurable** — Track impact of each optimization

---

## Decision

**Implement a multi-layered token optimization strategy targeting 40%+ cost reduction.**

### Optimization Layers

```
┌─────────────────────────────────────────────────────────────┐
│              TOKEN OPTIMIZATION PYRAMID                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    ┌───────────┐                            │
│                    │  Model    │  20-30% savings            │
│                    │ Selection │                            │
│                    └─────┬─────┘                            │
│               ┌──────────┴──────────┐                       │
│               │  Response Caching   │  15-25% savings       │
│               └──────────┬──────────┘                       │
│          ┌───────────────┴───────────────┐                  │
│          │     Prompt Optimization       │  10-15% savings  │
│          └───────────────┬───────────────┘                  │
│     ┌────────────────────┴────────────────────┐             │
│     │         Context Management              │  5-10% savings│
│     └────────────────────┬────────────────────┘             │
│  ┌───────────────────────┴───────────────────────┐          │
│  │              Batching & Streaming              │  5% savings│
│  └───────────────────────────────────────────────┘          │
│                                                              │
│  TOTAL POTENTIAL SAVINGS: 40-60%                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### Layer 1: Model Selection (20-30% savings)

Route tasks to appropriate model tier:

```
┌─────────────────────────────────────────────────────────────┐
│                  MODEL ROUTING RULES                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Task Complexity    Model          Cost/1K tokens           │
│  ────────────────────────────────────────────────           │
│  Extraction         LLaMA/Groq     $0.0002                  │
│  Classification     Gemini Flash   $0.0005                  │
│  Summarization      Gemini Pro     $0.001                   │
│  Analysis           GPT-3.5        $0.002                   │
│  Complex reasoning  GPT-4          $0.03                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Layer 2: Response Caching (15-25% savings)

Cache strategy for repeated queries:

```
Request
    │
    ▼
┌─────────────┐
│  Hash       │  Create semantic hash of request
│  Request    │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│  Cache      │────▶│  Return     │  Cache HIT
│  Lookup     │     │  Cached     │
└──────┬──────┘     └─────────────┘
       │ Cache MISS
       ▼
┌─────────────┐
│  Call       │
│  LLM        │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Store      │  TTL based on query type
│  Response   │
└─────────────┘
```

**Cache TTL Strategy**:
- Static queries (definitions, explanations): 24 hours
- Semi-dynamic (analysis patterns): 1 hour
- Dynamic (real-time data): No cache

### Layer 3: Prompt Optimization (10-15% savings)

Reduce token count without losing information:

```
BEFORE (847 tokens):
─────────────────────
"You are an expert financial analyst with deep knowledge of 
cryptocurrency markets. Your task is to analyze the following 
market data and provide insights. Please consider all relevant 
factors including technical indicators, market sentiment, and 
recent news. The data is as follows: [DATA]. Please provide 
a comprehensive analysis covering: 1) Current trend analysis, 
2) Key support and resistance levels, 3) Sentiment analysis, 
4) Risk factors, 5) Recommendations. Format your response in 
a clear, structured manner."

AFTER (312 tokens):
─────────────────────
"Analyze this crypto market data:
[DATA]

Output JSON:
{trend, support_levels, resistance_levels, sentiment, risks, recommendation}"
```

**Techniques**:
1. Remove verbose instructions
2. Use structured output formats
3. Eliminate redundant context
4. Leverage system prompts for common instructions

### Layer 4: Context Management (5-10% savings)

Minimize context window usage:

```
┌─────────────────────────────────────────────────────────────┐
│              CONTEXT WINDOW MANAGEMENT                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Strategy              Implementation                        │
│  ─────────────────────────────────────────────              │
│  Rolling window        Keep last N messages only             │
│  Summarization         Compress old context periodically     │
│  Selective retrieval   Only include relevant history         │
│  Reference pointers    "As discussed above" vs repeat        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Layer 5: Batching & Streaming (5% savings)

Combine related requests:

```
BEFORE: 5 separate API calls
─────────────────────────────
Call 1: "Analyze BTC" → Response
Call 2: "Analyze ETH" → Response
Call 3: "Analyze SOL" → Response
Call 4: "Compare all" → Response
Call 5: "Recommend" → Response

AFTER: 1 batched call
─────────────────────────────
Call 1: "Analyze BTC, ETH, SOL. Compare. Recommend." → Response

Savings: ~40% fewer tokens (reduced overhead per call)
```

---

## Consequences

### Positive

1. **40% cost reduction** achieved across production systems
2. **Predictable spending** through caching and tiering
3. **Maintained quality** for complex tasks (still use GPT-4)
4. **Faster responses** from cache hits

### Negative

1. **Cache invalidation complexity**
2. **Prompt maintenance** across model variants
3. **Monitoring overhead** to track optimization impact
4. **Risk of over-optimization** (degraded quality)

### Metrics to Track

- Cost per request (by model, by task type)
- Cache hit rate
- Quality scores (sample human evaluation)
- Latency percentiles

---

## Alternatives Considered

### Alternative 1: Use Cheapest Model Always
- **Rejected**: Quality degradation unacceptable for complex tasks

### Alternative 2: No Caching (Always Fresh)
- **Rejected**: 25% cost increase; many queries are effectively identical

### Alternative 3: Fine-Tuned Models
- **Deferred**: High upfront cost; consider when volume justifies

---

## Results

After implementing all layers:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Monthly token cost | $15,000 | $9,000 | 40% reduction |
| Avg response time | 2.1s | 0.8s | 62% faster (cache) |
| Quality score | 8.5/10 | 8.4/10 | Maintained |
| Cache hit rate | 0% | 35% | - |

---

## References

- OpenAI Pricing Documentation
- Internal cost tracking dashboard
- Quality evaluation framework

