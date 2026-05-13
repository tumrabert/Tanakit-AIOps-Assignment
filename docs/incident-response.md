# Incident Response: Rejection Rate Spike

**Scenario**: You receive a `HighRejectionRate` or `RejectionRateSpike` alert at 3am.  
**System**: Agent API — an LLM-style request classifier running at `http://localhost:8080`

---

## 1. Initial Triage (first 5 minutes)

**Is the API up?**
```bash
curl http://localhost:8080/healthz
```
Expected: `{"status": "healthy", "prompt_version": "v1.0.0"}`  
If this fails → the issue is availability, not rejection rate. See [API Down](#api-down).

**What is the current rejection rate?**
```bash
curl http://localhost:8080/metrics | grep agent_rejections_total
```

Or in Prometheus:
```promql
sum(rate(agent_rejections_total[5m])) / sum(rate(agent_requests_total{route="/ask"}[5m]))
```

Normal baseline is ~15% (configured via `REJECTION_MIX_RATIO=0.15`). Anything above 30% sustained for 5+ minutes is abnormal.

---

## 2. Investigation

### Step 1 — Identify which rejection reason is spiking

```promql
sum by(reason)(rate(agent_rejections_total[5m]))
```

Or via metrics endpoint:
```bash
curl http://localhost:8080/metrics | grep agent_rejections_total
```

This tells you whether the spike is driven by `prompt_injection`, `secrets_request`, or `dangerous_action`.

### Step 2 — Check if a new prompt version was deployed

```promql
count by(prompt_version)(agent_requests_total)
```

A new `prompt_version` label appearing correlates with a recent deployment. Check git log:
```bash
git log --oneline -10
```

Also check the deployment manifest:
```bash
cat deployment/manifest.yml
```

### Step 3 — Check if traffic pattern changed

Goes to Prometheus

```promql
rate(agent_requests_total{route="/ask"}[5m])
```

If request volume also spiked, the traffic generator config or upstream traffic mix may have changed (`REJECTION_MIX_RATIO` env var).

### Step 4 — Check for latency or errors alongside the spike

```promql
# p95 latency
histogram_quantile(0.95, rate(agent_request_latency_seconds_bucket{route="/ask"}[5m]))

# Error rate
sum by(status_code)(rate(agent_errors_total[5m]))

# In-flight requests
agent_in_flight_requests
```

If latency is also elevated and in-flight requests are high → system is under load, not just receiving bad traffic.

---

## 3. Decision Framework

| Observation | Action |
|-------------|--------|
| One rejection reason spiking (e.g. only `prompt_injection`) | Likely an attack or bad client — monitor, no rollback needed |
| All reasons spiking equally | Traffic mix changed — check `REJECTION_MIX_RATIO`, check traffic generator |
| Spike correlates with new `prompt_version` | Prompt regression — rollback deployment |
| Spike + high latency + high in-flight | System under stress — scale or restart |
| Spike + 500 errors appearing | Application bug — rollback immediately |

**Rollback deployment:**
```bash
# Update image_tag in deployment manifest to last known good commit
git log --oneline -5  # find last good SHA
# Update deployment/manifest.yml image_tag and redeploy
```

**Restart the stack:**
```bash
make down && make up
```

**Escalate if:**
- 500 errors are present and increasing
- API becomes unreachable
- Rollback does not resolve the spike within 10 minutes

---

## 4. Post-Incident Actions

**Immediately after resolution:**
- Note the time, duration, and peak rejection rate
- Identify root cause (attack, deployment, config change)

**Within 24 hours:**
- Write a brief post-mortem with timeline and root cause
- If caused by a deployment: add eval gate to CI to catch the regression
- If caused by an attack: review rejection patterns and tighten rules if needed
- If caused by config: document safe ranges for `REJECTION_MIX_RATIO`

**Improve alerting if needed:**
- If alert fired too late → lower the `for` duration on `HighRejectionRate`
- If alert was noisy → raise the threshold or increase `for` duration

---

## API Down

If `/healthz` fails:

```bash
# Check container status
docker ps

# Check logs
docker logs agent-api --tail 50

# Restart
make down && make up
```

If container is running but `/healthz` fails → check for OOM or port conflict:
```bash
docker stats agent-api
```
