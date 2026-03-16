# ADR-005: IoT Ingestion Pipeline Design

**Status**: Accepted
**Date**: 2025–2026
**Context**: B2B SaaS Platform — Real-time IoT Meter Monitoring

---

## Context

A B2B SaaS platform needs to ingest telemetry data from physical IoT meters (heat cost allocators, water meters, heat meters, electricity meters, smoke detectors) across multiple buildings. The data arrives through two channels:

1. **Real-time** — MQTT telegrams from IoT gateways broadcasting wireless meter protocol data
2. **Batch** — CSV file uploads from property managers using various manufacturer formats

### The Challenges

```
┌─────────────────────────────────────────────────────────────┐
│              INGESTION CHALLENGES                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Multiple protocols                                       │
│     MQTT (real-time) vs CSV (batch) vs future APIs           │
│                                                              │
│  2. Multiple manufacturers                                   │
│     Different vendors — different data formats               │
│     Vertical CSV, horizontal CSV, locale-specific decimals   │
│                                                              │
│  3. Cumulative vs consumption readings                       │
│     Meters report cumulative totals                          │
│     Users need consumption per period (deltas)               │
│                                                              │
│  4. Deduplication                                            │
│     Same data can arrive via MQTT and CSV                    │
│     Re-uploads must not create phantom readings              │
│                                                              │
│  5. Device type filtering                                    │
│     Heat cost allocators behave differently than             │
│     direct meters — must be classified in all queries        │
│                                                              │
│  6. Scale progression                                        │
│     Small deployment today → potentially hundreds            │
│     of buildings without rearchitecting                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Decision

**Build a protocol-agnostic ingestion layer with signature-based deduplication, separated from the web application as a Dockerized microservice architecture.**

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              INGESTION PIPELINE                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Physical Meters (5 types)                                   │
│       │                                                      │
│       ▼                                                      │
│  ┌──────────────────┐   ┌──────────────────┐               │
│  │  MQTT Channel    │   │  CSV Channel     │               │
│  │  (Real-time)     │   │  (Batch upload)  │               │
│  │                  │   │                  │               │
│  │  Gateway →       │   │  File upload →   │               │
│  │  Broker →        │   │  Format detect → │               │
│  │  Decode →        │   │  Layout detect → │               │
│  │  Parse           │   │  Decimal handle  │               │
│  └────────┬─────────┘   └────────┬─────────┘               │
│           │                      │                          │
│           └──────────┬───────────┘                          │
│                      ▼                                      │
│           ┌──────────────────┐                              │
│           │  Normalize       │                              │
│           │  (common schema) │                              │
│           └────────┬─────────┘                              │
│                    ▼                                        │
│           ┌──────────────────┐                              │
│           │  Deduplicate     │                              │
│           │  (date+device_id │                              │
│           │   signature)     │                              │
│           └────────┬─────────┘                              │
│                    ▼                                        │
│           ┌──────────────────┐                              │
│           │  Classify        │                              │
│           │  (device type)   │                              │
│           └────────┬─────────┘                              │
│                    ▼                                        │
│           ┌──────────────────┐                              │
│           │  Persist         │                              │
│           │  (PostgreSQL)    │                              │
│           └──────────────────┘                              │
│                                                              │
│  Dashboard reads: consumption deltas, not cumulative         │
│  All queries filter by device type                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

1. **Microservice separation** — The MQTT ingestion service runs as a Dockerized process, independent of the web app. This allows independent scaling and deployment.

2. **Signature-based deduplication** — Each reading is keyed by `date + device_id`. If the same reading arrives via MQTT and then CSV (or via re-upload), it's deduplicated at insert time. This prevented a phantom device counting bug where hundreds of readings appeared as unique devices when only ~44 physical devices existed.

3. **Format-agnostic CSV parsing** — Rather than hardcoding layouts per manufacturer, the parser auto-detects vertical vs. horizontal CSV format and locale-specific decimal conventions.

4. **Consumption deltas at query time** — Raw data is stored as cumulative readings (as meters report them). The dashboard calculates deltas between consecutive readings. This preserves data fidelity while displaying what users actually need.

5. **Device type classification** — Heat cost allocators have different behavioral patterns than direct meters. Classification happens at ingestion, and every dashboard query filters on device type to prevent misleading aggregations.

---

## Implementation Details

### MQTT Ingestion Service (Docker)

Three containers:
- **mqtt-broker** — Mosquitto configuration for IoT gateway connectivity
- **ingestion-service** — Node.js subscriber: receive → decode → parse → persist to PostgreSQL
- **mock-gateway** — Telegram generator for development and integration testing

### CSV Parser (Edge Function)

Handles:
- Vertical CSV (one reading per row) and horizontal CSV (one device per row, dates as columns)
- Locale-specific decimal handling (comma vs. period separators)
- Device type classification across 5 meter categories
- Error notifications on parse failures
- Multi-manufacturer format support

### Notification Engine (3-Group System)

Built on top of the ingestion pipeline:
- **Group 1**: Device error flags and hardware status
- **Group 2**: Signal quality and connectivity issues
- **Group 3**: Consumption anomalies (burst pipe detection, no-data alerts)

---

## Consequences

### Positive

1. **Protocol-agnostic** — Adding a new data source requires only a new channel adapter, not pipeline changes
2. **Deduplication eliminated phantom data** — Fixed a counting bug that inflated device counts by ~10x
3. **Independent scaling** — IoT ingestion scales separately from the web application
4. **Data fidelity preserved** — Raw cumulative readings stored; deltas computed at query time
5. **Testable** — Mock gateway enables full pipeline testing without physical hardware

### Negative

1. **Operational complexity** — Docker containers to manage alongside the serverless web app
2. **Single DB for IoT + web** — May need separation at higher scale
3. **Edge Function cold starts** — CSV parsing has variable latency on first invocation

### Trade-offs Accepted

- Single DB for simplicity at current scale; plan to separate IoT data store at higher volume
- Docker for MQTT vs. managed IoT service (AWS IoT Core) — chose simplicity and cost control for early deployment

---

## Results

| Metric | Value |
|--------|-------|
| Device types supported | 5 (heat allocators, water, heat, electricity, smoke) |
| CSV formats handled | 2 layouts × multiple manufacturers |
| Deduplication accuracy | 100% — phantom devices eliminated |
| Buildings deployed | Multiple (live production) |
| Notification groups | 3 (errors, signal quality, consumption anomalies) |
| Pipeline testability | Full mock gateway for development |

---

## Alternatives Considered

### Alternative 1: AWS IoT Core + Kinesis
- **Rejected**: Over-engineered for initial deployment; significant minimum monthly cost vs. Docker at ~$0

### Alternative 2: Store Deltas Directly
- **Rejected**: Loses data fidelity; if a reading is missed, all subsequent deltas are wrong. Storing cumulative values and computing deltas is self-correcting.

### Alternative 3: Separate Time-Series DB (InfluxDB/TimescaleDB)
- **Deferred**: Justified at scale; current PostgreSQL volume is well within limits
