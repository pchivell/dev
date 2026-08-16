# Cross-Cutting: Scalability & Performance

Load this file when a system has SLA requirements, expects significant load,
or when cost efficiency at scale is a stated concern.

---

## Scalability Decision Tree

```
Can the problem be solved by making the server bigger (vertical scale)?
  YES, and it's cheaper → vertical scale first; defer complexity
  NO or cost-prohibitive → horizontal scale

Can you add stateless replicas behind a load balancer?
  YES → horizontal scale; ensure session state is in Redis/DB, not memory
  NO  → identify the stateful bottleneck and extract it
```

**Always measure before optimising.** Premature optimisation is the root of unnecessary complexity.

---

## Caching Strategy

### Cache Layers (outermost → innermost)

```
Browser cache (Cache-Control headers)
  → CDN / Edge cache (Cloudflare, CloudFront)
    → Application-level cache (Redis / Memcached)
      → Query-level cache (DB result cache)
        → DB (source of truth)
```

### Cache Invalidation Patterns

| Pattern | When |
|---------|------|
| **TTL (time-to-live)** | Acceptable staleness; simplest |
| **Cache-aside (lazy load)** | Read from cache; on miss, load from DB and populate |
| **Write-through** | Write to cache and DB simultaneously; cache always warm |
| **Write-behind** | Write to cache; async flush to DB; risk of data loss |
| **Event-driven invalidation** | Domain event triggers cache eviction; most accurate |

Cache keys: deterministic, versioned (`v1:user:{id}:profile`). Include a version prefix to bust the entire cache on breaking schema changes.

---

## Database Scaling

| Technique | When |
|-----------|------|
| **Connection pooling** (PgBouncer) | Always — never open a raw connection per request |
| **Read replicas** | Read-heavy workloads; analytics queries; reporting |
| **Vertical scale** (larger instance) | First step; often sufficient longer than expected |
| **Indexes** | Every FK, every WHERE/ORDER BY column; explain-analyse first |
| **Partitioning** (by date/tenant) | Tables > 100M rows; time-series data |
| **Caching hot reads** | Frequently read, rarely changed data (config, enums) → Redis |
| **Horizontal sharding** | Last resort; enormous complexity; rarely needed below 10 TB |

### Index Checklist

```sql
-- Foreign keys (almost always missing)
CREATE INDEX ON orders(user_id);
-- Composite indexes for common WHERE combinations
CREATE INDEX ON events(tenant_id, created_at DESC);
-- Partial indexes for filtered queries
CREATE INDEX ON jobs(status) WHERE status = 'pending';
-- EXPLAIN ANALYZE before adding; check index usage after
```

---

## API & Service Performance

| Pattern | Detail |
|---------|--------|
| **Pagination** | Cursor-based (opaque) for large/live datasets; offset for simple cases |
| **Field projection** | `SELECT` only needed columns; never `SELECT *` in production |
| **Async for slow operations** | Anything > 200 ms → task queue (BullMQ/Celery) + webhook/polling |
| **HTTP caching** | `ETag` + `Cache-Control` on read endpoints |
| **Compression** | `gzip`/`br` on HTTP responses; binary formats (Protobuf/MessagePack) for high-volume APIs |
| **Connection reuse** | HTTP keep-alive; reuse DB/Redis connections across requests |

---

## Load Testing Baselines

Before production launch, run:

1. **Baseline test**: single user, confirm p50/p95/p99 latency under no load
2. **Load test**: ramp to expected peak; confirm latency stays within SLA
3. **Stress test**: ramp beyond peak until failure; identify the bottleneck
4. **Soak test**: sustained load for hours; identify memory leaks and connection pool exhaustion

Tools: **k6** (scripts), **Locust** (Python), **Artillery** (Node).

---

## Key Scalability Metrics to Define Upfront

| Metric | Example target |
|--------|---------------|
| Concurrent users | 10 000 |
| Requests per second (RPS) | 500 RPS peak |
| p95 API latency | < 200 ms |
| p99 API latency | < 1 s |
| Availability SLA | 99.9% (8.7 h downtime/year) |
| Recovery Time Objective (RTO) | < 1 hour |
| Recovery Point Objective (RPO) | < 5 minutes |

---

## Common Pitfalls

- **Optimising before measuring** — profile first; the bottleneck is almost never where you think it is.
- **No connection pooling** — opening a new DB connection per request crushes Postgres at moderate load.
- **Unbounded queries** — `SELECT * FROM orders` with no `WHERE`/`LIMIT` returns everything; add pagination.
- **Synchronous calls to slow dependencies** — one slow external API makes your API slow; use timeouts + async.
- **No cache eviction strategy** — stale cache served forever after a write; define TTL or event invalidation.
- **Sessions in local memory** — sticky sessions or lost sessions on restart; use Redis.
