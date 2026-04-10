# IoT Utility SaaS Platform

> 15-person team led, 1,746 commits shipped, 82% page load improvement on production B2B SaaS

**Role**: Tech Lead / Lead Engineer
**Domain**: PropTech / IoT / Utility Management
**Duration**: 12+ months (ongoing)
**Outcome**: Production B2B SaaS platform with live building deployments

---

## The Problem

Property managers face a painful regulatory and operational burden:

- **Manual meter reading** — Heat cost allocators, water meters, electricity meters across multiple buildings, read by hand or via fragmented vendor tools
- **Complex billing regulations** — Legally compliant heating cost allocation with consumption-based splits, CO2 cost calculations, and cost category breakdowns
- **No real-time visibility** — Building managers couldn't see consumption data until annual billing cycles
- **Fragmented IoT landscape** — Multiple meter manufacturers with different data formats and protocols
- **Tenant communication gaps** — No self-service portal for tenants to view their own consumption data

---

## My Role

As **Tech Lead**, I was responsible for architecture decisions, code quality, security, CI/CD, team coordination, and hands-on development across the entire stack.

| Metric | Value |
|--------|-------|
| My commits | 337 (209 feature + 128 merge) |
| PRs merged (by me) | 100+ |
| Total team commits | 1,746 across 15 contributors |
| Breadth | Architecture, IoT, security, DevOps, frontend, DB |

---

## My Solution

Built a full-stack B2B SaaS platform that ingests real-time IoT meter data, processes telemetry across manufacturers, and generates regulation-compliant heating bills — with tenant portals and proactive anomaly detection.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYSTEM ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Physical Layer (5 device types)                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Heat    │  │  Water   │  │  Heat    │  │  Smoke   │           │
│  │  Alloc.  │  │  Meters  │  │  Meters  │  │  Detect. │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┴──────────────┘                │
│                          │                                           │
│                          ▼                                           │
│  ┌──────────────────────────────────────┐                           │
│  │     IoT Gateway (wireless meters)    │                           │
│  └──────────────┬───────────────────────┘                           │
│                 │                                                    │
│            ┌────┴────┐                                              │
│            ▼         ▼                                              │
│      ┌──────────┐ ┌──────────┐                                     │
│      │  MQTT    │ │  CSV     │                                     │
│      │  Broker  │ │  Upload  │                                     │
│      └────┬─────┘ └────┬─────┘                                     │
│           │             │                                           │
│           ▼             ▼                                           │
│  ┌──────────────────────────────────────┐                           │
│  │     Ingestion Layer (Docker)         │                           │
│  │  Decode → Parse → Dedupe            │                           │
│  │  → Classify → Normalize → Persist   │                           │
│  └──────────────┬───────────────────────┘                           │
│                 │                                                    │
│                 ▼                                                    │
│  ┌──────────────────────────────────────┐                           │
│  │     PostgreSQL + RLS                 │                           │
│  │  20+ tables, 4-role isolation        │                           │
│  │  Drizzle ORM, 9 SQL migrations      │                           │
│  └──────────────┬───────────────────────┘                           │
│                 │                                                    │
│       ┌─────────┼─────────┐                                        │
│       ▼         ▼         ▼                                        │
│  ┌────────┐ ┌────────┐ ┌────────┐                                  │
│  │ Admin  │ │ Tenant │ │ Shared │                                  │
│  │ Portal │ │ Portal │ │ Dashb. │                                  │
│  └────────┘ └────────┘ └────────┘                                  │
│                                                                      │
│  Next.js 16 · 119 routes · 368+ components · 62 API endpoints      │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Technical Decisions

1. **Dockerized MQTT Microservice Separation**
   - IoT ingestion runs independently of the Next.js web app
   - MQTT broker + ingestion service + mock gateway for testing
   - Independent scaling of IoT pipeline vs. web layer

2. **Signature-Based Deduplication**
   - Each reading keyed by `date + device_id`
   - Same data arriving via MQTT and CSV is deduplicated at insert time
   - Prevented a phantom device counting bug (480 phantom devices → 44 real)

3. **Consumption Deltas at Query Time, Not Storage**
   - Raw cumulative readings stored as meters report them
   - Dashboard calculates deltas between consecutive readings
   - Preserves data fidelity — if a reading is missed, subsequent deltas self-correct

4. **4-Role RLS Architecture**
   - super_admin → full access | admin → managed properties | user → assigned buildings | tenant → own unit only
   - RLS on every table, not just API-level checks
   - Platform auth + custom tenant auth (bcrypt) for the tenant portal

5. **Format-Agnostic CSV Parsing**
   - Auto-detects vertical vs. horizontal CSV layout
   - Locale-specific decimal handling (German comma vs. period separators)
   - Device type classification across 5 meter categories
   - wMBus protocol parsing for Engelmann HCA devices — discovered and corrected incorrect vendor device spec during implementation

6. **bved API Integration**
   - Standardized API implementation for connecting with 5 German measurement service company (MSC) platforms
   - Comprehensive documentation package: API spec, implementation roadmap, quick start guide, platform onboarding templates

7. **AI-Powered Invoice Processing**
   - Document upload with AI analysis for heating cost invoices via Vercel AI SDK
   - Automated extraction of cost categories and allocation factors

---

## Technical Implementation

### IoT Ingestion Pipeline

