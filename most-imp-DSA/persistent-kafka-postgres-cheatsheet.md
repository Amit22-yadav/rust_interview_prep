# Kafka + PostgreSQL + REST/Axum Cheat Sheet — Persistent Interview

Your goal: hold a **fluent intermediate conversation** on Kafka, RabbitMQ, PostgreSQL, REST APIs, Azure DevOps, and Docker/K8s. Not deep expert. Fluent enough that nothing in the JD makes you stumble.

Memorize the **phrases in bold quotes** — those are interview-ready talking points.

---

## 1. Apache Kafka

### Mental model

Kafka is a **distributed, partitioned, replicated commit log**. Not a queue — a log. Consumers read by offset, can replay history, multiple consumer groups can read the same data independently.

### Core concepts (must know cold)

| Term | What it is |
|---|---|
| **Broker** | A Kafka server. A cluster has many. |
| **Topic** | A named stream. Like a table name. |
| **Partition** | A topic is split into N partitions. Each partition is an ordered, immutable log. |
| **Offset** | Position of a record within a partition. Monotonically increasing. |
| **Producer** | Writes records. Chooses partition via key hash (or round-robin if no key). |
| **Consumer** | Reads records. Tracks its own offset. |
| **Consumer Group** | Set of consumers cooperating to read a topic. Each partition is consumed by exactly one consumer in the group. |
| **Replication factor** | How many brokers hold a copy of each partition. Typical: 3. |
| **Leader / Followers** | One broker is leader for a partition; others replicate. Reads + writes go to leader. |
| **ISR (In-Sync Replicas)** | Followers fully caught up to the leader. |

### Producer essentials

```rust
// Conceptual rdkafka producer setup
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "broker:9092")
    .set("message.timeout.ms", "5000")
    .set("acks", "all")              // wait for all ISRs — strongest durability
    .set("enable.idempotence", "true") // dedupe on producer-side retries
    .create()?;

producer.send(
    FutureRecord::to("orders")
        .key(&user_id.to_string())   // partition by user_id → ordering preserved per user
        .payload(&json_bytes),
    Duration::from_secs(5),
).await?;
```

**`acks` values**:
- `acks=0` — fire and forget. Fastest. Can lose data.
- `acks=1` — wait for leader only. Good balance.
- `acks=all` — wait for all ISRs. Strongest durability. Slightly slower.

### Consumer essentials

```rust
let consumer: StreamConsumer = ClientConfig::new()
    .set("bootstrap.servers", "broker:9092")
    .set("group.id", "order-processor")
    .set("enable.auto.commit", "false") // manual commit for at-least-once
    .set("auto.offset.reset", "earliest") // start from beginning if no offset
    .create()?;

consumer.subscribe(&["orders"])?;

while let Some(msg) = consumer.stream().next().await {
    let msg = msg?;
    process(&msg).await?;
    consumer.commit_message(&msg, CommitMode::Async)?;
}
```

### Delivery semantics — the canonical question

| Semantic | How |
|---|---|
| **At-most-once** | Commit before processing. Fast, can lose data on crash. |
| **At-least-once** | Process, then commit. **Default for most systems.** Duplicates possible — must make handler idempotent. |
| **Exactly-once** | Kafka transactions + idempotent producer + `read_committed` consumer. More complex; lower throughput. |

> **Talking point**: *"For 95% of use cases I default to at-least-once with idempotent handlers — it's simpler than exactly-once, and the handler-side idempotency (via a dedupe key or `INSERT ON CONFLICT`) gives you the same end-to-end behavior with far less operational complexity."*

### Partitioning strategy

- **Partition count** = max parallelism for consumers. 1 partition = 1 consumer max in a group.
- **Partition key** determines partition. **Same key → same partition → same consumer → ordering preserved per key.**
- Rule of thumb: partitions = expected throughput / max-throughput-per-partition. Often 12–48 for medium workloads.
- **Hot partition problem**: one key dominates traffic → one consumer overloaded. Mitigate with key sharding (`user_id + random suffix`) if ordering tolerates it.

### Rebalancing

When a consumer joins/leaves a group, Kafka **rebalances** partition assignments. During rebalance, consumption pauses briefly. Modern Kafka has cooperative rebalancing — only affected partitions pause.

