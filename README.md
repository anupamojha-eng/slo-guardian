# SLO Guardian
**Real-time SLO risk scoring with cloud-native ML inference**

SLO Guardian is a production-oriented system that ingests service telemetry in real time, computes a **risk score for SLO degradation (latency spikes / incident likelihood)**, and emits actionable signals with **operability, cost control, and failure modes** explicitly designed in.

This project is intentionally opinionated: **the system is the product, not the model**.

---

## Why this project exists

Most ML demos stop at model accuracy. In real infrastructure, failures come from **latency, backpressure, cost overruns, and blind spots**, not from marginally better predictions.

This project focuses on:
- Low-latency, reliable inference
- Clear data contracts
- Observability and debuggability
- Explicit tradeoffs and failure handling
- Cloud cost discipline

---
## High-level architecture

1. A **Telemetry Producer** publishes service telemetry events.
2. Events are written to a **Pub/Sub topic (`telemetry-events`)**.
3. A **Risk Scorer (Cloud Run, Java)**:
   - performs feature extraction
   - validates event contracts
   - applies retries and backoff
4. The Risk Scorer calls the **Inference Service (Cloud Run, Java)**, which:
   - loads a versioned model artifact
   - returns a risk score with reason codes
5. The resulting signal is published to **Pub/Sub (`risk-scores`)**.

The architecture emphasizes explicit data contracts, bounded responsibilities per service, and failure isolation between ingestion, scoring, and inference.



Model **training** is performed offline in Python and produces a versioned model artifact consumed by the Java inference service.

---

## Core design decisions

### Python for training, Java for serving

- Training benefits from Python’s ML ecosystem and rapid iteration.
- Serving benefits from Java’s predictable latency, memory behavior, and operational maturity.
- This mirrors real-world production separation of concerns.

### Cloud Run + Pub/Sub

- Autoscaling with **scale-to-zero** keeps cost near zero when idle.
- Managed messaging avoids self-hosted Kafka overhead for this scope.
- IAM, retries, and delivery semantics are treated as first-class concerns.

### Simple model by design

- Baseline models (e.g., logistic regression or gradient boosting) are sufficient to demonstrate:
  - end-to-end flow
  - versioning and rollback
  - drift awareness
- Model sophistication is a **non-goal** for the initial milestone.

---

## Service-level objectives (SLOs)

Design targets:
- **Inference latency:** p95 < 50 ms
- **End-to-end (ingest → risk score):** p95 < 250 ms
- **Availability:** graceful degradation on partial failures

---

## Data contracts

### TelemetryEvent

Represents a single service request observation.

Fields (non-exhaustive):
- `timestamp`
- `service`
- `route`
- `status_code`
- `latency_ms`
- `error`
- `trace_id`
- `env`
- `region`

### RiskScoreEvent

Represents computed SLO risk.

Fields:
- `timestamp`
- `service`
- `route`
- `risk_score` (0.0–1.0)
- `model_version`
- `decision` (OK / WARN / ALERT)
- `reason_codes` (top contributing features)

Reason codes are included explicitly to support **debuggability and trust**.

---

## Observability

Operability is treated as a first-class requirement:
- Structured JSON logs including `trace_id`, `model_version`, and `risk_score`
- Metrics:
  - inference latency
  - error rate
  - prediction distribution
- Alert examples:
  - sustained p95 latency breach
  - anomalous prediction distribution shift

---

## Failure modes and mitigations

| Failure | Mitigation |
|------|-----------|
| Pub/Sub delivery spike | Backoff and concurrency limits |
| Inference service unavailable | Circuit breaker and fallback score |
| Bad model artifact | Version pinning and rollback |
| Schema mismatch | Contract validation and DLQ |
| Traffic surge | Cloud Run autoscaling and load shedding |

Detailed analysis is documented in `architecture/failure-modes.md`.

---

## Cost control

This project is designed to run cheaply:
- Cloud Run scales to zero when idle
- Pub/Sub costs are negligible at low volume
- Logging volume is intentionally constrained

**Expected cost:** ~$5–10/month  
A **$20/month budget alert** is recommended.

---

## Repository layout

## Repository layout

- `architecture/` — Design, tradeoffs, failure analysis  
- `schemas/` — Event contracts  
- `training/` — Python model training and export  
- `services/`
  - `inference-service/` — Java model serving  
  - `risk-scorer/` — Java event processor  
- `gcp/` — Deployment scripts and notes  



---

## Security and secrets

- No credentials, datasets, or trained artifacts are committed.
- Local development uses Application Default Credentials.
- Cloud Run relies on managed service identities.

---

## Non-goals (current phase)

- Automated retraining pipelines
- Advanced feature stores
- Online experimentation / A/B testing
- GKE or self-hosted Kafka

These are deliberate exclusions to keep scope focused and the signal high.

---

## What this demonstrates

- Principal-level system ownership
- ML treated as a production dependency, not a science project
- Cloud judgment and cost awareness
- Failure-aware design
- Clear contracts and operability discipline

---

## Next steps

- Canary model routing
- Drift scoring job
- Manual retraining trigger
- Expanded alerting
