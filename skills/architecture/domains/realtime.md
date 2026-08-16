# Domain: Real-time Systems

Applies to: live collaboration tools, chat/messaging, multiplayer games,
live dashboards, trading systems, notification platforms, and any system
where data must reach clients within seconds of an event occurring.

---

## Domain Characteristics

- **Latency target**: typically < 100 ms end-to-end for user-facing; < 10 ms for system-to-system
- **Connection model**: persistent connections vs polling vs push
- **State sync**: server is authoritative; clients derive/cache; conflicts must be resolved
- **Scale**: connection count (not just request rate) is the scaling unit
- **Failure modes**: message loss, duplicate delivery, ordering violations, split-brain

---

## Transport Selection

| Transport | Latency | Direction | When |
|-----------|---------|-----------|------|
| **WebSocket** | < 10 ms | Bidirectional | Chat, collaboration, gaming, live feeds |
| **Server-Sent Events (SSE)** | < 50 ms | Server → Client | Notifications, live dashboards (read-only stream) |
| **HTTP Long-polling** | 50–500 ms | Server → Client | Fallback when WS/SSE not available |
| **HTTP/2 Push** | < 50 ms | Server → Client | Browser push; limited adoption |
| **WebRTC (Data Channel)** | < 10 ms | Peer-to-peer | Video/audio; file transfer; avoid server hop |
| **MQTT over WS** | < 10 ms | Bidirectional | IoT devices; pub-sub patterns |

**Default: WebSocket** for bidirectional; **SSE** for server-push-only (simpler, HTTP/2 friendly).

---

## Architecture Patterns

### Pub-Sub Fan-out

```
Client A publishes event
  → API server validates + persists
  → publishes to message broker (Redis Pub/Sub / Kafka / NATS)
  → all API server instances subscribed to that channel receive it
  → each instance pushes to its locally connected clients in that channel
```

This pattern scales horizontally: add API server instances; the broker fans out.

### Presence and Connection State

Track who is online and in which "room":

```
Redis Hash:  presence:{room_id} → { user_id: last_seen_ts }
On connect:  HSET presence:{room} user_id NOW; broadcast join event
On disconnect (or TTL): HDEL; broadcast leave event
Heartbeat:   client pings every 30 s; server resets TTL
```

### Conflict Resolution for Collaborative Editing

| Algorithm | When |
|-----------|------|
| **Last-write-wins (LWW)** | Non-conflicting fields; simple; acceptable loss |
| **Operational Transformation (OT)** | Document editing; proven; Google Docs model |
| **CRDTs** (Conflict-free Replicated Data Types) | Offline-first + sync; no central authority needed; Yjs |
| **Event sourcing** | Replay events to reconstruct state; full history |

---

## Technology Selection Framework

| Concern | Default tool | Alternatives |
|---------|-------------|-------------|
| WebSocket server | **Socket.io (Node.js)** | native ws, uWebSockets.js (performance) |
| Pub-sub broker | **Redis Pub/Sub** | NATS, Kafka (durability), Ably (managed) |
| Managed real-time | **Ably** or **Pusher** | Firebase Realtime DB, Supabase Realtime |
| Collaborative editing | **Yjs + y-websocket** | ShareDB (OT), Automerge |
| Presence | **Redis** | Upstash, in-memory (single-node only) |
| Message queue (async) | **BullMQ (Redis)** | RabbitMQ |

---

## Scaling WebSocket Connections

- Nginx / HAProxy must be configured for WebSocket upgrade: `proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade;`
- Sticky sessions OR broker fan-out (prefer broker fan-out — no sticky session dependency)
- Each Node.js process handles 10k–100k concurrent connections; scale by adding processes/pods
- Use a **horizontal pod autoscaler on connection count**, not CPU

---

## Message Delivery Guarantees

| Guarantee | How to achieve |
|-----------|---------------|
| **At-most-once** | Fire and forget; no ack — default WebSocket |
| **At-least-once** | Ack + retry; idempotent consumers; use message ID to dedupe |
| **Exactly-once** | Idempotency key + transactional outbox pattern; expensive |

For most UX (chat, notifications), **at-least-once with client-side deduplication** is the right trade-off.

---

## Key Decision Checkpoints

1. **Bidirectional or push-only?** If push-only, SSE is simpler than WebSocket.
2. **Message history / persistence?** If clients need to catch up after reconnect, persist to DB and replay.
3. **Scale target (concurrent connections)?** Determines whether managed service or custom infra.
4. **Offline-first + sync?** CRDTs (Yjs) if users edit while disconnected and changes must merge.
5. **Ordering guarantees?** Per-channel ordering via sequence numbers or Kafka partition ordering.

---

## Common Pitfalls

- **In-memory pub-sub without a broker** — works on one server; breaks the moment you scale to two.
- **No heartbeat / keepalive** — connections silently drop behind NAT gateways and firewalls; clients don't know.
- **Unbounded room size** — broadcasting to 100k clients in one room fans out to 100k writes; limit room size or use layers.
- **No reconnect with backoff on client** — thundering herd on server restart; use exponential backoff + jitter.
- **Storing large payloads in messages** — pass an ID + fetch payload from REST; keep messages small.
- **No message deduplication** — at-least-once delivery means duplicates are real; handle client-side.
