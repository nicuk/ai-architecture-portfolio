# AI-Powered Marketing Automation Platform — Full Lifecycle Content, Leads & Outreach

**Role:** Fractional CTO & AI Lead Architect | Sole Engineer
**Engagement:** 18-week client delivery | 6 milestones | Production system deployed and in daily use

---

## Executive Summary

Designed, architected, and delivered — as the sole engineer — a production multi-agent AI marketing platform that automates the full content-to-outreach lifecycle for a solo creative business. The system orchestrates five specialized AI tools — Content Engine, Lead Discovery, Email Intelligence, Content Ops, and Image Bot — through an Enhanced Supervisor pattern, replacing 15–20 hours/week of manual marketing work with AI-driven automation.

The hardest constraint wasn't technical — it was fidelity. The system had to replicate the client's distinctive writing voice with enough precision that galleries, collectors, and industry contacts couldn't distinguish AI-generated communications from the artist's own writing. A hybrid RAG pipeline trained on 80+ writing samples achieved this through dual-retrieval scoring (70% vector similarity, 30% full-text keyword) — validated by the client's professional network, where gallery contacts and collectors consistently could not identify which communications were AI-generated versus hand-written.

End-to-end sole delivery: architecture design, tech stack selection, database modelling, AI pipeline engineering, third-party API integrations, UI/UX, security hardening, deployment, and iterative client feedback across 6 paid milestones. No additional engineering resources.

---

## The Problem I Was Hired to Solve

A globally exhibited contemporary abstract artist was losing 15–20 hours per week to marketing operations: writing blog posts, adapting content for social media, manually sourcing gallery leads, scoring prospects by hand, managing email follow-ups, and uploading artwork images one by one. Every hour spent on admin was an hour not painting — and the inconsistency of manual outreach was leaving money on the table.

**Three constraints shaped the architecture:**

1. **Voice authenticity** — AI-generated content had to be indistinguishable from the client's natural voice (a blend of cosmic philosophy, technical art vocabulary, and emotional vulnerability). Generic AI output would damage gallery relationships.
2. **Operational cost** — A solo business can't afford enterprise SaaS pricing. The system needed to run on < $225/month total API spend.
3. **Single-operator UX** — No team to manage the platform. Every workflow had to be operable from a single dashboard with minimal clicks.

---

## Architecture: Enhanced Supervisor Pattern

I evaluated three architectural approaches before committing:

| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| **Autonomous multi-agent (LangGraph/CrewAI-style)** | Flexible, extensible | Inter-agent serialization overhead, state sync complexity, unpredictable costs from agent loops | Rejected |
| **Microservices with message queue** | Clean separation, independently deployable | Operational overhead disproportionate to solopreneur scale; monitoring/retry complexity | Rejected |
| **Enhanced Supervisor with tool dispatch** | Shared context, zero coordination overhead, predictable cost | Tighter coupling (acceptable at this scale) | **Selected** |

The Enhanced Supervisor acts as a task router — not an LLM loop. It lazy-instantiates specialized tools and dispatches via typed `execute({ type, payload })` calls. Every tool shares the same voice model, lead database, conversation memory, and engagement history through a common Supabase layer.

```
                         ┌────────────────────────────┐
                         │    Enhanced Supervisor      │
                         │    (Task Router + State)    │
                         └─────────────┬──────────────┘
              ┌────────────┬───────────┼───────────┬────────────┐
              ▼            ▼           ▼           ▼            ▼
       ┌────────────┐┌────────────┐┌─────────┐┌────────────┐┌─────────┐
       │  Content   ││   Lead     ││  Email  ││  Content   ││  Image  │
       │  Engine    ││ Discovery  ││  Intel  ││    Ops     ││   Bot   │
       └──────┬─────┘└──────┬─────┘└────┬────┘└──────┬─────┘└────┬────┘
              │             │           │            │           │
              ▼             ▼           ▼            ▼           ▼
       ┌────────────┐┌────────────┐┌─────────┐┌────────────┐┌─────────┐
       │ Voice RAG  ││ Apollo.io  ││ Gmail   ││ WordPress  ││ AI Tag  │
       │ + pgvector ││ Firecrawl  ││ OAuth   ││ REST API   ││ Engine  │
       │            ││ Hunter.io  ││ Pub/Sub ││ Social     ││ 552 img │
       │            ││            ││ SendGrid││ Adapters   ││ indexed │
       └────────────┘└────────────┘└─────────┘└────────────┘└─────────┘
              │             │           │            │           │
              └─────────────┴───────────┴────────────┴───────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  Supabase Postgres  │
                              │  pgvector · RLS     │
                              │  22 migrations      │
                              │  Shared state layer  │
                              └────────────────────┘
```

