# Real-time Notifier — Design Spec

**Date:** 2026-05-03
**Status:** Draft → pending review
**Stack:** Java 21 · Spring Boot 3.3 · raw WebSocket · Lettuce (Redis) · Maven · JUnit 5 · Testcontainers · Micrometer

---

## 1. Goal

A small, production-shaped notification service that pushes events from backend producers to connected end-users in real time. Multi-node capable, defensible in a system-design interview.

Concretely:

- **Fanout** — one event reaches every subscribed client across N service nodes.
- **Real-time** — push over WebSocket, not poll.
- **At-least-once delivery** — clients that disconnect briefly resume without missing events (per-channel ring buffer).
- **Backpressure** — slow consumers get a bounded queue; we drop or disconnect rather than blow up the heap.
- **Observability** — connection count, queue depth, dropped-message count, p50/p95 broadcast latency.

---

## 2. Out of scope (deliberately)

- Per-user authorization beyond a simple shared-secret token.
- Persistent storage of every event forever.
- Mobile push, email, SMS bridges.
- Cross-region replication.

---

## 3. Cast

| Actor | Role |
|---|---|
| **Producer** | Backend system that calls our REST publish endpoint when something interesting happens. Doesn't know who's listening. |
| **Notifier server** | Our Spring Boot app. Runs as 1+ identical instances. Holds WebSocket connections to clients. Coordinates via Redis. |
| **Redis** | Shared pub/sub bus between notifier servers. Never talks to clients. |
| **Client** | Browser/app holding a WebSocket open to one notifier server. |

**Talks-to matrix:**

- Producer → one notifier server (REST).
- Notifier servers ↔ Redis (pub/sub).
- Notifier servers ↔ clients (WebSocket).
- Redis never talks to clients. Clients never talk to Redis.

---

## 4. Architecture

```
                                                  ┌─► Notifier #1 ─► its WS clients
                                                  │
   Producer ─► Notifier #2 ─► Redis (broadcast) ──┼─► Notifier #2 ─► its WS clients
                                                  │
                                                  └─► Notifier #3 ─► its WS clients
```

Each notifier:

- Holds open WS sessions for some subset of users.
- Subscribes to all relevant Redis channels.
- On a Redis message, looks up which of *its* sessions care about that topic and pushes to each.

---

## 5. Wire protocol (WebSocket, JSON)

**Client → Server:**
```json
{ "type": "subscribe",   "topics": ["orders", "alerts"] }
{ "type": "unsubscribe", "topics": ["orders"] }
{ "type": "ack",         "eventId": 1042 }
{ "type": "replay",      "topic": "orders", "sinceEventId": 998 }
{ "type": "ping" }
```

**Server → Client:**
```json
{ "type": "event",       "topic": "orders", "eventId": 1043, "payload": {...}, "ts": 1714... }
{ "type": "error",       "code": "TOPIC_NOT_ALLOWED", "msg": "..." }
{ "type": "dropped",     "topic": "orders", "count": 12 }
{ "type": "replay_gap",  "topic": "orders" }
{ "type": "pong" }
```

**Auth:** `?token=...` query param at WS connect. `HandshakeInterceptor` validates; rejects with 401 before upgrade.

---

## 6. Components

| Component | Responsibility | Interface |
|---|---|---|
| `ProducerController` | REST `POST /events` → publish to Redis | `publish(topic, payload)` |
| `RedisFanoutSubscriber` | Listens on Redis channels, hands events to dispatcher | `onMessage(topic, event)` |
| `Dispatcher` | Looks up sessions for a topic, enqueues into each session's outbound queue | `dispatch(topic, event)` |
| `SessionRegistry` | `sessionId ↔ Set<topic>` mapping; subscribe/unsubscribe | `subscribe`, `unsubscribe`, `sessionsFor(topic)` |
| `OutboundQueue` (per session) | Bounded `ArrayBlockingQueue<Event>`; drop policy | `offer(event)` |
| `SessionWriter` (per session, virtual thread) | Drains queue → `session.sendMessage(...)` | runnable loop |
| `ReplayBuffer` (per topic) | Bounded ring of last N events with monotonic IDs | `append`, `since(eventId)` |
| `WsHandler` | Spring `TextWebSocketHandler`; lifecycle hooks | `afterConnectionEstablished`, `handleMessage`, `afterConnectionClosed` |
| `HandshakeInterceptor` | Validates token before WS upgrade | `beforeHandshake` |
| `MetricsRegistrar` | Connection count, queue depth, drop count, latencies | Micrometer + `/actuator/prometheus` |

