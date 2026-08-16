# Cross-Cutting: Data Modeling

Load this file when a system has a non-trivial schema, multiple tables,
relationships across domains, migration requirements, or multi-database concerns.

---

## Modeling Process

1. **Identify entities** — what are the nouns in the domain? (User, Order, Product, Invoice)
2. **Identify relationships** — one-to-many, many-to-many, optional vs required
3. **Identify cardinality** — how many of each? (one user → many orders; one order → many items)
4. **Identify access patterns** — what queries will be run? (shapes the indexes and denormalisation decisions)
5. **Draw the ERD** before writing any DDL

---

## Normalisation Defaults

| Normal Form | Rule | Apply when |
|-------------|------|-----------|
| **1NF** | No repeating groups; atomic columns | Always |
| **2NF** | No partial dependencies on composite PK | Always |
| **3NF** | No transitive dependencies | Default for OLTP |
| **Denormalised** | Duplicate data to avoid joins | Read-heavy reporting / OLAP |

Start at **3NF for OLTP**. Denormalise deliberately (with justification) for read performance.

---

## Primary Key Strategy

| Strategy | Type | When |
|----------|------|------|
| **ULID / UUIDv7** | Sortable UUID | Default for new systems; monotonic → better index performance than random UUID |
| **UUID v4** | Random UUID | Distributed generation without coordination; fine if insert rate is low |
| **Serial / BIGSERIAL** | Auto-increment integer | Legacy systems; internal tables; never expose externally (enumerable) |
| **Natural key** | Business identifier | Only if truly unique and immutable (rare) |

**Default: UUIDv7 or ULID** — globally unique, sortable, safe to expose in URLs.
Never use auto-increment IDs in public URLs — they leak row counts and are trivially enumerable.

---

## Relationships

### One-to-Many

```sql
-- orders belong to users
orders.user_id REFERENCES users(id) ON DELETE CASCADE | RESTRICT | SET NULL
```
Choose `ON DELETE` carefully:
- `CASCADE` — child rows deleted with parent (cascade deletes of weak entities)
- `RESTRICT` — prevent deletion if children exist (protect data integrity)
- `SET NULL` — orphan children (rarely correct; think carefully)

### Many-to-Many

Always via a junction table; add business attributes to the junction when needed:

```sql
CREATE TABLE product_tags (
  product_id UUID REFERENCES products(id),
  tag_id     UUID REFERENCES tags(id),
  added_at   TIMESTAMPTZ DEFAULT NOW(),
  added_by   UUID REFERENCES users(id),
  PRIMARY KEY (product_id, tag_id)
);
```

---

## Soft Deletes vs Hard Deletes

| Approach | When |
|----------|------|
| **Hard delete** | Non-regulated data; no audit requirement; simplest |
| **Soft delete** (`deleted_at TIMESTAMPTZ`) | Audit trail; undo capability; regulated data |
| **Archive table** | Long-term history; keep hot table small |

If using soft deletes:
- Add `WHERE deleted_at IS NULL` to ALL queries (use a DB view or ORM default scope).
- Index on `deleted_at` or use a partial index: `CREATE INDEX ON orders(id) WHERE deleted_at IS NULL`.

---

## Temporal / Versioned Data

When history matters (contract versions, price history, config snapshots):

```sql
-- Bi-temporal table pattern
entity_versions (
  id          UUID PRIMARY KEY,
  entity_id   UUID NOT NULL,       -- which entity
  valid_from  TIMESTAMPTZ NOT NULL, -- when this version became true in the world
  valid_to    TIMESTAMPTZ,          -- NULL = current version
  created_at  TIMESTAMPTZ,          -- when this row was inserted (system time)
  -- ... all versioned columns
)
```

---

## Migrations

- **Every schema change is a migration file** — never edit the DB schema manually in production.
- Migrations are forwards and **backwards** (down migration) wherever possible.
- Run migrations in **CI** before deployment; deployment fails if migration fails.
- **Zero-downtime migration checklist** for high-traffic tables:
  1. Add column as nullable (no default that locks table)
  2. Backfill in batches (small transactions, not one giant UPDATE)
  3. Add NOT NULL constraint after backfill
  4. Deploy code that reads/writes new column
  5. Drop old column in a later release

---

## Naming Conventions

```
Tables:       plural snake_case          (users, order_items, payment_methods)
Columns:      singular snake_case        (user_id, created_at, is_active)
Primary keys: id                         (always)
Foreign keys: {referenced_table_singular}_id  (user_id, product_id)
Timestamps:   created_at, updated_at, deleted_at  (consistent everywhere)
Booleans:     is_{state} or has_{thing}  (is_active, has_subscription)
Indexes:      idx_{table}_{columns}      (idx_orders_user_id_created_at)
```

---

## Common Pitfalls

- **No foreign key constraints** — referential integrity enforced only in application code; DB can contain orphans.
- **Storing arrays/JSON where relations belong** — `tags: ["a","b"]` in a column can't be queried efficiently; use a junction table.
- **`updated_at` not automatically maintained** — use a DB trigger or ORM hook; don't rely on application code.
- **No index on foreign keys** — Postgres does NOT auto-index FKs; add them manually.
- **Single column for polymorphic type** — `entity_type` + `entity_id` patterns are hard to constrain and query; prefer separate tables.
- **Migrations that lock tables** — `ALTER TABLE ADD COLUMN NOT NULL` on a large table takes a full table lock; use the backfill pattern above.