**Why this matters:** When Lead Discovery finds a gallery and Email Intelligence tracks a conversation with them, Content Engine already knows their preferences when generating a follow-up proposal — zero additional API calls, zero context-passing overhead. Cross-tool intelligence is a consequence of the architecture, not a feature bolted on afterward.

**The dispatch pattern in practice:**

```typescript
// Enhanced Supervisor — typed task dispatch with lazy tool instantiation
async execute(task: { type: string; payload: any }): Promise<any> {
  const startTime = Date.now();
  const { type, payload } = task;

  switch (type) {
    case 'leadDiscovery':
      return await this.getLeadDiscoveryTool().discover(payload);
    case 'leadScoring':
      return await this.getLeadScoringTool().score(payload);
    case 'outreach':
      return await this.getOutreachTool().send(payload);
    case 'crm':
      return await this.getCRMTool().sync(payload);
    default:
      throw new Error(`Unknown task type: ${type}`);
  }
}
```

Each getter lazily instantiates its tool on first use, keeping memory allocation proportional to active workflows — not total system capacity.

---

## The Five AI Tools

### 1. Content Engine — Brand Voice at Scale

**Problem:** Generic AI content would be immediately recognizable to the client's professional network. The system needed to replicate voice with enough fidelity that recipients couldn't tell the difference.

**Architecture decision — Hybrid RAG over fine-tuning:**

Fine-tuning creates a static model that degrades as the client's voice evolves and requires expensive retraining cycles. I built a hybrid retrieval pipeline instead:

- **80+ writing samples** ingested with OpenAI `text-embedding-3-small` embeddings stored in pgvector
- **Dual retrieval:** vector similarity search (pgvector RPC) + PostgreSQL full-text search with Reciprocal Rank Fusion — weighted 70% dense / 30% sparse
- **Dynamic adaptation:** new samples are indexed immediately — no retraining, no waiting, no cost
- **Context window optimization:** top-K relevant samples injected into the generation prompt, keeping token usage minimal

```typescript
// Hybrid RAG — Reciprocal Rank Fusion combining vector + keyword search
async hybridSearch(query: string, topK = 5, vectorWeight = 0.7, keywordWeight = 0.3) {
  // Dense: pgvector cosine similarity via Supabase RPC
  const vectorResults = await this.supabase.rpc('match_voice_samples', {
    query_embedding: await this.embed(query),
    match_count: 20,
  });

  // Sparse: PostgreSQL full-text search on tsvector column
  const keywordResults = await this.supabase
    .from('voice_samples')
    .select('id, content, type')
    .textSearch('fts', sanitizedQuery, { type: 'websearch', config: 'english' })
    .limit(20);

  // Fuse: vector similarity * 0.7 + RRF(keyword_rank, k=60) * 0.3
  return this.combineScores(vectorResults, keywordResults, vectorWeight, keywordWeight, topK);
}
```

**Result:** Voice fidelity validated by the client's professional gallery network — contacts and collectors consistently unable to distinguish AI-generated communications from the artist's own writing. 8–13 second generation time for 800+ word blog posts.

---

### 2. Lead Discovery — Multi-Source Intelligence Pipeline

**Problem:** The client was manually Googling galleries and copying contact info into spreadsheets. No systematic prospecting, no scoring, no enrichment.

**What I built:**

- **Multi-source discovery:** Apollo.io (structured B2B search), Firecrawl (AI-planned web crawl with LLM-generated search strategies), Hunter.io (email verification + fallback)
- **AI discovery agent:** A Firecrawl-powered `source-crawl-agent` that uses LLM planning to generate discovery strategies, crawl relevant web pages, and extract structured lead data
- **Weighted scoring engine:** 0–10 scale across 6 profile dimensions (lead type 30%, job title 20%, location 15%, company 15%, budget signal 10%, industry relevance 10%) plus an additive engagement bonus (up to +3 points from conversation signals), with 9 market-specific lead type categories and calibrated base scores

