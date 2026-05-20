# Exchange Architecture — Interview Doc

> Three-service crypto exchange. Order ingress in Go, matching in Rust, stop-loss in Go. Kafka is the spine. Postgres/TiDB is the system of record. The matching engine is the hot heart — everything else is built so the engine can stay simple and fast.

---

## 1. High-Level Topology

```
                                  ┌──────────────┐
                                  │   Clients    │
                                  │ (HTTP REST)  │
                                  └──────┬───────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────┐
                  │            Order Service (Go)            │
                  │  ───────────────────────────────────────  │
                  │  • REST API (/order, /cancel, /modify)    │
                  │  • Auth, validation, balance check        │
                  │  • Hot in-memory balance engine           │
                  │  • Batch writer → 48-byte binary frames   │
                  │  • Prisma ORM → Postgres/TiDB             │
                  └────────────┬────────────────┬────────────┘
                               │                │
                Kafka:         │                │     Kafka:
        order-commands-<PAIR>  │                │  engine-events-<PAIR>
                               ▼                ▲
                  ┌──────────────────────────────────────────┐
                  │        Matching Engine (Rust)            │
                  │  ───────────────────────────────────────  │
                  │  • Single-threaded core per pair          │
                  │  • CPU-pinned, SCHED_FIFO (Linux)         │
                  │  • Slab + intrusive lists + hot/cold book │
                  │  • Lock-free SPSC ring buffers (ingress / │
                  │    egress)                                 │
                  │  • Zero hot-path allocation               │
                  └──────────────┬───────────────────────────┘
                                 │
                Kafka:           │
        engine-events-<PAIR>     ▼
              ┌──────────────────────────────────────────────┐
              │           Downstream Consumers               │
              │  ──────────────────────────────────────────  │
              │  • Order Service (state reconciliation)      │
              │  • Stop-Loss Service (Go, MySQL + Redis)     │
              │  • Market data / audit / analytics           │
              └──────────────────────────────────────────────┘
```

**One-line summary:** Order Service is the durable, user-facing front door; Matching Engine is the deterministic in-memory hot path; Stop-Loss is a side-car that watches the market data stream and injects stop orders back into the engine. Everything talks via Kafka in a binary wire format.

---

## 2. The Three Services

### 2.1 Order Service — Go

**Location:** [order-service/](order-service/)

**Responsibilities:**
- REST API: `POST /order`, cancel, modify.
- Authn/z, input validation, symbol/precision rules.
- Balance engine: in-memory map of user → balances, with a write-behind flush to the DB every 100 ms.
- Order persistence: every accepted order is inserted into Postgres/TiDB via Prisma before going to Kafka. The DB is the system of record.
- Batch writer: a pool of 32 parallel workers reads pending orders from the DB, encodes them into the 48-byte binary command frame, and publishes to `order-commands-<PAIR>`.
- Consumer of `engine-events-<PAIR>`: applies fills, cancels, rejects back to DB rows so the user-facing order state is always up to date.

**Tech stack:**
- Go, standard `net/http` + a router.
- Prisma migrations in [order-service/prisma/migrations/](order-service/prisma/migrations/).
- `confluent-kafka-go` (librdkafka) for Kafka.
- pprof on `:2904`, Swagger on `:2903`.

**Why a Go front end for a Rust engine:** Go is a great fit for the I/O-bound REST surface — connection pool, request validation, ORM. The matching engine doesn't deserve to be near HTTP. Splitting concerns means the engine stays in Rust without the noise of frameworks, and the front end can scale horizontally behind a load balancer without touching the engine.

### 2.2 Matching Engine — Rust

**Location:** [matching-engine/](matching-engine/)

**Responsibilities:** This is the topic of the second doc — see [MATCHING_ENGINE_DEEP_DIVE.md](MATCHING_ENGINE_DEEP_DIVE.md). One-line summary: it consumes commands from Kafka, applies them deterministically to an in-memory order book, and emits events back to Kafka. No DB, no locks, no allocator pressure on the hot path.

**Two operating modes** (env `ENGINE_MODE`):
- **Dedicated:** one process per pair. CPU-pinned, real-time scheduled. Lowest tail latency. Use for BTC/USDT, ETH/USDT.
- **Shared:** one process hosts many pairs, OS-scheduled. Higher density, slightly looser tails. Use for the long tail of illiquid pairs.

### 2.3 Stop-Loss Service — Go

**Location:** [stop-loss-service/](stop-loss-service/)

