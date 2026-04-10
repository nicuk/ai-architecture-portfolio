# ADR-004: Immutable Audit Architecture for Regulated Industries

**Status**: Accepted  
**Date**: 2026  
**Context**: Pharma SaaS Platform — Production Readiness Engagement (see [case study](../case-studies/production-readiness-pharma-saas.md))

---

## Context

Enterprise clients in regulated industries require audit trails that meet or approach SOC 2 Type II, ISO 27001, and industry-specific compliance standards. The core requirements:

1. **Immutability** — Logs cannot be modified or deleted by any application role
2. **Traceability** — Every action traceable to a specific user, tenant, and request
3. **AI/LLM accountability** — Every model call traceable with input/output tokens, latency, and the triggering user
4. **Retention** — Minimum 365-day retention with queryable access
5. **Cost** — No additional paid tooling without approval

### The Decision Space

```
┌─────────────────────────────────────────────────────────────┐
│              AUDIT ARCHITECTURE OPTIONS                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Option A: Application Database Tables                       │
│     + Easy to query                                          │
│     - Mutable (DELETE/UPDATE possible)                       │
│     - Same DB = same admin credentials                       │
│     - Fails immutability requirement                         │
│                                                              │
│  Option B: Blockchain / Custom Append-Only Store             │
│     + Cryptographic immutability                             │
│     - Massive over-engineering for the problem               │
│     - Operational complexity, cost                           │
│                                                              │
│  Option C: CloudWatch Logs ← SELECTED                       │
│     + Inherently append-only (no delete API)                 │
│     + Already available via AWS stack                        │
│     + Configurable retention (365 days)                      │
│     + Queryable via CloudWatch Insights                      │
│     + Zero marginal cost                                     │
│     + Application IAM roles cannot delete logs               │
│                                                              │
│  Option D: Third-Party SIEM (Splunk, Datadog)               │
│     + Feature-rich                                           │
│     - $5K-15K/month at production volume                     │
│     - Budget constraints                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Decision

**Use CloudWatch Logs as the immutable audit store, with a structured AuditLogger service and RequestContextMiddleware for distributed traceability.**

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              AUDIT TRAIL ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Incoming Request                                            │
│       │                                                      │
│       ▼                                                      │
│  ┌──────────────────────┐                                   │
│  │ RequestContext        │  Generates X-Request-ID           │
│  │ Middleware            │  Attaches to thread-local         │
│  └──────────┬───────────┘                                   │
│             │                                                │
│       ┌─────┼──────────┐                                    │
│       ▼     ▼          ▼                                    │
│  ┌────────┐ ┌────────┐ ┌────────┐                          │
│  │  Auth  │ │  CRUD  │ │  AI    │                          │
│  │ Events │ │ Events │ │ Events │                          │
│  └───┬────┘ └───┬────┘ └───┬────┘                          │
│      └──────────┼──────────┘                                │
│                 ▼                                            │
│  ┌──────────────────────┐                                   │
│  │    AuditLogger       │  Standardized fields:             │
│  │    Service           │  timestamp, actor, tenant,        │
│  │                      │  action, object_id, outcome,      │
│  │                      │  request_id, ip_address           │
│  └──────────┬───────────┘                                   │
│             │                                                │
│             ▼                                                │
│  ┌──────────────────────┐                                   │
│  │  CloudWatch Logs     │  Append-only                      │
│  │  (365-day retention) │  No delete API for app roles      │
│  │                      │  Queryable via Insights            │
│  └──────────────────────┘                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Audited Actions (12+)

```
Authentication:  LOGIN_SUCCESS, LOGIN_FAILURE, LOGOUT,
                 TOKEN_REFRESH, PASSWORD_CHANGE

Resources:       CREATE, UPDATE, DELETE, DOWNLOAD

AI / LLM:       MODEL_QUERY, ANALYSIS_CREATE, ANALYSIS_UPDATE
```

### AI/LLM Call Tracing

Every LLM invocation logs:

```
{
  "action": "LLM_CALL",
  "model": "<model_name>",
  "tenant_id": "tenant-123",
  "user_id": "user-456",
  "request_id": "req-789",
  "tokens_input": 1250,
  "tokens_output": 340,
  "latency_ms": 2100,
  "guardrail_triggered": false,
  "timestamp": "2026-01-15T14:30:00Z"
}
```

---

## Implementation Details

### RequestContextMiddleware

Injects a unique `X-Request-ID` into every request. If the client sends one (from a load balancer or API gateway), it's preserved. Otherwise, a UUID is generated. This ID propagates through every log entry, audit record, and LLM call trace — enabling full request reconstruction.

### AuditLogger Service

Centralized logging service with:
- **Standardized schema** — Every event has the same field structure
- **Automatic enrichment** — User, tenant, IP, and request_id pulled from request context
- **Dual output** — Logs to both CloudWatch (immutable) and application database (queryable for dashboards)
- **Failure isolation** — Audit logging failures don't block the primary operation

### Guardrail Integration

Input and output guardrails log independently:
- **InputGuardrail**: Instruction override, role manipulation, jailbreak, data exfiltration detection
- **OutputGuardrail**: PII detection, credential scanning
- Each guardrail event includes the triggering pattern, confidence score, and action taken (blocked vs. logged)

---

## Consequences

### Positive

1. **Immutability proven** — CloudWatch Logs have no delete API for application IAM roles
2. **Zero marginal cost** — Already within the AWS stack
3. **Full traceability** — Any action can be traced to user → tenant → request → LLM call
4. **Compliance foundation** — Supports SOC 2, ISO 27001, and industry-specific audit requirements
5. **365-day queryable retention** — Meets most regulatory retention requirements

### Negative

1. **CloudWatch Insights queries are slower** than dedicated SIEM tools for complex analysis
2. **No real-time alerting on audit events** without additional metric filters (implemented separately)
3. **Cross-region DR** would require additional log replication configuration
4. **Vendor lock-in** to AWS CloudWatch for the audit store

### Trade-offs Accepted

- Chose pragmatic compliance over perfect compliance — this is a foundation, not a certification claim
- CloudWatch query limitations accepted as reasonable for pilot-phase volume
- Metric filters added for the most critical event patterns to bridge the alerting gap

---

## Results

| Metric | Value |
|--------|-------|
| Audited actions | 12+ critical actions |
| Retention | 365 days (configurable) |
| Immutability | Verified — no delete/update API for app roles |
| Request traceability | 100% — every action linked via X-Request-ID |
| LLM call traceability | 100% — tenant, user, request, tokens, latency |
| Additional cost | $0 (CloudWatch already in stack) |

---

## Alternatives Considered

### Alternative 1: PostgreSQL Audit Table
- **Rejected**: Application database credentials can UPDATE/DELETE rows — fails immutability requirement

### Alternative 2: Custom Blockchain Audit Log
- **Rejected**: Massive over-engineering; CloudWatch provides equivalent immutability guarantees for this context

### Alternative 3: Datadog / Splunk
- **Rejected**: $5K-15K/month; budget constraints at pilot stage

### Alternative 4: S3 + Athena
- **Deferred**: Better for long-term archival (years); consider when volume exceeds CloudWatch cost-efficiency threshold
