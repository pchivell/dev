# Cross-Cutting: Observability

Load this file for any production system, multi-service architecture,
or any system with an on-call or SLA requirement.

---

## The Three Pillars

| Pillar | Answers | Tooling |
|--------|---------|---------|
| **Logs** | What happened? | stdout → aggregator (Loki, CloudWatch, Datadog) |
| **Metrics** | How much / how fast? | Prometheus + Grafana, Datadog, CloudWatch |
| **Traces** | Why did it take so long? | OpenTelemetry → Jaeger, Tempo, Datadog APM |

A system is observable when you can answer "what is wrong and why" from external outputs
alone — without deploying new code.

---

## Logging

### Rules

- Log to **stdout** (structured JSON); aggregator collects — never write to local files in containers.
- Every log line includes: `timestamp`, `level`, `service`, `trace_id`, `user_id` (if available), `message`.
- **Log levels**: ERROR (actionable, paging), WARN (degraded, investigate), INFO (key lifecycle events), DEBUG (off in production).
- Never log: passwords, tokens, card numbers, PII (or mask/hash them).
- Log at **boundaries**: incoming request, outgoing call, queue message consumed, error thrown.

### Structured Log Format

```json
{
  "ts": "2024-01-15T10:23:45.123Z",
  "level": "error",
  "service": "order-service",
  "trace_id": "abc123",
  "user_id": "usr_789",
  "msg": "Payment gateway timeout",
  "duration_ms": 5003,
  "gateway": "stripe",
  "http_status": 504
}
```

---

## Metrics

### Golden Signals (instrument ALL of these)

| Signal | What to measure |
|--------|----------------|
| **Latency** | p50, p95, p99 per endpoint/operation |
| **Traffic** | Requests per second; messages per second |
| **Errors** | Error rate (4xx, 5xx) per endpoint |
| **Saturation** | CPU %, memory %, DB connection pool utilisation, queue depth |

### Additional Metrics by Domain

- **API**: request rate, error rate, p99 latency per route
- **Queue**: queue depth, processing lag, consumer count
- **DB**: query duration, connection pool size, slow query count
- **Cache**: hit rate, eviction rate, memory usage
- **Background jobs**: job duration, failure rate, retry count

### Alerting Rules

| Alert | Condition | Severity |
|-------|-----------|---------|
| High error rate | error rate > 1% for 5 min | PagerDuty (P1) |
| High latency | p99 > 2s for 5 min | PagerDuty (P2) |
| DB connections near limit | connection pool > 80% | Slack (P3) |
| Queue backing up | queue depth > 1000 for 10 min | Slack (P3) |

---

## Distributed Tracing

Every inter-service call must propagate a trace context (W3C `traceparent` header).

```
Incoming request → generate trace_id + span_id
  → log with trace_id
  → pass trace_id in downstream HTTP headers
  → each service creates a child span
  → spans collected in Jaeger / Tempo / Datadog
```

**OpenTelemetry** (vendor-neutral SDK) is the standard instrumentation layer:
- Auto-instrument HTTP, gRPC, DB queries, and queue operations
- Export to any backend (Jaeger, Grafana Tempo, Datadog, Honeycomb)

---

## Health Endpoints

Every service must expose:

```
GET /health/live    → 200 if process is alive (liveness probe for K8s restart)
GET /health/ready   → 200 if all dependencies are up (readiness probe — remove from LB if failing)
GET /metrics        → Prometheus text format (scrape endpoint)
```

Readiness check should verify: DB connection, cache connection, required env vars.

---

## Runbook Template

For every alert, maintain a runbook:

```markdown
## Alert: HighPaymentErrorRate

**Description**: Payment error rate > 1% for 5 minutes

**Impact**: Users cannot complete purchases

**Triage steps**:
1. Check Datadog → APM → payment-service → errors for stack trace
2. Check Stripe status page (https://status.stripe.com)
3. Check recent deployments (last 30 min in Argo CD)
4. Check DB connection pool saturation

**Remediation**:
- If Stripe outage: enable maintenance mode (feature flag `disable_checkout`)
- If bad deploy: rollback via Argo CD → Rollback
- If DB exhaustion: restart connection pooler pod
```

---

## Common Pitfalls

- **Logging without trace IDs** — you cannot correlate logs across services without a shared trace ID.
- **Logging raw exceptions with no context** — include what was being attempted, not just the stack trace.
- **Alerting on every log line** — alert on symptoms (error rate, latency), not individual events (noise fatigue).
- **No /health/ready endpoint** — K8s sends traffic to pods that aren't ready; silent errors.
- **Metrics without percentiles** — mean latency hides tail latency misery; always report p95/p99.
- **No on-call runbook** — an alert with no runbook teaches nothing at 2 AM.