```typescript
// Scoring weights — 6 profile dimensions sum to 1.0
const weights = {
  leadType:           0.30,  // Gallery, collector, HNWI, art consultant, etc.
  jobTitle:           0.20,  // Decision-making authority
  location:           0.15,  // Primary/secondary art market city
  company:            0.15,  // Gallery, museum, design firm recognition
  budgetFit:          0.10,  // Purchasing signal
  industryRelevance:  0.10,  // Art, design, real estate adjacency
};
// Engagement: additive bonus (0–10 × 0.30 = up to +3 on top of profile score)
// Sources: email thread depth, conversation stage, relationship status, inbound signals
```

- **HITL review queue:** Discovered leads enter a queue for human review before entering the pipeline — preventing garbage-in problems
- **Deduplication + enrichment:** Cross-source dedup utilities, enrichment data preserved across re-scoring cycles
- **CRM sync:** Bidirectional Pipedrive integration with dedicated migration and sync service

**Result:** 69+ qualified leads discovered and scored in the live system. 9 lead type categories with differentiated scoring (a gallery director in London scores differently than an interior designer in Dubai). Temperature classification (hot/warm/cold) driven by real engagement signals, not just profile fit.

---

### 3. Email Intelligence — Context-Aware Conversation Management

**Problem:** The client was losing track of conversations, missing follow-ups, and spending hours drafting replies that referenced past context.

**What I built:**

- **Real-time monitoring:** Gmail OAuth 2.0 + Google Pub/Sub webhooks for instant email detection, with Vercel cron fallback polling every 2 minutes
- **Intent classification:** Incoming emails classified by LLM into structured types (inquiry, booking, complaint, follow-up, question) with sentiment, urgency, and stage extraction
- **Memory extraction:** Each email interaction extracts structured memory blocks — who they are, their situation, preferences, budget signals, relationship history
- **Voice-grounded drafting:** Reply drafts generated using conversation history + lead memory + voice RAG samples, producing responses in the client's authentic voice
- **8-stage engagement pipeline:** Relationships tracked from Prospecting → Meeting Booked with stage-specific follow-up timing and priority-based automation
- **Pattern learning:** `lead_patterns` table + quality monitoring service learn from classification outcomes to improve accuracy over time

**Result:** 5%+ email response rate on cold creative outreach (industry benchmarks for cold email in creative industries typically range 2–3%). 100+ relationships tracked with full context history. Automated follow-up queue with priority-based scheduling.

---

### 4. Content Ops — Single-Source Multi-Channel Publishing

**Problem:** One blog post had to be manually reformatted for Instagram (with hashtags and alt text), LinkedIn, Twitter/X (under 280 chars), WordPress, and email newsletters. Each adaptation took 15–30 minutes.

**What I built:**

- **Content polish pipeline:** Raw notes → structured blog with title, meta description, SEO fields, and image placement markers
- **Multi-format adaptation:** Single content item auto-adapted to blog, Instagram caption (hashtags, alt text, preserved links), LinkedIn post, Twitter/X thread, and newsletter HTML
- **WordPress publishing:** Direct REST API integration with featured image support, SEO metadata, and media linkage
- **Content calendar + library:** Centralized content management with status tracking, scheduling, and historical reference

**Result:** 855-word polished blog from rough notes in ~38 seconds. One-click publishing to WordPress. Auto-adapted to 4+ social formats with zero manual reformatting.

---

### 5. Image Bot — AI-Powered Artwork Library

**Problem:** 552 artwork images with no organization system. Manually searching for the right image for each blog section was tedious and inconsistent.

**What I built:**

- **AI analysis pipeline:** Every image processed at upload with AI vision analysis (Gemini Flash via OpenRouter) — themes, colours, mood, style, subjects, SEO keywords stored as pre-computed tags and vector embeddings
- **Two-tier matching:** Fast tag-based matching on stored metadata first (zero API cost); vector similarity fallback via pgvector RPC only when tag results are insufficient
- **Section-aware suggestions:** Blog `[Insert Image: description]` markers matched to relevant artwork using keyword extraction against pre-computed tags — 4 targeted suggestions per marker with weighted random sampling to ensure diversity
- **Format support:** JPEG, PNG, WebP, and HEIC (iPhone photos) with `sharp` and `heic-convert`
- **Featured image auto-selection:** Best match automatically suggested as the blog's hero image

