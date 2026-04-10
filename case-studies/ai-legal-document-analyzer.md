# AI-Powered Legal Document Analysis Platform — LPA Analyzer

> 150-200 page documents processed in minutes, 6 clause categories extracted, 4-6 hours saved per review

**Role**: Lead AI Architect & Full-Stack Engineer
**Domain**: Legal Tech / Investment Fund Law
**Outcome**: Production AI legal analysis platform with RAG-powered document intelligence and Microsoft Word Add-in

---

## The Problem

Investment fund lawyers spend 4-8 hours per LPA manually reviewing 150-200 page documents, hunting for key clauses buried across dozens of sections. They cross-reference definitions scattered throughout the agreement, compare terms against market standards from memory, and flag risks based on years of pattern recognition. When a fund is raising capital, a single lawyer might review 10-20 LPAs in a month — that's 40-160 hours of dense, repetitive analytical work.

The firm needed a system that could read an entire LPA, extract the clauses that matter, analyze them against market standards, and surface risks — without the lawyer ever leaving Microsoft Word. Not a generic document chatbot. A purpose-built legal analysis engine that understands investment fund terminology, cross-references, and the specific clause categories that drive deal negotiations.

---

## My Solution

A production AI legal analysis platform that transforms how lawyers review Limited Partnership Agreements. Two interfaces — a web dashboard for document management and a Microsoft Word Add-in for in-document analysis — powered by a unified RAG backend that learns from the firm's own document library.

**The core challenge:** Legal documents aren't like blog posts or emails. A single clause can span 3 pages, reference 15 defined terms from other sections, and contain nested exceptions that completely change its meaning. The system had to parse document structure intelligently, chunk without breaking legal context, and deliver analysis that a senior fund lawyer would trust enough to use in client negotiations.

### Architecture Overview

```
                    ┌──────────────────────────────────┐
                    │         Web Dashboard             │
                    │   (Next.js 15 + React 19)         │
                    └───────────────┬──────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
    ┌───────▼────────┐     ┌───────▼────────┐     ┌───────▼────────┐
    │  Word Add-in   │     │   API Routes   │     │   Dashboard    │
    │  (Office.js)   │     │  (Serverless)  │     │     Pages      │
    └───────┬────────┘     └───────┬────────┘     └────────────────┘
            │                      │
            └──────────┬───────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼──────┐┌──────▼──────┐┌──────▼──────┐
│   Document   ││    RAG      ││  Analysis   │
│  Processing  ││  Pipeline   ││    Cache    │
│   (QStash)   ││(Dify Cloud) ││ (PostgreSQL)│
└───────┬──────┘└──────┬──────┘└─────────────┘
        │              │
        └──────┬───────┘
               │
    ┌──────────▼──────────┐
    │   Neon PostgreSQL   │
    │  + pgvector (1536d) │
    │  Multi-Tenant       │
    └─────────────────────┘
```

### Key Technical Decisions

1. **Parallel Chunk Processing Over Sequential**
   - A 200-page LPA produces ~50 chunks at 4,000 chars each
   - Sequential processing: 25-50 minutes. Parallel via QStash: minutes
   - Trade-off: infrastructure complexity (atomic progress tracking, deduplication, rate-limit management) for dramatic UX improvement

2. **TOC-Aware Chunking Over Naive Splitting**
   - Naive text splitting breaks clauses mid-sentence or separates provisions from their definitions
   - TOC-aware chunking maps document structure first, creates chunks respecting section boundaries
   - Each chunk contains a complete legal thought — the AI never analyzes half a clause

3. **Microsoft Word Add-in Over Standalone Web App**
   - Lawyers live in Word. An analysis tool requiring copy-paste to a browser is dead on arrival
   - The Add-in keeps clause, surrounding language, analysis, and drafting suggestions in one view
   - Adoption becomes frictionless — meets the user where they already work

4. **Multi-Tenant Isolation at Database Level**
   - Explicit organization_id enforcement on every table, every query, every index
   - Composite indexes starting with organization_id ensure tenant-scoped queries use index scans
   - CASCADE deletes for clean tenant data removal — defense-in-depth for confidential legal documents

5. **Dify Cloud Over Custom RAG Pipeline**
   - Provides workflow orchestration, knowledge base management, model routing, and zero-retention processing out of the box
   - Building from scratch would add 4-6 weeks for infrastructure that doesn't differentiate the product
   - The value is in the legal domain logic — prompts, clause-type calibration, market-standard benchmarking

---

## Technical Implementation

### 1. Intelligent Document Processing Engine

Takes a raw .docx LPA (150-200 pages), parses its structure, and extracts every instance of 6 critical clause categories with confidence scoring.

