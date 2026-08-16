# Domain: Enterprise Business Application

Applies to: multi-tenant SaaS platforms, ERP/CRM systems, internal
line-of-business tools, workflow automation, and any system where
the core value is in modelling a rich, complex business domain rather
than in infrastructure scale.

---

## Domain Characteristics

- **Complexity source**: the business rules, not the technology
- **Longevity**: expected to run for 5–15 years; architectural decisions compound
- **Team size**: multiple teams owning different bounded contexts
- **Compliance**: audit trail, data retention, GDPR/SOC2/HIPAA are frequent requirements
- **Change rate**: business rules change frequently; infrastructure changes rarely

---

## Recommended Architecture: Domain-Driven Design (DDD)

### Strategic Design First

Before writing code, map the business domain:

```
1. Event Storming / Domain Discovery
   → identify domain events (OrderPlaced, PaymentFailed, UserProvisioned)
   → identify commands (PlaceOrder, ProcessPayment, ProvisionUser)
   → identify aggregates (Order, Account, Subscription)

2. Bounded Contexts
   → group related concepts into cohesive sub-domains
   → each context has its own ubiquitous language and model
   → contexts communicate via defined contracts (not shared DB tables)

3. Context Map
   → show how contexts relate: upstream/downstream, anti-corruption layer, shared kernel
```

### Tactical Patterns

| Pattern | Purpose |
|---------|---------|
| **Entity** | Object with a unique identity that persists over time (Order #12345) |
| **Value Object** | Immutable; no identity; defined by attributes (Money, Address, DateRange) |
| **Aggregate** | Cluster of entities/VOs with one Aggregate Root; transactional boundary |
| **Repository** | Persist and retrieve aggregates; hides storage mechanism |
| **Domain Service** | Stateless operation that doesn't belong to one aggregate (PricingService) |
| **Domain Event** | Something that happened; past tense; triggers downstream reactions |
| **Application Service** | Orchestrates use cases; no business logic; calls domain + infrastructure |

### CQRS (Command Query Responsibility Segregation)

Use when read models and write models have different shapes or scale differently:

```
Write side:
  Command → Command Handler → Aggregate → Domain Event → Event Store

Read side:
  Event → Projection → Read Model (denormalised, query-optimised)
  Query → Query Handler → Read Model → Response
```

**Don't CQRS everything** — apply it to the bounded contexts where read/write asymmetry is real.

### Multi-Tenancy Models

| Model | Isolation | Cost | When |
|-------|-----------|------|------|
| **Silo** (DB per tenant) | High | High | Regulated industries; large enterprise customers |
| **Bridge** (schema per tenant) | Medium | Medium | Mid-market SaaS; PostgreSQL schemas |
| **Pool** (shared tables + tenant_id) | Low | Low | SMB SaaS; simple compliance; most common default |

Default: **Pool model with Row-Level Security (RLS)** in PostgreSQL — simplest ops, adequate isolation.

---

## Audit Trail Pattern

Every mutation to regulated data must produce an immutable audit record:

```sql
audit_log (
  id          UUID PRIMARY KEY,
  entity_type TEXT,        -- 'Order'
  entity_id   UUID,        -- the affected record
  action      TEXT,        -- 'created' | 'updated' | 'deleted'
  changes     JSONB,       -- {field: [old_value, new_value]}
  actor_id    UUID,        -- who made the change
  actor_type  TEXT,        -- 'user' | 'system' | 'api_key'
  ip_address  INET,
  created_at  TIMESTAMPTZ  -- never updated
)
```

Append-only — never UPDATE or DELETE audit rows.

---

## Technology Selection Framework

| Decision | Default |
|----------|---------|
| Language | **TypeScript (NestJS)** or **Java (Spring Boot)** |
| ORM | **Prisma** (TS) / **Hibernate** (Java) |
| Event bus (internal) | **NestJS EventEmitter** or **MediatR** pattern |
| Event bus (cross-service) | **Kafka** or **RabbitMQ** |
| Auth | **Keycloak** (enterprise SSO) or **Auth0** |
| Search | **Elasticsearch** or **PostgreSQL FTS** |
| Reporting | **Metabase** (embedded) or custom with **Recharts** |

---

## Key Decision Checkpoints

1. **Domain complexity worth DDD?** If the business rules fit on one page, a simple CRUD app is fine.
2. **Event sourcing vs state-based?** Event sourcing gives full audit + replay; adds operational complexity.
3. **Multi-tenancy isolation level?** Driven by customer contracts and compliance requirements.
4. **Workflows / BPM?** Long-running business processes → consider a workflow engine (Temporal, Camunda).
5. **Regulatory requirements?** Data residency, retention policies, right-to-erasure impact the data model.
6. **Integration with existing systems?** Anti-Corruption Layer (ACL) prevents legacy models polluting the new domain.

---

## Common Pitfalls

- **Anemic domain model** — entities with no behaviour (only getters/setters); business logic leaks into services.
- **Big Ball of Mud** — no bounded context boundaries; one giant shared model that everyone modifies.
- **Skipping ubiquitous language** — code names that don't match business terms; knowledge translation overhead.
- **Shared DB between bounded contexts** — schema coupling prevents independent evolution.
- **No audit trail** — retrofitting audit is expensive and incomplete; build it from day one.
- **CQRS everywhere** — adds complexity; apply only where there is genuine read/write asymmetry.