**Responsibilities:**
- Persists stop / take-profit / trailing-stop orders in MySQL.
- Subscribes to `engine-events-<PAIR>` for trade prints — that's the market data feed.
- Maintains a price-level book and trail anchors (in Redis) so trailing stops update with each tick.
- When a stop is triggered, encodes a regular order command and publishes to `stop-commands-<PAIR>`, which the matching engine consumes alongside the normal `order-commands` topic.
- Publishes `stop-events-<PAIR>` so the Order Service knows the stop was fired.

**Why separated from the matching engine:** stop orders depend on market state (last trade price), not on the book directly. Triggering logic doesn't need to be on the hot path — it can be slightly behind without impacting fairness. Keeping it out of the engine lets the engine stay tiny and deterministic.

---

## 3. The Kafka Spine

| Topic | Producer | Consumer | Payload |
|---|---|---|---|
| `order-commands-<PAIR>` | Order Service | Matching Engine | `NewOrder` / `Cancel` / `Modify`, 48 B |
| `stop-commands-<PAIR>` | Stop-Loss Service | Matching Engine | Same 48 B command frame |
| `engine-events-<PAIR>` | Matching Engine | Order Service, Stop-Loss, Market Data, Audit | `MakerFill` / `TakerFill` / `OrderPlaced` / `OrderCanceled` / `OrderRejected` / etc. |
| `stop-events-<PAIR>` | Stop-Loss Service | Order Service | Stop activation / completion |

**Why Kafka and not gRPC / queues:**
- **Durability + replay.** Engine restart? Replay from the last committed offset. New downstream consumer? Read from the beginning.
- **Fan-out for free.** One engine event reaches order-service, stop-loss, market-data, and audit without the engine knowing about any of them.
- **Backpressure boundary.** If the engine pauses (e.g. egress ring backpressure), commands queue in Kafka, not in memory — Kafka *is* the durable queue.
- **Per-pair partitioning.** Topic name carries the pair; an engine process for BTC-USDT only subscribes to `order-commands-BTC-USDT`. No cross-pair noise.

