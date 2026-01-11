# ADR-002: Real-Time vs Batch Processing Architecture

**Status**: Accepted  
**Date**: 2025  
**Context**: Data Processing Systems

---

## Context

Building data-intensive applications requires choosing between real-time streaming and batch processing — or finding the right hybrid approach.

### The Spectrum

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   BATCH                    HYBRID                 REAL-TIME  │
│     │                        │                        │      │
│     ▼                        ▼                        ▼      │
│  Minutes/Hours          Seconds/Minutes          Milliseconds│
│  High throughput        Balanced                 Low latency │
│  Simple error handling  Moderate complexity      Complex     │
│  Cheap compute          Moderate cost            Expensive   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Use Cases Analyzed

| Use Case | Latency Requirement | Volume | Winner |
|----------|-------------------|--------|--------|
| Market alerts | <1 second | Medium | Real-time |
| Daily reports | Hours | High | Batch |
| User dashboards | <5 seconds | Low | Real-time |
| AI analysis | <30 seconds | Medium | Hybrid |
| Historical queries | Flexible | Very high | Batch |

---

## Decision

**Implement a Lambda Architecture with clear boundaries between hot and cold paths.**

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LAMBDA ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                      Data Sources                            │
│                           │                                  │
│              ┌────────────┴────────────┐                    │
│              ▼                         ▼                    │
│     ┌────────────────┐       ┌────────────────┐            │
│     │   HOT PATH     │       │   COLD PATH    │            │
│     │   (Real-time)  │       │   (Batch)      │            │
│     └───────┬────────┘       └───────┬────────┘            │
│             │                        │                      │
│             ▼                        ▼                      │
│     ┌────────────────┐       ┌────────────────┐            │
│     │  Stream        │       │  Data Lake     │            │
│     │  Processor     │       │  (Raw Storage) │            │
│     └───────┬────────┘       └───────┬────────┘            │
│             │                        │                      │
│             ▼                        ▼                      │
│     ┌────────────────┐       ┌────────────────┐            │
│     │  Real-time     │       │  Batch         │            │
│     │  Cache (Redis) │       │  Analytics     │            │
│     └───────┬────────┘       └───────┬────────┘            │
│             │                        │                      │
│             └──────────┬─────────────┘                      │
│                        ▼                                    │
│               ┌────────────────┐                            │
│               │  Serving Layer │                            │
│               │  (Merged View) │                            │
│               └────────────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Path Assignment Rules

**Hot Path (Real-time)**:
- User-facing latency requirements (<5s)
- Event-driven triggers (alerts, notifications)
- Live dashboards and monitoring
- Session-bound data

**Cold Path (Batch)**:
- Historical analysis
- ML model training
- Compliance reporting
- Data aggregations (daily/weekly/monthly)

**Hybrid (Both)**:
- AI analysis (real-time first pass, batch refinement)
- Search indexes (real-time updates, batch rebuilds)
- Caching layers (real-time invalidation, batch warming)

---

## Consequences

### Positive

1. **Right tool for right job** — Latency-critical paths get streaming; heavy analytics get batch
2. **Cost optimization** — Batch processing on spot instances; streaming only where needed
3. **Scalability** — Each path scales independently
4. **Flexibility** — Easy to move workloads between paths as requirements change

### Negative

1. **Operational complexity** — Two processing paradigms to maintain
2. **Data consistency challenges** — Hot and cold views may temporarily diverge
3. **Skill requirements** — Team needs expertise in both paradigms
4. **Infrastructure overhead** — More systems to monitor

### Mitigations

- Clear documentation on path assignment
- Reconciliation jobs to align hot/cold views
- Training for team on both paradigms
- Unified monitoring dashboard

---

## Implementation Guidelines

### When to Use Real-Time

```
IF any of:
  - User waiting for response
  - Alert/notification trigger
  - Session-bound data
  - Freshness < 5 seconds required
THEN → Real-time path
```

### When to Use Batch

```
IF all of:
  - No user directly waiting
  - Data volume > 10K records
  - Acceptable staleness > 1 minute
  - Complex aggregations needed
THEN → Batch path
```

### Handling Edge Cases

**Problem**: AI analysis needs fast first response but benefits from deeper batch analysis

**Solution**: 
1. Real-time: Quick classification/summary (<3s)
2. Batch: Deep analysis queued for background processing
3. Push update when batch completes

---

## Alternatives Considered

### Alternative 1: Pure Real-Time (Kafka Everything)
- **Rejected because**: Overkill for historical queries; expensive for batch workloads

### Alternative 2: Pure Batch (Hourly ETL)
- **Rejected because**: User-facing latency requirements not met

### Alternative 3: Kappa Architecture (Single Path)
- **Considered**: Simpler, but batch workloads become expensive on streaming infrastructure

---

## References

- Nathan Marz, "Big Data: Principles and Best Practices"
- Internal latency benchmarks (P99 requirements)
- Cost analysis: Streaming vs batch compute

