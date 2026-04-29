# realtime-notifier

Event-driven fanout service that delivers real-time notifications over WebSockets, fanning out via Redis pub/sub with at-least-once replay and bounded-queue backpressure for slow consumers.

**Status:** in progress · **Target:** 14 days · **Stack:** Java · Redis Pub/Sub · WebSockets · Async

---

## Goal

Build a small, production-shaped notification service that I can defend in a system-design interview:

- **Fanout** — one published event reaches every subscribed client across N service nodes (Redis pub/sub channel).
- **Real-time** — push over WebSocket, not poll.
- **At-least-once delivery** — clients that disconnect briefly resume without missing events (replay from a per-channel ring buffer).
- **Backpressure** — slow consumers get a bounded outbound queue; we drop or disconnect rather than blow up the heap.
- **Observability** — connection count, queue depth, dropped-message count, p50/p95 broadcast latency.

Out of scope (deliberate): per-user authz beyond a simple token, persistent storage of every event forever, mobile push, email/SMS bridges.

---

## 14-Day Roadmap

| Day | Milestone | Outcome |
|-----|-----------|---------|
| 1 | Project scaffold + README | Maven project, package layout, this README |
| 2 | WebSocket endpoint (single node) | `/ws/notifications` accepts connections, echoes test events |
| 3 | In-process pub/sub fanout | One publisher → N local subscribers via an in-memory broker |
| 4 | Redis pub/sub integration | Replace in-memory broker with Redis channel; multi-node fanout works |
| 5 | Channel/topic model + subscribe API | Clients subscribe to specific topics, not the global firehose |
| 6 | Bounded outbound queue per client | Each WS session has a fixed-size queue; no unbounded growth |
| 7 | Backpressure policy (drop-oldest + disconnect) | Slow consumers are detected and handled, with metrics |
| 8 | At-least-once replay buffer | Per-channel ring buffer; clients reconnect with `lastEventId` and catch up |
| 9 | Reconnect + resume protocol | Heartbeats, idle timeouts, clean reconnect semantics |
| 10 | Auth handshake (token on connect) | Token-validated WS handshake; reject unauth'd before upgrade |
| 11 | Concurrency + load test harness | JMH or custom client generating N concurrent subscribers |
| 12 | Benchmarks (connections, msgs/sec, broadcast latency) | Numbers I can put in the resume bullet |
| 13 | Failure tests (Redis down, slow consumer, kill -9) | Documented behavior under each failure mode |
| 14 | Architecture diagram + final README | Excalidraw diagram, benchmark results, public-facing docs |

I'll mark days `done` in the linked tracker and update this table as I ship.

---

## Architecture (planned)

```
   ┌───────────────┐   publish    ┌──────────────────┐
   │  Producer API │ ───────────► │  Redis Pub/Sub   │
   └───────────────┘              │  channel:<topic> │
                                  └────────┬─────────┘
                                           │ subscribe
              ┌────────────────────────────┴───────────────────────┐
              │                                                    │
        ┌─────▼──────┐                                       ┌─────▼──────┐
        │  Node A    │                                       │  Node B    │
        │  (WS hub)  │                                       │  (WS hub)  │
        │            │                                       │            │
        │  per-conn  │                                       │  per-conn  │
        │  bounded   │                                       │  bounded   │
        │  queue     │                                       │  queue     │
        └─────┬──────┘                                       └─────┬──────┘
              │ WebSocket                                          │ WebSocket
        ┌─────▼──────┐  ┌────────────┐                       ┌─────▼──────┐
        │  client 1  │  │  client 2  │           ...         │  client N  │
        └────────────┘  └────────────┘                       └────────────┘
```

- Replay buffer lives per channel inside each node (small ring, configurable retention).
- Producers publish to Redis; nodes are stateless w.r.t. delivery and can scale horizontally.
- A client reconnecting with a `lastEventId` query param drains the ring buffer before live-tailing.

---

## Why this project

Listed on my resume as the "Real-time Notification Service." It backs the **event-driven architecture** + **concurrency** + **Redis pub/sub** claims, and pairs with the [distributed-kv-store](https://github.com/arundhaas1/distributed-kv-store) project to round out the distributed-systems story.

## License

MIT.