```
Gateway Telegram (MQTT)
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  MQTT Sub   │────▶│  Decode     │────▶│  Protocol   │
│  (Node.js)  │     │  (CBOR)     │     │  Parse      │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                         ┌────────────────────┤
                         ▼                    ▼
                   ┌──────────┐        ┌──────────┐
                   │  Heat    │        │  Water/  │
                   │  Alloc.  │        │  Heat/   │
                   │          │        │  Elec.   │
                   └────┬─────┘        └────┬─────┘
                        └────────┬──────────┘
                                 ▼
                   ┌──────────────────────┐
                   │  Deduplicate         │
                   │  (date + device_id)  │
                   └──────────┬───────────┘
                              ▼
                   ┌──────────────────────┐
                   │  PostgreSQL Insert   │
                   └──────────────────────┘
```

### Database Architecture

```
Relational Model (20+ tables, Drizzle ORM):

Users ──┐
        ├──▶ Buildings
        │         │
        │         ├──▶ Units
        │         │       │
        │         │       ├──▶ Contracts
        │         │       │       └──▶ Contractors
        │         │       │
        │         │       └──▶ Unit_Meters
        │         │               └──▶ Meter_Readings
        │         │
        │         ├──▶ Billing_Documents
        │         │       └──▶ Cost_Categories
        │         │
        │         └──▶ Invoice_Documents
        │
        └──▶ RLS Policies (every table)
                 ├── is_admin() checks
                 ├── auth.uid() scoping
                 └── 4-role visibility hierarchy
```

### Performance: 82% LCP Improvement

```
Before: 15.6s mobile LCP

Optimizations Applied:
  ├── Deferred Lottie animations (682KB → 277KB, 59% payload reduction)
  ├── WebP/AVIF image enablement
  ├── Reduced hero image dimensions
  ├── Font loading optimization
  ├── Hero image priority hints
  └── Dashboard query index optimization

After: 2.8s mobile LCP ✅
```

### Challenges Solved

| Challenge | Solution |
|-----------|----------|
| Multiple CSV formats from different manufacturers | Format-agnostic parser with vertical/horizontal layout detection and locale-specific decimal handling |
| Wireless meter protocol decoding | Protocol-level parsing; discovered and corrected incorrect device spec from vendor documentation |
| Dashboard showing cumulative instead of consumption | Rewrote chart logic to calculate deltas between consecutive readings |
| 15.6s mobile LCP | 6-layer systematic optimization informed by Lighthouse profiling |
| Phantom devices from counting bug | Signature-based deduplication (date + device_id) — eliminated ~10x inflated count |
| Admin crashes from undefined user IDs | 5-layer guard system: state deserialization fix, URL guards, auth hook guards, error boundaries |
| Production CVE in React core | Same-day patch of CVE-2025-55182 with React 19.1.2 upgrade |

---

## Results

### Codebase Scale

| Metric | Value |
|--------|-------|
| Total commits | 1,746 across 15 contributors |
| My commits | 337 (209 feature + 128 merge) |
| PRs merged (by me) | 100+ |
| API endpoints | 62 in production |
| Page routes | 119 across admin, tenant, and shared flows |
| Components | 368+ organized by domain |
| Server actions | 49 with admin/user permission separation |
| Database tables | 20+ with full RLS coverage |

### Performance

| Metric | Before → After | Improvement |
|--------|---------------|-------------|
| Mobile LCP | 15.6s → 2.8s | 82% |
| Animation payload | 682KB → 277KB | 59% reduction |
| Device counting accuracy | ~10x inflated → correct | Fixed counting bug |

### Security

| Measure | Implementation |
|---------|---------------|
| CVE Response | Same-day patch of CVE-2025-55182 (React RCE) with React 19.1.2 upgrade |
| Spam Protection | 5-layer system: honeypot, rate limiting, validation |
| Webhook Security | Removed exposed URLs, fixed race conditions |
| RLS Architecture | Row-level security on every table with 4-role policy system |
| GDPR Compliance | Data isolation by design, encrypted credentials, secure credential sharing |
| Admin Stability | 5-layer guard: Zustand deserialization fix, URL guards, auth hook guards, error boundaries |

---

## Architecture Principles Applied

1. **Fix deduplication at ingestion, not at query time** — The phantom device bug proved that without signature-based deduplication, every downstream metric, dashboard, and billing calculation is wrong.

2. **RLS at the database level, not just the API** — API-level permission checks can be bypassed or forgotten. RLS on every table means a developer mistake can't leak tenant data.

3. **Store raw readings, compute deltas** — Cumulative readings preserve data fidelity. If a reading is missed, delta computation self-corrects. Storing pre-computed deltas propagates errors permanently.

---

## Tech Stack

- **Framework**: Next.js 16 (App Router, Turbopack), React 19, TypeScript 5
- **Database**: Supabase (PostgreSQL), Drizzle ORM, Row-Level Security
- **Auth**: Platform auth + custom tenant auth (bcrypt)
- **IoT**: MQTT (Mosquitto), wireless meter protocol parsing, Docker microservices
- **State**: Zustand, TanStack React Query
- **UI**: Tailwind CSS 4, Radix UI, Recharts, Framer Motion
- **PDF**: React-PDF Renderer (regulation-compliant billing documents)
- **Email**: React Email + Edge Functions (cron delivery)
- **AI**: Vercel AI SDK (invoice processing)
- **Automation**: Webhook integrations, Slack API
- **Hosting**: Vercel (serverless)