### Common pitfalls (mention these to show depth)

1. **Long-running message handlers** → consumer misses heartbeat → kicked out → rebalance storm. Fix: keep handlers fast or use background processing.
2. **`auto.commit=true` + crash** → can lose unprocessed messages. Always manual-commit for at-least-once.
3. **Too few partitions** → can't scale consumers. Hard to repartition after the fact.
4. **No retention policy** → disk fills up. Set `retention.ms` per topic.

### Kafka vs RabbitMQ — be ready

| | Kafka | RabbitMQ |
|---|---|---|
| **Model** | Log (replayable) | Queue (consumed once) |
| **Throughput** | Very high (100K+ msg/sec per node) | Moderate (10K–30K msg/sec) |
| **Ordering** | Per-partition | Per-queue |
| **Routing** | Simple (topic, partition) | Rich (direct, topic, fanout, headers exchanges) |
| **Replay** | Yes — rewind offset | No — once consumed, gone |
| **Best for** | Event streaming, log aggregation, CDC | Task queues, complex routing, request/reply |

> **Talking point**: *"Kafka if you need a replayable event log with high throughput and per-key ordering. RabbitMQ if you need rich routing patterns, fanout to many consumers, or traditional work-queue semantics with priority and TTL."*

---

## 2. PostgreSQL

### Indexing — the most common interview topic

**B-tree** (default): equality + range queries on ordered data. Most columns use this.

**Hash**: equality only. Rarely used — B-tree handles equality fine.

**GIN** (Generalized Inverted Index): for arrays, JSONB, full-text search. `WHERE tags @> ARRAY['x']`.

**GiST**: geospatial, range types, custom indexable operators.

**BRIN** (Block Range Index): cheap, lossy index for huge tables with natural ordering (e.g., time-series append-only).

> **Talking point**: *"99% of my indexes are B-tree. I reach for GIN when I need to index JSONB or array containment, and BRIN for time-series append-only tables where rows are clustered by insert order."*

### Reading EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'open';
```

Look for:
- **Seq Scan** on a big table = missing index. Bad.
- **Index Scan** = using an index. Good.
- **Index Only Scan** = data came entirely from the index (covering). Great.
- **Bitmap Index Scan + Bitmap Heap Scan** = used an index but had to fetch many rows.
- **Hash Join / Merge Join / Nested Loop** = join strategy. Nested loop on big tables is usually bad.

The numbers at each node: `cost=X..Y rows=N width=W (actual time=A..B rows=N loops=L)`. Big gap between estimated and actual rows → stale statistics → run `ANALYZE`.

### Query optimization tips (memorize 5)

1. **Index foreign keys** — Postgres doesn't auto-index FKs. Missing FK indexes cause slow joins and DELETE cascades.
2. **Composite index order matters** — `(user_id, created_at)` works for filters on `user_id` alone and `user_id + created_at`, but not for `created_at` alone.
3. **`LIMIT` without `ORDER BY` is non-deterministic** — always pair them. For paginated APIs prefer **keyset pagination** over `OFFSET`.
4. **`SELECT *` in production code** is a smell — explicit columns enable index-only scans.
5. **Watch for N+1**: fetching parent then looping to fetch children. Fix with JOIN or `WHERE id IN (...)`.

### Transactions + isolation

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;  -- default
-- ...
COMMIT;
```

| Level | What you get |
|---|---|
| **READ UNCOMMITTED** | (Postgres treats this as READ COMMITTED.) |
| **READ COMMITTED** | See committed data. Non-repeatable reads possible. **Default.** |
| **REPEATABLE READ** | Snapshot at txn start. Phantom reads prevented in PG. |
| **SERIALIZABLE** | Full serializability via SSI (Serializable Snapshot Isolation). Can throw serialization failures — must retry. |

> **Talking point**: *"For most CRUD I stay at READ COMMITTED. For multi-step business operations like 'transfer money', I either use SERIALIZABLE with retry-on-conflict, or take row-level locks via `SELECT ... FOR UPDATE` in a single transaction."*

### Idempotency in Postgres — the canonical pattern

```sql
INSERT INTO payments (idempotency_key, user_id, amount)
VALUES ($1, $2, $3)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id;
```