Each is small (50-200 lines), focused, independently testable.

---

## 7. Concurrency model

- **Redis listener thread** (Lettuce-managed): receives → calls `dispatcher.dispatch(topic, event)`. Must not block.
- **Dispatcher**: synchronous, fast. Looks up sessions, calls non-blocking `queue.offer(event)` on each. Increments drop metric on full queue.
- **Per-session virtual thread**: one per WS connection. Loops `queue.take()` → `session.sendMessage(...)`. Java 21 virtual threads make per-session threads ~free (can have 100K of them).
- **Replay buffer**: append-only ring (`AtomicReferenceArray` + `AtomicLong` index).
- **Session registry**: `ConcurrentHashMap<String, Set<String>>` for sessions→topics + a reverse index `topic → Set<sessionId>`.

**Why this works:** Redis listener stays fast (no fanout work in its thread). Per-session writes never block dispatcher. Virtual threads remove the "thread per session is too expensive" objection.

---

## 8. Backpressure policy

- Per-session queue size: **256 events** (configurable).
- On `queue.offer(event) == false`:
  1. **Drop-oldest:** `queue.poll()`, then re-`offer()`. Increment `dropped` counter for that session.
  2. If 5 drops in 10 seconds → **disconnect** with code `1011`, reason `"slow consumer"`.
- Client receives a `dropped` notice frame *before* disconnect so it can reconnect with `lastEventId` and replay.

---

## 9. Replay buffer

- Per-topic, in-memory, ring of last **1024 events** (configurable per topic).
- Monotonic `eventId` from a per-topic `AtomicLong`.
- On client `replay { sinceEventId }`: scan ring for events with `id > sinceEventId`, push in order, then resume live tail.
- **Limitation (deliberate):** if more than 1024 events happened since disconnect, server returns `replay_gap` — client must resync via the application's source-of-truth API. We do NOT promise unbounded replay.

---

## 10. Failure modes (deliberately tested)

| Failure | Behavior |
|---|---|
| Redis goes down | Lettuce auto-reconnects; existing WS sessions stay alive but receive no new events; replay buffer (in-memory) survives; on Redis recovery, fanout resumes. |
| Slow consumer | Drop-oldest → disconnect at threshold → client reconnects + replays. |
| Producer crashes mid-publish | Event either reached Redis or didn't; we do not promise transactional publish. |
| Notifier node crashes (kill -9) | Connected clients get TCP RST → reconnect to surviving node → replay from last `eventId`. |
| Client briefly disconnects | Replay protocol catches them up if within ring buffer window. |

---

## 11. Testing strategy

| Layer | What | Tool |
|---|---|---|
| Unit | `OutboundQueue` drop policy, `ReplayBuffer` ring semantics, `SessionRegistry` subscribe/unsubscribe | JUnit 5 |
| Integration (single node) | Full publish → fanout → WS receive | Spring Boot Test + Testcontainers Redis |
| Integration (multi-node) | Publish to node A → received by client on node B | 2× Spring contexts in one JVM, shared Redis container |
| Failure | Pause/unpause Redis container mid-test | Testcontainers control APIs |
| Load | 10K concurrent subscribers, 1K events/sec, capture p50/p95 latency | Custom Java client w/ OkHttp WS |

**Coverage target:** 75%+ on logic classes (registry, queue, replay buffer). WS glue is exercised via integration tests, not unit-mocked.

---

## 12. Stack & rationale

| Choice | Why |
|---|---|
| Java 21 | Virtual threads — perfect fit for per-session writers; modern records, pattern matching. |
| Spring Boot 3.3 | Industry-standard SaaS framework; closes a major resume gap. |
| Raw WebSocket (not STOMP) | STOMP adds a layer that hides what we're doing. We want to *show* the WebSocket internals. |
| Lettuce | Default Redis client in `spring-boot-starter-data-redis`; non-blocking, supports pub/sub. |
| Maven | Already in README; no reason to switch. |
| Testcontainers | Real Redis in tests. No mocks for infra (mocks lie). |
| Micrometer + Prometheus | Standard Spring Boot metrics; one annotation enables `/actuator/prometheus`. |

---

## 13. Learn-while-build roadmap (20 days)