**Performance engineering decision:** The original approach would have required an LLM call for every image comparison. With 552 images × 5–8 markers per blog, that's 2,700–4,400 API calls per content piece. I redesigned this as a pre-computed tag-matching system — one-time AI analysis per image at upload, then pure keyword/tag matching at query time with vector fallback.

```typescript
// Image matching — tags first, vectors only as fallback
async matchImagesToContent(content: string, options?) {
  // Fast path: keyword extraction → match against stored themes, subjects, colors, mood
  const tagResults = await this.matchByTags(content, options);

  if (tagResults.length >= (options?.limit || 5) || options?.useTagsOnly) {
    return tagResults; // Zero API cost
  }

  // Slow path: generate embedding → pgvector cosine similarity (only if tags insufficient)
  const embedding = await this.generateEmbedding(this.extractContentThemes(content));
  return await this.supabase.rpc('match_artwork_images', {
    query_embedding: embedding,
    match_threshold: 0.3,
    match_count: options?.limit || 5
  });
}
```

**Result:** Sub-1-second tag-based matching (down from 12–16 seconds with per-request LLM calls). Zero incremental API cost per match on the fast path. 552 images fully indexed and searchable.

---

## Technical Decisions That Defined the Project

### Database Architecture — 22 Migrations, Schema Evolution Under Production Load

The Supabase PostgreSQL schema evolved across 22 migrations (numbered non-sequentially up to 027, reflecting real-world iteration) as requirements crystallized through client feedback:

| Domain | Key Tables | Notable Design |
|--------|-----------|----------------|
| Voice/RAG | `voice_samples` | pgvector embeddings + FTS tsvector columns + `match_voice_samples` RPC with type filtering |
| Leads | `m3_leads`, `discovery_queue`, `lead_patterns` | Multi-source normalization, HITL review queue, pattern learning |
| Email | `conversation_history`, `lead_memory`, `processed_emails` | Memory block extraction, engagement event tracking |
| Content | `content_items`, `content_calendar`, `wordpress_config` | Multi-format JSON, SEO fields added in later migration |
| Images | `artwork_images` | AI tags, vector embeddings, FTS, usage tracking with increment functions |
| CRM | `pipedrive_sync` | Bidirectional sync state management |
| Security | RLS policies (migration 014), secure functions (015) | Row-level isolation, internal API key auth |
| Observability | `quality_metrics`, `activity_log`, `engagement_events` | Operational monitoring, pattern-based learning |

**Key decision:** I chose schema evolution through numbered migrations over a "design it all upfront" approach. In a milestone-based client engagement, requirements shift as the client uses each delivered tool. Later migrations (nullable email normalization, source cleanup, SEO fields, relationship status, voice search type filters) all emerged from real production usage feedback — not upfront guesswork.

### Security Posture

- **Authentication:** Cookie-based session with bcrypt password hashing — appropriate for single-operator platform
- **Authorization:** Row Level Security enabled across all tables (migration 014) with secure function wrappers (migration 015)
- **API protection:** `INTERNAL_API_KEY` guards internal routes; middleware restricts access to whitelisted emails
- **Gmail webhook:** Public endpoint with Google Pub/Sub verification, processed emails stored with deduplication
- **Unified spam classification:** Dedicated migration for spam pattern unification across email and lead sources

### Cost Engineering — Production AI at Solopreneur Scale

Keeping total operational cost under $225/month required deliberate cost-aware architecture:

| Cost Decision | Impact |
|---------------|--------|
| OpenRouter as LLM gateway | Single billing surface, instant model switching (Haiku for drafts, Sonnet for complex tasks), no vendor lock-in |
| Hybrid RAG over fine-tuning | $0 retraining cost when voice evolves; embedding cost amortized at ingest time |
| Pre-computed image tags | Eliminated 2,700–4,400 LLM calls per content piece; reduced to zero incremental cost on the fast path |
| Vercel edge + serverless | No idle compute costs; cron-based polling instead of persistent connections where possible |
| Selective model routing | Claude Haiku for routine content; Sonnet for high-stakes drafts; Gemini Flash for image vision analysis |

