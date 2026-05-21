# JD-Driven Interview Prep — Low-Latency Rust Trading Engineer

> This doc maps the JD line by line to what you've actually built, gives you the exact questions they'll ask given this wording, and arms you with crisp answers. Read after the deep-dive doc; use this to anticipate their angle of attack.

---

## The JD, Decoded

| JD phrase | What they're really probing |
|---|---|
| "Architect and optimize low-latency Order Book, spot + futures engine" | Do you understand book data structures, matching, and latency? **(Your matching engine answers this directly.)** |
| "Production-grade Rust" | Idiomatic Rust, no `unwrap()` everywhere, error handling, `unsafe` discipline, testing |
| "Fearless concurrency, rayon, tokio" | Send/Sync, async runtime model, when to use which tool — and when NOT to use them |
| "Asynchronous, parallelized, event-driven" | Tokio basics + when async is the wrong answer |
| "Scale to thousands of trading pairs without blocking" | Sharding, per-pair isolation, dedicated vs shared modes |
| "System profiling, memory management" | `perf`, `flamegraph`, `dhat`, page faults, NUMA, cache behaviour |
| "tokio task scheduling, minimize bottlenecks and race conditions" | Work-stealing, blocking-in-async, cancellation safety, lock contention |
| "Deterministic, crash-free matching engines" | Single-threaded core, replay, snapshotting, no panics on the hot path |
| "Highly parallel exchange connectivity layers" | Kafka/WebSocket gateways, fan-out, backpressure |

**Read between the lines:** they want someone who can think clearly about *when* async helps and *when* it hurts. The answer "I'll wrap everything in `async`/`await`" is wrong for matching. The answer "I'll put `tokio::Mutex` everywhere" is wrong for the hot path. A senior engineer knows the difference.

---

## Section 1 — Bullet-by-Bullet Talking Points

### Bullet 1: "Architect and optimize low-latency Order Book, spot + futures engine"

**Lead with your matching engine.** Don't wait for them to ask — open with it in your intro.

**Soundbite:** "I'm currently building a Rust matching engine for a CEX. Single-threaded core per pair, lock-free SPSC ring buffers for ingress/egress, slab allocator with generation-stamped tokens for O(1) cancel, hot/cold hybrid order book — flat array for the body of the book, BTreeMap for the tails. Sustains 728K orders/sec on commodity hardware."

**Spot vs futures — be ready:**
- **Spot:** the engine you've built. Match a buy against an ask, settle two assets, done.
- **Futures:** same matching engine logically, but with extra layers around it:
  - **Mark price / index price** feed (separate service) used for liquidations and unrealised PnL.
  - **Funding rate** computation (separate service) that periodically settles positions.
  - **Liquidation engine** that submits market orders to the matching engine when maintenance margin is breached.
  - **Position manager** that tracks open positions and margin per user.

**Honest framing:** "I built the matching layer for spot. Futures uses the same matching engine — the additional logic for margin, funding, and liquidation lives in surrounding services, not in the matching engine. The engine itself is asset-class-agnostic; it just matches orders."

### Bullet 2: "Production-grade Rust"

What "production-grade" means to interviewers:
- **No `unwrap()` outside tests and provable-impossible paths.** Use `expect("...")` with a reason at minimum.
- **`Result` propagation with `?` and `thiserror`** at lib boundaries, `anyhow` at app boundaries.
- **`unsafe` is wrapped in safe APIs, with SAFETY comments documenting every invariant.**
- **Tests:** integration tests against the public API, unit tests on the tricky bits, property tests for invariants.
- **Tracing/structured logging** via `tracing`, not `println!`.
- **Metrics:** Prometheus or similar; latency histograms, throughput counters, ring occupancy.
- **No `Box<dyn Error>` panicking on a hot path.** Errors are events, not exceptions.

**Soundbite:** "In our engine, every `unsafe` block has a SAFETY comment. The matching loop never calls `unwrap`. Errors are pushed as `OrderRejected` events with explicit reason codes, not panics. The one place we panic is egress-ring-full, because that's a correctness violation and we want it loud."

### Bullet 3: "Fearless concurrency, rayon, tokio"

This is the key bullet for them. Be precise about *which tool when*.