If the row already exists, no insert happens, `RETURNING id` returns nothing — caller knows the request was a duplicate.

### Connection pooling

Postgres process-per-connection is expensive. Always pool.

- **Application-level pool**: `bb8`, `deadpool-postgres`, `sqlx::PgPool`. Typical size: 10–30 per service instance.
- **External pooler**: `pgbouncer` in front of Postgres. Use `transaction` pooling mode for max efficiency.

> **Talking point**: *"For a Rust service I'd use `sqlx::PgPool` with 10–20 connections, and put pgbouncer in front of Postgres in transaction-pooling mode if I have many service instances — that lets each instance maintain a small pool while pgbouncer multiplexes onto a much smaller real-Postgres connection pool."*

### Partitioning large tables

For tables expected to exceed ~100M rows:
```sql
CREATE TABLE orders (id BIGSERIAL, user_id INT, created_at TIMESTAMPTZ, ...)
PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_06 PARTITION OF orders
FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

Benefits: smaller indexes per partition, fast partition pruning on date queries, easy archival (just drop old partitions).

### Common pitfalls

1. **Long-running transactions hold locks** → blocking writers, bloating tables. Keep transactions short.
2. **Forgotten `VACUUM`** → table bloat. Autovacuum tunes this; monitor `pg_stat_user_tables.n_dead_tup`.
3. **Over-indexing** → write amplification. Every INSERT updates every index. Drop unused indexes.
4. **`WHERE function(col) = X`** → can't use index on `col`. Rewrite or create expression index.

---

## 3. REST APIs in Rust with Axum

### Minimal Axum service

```rust
use axum::{routing::get, routing::post, Router, Json, extract::{State, Path}};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use std::sync::Arc;

#[derive(Serialize, Deserialize)]
struct Order { id: i64, symbol: String, qty: i64 }

#[derive(Clone)]
struct AppState { db: PgPool }

async fn get_order(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i64>,
) -> Result<Json<Order>, axum::http::StatusCode> {
    let row = sqlx::query_as!(
        Order, "SELECT id, symbol, qty FROM orders WHERE id = $1", id
    ).fetch_optional(&state.db).await
     .map_err(|_| axum::http::StatusCode::INTERNAL_SERVER_ERROR)?
     .ok_or(axum::http::StatusCode::NOT_FOUND)?;
    Ok(Json(row))
}

async fn create_order(
    State(state): State<Arc<AppState>>,
    Json(body): Json<Order>,
) -> Result<Json<Order>, axum::http::StatusCode> {
    let id = sqlx::query_scalar!(
        "INSERT INTO orders (symbol, qty) VALUES ($1, $2) RETURNING id",
        body.symbol, body.qty
    ).fetch_one(&state.db).await
     .map_err(|_| axum::http::StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(Order { id, ..body }))
}