The pipeline starts with TOC extraction using 4 regex patterns to map the document's structure. A structure analyzer identifies section boundaries, headings, and clause-type hints. Smart chunking splits the document into ~4,000-character segments that respect section boundaries. Each chunk is queued as an independent background job via QStash, enabling 100+ chunks to process in parallel across serverless functions.

**Two-pass extraction strategy:**
- **Phase 1**: TOC navigation for targeted extraction from likely sections (high confidence)
- **Phase 2**: Comprehensive fallback extraction for anything missed, handling oversized sections (>15,000 chars) that Phase 1 intentionally skips for speed

### 2. RAG-Powered Clause Analysis

When a lawyer clicks "Analyze" on a clause:
1. Check analysis cache (SHA-256 hash of clause text, 7-day TTL)
2. On cache miss, call Dify workflow querying Knowledge Base (top-k: 10 chunks from the firm's LPA corpus)
3. GPT-4o generates structured analysis with cross-document context
4. Response parsed into 4 sections and cached

**4-part analysis framework** designed around how senior fund lawyers actually think:
- **Summary:** 2-3 sentences with section citations — what a partner would say in a 30-second elevator brief
- **Detailed Analysis:** Thresholds, triggers, consequences, fully unpacked cross-references
- **Key Considerations:** Market standard comparison (e.g., "GP removal typically requires 50-75% LP vote"), LP vs GP favorability, negotiation leverage, red flags
- **Definitions:** Every defined term referenced in the clause, with full definitions and source locations

Each clause type has calibrated focus areas. For Waterfall clauses: benchmarks against typical 8% hurdle rates and 80/20 LP/GP splits. For Key Person clauses: evaluates time commitment and replacement procedures. For GP Removal: flags voting thresholds outside 50-75% market standard.

### 3. Microsoft Word Add-in

Built with Office.js and React, the Add-in installs as a sidebar panel. Each clause category gets a distinct color:
- **Key Person:** Yellow — time commitments, named individuals, replacement triggers
- **GP Removal:** Blue — voting thresholds, cause definitions, cure periods
- **Term & Termination:** Green — partnership term, extension mechanics, dissolution events
- **Waterfall:** Purple — distribution priorities, hurdle rates, catch-up provisions
- **Clawback:** Red — trigger events, calculation methods, escrow requirements
- **LPAC:** Grey — committee composition, conflict resolution, advisory powers

Click any clause in the sidebar → navigates to exact location in document with highlighting. Expand for one-click AI analysis, drafting suggestions, and cross-document context — all inline without leaving Word.

---

## Results

| Metric | Value |
|--------|-------|
| Document processing | 150-200 page LPAs in minutes (parallel) |
| Clause categories | 6 critical types (expandable to 20) |
| Chunk processing | 100+ simultaneous jobs via QStash |
| Analysis depth | 4-part structured output per clause |
| RAG retrieval | Top-k 10 chunks from full LPA corpus |
| Analysis caching | 7-day TTL, SHA-256 hash keys |
| API cost reduction | ~60% on repeat queries via caching |
| Drafting suggestions | 2-3 per clause with severity prioritization |
| Vector dimensions | 1536 (OpenAI ada-002) with IVFFlat indexing |
| Tenant isolation | Organization-scoped on all 11 tables |
| Estimated time saved | 4-6 hours per LPA review |

---

## Architecture Principles Applied

1. **Meet users where they work** — The Word Add-in decision was the single most important architectural choice. A superior web-only tool that disrupts existing workflow loses to a good-enough tool embedded in the existing workflow.

2. **Structure-aware processing beats brute force** — Understanding document structure (TOC, sections, clause boundaries) before chunking produces dramatically better extraction quality than naive text splitting.

3. **Cache at the semantic level** — SHA-256 hashing of clause text means identical clauses across different LPAs share cached analysis. 60% cost reduction from repeat queries across the firm's document library.

4. **Zero-retention is a feature, not a constraint** — For legal documents, the inability to store client data in third-party AI services is a selling point. Architecture that enforces zero-retention by design builds trust with regulated clients.

---

## Tech Stack

- **AI/LLM:** Dify Cloud (GPT-4o, multi-model routing), OpenAI Embeddings (ada-002), pgvector
- **Framework:** Next.js 15, React 19, TypeScript, App Router
- **Database:** Neon PostgreSQL, pgvector (1536-dim), multi-tenant schema
- **UI:** Tailwind CSS, Radix UI, Framer Motion, next-themes (dark mode)
- **Word Plugin:** Office.js, React, TypeScript, Webpack
- **Auth:** Clerk (SSO, organization management, webhook sync)
- **Background Jobs:** QStash (Upstash) with retry logic and rate limiting
- **Document Processing:** Mammoth (.docx parser), custom structure analyzer, targeted extractor
- **Deployment:** Vercel (edge + serverless), Neon (managed PostgreSQL)
- **Security:** Zero-retention AI, multi-tenant isolation, audit logging, HTTPS-only