| Tool | When you use it | When you don't |
|---|---|---|
| **`std::thread`** | The matching engine core thread — needs CPU pin + RT priority, runs forever | Anywhere with many short-lived tasks |
| **`tokio`** | Kafka producers/consumers, HTTP handlers, gateways, websocket fanout — I/O-bound | Inside the matching engine; tasks that block; CPU-bound work |
| **`rayon`** | Embarrassingly parallel CPU work — batch processing, parallel snapshot serialisation, parallel pre-validation, analytics jobs | Inside the matching engine hot path; I/O-bound work |
| **Channels (`crossbeam`, `flume`, custom SPSC)** | Crossing thread boundaries when you want zero contention | Inside a single thread; when ordering doesn't matter |
| **`tokio::sync` primitives** | Sharing state between async tasks | Anywhere a thread can spin instead |

**Fearless concurrency:** the catchphrase from the Rust book. Translate it: "the compiler refuses to compile a data race." That's `Send`/`Sync`. Be ready to explain both.

**Why we *don't* use rayon in the matching engine:** rayon parallelises by splitting work; matching has a strict total order. You can't `par_iter` across a sequence of price-time-priority events. We use rayon outside the engine — for example, parallel snapshot serialisation, batch backtests, or aggregating per-pair metrics.

**Why we don't use tokio in the matching engine core:** the core needs to *busy-spin*. Tokio is built around "yield when nothing to do" — exactly what we don't want. A busy-spinning OS thread on a pinned core gives us sub-microsecond pickup latency; a tokio task on a shared runtime gives us tens of microseconds because of the scheduler.

**Soundbite:** "Tokio for I/O at the edges, std::thread for the matching core with CPU pinning, rayon for embarrassingly parallel offline work — snapshots, backtests, analytics. I treat them as three different tools, not interchangeable."

### Bullet 4: "Asynchronous, parallelized, event-driven"

The trap: an interviewer who's heard "async Rust" too many times will assume the matching engine is async. Don't let them — calmly correct the framing.

**Correct framing:** "The *exchange* is event-driven and async at every boundary. Kafka is the event log. HTTP handlers, gateways, fanout — all async on tokio. But the matching engine itself is *not* async. Async is the right answer for I/O-bound work where waiting dominates execution; matching is CPU-bound and latency-sensitive, so a busy-spinning dedicated thread wins. Wrapping it in `async fn` would add a state machine, a scheduler poll, and a `Pin<Box<dyn Future>>` for no benefit."

### Bullet 5: "Scale to thousands of trading pairs without blocking"

This is the biggest open-ended question they can ask. Here's the answer.

**Per-pair isolation strategy — two modes:**

**Dedicated mode** (top pairs):
- One process per pair (`PAIRS=BTC-USDT`).
- CPU-pinned to a specific core.
- SCHED_FIFO real-time priority.
- ~512 MB arena per pair.
- For ~20 top pairs on a single host with 32 cores, you can run all of them dedicated.

**Shared mode** (long tail):
- One process hosts many pairs (`PAIRS=DOGE-USDT,SHIB-USDT,...`).
- OS-scheduled, not pinned.
- Per-pair config overrides via env (`DOGE_USDT_ARENA_CAPACITY=...`).
- Slightly looser tail latency, much higher density — thousands of pairs per process.

**For "thousands of pairs" specifically:**
- Top ~50 pairs in dedicated processes (one per pair, CPU-pinned).
- Remaining ~2,000 pairs sharded across N shared-mode processes (e.g., 8 processes × 250 pairs each).
- Per-pair Kafka partitioning means each engine subscribes only to its own pair's topic — no cross-pair noise.
- Each pair has its own ingress ring, egress ring, slab, order book. **No shared state between pairs.**

**Why "without blocking":**
- Per-pair processes can't interfere with each other — separate address spaces, separate cores in the dedicated case.
- Even within a shared-mode process, each pair has its own `OrderBook` and `Slab`; the engine thread is per-pair.
- Kafka partitioning gives us per-pair backpressure isolation — a slow consumer on ETH-USDT can't pause BTC-USDT.

**Soundbite:** "Sharding by symbol. Top pairs get dedicated CPU-pinned processes; long tail gets co-tenanted shared processes. No cross-pair shared mutable state, no cross-pair locking. Adding pairs is horizontal — you spin up another process and update the gateway routing table."

### Bullet 6: "System profiling, memory management"

Tools you should mention by name, even if you've used them lightly:

| Tool | What for |
|---|---|
| **`perf`** (Linux) | CPU sampling, cache misses, branch mispredicts |
| **`flamegraph`** (perf + Brendan Gregg's tooling, `cargo-flamegraph`) | Visualise where time is spent |
| **`dhat`** | Heap profiling — alloc count, allocated bytes per call site |
| **`valgrind --tool=cachegrind`** | Cache simulation (slow, but exact) |
| **`criterion`** | Microbenchmarks with statistical rigour |
| **`tokio-console`** | Async task introspection |
| **`heaptrack`** | Allocation timeline |
| **`/proc/<pid>/status`** | VmRSS, VmSwap — confirm pre-fault worked |

**Memory management talking points:**
- Pre-fault arena at startup (no minor page faults during trading).
- `madvise(MADV_HUGEPAGE)` for the arena — fewer TLB misses.
- Per-pair memory isolation — each pair owns its arena.
- No `Box`/`Arc`/`Rc` on the hot path; all references are by slab index.
- Reused buffers (`Vec<Fill>` cleared between commands) so amortised alloc is zero.

**Soundbite:** "We profile with `perf` and `flamegraph` for CPU, `dhat` for allocations. Pre-fault the arena, hugepage advise, and verify with `VmRSS` after startup. The criterion bench gates regressions in CI — if median latency moves more than a threshold, the build fails."

### Bullet 7: "tokio task scheduling, minimize bottlenecks and race conditions"

This is where they'll ask the async-pitfalls questions.

**Top tokio pitfalls — know these cold:**

1. **Blocking inside async.** Reading a file with `std::fs::File::open`, calling `std::sync::Mutex::lock` for a long time, doing CPU-heavy work in an `async fn` — all of these block a worker thread and starve other tasks scheduled on the same worker. Fix: `tokio::task::spawn_blocking` for sync/CPU work; `tokio::fs` for files.

2. **Holding a sync mutex across `.await`.** Classic deadlock pattern. Worker thread parks while holding the lock; another task on the same worker tries to acquire the same lock; deadlock. Fix: `tokio::sync::Mutex`, or scope the critical section to *not* span an await.

3. **Unbounded channels.** Memory grows without bound under load. Fix: bounded `mpsc::channel(N)` so producers backpressure when consumers are slow.

4. **`select!` cancellation safety.** `select!` drops the losing branch's future. If that future has consumed external state (read bytes from a socket, taken an item out of a queue), you lose data. Fix: only put cancel-safe futures in `select!`; check Tokio's docs for each method's cancellation safety.

5. **`tokio::spawn` requires `Send + 'static`.** Capturing a non-`Send` type like `Rc` across an `.await` makes the future `!Send`. Fix: use `Arc`, or use `tokio::task::spawn_local` on a `LocalSet`.

6. **Forgetting to `.await`.** A future that's never awaited does nothing. The compiler warns; pay attention.

**Race conditions:**
- In sync code: data race = two threads, same memory, at least one writer, no synchronisation. Rust's `Send`/`Sync` catches these at compile time.
- In async code: data races aren't possible (single thread per task by default), but *logical* races are — two tasks reading and writing the same shared state across awaits. Fix: actor pattern (one task owns the state; others send messages) or proper async mutex with a clean critical section.

**Soundbite:** "The most common tokio bug I see is holding `std::sync::Mutex` across `.await` — it deadlocks the worker. The second most common is unbounded channels — memory goes to the moon. Past that, the actor pattern solves most concurrency questions: one owner per piece of state, talk to it via a channel."

### Bullet 8: "Deterministic, crash-free matching engines"

**Deterministic:**
- Single-threaded core. Same input sequence → same output sequence, always.
- Integer-only arithmetic (no floats, no NaN, no rounding ambiguity).
- No `HashMap` iteration order dependence on the hot path (and where we do use HashMap, we don't iterate it).
- Replay is the recovery model — replay `engine-events` from offset N and you get the same book.

**Crash-free:**
- No `unwrap()` on the hot path.
- No `panic!` except the one deliberate egress-ring-full panic (correctness fail).
- `Result` propagation with explicit reject events for every failure mode (`InvalidParams`, `ArenaFull`, `NoLiquidity`, `PartialFillDiscarded`).
- Defensive bounds checking via `validate(token)` on every token-bearing command.
- Tested with 1000+ integration tests in `tests/matching_tests.rs`.

**Soundbite:** "Determinism is a property of the architecture, not just the code. Single-threaded core means there's exactly one event ordering. Reject events instead of panics mean a malformed command doesn't crash the process. Replay is the recovery mechanism — same input log gives the same output state, every time."

### Bullet 9: "Highly parallel exchange connectivity layers"

What "connectivity" means in a CEX:
- Inbound: REST API, WebSocket order entry, FIX gateway.
- Outbound: market data WebSocket fan-out, FIX drop copy, public API trade feed.
- Internal: Kafka producers/consumers.

**Parallelism strategy:**
- Each gateway process is async/tokio-based. Per-connection task, bounded mpsc to the upstream Kafka producer.
- Market data fan-out uses `tokio::sync::broadcast` — slow subscribers can lag, with a "gap" event and resnapshot on the slow consumer side.
- Connectivity layer is *not* in the matching engine — that's the point. The engine speaks Kafka; the connectivity is bridges in front of and behind the engine.
- Per-connection backpressure prevents one slow client from affecting others.

**Soundbite:** "Connectivity is async/tokio with bounded channels everywhere. Each connection is a tokio task. Market data fan-out uses tokio's broadcast channel with a documented lag-and-resnapshot semantics — we never block matching for a slow subscriber. The matching engine has *one* output channel; everything past that point is the fanout layer's problem."

---

## Section 2 — The Questions This JD Predicts

These are the questions a hiring manager who wrote that JD will ask. Have answers ready.

### Q1. "Walk me through how an order flows through your system."

(Same as in the previous prep doc — Section 3 Q1 of `RUST_INTERVIEW_PREP.md`. Use the same answer.)

### Q2. "How do you architect for thousands of pairs?"

Use the dedicated/shared split from Bullet 5 above. Be explicit: "Top pairs dedicated and pinned; long tail co-tenanted in shared processes. Per-pair state — no shared mutable book between pairs."

### Q3. "When would you use tokio vs std::thread vs rayon in this system?"

(See Bullet 3 table.) Lead with: "Different tools for different shapes of work. Tokio for I/O-bound concurrency at the edges. std::thread for the matching core because it needs to busy-spin on a pinned RT core. Rayon for embarrassingly parallel offline work — never on the hot path."

### Q4. "Why isn't your matching engine async?"

**Answer:** "Async is the right answer for I/O-bound work where you have many concurrent waits. The matching engine is CPU-bound and latency-sensitive — there's nothing to wait on. The cost of being async is a state machine, scheduler poll overhead, and a Pin<Box> for the future. The benefit is yielding while waiting — but we never wait. So async is pure overhead for this workload. A busy-spinning OS thread on a pinned core with SCHED_FIFO is the right primitive."

**Be ready for the follow-up:** "But isn't a busy-spinning thread wasteful?" Answer: "Yes, it pegs a core. We're explicitly trading a core for sub-microsecond response time. At our throughput, that core is fully utilised anyway. We're not running on a $5 droplet."

### Q5. "Tell me about a race condition you've debugged."

Have a story ready. Options:

**Option A (Rust race in the matching engine):** Early in development, before the SPSC ring design, we tried a `Mutex<VecDeque<Command>>` for ingress. Under load, the engine thread was spending most of its time on lock contention with the Kafka thread — visible in `perf` as time in `pthread_mutex_lock`. The race wasn't a correctness race (Rust caught those at compile time), but a *performance* race: both threads serialising on a single lock. The fix was the SPSC ring buffer — lock-free, cache-padded cursors. Lock-contention time went to zero.

**Option B (Go race from your Hydragon work):** the `HasQuorum` quorum-math bug. For validator sets under 4 nodes, the threshold computation was wrong, leading to 100% block stalls. Not a data race, but a logic race condition — the system's progress depended on a quorum check that misbehaved at the boundary. Fixed by re-deriving the threshold formula and unit-testing all set-sizes from 1 to 20.

Pick whichever feels more natural. Option A is more relevant to this JD; Option B is more dramatic if you want to show off the consensus work.

### Q6. "How do you profile a Rust application?"

(See Bullet 6 tools list.) Walk through a recent example: "When I saw tail-latency spikes in the matching engine, I started with `perf record` to get CPU samples, then `cargo flamegraph` to visualise. Saw allocations from `BTreeMap` node creation as a bottleneck. Confirmed with `dhat` that allocations were spiky. Refactored to slab + intrusive list; flamegraph and dhat both confirmed the fix."

### Q7. "What's your strategy for testing a matching engine?"

- **Unit tests** for each data structure (slab, ring, intrusive list, hot/cold book).
- **Integration tests** via a `TestHarness` that builds the engine in-process, feeds commands deterministically via `step()`, and asserts on emitted events. 1000+ such tests in `tests/matching_tests.rs`.
- **Property tests** (proptest/quickcheck) for invariants like "total qty at a price level equals sum of order qtys", "every alloc has a matching dealloc on cancel/fill", "token validation rejects stale tokens".
- **Fuzz tests** on the wire decoder (`Command::decode`) — malformed 48-byte frames must reject cleanly, never panic.
- **Replay tests** — capture a production-shaped event stream, replay it nightly in CI, assert the resulting book matches a golden snapshot.
- **Latency regression tests** via criterion in CI — median and p99 must stay within bands.

**Soundbite:** "Determinism makes testing tractable. Same input → same output, so I can capture a stream, replay it, and golden-compare the final book."

### Q8. "How do you handle deployment and rollouts on a system this latency-sensitive?"

- Engine processes are stateless from the persistence standpoint — restart-safe via Kafka replay.
- Snapshot every N seconds and commit a Kafka offset alongside, so restart replays only the tail.
- Deploys are per-pair: drain one pair's process, deploy new binary, replay from snapshot, resume.
- Blue/green for the gateways since they're stateless.
- Canary the new engine binary on a single low-traffic pair first.

### Q9. "What's the most challenging part of building this?"

Pick one specific thing, not a generic answer.

**Suggested answer:** "Getting deterministic tail latency. Throughput is the easy metric — bench it once and move on. p99 / p99.9 are the real test. The work was a chain of profile-find-fix cycles: page faults, allocations, cache misses, false sharing, BTreeMap inserts. Each pass dropped the tail by a multiple. The hot/cold book + slab + intrusive list combination was the breakthrough — together they removed the last sources of jitter on the matching path."

### Q10. "If you had to add futures (perpetual swaps) to this, what would you change?"

The matching engine itself wouldn't change much — it's asset-class-agnostic. Around it:

1. **Position service:** track user's open positions per contract. Updates on every fill event from the engine.
2. **Mark price service:** subscribes to multiple spot venues, computes a robust mark price (median, outlier filter), publishes to a Kafka topic.
3. **Funding rate service:** computes funding every 8 hours; settles positions by injecting fill events or balance adjustments.
4. **Liquidation engine:** reads positions and mark price, computes margin ratio, when below maintenance — submits a market order on behalf of the user to the matching engine. Liquidation orders are normal orders to the engine.
5. **Risk pre-check:** before an order enters the matching engine, the order service must check that the user has enough margin for the resulting position. This is the place to be careful — it must not become a bottleneck. We'd shard it per-user and keep margin state hot in memory.

**Soundbite:** "The matching engine doesn't know it's futures. Margin, mark, funding, and liquidation live in surrounding services. Liquidations enter as ordinary market orders. The hardest engineering problem isn't matching — it's keeping the margin pre-check fast enough not to slow the order path."

### Q11. "What's the role of `unsafe` in your code, and how do you keep it sound?"

(See deep-dive doc Section 11 / Q on unsafe.) Summary: "Three places — ring buffer slot access, slab arena access, intrusive list pointer updates. Each unsafe block has a SAFETY comment naming the invariant. Public APIs are safe; unsafety is implementation detail. We test extensively, especially the ring buffer under concurrent access in `tests/ring_buffer_tests.rs`. In debug builds, the slab also tracks per-slot occupancy and asserts on misuse — belt and braces."

### Q12. "How do you measure latency? What's your p99?"

- Latency is measured at two points: command ingress timestamp → first fill event egress timestamp (per-order).
- Captured via timestamps on the command frame (set by the gateway) and the event frame.
- Histogrammed with `hdrhistogram` for accurate percentiles.
- Exported to Prometheus.
- p50 sub-microsecond on the engine's portion (excluding Kafka). p99 under 5 μs. p99.9 under 20 μs (occasional JVM-style GC pauses don't exist in Rust, but page faults and cache misses can spike).

**Be honest about what you have and don't have:** "Those are bench numbers, not production — we haven't reached production load yet. The bench harness measures the engine's portion in isolation. End-to-end including Kafka is in the tens of milliseconds, dominated by Kafka commit."

---

## Section 3 — Quick-Hit Vocab Drill

You will be asked these in some form. Have a one-sentence answer ready:

- **Send:** type can be moved to another thread.
- **Sync:** `&T` can be shared across threads (equivalently, `&T: Send`).
- **Pin:** promise that a value won't move; required for self-referential futures.
- **Future:** poll-driven state machine; nothing runs until polled.
- **`.await`:** call `poll`; on `Pending`, yield to the runtime with a `Waker`.
- **Tokio runtime:** M:N work-stealing scheduler; tasks (green) on workers (OS threads).
- **`spawn_blocking`:** for sync/CPU-bound work; runs on a separate thread pool.
- **Backpressure:** propagating "slow down" upstream via bounded channels.
- **MPSC / SPSC:** multi/single producer; single consumer.
- **Disruptor:** LMAX pattern — ring buffer + sequence barriers for single-threaded core + parallel consumers.
- **Cache line:** 64 B on x86; false sharing happens when independent variables sit on the same line.
- **NUMA:** non-uniform memory access; local memory faster than remote-socket memory.
- **TLB:** translation lookaside buffer; cache of virtual→physical page mappings.
- **Hugepages:** 2 MB / 1 GB pages instead of 4 KB; fewer TLB misses.
- **SCHED_FIFO:** real-time scheduling class on Linux; preempts SCHED_OTHER.
- **`madvise(MADV_HUGEPAGE)`:** hint that a memory region should use hugepages.

---

## Section 4 — How to Open the Interview

When they say "tell me about yourself" or "walk me through your background":

> "I'm a backend and protocol engineer with about four years of production Rust and Go. Most recently I've been building a high-throughput matching engine in Rust for a centralised exchange — single-threaded core per pair, lock-free SPSC ring buffers at the boundaries, slab allocator with generation-stamped tokens, hot/cold hybrid order book. Sustains 728K orders/sec on commodity hardware, with sub-microsecond p99 on the engine portion.
>
> Before that I was on Hydragon, a PolyBFT EVM chain — Go side, shipped a validator slashing module across five PRs, did the London-to-Shanghai EVM hard-fork upgrade with four EIPs, and fixed a quorum-calculation bug that was stalling small validator sets. Before that, Substrate/FRAME pallets on 5ireChain — wrote the ESG pallet and integrated Frontier for full EVM compatibility — and five production Sui Move contracts for the HYDRO token economy.
>
> For this role I'm most excited about the matching engine and connectivity layer — that's where I've been spending my time, and it lines up with the JD."

Adjust length to read the room. If they want short: cut to the first paragraph. If they want detail: continue into Hydragon.

---

## Section 5 — Closing Questions to Ask Them

Senior interviewers expect senior questions back. Pick 2–3:

1. "What's the current engine's throughput and tail latency, and what's the next bottleneck you're trying to solve?"
2. "How do you handle multi-region deployment — is matching co-located with the gateway, or do orders cross regions?"
3. "How is the team split between matching, connectivity, risk, and infrastructure?"
4. "What's the testing/replay infrastructure like — do you have a captured production stream you can replay against a candidate binary?"
5. "What's the futures roadmap? Perp swaps? Options?"
6. "How do you handle a noisy market open? Do you have a circuit-breaker / pre-trade risk system above the engine?"
7. "What's the deployment cadence for the engine itself, and how do you do zero-downtime upgrades on a stateful in-memory system?"

These signal you've thought about the problem at the architecture level, not just the language level.

---

## Section 6 — Last-Hour Checklist (Tonight)

1. Re-read the [MATCHING_ENGINE_DEEP_DIVE.md](MATCHING_ENGINE_DEEP_DIVE.md) sections on ring buffers (4) and slab (7).
2. Re-read [RUST_INTERVIEW_PREP.md](RUST_INTERVIEW_PREP.md) Section 1 Q1–Q5 (ownership, borrowing, lifetimes, String/&str, Result).
3. Re-read this doc's Section 1 Bullet 3 (tokio vs std::thread vs rayon) and Bullet 5 (thousands of pairs).
4. Have one specific war story in your head: the slow-tail bug → slab + intrusive list fix.
5. Have the architecture diagram ready to draw on a whiteboard in 60 seconds.

You've shipped this stuff in production. Walk in, answer in your own voice, ask good questions, and let the work speak. You've got this.
