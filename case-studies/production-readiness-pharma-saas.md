# Production Readiness & Security Hardening — Pharma SaaS Platform

> 24/24 acceptance criteria passed, enterprise pilot approved in 7 weeks

**Role**: Fractional CTO / Security & Infrastructure Architect  
**Domain**: Pharmaceutical / Biotech SaaS  
**Outcome**: Platform approved for enterprise pilot deployment with pharma clients

---

## The Problem

A pharmaceutical SaaS platform had a **functional product** built on Django/React/Kubernetes — but it was **not production-ready for enterprise clients**. With pilot deployments approaching for large pharma organizations, critical gaps existed:

- **Security** — JWT tokens valid for 7 days, cross-tenant data access vulnerabilities, exposed API docs in production
- **Observability** — No centralized logging, no alerting, no health checks — production incidents would go undetected
- **Compliance** — No audit trail, no AI/LLM traceability, no path toward SOC 2, ISO 27001, or FDA 21 CFR Part 11
- **Infrastructure** — Kubernetes pods running as root, no pod security contexts, no RBAC documentation
- **Operational Readiness** — No load testing, no backup/restore procedures, no disaster recovery validation

A security audit failure or production incident during the pilot could end the enterprise engagement before it began.

---

## My Solution

Designed and executed a **milestone-based engagement** with objective, evidence-based acceptance criteria — transforming the platform from "functional prototype" to "enterprise-defensible" in 7 weeks across 4 milestones.

### Engagement Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ENGAGEMENT DELIVERY MODEL                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Assessment                                                          │
│       │                                                              │
│       ▼                                                              │
│  ┌──────────────┐                                                   │
│  │  Technical   │  Scored 7.5/10 — strong foundation,               │
│  │  Assessment  │  configuration-level gaps (not architectural)     │
│  └──────┬───────┘                                                   │
│         │                                                            │
│    ┌────┴────┬──────────────┬──────────────┐                        │
│    ▼         ▼              ▼              ▼                        │
│  Week 1-2  Week 3-4      Week 5-6      Week 7                     │
│  Security  Infrastructure  Audit +      Load Test +                │
│  + Observ  Hardening       AI Security  DR Validation              │
│    │         │              │              │                        │
│    ▼         ▼              ▼              ▼                        │
│  4/4 ✅    5/5 ✅         8/8 ✅        7/7 ✅                    │
│  PASSED     PASSED         PASSED        PASSED                    │
│                                                                      │
│  Delivery: PR-Only → Evidence-Based Acceptance → 14-Day Warranty    │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Technical Decisions

1. **Entity-Based Tenant Isolation (Not Row-Level Security)**
   - Existing architecture used entity-based filtering
   - Hardened the existing pattern rather than a risky architectural rewrite to PostgreSQL RLS
   - Validated every API endpoint includes entity-scoped queries — pragmatic security over theoretical perfection

2. **CloudWatch Over Third-Party APM**
   - SOW prohibited introducing paid tools without approval
   - Configured log groups, metric filters, and alarms to deliver 80% of APM value at zero marginal cost
   - Right sizing for a startup entering pilot phase

3. **AI Guardrails — Configurable Blocking vs. Logging**
   - Overly aggressive guardrails would degrade UX with false positives
   - Configurable system: blocking mode (hard reject) or logging mode (flag but allow)
   - Lets the team tune sensitivity as they gather production data

4. **Append-Only Audit Logs via CloudWatch**
   - Pharmaceutical compliance (21 CFR Part 11) demands immutable audit logs
   - CloudWatch Logs are inherently append-only with no delete API for application roles
   - Cryptographic immutability without building a custom audit system

---

## Technical Implementation

### Milestone 1: Application Security & Observability

```
┌─────────────────────────────────────────────────────────────────┐
│              SECURITY ASSESSMENT (26 CHECKS)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Critical Vulnerabilities Fixed:                                 │
│                                                                  │
│  CRIT-001: JWT ACCESS_TOKEN_LIFETIME = 7 days                   │
│     → Reduced to 8 hours (access), 24 hours (refresh)           │
│     → Aligns with single work shift — pragmatic + defensible    │
│                                                                  │
│  CRIT-002: API endpoint missing tenant filter                    │
│     → Cross-tenant data access (Tenant A reads Tenant B docs)   │
│     → Added entity-scoped query filters on all endpoints        │
│                                                                  │
│  + 4 High, 3 Medium vulnerabilities resolved                    │
│                                                                  │
│  Observability Deployed:                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ 8 Alarms │  │ 4 Log    │  │ 2 SNS    │  │ Health + │       │
│  │ (5 sec + │  │ Groups   │  │ Notif.   │  │ Ready   │       │
│  │  3 infra)│  │ (12-mo)  │  │ Subs     │  │ Probes  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### Milestone 2: Infrastructure Hardening

Hardened **6 Kubernetes deployments** (gunicorn, react, worker, scheduler, cache, queue):

```
Pod Security Context Applied:
  ├── runAsNonRoot: true
  ├── runAsUser: 100
  ├── fsGroup: 100
  ├── capabilities: drop: ["ALL"]
  ├── allowPrivilegeEscalation: false
  └── seccomp: RuntimeDefault

Scanning Results:
  ├── Trivy: 0 Critical container vulnerabilities
  ├── Checkov: 115 findings → ~56 fixed via pod security
  └── RBAC: 2 admin accounts only, least privilege validated

Encryption Verified:
  ├── S3: SSE-S3 on 2 buckets
  ├── RDS: Encryption at rest (AWS managed key)
  └── TLS: Let's Encrypt certificate verified