**Total monthly operating cost:** ~$180–225 across all integrated APIs (OpenRouter, Supabase, Gmail, Apollo, Hunter, Firecrawl, Vercel).

### Operational Automation

Three Vercel cron jobs handle background operations without manual intervention:

- **Email polling** — Every 2 minutes, fallback for Pub/Sub webhook gaps
- **Weekly lead discovery** — Automated prospecting runs with configured search parameters
- **Monday discover job** — Scheduled market scanning to refresh the pipeline

---

## Challenges and Key Pivots

### Voice RAG vs. Direct Prompting (M1 → M2)

The initial approach embedded voice instructions directly in the system prompt. This produced generically "well-written" content but missed the client's specific cadence — the way they layer cosmic metaphors with grounded studio language. I pivoted to the hybrid RAG pipeline in late M1, which required building the full ingestion, embedding, and dual-retrieval system. The result was a qualitative step-change in output quality, but it added ~2 weeks to the M1 timeline.

### Content Generation Decoupled from Supervisor (M4)

Content generation was originally routed through the Supervisor's `execute()` dispatch. As the Content Engine's requirements grew (voice scoring, multi-format output, SEO field generation, image marker insertion), the coupling became a liability — every Content Engine change forced regression testing across the Supervisor. I extracted it into a standalone `ContentPolishTool` accessed via its own API route. This was the right call for maintainability, but it broke the "everything through the Supervisor" abstraction, requiring documentation and API route restructuring.

### Image Matching — From LLM-Per-Comparison to Pre-Computed Tags (M5 → M6)

The original image matching design called the LLM for every image-to-content comparison. First production test with 552 images: 16+ seconds per marker, $2–4 per blog post in API calls. Completely unacceptable at solopreneur scale. I redesigned the pipeline as a one-time AI vision analysis at upload (Gemini Flash) storing structured tags, then pure Postgres keyword matching at query time with vector fallback. Reduced latency from 16s to <1s, cost from $2–4 to effectively $0 per match.

### Google Places Integration — Scope Decision

I built a Google Places API service for location-based gallery discovery. During integration testing, the data quality for art-specific queries was poor compared to Apollo.io + Firecrawl's targeted crawling. Rather than ship a feature that would pollute the lead pipeline with low-quality data, I kept the service implemented but unwired — available for future activation if the client expands into location-based prospecting. This was a deliberate scope trade-off: shipping less, but shipping clean.

---

## Measurable Outcomes

| Metric | Value | Context |
|--------|-------|---------|
| Voice fidelity | Validated by professional network | Gallery contacts unable to distinguish AI from hand-written communications |
| Content generation | 8–13s / 800+ word blog | Via OpenRouter model routing |
| Image matching | < 1 second (fast path) | Down from 16s; zero incremental API cost on tag-based matches |
| Email response rate | 5%+ | Cold creative outreach; industry benchmark ~2–3% for comparable segments |
| Relationships tracked | 100+ | Full context history, 8-stage engagement pipeline |
| Lead type categories | 9 | Market-specific with calibrated scoring weights and engagement signals |
| Artwork indexed | 552 | AI-analysed with themes, mood, style, colour, subject, SEO keyword tags |
| Monthly API cost | ~$180–225 | All integrations combined |
| Time recovered | 15–20 hrs/week | Client's own estimate of manual work eliminated |
| Codebase | 204 TS/TSX files | 22 database migrations, Playwright e2e + API tests |

<!-- NOTE FOR PORTFOLIO: If you have specific client outcomes to add (e.g., gallery conversations
     initiated, exhibitions booked, or revenue attributed to automated outreach), replace this
     comment with a "Client Business Impact" row. Concrete outcomes elevate this from a
     technical case study to a business case study. -->

---

## Delivery Methodology

Structured as a milestone-based engagement with iterative delivery and client feedback loops:

| Milestone | Scope | Architecture Focus |
|-----------|-------|--------------------|
| **M1** | Foundation + Voice Training | Core infrastructure, Supabase schema, OpenRouter integration, hybrid RAG pipeline |
| **M2** | Lead Discovery + Outreach | Multi-source lead pipeline, scoring engine, Apollo/Firecrawl/Hunter integration |
| **M3** | Conversation Intelligence | Gmail OAuth + Pub/Sub, intent classification, memory extraction, engagement pipeline |
| **M4** | Content Operations | Multi-format adaptation, WordPress publishing, content calendar |
| **M5** | Image Library + WordPress | AI analysis pipeline, tag-based matching, HEIC support, media management |
| **M6** | Performance + Polish | Sub-second image matching, bug fixes, UX refinements, production hardening |

