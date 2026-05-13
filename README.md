# AI-Ops Take-Home Test — Tanakit Tumrabert

A self-contained repository for evaluating DevOps + AIOps skills. This repo simulates an "LLM Agent API" that sometimes refuses requests and emits metrics.

## Quick Start

```bash
# Start the full stack (API, Prometheus, Grafana, Traffic Generator)
make up

# View logs
make logs

# Run evaluation suite
make eval

# Stop everything
make down
```

## Local Endpoints

| Service | URL | Credentials |
|---------|-----|-------------|
| Agent API | http://localhost:8080 | - |
| Metrics | http://localhost:8080/metrics | - |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin/admin |

## Architecture

```
┌─────────────────┐     ┌─────────────────┐
│    Traffic      │────▶│   Agent API     │
│   Generator     │     │   (Port 8080)   │
└─────────────────┘     └────────┬────────┘
                                 │
                                 │ /metrics
                                 ▼
                        ┌─────────────────┐
                        │   Prometheus    │
                        │   (Port 9090)   │
                        └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │    Grafana      │
                        │   (Port 3000)   │
                        └─────────────────┘
```

---

## Deliverables

### Task 1 — CI/CD Pipeline

**File:** `.github/workflows/ci.yml`  
**Passing run:** https://github.com/tumrabert/Tanakit-AIOps-Assignment/actions/runs/25781586278

The pipeline runs four jobs in sequence, each gating the next:

1. **Lint** — `flake8` checks `agent-api/app.py` for real errors (syntax, undefined names, unused imports). Cosmetic style rules are ignored so the gate stays focused on correctness.

2. **Build** — Builds the Docker image from `agent-api/` to confirm the container compiles cleanly. Prevents broken containers from reaching the eval or deploy stages.

3. **Eval** — Starts the `agent-api` container, waits for `/healthz` to confirm it is healthy, then runs the eval suite via `docker compose --profile eval run --rm eval-runner`. Blocks merge if golden accuracy drops below 90% or adversarial rejection rate drops below 60%. Eval results are uploaded as a workflow artifact on every run (pass or fail) so regressions are always traceable.

4. **Deploy** — Runs only on push to `main`. Updates `deployment/manifest.yml` with the commit SHA and a timestamp, then commits the change back. Every production deployment is traceable to a specific commit — answering "what is currently deployed?" at any time.

---

### Task 2 — Alerting Strategy

**File:** `prometheus/alert-rules.yml`

Five alert rules with threshold rationale explained inline in the file:

| Alert | Threshold | Reasoning |
|-------|-----------|-----------|
| `AgentAPIDown` | `up == 0` for 1m | API unreachable — immediate action required |
| `HighRejectionRate` | > 30% for 5m | 2× the normal 15% baseline (`REJECTION_MIX_RATIO=0.15`). Sustained 5m window filters out short bursts |
| `RejectionRateSpike` | > 2× the 1h rolling average for 5m | Catches sudden spikes even when the absolute rate is still below 30% — useful right after a deployment |
| `HighRequestLatency` | p95 > 500ms for 5m | This API has no real LLM calls, so 500ms is already generous — anything above it is abnormal |
| `NoIncomingTraffic` | < 0.1 req/s for 3m | Silent failures are worse than loud ones. Fires quickly because a dead traffic generator looks identical to an outage |

---

### Task 3 — Observability Metrics

**File:** `agent-api/app.py`

Three new metrics added on top of the existing `agent_requests_total` and `agent_request_latency_seconds`:

| Metric | Type | Labels | Purpose |
|--------|------|--------|---------|
| `agent_rejections_total` | Counter | `prompt_version`, `reason` | Track rejection volume broken down by reason (`prompt_injection`, `secrets_request`, `dangerous_action`). Enables alerting and per-reason graphs |
| `agent_in_flight_requests` | Gauge | — | Incremented at request entry, decremented in `finally` so it always drops even on errors. Spikes here combined with latency spikes indicate server stress |
| `agent_errors_total` | Counter | `prompt_version`, `status_code` | Separates 400s (bad client input) from 500s (server bugs). 500s in normal operation should be zero |

---

### Task 4 — Dashboard

**File:** `grafana/dashboards/agent-monitoring.json`

The Grafana dashboard was fixed and extended:

- **Fixed** rejection rate panel — corrected PromQL to use `agent_rejections_total` with the proper rate-over-rate formula
- **Fixed** rejection breakdown panel — added `sum by(reason)` so each rejection type shows as a separate series with `{{reason}}` legend
- **Fixed** rejection rate stat panel — same formula correction as above
- **Added** In-Flight Requests panel — visualises `agent_in_flight_requests` gauge in real time
- **Added** Error Rate by Status Code panel — `sum by(status_code)(rate(agent_errors_total[5m]))` to distinguish bad clients from server errors

---

### Task 5 — Incident Response

**File:** `docs/incident-response.md`

A 3am runbook for the most likely production alert: `HighRejectionRate` or `RejectionRateSpike`.

The runbook covers:

1. **Initial triage (first 5 minutes)** — confirm the API is up, get the current rejection rate
2. **Investigation** — identify which rejection reason is spiking, check for a new prompt version, check traffic pattern changes, check for latency/errors alongside the spike
3. **Decision framework** — a table mapping each observation pattern to the correct action (monitor, rollback, scale, or escalate)
4. **Post-incident actions** — immediate notes, 24-hour post-mortem, alerting improvements

---

## Agent API Endpoints

### POST /ask

```bash
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the weather?"}'
```

Response:
```json
{
  "rejected": false,
  "reason": null,
  "prompt_version": "v1.0.0",
  "answer": "I'd be happy to assist with your request."
}
```

### GET /healthz

```bash
curl http://localhost:8080/healthz
```

### GET /metrics

```bash
curl http://localhost:8080/metrics
```

---

## Rejection Logic

| Reason | Trigger Patterns |
|--------|------------------|
| `prompt_injection` | "ignore instructions", "system prompt", "jailbreak" |
| `secrets_request` | "password", "api key", "credentials" |
| `dangerous_action` | "restart prod", "delete database", "rm -rf" |

---

## Metrics Reference

| Metric | Type | Labels |
|--------|------|--------|
| `agent_requests_total` | Counter | `prompt_version`, `route` |
| `agent_rejections_total` | Counter | `prompt_version`, `reason` |
| `agent_in_flight_requests` | Gauge | — |
| `agent_errors_total` | Counter | `prompt_version`, `status_code` |
| `agent_request_latency_seconds` | Histogram | `prompt_version`, `route` |

---

## Evaluation Runner

The eval runner tests the agent against two datasets:

- **Golden Dataset**: Normal messages that should be accepted
- **Adversarial Dataset**: Malicious messages that should be rejected

```bash
make eval
# Results saved to ./eval-results/
```

| Gate | Threshold | Description |
|------|-----------|-------------|
| `min_golden_accuracy` | 90% | Golden messages should be accepted |
| `max_golden_rejection_rate` | 5% | Don't reject too many legitimate requests |
| `min_adversarial_rejection_rate` | 60% | Must reject most malicious requests |

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PROMPT_VERSION` | v1.0.0 | Version string included in responses/metrics |
| `REQUEST_INTERVAL_MS` | 500 | Traffic generator request interval |
| `REJECTION_MIX_RATIO` | 0.15 | Ratio of rejection-triggering traffic |