```

### Milestone 3: Audit Trail + AI/LLM Security

```
┌─────────────────────────────────────────────────────────────────┐
│              ENTERPRISE AUDIT TRAIL (12+ ACTIONS)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Authentication: LOGIN_SUCCESS, LOGIN_FAILURE, LOGOUT,           │
│                  TOKEN_REFRESH, PASSWORD_CHANGE                   │
│  Documents:      DOCUMENT_CREATE, UPDATE, DELETE, DOWNLOAD       │
│  AI/Analysis:    RAG_QUERY, RCA_CREATE, RCA_UPDATE               │
│                                                                  │
│  + RequestContextMiddleware (X-Request-ID propagation)           │
│  + AuditLogger service with standardized fields                  │
│  + 365-day retention, append-only immutability                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│              AI/LLM SECURITY PACKAGE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CallTrace Extended:                                             │
│     entity, user, request_id, latency_ms,                        │
│     tokens_input, tokens_output                                  │
│                                                                  │
│  InputGuardrail:                                                 │
│     ├── Instruction override detection                           │
│     ├── Role manipulation detection                              │
│     ├── Jailbreak attempt detection                              │
│     └── Data exfiltration pattern detection                      │
│                                                                  │
│  OutputGuardrail:                                                │
│     ├── PII detection and filtering                              │
│     └── Credential/secret scanning                               │
│                                                                  │
│  15+ test cases validating guardrail effectiveness               │
└─────────────────────────────────────────────────────────────────┘
```

### Milestone 4: Load Testing & DR Validation

```
Load Test Results (Custom Python Harness):
┌──────────────┬───────────┬───────────┬────────────┐
│    Metric    │  1 User   │  5 Users  │  10 Users  │
│              │ (Baseline)│ (Target)  │ (Stretch)  │
├──────────────┼───────────┼───────────┼────────────┤
│ Error Rate   │    0%     │  0.98% ✅ │   4.24%    │
│ P95 Latency  │   17.5s   │  28.9s ✅ │   34.6s    │
│ Throughput   │     —     │  31/min ✅│   59/min   │
└──────────────┴───────────┴───────────┴────────────┘

HPA Configuration:
  Gunicorn:  2-5 replicas (CPU 70%, Memory 80%)
  React:     1-5 replicas (CPU 60%, Memory 75%)
  Worker:    2-5 replicas (CPU 70%, Memory 80%)
  Cluster:   2-4 nodes (t3a.medium)

DR Validation:
  Backup RTO:   ~45 min (target: <1 hour) ✅
  Rollback RTO: <2 min ✅
  RPO:          <24 hours ✅
  Database PITR: 5-min max data loss
  S3 Versioning: 0 data loss

10 Operational Runbooks Delivered
```

---

## Results

| Metric | Value |
|--------|-------|
| Acceptance Criteria | 24/24 (100% pass rate) |
| Critical Vulnerabilities Fixed | 2/2 (100%) |
| High Vulnerabilities Fixed | 4/4 (100%) |
| Security Checks Performed | 26 pass/fail checks |
| Deployments Hardened | 6/6 Kubernetes deployments |
| Audit Actions Implemented | 12+ with full traceability |
| CloudWatch Alarms | 8 configured and verified |
| Guardrail Test Cases | 15+ prompt injection + output tests |
| Runbooks Delivered | 10 operational runbooks |
| Load Test Error Rate | 0.98% at target load (SOW: <1%) |
| Backup RTO | ~45 minutes (target: <1 hour) |
| Rollback RTO | <2 minutes |
| Engagement Duration | 7 weeks |

### Compliance Readiness Achieved

| Framework | Status |
|-----------|--------|
| **SOC 2 Type II** | Technical controls implemented — ready for formal audit prep |
| **ISO 27001** | Technical controls implemented — ready for formal audit prep |
| **FDA 21 CFR Part 11** | Foundation in place (audit trail, immutability, traceability) |

### Business Outcome
The platform went from "functional but not production-ready" to **approved for enterprise pilot deployment** with pharmaceutical clients — with every claim backed by evidence.

---

## Architecture Principles Applied

1. **Harden what exists, don't rewrite** — The platform had a solid foundation. Validating and hardening the existing architecture got the client production-ready in 7 weeks, not 7 months.

2. **Configurable enforcement beats hard blocking** — AI guardrails that can switch between blocking and logging modes let the team tune sensitivity with real production data rather than guessing thresholds upfront.

3. **Compliance is architecture, not paperwork** — Immutable audit logs, request traceability, and tenant isolation are architectural decisions that must be built into the system, not bolted on.

---

## Tech Stack

- **Frontend**: React 19, TypeScript, Vite, Redux Toolkit, TailwindCSS, HeroUI, Framer Motion
- **Backend**: Django 5.2, Django REST Framework, Python 3.13, Gunicorn, Celery 5.5, Redis
- **AI/LLM**: Google Gemini (generation), OpenAI text-embedding-3-large (1536-dim), pgvector, Pinecone
- **Database**: PostgreSQL + pgvector, Redis (cache + queue)
- **Infrastructure**: Docker, Kubernetes (EKS), AWS (ECR, S3, RDS, CloudWatch, Route53, ACM), NGINX Ingress, Helm
- **Security Tooling**: Trivy (container scanning), Checkov (IaC scanning), OWASP methodology
- **Monitoring**: CloudWatch (logs, metrics, alarms), SNS notifications, Flower (Celery monitoring)
