# Portfolio Case Study: AI-Powered Trader Psychology Platform

## At a Glance

> **Architected and shipped a dual-app behavioral trading platform as sole technical lead — designing a hybrid intelligence system where deterministic scoring handles classification and LLMs handle insight generation, a content architecture that turns a 16-page research paper into 5 fully distinct interactive product experiences, and a revenue funnel that replaces 1:1 discovery calls with an automated diagnosis-to-coaching pipeline integrated with CRM automation.**

---

## Overview

**Project:** AI Trader Psychology Platform
**Role:** Fractional CTO / AI Lead Architect / Sole Developer
**Duration:** ~6 months (two major delivery phases)
**Client:** Founder of a trader education brand with 25+ years of financial advisory experience and proprietary behavioral research based on thousands of retail trader conversations
**Status:** Production (deployed on Vercel, generating revenue)

### Tech Stack Decisions

| Layer | Choice | Why This Over Alternatives |
|-------|--------|---------------------------|
| Framework | Next.js 14 (App Router) | Dynamic `[avatar]` routes serve 5 distinct experiences from one codebase — SSR for SEO on landing/quiz, client-side for interactive report components |
| AI (Quiz) | OpenAI GPT-4o-mini | Fast, cheap ($0.15/1M input), structured output via Zod schemas — personality insights don't need frontier reasoning |
| AI (Consultation) | Google Gemini 2.5 Flash | Long-context adaptive questioning at lower cost than GPT-4 — consultation sessions run 50+ exchanges |
| Scoring | Centroid-based + AI overlay | Deterministic classification prevents hallucinated personality types; AI adds insight layer on top of verified scores |
| Payments | GHL Payment Links + Stripe verification | Client's existing GHL ecosystem handles checkout; Stripe as server-side verification fallback for edge cases |
| PDF | @react-pdf/renderer | Native vector PDF — no browser dependency, no html2canvas quality loss, prints cleanly at any DPI |
| CRM | GoHighLevel (4 triggers, 13 tags) | Client's existing CRM; non-blocking integration so CRM failures never break the user funnel |
| State | localStorage + Supabase | Quiz funnel requires zero-auth UX (no login wall before checkout); Supabase for server-side session persistence in consultation engine |
| Hosting | Vercel (serverless) | Zero-config deployment, edge functions for API routes, automatic preview deploys for client review |

---

## The Architecture Problem

The client had a 16-page behavioral research paper based on thousands of trader conversations, a proven 90-minute consultation methodology, and a coaching program — but no way to deliver any of it at scale. Every diagnosis required a live 1:1 call. Every conversion depended on the founder's personal time.

**The architecture challenge was three-fold:**

1. **Scale the diagnosis** — turn a subjective 1:1 assessment into a repeatable, deterministic classification system that still feels personal
2. **Scale the content** — deliver 5 psychologically distinct product experiences without maintaining 5 separate codebases or copy-pasting content with find-and-replace
3. **Scale the conversion** — build a funnel where the diagnosis itself creates the demand for coaching, with CRM automation handling the follow-up

---

## Architecture Decision Records

### ADR-1: Hybrid Intelligence — Deterministic Scoring + AI Insight Layer

**Context:** The platform classifies traders into 5 personality archetypes. A pure AI approach (send answers to GPT, get a type back) would be simpler but introduces a critical risk: LLMs can hallucinate classifications, and an inconsistent personality result destroys product credibility.

**Decision:** Two-layer architecture:
- **Layer 1 (Deterministic):** Centroid-based scoring across 4 behavioral dimensions (Risk Tolerance, Emotional Control, Methodical Approach, Discipline). Each answer maps to dimension weights. Final classification is the nearest centroid — mathematically reproducible, auditable, zero hallucination risk.
- **Layer 2 (AI Insight):** GPT-4o-mini receives the verified scores and generates mathematical trading insights, psychological micro-traits, and personalized recommendations. The AI enhances the result but never determines it.

**Trade-off:** More complex architecture than a single LLM call. Worth it because the personality type drives everything downstream — the report content, the CRM tags, the coaching recommendation, and the PDF. A wrong classification cascades through the entire product.

