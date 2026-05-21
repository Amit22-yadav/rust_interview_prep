# Matching Engine Deep Dive — Interview Doc

> Single-threaded Rust matching engine. CPU-pinned, real-time scheduled, lock-free at every boundary, zero allocation on the hot path. This document walks every data structure, every ring buffer, every byte of layout, and explains *why* each choice was made.

All code references are real file paths and line numbers — open the file alongside this doc and read together.

---

## Table of Contents

1. [Process & Thread Model](#1-process--thread-model)
2. [Directory Layout](#2-directory-layout)
3. [The Engine Run Loop](#3-the-engine-run-loop)
4. [The Ring Buffers — Ingress & Egress](#4-the-ring-buffers--ingress--egress)
5. [The Order Book — Hot / Cold Hybrid](#5-the-order-book--hot--cold-hybrid)
6. [PriceLevel & Intrusive List](#6-pricelevel--intrusive-list)
7. [The Slab Allocator & Generation Tokens](#7-the-slab-allocator--generation-tokens)
8. [The Order Struct — 128 Bytes, Two Cache Lines](#8-the-order-struct--128-bytes-two-cache-lines)
9. [Matching Algorithm — Price-Time Priority](#9-matching-algorithm--price-time-priority)
10. [Cancel & Modify](#10-cancel--modify)
11. [How Low Latency Is Achieved (every trick)](#11-how-low-latency-is-achieved-every-trick)
12. [Backpressure, Failure, Shutdown](#12-backpressure-failure-shutdown)
13. [Likely Interview Questions](#13-likely-interview-questions)

---

## 1. Process & Thread Model

The engine is a single binary that spawns **four threads** in dedicated mode:

```
┌────────────────────────────────────────────────────────────┐
│ Process: matching-engine (PAIR=BTC-USDT)                   │
│                                                             │
│   ┌──────────────────┐                                      │
│   │  kafka-ingress   │  Kafka → decode 48B → ingress ring   │
│   │  (OS-scheduled)  │                                      │
│   └────────┬─────────┘                                      │
│            │                                                 │
│            ▼                                                 │
│   ┌───────────────────────────┐                              │
│   │      INGRESS RING         │  SPSC, 2^20 slots            │
│   │   (Command, 48 B/slot)    │                              │
│   └────────┬──────────────────┘                              │
│            │                                                 │
│            ▼                                                 │
│   ┌──────────────────────────────────┐                       │
│   │      ENGINE CORE THREAD          │                       │
│   │  • CPU-pinned                    │                       │
│   │  • SCHED_FIFO prio 49 (Linux)    │                       │
│   │  • busy-spin loop                │                       │
│   │  • owns OrderBook + Slab         │                       │
│   │  • no syscalls, no allocations   │                       │
│   └────────┬─────────────────────────┘                       │
│            │                                                 │
│            ▼                                                 │
│   ┌───────────────────────────┐                              │
│   │       EGRESS RING         │  SPSC, 2^20 slots            │
│   │     (Event, 56 B/slot)    │                              │
│   └────────┬──────────────────┘                              │
│            │                                                 │
│            ▼                                                 │
│   ┌──────────────────┐                                      │
│   │  kafka-egress    │  drain ring → Kafka produce          │
│   └──────────────────┘                                      │
│                                                             │
│   shutdown-monitor (signal pipe → AtomicBool::stop)         │
└────────────────────────────────────────────────────────────┘
```

**Why this layout:**
- Kafka I/O is OS-scheduled and may block on network. We **never** want the core thread to share that fate. The ring buffer is a bulkhead.
- The core thread does pure CPU work — pull a command, mutate the book, push events. No syscalls. No allocations. No blocking.
- The two Kafka threads can preempt and migrate freely; the core thread doesn't care because the rings absorb the jitter.

**Where this is wired up:** [matching-engine/src/main.rs:63–204](matching-engine/src/main.rs#L63-L204).

**CPU pinning** ([main.rs:45–68](matching-engine/src/main.rs#L45-L68)): the engine thread is set to a specific core via `core_affinity::set_for_current()`. Default is the last available core; override with `ENGINE_CPU=N`.

**Real-time scheduling** ([main.rs:398–428](matching-engine/src/main.rs#L398-L428)): on Linux we call `sched_setscheduler(0, SCHED_FIFO, prio=49)`. This requires `cap_sys_nice` capability on the binary. Effect: the engine thread is not preemptable by any normal-priority task. The kernel will not steal its core to run cron, the package manager, or another tenant.

**Pre-faulting the arena** ([main.rs:314–334](matching-engine/src/main.rs#L314-L334)): before trading starts, we touch every page of the slab arena (`engine.book.slab.prefault()`). Result: zero page faults at runtime. A page fault costs microseconds, which would blow the latency budget on the very first order to touch a fresh page.

---

## 2. Directory Layout

```
matching-engine/src/
├── main.rs                  ← binary entrypoint, thread wiring, CPU/RT setup
├── lib.rs                   ← module exports
├── matching/
│   ├── engine.rs            ← run loop, dispatch, handle_new_order/cancel/modify
│   ├── matcher.rs           ← Matcher::execute (price-time priority)
│   └── result.rs            ← MatchResult enum, Fill struct
├── orderbook/
│   ├── book.rs              ← OrderBook (bids, asks, slab)
│   ├── side.rs              ← BookSide (hot + cold)
│   ├── hot_book.rs          ← flat-array hot window
│   ├── cold_book.rs         ← BTreeMap for distant prices
│   ├── price_level.rs       ← FIFO queue + total_qty cache
│   └── intrusive_list.rs    ← doubly-linked list across slab indices
├── memory/
│   ├── slab.rs              ← generation-stamped allocator
│   ├── arena.rs             ← pre-allocated contiguous Vec<MaybeUninit<Order>>
│   └── pool.rs              ← hugepage advise helper
├── io/
│   ├── ring_buffer.rs       ← SPSC Disruptor ring
│   ├── ingress.rs           ← IngressWriter / IngressPoller wrappers
│   ├── egress.rs            ← EgressWriter / EgressPoller + backpressure
│   ├── kafka_ingress.rs     ← Kafka consumer thread
│   └── kafka_egress.rs      ← Kafka producer thread
├── types/
│   ├── order.rs             ← Order struct (128 B, repr(C, align(64)))
│   ├── price.rs             ← Price (u64 ticks)
│   ├── quantity.rs          ← Quantity (u64)
│   ├── command.rs           ← Command enum + 48B wire decode
│   ├── trade.rs             ← TradeId
│   └── stp.rs               ← StpMode (self-trade prevention)
├── events/event.rs          ← Event enum, RejectReason
├── config/engine_config.rs  ← EngineConfig, env overrides
└── utils/
    ├── cache_pad.rs         ← CachePadded<T> (64-byte aligned)
    └── id_gen.rs            ← monotonic ID generator
```

---

## 3. The Engine Run Loop

**File:** [matching-engine/src/matching/engine.rs:63–74](matching-engine/src/matching/engine.rs#L63-L74)

```rust
pub fn run(&mut self) {
    loop {
        if self.stop.load(Ordering::Acquire) {
            break;
        }
        if let Some(cmd) = self.ingress.poll() {
            self.dispatch_cmd(cmd);
        } else {
            std::hint::spin_loop();
        }
    }
}
```

That's the entire main loop. Notable properties:

- **Busy-spin, not block.** We never call `recv()` or `epoll_wait()`. Reason: a syscall to park/unpark the thread costs more than the work we'd do. At our rates, the ring is rarely empty for long.
- **`std::hint::spin_loop()`** when idle. This emits the `PAUSE` instruction on x86, which lowers power consumption inside the spin loop and avoids memory-order speculation that would hurt the producer.
- **`AtomicBool::Acquire`** on the stop flag. Acquire pairs with the Release in the signal handler, ensuring we see the flag set in finite time.
- **No lock, no mutex, no channel.** The ring buffer's `try_pop` is a couple of atomic loads and a `memcpy`.

**`step()`** ([engine.rs:80–107](matching-engine/src/matching/engine.rs#L80-L107)) is the same logic but processes exactly one command and returns — used by the test harness for deterministic stepping.

**Dispatch** ([engine.rs:214–260](matching-engine/src/matching/engine.rs#L214-L260)):

```rust
fn dispatch(&mut self, cmd: Command) -> bool {
    match cmd {
        Command::NewOrder { side, order_type, price, quantity, timestamp_ns, client_order_id, user_id } => {
            self.handle_new_order(...);
            true
        }
        Command::Cancel { token, client_order_id } => {
            self.handle_cancel(token, client_order_id);
            false
        }
        Command::Modify { token, new_price, new_quantity, timestamp_ns, client_order_id } => {
            self.handle_modify(...);
            false
        }
    }
}
```

A flat enum dispatch — three branches. The compiler turns this into a jump table. No vtable, no dynamic dispatch, no `Box<dyn Trait>` on the hot path.

---

## 4. The Ring Buffers — Ingress & Egress

This is the most important subsystem to understand in detail. The interviewer will probe here.

**File:** [matching-engine/src/io/ring_buffer.rs:1–107](matching-engine/src/io/ring_buffer.rs#L1-L107)

### 4.1 The Type

```rust
pub struct RingBuffer<T> {
    buf: Vec<UnsafeCell<MaybeUninit<T>>>,
    mask: usize,                              // capacity - 1, capacity is power-of-two
    head: CachePadded<AtomicUsize>,           // producer cursor
    tail: CachePadded<AtomicUsize>,           // consumer cursor
}

unsafe impl<T: Send> Send for RingBuffer<T> {}
unsafe impl<T: Send> Sync for RingBuffer<T> {}
```

This is a **Disruptor-style SPSC ring buffer**. Let me unpack each design choice.

### 4.2 `Vec<UnsafeCell<MaybeUninit<T>>>`

- **`Vec<...>`** of fixed capacity — allocated once at construction, never grows.
- **`MaybeUninit<T>`** because slots may be logically uninitialised. We don't pay the cost of writing a default value into every slot at startup.
- **`UnsafeCell<...>`** because the slot is interior-mutable through a shared `&RingBuffer`. Multiple threads hold the same `&RingBuffer` (producer and consumer); the cursors enforce exclusion, not the type system.
- **No per-slot tag.** Occupancy is encoded purely by the relationship between `head` and `tail`. That's one fewer atomic per push/pop compared to designs that mark each slot.

### 4.3 Power-of-Two Capacity

```rust
pub fn new(capacity: usize) -> Self {
    let cap = capacity.next_power_of_two();
    ...
    Self { buf, mask: cap - 1, ... }
}
```

If capacity is `2^N`, then `index % capacity == index & (capacity - 1)`. We store `mask = capacity - 1` and use bitwise AND to index. A modulo is ~20–40 cycles on x86; a bitwise AND is one cycle. On the hot path, that matters.

### 4.4 `CachePadded` Cursors — Avoiding False Sharing

```rust
head: CachePadded<AtomicUsize>,
tail: CachePadded<AtomicUsize>,
```

**`CachePadded<T>`** ([utils/cache_pad.rs](matching-engine/src/utils/cache_pad.rs)) wraps a value with padding to align it to 64 bytes — the size of a cache line on x86-64.

**Why this matters:** without padding, `head` and `tail` would likely share a cache line. The producer writes `head` constantly; the consumer writes `tail` constantly. Every write invalidates the line in the other core's L1. That's **false sharing** — even though they're touching different variables, the cache coherence protocol thinks they're contending. Pinging a cache line between two cores costs ~40 ns each way. At a million ops/sec, that's the entire latency budget gone.

By padding each cursor to its own cache line, producer and consumer touch independent lines. Each cursor stays hot in the owning core's L1.

### 4.5 `try_push` — The Producer Side

```rust
#[inline]
pub fn try_push(&self, value: T) -> Result<(), T> {
    let head = self.head.get().load(Ordering::Relaxed);
    let tail = self.tail.get().load(Ordering::Acquire);

    if head.wrapping_sub(tail) == self.buf.len() {
        return Err(value);                       // full
    }

    unsafe {
        (*self.buf[head & self.mask].get()).write(value);
    }
    self.head.get().store(head.wrapping_add(1), Ordering::Release);
    Ok(())
}
```

Walk it line by line:

1. **`head` load is `Relaxed`** — only the producer ever writes `head`, so no synchronisation is needed to read its own most recent value.
2. **`tail` load is `Acquire`** — the consumer publishes `tail` updates via `Release`. Acquire here pairs with that Release: it guarantees that any reads we do after this point see the consumer's prior actions. Specifically, we need to know that slots up to `tail` are *truly free* (the consumer has finished reading them) before we overwrite them.
3. **`head.wrapping_sub(tail) == capacity`** is the "full" check. Wrapping subtraction handles the case where `head` has wrapped past `usize::MAX` (decades of operation, but correct).
4. **Slot write** — unsafe because we're writing into `MaybeUninit` through `UnsafeCell`. Safety relies on the invariant that the slot at `head & mask` is logically free (the full check just proved it).
5. **`head` store is `Release`** — this publishes the slot write to the consumer. The Release ensures that when the consumer's Acquire-load on `head` sees the new value, the slot data is also visible. This is the **publication fence**.

### 4.6 `try_pop` — The Consumer Side

```rust
#[inline]
pub fn try_pop(&self) -> Option<T> {
    let tail = self.tail.get().load(Ordering::Relaxed);
    let head = self.head.get().load(Ordering::Acquire);

    if tail == head {
        return None;                              // empty
    }

    let value = unsafe { (*self.buf[tail & self.mask].get()).assume_init_read() };
    self.tail.get().store(tail.wrapping_add(1), Ordering::Release);
    Some(value)
}
```

Mirror image:

1. **`tail` Relaxed** — we own `tail`.
2. **`head` Acquire** — pairs with producer's Release; we see producer's slot writes.
3. **Empty when `tail == head`.**
4. **`assume_init_read`** moves the value out of the slot. After this read, the slot is logically uninitialised again.
5. **`tail` Release** — publishes that this slot is free for the producer to reuse.

### 4.7 Why SPSC and Not MPMC?

Because the engine has exactly one producer (the Kafka ingress thread) and one consumer (the engine core), SPSC is the right primitive. MPMC rings (like crossbeam's `ArrayQueue`) need CAS loops for both head and tail, which is slower. SPSC needs only plain loads and stores — no CAS.

For the egress ring, symmetric: engine produces, Kafka egress consumes.

### 4.8 Sizes

Configured via env vars ([config/engine_config.rs](matching-engine/src/config/engine_config.rs)):

| Var | Default | Meaning |
|---|---|---|
| `INGRESS_RING_SIZE` | 1,048,576 (2^20) | ~50 MB at 48 B/command |
| `EGRESS_RING_SIZE` | 1,048,576 (2^20) | ~56 MB at 56 B/event |

Sized to absorb seconds of burst at peak rate before backpressure kicks in. Power-of-two enforced; non-power-of-two inputs round up.

### 4.9 The `Ingress` and `Egress` Wrappers

[io/ingress.rs:1–84](matching-engine/src/io/ingress.rs#L1-L84):

```rust
pub struct IngressWriter { ring: Arc<RingBuffer<Command>>, dropped: AtomicU64 }
pub struct IngressPoller { ring: Arc<RingBuffer<Command>> }

impl IngressWriter {
    pub fn publish(&self, cmd: Command) {
        if self.ring.try_push(cmd).is_err() {
            self.dropped.fetch_add(1, Ordering::Relaxed);
        }
    }
}
```

Note: ingress drops on full and increments a counter. Dropped commands surface as a metric. The producer is the Kafka thread; if the engine falls behind for a sustained period, we drop — but Kafka itself is the durable backstop, so on recovery we replay.

[io/egress.rs:1–108](matching-engine/src/io/egress.rs#L1-L108):

```rust
impl EgressWriter {
    pub fn publish(&self, event: Event) {
        if self.ring.try_push(event).is_err() {
            panic!("[egress] ring full — consumer stalled; halting engine");
        }
    }
}

pub struct EgressBackpressure {
    ring: Arc<RingBuffer<Event>>,
    high_water: usize,    // default 75% of capacity
    low_water: usize,     // 37.5% (hysteresis)
}
```

Egress is **panic-on-full**, never drop. Reason: an unobserved event is a correctness bug — a fill that we matched but didn't publish would leave the order service desynchronised. So we make sure the ring never fills by **applying backpressure to the ingress thread** when egress occupancy crosses the high-water mark. The Kafka ingress thread polls `EgressBackpressure::is_above_high_water()` and pauses consuming from Kafka until occupancy drops below low-water (37.5% — hysteresis to avoid flapping).

This means the engine itself never blocks. The block happens at the Kafka layer, which has Kafka's commit log as the durable buffer.

---

## 5. The Order Book — Hot / Cold Hybrid

This is the second piece interviewers will dig into.

### 5.1 Top-Level: `OrderBook`

**File:** [orderbook/book.rs:1–56](matching-engine/src/orderbook/book.rs#L1-L56)

```rust
pub struct OrderBook {
    pub bids: BookSide,
    pub asks: BookSide,
    pub slab:  Slab,
}
```

Both sides share a single slab — orders are referenced by `(slab_index, generation)` tokens. Cross-side cancellation, modify, all use the same index space.

### 5.2 `BookSide` — Hot/Cold Routing

**File:** [orderbook/side.rs:1–150](matching-engine/src/orderbook/side.rs#L1-L150)

```rust
pub struct BookSide {
    pub hot:        HotBook,    // flat array, O(1)
    pub cold:       ColdBook,   // BTreeMap<Price, PriceLevel>, O(log N)
    pub hot_ticks:  usize,
}
```

**The insight:** in any active market, ~99% of orders cluster within a few percent of the current price. The book has a tight body and long thin tails. A pure `BTreeMap<Price, PriceLevel>` would pay O(log N) for every operation regardless — wasted on the body where the answer is "look at the array index right next to the last one."

So we run a **hybrid**:
- **Hot book:** a flat array covering a window of `hot_ticks` ticks around a center price. Index by direct subscript: `array[(price - base_ticks)]`. O(1) access, sequential memory layout, cache-friendly.
- **Cold book:** a `BTreeMap<Price, PriceLevel>` for orders outside the hot window. Rare, slower, but unbounded in range.

The cached `best_bid_idx` and `best_ask_idx` make "what's the best price?" O(1) — no scan, just read a cached integer.

### 5.3 `HotBook`

**File:** [orderbook/hot_book.rs](matching-engine/src/orderbook/hot_book.rs)

Conceptually:

```rust
pub struct HotBook {
    levels:        Vec<Option<PriceLevel>>,   // indexed by (price - base_ticks)
    base_ticks:    Price,
    best_bid_idx:  Option<usize>,             // cached best-price pointer
    best_ask_idx:  Option<usize>,
}
```

- **`Vec<Option<PriceLevel>>`** of length `hot_ticks` (default 2000) — each slot is either `None` (no resting orders at this tick) or `Some(PriceLevel)`.
- `best_bid_idx`/`best_ask_idx` are maintained as orders are inserted/removed. After a match, we walk down/up to find the next non-empty slot — but only by a few slots typically, not the whole array.

**Re-centering** ([orderbook/side.rs](matching-engine/src/orderbook/side.rs)): when the last trade price drifts into the edge zone (default last 10%), we shift the hot window. Orders inside the new window stay in hot; orders outside fall to cold; orders in cold that are now inside hot get promoted. This is rare (only on big moves) and uses O(1) intrusive-list splices to migrate the per-level FIFOs without copying any orders.

### 5.4 `ColdBook`

**File:** [orderbook/cold_book.rs](matching-engine/src/orderbook/cold_book.rs)

```rust
pub struct ColdBook {
    levels: BTreeMap<Price, PriceLevel>,
}
```

Plain BTreeMap. Used for orders outside the hot window. O(log N) ops, but N is small (the tails), and operations on cold are rare relative to hot.

**Why BTreeMap, not Vec or skiplist:**
- Need ordered access to best price (next-best on either side).
- BTreeMap is a B-tree internally — many keys per node, cache-friendly compared to a binary tree of pointers.
- In std, well-tested, no dependency.
- A Vec would be O(N) insert in the middle. A skiplist would be similar to BTreeMap asymptotically but worse cache behaviour and not in std.

---

## 6. PriceLevel & Intrusive List

### 6.1 `PriceLevel`

**File:** [orderbook/price_level.rs:1–74](matching-engine/src/orderbook/price_level.rs#L1-L74)

```rust
pub struct PriceLevel {
    pub orders:    IntrusiveList,    // FIFO of slab indices
    pub total_qty: Quantity,         // cached sum of all remaining quantities
}
```

- **`orders`** — a doubly-linked list of orders at this price, FIFO order (head = oldest = highest time priority).
- **`total_qty`** — cached aggregate. Used for market-data snapshots and to short-circuit "is there enough size?" checks without walking the list.

### 6.2 `IntrusiveList` — Linked List in the Arena

**File:** [orderbook/intrusive_list.rs:1–122](matching-engine/src/orderbook/intrusive_list.rs#L1-L122)

```rust
pub struct IntrusiveList {
    head: u64,    // slab index of head order (or NULL sentinel)
    tail: u64,
}
```

The list nodes are **the orders themselves**, in the slab. Each `Order` carries `prev: u64` and `next: u64` fields ([types/order.rs:42–94](matching-engine/src/types/order.rs#L42-L94)). To traverse the list you index into the slab.

**Why intrusive, not `VecDeque` or `LinkedList`:**

| Op | `VecDeque<OrderId>` | `LinkedList<Order>` (std) | **Intrusive in slab** |
|---|---|---|---|
| Push back | O(1) amortised, may realloc | O(1), heap alloc per node | **O(1), no alloc** |
| Pop front (full fill) | O(1) | O(1), node freed | **O(1), slab dealloc** |
| Remove middle (cancel) | **O(N)** (shift) | O(1) given node ptr | **O(1) given slab index** |
| Cache locality | Good for sequential | Poor (heap-scattered) | **Good (arena-contiguous)** |
| Splice two lists | O(N) | O(1) | **O(1)** |

The killer feature: **O(1) cancel** of any order in the middle of the queue, given only its slab index. A cancel command arrives with a token; we extract the slab index; we look up the order in the slab; we read its `prev`/`next`; we rewire the neighbours' pointers. Done in nanoseconds, with no memory copies.

`std::collections::LinkedList` also has O(1) middle-remove but allocates per node on the heap and has no cursor API in stable Rust. The intrusive design is strictly better for our access pattern.

**O(1) splice** is also useful for hot-book re-centering — we splice entire FIFOs between hot and cold without touching individual orders.

---

## 7. The Slab Allocator & Generation Tokens

**File:** [memory/slab.rs:1–175](matching-engine/src/memory/slab.rs#L1-L175)

### 7.1 The Type

```rust
pub struct Slab {
    arena:        Arena,                 // Vec<MaybeUninit<Order>>
    free_list:    Vec<SlabIndex>,        // stack of available indices
    generations:  Vec<u32>,              // gen counter per slot
    #[cfg(debug_assertions)]
    occupied:     Vec<bool>,             // dev-only sanity check
}
```

- **`arena`** ([memory/arena.rs](matching-engine/src/memory/arena.rs)): a `Vec<MaybeUninit<Order>>` of fixed capacity, allocated once at startup, pre-faulted into RAM. Default capacity 2^22 = 4,194,304 orders × 128 B = **512 MB**.
- **`free_list`** — a stack of free slot indices. `alloc` pops, `dealloc` pushes. O(1) both ways.
- **`generations`** — for each slot, a `u32` counter that bumps on every deallocation. This is what makes tokens safe.

### 7.2 The Token

```rust
pub type Token = u64;
//  bits 63..32 : generation (u32)
//  bits 31..0  : slab index (u32)

pub fn pack_token(index: SlabIndex, generation: u32) -> Token {
    ((generation as u64) << 32) | (index as u64)
}
```

When an order is allocated, we return a token that combines `(slab_index, current_generation_of_that_slot)`. The order service quotes this token in cancel/modify commands.

### 7.3 Allocation, Deallocation, Validation

**Alloc** ([slab.rs](matching-engine/src/memory/slab.rs), O(1)):

```rust
pub fn alloc(&mut self, order: Order) -> Option<Token> {
    let index = self.free_list.pop()?;
    unsafe { self.arena.write(index, order); }
    Some(pack_token(index, self.generations[index]))
}
```

**Dealloc** (O(1)):

```rust
pub unsafe fn dealloc(&mut self, index: SlabIndex) {
    self.generations[index] = self.generations[index].wrapping_add(1);   // bump gen
    self.free_list.push(index);                                          // back to pool
}
```

The generation bump is the magic: **any token previously issued for this slot is now stale**. If a Cancel arrives quoting an old token after the slot has been reused for a new order, validation rejects it.

**Validate** (O(1)):

```rust
pub fn validate(&self, token: Token) -> Option<SlabIndex> {
    let index = token_index(token);
    let stored_gen = token_generation(token);
    if index < self.generations.len() && self.generations[index] == stored_gen {
        Some(index)
    } else {
        None
    }
}
```

Three checks, all O(1): index in range, generation matches. No hash lookup, no pointer chase.

### 7.4 Why This Pattern (vs HashMap<OrderId, *Order>)

The naive alternative is a `HashMap<OrderId, Box<Order>>`. Problems:
- Heap allocation per order (slow, fragments, page faults).
- Hash lookup on every cancel — branchy, cache-miss-heavy.
- No protection against stale IDs — you'd need to delete from the map and hope no concurrent reader holds a dangling pointer.

The slab + generation pattern is what game engines, kernels, and high-perf databases use for the same reasons: **predictable memory, O(1) safe lookup, ABA-free reuse**.

### 7.5 Pre-Faulting

[main.rs:314–334](matching-engine/src/main.rs#L314-L334) calls `slab.prefault()` before trading. This iterates every page of the arena's underlying memory, faulting it in. After this, the kernel has mapped all pages to physical RAM. Without pre-faulting, the first write to each page would trap into the kernel for a minor page fault — adding microseconds to whichever unlucky order happens to be first into a fresh page. With pre-faulting, zero page faults during trading.

---

## 8. The Order Struct — 128 Bytes, Two Cache Lines

**File:** [types/order.rs:42–94](matching-engine/src/types/order.rs#L42-L94)

```rust
#[repr(C, align(64))]
pub struct Order {
    pub token:           Token,       // 8 B
    pub price:           Price,       // 8 B
    pub client_order_id: u64,         // 8 B
    pub remaining_qty:   Quantity,    // 8 B
    pub timestamp_ns:    u64,         // 8 B
    pub prev:            u64,         // 8 B  ← intrusive list
    pub next:            u64,         // 8 B  ← intrusive list
    pub side:            Side,        // 1 B
    pub order_type:      OrderType,   // 1 B
    pub _pad0:           [u8; 6],     // 6 B
    // ─── 64 byte boundary ───
    pub user_id:         u64,         // 8 B (for STP)
    pub _pad1:           [u8; 56],    // 56 B
}
// Total: 128 bytes exactly. align(64) forces L1-cache-line alignment.
```

**Design choices:**

- **`#[repr(C)]`** — predictable field layout, no rearranging by the compiler. Useful for FFI and for reasoning about layout.
- **`align(64)`** — start of every `Order` is aligned to a cache line boundary. Matters because if an `Order` straddled a line boundary, reading it would touch two cache lines instead of one.
- **128 bytes = 2 cache lines.** Hot fields (token, price, qty, prev, next) live in line 0. STP-related `user_id` and padding live in line 1. The matcher only touches line 0 for the common path — line 1 is loaded only when STP is enabled.
- **Padding** to round up to exactly 128 bytes. This makes pointer arithmetic in the arena trivial: `arena_base + index * 128`.
- **`prev` / `next` as `u64`** — slab indices, not pointers. Indexing is offset arithmetic, not pointer chase; the arena base is in a register.

**Why 128 and not 64?** Could we fit in one line? The STP fields plus padding push us into a second line. We chose to keep STP in the struct (rather than a side-table) because user_id check is on the hot path when STP is enabled. The 2nd line is loaded only when needed.

---

## 9. Matching Algorithm — Price-Time Priority

**File:** [matching/matcher.rs:1–160](matching-engine/src/matching/matcher.rs#L1-L160)

```rust
pub struct Matcher;
impl Matcher {
    pub fn execute<'a>(
        book:          &mut OrderBook,
        taker:         &mut Order,
        fills_buffer:  &'a mut Vec<Fill>,
        stp_removals:  &mut Vec<StpRemoval>,
        stp_mode:      StpMode,
    ) -> MatchResult<'a> { ... }
}
```

Stateless — `Matcher` is a zero-sized type, just a namespace for the matching function. The caller (engine.rs) owns the buffers; the matcher fills them.

### 9.1 The Algorithm

For a taker on side `S`, match against opposing side:

```
loop:
  best_price = opposing_side.best_price()
  if no best_price → break (no liquidity)

  if taker.order_type == Limit:
    if best_price does not cross taker.price → break (no more match)
  else (Market):
    // market orders ignore price; the sentinel values
    // u64::MAX (market buy) / 0 (market sell) never block the cross check

  level = opposing_side.get_mut(best_price)

  while level not empty AND taker.remaining_qty > 0:
    maker_idx = level.orders.head
    maker = slab.get(maker_idx)

    // ─── Self-Trade Prevention ───
    if stp_mode != Off and maker.user_id == taker.user_id:
      apply STP policy (cancel maker, cancel taker, cancel both)
      continue / break accordingly

    // ─── Normal fill ───
    fill_qty = min(taker.remaining, maker.remaining)
    maker.remaining_qty -= fill_qty
    taker.remaining_qty -= fill_qty

    fills_buffer.push(Fill { maker_token, price: best_price, qty: fill_qty, ... })

    if maker.remaining_qty == 0:
      level.orders.remove(maker_idx)       // O(1) intrusive
      slab.dealloc(maker_idx)               // O(1), bumps generation
    else:
      level.total_qty -= fill_qty           // update cache

  if level is empty:
    opposing_side.remove_if_empty(best_price)

  if taker.remaining_qty == 0:
    break
```

### 9.2 Key Properties

- **Price priority:** we always match against the *best* opposing price first. The hot-book's cached `best_*_idx` gives that in O(1).
- **Time priority:** within a price level, we always pop from the head (FIFO). Earliest order wins.
- **Fill price is the maker's price.** A buy at limit 100 hitting a sell at 99 trades at 99, not 100. This is exchange-standard and rewards passive liquidity.
- **Single pass.** We never revisit a maker — once it's filled or skipped, we move on. No backtracking.
- **Buffers are reused.** `fills_buffer` is a `Vec` reused across orders. Allocated once at engine startup, cleared between commands. Zero allocation on the hot path.

### 9.3 The `MatchResult` Enum

[matching/result.rs](matching-engine/src/matching/result.rs):

```rust
pub enum MatchResult<'a> {
    Resting,                                                  // no fills, rest on book
    FullFill    { fills: &'a [Fill] },                        // taker fully consumed
    PartialFill { fills: &'a [Fill], remaining_qty: Quantity }, // rest residual on book
    StpCancelled { fills: &'a [Fill], remaining_qty: Quantity },
}
```

The engine inspects this and decides whether to insert the residual back onto the book.

---

## 10. Cancel & Modify

### 10.1 Cancel

**File:** [matching/engine.rs:571–623](matching-engine/src/matching/engine.rs#L571-L623)

```rust
fn handle_cancel(&mut self, token: Token, client_order_id: u64) {
    // 1. Validate token — O(1) generation check
    let slab_idx = match self.book.slab.validate(token) {
        Some(idx) => idx,
        None => {
            self.egress.publish(Event::OrderCancelRejected { token, client_order_id: 0 });
            return;
        }
    };

    // 2. Read order metadata
    let (price, remaining, side, stored_coid) = unsafe {
        let order = self.book.slab.get(slab_idx);
        (order.price, order.remaining_qty, order.side, order.client_order_id)
    };

    // 3. Optional client_order_id ownership check
    if client_order_id != 0 && client_order_id != stored_coid {
        self.egress.publish(Event::OrderCancelRejected { token, client_order_id: stored_coid });
        return;
    }

    // 4. Remove from price level's intrusive list — O(1)
    let book_side = match side { Side::Bid => &mut self.book.bids, Side::Ask => &mut self.book.asks };
    if let Some(level) = book_side.get_mut(price) {
        unsafe { level.remove_order(slab_idx as u64, remaining, &mut self.book.slab); }
    }
    book_side.remove_if_empty(price);

    // 5. Free slab slot — bumps generation, makes token stale
    unsafe { self.book.slab.dealloc(slab_idx); }

    // 6. Emit event
    self.egress.publish(Event::OrderCanceled { token, client_order_id });
}
```

**Cost:** four memory accesses (token validate, read order, list unlink, slab dealloc) plus one ring buffer push. Sub-microsecond, no allocations.

### 10.2 Modify

[matching/engine.rs:499–569](matching-engine/src/matching/engine.rs#L499-L569)

Implemented as **cancel + re-insert**:
1. Validate token.
2. Read old order's side.
3. Cancel the old order (same path as above).
4. Emit `OrderCanceled` for the old token.
5. Call `handle_new_order` with the new price/qty (which may match immediately if price improved).
6. The new order gets a *new* token — its generation has been bumped.

**Why cancel-and-reinsert, not in-place edit?** Because changing the price means losing time priority — and that's the exchange-correct behaviour. Letting an order keep its time priority while moving in price would be unfair to other orders that came after it at the new price.

(An edge case: shrinking quantity without changing price could in principle preserve priority. We chose simplicity and consistency — always cancel-reinsert. The hot path stays small and uniform.)

---

## 11. How Low Latency Is Achieved — Every Trick

Here are all the latency-reduction techniques in the engine, with the rationale for each.

| # | Technique | Where | Why it matters |
|---|---|---|---|
| 1 | **Single-threaded core** | [engine.rs:63–74](matching-engine/src/matching/engine.rs#L63-L74) | Zero synchronisation. The book is owned by one thread; no `Mutex`, no `Arc<RefCell>`, no atomic CAS on the matching path. Deterministic output. |
| 2 | **CPU pinning** | [main.rs:45–68](matching-engine/src/main.rs#L45-L68) | Engine thread never migrates to a different core. Caches stay warm. No TLB shootdown surprises. |
| 3 | **SCHED_FIFO real-time priority** | [main.rs:398–428](matching-engine/src/main.rs#L398-L428) | OS will not preempt us for normal-priority work. Cron, package updates, other tenants — none of them can steal our core. |
| 4 | **Pre-faulted arena** | [main.rs:314–334](matching-engine/src/main.rs#L314-L334) | Zero page faults during trading. Each fault would cost μs. |
| 5 | **Busy-spin instead of blocking** | [engine.rs:69](matching-engine/src/matching/engine.rs#L69) | No `recv()` syscall. The cost of unparking a thread is much higher than the work we'd do. We pay one core for sub-μs wake-up. |
| 6 | **`std::hint::spin_loop()` while idle** | [engine.rs:71](matching-engine/src/matching/engine.rs#L71) | Emits `PAUSE` on x86 — reduces power, frees execution-unit slots for the SMT sibling (if any), and avoids memory-order speculation that would harm the producer. |
| 7 | **SPSC ring buffers** | [io/ring_buffer.rs](matching-engine/src/io/ring_buffer.rs) | Lock-free, no CAS, just Acquire/Release on two cursors. Single load + single store per push/pop. |
| 8 | **Cache-line padded cursors** | [io/ring_buffer.rs:18–19](matching-engine/src/io/ring_buffer.rs#L18-L19), [utils/cache_pad.rs](matching-engine/src/utils/cache_pad.rs) | Eliminates false sharing between producer and consumer. Without padding, every push invalidates the consumer's cache line and vice versa. |
| 9 | **Power-of-two ring size** | [ring_buffer.rs:28, 34](matching-engine/src/io/ring_buffer.rs#L28) | `idx % cap` becomes `idx & mask` — 1 cycle vs ~20. |
| 10 | **Fixed-size 48-byte command frames** | [types/command.rs](matching-engine/src/types/command.rs) | No parsing, no allocation. Decode is field reads from a known offset. Tag byte at offset 42 chooses the variant. |
| 11 | **Slab allocator with O(1) alloc/free** | [memory/slab.rs](matching-engine/src/memory/slab.rs) | No `malloc`/`free` on the hot path. Free list is a `Vec<usize>`; pop/push is constant time. |
| 12 | **Generation-stamped tokens** | [memory/slab.rs](matching-engine/src/memory/slab.rs) | O(1) validation. Stale tokens reject immediately. No HashMap lookup, no pointer chase. |
| 13 | **Intrusive doubly-linked lists** | [orderbook/intrusive_list.rs](matching-engine/src/orderbook/intrusive_list.rs) | O(1) cancel of any order in the queue (given its slab index). No `VecDeque::remove` shift cost. List nodes live inside the slab — contiguous memory, no per-node heap allocation. |
| 14 | **Hot/cold order book** | [orderbook/side.rs](matching-engine/src/orderbook/side.rs), [hot_book.rs](matching-engine/src/orderbook/hot_book.rs) | The body of the book lives in a flat array with O(1) lookup. Only the tails fall through to BTreeMap. Best-price access is O(1) via cached index. |
| 15 | **128-byte cache-aligned Order struct** | [types/order.rs:42–94](matching-engine/src/types/order.rs#L42-L94) | One order = exactly two cache lines, aligned. Hot fields in line 0; STP fields in line 1 (only loaded when STP enabled). |
| 16 | **Reused output buffers** | [engine.rs](matching-engine/src/matching/engine.rs) — `fills_buffer: Vec<Fill>` | The matcher writes into a `Vec` that's `.clear()`'d between orders. Allocated once at startup. Zero hot-path allocation. |
| 17 | **`#[inline]` on hot functions** | ring_buffer push/pop, intrusive list ops | Inlining lets the compiler see across the boundary, hoist atomic ops, and avoid function-call overhead. |
| 18 | **Flat enum dispatch, no `dyn Trait`** | [engine.rs:dispatch](matching-engine/src/matching/engine.rs#L214-L260) | Three-way match becomes a jump table. No vtable lookups, full inlining of each handler. |
| 19 | **Integer-only price arithmetic** | [types/price.rs](matching-engine/src/types/price.rs), [types/quantity.rs](matching-engine/src/types/quantity.rs) | Prices are `u64` ticks, never `f64`. No NaN, no rounding, no float-comparison ambiguity. Comparisons are integer ops. |
| 20 | **Backpressure at the I/O boundary** | [io/egress.rs:EgressBackpressure](matching-engine/src/io/egress.rs) | Engine never blocks; the Kafka ingress thread pauses Kafka consumption when egress occupancy is high. Kafka becomes the durable buffer. |
| 21 | **`MaybeUninit<T>` slots** | [ring_buffer.rs:16](matching-engine/src/io/ring_buffer.rs#L16), [arena.rs](matching-engine/src/memory/arena.rs) | We don't pay to default-initialise every slot at startup. Occupancy tracked by cursors / free list. |
| 22 | **One ring per direction** | ingress vs egress | Each ring has one writer and one reader — true SPSC. If we used one shared ring we'd need MPMC and pay CAS overhead. |
| 23 | **Per-pair process (dedicated mode)** | [main.rs](matching-engine/src/main.rs) | No cross-pair contention. Each pair owns a CPU core. |
| 24 | **Total `total_qty` cache per price level** | [price_level.rs](matching-engine/src/orderbook/price_level.rs) | Aggregate quantity at a price is read constantly (depth snapshots, sufficiency checks). Caching avoids walking the order list. |

**The general principle:** at every layer, we asked "what does the OS / allocator / compiler do that we *don't* need?" and removed it. The engine does the minimum work to be correct, and does it on memory we already own.

---

## 12. Backpressure, Failure, Shutdown

### 12.1 Ingress Backpressure

If the engine falls behind Kafka, the ingress ring fills. `IngressWriter::publish` drops on full and increments a counter. Kafka itself retains the durable record; on a restart, replay catches us up.

### 12.2 Egress Backpressure

The engine **must not lose events** — every fill is a correctness-critical announcement. So:
- `EgressWriter::publish` **panics** if the ring is full. We treat a full egress ring as an unrecoverable situation that needs operator attention.
- To make sure that never happens, `EgressBackpressure` ([io/egress.rs:EgressBackpressure](matching-engine/src/io/egress.rs)) exposes high/low water marks. The Kafka ingress thread polls this and pauses Kafka consumption when occupancy ≥ 75%, resumes at ≤ 37.5%. Hysteresis prevents oscillation.

Net result: when egress is slow (e.g., Kafka broker GC pause), ingress pauses. Kafka becomes the durable buffer. The engine never blocks the matching loop.

### 12.3 Shutdown

[main.rs:199–201](matching-engine/src/main.rs#L199-L201) registers SIGINT/SIGTERM handlers that write to a pipe. The `shutdown-monitor` thread blocks on `read`, unblocks on signal, and stores `true` into the `stop: Arc<AtomicBool>` with `Ordering::Release`. The engine's `run()` loop sees the flag (`Acquire` load) on the next iteration and breaks. All threads join. The Kafka producer flushes.

### 12.4 Crash Recovery

Engine state is purely in-memory. On crash:
1. Restart the binary.
2. Pre-fault, build empty book.
3. Replay `engine-events-<PAIR>` from a snapshot offset (or beginning of retention) to rebuild the book.
4. Resume consuming `order-commands-<PAIR>` from last committed offset.

Snapshotting on a cadence is the natural next step — periodically serialize the book and commit a Kafka offset marker so replay is bounded.

---

## 13. Likely Interview Questions

These are the questions an interviewer will ask after you describe the engine. Have crisp answers ready.

### Q: Why single-threaded? Don't you waste cores?

A: Within a symbol, price-time priority *requires* a single serialiser — events must have a total order. Multithreading inside one symbol just means contending for the same lock, which is slower than not having the lock at all. We scale across symbols, not within. At 100K+ orders/sec on one core, the engine isn't the bottleneck — Kafka and the database are. We deploy one engine process per pair (dedicated mode) or one process for many pairs (shared mode); both run the core thread alone on its core.

### Q: Why a ring buffer instead of a Tokio channel?

A: Tokio channels (`mpsc`) are MPSC with allocation on each push, atomic CAS in the implementation, and a wait/wake protocol. Our use is strictly SPSC, no allocation needed, and the consumer never wants to block — it wants to spin. A purpose-built SPSC ring with cache-padded cursors is much faster: a couple of atomic loads and a memcpy per op. We measured the difference; it's significant on the hot path.

### Q: Walk me through what happens when a market buy arrives.

A: The Kafka thread decodes a 48-byte frame; tag byte = 0 means NewOrder; `price` field = `u64::MAX` is the market-buy sentinel. It `try_push`es a `Command::NewOrder` into the ingress ring. The engine core thread, busy-spinning on `try_pop`, picks it up and calls `handle_new_order`. That allocates an Order in the slab (O(1) free-list pop) and calls `Matcher::execute`. The matcher reads the asks side's best ask via the hot book's cached index — O(1) — and matches FIFO down that level's intrusive list. For each maker we fill, we push a `Fill` into the reusable buffer; if the maker is fully consumed, we unlink it from the list and dealloc its slab slot (bumping the generation). When the taker is exhausted or no liquidity remains, we exit. The engine then converts the Fills into `MakerFill`, `TakerFill`, and `Trade` events and pushes each into the egress ring. The Kafka egress thread drains and produces to Kafka. End to end, the core-thread portion takes a few hundred nanoseconds for a typical match.

### Q: How do you handle order cancellation in O(1)?

A: Three pieces. (1) Tokens encode `(slab_index, generation)`, so token-to-slot is a single integer extract — no hash. (2) Generation validation is O(1) — compare the token's generation against the slab's current generation for that slot. (3) Once we have the order, the intrusive doubly-linked list lets us splice it out of the price-level queue in O(1) using its `prev` and `next` fields. Then we dealloc the slot, which bumps its generation — any stale token referring to it now rejects.

### Q: What's the memory ordering on the ring buffer?

A: Producer: relaxed load of own head, acquire load of consumer's tail (to see slot-freed publication), then write the slot, then release-store the new head (publishing the slot data to the consumer). Consumer is symmetric — relaxed on own tail, acquire on producer's head, read the slot, release-store new tail. The acquire/release pair is the synchronisation; the slot write happens between the acquire-load and the release-store on the producer, so the consumer is guaranteed to see it when its acquire-load picks up the new head. Standard Disruptor pattern.

### Q: Why pre-fault the arena?

A: Linux maps virtual memory lazily — the physical page isn't allocated until first touch, which triggers a minor page fault (a kernel trap, kernel-side allocation, TLB update, then resume). That's microseconds, fine in steady state but catastrophic for tail latency on the first orders. We touch every page once at startup so the OS has fully mapped the arena before trading starts. No page faults during trading.

### Q: Why a hot/cold book and not just one BTreeMap?

A: In active markets, 99% of orders sit within a few percent of the current price. For that 99%, a flat array indexed by `price - base_ticks` gives O(1) access with sequential memory layout — better than the O(log N) plus pointer chasing of BTreeMap. The tail of the distribution (limit orders far from the mid) is rare, so we let those fall into BTreeMap where O(log N) is fine. The hot book also caches `best_bid_idx` and `best_ask_idx`, so "best price" is a single integer read — no traversal.

### Q: What's the role of the user_id field in Order?

A: Self-trade prevention. When a taker matches a maker with the same `user_id`, we apply the configured STP policy: `CancelMaker` removes the maker and continues matching; `CancelTaker` cancels the incoming order; `CancelBoth` does both. This is a regulatory and risk-management requirement on most CEXes — accounts shouldn't trade with themselves. The check is on the hot path but only when `stp_mode != Off`, and the user_id lives in the second cache line of the Order so the common path (STP off) doesn't even load it.

### Q: How do you benchmark?

A: Two harnesses. (1) `kafka_producer` ([src/bin/kafka_producer.rs](matching-engine/src/bin/kafka_producer.rs)) generates a synthetic order stream via a geometric-Brownian-motion price walk with configurable fill/cancel ratios. (2) `kafka_bench` ([src/bin/kafka_bench.rs](matching-engine/src/bin/kafka_bench.rs)) runs the full pipeline (Kafka → engine → Kafka) and emits an HTML report of throughput and latency percentiles. Criterion benches cover micro-level pieces — ring buffer push/pop, slab alloc/dealloc, matcher with a primed book.

### Q: What's the worst case?

A: A sweep market order that walks the entire book. Cost is O(K) in the number of makers consumed. If K is large (thousands), we do thousands of slab deallocations and list unlinks — still microseconds total, but the p99 will reflect it. For protection, exchanges typically apply max-notional / price-band checks before the order reaches the engine. Our engine respects those upstream guards; it doesn't enforce them itself.

### Q: How would you go from 100K to 10M orders/sec?

A: A few directions, in order of cost. (1) Reduce per-order work — struct-of-arrays layout for hot fields, branchless match inner loop. (2) Faster Kafka — io_uring with submission-queue polling, kernel bypass for the ingress/egress threads. (3) Shard within a symbol — accept some loss of strict price-time priority within microsecond windows in exchange for parallel pre-validation, like a multi-stage pipeline where validation parallelises but the matching step stays single-threaded. (4) Hardware FPGA matching for the very top tier. But honestly, at 10M orders/sec the dominant problem becomes network and serialization, not matching.

### Q: Any unsafe code? Why is it sound?

A: Yes, in three places. (1) Ring buffer slot reads/writes — sound because the cursors enforce mutual exclusion on each slot index. The full/empty checks prove the slot is exclusively the producer's (or consumer's) at the moment of access. (2) Slab arena access — sound because the slab tracks occupancy via free list + generation, and we validate the token before each access. In debug builds we also track per-slot `occupied: bool` and assert. (3) Intrusive list pointer manipulation — sound because the slab indices come from validated tokens and the order is exclusively owned by the engine thread while we mutate it. Every unsafe block has a SAFETY comment documenting the invariant.

### Q: How is this different from LMAX Disruptor?

A: Same lineage and many of the same principles — single-threaded business logic, ring buffers at the boundaries, mechanical sympathy for the CPU. Differences: LMAX Disruptor is a Java framework with a more elaborate sequence-barrier protocol because it supports multiple consumer dependencies. We have exactly one consumer per ring, so we can use the simpler two-cursor SPSC design. We're also Rust, so `MaybeUninit` and `UnsafeCell` give us the same uninit-slot pattern without Java's null tax.

---

## 14. Soundbites for the Interview

- "Single thread, no locks, no allocations on the hot path. That's the whole design."
- "We don't use Tokio channels for ingress/egress — we use cache-padded SPSC Disruptor rings. CAS is not free."
- "The order book is hybrid hot/cold: flat-array O(1) for the body, BTreeMap O(log N) for the tails. 99% of accesses hit the array."
- "Cancels are O(1) thanks to intrusive linked lists in the slab — the order *is* the list node."
- "Tokens encode (slab_index, generation). Stale tokens reject instantly. No hash lookup, no ABA."
- "We pre-fault every page of the arena at startup. Zero page faults during trading."
- "Egress is panic-on-full, never drop — and we apply backpressure at the Kafka ingress thread so it never actually fills."
- "Two operating modes — dedicated for top pairs, shared for the long tail. Same binary, env-configured."

---

Open this doc next to the code. The most important files to be able to navigate cold in the interview:

1. [matching-engine/src/io/ring_buffer.rs](matching-engine/src/io/ring_buffer.rs) — 107 lines, know every one.
2. [matching-engine/src/matching/engine.rs:63–74](matching-engine/src/matching/engine.rs#L63-L74) — the run loop.
3. [matching-engine/src/types/order.rs:42–94](matching-engine/src/types/order.rs#L42-L94) — the Order layout.
4. [matching-engine/src/memory/slab.rs](matching-engine/src/memory/slab.rs) — alloc/dealloc/validate.
5. [matching-engine/src/orderbook/intrusive_list.rs](matching-engine/src/orderbook/intrusive_list.rs) — O(1) cancel.
6. [matching-engine/src/matching/matcher.rs](matching-engine/src/matching/matcher.rs) — the matching algorithm.

You built this. You know it. Walk in confident.