#[tokio::main]
async fn main() {
    let db = PgPool::connect("postgres://...").await.unwrap();
    let state = Arc::new(AppState { db });
    let app = Router::new()
        .route("/orders/:id", get(get_order))
        .route("/orders", post(create_order))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### REST best practices to mention

- **Resource-based URLs**: `/orders/123`, not `/getOrder?id=123`
- **HTTP verbs**: GET (read), POST (create), PUT (replace), PATCH (partial), DELETE
- **Status codes**: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal, 503 Unavailable
- **Versioning**: `/v1/orders` or `Accept: application/vnd.api.v1+json` header
- **Pagination**: keyset (`?after=cursor&limit=20`) > offset (`?offset=100&limit=20`)
- **Idempotency**: clients send `Idempotency-Key` header, server dedupes via Postgres unique constraint
- **Observability**: `tracing` + `tracing-subscriber` + Prometheus metrics via `metrics` crate

> **Talking point**: *"For middleware I'd layer in this order: trace_id → request logger → auth → rate limiter → cors → app handler. Each is a Tower layer in Axum, composable and testable."*

---

## 4. Azure DevOps + Azure Repos

You don't have deep Azure experience. Don't fake it. Show **awareness + transferable knowledge**.

### What Azure DevOps is (one-paragraph mental model)

Microsoft's hosted DevOps suite, equivalent to GitLab. Five components:
1. **Azure Repos** — Git hosting (like GitHub). Supports PRs, branch policies, code review.
2. **Azure Pipelines** — CI/CD. YAML-defined pipelines, agents (Microsoft-hosted or self-hosted), build + release stages.
3. **Azure Boards** — Jira-equivalent. Work items, sprints, kanban.
4. **Azure Artifacts** — package feeds (NuGet, npm, Maven, generic).
5. **Azure Test Plans** — test management.

### What a Rust CI pipeline looks like (azure-pipelines.yml sketch)

```yaml
trigger: [main]

pool:
  vmImage: ubuntu-latest

steps:
  - task: Cache@2
    inputs:
      key: 'cargo | "$(Agent.OS)" | Cargo.lock'
      path: ~/.cargo

  - script: |
      curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
      source $HOME/.cargo/env
      rustc --version
    displayName: 'Install Rust'

  - script: cargo fmt -- --check
    displayName: 'Format check'

  - script: cargo clippy -- -D warnings
    displayName: 'Lint'

  - script: cargo test --all-features
    displayName: 'Run tests'

  - script: cargo build --release
    displayName: 'Build release'

  - task: Docker@2
    inputs:
      command: 'buildAndPush'
      repository: 'myorg/myservice'
      tags: '$(Build.BuildId)'
```

### Honest framing in the interview

> **Talking point**: *"My production CI work has been GitHub Actions and GitLab CI — the Azure DevOps equivalents map cleanly: Azure Repos = Git hosting, Pipelines = YAML CI/CD, Boards = work tracking. The mental model transfers directly; I'd be productive in Azure DevOps within a week."*

---

## 5. Docker + Kubernetes

### Docker for Rust services — multi-stage build

```dockerfile
# Builder
FROM rust:1.75 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

# Runtime — small image
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myservice /usr/local/bin/myservice
EXPOSE 3000
CMD ["myservice"]
```

**Talking points**:
- **Multi-stage** drops the 1GB+ Rust toolchain from final image — final is ~100MB
- For ultra-small: build statically linked binary with `musl` target, base image `alpine` or `distroless` (~10MB total)
- Use `.dockerignore` to skip `target/`, `.git/`, etc.

### Kubernetes basics (be honest — say "working knowledge")

Concepts you should know:
- **Pod** — smallest unit, one or more containers
- **Deployment** — declarative spec for N replicas of a pod
- **Service** — stable DNS name + load balancing across pods
- **Ingress** — external HTTP routing
- **ConfigMap / Secret** — config and credentials injected as env vars or files
- **HPA (Horizontal Pod Autoscaler)** — scale replicas on CPU/memory/custom metrics
- **Liveness probe** — restart if unhealthy
- **Readiness probe** — only route traffic when ready

> **Talking point**: *"I have working knowledge of Kubernetes — Docker is my daily driver, K8s I understand at the deployment/service/ingress level. For production rollouts I'd lean on a platform team or use managed K8s (AKS on Azure) rather than running the cluster myself."*

---

## 6. System design — two scenarios to rehearse out loud

### Scenario A: "Design a service that ingests 100K events/sec from Kafka, validates, writes to Postgres, exposes a query API."

**Structure your answer (12 min)**:

1. **Clarify** (1 min): event size? avg vs p99 throughput? read:write ratio on the query API? latency requirements?
2. **High-level architecture** (3 min):
   - Kafka topic with 32 partitions, replication 3
   - Rust consumer service, N replicas, each owning a subset of partitions
   - Each replica: Kafka consumer → validation → batched Postgres insert → ack
   - Read API: separate Axum service hitting Postgres + read replica + Redis cache for hot keys
3. **Throughput math** (2 min):
   - 100K/sec ÷ 32 partitions = ~3.2K/sec per partition
   - With 8 consumer replicas (4 partitions each) → ~13K events/sec per replica
   - Batched inserts: 500 events/batch, 25 batches/sec, ~1ms each = very feasible
4. **Backpressure + failure** (2 min):
   - Bounded channel between consumer task and DB-writer task
   - On DB unavailability: pause consumption (Kafka consumer pause/resume) instead of dropping
   - Dead-letter topic for validation failures
   - Idempotency via unique constraint on `(event_id)` so duplicate-replay is safe
5. **Observability** (2 min):
   - Prometheus metrics: consumer lag, batch size, insert latency, validation failures
   - Tracing per-event with `event_id` as trace_id
   - Alert on consumer lag > 30s
6. **Scaling levers** (2 min):
   - Add partitions + replicas linearly
   - Partition Postgres by time if write volume outgrows single node
   - Move to columnar (ClickHouse, TimescaleDB) for analytics queries

### Scenario B: "p99 latency on a Rust + Postgres service just went 10× higher. How do you diagnose?"

**Structure your answer (10 min)**:

1. **Confirm the signal** (1 min): "Is this from real users, synthetic monitoring, or internal logs? Has anything deployed recently?"
2. **Layered diagnosis** (5 min) — top to bottom:
   - **Client side**: did request volume spike? Is it a thundering herd from one client?
   - **Network/load balancer**: any retries? Connection saturation?
   - **App-level Rust metrics**: which endpoints? p99 went up because more requests, or because each request slower?
   - **`tokio-console` or `console-subscriber`**: are tasks blocking the runtime? Long-running futures?
   - **Database**: connection pool saturation? Long-running queries? Lock contention? Check `pg_stat_activity` for `wait_event`.
   - **Postgres internals**: `EXPLAIN ANALYZE` on suspect queries. Recent stats change? Missing index?
3. **Most-likely root causes** (3 min):
   - Connection pool exhausted (a downstream slowdown cascades)
   - A query that was fast on small data is slow now that the table grew (missing index or stale plan)
   - Lock contention from a new feature (long transaction holding row locks)
   - GC-like pauses — in Rust this is more often allocator contention or oversubscribed runtime
4. **Quick mitigations vs root fix** (1 min): scale up replicas as triage, then root-cause and patch.

---

## 7. Quick-fire facts to memorize

These come up as one-liners and you should answer in <10 seconds.

- **Kafka default port**: 9092
- **Postgres default port**: 5432
- **RabbitMQ default port**: 5672 (AMQP), 15672 (management UI)
- **Redis default port**: 6379
- **HTTP/1.1 vs HTTP/2**: H2 is multiplexed (multiple requests on one TCP connection), binary framing, header compression. For internal gRPC use H2; for public APIs H1 is still fine.
- **What's an idempotent HTTP method?** GET, PUT, DELETE, HEAD, OPTIONS. POST and PATCH are not.
- **What's a 502 vs 503 vs 504?** 502 Bad Gateway (upstream gave a bad response), 503 Service Unavailable (overloaded or maintenance), 504 Gateway Timeout (upstream didn't respond in time).
- **JSON vs Protobuf**: JSON is human-readable but verbose; Protobuf is binary, schema-defined, much smaller and faster to parse — used in gRPC.
- **What's a circuit breaker?** Open after N failures, fail fast for cooldown, half-open to test, close on success. Prevents cascading failure.
- **What's CDC (Change Data Capture)?** Stream DB changes (inserts/updates/deletes) to a topic. Debezium is the standard tool. Postgres CDC = logical replication slot.

---

## 8. The 5 phrases to drop in any technical conversation

Memorize these. Drop one or two in your interview and it lands like a senior engineer:

1. *"For money I'd use `rust_decimal` rather than `f64` — floating-point errors compound badly in financial sums."*
2. *"Idempotency at the API layer via an `Idempotency-Key` header + unique constraint in Postgres is the simplest pattern that survives retries."*
3. *"Default to at-least-once delivery with idempotent handlers. Exactly-once via Kafka transactions is real but rarely worth the operational cost."*
4. *"I'd add observability before optimization — `tracing` + Prometheus + structured logs. You can't fix what you can't measure."*
5. *"For Postgres performance my first move is always `EXPLAIN ANALYZE` on the slow query — most issues come down to missing indexes, stale statistics, or wrong join order."*

---

## Final note

You don't need to be an expert in Kafka or Postgres. The interviewer is checking that you can **think in distributed-systems terms** and that the **JD vocabulary doesn't trip you up**. Your matching engine work, consensus background, and async Rust depth are your real assets — Kafka/Postgres are just the table stakes you need to make sure don't become disqualifiers.

You've got this.