**Hybrid Detection:** When a user's distance between their top two archetype centroids exceeds a threshold (>0.30), they're flagged as a hybrid. The report renders the primary type's full experience with a secondary type note and dual KPI guidance. This handles the ~15% of users who genuinely sit between two types instead of forcing them into a bad fit.

### ADR-2: Content Architecture — Typed Data Model Over Hardcoded Views

**Context:** 5 archetypes, each needing 25+ unique content elements (4 score meanings, 5 blind spots with examples, cost calculator framing, 3 Week 1 tasks, 3 locked week teasers, 5 coaching bullets, a tagline, and an "honest reality" paragraph). Hardcoding this into 5 separate page files would create unmaintainable duplication.

**Decision:** Single typed data model (`AvatarV5Data` interface) storing all content per archetype. 9 shared React components consume this data through props. One codebase, one set of components, 5 distinct experiences.

**Result:** 311 lines of typed content drives the entire V5 report. Adding a 6th archetype would require adding one data entry — zero component changes. Content accuracy is enforced by TypeScript: if a field is missing, the build fails.

**Content Source:** Every data entry maps directly to a specific section of the client's behavioral research. This isn't generated content — it's the client's domain expertise structured into a typed product model. The AI didn't write the blind spots; the client's research defined them, and the architecture made them deliverable at scale.

### ADR-3: Revenue Funnel Architecture — Perceived Value Gap

**Context:** The product needs to convert free quiz-takers into paid report buyers. The standard approach (paywall after quiz) creates friction. The client's insight was that traders need to *see* how much they don't know before they'll pay.

**Decision:** Four-stage value architecture:
1. **Free quiz** — zero barriers, no login, auto-advance UX
2. **Email gate** — captures lead data and syncs 8 CRM tags before showing any results (lead is captured regardless of purchase)
3. **Preview with blur** — shows the avatar type, basic teaser, and score bars for free. Premium sections (blind spots, cost calculator, protocol, coaching CTA) are visible but blurred behind a gradient overlay. The user can see the *structure* of what they're missing.
4. **Paid report** — full interactive experience with PDF download

**Why this works architecturally:** The blur/lock pattern isn't just a paywall — it's a product demo. The user sees section headers, layout structure, and locked checkboxes. The perceived value gap between what they got for free and what they'd get for the price drives conversion without requiring sales copy to do the heavy lifting.

**CRM integration point:** By the time a user reaches the preview, GHL already has their personality type, 4 dimension scores, primary weakness, and UTM source. Even if they never buy, the client has a fully tagged lead for automated nurture.

### ADR-4: Dual LLM Strategy — Cost Optimization by Task Complexity

**Context:** The platform has two distinct AI workloads with very different requirements:

| Workload | Input Size | Output Complexity | Latency Tolerance | Volume |
|----------|-----------|-------------------|-------------------|--------|
| Quiz analysis | ~20 answers (small) | Structured JSON (insights, traits) | <5s (user waiting) | Every quiz completion |
| Consultation | 50+ Q&A exchanges (large) | Adaptive question selection + plan generation | <3s per question, minutes for plan | Lower volume, higher depth |

**Decision:**
- **GPT-4o-mini** for quiz analysis — fast, cheap ($0.15/1M input tokens), excellent at structured output with Zod schemas. The task is bounded: take verified scores, generate insights. Frontier reasoning isn't needed.
- **Gemini 2.5 Flash** for consultation — better long-context performance for maintaining coherence across 50+ exchange sessions. Lower cost for the token volumes involved. Adaptive question selection requires understanding the full conversation history.

**Fallback strategy:** Both AI paths have graceful degradation. If OpenAI is down, the quiz still works (deterministic scoring is the source of truth, AI insights are supplementary). If Gemini is unavailable, the consultation falls back to rule-based question selection from a 100+ question bank organized into 4 methodology phases.

### ADR-5: Non-Blocking CRM Integration

**Context:** GHL CRM calls are external API requests that can fail (rate limits, downtime, network issues). The user funnel must never break because a CRM call failed.