**Wire format (48 B, fixed):** see [matching-engine/src/types/command.rs:99–150](matching-engine/src/types/command.rs#L99-L150). All little-endian. Tag byte at offset 42 distinguishes NewOrder (0), Cancel (1), Modify (2). Fixed-size frames mean zero parsing overhead — it's a `memcpy` plus a few field reads.

```
NewOrder (tag=0, 48 B):
  [0..7]    client_order_id   u64 LE
  [8..15]   price (ticks)     u64 LE  (0xFFFF_FFFF_FFFF_FFFF = market buy)
  [16..23]  quantity          u64 LE
  [24..31]  timestamp_ns      u64 LE
  [32..39]  user_id (STP)     u64 LE
  [40]      side              0=Bid, 1=Ask
  [41]      order_type        0=Limit, 1=Market
  [42]      tag = 0x00
  [43..47]  reserved

Cancel (tag=1, 48 B):
  [0..7]    token             u64 LE   (gen<<32 | slab_index)
  [8..15]   client_order_id   u64 LE   (0 = match-any)
  [42]      tag = 0x01

Modify (tag=2, 48 B):
  [0..7]    token             u64 LE
  [8..15]   new_price         u64 LE
  [16..23]  new_quantity      u64 LE
  [24..31]  timestamp_ns      u64 LE
  [32..39]  client_order_id   u64 LE
  [42]      tag = 0x02
```

**Talking point for interviews:** fixed-size frames are deliberate. Variable-length protobuf would mean parsing on the hot path — and the hot path is microseconds-sensitive. Fixed-size 48 bytes means the Kafka ingress thread reads, validates the tag, and pushes the decoded `Command` enum into the ring buffer with no heap allocation.

---

## 4. Storage

- **Postgres / TiDB** (via Prisma): orders, trades, balances, stop_orders. System of record. Order Service is the only writer to the user-facing tables; it reconciles from `engine-events` to keep order rows accurate.
- **MySQL** (Stop-Loss): stop_orders table. Separate DB so the stop-loss service can scale independently and a hot read of the open-stops set is cheap.
- **Redis** (Stop-Loss): trail anchors — small, hot, key-value, perfect Redis use case.
- **Kafka** is the *event log*, not the system of record. Trades are written to Postgres for billing/reporting.
- **The matching engine has zero DB access.** This is the single most important architectural decision after "engine is single-threaded": no syscalls, no network I/O, no DB latency on the hot path.

---

## 5. End-to-End Request Flow

A limit buy on BTC/USDT, walked through:

1. Client sends `POST /order { side: buy, type: limit, price: 50000, qty: 0.1 }`.
2. **Order Service** auths the request, checks user balance against the in-memory balance engine, reserves the quote-currency amount.
3. Order row is inserted into Postgres with status `PENDING`. The DB row is the durable receipt.
4. The order is handed to a batch-writer worker, which encodes a 48-byte `NewOrder` command frame and `produce`s it to Kafka topic `order-commands-BTC-USDT`.
5. **Matching Engine** Kafka ingress thread polls the partition, decodes the frame into a `Command::NewOrder { … }`, and `try_push`es it into the ingress ring buffer.
6. **Engine core thread** (busy-spinning) `try_pop`s the command, walks the opposing side of the book, performs matches at maker prices (price-time priority), and writes `Event` records to the egress ring buffer.
7. **Kafka egress thread** drains the egress ring and `produce`s each event to `engine-events-BTC-USDT`.
8. Order Service consumes the events, updates the order row to `FILLED`/`PARTIALLY_FILLED`, updates the user's balance, persists trade rows.
9. Stop-Loss Service also sees the prints, may trigger stops, sends new commands into `stop-commands-BTC-USDT`.
10. Client polls `GET /order/:id` or subscribes to a websocket feed and sees the result.

**Latency budget per stage** (rough):
- HTTP → DB insert → Kafka publish: 5–20 ms (dominated by DB and Kafka commit)
- Kafka → ingress ring → engine core → egress ring: **microseconds** (this is the engine's domain)
- Egress ring → Kafka → Order Service DB update: 5–20 ms

The point of the engine being ultra-fast isn't that the user sees < 1 ms — they don't, because Kafka and Postgres dominate. The point is **deterministic matching** and **headroom**: at 100K+ orders/sec sustained, the engine is never the bottleneck.

---

## 6. Why This Architecture

**Why Rust for matching, Go for everything else?**
- Rust gives us zero-cost abstractions, no GC pauses, fine-grained memory control — exactly what a matching engine needs.
- Go gives us fast development for the I/O-bound surface and is plenty fast for HTTP, DB ORMs, and Kafka glue.
- Splitting languages is a feature, not a bug — each service uses what fits.

**Why microservices and not a monolith?**
- The engine has a fundamentally different scaling and reliability model. It runs CPU-pinned, real-time scheduled. You don't want an HTTP framework sharing a process with that.
- Stop-loss has different durability/latency requirements and a different team-velocity profile. Independent deploys.
- The Kafka boundary is the natural service boundary — the engine's API *is* a stream of events.

**Why Kafka and not a queue (Rabbit, NATS)?**
- We need durability and replay for crash recovery.
- We need multi-consumer fan-out (audit, market-data, stop-loss all consume engine events).
- Partition-per-pair maps perfectly to one-engine-per-pair.

**Why a hot/cold order book instead of a single BTreeMap?**
- See the deep-dive doc. Short version: 99% of activity happens within a few percent of the current price; flat-array O(1) lookup wins for that 99%, BTreeMap handles the long tail.

**What I'd reconsider:**
- The 48-byte frame leaves no room for variable-length client data. If we ever need richer metadata (algo tags, prime-broker routing hints) we'll need a v2 frame with a length prefix and a versioned tag.
- Stop-loss in Go is fine, but the trail-anchor logic in Redis adds a network round-trip per tick. At higher market-data rates, that becomes the bottleneck — an in-process trail engine would be faster.

---

## 7. Failure Model

| Failure | Behaviour |
|---|---|
| Order Service crashes | DB row stays in `PENDING`; on restart, batch writer picks up unprocessed rows and re-publishes to Kafka. Idempotent at the engine via `client_order_id`. |
| Matching engine crashes | In-memory book is lost. On restart, replay `engine-events-<PAIR>` from the last snapshot offset to rebuild the book. (Snapshotting is a planned addition — currently we replay from beginning of retention.) |
| Kafka broker outage | Order Service queues writes; clients see degraded acceptance. Engine pauses ingress. No data loss for committed orders. |
| Stop-Loss outage | Stops don't fire until recovery. New stops queue in MySQL. Engine and order flow unaffected. |
| Egress ring fills (slow Kafka egress) | Engine applies backpressure to ingress thread, which pauses Kafka consumption, which makes Kafka the queue. Engine never blocks the matching loop itself. |

---

## 8. Interview Soundbites

- "Three services around a Kafka spine: Go for I/O, Rust for matching, Go for stops. Each one solves a different problem at the right pace."
- "The engine has zero DB access. That's the architectural commitment that makes microsecond latency possible."
- "Fixed-size 48-byte binary command frames — no parsing, just a tagged dispatch. Variable-length serialization belongs above the engine, not inside it."
- "Kafka isn't a message bus, it's the durable event log. Replay is the recovery mechanism, not point-to-point retries."
- "Per-pair partitioning means one engine process serves one pair. Sharding is by symbol, not by thread."
- "We have two deployment modes — dedicated for the top pairs, shared for the long tail. Same binary, configured by env."