Each milestone included: architecture review → implementation → client demo → feedback integration → deployment to production. The system was in active client use from M2 onward, with each milestone building on real usage data.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **AI / LLM** | OpenRouter (Claude Haiku / Sonnet, Gemini Flash), OpenAI Embeddings (`text-embedding-3-small`), pgvector |
| **Framework** | Next.js 14, TypeScript 5, React 18, App Router |
| **Database** | Supabase PostgreSQL, pgvector, Row Level Security, 22 migrations |
| **UI** | Tailwind CSS 4, Radix UI, shadcn/ui, Recharts, Geist |
| **Email** | Gmail OAuth 2.0, Google Pub/Sub webhooks, SendGrid |
| **CRM** | Pipedrive (bidirectional sync) |
| **Lead Intelligence** | Apollo.io, Firecrawl, Hunter.io |
| **Publishing** | WordPress REST API |
| **Deployment** | Vercel (edge + serverless + cron) |
| **Image Processing** | sharp, heic-convert, AI vision analysis pipeline (Gemini Flash) |
| **Testing** | Playwright (e2e + API) |
| **Validation** | Zod, react-hook-form |

---

## What This Demonstrates

**As a Fractional CTO**, this project shows:

- **Architecture under constraints** — Choosing the right level of abstraction for a solopreneur context (Enhanced Supervisor over autonomous agents) and defending that decision against the complexity tax of trendier approaches
- **Cost-aware system design** — Building production AI that operates at $180–225/month, not $2,000/month, through deliberate model routing, pre-computation strategies, and API cost elimination
- **Milestone-driven delivery** — Structuring a complex system build as iterative, demonstrable milestones with client feedback loops — not a waterfall spec-then-build
- **Schema evolution discipline** — 22 production migrations that evolved the data model based on real usage, not upfront guesswork
- **Security by default** — RLS, API key isolation, webhook verification, and session management as architectural primitives, not afterthoughts
- **Scope discipline** — Cutting features that would degrade data quality (Google Places) rather than shipping everything possible

**As an AI Lead Architect**, this project shows:

- **Hybrid RAG engineering** — Vector + full-text search with Reciprocal Rank Fusion for brand voice replication, achieving fidelity that survived professional scrutiny
- **Multi-source data pipeline design** — Three external APIs (Apollo, Firecrawl, Hunter) unified through a common enrichment and scoring layer with HITL review gates
- **Event-driven email architecture** — Gmail Pub/Sub + cron fallback + intent classification + memory extraction, producing context-aware draft replies grounded in conversation history
- **Performance engineering** — Identifying and eliminating a 2,700–4,400 API call bottleneck through pre-computed tag analysis, reducing latency from 16s to < 1s at zero incremental cost
- **Production AI operations** — Pattern learning, quality monitoring, spam classification, and activity logging as first-class system capabilities

---

## Reflections

**What I'd change with hindsight:**

The Enhanced Supervisor was the right call at this scale, but if I were building this for a multi-operator team, I'd introduce a queue-based dispatch layer (BullMQ or Inngest) between the Supervisor and tools from M1 — the direct method call pattern makes horizontal scaling a retrofit rather than a natural extension. The same applies to observability: I added quality metrics and activity logging mid-project, but instrumenting from day one would have caught the image matching performance issue earlier.

**What this architecture looks like at 10× scale:**

The core patterns — hybrid RAG for voice, pre-computed tag matching, engagement-weighted scoring — are scale-invariant. What changes is the dispatch mechanism (queue-based with retry/dead-letter instead of direct calls), the state layer (read replicas or connection pooling under high concurrency), and the ingestion pipeline (streaming embeddings instead of synchronous per-upload). The tool interface (`execute({ type, payload })`) was deliberately designed to make this transition a plumbing change, not an architectural rewrite.

---

**Status:** Production — deployed and in daily client use
**Key surfaces:** Dashboard, Content Creation, Content Library, Leads, Image Library, Voice Training, Inbox