**Decision:** Every GHL API call is wrapped in a non-blocking try/catch. Failures are logged but invisible to the user. The funnel continues regardless. CRM data sync is important for the client's follow-up workflow but is never on the critical path for user experience.

**Payment recovery corollary:** Because payment redirect can fail (browser tab closed, network drop), I built an email-based recovery flow. The user enters their email on the report page → API checks GHL for buyer tag → if found, unlocks report. This handles the edge case where GHL has the payment record but the user never made it back to the app.

---

## What I Built

### Quiz + Report Funnel (11 API Routes, 16 Pages, 33 Components)

- **20-question behavioral quiz** with auto-advance, hybrid detection, centroid-based 4-dimensional scoring
- **AI-enhanced analysis** — GPT-4o-mini structured output via Zod schemas for mathematical insights and micro-traits
- **Email gate** syncing to Supabase + GHL CRM (8 tags per contact)
- **Preview results** — free content + blurred premium sections with gradient overlay
- **Premium interactive report** — 9 components: animated score bars (tap-to-expand), blind spots diagnostic (checkbox + resonance score), interactive cost calculator (slider), Week 1 protocol (free) + locked Weeks 2-4, coaching comparison grid, tiered CTA with CRM interest tagging, sticky navigation, per-avatar accent colors
- **6-page native vector PDF** via `@react-pdf/renderer` — avatar-specific content, proper page breaks, print-ready
- **Email-based payment recovery** — verifies buyer status against CRM tags
- **Per-avatar theming** — 5 distinct color systems (Red, Blue, Purple, Amber, Green) flowing through all components

### AI Consultation Engine (10 API Routes, 9 Pages, 29 Components)

- **100+ question bank** organized into 4 phases: Exploration, Focused Assessment, Completion, Validation
- **Adaptive questioning** — Gemini 2.5 Flash selects questions based on personality context and conversation history
- **Voice input processing** — speech-to-text with AI interpretation of tone, confidence, and intent
- **Persistent sessions** — Supabase-backed state tracking with completion limits
- **Plan generation** — AI synthesizes answers into a personalized trading plan (uploaded to Supabase Storage)
- **Slide viewer** — metric, quote, bullet, and table layouts for plan presentation
- **Server-side PDF export** — Puppeteer/Chromium for high-fidelity plan documents

### CRM Automation Layer (GoHighLevel)

- **4 trigger points:** email gate (lead + 8 tags), payment (buyer + revenue), coaching purchase (cohort + onboarding), nurture graduation (7-day sequence)
- **13 unique tags** per contact lifecycle
- **Non-blocking architecture** — CRM failures are invisible to users

---

## Content Architecture at Scale

The 5 archetype experiences are driven by a single typed data model. Zero content duplication. TypeScript enforces completeness — missing content fails the build.

| Per Avatar | Count | Total Across 5 |
|-----------|-------|----------------|
| Score meanings | 4 | 20 |
| Blind spots (with real examples) | 5 | 25 |
| Cost calculator framing | 1 | 5 |
| Week 1 protocol tasks | 3 | 15 |
| Locked week teasers | 3 | 15 |
| Coaching CTA bullets | 5 | 25 |
| Honest reality paragraph | 1 | 5 |
| **Unique content elements** | **22** | **110** |

Every content element traces to a specific section of the client's behavioral research. This is domain-expert content structured for product delivery — not AI-generated filler.

### The 5 Trader Archetypes

| Avatar | Color | Protocol | Core Failure Pattern |
|--------|-------|----------|---------------------|
| **Reactor** | Red | "Stabilize" | Emotional execution collapse under loss pressure |
| **Analyzer** | Blue | "Awareness" | Analysis paralysis — seeks certainty that never arrives |
| **Strategist** | Purple | "Freeze" | Over-optimization — changes system before gathering enough data |
| **Opportunist** | Amber | "Gate" | Uncontained instinct — speed without structure |
| **Maverick** | Green | "Contain" | No structural limits — conviction without containment |

---

## Challenges & Solutions

