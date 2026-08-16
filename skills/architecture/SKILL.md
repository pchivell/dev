---
name: architecture
description: >
  High-level architecture guidance across all system and business types —
  from embedded firmware to enterprise cloud applications. Classifies the
  system domain, routes to the matching architectural template, and surfaces
  cross-cutting concerns before any code is written.
triggers:
  - design, architect, system design, architecture, structure
  - how should I build, what pattern, tech stack, scaffold, blueprint
  - any new project or system being scoped from scratch
  - refactor, restructure, rethink an existing system
version: 1.0.0
---

# Architecture Skill — Master Guide

## Purpose
Before writing code for any new system (or significantly restructuring an existing one),
classify the domain, load the matching architectural template, and confirm the design
with the user. This skill is the entry point; the domain files do the heavy lifting.

---

## Step 1 — Classify the System Domain

Read the user's description and assign **one primary domain** plus any secondary domains:

| Domain | Key signals |
|--------|-------------|
| **Embedded / Firmware** | MCU, microcontroller, bare metal, RTOS, GPIO, sensors, actuators, tight RAM/flash budget, hard real-time |
| **IoT / Edge** | device fleet, telemetry, MQTT, OTA updates, edge processing, gateway, sensor-to-cloud pipeline |
| **Mobile** | iOS, Android, React Native, Expo, phone/tablet UX, offline-first, push notifications |
| **Web Frontend** | browser, SPA, SSR, SSG, Next.js/React/Vue/Svelte, UI/UX, client-side state |
| **Backend API / Services** | REST, GraphQL, gRPC, server-side business logic, relational/document DB, auth |
| **Cloud-Native / Microservices** | containers, Kubernetes, Docker Compose, Lambda/serverless, multi-service topology |
| **Enterprise Business Application** | multi-tenant SaaS, DDD, domain model, audit trail, compliance, large org workflows |
| **Data Platform** | ETL/ELT, data warehouse, lakehouse, analytics, ML feature store, batch or streaming pipelines |
| **Real-time System** | sub-second latency, WebSocket/SSE, live collaboration, event streaming (Kafka/NATS), pub-sub |

> **Tip:** Full-stack or product systems commonly span multiple domains.
> Load ALL matching domain guides when that is the case.

---

## Step 2 — Load the Domain Guide

Fetch and read the relevant file from `skills/architecture/domains/`:

```
Embedded / Firmware         → domains/embedded.md
IoT / Edge                  → domains/iot-edge.md
Mobile                      → domains/mobile.md
Web Frontend                → domains/web-frontend.md
Backend API / Services      → domains/backend-api.md
Cloud-Native / Microservices→ domains/cloud-native.md
Enterprise Business App     → domains/enterprise.md
Data Platform               → domains/data-platform.md
Real-time System            → domains/realtime.md
```

Each domain guide contains:
- Domain characteristics and constraints
- Recommended architectural layers and patterns
- Technology selection framework
- Common pitfalls and anti-patterns
- Key decision checkpoints to discuss with the user

---

## Step 3 — Apply Cross-Cutting Concerns

After loading domain guides, check which of these apply and load the file:

| Concern | When to load |
|---------|--------------|
| Security | any system with users, data, or external exposure → `cross-cutting/security.md` |
| Scalability | traffic > single machine, SLA requirements, cost sensitivity → `cross-cutting/scalability.md` |
| Observability | production system, multi-service, or on-call requirement → `cross-cutting/observability.md` |
| Data Modeling | non-trivial schema, migrations, multi-DB, or reporting needs → `cross-cutting/data-modeling.md` |

---

## Step 4 — Present an Architecture Decision Record (ADR)

Before writing any code, present a brief ADR to the user for approval:

```
### ADR-001: <Decision title>

**Context**    What is the situation, constraints, and key unknowns?
**Options**    What were the realistic alternatives?
**Decision**   What was chosen?
**Rationale**  Why this over the alternatives?
**Trade-offs** What are the costs and risks of this decision?
**Open items** What still needs to be resolved?
```

Get explicit user approval before proceeding to implementation.

---

## Universal Principles (apply to ALL domains)

1. **Separation of concerns** — each component, module, or service has one reason to change.
2. **Dependency direction** — dependencies always point inward toward domain/business logic, never outward toward infrastructure.
3. **Explicit over implicit** — contracts, configuration, and interfaces are written down, not assumed.
4. **Design for failure** — every external call can fail; every process can restart; design accordingly.
5. **Reversibility** — prefer decisions that can be undone; make irreversible ones consciously and document them.
6. **Keep it boring** — use the simplest solution that satisfies the constraints; earn complexity only when forced.
7. **Defer decisions** — don't choose a database, framework, or protocol before constraints are known.
8. **Boundaries over big balls of mud** — define hard module/service boundaries early; they are cheap now and expensive later.

---

## Interaction Protocol when Coding

1. Detect that a new system or significant feature is being built.
2. Silently classify the domain using Step 1.
3. Load the matching domain guide(s) and relevant cross-cutting files.
4. Ask the minimum clarifying questions needed (one per unknown dimension: scale, constraints, team, existing stack).
5. Present the ADR — architecture proposal in plain language — before writing any code.
6. On approval, implement following the patterns in the loaded guides.
7. Flag any deviation from the proposed architecture as it arises.