| Day | Build | Spring concept introduced |
|---|---|---|
| 1 | `start.spring.io` → `web` + `websocket` + `data-redis` + `actuator`. Hello-world `@RestController`. Run with `mvn spring-boot:run`. | `@SpringBootApplication`, startup, `application.yml` |
| 2 | `WsHandler extends TextWebSocketHandler`. Register via `WebSocketConfigurer`. Echoes messages. | `@Configuration`, `@Bean`, handler registration |
| 3 | `SessionRegistry` as `@Component`. Constructor-inject into `WsHandler`. Track sessions. | DI, constructor injection, `@Component` |
| 4 | In-process broker: a `Dispatcher` receives events from a test endpoint, pushes to all sessions for a topic. | `@PostConstruct`, scoped beans |
| 5 | Replace in-process with **Redis pub/sub**. `RedisMessageListenerContainer` + `MessageListener`. | Auto-config, `RedisTemplate` bean |
| 6 | Subscribe protocol: parse JSON `subscribe`/`unsubscribe` messages, update registry. | Jackson auto-config, `ObjectMapper` injection |
| 7 | `OutboundQueue` (bounded `ArrayBlockingQueue`) per session. Virtual-thread writer per session. | `spring.threads.virtual.enabled=true`, `Executors.newVirtualThreadPerTaskExecutor()` |
| 8 | Backpressure policy: drop-oldest, threshold disconnect, metrics. | Micrometer `MeterRegistry`, counters/gauges |
| 9 | `ReplayBuffer` per topic. Wire into dispatcher. | Pure Java; tests use `@SpringBootTest` slice |
| 10 | Reconnect protocol: `replay` message, heartbeat ping/pong via `@Scheduled`. | `@EnableScheduling`, `@Scheduled` |
| 11 | Token auth: `HandshakeInterceptor`, reject unauth'd before upgrade. | Interceptors, request-scoped data |
| 12 | Producer REST API: `POST /events {topic, payload}` → publish to Redis. | `@RestController`, `@PostMapping`, `@Valid` |
| 13 | Testcontainers Redis. Integration test: publish → assert WS client receives. | `@Testcontainers`, `@DynamicPropertySource` |
| 14 | Multi-node integration test: 2 contexts in one JVM, shared Redis. | Multiple `ApplicationContext` instances |
| 15 | Failure tests: pause Redis container, slow consumer simulation, kill client. | Testcontainers control APIs |
| 16 | Load test harness: standalone Java client, 10K subscribers, capture p50/p95. | (no Spring) |
| 17 | Tune queue sizes, virtual-thread settings; capture benchmark numbers. | profiling, JFR basics |
| 18 | Observability: Prometheus endpoint via Actuator, dashboard JSON. | `spring-boot-starter-actuator` |
| 19 | **Buffer day** for Spring rabbit-holes (auto-config, bean scopes, classpath conflicts). | — |
| 20 | README rewrite + architecture diagram + demo script + benchmark results. Push to GitHub. Link from resume. | — |

**Realistic timeline:** 20 calendar days at ~3-4 focused hours/day. Faster if you can do full days.

---

## 14. Resume bullet (target, post-completion)

> Built a multi-node WebSocket notification fanout service in Spring Boot 3 + Java 21 (virtual threads). Redis pub/sub for cross-node delivery, per-session bounded queue with drop-oldest backpressure, per-topic replay ring buffer for reconnecting clients. Validated under N concurrent subscribers, p95 broadcast latency Xms; tested kill-Redis / slow-consumer / kill-client failure modes.

(Numbers filled in from Day 17.)

---

## 15. Open decisions confirmed

- [x] Java 21 + Spring Boot 3.3 + virtual threads
- [x] Maven (per existing README)
- [x] Raw WebSocket (not STOMP)
- [x] Token auth: simple HMAC-signed JWT, generated by a tiny `/auth` endpoint (~30 LOC)
- [x] 20-day learn-while-build timeline (vs. original 14-day plan)

---

## 16. Demo script (final deliverable)

1. `docker compose up -d redis`
2. `mvn spring-boot:run` on port 8080.
3. `mvn spring-boot:run` on port 8081 (second instance).
4. Open browser tab, connect WebSocket to `ws://localhost:8080/ws?token=...`. Subscribe to `orders`.
5. From terminal: `curl -X POST localhost:8081/events -d '{"topic":"orders","payload":{"id":42}}'`.
6. Event appears in browser tab connected to **8080** — proves cross-node fanout via Redis.
7. Run `LoadClient.java`: 10,000 fake subscribers connect, publisher fires 1000 events/sec, console prints p50/p95 latency.
8. `docker pause redis` mid-load. Observe graceful degradation. `docker unpause redis`. Observe recovery.