| Challenge | Architectural Solution |
|-----------|----------------------|
| Personality classification must be deterministic but insights must feel personal | Hybrid intelligence: centroid-based scoring for classification, LLM for insight generation — AI never determines the type |
| 5 avatars needing distinct content without 5x maintenance burden | Typed data model + shared component tree — one codebase, 5 experiences, TypeScript enforces completeness |
| Payment redirects fail ~5% of the time (closed tabs, network) | Email-based recovery verifying buyer status against CRM tags — decouples payment confirmation from redirect success |
| Quiz funnel needs zero friction but CRM needs maximum data | Email gate positioned after quiz completion but before results — lead is captured at peak curiosity |
| AI consultation must replicate a specific expert methodology | 100+ question bank organized into 4 phases matching the expert's structure, with Gemini adapting selection per conversation |
| CRM integration cannot be a single point of failure | Non-blocking try/catch on all CRM calls — user funnel is decoupled from CRM availability |
| PDF rendering in `@react-pdf/renderer` has different flexbox behavior | Traced layout issues through nested flex containers — `alignItems: 'center'` on grandparent containers silently shrinks children |

---

## Scale & Metrics

### Codebase

| Metric | Report Funnel | Consultation Engine | Combined |
|--------|--------------|-------------------|----------|
| API routes | 11 | 10 | **21** |
| Page routes | 16 | 9 | **25** |
| Components | 33 | 29 | **62** |
| Question bank | 20 (quiz) | 100+ (consultation) | **120+** |
| CRM tags per contact | 13 | — | **13** |
| Unique content elements | 110 | — | **110** |

### Architecture

| Metric | Detail |
|--------|--------|
| AI Models | 2 (GPT-4o-mini + Gemini 2.5 Flash) |
| AI fallback paths | 2 (rule-based scoring, rule-based question selection) |
| CRM trigger points | 4 |
| Payment paths | 3 (GHL primary, Stripe verification, email recovery) |
| Archetype experiences | 5 (from 1 codebase) |
| PDF pages per report | 6 (native vector, not screenshot-based) |

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript, React 18 |
| Database | Supabase (PostgreSQL + Storage) |
| AI | OpenAI GPT-4o-mini, Google Gemini 2.5 Flash (Vercel AI SDK) |
| Payments | GHL Payment Links, Stripe (verification fallback) |
| CRM | GoHighLevel — 4 trigger points, 13 tags, non-blocking |
| PDF | @react-pdf/renderer (reports), Puppeteer/Chromium (plans) |
| UI | Tailwind CSS, Radix UI, Framer Motion, Recharts |
| Voice | Speech-to-text with AI tone/intent interpretation |
| Validation | Zod schemas (API input + AI structured output) |
| Testing | Playwright |
| Hosting | Vercel (serverless) |

---

## Key Takeaways

1. **Hybrid intelligence architecture** — Deterministic scoring for classification reliability, LLMs for insight richness. The AI enhances the product but never controls the critical path. This pattern applies to any domain where classification accuracy matters more than generative creativity.

2. **Content architecture as a scaling strategy** — A typed data model turns domain expertise into a product system. 110 unique content elements across 5 archetypes, driven by 9 shared components and enforced by TypeScript. Adding a new archetype is a data entry, not a codebase fork.

3. **Revenue funnel as architecture** — The blur/lock preview pattern, the email gate positioning, and the CRM tag strategy are architectural decisions that directly enable business outcomes. The technical design and the conversion strategy are the same thing.

4. **Dual LLM cost optimization** — Different AI workloads have different cost/quality/latency profiles. Matching the right model to the right task (GPT-4o-mini for fast structured output, Gemini Flash for long-context adaptive sessions) reduces cost without sacrificing quality.

5. **Non-blocking integration philosophy** — External services (CRM, payment processors) are never on the critical user path. Failures are silent. Recovery flows handle edge cases. The user funnel is architecturally decoupled from third-party availability.

6. **Research-to-product pipeline** — Translating a 16-page behavioral study into an interactive product isn't a content task — it's an architecture task. The data model, the component contracts, and the type system are what make domain expertise deliverable at scale.

---

*Last updated: April 10, 2026*
*Author: Nic Chin (@nicuk)*
