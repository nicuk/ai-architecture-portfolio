# ADR-002: Real-Time vs Batch Processing Architecture

**Status**: Accepted  
**Date**: 2025–2026  
**Context**: Pattern applied across IoT SaaS Platform, AI Marketing Platform, and AI Legal Document Analyzer

---

## Context

Three production systems each faced the same decision — real-time processing, batch processing, or hybrid — with different answers driven by their specific data characteristics.

### The Actual Trade-offs (From Production)

| System | Real-Time Data | Batch Data | Why Both |
|--------|---------------|-----------|----------|
| IoT SaaS Platform | MQTT telegrams from physical meters | CSV file uploads from property managers | Same meter data arrives via both channels — must normalize and deduplicate |
| AI Marketing Platform | Gmail Pub/Sub webhooks (instant email detection) | Vercel cron jobs (lead discovery, email polling fallback) | Real-time for user-facing responsiveness, batch for background enrichment |
| AI Legal Document Analyzer | Analysis cache lookups (sub-second) | QStash parallel chunk processing (minutes) | Users need instant cache hits, but document processing is inherently async |

---

## Decision

**Use a hybrid approach where the split between real-time and batch is determined by who is waiting — a user or a background process.**

### Pattern 1: IoT SaaS — Dual-Channel Ingestion Into Shared Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│              IoT INGESTION (PRODUCTION)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REAL-TIME CHANNEL              BATCH CHANNEL                │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │  MQTT Telegrams   │          │  CSV File Upload  │         │
│  │  (IoT Gateways)   │          │  (Property Mgrs)  │         │
│  │                    │          │                    │         │
│  │  Gateway → Broker  │          │  Upload → Detect   │         │
│  │  → CBOR Decode     │          │  Layout → Parse    │         │
│  │  → Protocol Parse  │          │  → Decimal Handle  │         │
│  └────────┬───────────┘          └────────┬───────────┘         │
│           │                               │                  │
│           └───────────┬───────────────────┘                  │
│                       ▼                                      │
│            ┌──────────────────┐                              │
│            │  Normalize       │  Common schema               │
│            │  Deduplicate     │  date + device_id signature  │
│            │  Classify        │  5 device types              │
│            │  Persist         │  PostgreSQL                  │
│            └──────────────────┘                              │
│                                                              │
│  RESULT: Same data from MQTT and CSV converges into one      │
│  pipeline. 480 phantom devices → 44 real devices (fixed).    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

The MQTT channel runs as a Dockerized microservice (independent of the Next.js web app). The CSV channel runs as a Supabase Edge Function. Both feed into the same normalization and deduplication layer. The key architectural insight: **deduplication must happen at ingestion, not query time** — otherwise every downstream dashboard, billing calculation, and anomaly alert is wrong.

### Pattern 2: Marketing Platform — Real-Time Events with Batch Fallback

```
┌─────────────────────────────────────────────────────────────┐
│              EMAIL INTELLIGENCE (PRODUCTION)                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY (Real-Time):                                        │
│  Gmail OAuth → Google Pub/Sub → Webhook → Process instantly  │
│  Latency: seconds                                            │
│                                                              │
│  FALLBACK (Batch):                                           │
│  Vercel cron (every 2 min) → Gmail API poll → Process batch  │
│  Catches: Pub/Sub gaps, missed webhooks, cold starts         │
│                                                              │
│  BACKGROUND (Scheduled Batch):                               │
│  Weekly lead discovery cron → Apollo.io + Firecrawl           │
│  Monday market scanning → Refresh pipeline                   │
│                                                              │
│  DEDUPLICATION:                                              │
│  processed_emails table prevents double-processing           │
│  when both real-time and fallback detect the same email      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Real-time Pub/Sub handles the happy path. Batch cron handles the failure path. Both write to the same tables with deduplication. The user never knows which path processed their email — the experience is identical.

### Pattern 3: Legal Analyzer — Async Batch Processing with Real-Time Cache

```
┌─────────────────────────────────────────────────────────────┐
│              DOCUMENT PROCESSING (PRODUCTION)                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REAL-TIME PATH (Cache Hit):                                 │
│  User clicks "Analyze" → SHA-256 hash lookup → Cache hit     │
│  → Return cached analysis in milliseconds                    │
│                                                              │
│  BATCH PATH (Cache Miss):                                    │
│  Upload 200-page LPA → TOC parse → Smart chunk (~50 chunks)  │
│  → Queue 50 parallel QStash jobs → Process asynchronously    │
│  → Atomic progress counter → Real-time progress bar          │
│  → Results cached with 7-day TTL                             │
│                                                              │
│  WHY NOT ALL REAL-TIME:                                      │
│  50 chunks × 30-60s per Dify workflow = 25-50 min sequential │
│  Parallel via QStash: minutes (100+ simultaneous jobs)       │
│  But still async — no user should wait minutes staring at    │
│  a spinner. Progress bar + notification on completion.       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## The Decision Framework

