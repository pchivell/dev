# Domain: Backend API / Services

Applies to: server-side business logic, REST/GraphQL/gRPC APIs,
monolithic backends, and small-to-medium service architectures
(not full microservices — see cloud-native.md for that).

---

## Domain Characteristics

- **Primary concerns**: correctness, security, data integrity, and latency
- **Clients**: web browsers, mobile apps, third parties, or other internal services
- **State**: owned in a database; the API is a façade over the data model
- **Scaling unit**: typically a single deployable unit (monolith) to start
- **Failure modes**: dependency failures (DB, cache, external APIs), resource exhaustion

---

## Recommended Architecture

### Layered Monolith (default starting point)

```
┌──────────────────────────────────────────────┐
│  Transport Layer (HTTP / gRPC / WS)          │  ← routing, request parsing, auth middleware
├──────────────────────────────────────────────┤
│  Application Layer (Controllers / Handlers)  │  ← orchestrate use cases; no business logic
├──────────────────────────────────────────────┤
│  Domain / Service Layer                      │  ← business rules; pure; easily testable
├──────────────────────────────────────────────┤
│  Repository / Data Access Layer              │  ← DB queries; abstracts ORM/driver
├──────────────────────────────────────────────┤
│  Infrastructure                              │  ← DB connection, cache, email, queue clients
└──────────────────────────────────────────────┘
```

**The domain layer must not import from the transport or infrastructure layers.**
Dependencies flow downward; the domain never knows it's being called via HTTP.

### API Style Selection

| Style | When |
|-------|------|
| **REST** | Default; CRUD resources; public or partner-facing API; broad tooling |
| **GraphQL** | Multiple clients with different data-shape needs; mobile + web diverge |
| **gRPC** | Internal service-to-service; low latency; strong contract enforcement |
| **tRPC** | TypeScript monorepo; shared types between Next.js frontend and Node backend |
| **WebSocket** | Bidirectional real-time (see realtime.md) |

### Database Selection Framework

| Need | Default | Alternative |
|------|---------|-------------|
| Relational / transactional | **PostgreSQL** | MySQL, SQLite (small/embedded) |
| Document / flexible schema | **MongoDB** | Firestore, DynamoDB |
| Cache / sessions | **Redis** | Memcached |
| Full-text search | **PostgreSQL FTS** | Elasticsearch, Typesense |
| Time-series (IoT, metrics) | **TimescaleDB** | InfluxDB, QuestDB |
| Graph relationships | **Neo4j** | PostgreSQL with recursive CTEs |

### Authentication Patterns

| Pattern | When |
|---------|------|
| **JWT (access + refresh token)** | Stateless API; mobile or SPA clients |
| **Session cookie (httpOnly)** | Web app with SSR; simplest; CSRF protection needed |
| **API key** | Server-to-server; third-party integrations |
| **OAuth 2.0 + OIDC** | Federated identity; social login; enterprise SSO |
| **mTLS** | Internal microservice auth; zero-trust networks |

Access tokens: short-lived (15 min). Refresh tokens: long-lived, stored server-side (revocable).

---

## Technology Selection Framework

| Decision | Default (Node/TS) | Default (Python) |
|----------|------------------|-----------------|
| Framework | **NestJS** | **FastAPI** |
| ORM | **Prisma** | **SQLAlchemy 2.x** |
| Validation | **Zod / class-validator** | **Pydantic** |
| Auth | **Passport.js / JWT** | **python-jose / FastAPI-Users** |
| Testing | **Jest + Supertest** | **pytest + httpx** |
| Task queue | **BullMQ (Redis)** | **Celery (Redis/RabbitMQ)** |
| API docs | **Swagger (built-in NestJS)** | **OpenAPI (built-in FastAPI)** |

---

## Key Decision Checkpoints

1. **Monolith vs services from day one?** Start with a modular monolith; split only when a team or scaling boundary forces it.
2. **Sync vs async?** Use a task queue for anything > 200 ms (email, PDF generation, external API calls).
3. **DB per service or shared?** Shared DB is fine for a monolith; each service must own its schema if you go microservices.
4. **Versioning strategy?** `/v1/`, `/v2/` URL versioning is simple and explicit; header versioning for strict REST purists.
5. **Rate limiting from day one?** Yes — per user/IP; use Redis sliding window; prevents abuse and runaway clients.
6. **Idempotency?** All POST/mutation endpoints that create money, orders, or send comms must accept an idempotency key.

---

## Common Pitfalls

- **Business logic in controllers** — controllers orchestrate; services own rules; keep them separate.
- **N+1 queries** — always check ORM-generated SQL in development; use `include`/`select` to eager-load.
- **No request validation** — validate ALL inputs at the transport boundary; never trust the caller.
- **Passwords in plaintext or reversible encryption** — bcrypt/argon2 only; never log passwords.
- **Missing database indexes** — add indexes on every foreign key and every column used in `WHERE`/`ORDER BY`.
- **Long synchronous request chains** — anything involving email, PDF, or external API must go through a queue.
- **No pagination on list endpoints** — unbounded lists crash servers at scale; page from day one.
