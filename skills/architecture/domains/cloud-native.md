# Domain: Cloud-Native / Microservices

Applies to: container-orchestrated systems, microservice architectures,
serverless platforms, and any system designed for elastic scaling on
public cloud (AWS, GCP, Azure) or on-premise Kubernetes.

---

## Domain Characteristics

- **Scale**: horizontal; scale individual services independently
- **Teams**: Conway's Law — service boundaries should mirror team boundaries
- **Deployment**: immutable containers; GitOps; blue/green or canary releases
- **Failure**: partial failure is normal; every service must tolerate its dependencies being unavailable
- **Complexity**: network latency, distributed transactions, and observability are multiplied by service count

---

## When to Go Microservices

**Start with a well-structured monolith.** Migrate to microservices only when:
- A specific component has a fundamentally different scaling profile
- A team boundary requires independent deployment
- A regulatory or data isolation requirement forces separation
- The monolith's build/test cycle becomes a productivity bottleneck

Premature microservices are the leading cause of unnecessary complexity. If in doubt, stay modular monolith.

---

## Recommended Architecture

### Service Topology Patterns

| Pattern | When |
|---------|------|
| **API Gateway + backends** | Single entry point; fan-out to services; handles auth, rate limiting, routing |
| **Backend-for-Frontend (BFF)** | Web and mobile have divergent data needs; one BFF per client type |
| **Event-driven / CQRS** | High throughput writes; audit trail required; eventual consistency acceptable |
| **Saga pattern** | Distributed transactions across services (no 2-phase commit) |
| **Sidecar / Service Mesh** | Cross-cutting concerns (mTLS, retries, circuit breaker) without code changes |

### Twelve-Factor App (mandatory checklist)

1. **Codebase**: one repo per service; tracked in version control
2. **Dependencies**: declared explicitly (package.json, requirements.txt, go.mod)
3. **Config**: stored in environment variables, never in code
4. **Backing services**: DB, cache, queue treated as attached resources (swappable via env var)
5. **Build/release/run**: strict separation; build artefact is immutable
6. **Processes**: stateless; share nothing; state lives in backing services
7. **Port binding**: service exports itself via a port; no app server
8. **Concurrency**: scale out via process model
9. **Disposability**: fast startup; graceful shutdown (drain in-flight requests on SIGTERM)
10. **Dev/prod parity**: keep environments as similar as possible
11. **Logs**: treat as event streams; write to stdout; aggregator collects
12. **Admin processes**: run as one-off processes (migrations, scripts) in the same environment

### Container / Kubernetes Checklist

```
□ Dockerfile: multi-stage build; non-root user; minimal base image (distroless/alpine)
□ Health checks: /health/live (process up) and /health/ready (dependencies up)
□ Resource requests AND limits set on every container
□ Horizontal Pod Autoscaler (HPA) configured
□ Pod Disruption Budget for critical services
□ Secrets from Kubernetes Secrets or external vault (never baked into image)
□ ConfigMap for non-sensitive config
□ Graceful shutdown: SIGTERM handler drains connections before exit
□ Image tagged with git SHA, not :latest
```

---

## Inter-Service Communication

| Style | Default tool | When |
|-------|-------------|------|
| Sync request-reply | **HTTP/REST or gRPC** | Query that needs an immediate answer |
| Async event / message | **Kafka / NATS / RabbitMQ** | Fire-and-forget; fan-out; decoupled producers/consumers |
| Streaming | **gRPC streaming / Kafka** | Continuous data flow between services |

**Avoid synchronous chains longer than 2 hops** — latency compounds and failure cascades.

### Resilience Patterns

- **Circuit breaker**: stop calling a failing dependency; fail fast; try again after cooldown
- **Retry with exponential backoff + jitter**: transient failures recover without stampede
- **Timeout on every external call**: never wait indefinitely
- **Bulkhead**: isolate thread pools per dependency so one slow dep doesn't starve others
- **Dead-letter queue**: messages that fail repeatedly go to DLQ for inspection, not silent discard

---

## Key Decision Checkpoints

1. **Cloud provider lock-in tolerance?** Kubernetes abstracts compute; managed services (RDS, SQS) trade portability for ops savings.
2. **Event sourcing needed?** Adds auditability and replayability; adds operational complexity.
3. **Service mesh (Istio/Linkerd)?** Justified only for 10+ services; adds overhead below that.
4. **Multi-region / active-active?** Data consistency model (strong vs eventual) must be decided before DB selection.
5. **GitOps tooling?** ArgoCD or Flux from day one if using Kubernetes.
6. **Shared library vs duplication?** Prefer duplication over a shared library that couples release trains.

---

## Common Pitfalls

- **Microservices too early** — distributed monolith is worse than a monolith.
- **Synchronous request chain across 5+ services** — latency = sum of all hops; use async messaging.
- **No service-level SLOs** — without SLOs, you don't know when something is broken.
- **Shared database between services** — violates service autonomy; creates hidden coupling.
- **No distributed tracing** — debugging without trace IDs is guesswork.
- **`:latest` in production** — immutable image tags; always pin to a digest or git SHA.