After building these three systems, the routing decision comes down to:

```
WHO IS WAITING?

├── A user staring at a screen
│   └── Real-time (or instant cache hit)
│       Examples: Dashboard charts, email classification, cache lookups
│
├── A user who started an action and can wait
│   └── Async with progress feedback
│       Examples: Document processing, lead discovery, report generation
│
└── Nobody (background operation)
    └── Batch (cron / scheduled)
        Examples: Weekly lead scans, email polling fallback, data enrichment
```

---

## Consequences

### Positive

1. **Graceful degradation** — In the Marketing Platform, if Pub/Sub fails, the 2-minute cron catches it. Users never notice.
2. **Cost efficiency** — Batch processing (cron, scheduled jobs) runs on serverless compute that scales to zero when idle.
3. **Deduplication by design** — Both IoT and Marketing platforms solve the "same data, two paths" problem at the ingestion layer, not downstream.
4. **User experience** — Real-time for things users are waiting for, async with progress for things that take time.

### Negative

1. **Deduplication complexity** — Every hybrid system needs a deduplication strategy. The IoT platform uses `date + device_id` signatures. The Marketing platform uses `processed_emails` table.
2. **Two codepaths to maintain** — Each channel (real-time + batch) has its own parser, error handling, and monitoring.
3. **Testing requires both paths** — Must verify that real-time and batch produce identical results for the same input.

### Trade-offs Accepted

- Accepted operational complexity of Docker + Edge Functions in the IoT platform for independent scaling
- Accepted 2-minute email detection lag on the batch fallback path — acceptable for the Marketing Platform's use case
- Accepted that document processing in the Legal Analyzer is not instant — progress bar + caching makes the second analysis of the same clause instantaneous

---

## Results

| System | Real-Time Latency | Batch Processing | Key Metric |
|--------|------------------|------------------|------------|
| IoT SaaS Platform | MQTT: sub-second ingestion | CSV: seconds per upload | 480 phantom devices eliminated via deduplication |
| AI Marketing Platform | Pub/Sub: seconds | Cron: 2-min polling | 5%+ email response rate on cold outreach |
| AI Legal Document Analyzer | Cache hit: milliseconds | QStash: minutes for 200-page LPA | 60% cost reduction via 7-day analysis caching |

---

## Alternatives Considered

### Alternative 1: Pure Real-Time (Kafka/Kinesis for Everything)
- **Rejected**: Over-engineered for CSV uploads and weekly lead scans. Significant minimum monthly cost. The IoT platform chose Docker + Mosquitto at ~$0 over AWS IoT Core.

### Alternative 2: Pure Batch (Hourly ETL)
- **Rejected**: IoT dashboards need live data. Email response times measured in seconds, not hours.

### Alternative 3: Single Processing Path
- **Rejected**: MQTT and CSV have fundamentally different data shapes. Gmail Pub/Sub and cron polling have different reliability profiles. Forcing them through one path creates unnecessary complexity.
