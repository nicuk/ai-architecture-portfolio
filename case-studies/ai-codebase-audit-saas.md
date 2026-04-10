# AI Codebase Audit SaaS (SystemAudit.dev)

> 3-minute AI-powered code audits replacing 2-week manual review cycles — live at [systemaudit.dev](https://systemaudit.dev)

**Role**: Sole Architect & Developer  
**Domain**: Developer Tools / Technical Due Diligence  
**Outcome**: Production SaaS — deterministic-first analysis engine with multi-pass AI, serving founders, CTOs, and investors

---

## The Problem

Non-technical decision-makers are flying blind on their own codebases:

- **Founders** hire developers or use AI tools (Cursor, v0, Bolt) — they have no idea if the output is solid or a house of cards
- **Investors** are writing checks with zero visibility into technical risk — no way to know if the code is a liability
- **CTOs** inherit codebases and need fast answers — not a 2-week audit from a consultant at $15K+
- **"Vibe coders"** ship fast with AI — but can't tell if they've built something maintainable or a ticking time bomb

Existing options:
- **Manual audits**: 2–4 weeks, $10K–$50K, require deep engagement
- **Static analysis tools**: Built for developers, not decision-makers — SonarQube doesn't explain risk in money terms
- **Consultants**: Expensive, slow, subjective — different auditors, different conclusions

The market needed code audits that speak business, not bytecode.

---

## My Solution

Built a full-stack SaaS that takes a GitHub URL and returns a plain-English audit report in under 3 minutes — with deterministic scanning that costs $0 per free scan, and multi-pass AI analysis only when users pay.

### Architecture Overview

```mermaid
graph TB
    subgraph Input Layer
        URL[GitHub URL]
        OAUTH[GitHub OAuth]
        TREE[Repo Tree Fetch]
    end

    subgraph Deterministic Engine
        SCAN[9-Ecosystem Scanner]
        SECRETS[Secrets Scanner]
        HEALTH[Health Score Engine]
        IMPORT[Import Graph Analyzer]
    end

    subgraph AI Pipeline — Pro Tier
        FACTS[Fact Extractor]
        PASS1[Pass 1: Architecture — t=0.4]
        PASS2[Pass 2: Risks — t=0.1]
        PASS3[Pass 3: Business Translation — t=0.5]
        ENFORCE[Fact Enforcement]
    end

    subgraph Output Layer
        REPORT[Interactive Report]
        MAP[D3 System Map]
        PDF[PDF Export]
        CHAT[AI Chat Assistant]
    end

    URL --> TREE
    OAUTH --> TREE
    TREE --> SCAN
    SCAN --> SECRETS
    SCAN --> HEALTH
    SCAN --> IMPORT
    HEALTH --> REPORT

    SCAN --> FACTS
    FACTS --> PASS1
    PASS1 --> PASS2
    PASS1 --> PASS3
    PASS2 --> ENFORCE
    PASS3 --> ENFORCE
    ENFORCE --> REPORT
    REPORT --> MAP
    REPORT --> PDF
    REPORT --> CHAT
```

### Key Technical Decisions

1. **Deterministic-First Architecture**
   - Free tier runs zero AI — pure algorithmic analysis across 50+ languages
   - $0 marginal cost per free scan; AI costs only incurred on paid tier
   - Deterministic results are consistent and instant — same repo, same score, every time
   - Converts to paid via progressive disclosure: free → email unlock → Pro

2. **Multi-Pass AI with Task-Specific Parameters**
   - 3 separate LLM passes, each with tuned temperature and output limits
   - Pass 1 (Architecture, t=0.4, 12K tokens): high-level structure, tech debt, AI readiness
   - Pass 2 (Risks, t=0.1, 6K tokens): low temperature for precise, conservative risk identification
   - Pass 3 (Business Translation, t=0.5, 5K tokens): higher creativity for plain-English explanations
   - Passes 2 and 3 run in parallel where safe — shaves ~30% off total analysis time

3. **Post-AI Fact Enforcement**
   - Every AI-generated claim is cross-checked against deterministic scan data
   - TypeScript strict mode claims verified against actual `tsconfig.json`
   - Test coverage assertions validated against real test file counts
   - LOC-based evidence corrected when AI drifts >20% from measured values
   - Cost estimates capped against engineering benchmarks ($5K/$8K/$3K by priority tier)

4. **Automated Quality Regression (Eval Suite)**
   - 10-repo corpus spanning small TypeScript to 250K+ line monorepos
   - Grades on 4 dimensions: deterministic accuracy, free tier reliability, full analysis quality, business translation clarity
   - Runs before every change to prevent quality drift — 10/10 score maintained

---

## Technical Implementation

### Deterministic Scan Pipeline

```
GitHub URL
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Fetch Tree │────▶│  Estimate   │────▶│  Smart File │
│  + Languages│     │  Source LOC │     │  Selection  │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
                    ┌─────────────────────────┤
                    │                         │
                    ▼                         ▼
          ┌─────────────────┐     ┌─────────────────────┐
          │  9 Ecosystem    │     │  Security & Secrets  │
          │  Parsers        │     │  Scanner             │
          │                 │     │                      │
          │  · npm/yarn     │     │  · Regex patterns    │
          │  · pip/poetry   │     │  · Env file detect   │
          │  · Go modules   │     │  · AWS key patterns  │
          │  · Cargo/Rust   │     │  · Token/PEM detect  │
          │  · Gemfile/Ruby │     │  · False positive     │
          │  · Maven/Gradle │     │    filtering         │
          │  · Composer/PHP │     │  · Confidence scoring│
          │  · .NET/csproj  │     │                      │
          │  · setup.py     │     │                      │
          └────────┬────────┘     └──────────┬──────────┘
                   │                         │
                   └────────┬────────────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Health Score       │
                 │  Engine             │
                 │                     │
                 │  20+ weighted       │
                 │  signals → 0-100    │
                 │                     │
                 │  Sonar-style        │
                 │  ceiling from       │
                 │  critical findings  │
                 └─────────────────────┘
```

### Health Score Computation

The scoring engine combines weighted quality signals with a ceiling mechanism that prevents inflated scores when critical issues exist:

```
┌──────────────────────────────────────────────────────────┐
│              HEALTH SCORE ENGINE (0–100)                  │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  Weighted Signals:                                        │
│  ┌──────────────────────────────┬────────┐               │
│  │  Exposed secrets             │  10 pts │  ← highest   │
│  │  Security layer present      │   7 pts │               │
│  │  Test suite exists           │   7 pts │               │
│  │  CI/CD configured            │   7 pts │               │
│  │  Type safety (eco-adjusted)  │  3-7pts │               │
│  │  Deploy pipeline             │   5 pts │               │
│  │  Complexity distribution     │   5 pts │               │
│  │  Dependency count            │   5 pts │               │
│  │  Linter / Readme / License   │  2 each │               │
│  └──────────────────────────────┴────────┘               │
│                                                           │
│  Ceiling Override:                                        │
│  ┌────────────────────────────────────────┐              │
│  │  ≥3 critical findings → max score 30  │              │
│  │  2 critical           → max score 40  │              │
│  │  1 critical           → max score 55  │              │
│  │  ≥3 high              → max score 65  │              │
│  │  1 high               → max score 85  │              │
│  └────────────────────────────────────────┘              │
│                                                           │
│  Final = max(10, min(ceiling, postureScore))              │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Multi-Pass AI Pipeline

```
                   Deterministic Scan + File Contents
                              │
                              ▼
                   ┌─────────────────────┐
                   │  Fact Extractor     │
                   │                     │
                   │  · tsconfig parse   │
                   │  · test file census │
                   │  · import graph     │
                   │  · security patterns│
                   │  · component LOC    │
                   │  · dep health       │
                   │  · structure score  │
                   └──────────┬──────────┘
                              │
                              ▼
              ┌───────────────────────────────────┐
              │  PASS 1 — Architecture            │
              │  Claude Sonnet 4.6 · t=0.4        │
              │  maxTokens: 12,000                │
              │                                   │
              │  → exec summary, components,      │
              │    tech debt, AI readiness,        │
              │    mermaid diagram, recommendations│
              └───────────────┬───────────────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
  ┌────────────────────────┐   ┌────────────────────────┐
  │  PASS 2 — Risks       │   │  PASS 3 — Business     │
  │  t=0.1 (conservative) │   │  t=0.5 (creative)      │
  │  maxTokens: 6,000     │   │  maxTokens: 5,000      │
  │                        │   │                        │
  │  → risk list, severity,│   │  → plain-English risk, │
  │    security issues,    │   │    cost estimates,      │
  │    confidence scoring  │   │    action plan,         │
  │                        │   │    "what it means"      │
  └───────────┬────────────┘   └───────────┬────────────┘
               │                            │
               └──────────┬─────────────────┘
                          │
                          ▼
              ┌───────────────────────────────────┐
              │  FACT ENFORCEMENT                  │
              │                                   │
              │  · Cross-check claims vs code     │
              │  · Cap cost estimates ($10K max)  │
              │  · Validate risk confidence       │
              │  · Strip speculative revenue      │
              │  · ORM/SQLi heuristic correction  │
              │  · localStorage risk downgrade    │
              └───────────────────────────────────┘
```

### Monetization Architecture

```
                           ┌──────────────────┐
                           │   GitHub URL      │
                           │   (User Input)    │
                           └────────┬─────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │   FREE SCAN      │  │  EMAIL UNLOCK    │  │   PRO REPORT     │
   │                  │  │                  │  │                  │
   │  Health score    │  │  Full risk list  │  │  AI deep-dive    │
   │  Risk summary    │  │  Cost estimates  │  │  Business risks  │
   │  Security scan   │  │  Missing guards  │  │  Action plan     │
   │  Feature verify  │  │  Strengths       │  │  System map      │
   │  Architecture    │  │  Health checklist│  │  PDF export      │
   │  Quality grade   │  │                  │  │  AI chat         │
   │                  │  │                  │  │                  │
   │  Cost: $0        │  │  Cost: $0        │  │  $49 – $199      │
   │  AI: None        │  │  AI: None        │  │  (by LOC tier)   │
   └──────────────────┘  └──────────────────┘  └──────────────────┘
                                                       │
                              ┌─────────────────────────┤
                              │                         │
                              ▼                         ▼
                   ┌──────────────────┐     ┌──────────────────┐
                   │  Stripe Checkout │     │  Dynamic Pricing │
                   │                  │     │                  │
                   │  Session create  │     │  ≤30K LOC → $49  │
                   │  Webhook verify  │     │  ≤75K LOC → $99  │
                   │  HMAC session    │     │  ≤150K LOC→ $199 │
                   │  cookie auth     │     │  150K+    → Talk │
                   └──────────────────┘     └──────────────────┘
```

---

## Results

| Metric | Traditional Audit | SystemAudit.dev | Improvement |
|--------|-------------------|-----------------|-------------|
| Time to results | 2–4 weeks | < 3 minutes | ~99.9% faster |
| Cost (small project) | $10K–$15K | $49 | 99.5% cheaper |
| Languages supported | 2–3 per auditor | 50+ detected, 9 deep | 15x+ coverage |
| Consistency | Varies by auditor | Deterministic + fact-checked AI | Repeatable |
| Accessibility | Requires developer | Plain-English for non-technical | Universal |

### Technical Outcomes

- **10/10 eval score** on automated quality regression across diverse real-world repos
- **$0 marginal cost** for free tier — deterministic engine has zero AI spend
- **Anti-hallucination pipeline** — every AI finding fact-checked against scanned code
- **9 ecosystem parsers** — deep framework-aware analysis (Next.js, Django, Spring Boot, Rails, Laravel, Go, Rust, .NET, Flask)
- **Dynamic pricing** — project size detected automatically via hybrid LOC estimation

### Business Outcomes

- **Live SaaS** at [systemaudit.dev](https://systemaudit.dev) — fully operational with payments, email capture, analytics
- **Progressive monetization** — free scan hooks users, email unlock builds pipeline, Pro converts
- **3 pricing tiers** ($49/$99/$199) auto-selected by codebase size
- **Enterprise tier** for 150K+ LOC codebases with custom pricing

---

## Key Learnings

1. **Deterministic-first saves everything** — Running AI only when users pay means the free tier scales infinitely at zero cost. This is the single most important architecture decision in the product — it separates "freemium with crushing API bills" from "freemium that actually works"

2. **Temperature is a design parameter, not a default** — Different LLM passes need different temperatures. Risks at t=0.1 prevents the AI from inventing problems. Business copy at t=0.5 lets it write naturally. Treating temperature as task-specific produced dramatically better output than a single setting

3. **Fact enforcement is table stakes for production AI** — LLMs hallucinate. In a product that sells trust, every claim must be verifiable. Cross-checking AI output against deterministic data caught and corrected real inaccuracies — the kind that would destroy credibility with a paying customer

4. **Eval suites prevent silent regression** — When the product's value is "accurate analysis," you need automated quality testing that runs before every change. Without the 10-repo eval corpus, prompt changes that improved one case would silently break three others

5. **Speak business, not code** — The entire product thesis is that technical people already have linters. The gap is translating code health into dollars, timelines, and risk — language that founders and investors actually use to make decisions

---

## Tech Stack

- **Frontend**: Next.js 16 (App Router, Server Components), React 19, Tailwind CSS 4
- **AI Engine**: Claude Sonnet 4.6 (Anthropic) via Vercel AI Gateway — multi-pass with fact enforcement
- **Database**: PostgreSQL (Supabase) — reports, purchases, scans, GitHub OAuth tokens
- **Payments**: Stripe (Checkout Sessions, Webhooks, dynamic pricing by LOC)
- **Auth**: HMAC-SHA256 session cookies (Pro), GitHub OAuth (private repos), admin bypass
- **Visualization**: D3 force graphs, XY Flow, Mermaid — interactive system maps
- **Infrastructure**: Vercel (serverless), Upstash Redis (rate limiting), GitHub Actions CI
- **Quality**: Vitest, automated eval suite (10-repo corpus), repo boundary enforcement

