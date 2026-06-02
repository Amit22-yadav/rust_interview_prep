# Actlogica Rust Interview — Reference Solutions

Idiomatic Rust solutions for **Tier S** (canonical interview problems) and **Domain** (fintech-specific) problems from [actlogica-rust-coding.md](actlogica-rust-coding.md).

**How to use**: read the approach first, try to solve from scratch, then compare. Don't memorize — internalize the patterns. Every solution compiles on stable Rust 1.75+.

---

## Table of contents

**Tier S — Must solve cold**
1. [LRU Cache](#1-lru-cache)
2. [Two Sum](#2-two-sum)
3. [Valid Parentheses](#3-valid-parentheses)
4. [Merge Intervals](#4-merge-intervals)
5. [Top K Frequent Elements](#5-top-k-frequent-elements)
6. [Word Frequency Counter](#6-word-frequency-counter)
7. [Binary Search Variants](#7-binary-search-variants)
8. [Producer–Consumer with Bounded Channel](#8-producerconsumer-with-bounded-channel)

**Domain — Fintech / Wealth Management**

9. [VWAP](#9-vwap-volume-weighted-average-price)
10. [Order Book Top of Book](#10-order-book-top-of-book)
11. [Portfolio NAV](#11-portfolio-nav)
12. [Reconciliation](#12-reconciliation-two-source-diff)
13. [Time-Series OHLC](#13-time-series-ohlc-aggregation)
14. [Position Tracker](#14-position-tracker)
15. [Interest Rate Calculator](#15-interest-rate-calculator)

**Tier A — Solve at least once**

16. [Longest Substring Without Repeating Characters](#16-longest-substring-without-repeating-characters)
17. [Group Anagrams](#17-group-anagrams)
18. [Product of Array Except Self](#18-product-of-array-except-self)
19. [Maximum Subarray (Kadane's)](#19-maximum-subarray-kadanes)
20. [Best Time to Buy and Sell Stock](#20-best-time-to-buy-and-sell-stock)
21. [Implement a Trie](#21-implement-a-trie)
22. [Number of Islands](#22-number-of-islands)
23. [Course Schedule (Topological Sort)](#23-course-schedule-topological-sort)
24. [Kth Largest Element](#24-kth-largest-element)
25. [Merge K Sorted Lists](#25-merge-k-sorted-lists)
26. [Subsets / Permutations / Combinations](#26-subsets--permutations--combinations)
27. [Coin Change (DP)](#27-coin-change-dp)

**Tier B — Know the approach**

28. [Word Break](#28-word-break)
29. [Longest Palindromic Substring](#29-longest-palindromic-substring)
30. [Rotate Image](#30-rotate-image)
31. [Spiral Matrix](#31-spiral-matrix)
32. [Set Matrix Zeroes](#32-set-matrix-zeroes)
33. [Search in Rotated Sorted Array](#33-search-in-rotated-sorted-array)
34. [Linked List Cycle](#34-linked-list-cycle)
35. [Reverse Linked List](#35-reverse-linked-list)

**Concurrency & Async**

36. [Thread-Safe Counter](#36-thread-safe-counter)
37. [Rate Limiter (Token Bucket)](#37-rate-limiter-token-bucket)
38. [Concurrent Bounded Queue](#38-concurrent-bounded-queue)
39. [Fan-out / Fan-in](#39-fan-out--fan-in)
40. [Graceful Shutdown](#40-graceful-shutdown)
41. [Implement a Future from Scratch](#41-implement-a-future-from-scratch)
42. [Deadlock Detection / Avoidance](#42-deadlock-detection--avoidance)
43. [Channel-Based State Machine (Actor)](#43-channel-based-state-machine-actor)

**System Design coding-adjacent**

44. [URL Shortener](#44-url-shortener)
45. [Distributed Counter](#45-distributed-counter)
46. [Idempotent Payment API](#46-idempotent-payment-api)
47. [Retry Policy with Exponential Backoff](#47-retry-policy-with-exponential-backoff)

**Rust conceptual Q&A** — [section](#rust-conceptual-questions)

---

# Tier S — Solutions

## 1. LRU Cache

**Problem**: O(1) `get` and `put` for least-recently-used cache.

**Approach**: `HashMap<K, usize>` mapping keys to indices in a `VecDeque` is the simplest in Rust. The clean textbook answer uses a doubly linked list with `Rc<RefCell<Node>>`, but it's verbose and error-prone — most interviewers accept the cleaner version below.

**Idiomatic version** — uses a generation counter to avoid expensive `VecDeque` shifts:

```rust
use std::collections::HashMap;

pub struct LRUCache {
    capacity: usize,
    map: HashMap<i32, (i32, u64)>, // key -> (value, last_used_tick)
    tick: u64,
}

impl LRUCache {
    pub fn new(capacity: i32) -> Self {
        Self {
            capacity: capacity as usize,
            map: HashMap::with_capacity(capacity as usize),
            tick: 0,
        }
    }

    pub fn get(&mut self, key: i32) -> i32 {
        self.tick += 1;
        match self.map.get_mut(&key) {
            Some((value, last_used)) => {
                *last_used = self.tick;
                *value
            }
            None => -1,
        }
    }

    pub fn put(&mut self, key: i32, value: i32) {
        self.tick += 1;
        if self.map.contains_key(&key) {
            self.map.insert(key, (value, self.tick));
            return;
        }
        if self.map.len() >= self.capacity {
            // Evict LRU
            if let Some((&evict_key, _)) = self.map.iter().min_by_key(|(_, (_, t))| *t) {
                self.map.remove(&evict_key);
            }
        }
        self.map.insert(key, (value, self.tick));
    }
}
```

**Caveat to mention out loud**: "This eviction is O(n). For true O(1), I'd add a doubly-linked-list ordering with `Rc<RefCell<Node>>`, or use the `linked_hash_map` crate." Saying this earns you the technical credit without the bug-prone code.

**Production version** with proper doubly-linked O(1):

```rust
use std::collections::HashMap;

struct Node {
    key: i32,
    value: i32,
    prev: Option<usize>,
    next: Option<usize>,
}

pub struct LRUCacheFast {
    capacity: usize,
    nodes: Vec<Node>,
    free: Vec<usize>,
    map: HashMap<i32, usize>,
    head: Option<usize>,
    tail: Option<usize>,
}

impl LRUCacheFast {
    pub fn new(capacity: i32) -> Self {
        Self {
            capacity: capacity as usize,
            nodes: Vec::with_capacity(capacity as usize),
            free: Vec::new(),
            map: HashMap::with_capacity(capacity as usize),
            head: None,
            tail: None,
        }
    }

    fn detach(&mut self, idx: usize) {
        let (prev, next) = (self.nodes[idx].prev, self.nodes[idx].next);
        match prev {
            Some(p) => self.nodes[p].next = next,
            None => self.head = next,
        }
        match next {
            Some(n) => self.nodes[n].prev = prev,
            None => self.tail = prev,
        }
    }

    fn push_front(&mut self, idx: usize) {
        self.nodes[idx].prev = None;
        self.nodes[idx].next = self.head;
        if let Some(h) = self.head {
            self.nodes[h].prev = Some(idx);
        }
        self.head = Some(idx);
        if self.tail.is_none() {
            self.tail = Some(idx);
        }
    }

    pub fn get(&mut self, key: i32) -> i32 {
        if let Some(&idx) = self.map.get(&key) {
            self.detach(idx);
            self.push_front(idx);
            self.nodes[idx].value
        } else {
            -1
        }
    }

    pub fn put(&mut self, key: i32, value: i32) {
        if let Some(&idx) = self.map.get(&key) {
            self.nodes[idx].value = value;
            self.detach(idx);
            self.push_front(idx);
            return;
        }

        let idx = if self.map.len() >= self.capacity {
            let evict_idx = self.tail.unwrap();
            self.map.remove(&self.nodes[evict_idx].key);
            self.detach(evict_idx);
            self.nodes[evict_idx] = Node { key, value, prev: None, next: None };
            evict_idx
        } else if let Some(free_idx) = self.free.pop() {
            self.nodes[free_idx] = Node { key, value, prev: None, next: None };
            free_idx
        } else {
            self.nodes.push(Node { key, value, prev: None, next: None });
            self.nodes.len() - 1
        };

        self.push_front(idx);
        self.map.insert(key, idx);
    }
}
```

**Complexity**: O(1) get/put, O(capacity) space.

---

## 2. Two Sum

```rust
use std::collections::HashMap;

pub fn two_sum(nums: Vec<i32>, target: i32) -> Vec<i32> {
    let mut seen: HashMap<i32, i32> = HashMap::new();
    for (i, &n) in nums.iter().enumerate() {
        let complement = target - n;
        if let Some(&j) = seen.get(&complement) {
            return vec![j, i as i32];
        }
        seen.insert(n, i as i32);
    }
    vec![]
}
```

**Talking points**: O(n) time, O(n) space. Single pass — the trick is checking *before* inserting so you don't match yourself.

---

## 3. Valid Parentheses

```rust
pub fn is_valid(s: String) -> bool {
    let mut stack: Vec<char> = Vec::with_capacity(s.len());
    for c in s.chars() {
        match c {
            '(' | '[' | '{' => stack.push(c),
            ')' => if stack.pop() != Some('(') { return false; },
            ']' => if stack.pop() != Some('[') { return false; },
            '}' => if stack.pop() != Some('{') { return false; },
            _ => return false,
        }
    }
    stack.is_empty()
}
```

**Talking points**: O(n) time/space. Pattern matching on `char` is idiomatic. Mention that `s.chars()` is correct because `String` is UTF-8 — `s.bytes()` would also work here since all relevant chars are ASCII.

---

## 4. Merge Intervals

```rust
pub fn merge(mut intervals: Vec<Vec<i32>>) -> Vec<Vec<i32>> {
    if intervals.is_empty() {
        return vec![];
    }
    intervals.sort_by_key(|v| v[0]);
    let mut result: Vec<Vec<i32>> = Vec::with_capacity(intervals.len());
    result.push(intervals[0].clone());

    for interval in intervals.into_iter().skip(1) {
        let last = result.last_mut().unwrap();
        if interval[0] <= last[1] {
            last[1] = last[1].max(interval[1]);
        } else {
            result.push(interval);
        }
    }
    result
}
```

**Talking points**: O(n log n) due to sort. Sort by start, then walk and merge overlaps in one pass. `last_mut()` returns `Option<&mut T>` — explain why.

---

## 5. Top K Frequent Elements

```rust
use std::collections::{BinaryHeap, HashMap};
use std::cmp::Reverse;

pub fn top_k_frequent(nums: Vec<i32>, k: i32) -> Vec<i32> {
    let mut counts: HashMap<i32, i32> = HashMap::new();
    for n in nums {
        *counts.entry(n).or_insert(0) += 1;
    }

    // Min-heap of size k: keep the k largest by frequency
    let mut heap: BinaryHeap<Reverse<(i32, i32)>> = BinaryHeap::with_capacity(k as usize + 1);
    for (num, count) in counts {
        heap.push(Reverse((count, num)));
        if heap.len() > k as usize {
            heap.pop();
        }
    }

    heap.into_iter().map(|Reverse((_, n))| n).collect()
}
```

**Talking points**: O(n log k) time. `Reverse` flips `BinaryHeap` (max-heap by default) into a min-heap. `entry().or_insert(0)` is idiomatic for counting.

**Bucket-sort variant** (O(n) but more memory):

```rust
pub fn top_k_frequent_bucket(nums: Vec<i32>, k: i32) -> Vec<i32> {
    let n = nums.len();
    let mut counts: HashMap<i32, i32> = HashMap::new();
    for x in nums { *counts.entry(x).or_insert(0) += 1; }

    let mut buckets: Vec<Vec<i32>> = vec![Vec::new(); n + 1];
    for (num, c) in counts {
        buckets[c as usize].push(num);
    }

    let mut result = Vec::with_capacity(k as usize);
    for bucket in buckets.iter().rev() {
        for &num in bucket {
            if result.len() == k as usize { return result; }
            result.push(num);
        }
    }
    result
}
```

---

## 6. Word Frequency Counter

**In-memory version**:

```rust
use std::collections::HashMap;
use std::io::{self, BufRead};

pub fn top_n_words(reader: impl BufRead, n: usize) -> Vec<(String, u64)> {
    let mut counts: HashMap<String, u64> = HashMap::new();

    for line in reader.lines() {
        let line = match line {
            Ok(l) => l,
            Err(_) => continue,
        };
        for word in line.split_whitespace() {
            let normalized = word.trim_matches(|c: char| !c.is_alphanumeric()).to_lowercase();
            if normalized.is_empty() { continue; }
            *counts.entry(normalized).or_insert(0) += 1;
        }
    }

    let mut pairs: Vec<(String, u64)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    pairs.truncate(n);
    pairs
}

fn main() {
    let stdin = io::stdin();
    let top = top_n_words(stdin.lock(), 10);
    for (word, count) in top {
        println!("{}: {}", word, count);
    }
}
```

**Streaming/scale talking points** (mention these out loud):
- For data that doesn't fit in memory: external sort, then group-and-count
- For approximate top-K on a stream: **count-min sketch** + **min-heap of size K**
- For distributed: shard by `hash(word) % N`, then merge top-K from each shard

---

## 7. Binary Search Variants

**Standard binary search**:

```rust
pub fn binary_search(nums: &[i32], target: i32) -> i32 {
    let (mut lo, mut hi) = (0i32, nums.len() as i32 - 1);
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        match nums[mid as usize].cmp(&target) {
            std::cmp::Ordering::Equal => return mid,
            std::cmp::Ordering::Less => lo = mid + 1,
            std::cmp::Ordering::Greater => hi = mid - 1,
        }
    }
    -1
}
```

**First occurrence (leftmost)** — the more useful interview variant:

```rust
pub fn first_occurrence(nums: &[i32], target: i32) -> i32 {
    let (mut lo, mut hi) = (0usize, nums.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if nums[mid] < target {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    if lo < nums.len() && nums[lo] == target { lo as i32 } else { -1 }
}
```

**Last occurrence**:

```rust
pub fn last_occurrence(nums: &[i32], target: i32) -> i32 {
    let (mut lo, mut hi) = (0i64, nums.len() as i64);
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if nums[mid as usize] <= target {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    let idx = lo - 1;
    if idx >= 0 && nums[idx as usize] == target { idx as i32 } else { -1 }
}
```

**Talking points**: The half-open `[lo, hi)` invariant is the safer pattern — fewer off-by-one bugs than `[lo, hi]`. Practice both first/last so you can derive the other on the fly.

**Idiomatic Rust note**: in real code, use the standard library — `slice::binary_search` and `slice::partition_point`. Mention this: *"In production I'd use `slice::partition_point` for first-occurrence — it's already in std."*

---

## 8. Producer–Consumer with Bounded Channel

```rust
use tokio::sync::mpsc;
use tokio::task::JoinSet;
use std::time::Duration;

#[derive(Debug, Clone)]
struct WorkItem {
    id: u64,
    payload: String,
}

async fn producer(id: u64, tx: mpsc::Sender<WorkItem>, count: u64) {
    for i in 0..count {
        let item = WorkItem {
            id: id * 1000 + i,
            payload: format!("producer-{} item-{}", id, i),
        };
        if tx.send(item).await.is_err() {
            // Channel closed — receivers all dropped
            break;
        }
    }
}

async fn consumer(id: u64, mut rx: mpsc::Receiver<WorkItem>) -> u64 {
    let mut processed = 0u64;
    while let Some(item) = rx.recv().await {
        // Simulate work
        tokio::time::sleep(Duration::from_micros(10)).await;
        processed += 1;
        if processed % 100 == 0 {
            println!("consumer {} processed {} items (last: {})", id, processed, item.id);
        }
    }
    processed
}

#[tokio::main]
async fn main() {
    const N_PRODUCERS: u64 = 4;
    const N_CONSUMERS: u64 = 2;
    const ITEMS_PER_PRODUCER: u64 = 1000;
    const CHANNEL_CAPACITY: usize = 32;

    // Single channel: producers share senders, consumers compete on receiver.
    // For mpsc, we need to clone Sender across producers, and the single Receiver
    // is consumed by one task. For multiple consumers, use a broadcast or split work.

    // Approach: one channel, one consumer task. To scale consumers, use multiple channels
    // OR use `async_channel`/`flume` which support mpmc.
    let (tx, rx) = mpsc::channel::<WorkItem>(CHANNEL_CAPACITY);

    let mut producers = JoinSet::new();
    for i in 0..N_PRODUCERS {
        let tx_clone = tx.clone();
        producers.spawn(async move {
            producer(i, tx_clone, ITEMS_PER_PRODUCER).await;
        });
    }
    // Drop the original sender so the channel closes when all producers finish
    drop(tx);

    let consumer_handle = tokio::spawn(consumer(0, rx));

    while producers.join_next().await.is_some() {}
    let processed = consumer_handle.await.unwrap();
    println!("total processed: {} (expected: {})", processed, N_PRODUCERS * ITEMS_PER_PRODUCER);
}
```

**Talking points**:
- `mpsc` = multi-producer, single-consumer. For multiple consumers, mention `async_channel`, `flume`, or split-by-routing.
- **The drop-original-tx pattern** is critical — without it, the consumer hangs forever waiting on a channel that never closes.
- Backpressure: when the channel is full, `tx.send().await` suspends — that's automatic backpressure.
- `JoinSet` cleans up cancelled tasks on drop. Use it instead of bare `Vec<JoinHandle>`.

**`Cargo.toml`**:
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### Variant: Multi-Producer, Multi-Consumer (true MPMC)

`tokio::sync::mpsc` only supports a **single consumer** (one task owns the `Receiver`). For multiple consumers competing on the same queue, use `async_channel` (or `flume`), both of which are mpmc:

`Cargo.toml`:
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
async-channel = "2"
```

```rust
use async_channel::{bounded, Receiver, Sender};
use std::time::Duration;
use tokio::task::JoinSet;

#[derive(Debug, Clone)]
struct WorkItem {
    id: u64,
    payload: String,
}

async fn producer(id: u64, tx: Sender<WorkItem>, count: u64) {
    for i in 0..count {
        let item = WorkItem {
            id: id * 1_000_000 + i,
            payload: format!("p-{} item-{}", id, i),
        };
        // `send` suspends when the channel is full → automatic backpressure
        if tx.send(item).await.is_err() {
            break; // all receivers dropped
        }
    }
}

async fn consumer(id: u64, rx: Receiver<WorkItem>) -> u64 {
    let mut processed = 0u64;
    // `recv()` returns Err when senders are all dropped AND queue is empty
    while let Ok(item) = rx.recv().await {
        // Simulate per-item work
        tokio::time::sleep(Duration::from_micros(20)).await;
        processed += 1;
        if processed % 200 == 0 {
            println!("consumer {} processed {} (last id {})", id, processed, item.id);
        }
    }
    println!("consumer {} done, total {}", id, processed);
    processed
}

#[tokio::main]
async fn main() {
    const N_PRODUCERS: u64 = 4;
    const N_CONSUMERS: u64 = 3;
    const ITEMS_PER_PRODUCER: u64 = 5_000;
    const CHANNEL_CAPACITY: usize = 64;

    let (tx, rx) = bounded::<WorkItem>(CHANNEL_CAPACITY);

    // Spawn producers — each gets a Sender clone
    let mut producers = JoinSet::new();
    for i in 0..N_PRODUCERS {
        let tx_clone = tx.clone();
        producers.spawn(async move {
            producer(i, tx_clone, ITEMS_PER_PRODUCER).await;
        });
    }
    // Drop the original sender so the channel closes once all producers finish.
    // Without this drop, consumers wait forever on the never-closing channel.
    drop(tx);

    // Spawn consumers — each gets a Receiver clone. async_channel's Receiver
    // is Clone; competing receivers race on each item (work-stealing semantics).
    let mut consumers = JoinSet::new();
    for i in 0..N_CONSUMERS {
        let rx_clone = rx.clone();
        consumers.spawn(async move { consumer(i, rx_clone).await });
    }
    drop(rx);

    while producers.join_next().await.is_some() {}

    let mut total = 0u64;
    while let Some(res) = consumers.join_next().await {
        total += res.unwrap();
    }
    println!("grand total: {} (expected {})", total, N_PRODUCERS * ITEMS_PER_PRODUCER);
}
```

**Talking points for the MPMC variant**:
- **`async_channel` / `flume` / `kanal`** are the go-to crates for true mpmc in async Rust. `flume` is often the fastest; `async_channel` is the simplest API and from the smol-rs/async-std ecosystem.
- **Work-stealing semantics**: each item is delivered to exactly one consumer. Whichever consumer is currently parked on `recv()` and gets scheduled first wins the item. This naturally balances load.
- **Shutdown still needs the drop trick** — drop the original `tx` AND original `rx` so the channel cleanly closes when all clones go out of scope.
- **`tokio::sync::broadcast`** is a different beast — every consumer gets every item (fan-out, not load-balanced). Use it for pub-sub, not work distribution.

### Variant: Split-by-Routing (multiple mpsc channels, one per consumer)

If you must stick with `tokio::sync::mpsc` (no extra deps), give each consumer its own channel and route items deterministically — e.g., by `hash(key) % n_consumers`. This also gives you **per-key ordering**, which mpmc work-stealing does not:

```rust
use tokio::sync::mpsc;
use tokio::task::JoinSet;

#[derive(Debug)]
struct Order {
    user_id: u64,
    amount: u64,
}

fn shard_for(user_id: u64, n_shards: usize) -> usize {
    // Same user always lands on the same consumer → preserves ordering per user
    (user_id as usize) % n_shards
}

#[tokio::main]
async fn main() {
    const N_CONSUMERS: usize = 4;
    const CAPACITY: usize = 32;

    // One channel per consumer
    let mut senders = Vec::with_capacity(N_CONSUMERS);
    let mut consumers = JoinSet::new();

    for cid in 0..N_CONSUMERS {
        let (tx, mut rx) = mpsc::channel::<Order>(CAPACITY);
        senders.push(tx);
        consumers.spawn(async move {
            let mut processed = 0u64;
            while let Some(order) = rx.recv().await {
                // Per-user ordering preserved within this consumer
                processed += 1;
                println!("consumer {} got user={} amount={}", cid, order.user_id, order.amount);
            }
            processed
        });
    }

    // Producer routes by user_id
    let producer = tokio::spawn(async move {
        for i in 0..20u64 {
            let order = Order { user_id: i % 8, amount: 100 + i };
            let shard = shard_for(order.user_id, N_CONSUMERS);
            senders[shard].send(order).await.unwrap();
        }
        // Drop all senders → all consumers' channels close
        drop(senders);
    });

    producer.await.unwrap();
    let mut total = 0u64;
    while let Some(res) = consumers.join_next().await {
        total += res.unwrap();
    }
    println!("total processed: {}", total);
}
```

**Talking points for split-by-routing**:
- **Per-key ordering** — events for the same key never cross consumers, so processing order is preserved. This is critical for stateful processing (e.g., per-user balance updates, per-symbol order books).
- **No mpmc dependency** — pure `tokio::sync::mpsc`.
- **Downside**: load skew if your key distribution is uneven (a hot key floods one consumer). Mitigate with consistent hashing or by sharding hot keys further.
- **This is how Kafka consumer groups work** under the hood — partitions are routed to consumers, ordering is preserved per partition.

**Quick decision guide**:
- **Single consumer, multiple producers** → `tokio::sync::mpsc` (built-in, zero deps)
- **Multiple consumers, work-stealing, no ordering constraint** → `async_channel` or `flume` (mpmc)
- **Multiple consumers, must preserve per-key ordering** → split-by-routing with N × mpsc
- **Multiple consumers, every consumer gets every message** → `tokio::sync::broadcast` (pub-sub, not work distribution)

---

# Domain — Solutions

These problems use [`rust_decimal`](https://crates.io/crates/rust_decimal) for money arithmetic. Saying "I'd use `rust_decimal` instead of `f64` for money" in the interview is a strong signal.

`Cargo.toml`:
```toml
[dependencies]
rust_decimal = "1"
rust_decimal_macros = "1"
chrono = "0.4"
```

---

## 9. VWAP (Volume-Weighted Average Price)

**Problem**: VWAP = Σ(price × volume) / Σ(volume), per symbol.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Trade {
    pub symbol: String,
    pub price: Decimal,
    pub quantity: Decimal,
    pub timestamp_ms: i64,
}

pub fn vwap_by_symbol(trades: &[Trade]) -> HashMap<String, Decimal> {
    // (sum_price_times_volume, sum_volume) per symbol
    let mut acc: HashMap<String, (Decimal, Decimal)> = HashMap::new();

    for t in trades {
        let entry = acc.entry(t.symbol.clone()).or_insert((dec!(0), dec!(0)));
        entry.0 += t.price * t.quantity;
        entry.1 += t.quantity;
    }

    acc.into_iter()
        .filter_map(|(symbol, (pv_sum, v_sum))| {
            if v_sum.is_zero() { None } else { Some((symbol, pv_sum / v_sum)) }
        })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn vwap_basic() {
        let trades = vec![
            Trade { symbol: "INFY".into(), price: dec!(1500), quantity: dec!(10), timestamp_ms: 1 },
            Trade { symbol: "INFY".into(), price: dec!(1510), quantity: dec!(20), timestamp_ms: 2 },
            Trade { symbol: "TCS".into(),  price: dec!(3000), quantity: dec!(5),  timestamp_ms: 3 },
        ];
        let result = vwap_by_symbol(&trades);
        // INFY: (1500*10 + 1510*20) / 30 = (15000 + 30200) / 30 = 45200/30 = 1506.666...
        assert_eq!(result["TCS"], dec!(3000));
        // Decimal precision-aware comparison
        let infy = result["INFY"];
        assert!((infy - dec!(1506.6666666666666666666666667)).abs() < dec!(0.0000001));
    }
}
```

**Time-windowed variant** — VWAP over the last N seconds:

```rust
pub fn vwap_windowed(trades: &[Trade], symbol: &str, window_ms: i64, now_ms: i64) -> Option<Decimal> {
    let cutoff = now_ms - window_ms;
    let (pv_sum, v_sum) = trades.iter()
        .filter(|t| t.symbol == symbol && t.timestamp_ms >= cutoff)
        .fold((dec!(0), dec!(0)), |(pv, v), t| (pv + t.price * t.quantity, v + t.quantity));

    if v_sum.is_zero() { None } else { Some(pv_sum / v_sum) }
}
```

**Talking points**: O(n) time. Discuss numerical stability: for a streaming version, you'd maintain running `(pv_sum, v_sum)` and update incrementally — but be aware of catastrophic cancellation if you ever subtract (e.g., for trailing windows).

---

## 10. Order Book Top of Book

**Problem**: Maintain best bid (highest buy price) and best ask (lowest sell price) under add/cancel.

```rust
use rust_decimal::Decimal;
use std::collections::BTreeMap;
use std::cmp::Reverse;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side {
    Bid,
    Ask,
}

#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub side: Side,
    pub price: Decimal,
    pub quantity: Decimal,
}

pub struct OrderBook {
    // For bids, we want highest price first → use Reverse as key
    bids: BTreeMap<Reverse<Decimal>, Decimal>, // price -> aggregated quantity
    asks: BTreeMap<Decimal, Decimal>,
    orders: std::collections::HashMap<u64, (Side, Decimal, Decimal)>, // id -> (side, price, qty)
}

impl OrderBook {
    pub fn new() -> Self {
        Self {
            bids: BTreeMap::new(),
            asks: BTreeMap::new(),
            orders: std::collections::HashMap::new(),
        }
    }

    pub fn add(&mut self, order: Order) {
        self.orders.insert(order.id, (order.side, order.price, order.quantity));
        match order.side {
            Side::Bid => {
                *self.bids.entry(Reverse(order.price)).or_insert(Decimal::ZERO) += order.quantity;
            }
            Side::Ask => {
                *self.asks.entry(order.price).or_insert(Decimal::ZERO) += order.quantity;
            }
        }
    }

    pub fn cancel(&mut self, id: u64) -> bool {
        let Some((side, price, qty)) = self.orders.remove(&id) else { return false; };
        match side {
            Side::Bid => {
                if let Some(level_qty) = self.bids.get_mut(&Reverse(price)) {
                    *level_qty -= qty;
                    if level_qty.is_zero() {
                        self.bids.remove(&Reverse(price));
                    }
                }
            }
            Side::Ask => {
                if let Some(level_qty) = self.asks.get_mut(&price) {
                    *level_qty -= qty;
                    if level_qty.is_zero() {
                        self.asks.remove(&price);
                    }
                }
            }
        }
        true
    }

    pub fn best_bid(&self) -> Option<(Decimal, Decimal)> {
        self.bids.iter().next().map(|(Reverse(price), qty)| (*price, *qty))
    }

    pub fn best_ask(&self) -> Option<(Decimal, Decimal)> {
        self.asks.iter().next().map(|(price, qty)| (*price, *qty))
    }

    pub fn spread(&self) -> Option<Decimal> {
        let bid = self.best_bid()?.0;
        let ask = self.best_ask()?.0;
        Some(ask - bid)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use rust_decimal_macros::dec;

    #[test]
    fn order_book_top() {
        let mut book = OrderBook::new();
        book.add(Order { id: 1, side: Side::Bid, price: dec!(100), quantity: dec!(10) });
        book.add(Order { id: 2, side: Side::Bid, price: dec!(101), quantity: dec!(5) });
        book.add(Order { id: 3, side: Side::Ask, price: dec!(103), quantity: dec!(7) });
        book.add(Order { id: 4, side: Side::Ask, price: dec!(102), quantity: dec!(3) });

        assert_eq!(book.best_bid(), Some((dec!(101), dec!(5))));
        assert_eq!(book.best_ask(), Some((dec!(102), dec!(3))));
        assert_eq!(book.spread(), Some(dec!(1)));

        book.cancel(2);
        assert_eq!(book.best_bid(), Some((dec!(100), dec!(10))));
    }
}
```

**Talking points**:
- `BTreeMap::iter().next()` is O(log n) for the first element — best bid/ask in log time
- For real matching engines, the price-level aggregation is critical (many orders at same price)
- Mention: "For a real exchange I'd use a price-time-priority FIFO queue per price level, not just aggregated quantity — but for top-of-book the aggregate is enough."
- **This problem is your matching engine in miniature.** Lean in.

---

## 11. Portfolio NAV

**Problem**: Given holdings and prices, compute total portfolio value.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::collections::HashMap;

#[derive(Debug)]
pub struct Holding {
    pub symbol: String,
    pub quantity: Decimal,
}

#[derive(Debug, Clone)]
pub struct PriceTick {
    pub symbol: String,
    pub price: Decimal,
    pub timestamp_ms: i64,
}

#[derive(Debug)]
pub enum NavError {
    MissingPrice(String),
    StalePrice { symbol: String, age_ms: i64 },
}

pub struct PriceCache {
    prices: HashMap<String, PriceTick>,
    max_age_ms: i64,
}

impl PriceCache {
    pub fn new(max_age_ms: i64) -> Self {
        Self { prices: HashMap::new(), max_age_ms }
    }

    pub fn update(&mut self, tick: PriceTick) {
        // Only keep the latest tick per symbol
        match self.prices.get(&tick.symbol) {
            Some(existing) if existing.timestamp_ms >= tick.timestamp_ms => return,
            _ => { self.prices.insert(tick.symbol.clone(), tick); }
        }
    }

    pub fn compute_nav(&self, holdings: &[Holding], now_ms: i64) -> Result<Decimal, NavError> {
        let mut total = dec!(0);
        for h in holdings {
            let tick = self.prices.get(&h.symbol)
                .ok_or_else(|| NavError::MissingPrice(h.symbol.clone()))?;
            let age = now_ms - tick.timestamp_ms;
            if age > self.max_age_ms {
                return Err(NavError::StalePrice { symbol: h.symbol.clone(), age_ms: age });
            }
            total += h.quantity * tick.price;
        }
        Ok(total)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn nav_basic() {
        let mut cache = PriceCache::new(60_000);
        cache.update(PriceTick { symbol: "INFY".into(), price: dec!(1500), timestamp_ms: 1000 });
        cache.update(PriceTick { symbol: "TCS".into(),  price: dec!(3000), timestamp_ms: 1000 });

        let holdings = vec![
            Holding { symbol: "INFY".into(), quantity: dec!(100) },
            Holding { symbol: "TCS".into(),  quantity: dec!(50) },
        ];

        let nav = cache.compute_nav(&holdings, 30_000).unwrap();
        assert_eq!(nav, dec!(300_000)); // 100*1500 + 50*3000
    }

    #[test]
    fn nav_stale_price() {
        let mut cache = PriceCache::new(60_000);
        cache.update(PriceTick { symbol: "INFY".into(), price: dec!(1500), timestamp_ms: 1000 });

        let holdings = vec![Holding { symbol: "INFY".into(), quantity: dec!(100) }];
        // now is 1000 + 70_000ms > max_age 60_000
        let result = cache.compute_nav(&holdings, 71_000);
        assert!(matches!(result, Err(NavError::StalePrice { .. })));
    }
}
```

**Talking points**:
- **Stale-price detection** is the key signal. Mention: "In production, computing NAV with a 1-hour-old price is a correctness bug, not just bad data."
- For thousands of portfolios: parallel compute with `rayon::par_iter`
- Result type with custom errors is idiomatic — beats panicking or returning sentinel values
- Mention: "I'd add observability — emit a metric for `nav_compute_failures_total{reason=...}` so we can alert on missing prices."

---

## 12. Reconciliation: Two-Source Diff

**Problem**: Compare two lists of trades — internal ledger vs broker feed. Surface mismatches with tolerance for floating-point amounts.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct TradeKey {
    pub symbol: String,
    pub trade_date: String, // YYYY-MM-DD
    pub broker_ref: String, // unique reference shared between sources
}

#[derive(Debug, Clone)]
pub struct Trade {
    pub key: TradeKey,
    pub quantity: Decimal,
    pub price: Decimal,
}

#[derive(Debug)]
pub enum ReconResult {
    Matched,
    OnlyInInternal(Trade),
    OnlyInBroker(Trade),
    AmountMismatch { internal: Trade, broker: Trade, qty_diff: Decimal, price_diff: Decimal },
}

pub fn reconcile(
    internal: &[Trade],
    broker: &[Trade],
    qty_tolerance: Decimal,
    price_tolerance: Decimal,
) -> Vec<ReconResult> {
    let internal_map: HashMap<&TradeKey, &Trade> = internal.iter().map(|t| (&t.key, t)).collect();
    let broker_map: HashMap<&TradeKey, &Trade> = broker.iter().map(|t| (&t.key, t)).collect();

    let mut results = Vec::new();

    for (key, int_trade) in &internal_map {
        match broker_map.get(key) {
            None => results.push(ReconResult::OnlyInInternal((*int_trade).clone())),
            Some(brk_trade) => {
                let qty_diff = (int_trade.quantity - brk_trade.quantity).abs();
                let price_diff = (int_trade.price - brk_trade.price).abs();
                if qty_diff > qty_tolerance || price_diff > price_tolerance {
                    results.push(ReconResult::AmountMismatch {
                        internal: (*int_trade).clone(),
                        broker: (*brk_trade).clone(),
                        qty_diff,
                        price_diff,
                    });
                } else {
                    results.push(ReconResult::Matched);
                }
            }
        }
    }

    for (key, brk_trade) in &broker_map {
        if !internal_map.contains_key(key) {
            results.push(ReconResult::OnlyInBroker((*brk_trade).clone()));
        }
    }

    results
}

pub fn summarize(results: &[ReconResult]) -> (usize, usize, usize, usize) {
    let (mut matched, mut only_int, mut only_brk, mut mismatch) = (0, 0, 0, 0);
    for r in results {
        match r {
            ReconResult::Matched => matched += 1,
            ReconResult::OnlyInInternal(_) => only_int += 1,
            ReconResult::OnlyInBroker(_) => only_brk += 1,
            ReconResult::AmountMismatch { .. } => mismatch += 1,
        }
    }
    (matched, only_int, only_brk, mismatch)
}

#[cfg(test)]
mod tests {
    use super::*;

    fn key(s: &str, d: &str, r: &str) -> TradeKey {
        TradeKey { symbol: s.into(), trade_date: d.into(), broker_ref: r.into() }
    }

    #[test]
    fn recon_basic() {
        let internal = vec![
            Trade { key: key("INFY", "2026-06-01", "R1"), quantity: dec!(100), price: dec!(1500) },
            Trade { key: key("TCS",  "2026-06-01", "R2"), quantity: dec!(50),  price: dec!(3000) },
            Trade { key: key("HDFC", "2026-06-01", "R3"), quantity: dec!(200), price: dec!(1700) }, // only in internal
        ];
        let broker = vec![
            Trade { key: key("INFY", "2026-06-01", "R1"), quantity: dec!(100), price: dec!(1500) },
            Trade { key: key("TCS",  "2026-06-01", "R2"), quantity: dec!(50),  price: dec!(3001) }, // price diff
            Trade { key: key("WIPRO","2026-06-01", "R4"), quantity: dec!(80),  price: dec!(450) },  // only in broker
        ];

        let results = reconcile(&internal, &broker, dec!(0), dec!(0.5));
        let (m, oi, ob, mm) = summarize(&results);
        assert_eq!(m, 1);
        assert_eq!(oi, 1);
        assert_eq!(ob, 1);
        assert_eq!(mm, 1);
    }
}
```

**Talking points**:
- Hash-join approach is O(n + m). For sorted inputs, a merge-walk is O(n + m) with O(1) extra space.
- **Tolerance is critical**: floating-point money differences of ₹0.01 are usually rounding, not real mismatches. The tolerance parameter externalizes this judgment.
- For production: store recon results in a database for audit trail; alert on mismatch rate above threshold.
- Mention: "For very large recons (millions of trades), I'd shard by `hash(broker_ref) % N` and run per-shard in parallel with `rayon`."

---

## 13. Time-Series OHLC Aggregation

**Problem**: Given a stream of price ticks, emit 1-minute OHLC candles per symbol.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct Tick {
    pub symbol: String,
    pub price: Decimal,
    pub timestamp_ms: i64,
}

#[derive(Debug, Clone, PartialEq)]
pub struct Candle {
    pub symbol: String,
    pub bucket_start_ms: i64,
    pub open: Decimal,
    pub high: Decimal,
    pub low: Decimal,
    pub close: Decimal,
    pub tick_count: u64,
}

pub struct OhlcAggregator {
    bucket_size_ms: i64,
    open: HashMap<(String, i64), Candle>,
}

impl OhlcAggregator {
    pub fn new(bucket_size_ms: i64) -> Self {
        Self { bucket_size_ms, open: HashMap::new() }
    }

    pub fn ingest(&mut self, tick: &Tick) {
        let bucket = (tick.timestamp_ms / self.bucket_size_ms) * self.bucket_size_ms;
        let key = (tick.symbol.clone(), bucket);

        self.open.entry(key)
            .and_modify(|c| {
                c.high = c.high.max(tick.price);
                c.low = c.low.min(tick.price);
                c.close = tick.price;
                c.tick_count += 1;
            })
            .or_insert(Candle {
                symbol: tick.symbol.clone(),
                bucket_start_ms: bucket,
                open: tick.price,
                high: tick.price,
                low: tick.price,
                close: tick.price,
                tick_count: 1,
            });
    }

    /// Close all buckets whose end-time is ≤ `now_ms`. Returns the finalized candles.
    pub fn close_expired(&mut self, now_ms: i64) -> Vec<Candle> {
        let mut finalized = Vec::new();
        self.open.retain(|(_sym, bucket), candle| {
            if *bucket + self.bucket_size_ms <= now_ms {
                finalized.push(candle.clone());
                false
            } else {
                true
            }
        });
        finalized
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ohlc_aggregation() {
        let mut agg = OhlcAggregator::new(60_000); // 1-minute candles

        // All ticks in the [0, 60000) bucket
        agg.ingest(&Tick { symbol: "INFY".into(), price: dec!(1500), timestamp_ms: 1_000 });
        agg.ingest(&Tick { symbol: "INFY".into(), price: dec!(1510), timestamp_ms: 30_000 });
        agg.ingest(&Tick { symbol: "INFY".into(), price: dec!(1495), timestamp_ms: 45_000 });
        agg.ingest(&Tick { symbol: "INFY".into(), price: dec!(1505), timestamp_ms: 59_000 });

        let candles = agg.close_expired(120_000);
        assert_eq!(candles.len(), 1);
        let c = &candles[0];
        assert_eq!(c.open, dec!(1500));
        assert_eq!(c.high, dec!(1510));
        assert_eq!(c.low,  dec!(1495));
        assert_eq!(c.close, dec!(1505));
        assert_eq!(c.tick_count, 4);
    }
}
```

**Talking points**:
- **Late-arriving ticks**: if a tick arrives for a bucket that's already closed, you have to decide — drop it, log it, or reopen the bucket. Mention this is a policy decision (financial regulators often require dropping).
- For high-frequency streams (1M ticks/sec): partition by symbol-hash across workers; each worker owns a subset of symbols.
- Memory grows with `(symbols × open_buckets)`. Bound it by closing buckets aggressively.

---

## 14. Position Tracker

**Problem**: Track running positions and average cost basis per symbol from a trade stream.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TradeSide {
    Buy,
    Sell,
}

#[derive(Debug, Clone)]
pub struct ExecutedTrade {
    pub symbol: String,
    pub side: TradeSide,
    pub quantity: Decimal,
    pub price: Decimal,
}

#[derive(Debug, Clone, Default)]
pub struct Position {
    pub quantity: Decimal, // negative = short
    pub avg_cost: Decimal,
    pub realized_pnl: Decimal,
}

pub struct PositionTracker {
    positions: HashMap<String, Position>,
}

impl PositionTracker {
    pub fn new() -> Self { Self { positions: HashMap::new() } }

    pub fn apply(&mut self, trade: &ExecutedTrade) {
        let pos = self.positions.entry(trade.symbol.clone()).or_default();
        let signed_qty = match trade.side {
            TradeSide::Buy => trade.quantity,
            TradeSide::Sell => -trade.quantity,
        };

        // Same direction (both long or both short) → blend cost basis
        // Opposite direction → realize P&L on the matched portion
        let new_qty = pos.quantity + signed_qty;

        if pos.quantity.is_zero() || (pos.quantity.signum() == signed_qty.signum()) {
            // Same direction or opening from flat — blend cost
            let total_cost = pos.quantity * pos.avg_cost + signed_qty * trade.price;
            pos.avg_cost = if new_qty.is_zero() { dec!(0) } else { total_cost / new_qty };
        } else {
            // Opposite direction — realize P&L on the matched portion
            let matched = pos.quantity.abs().min(signed_qty.abs());
            // P&L = (exit_price - avg_cost) × matched_qty × direction_sign
            let pnl_per_unit = if pos.quantity > dec!(0) {
                trade.price - pos.avg_cost // closing long
            } else {
                pos.avg_cost - trade.price // closing short
            };
            pos.realized_pnl += pnl_per_unit * matched;

            // If we flipped direction, the residual takes on the new trade's price
            if (pos.quantity + signed_qty).signum() != pos.quantity.signum() && !new_qty.is_zero() {
                pos.avg_cost = trade.price;
            }
            // If we just reduced (didn't flip), avg_cost stays the same.
        }

        pos.quantity = new_qty;
    }

    pub fn position(&self, symbol: &str) -> Option<&Position> {
        self.positions.get(symbol)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn buy_then_buy_blends_cost() {
        let mut t = PositionTracker::new();
        t.apply(&ExecutedTrade { symbol: "INFY".into(), side: TradeSide::Buy, quantity: dec!(10), price: dec!(1500) });
        t.apply(&ExecutedTrade { symbol: "INFY".into(), side: TradeSide::Buy, quantity: dec!(10), price: dec!(1600) });
        let p = t.position("INFY").unwrap();
        assert_eq!(p.quantity, dec!(20));
        assert_eq!(p.avg_cost, dec!(1550));
        assert_eq!(p.realized_pnl, dec!(0));
    }

    #[test]
    fn buy_then_partial_sell_realizes_pnl() {
        let mut t = PositionTracker::new();
        t.apply(&ExecutedTrade { symbol: "INFY".into(), side: TradeSide::Buy, quantity: dec!(10), price: dec!(1500) });
        t.apply(&ExecutedTrade { symbol: "INFY".into(), side: TradeSide::Sell, quantity: dec!(4),  price: dec!(1600) });
        let p = t.position("INFY").unwrap();
        assert_eq!(p.quantity, dec!(6));
        assert_eq!(p.avg_cost, dec!(1500));
        assert_eq!(p.realized_pnl, dec!(400)); // (1600 - 1500) * 4
    }
}
```

**Talking points**:
- The trickiest case is **direction flip** (e.g., long 10 → sell 15 → now short 5). Walk through this in the interview.
- Mention FIFO vs LIFO vs average-cost accounting — average-cost is what most regulators / GAAP allow but different jurisdictions differ. (FinFlo would care about this!)
- Wash-sale rules, corporate actions (splits, dividends) — out of scope but worth mentioning as awareness.

---

## 15. Interest Rate Calculator

**Problem**: Compute compound interest with day-count conventions.

```rust
use rust_decimal::Decimal;
use rust_decimal::prelude::*;
use rust_decimal_macros::dec;
use chrono::NaiveDate;

#[derive(Debug, Clone, Copy)]
pub enum DayCount {
    Act360,
    Act365,
    Thirty360,
}

pub fn day_count_fraction(start: NaiveDate, end: NaiveDate, convention: DayCount) -> Decimal {
    match convention {
        DayCount::Act360 => Decimal::from(end.signed_duration_since(start).num_days()) / dec!(360),
        DayCount::Act365 => Decimal::from(end.signed_duration_since(start).num_days()) / dec!(365),
        DayCount::Thirty360 => {
            // US 30/360: simplified version
            let d1 = start.day().min(30) as i64;
            let d2 = if start.day() >= 30 { end.day().min(30) } else { end.day() } as i64;
            let m1 = start.month() as i64;
            let m2 = end.month() as i64;
            let y1 = start.year() as i64;
            let y2 = end.year() as i64;
            let days = 360 * (y2 - y1) + 30 * (m2 - m1) + (d2 - d1);
            Decimal::from(days) / dec!(360)
        }
    }
}

pub fn simple_interest(principal: Decimal, annual_rate: Decimal, day_count: Decimal) -> Decimal {
    principal * annual_rate * day_count
}

pub fn compound_interest(
    principal: Decimal,
    annual_rate: Decimal,
    day_count: Decimal,
    compounding_per_year: u32,
) -> Decimal {
    // A = P * (1 + r/n)^(n*t)
    let n = Decimal::from(compounding_per_year);
    let exponent = day_count * n;
    let base = dec!(1) + annual_rate / n;
    // No native Decimal pow with Decimal exponent — convert exponent to f64 for the exponent only
    let exp_f64 = exponent.to_f64().unwrap_or(0.0);
    let base_f64 = base.to_f64().unwrap_or(0.0);
    let multiplier = Decimal::from_f64(base_f64.powf(exp_f64)).unwrap_or(dec!(1));
    principal * multiplier
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn dcf_act_365() {
        let start = NaiveDate::from_ymd_opt(2026, 1, 1).unwrap();
        let end = NaiveDate::from_ymd_opt(2027, 1, 1).unwrap();
        let dcf = day_count_fraction(start, end, DayCount::Act365);
        assert_eq!(dcf, dec!(1));
    }

    #[test]
    fn simple_interest_basic() {
        let interest = simple_interest(dec!(100_000), dec!(0.08), dec!(1));
        assert_eq!(interest, dec!(8000));
    }
}
```

**Talking points**:
- **Floating-point creep**: pure-Decimal compound interest is impossible without a fixed-point pow function. Acknowledge this honestly — most production systems use Decimal everywhere except the exponentiation step.
- Day-count conventions are domain-specific. Mention: "I'd implement these from the ISDA spec rather than guess — Act/360 vs Act/365 vs 30/360 each have different edge cases for month-end."
- For actuarial work, mention `mortality tables`, `discount factors`, `present value` — even saying these words signals you've thought about the domain.

---

# Tier A — Solutions

## 16. Longest Substring Without Repeating Characters

```rust
use std::collections::HashMap;

pub fn length_of_longest_substring(s: String) -> i32 {
    let mut last_seen: HashMap<char, usize> = HashMap::new();
    let mut start = 0usize;
    let mut best = 0usize;

    for (i, c) in s.chars().enumerate() {
        if let Some(&prev) = last_seen.get(&c) {
            if prev >= start {
                start = prev + 1;
            }
        }
        last_seen.insert(c, i);
        best = best.max(i - start + 1);
    }
    best as i32
}
```

**Talking points**: Sliding window with a map of last-seen index. O(n) time, O(min(n, alphabet)) space. The `prev >= start` check is critical — without it, characters outside the window incorrectly shrink it.

---

## 17. Group Anagrams

```rust
use std::collections::HashMap;

pub fn group_anagrams(strs: Vec<String>) -> Vec<Vec<String>> {
    let mut groups: HashMap<[u8; 26], Vec<String>> = HashMap::new();
    for s in strs {
        let mut key = [0u8; 26];
        for b in s.bytes() {
            key[(b - b'a') as usize] += 1;
        }
        groups.entry(key).or_default().push(s);
    }
    groups.into_values().collect()
}
```

**Talking points**: Counting array `[u8; 26]` as the key beats sorting (`O(n·k)` vs `O(n·k log k)`). Mention that this assumes lowercase ASCII — for Unicode you'd hash the sorted char vector.

---

## 18. Product of Array Except Self

```rust
pub fn product_except_self(nums: Vec<i32>) -> Vec<i32> {
    let n = nums.len();
    let mut result = vec![1i32; n];

    // First pass: result[i] = product of everything to the left of i
    let mut left = 1i32;
    for i in 0..n {
        result[i] = left;
        left *= nums[i];
    }

    // Second pass: multiply by product of everything to the right of i
    let mut right = 1i32;
    for i in (0..n).rev() {
        result[i] *= right;
        right *= nums[i];
    }
    result
}
```

**Talking points**: O(n) time, O(1) extra space (output array doesn't count). No division required — important if the problem disallows it or zeros are present.

---

## 19. Maximum Subarray (Kadane's)

```rust
pub fn max_sub_array(nums: Vec<i32>) -> i32 {
    let mut current = nums[0];
    let mut best = nums[0];
    for &n in nums.iter().skip(1) {
        current = n.max(current + n);
        best = best.max(current);
    }
    best
}
```

**Talking points**: O(n) time, O(1) space. The invariant: `current` = max-sum subarray ending at index `i`. Reset to `n` whenever the running sum hurts. Classic DP collapsed to two variables.

---

## 20. Best Time to Buy and Sell Stock

**Single transaction (LC 121)**:

```rust
pub fn max_profit(prices: Vec<i32>) -> i32 {
    let mut min_price = i32::MAX;
    let mut best = 0;
    for p in prices {
        min_price = min_price.min(p);
        best = best.max(p - min_price);
    }
    best
}
```

**Multiple transactions (LC 122)** — buy/sell unlimited times:

```rust
pub fn max_profit_unlimited(prices: Vec<i32>) -> i32 {
    prices.windows(2).map(|w| (w[1] - w[0]).max(0)).sum()
}
```

**At most 2 transactions (LC 123)** — DP:

```rust
pub fn max_profit_two(prices: Vec<i32>) -> i32 {
    let (mut buy1, mut sell1, mut buy2, mut sell2) = (i32::MIN, 0, i32::MIN, 0);
    for p in prices {
        buy1 = buy1.max(-p);
        sell1 = sell1.max(buy1 + p);
        buy2 = buy2.max(sell1 - p);
        sell2 = sell2.max(buy2 + p);
    }
    sell2
}
```

**Talking points**: All O(n). The state-machine framing (buy1 → sell1 → buy2 → sell2) generalizes to k transactions. **Domain-relevant** — Actlogica builds portfolio software where buy/sell timing matters.

---

## 21. Implement a Trie

```rust
use std::collections::HashMap;

#[derive(Default)]
pub struct Trie {
    children: HashMap<char, Trie>,
    is_end: bool,
}

impl Trie {
    pub fn new() -> Self { Self::default() }

    pub fn insert(&mut self, word: String) {
        let mut node = self;
        for c in word.chars() {
            node = node.children.entry(c).or_default();
        }
        node.is_end = true;
    }

    pub fn search(&self, word: String) -> bool {
        self.find(&word).map_or(false, |n| n.is_end)
    }

    pub fn starts_with(&self, prefix: String) -> bool {
        self.find(&prefix).is_some()
    }

    fn find(&self, s: &str) -> Option<&Trie> {
        let mut node = self;
        for c in s.chars() {
            node = node.children.get(&c)?;
        }
        Some(node)
    }
}
```

**Talking points**: O(k) insert/search/prefix where k = word length. For ASCII-only, `[Option<Box<Trie>>; 26]` is more cache-friendly but uglier. The `find` helper deduplicates `search` and `starts_with`.

---

## 22. Number of Islands

```rust
pub fn num_islands(mut grid: Vec<Vec<char>>) -> i32 {
    let (rows, cols) = (grid.len(), grid[0].len());
    let mut count = 0;
    for r in 0..rows {
        for c in 0..cols {
            if grid[r][c] == '1' {
                count += 1;
                dfs(&mut grid, r as i32, c as i32, rows as i32, cols as i32);
            }
        }
    }
    count
}

fn dfs(grid: &mut Vec<Vec<char>>, r: i32, c: i32, rows: i32, cols: i32) {
    if r < 0 || c < 0 || r >= rows || c >= cols || grid[r as usize][c as usize] != '1' {
        return;
    }
    grid[r as usize][c as usize] = '0';
    dfs(grid, r + 1, c, rows, cols);
    dfs(grid, r - 1, c, rows, cols);
    dfs(grid, r, c + 1, rows, cols);
    dfs(grid, r, c - 1, rows, cols);
}
```

**Talking points**: O(R·C) time and space (recursion stack worst case). For very large grids, switch to iterative BFS with a `VecDeque` to avoid stack overflow. Union-find is an alternative — useful if islands need to be merged dynamically.

---

## 23. Course Schedule (Topological Sort)

```rust
use std::collections::VecDeque;

pub fn can_finish(num_courses: i32, prerequisites: Vec<Vec<i32>>) -> bool {
    let n = num_courses as usize;
    let mut graph: Vec<Vec<usize>> = vec![Vec::new(); n];
    let mut in_degree = vec![0i32; n];

    for p in prerequisites {
        let (course, prereq) = (p[0] as usize, p[1] as usize);
        graph[prereq].push(course);
        in_degree[course] += 1;
    }

    let mut queue: VecDeque<usize> = (0..n).filter(|&i| in_degree[i] == 0).collect();
    let mut processed = 0;

    while let Some(u) = queue.pop_front() {
        processed += 1;
        for &v in &graph[u] {
            in_degree[v] -= 1;
            if in_degree[v] == 0 {
                queue.push_back(v);
            }
        }
    }
    processed == n
}
```

**Talking points**: Kahn's BFS topological sort. O(V + E). If the queue runs dry before processing all nodes, there's a cycle. DFS with color-marking (white/gray/black) is the alternative.

---

## 24. Kth Largest Element

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

pub fn find_kth_largest(nums: Vec<i32>, k: i32) -> i32 {
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::with_capacity(k as usize + 1);
    for n in nums {
        heap.push(Reverse(n));
        if heap.len() > k as usize {
            heap.pop();
        }
    }
    heap.peek().unwrap().0
}
```

**Talking points**: Min-heap of size k → O(n log k). Quickselect is O(n) average but worst-case O(n²) — heap is the safer interview answer. For massive datasets, mention reservoir sampling for streaming top-K.

---

## 25. Merge K Sorted Lists

For interview Rust, linked lists are awkward. Here's the **`Vec<Vec<i32>>` version** which interviewers usually accept:

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

pub fn merge_k_sorted(lists: Vec<Vec<i32>>) -> Vec<i32> {
    let mut heap: BinaryHeap<Reverse<(i32, usize, usize)>> = BinaryHeap::new();
    // Seed with first element of each list
    for (i, list) in lists.iter().enumerate() {
        if let Some(&v) = list.first() {
            heap.push(Reverse((v, i, 0)));
        }
    }

    let mut result = Vec::new();
    while let Some(Reverse((val, list_id, idx))) = heap.pop() {
        result.push(val);
        if idx + 1 < lists[list_id].len() {
            heap.push(Reverse((lists[list_id][idx + 1], list_id, idx + 1)));
        }
    }
    result
}
```

**Talking points**: O(N log k) where N is total elements, k is number of lists. The triple `(value, list_id, index)` lets the heap break ties deterministically and advance the right list on pop.

---

## 26. Subsets / Permutations / Combinations

**Subsets** (LC 78):

```rust
pub fn subsets(nums: Vec<i32>) -> Vec<Vec<i32>> {
    let mut result = Vec::new();
    let mut current = Vec::new();
    backtrack_subsets(&nums, 0, &mut current, &mut result);
    result
}

fn backtrack_subsets(nums: &[i32], start: usize, current: &mut Vec<i32>, result: &mut Vec<Vec<i32>>) {
    result.push(current.clone());
    for i in start..nums.len() {
        current.push(nums[i]);
        backtrack_subsets(nums, i + 1, current, result);
        current.pop();
    }
}
```

**Permutations** (LC 46):

```rust
pub fn permute(nums: Vec<i32>) -> Vec<Vec<i32>> {
    let mut result = Vec::new();
    let mut current = Vec::new();
    let mut used = vec![false; nums.len()];
    backtrack_perm(&nums, &mut used, &mut current, &mut result);
    result
}

fn backtrack_perm(nums: &[i32], used: &mut Vec<bool>, current: &mut Vec<i32>, result: &mut Vec<Vec<i32>>) {
    if current.len() == nums.len() {
        result.push(current.clone());
        return;
    }
    for i in 0..nums.len() {
        if used[i] { continue; }
        used[i] = true;
        current.push(nums[i]);
        backtrack_perm(nums, used, current, result);
        current.pop();
        used[i] = false;
    }
}
```

**Combinations** (LC 77) — pick k from 1..=n:

```rust
pub fn combine(n: i32, k: i32) -> Vec<Vec<i32>> {
    let mut result = Vec::new();
    let mut current = Vec::new();
    backtrack_comb(1, n, k, &mut current, &mut result);
    result
}

fn backtrack_comb(start: i32, n: i32, k: i32, current: &mut Vec<i32>, result: &mut Vec<Vec<i32>>) {
    if current.len() as i32 == k {
        result.push(current.clone());
        return;
    }
    for i in start..=n {
        current.push(i);
        backtrack_comb(i + 1, n, k, current, result);
        current.pop();
    }
}
```

**Talking points**: All backtracking templates share the shape: choose → recurse → unchoose. Subsets = O(2^n · n), Permutations = O(n! · n), Combinations = O(C(n,k) · k). Be ready to draw the recursion tree.

---

## 27. Coin Change (DP)

```rust
pub fn coin_change(coins: Vec<i32>, amount: i32) -> i32 {
    let amount = amount as usize;
    let mut dp = vec![i32::MAX; amount + 1];
    dp[0] = 0;
    for i in 1..=amount {
        for &c in &coins {
            if c as usize <= i && dp[i - c as usize] != i32::MAX {
                dp[i] = dp[i].min(dp[i - c as usize] + 1);
            }
        }
    }
    if dp[amount] == i32::MAX { -1 } else { dp[amount] }
}
```

**Talking points**: O(amount · coins) time, O(amount) space. Bottom-up DP — `dp[i]` = min coins to make amount `i`. The greedy approach fails for non-canonical coin systems (e.g., coins = [1, 3, 4], amount = 6 → greedy gives 3, optimal is 2).

---

# Tier B — Solutions

## 28. Word Break

```rust
use std::collections::HashSet;

pub fn word_break(s: String, word_dict: Vec<String>) -> bool {
    let dict: HashSet<&str> = word_dict.iter().map(|s| s.as_str()).collect();
    let n = s.len();
    let mut dp = vec![false; n + 1];
    dp[0] = true;
    for i in 1..=n {
        for j in 0..i {
            if dp[j] && dict.contains(&s[j..i]) {
                dp[i] = true;
                break;
            }
        }
    }
    dp[n]
}
```

**Talking points**: O(n² · m) where m is avg word length (string slicing + hash lookup). `dp[i]` = "can s[..i] be segmented." Storing the dictionary as `HashSet<&str>` avoids per-lookup allocation.

---

## 29. Longest Palindromic Substring

```rust
pub fn longest_palindrome(s: String) -> String {
    let chars: Vec<char> = s.chars().collect();
    if chars.is_empty() { return String::new(); }
    let (mut start, mut max_len) = (0, 1);

    for i in 0..chars.len() {
        let (s1, l1) = expand(&chars, i, i);       // odd-length center
        let (s2, l2) = expand(&chars, i, i + 1);   // even-length center
        if l1 > max_len { start = s1; max_len = l1; }
        if l2 > max_len { start = s2; max_len = l2; }
    }
    chars[start..start + max_len].iter().collect()
}

fn expand(chars: &[char], mut l: usize, mut r: usize) -> (usize, usize) {
    while r < chars.len() && l <= r && chars[l] == chars[r] {
        if l == 0 { return (0, r + 1); }
        l -= 1;
    }
    (l + 1, r - l - 1)
}
```

**Talking points**: O(n²) time, O(1) space (expand-around-center). Manacher's algorithm gives O(n) but is rarely required. The trick is handling both odd-length and even-length palindromes by trying two centers.

---

## 30. Rotate Image

```rust
pub fn rotate(matrix: &mut Vec<Vec<i32>>) {
    let n = matrix.len();
    // Transpose
    for i in 0..n {
        for j in i + 1..n {
            let tmp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = tmp;
        }
    }
    // Reverse each row
    for row in matrix.iter_mut() {
        row.reverse();
    }
}
```

**Talking points**: O(n²) time, O(1) extra space (in-place). Two-step transform: transpose, then reverse rows = 90° clockwise rotation. For counter-clockwise, reverse rows first, then transpose.

---

## 31. Spiral Matrix

```rust
pub fn spiral_order(matrix: Vec<Vec<i32>>) -> Vec<i32> {
    let (rows, cols) = (matrix.len(), matrix[0].len());
    let mut result = Vec::with_capacity(rows * cols);
    let (mut top, mut bottom, mut left, mut right) = (0usize, rows - 1, 0usize, cols - 1);

    while result.len() < rows * cols {
        for c in left..=right { result.push(matrix[top][c]); }
        if top == bottom { break; }
        top += 1;

        for r in top..=bottom { result.push(matrix[r][right]); }
        if left == right { break; }
        if right == 0 { break; }
        right -= 1;

        for c in (left..=right).rev() { result.push(matrix[bottom][c]); }
        if top > bottom { break; }
        bottom -= 1;

        for r in (top..=bottom).rev() { result.push(matrix[r][left]); }
        left += 1;
    }
    result
}
```

**Talking points**: O(rows · cols) time. Four boundary pointers (`top, bottom, left, right`) shrink after each traversal. Be careful with the underflow on `usize` boundaries — Rust will panic where C wraps silently.

---

## 32. Set Matrix Zeroes

```rust
pub fn set_zeroes(matrix: &mut Vec<Vec<i32>>) {
    let (rows, cols) = (matrix.len(), matrix[0].len());
    let mut first_row_zero = false;
    let mut first_col_zero = false;

    for c in 0..cols { if matrix[0][c] == 0 { first_row_zero = true; break; } }
    for r in 0..rows { if matrix[r][0] == 0 { first_col_zero = true; break; } }

    // Use first row/col as markers for the rest
    for r in 1..rows {
        for c in 1..cols {
            if matrix[r][c] == 0 {
                matrix[r][0] = 0;
                matrix[0][c] = 0;
            }
        }
    }

    for r in 1..rows {
        for c in 1..cols {
            if matrix[r][0] == 0 || matrix[0][c] == 0 {
                matrix[r][c] = 0;
            }
        }
    }

    if first_row_zero { for c in 0..cols { matrix[0][c] = 0; } }
    if first_col_zero { for r in 0..rows { matrix[r][0] = 0; } }
}
```

**Talking points**: O(rows · cols) time, O(1) extra space. The first row and column double as zero-flags — but you have to remember if they themselves had zeros, hence the two boolean flags.

---

## 33. Search in Rotated Sorted Array

```rust
pub fn search(nums: Vec<i32>, target: i32) -> i32 {
    let (mut lo, mut hi) = (0i32, nums.len() as i32 - 1);
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        let m = mid as usize;
        if nums[m] == target { return mid; }

        // Determine which half is sorted
        if nums[lo as usize] <= nums[m] {
            // Left half is sorted
            if nums[lo as usize] <= target && target < nums[m] {
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        } else {
            // Right half is sorted
            if nums[m] < target && target <= nums[hi as usize] {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
    }
    -1
}
```

**Talking points**: O(log n). At every midpoint, at least one half is properly sorted — figure out which, check if target falls in it, recurse. Edge case: duplicates degrade this to O(n) — handle separately if asked.

---

## 34. Linked List Cycle

Linked lists in Rust are awkward — interviewers usually accept the high-level description. Here's a working version using the LeetCode-style `ListNode`:

```rust
// LeetCode's ListNode definition:
// pub struct ListNode { pub val: i32, pub next: Option<Box<ListNode>> }

// In real Rust this is hard because Box is owning — a cycle would mean two owners.
// LeetCode's harness uses unsafe raw pointers internally to construct the cycle.
// For interview: describe Floyd's algorithm verbally and write pseudocode.

// Pseudocode you should be able to write/explain:
//   let mut slow = head;
//   let mut fast = head;
//   loop {
//       fast = fast?.next?.next;
//       slow = slow?.next;
//       if fast as *const _ == slow as *const _ { return true; }
//       if fast is None { return false; }
//   }
```

**Talking points**: **Tell the interviewer up front** — "Linked lists in safe Rust require `Rc<RefCell<T>>` or arena allocation; a cycle of `Box`-owned nodes is impossible in safe code. I'll describe Floyd's tortoise-and-hare verbally and write the algorithm in pseudocode, or implement it with an arena/index-based list if you prefer." That's a senior-level answer.

---

## 35. Reverse Linked List

Same caveat as #34. The clean Rust version using owned `Option<Box<ListNode>>`:

```rust
// pub struct ListNode { pub val: i32, pub next: Option<Box<ListNode>> }

pub fn reverse_list(head: Option<Box<ListNode>>) -> Option<Box<ListNode>> {
    let mut prev = None;
    let mut current = head;
    while let Some(mut node) = current {
        let next = node.next.take(); // detach
        node.next = prev;             // reverse pointer
        prev = Some(node);
        current = next;
    }
    prev
}
```

**Talking points**: O(n) time, O(1) extra space. The `node.next.take()` is the Rust-specific trick — it moves the `next` out and replaces it with `None`, avoiding ownership conflicts. Recursive version is cleaner but uses O(n) stack.

---

# Concurrency & Async — Solutions

## 36. Thread-Safe Counter

**Version A — `Arc<Mutex<u64>>`** (general, supports complex updates):

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn counter_mutex() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = Vec::new();
    for _ in 0..8 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1_000_000 {
                let mut guard = c.lock().unwrap();
                *guard += 1;
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("final: {}", *counter.lock().unwrap());
}
```

**Version B — `Arc<AtomicU64>`** (faster, primitive ops only):

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

fn counter_atomic() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();
    for _ in 0..8 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1_000_000 {
                c.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("final: {}", counter.load(Ordering::Relaxed));
}
```

**Talking points**:
- `AtomicU64` is ~5–20× faster for pure counter use. `Mutex` is needed when you must update *related* state atomically.
- `Ordering::Relaxed` is correct for a counter — no other memory needs to be synchronized. For a flag controlling access to other data, use `Acquire`/`Release`.
- Memory orderings: `Relaxed` (no sync), `Acquire`/`Release` (publish/observe), `SeqCst` (total order, slowest). **Know these names.**

---

## 37. Rate Limiter (Token Bucket)

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use std::time::Instant;

pub struct TokenBucket {
    capacity: f64,
    tokens: f64,
    refill_per_sec: f64,
    last_refill: Instant,
}

impl TokenBucket {
    pub fn new(capacity: f64, refill_per_sec: f64) -> Self {
        Self { capacity, tokens: capacity, refill_per_sec, last_refill: Instant::now() }
    }

    pub fn try_acquire(&mut self, cost: f64) -> bool {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_per_sec).min(self.capacity);
        self.last_refill = now;

        if self.tokens >= cost {
            self.tokens -= cost;
            true
        } else {
            false
        }
    }
}

pub struct RateLimiter {
    buckets: Mutex<HashMap<String, TokenBucket>>,
    capacity: f64,
    refill_per_sec: f64,
}

impl RateLimiter {
    pub fn new(capacity: f64, refill_per_sec: f64) -> Self {
        Self { buckets: Mutex::new(HashMap::new()), capacity, refill_per_sec }
    }

    pub fn try_acquire(&self, key: &str) -> bool {
        let mut buckets = self.buckets.lock().unwrap();
        let bucket = buckets.entry(key.to_string())
            .or_insert_with(|| TokenBucket::new(self.capacity, self.refill_per_sec));
        bucket.try_acquire(1.0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread::sleep;
    use std::time::Duration;

    #[test]
    fn rate_limiter_basic() {
        let rl = RateLimiter::new(5.0, 10.0); // 5 burst, 10/sec refill
        for _ in 0..5 { assert!(rl.try_acquire("user-1")); }
        assert!(!rl.try_acquire("user-1"));   // bucket empty
        sleep(Duration::from_millis(200));    // refill 2 tokens
        assert!(rl.try_acquire("user-1"));
        assert!(rl.try_acquire("user-1"));
    }
}
```

**Talking points**:
- Lazy refill on each call avoids a background timer thread.
- For high-contention scenarios use `DashMap` instead of `Mutex<HashMap>` (shard-locked map).
- For distributed limiting: Lua script in Redis (atomic check-and-decrement) — mention this.
- For async, replace `std::sync::Mutex` with `tokio::sync::Mutex` (though for short critical sections, std Mutex is often fine inside async code).

---

## 38. Concurrent Bounded Queue

Idiomatic Rust uses `tokio::sync::mpsc` directly:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<i32>(4);

    let producer = tokio::spawn(async move {
        for i in 0..10 {
            // Suspends when full — automatic backpressure
            tx.send(i).await.unwrap();
            println!("sent {}", i);
        }
        // tx dropped here → channel closes
    });

    let consumer = tokio::spawn(async move {
        while let Some(v) = rx.recv().await {
            println!("recv {}", v);
            tokio::time::sleep(std::time::Duration::from_millis(50)).await;
        }
    });

    producer.await.unwrap();
    consumer.await.unwrap();
}
```

**If they ask "implement it yourself" (rare, senior screen)**, use a `Mutex<VecDeque>` + two `Notify`:

```rust
use std::collections::VecDeque;
use tokio::sync::{Mutex, Notify};
use std::sync::Arc;

pub struct BoundedQueue<T> {
    capacity: usize,
    inner: Mutex<VecDeque<T>>,
    not_full: Notify,
    not_empty: Notify,
}

impl<T> BoundedQueue<T> {
    pub fn new(capacity: usize) -> Arc<Self> {
        Arc::new(Self {
            capacity,
            inner: Mutex::new(VecDeque::with_capacity(capacity)),
            not_full: Notify::new(),
            not_empty: Notify::new(),
        })
    }

    pub async fn push(&self, item: T) {
        loop {
            let mut q = self.inner.lock().await;
            if q.len() < self.capacity {
                q.push_back(item);
                self.not_empty.notify_one();
                return;
            }
            drop(q);
            self.not_full.notified().await;
        }
    }

    pub async fn pop(&self) -> T {
        loop {
            let mut q = self.inner.lock().await;
            if let Some(item) = q.pop_front() {
                self.not_full.notify_one();
                return item;
            }
            drop(q);
            self.not_empty.notified().await;
        }
    }
}
```

**Talking points**: This is the classic "condvar pattern" translated to async. The lost-wakeup bug is easy to introduce — explain the loop-then-check pattern. In production, just use `mpsc`.

---

## 39. Fan-out / Fan-in

```rust
use tokio::sync::Semaphore;
use tokio::task::JoinSet;
use std::sync::Arc;

pub async fn fetch_all(urls: Vec<String>, max_concurrent: usize) -> Vec<Result<String, String>> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut set = JoinSet::new();

    for url in urls {
        let sem = Arc::clone(&semaphore);
        set.spawn(async move {
            let _permit = sem.acquire().await.unwrap();
            // Simulated fetch
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            Ok::<_, String>(format!("body of {}", url))
        });
    }

    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        match res {
            Ok(Ok(body)) => results.push(Ok(body)),
            Ok(Err(e)) => results.push(Err(e)),
            Err(join_err) => results.push(Err(format!("join error: {}", join_err))),
        }
    }
    results
}
```

**Talking points**:
- `Semaphore` caps concurrency — common pattern when downstream service has rate limits.
- `JoinSet` over `Vec<JoinHandle>` because it cleans up on drop.
- For *ordered* results, use `futures::stream::iter(...).buffer_unordered(n).collect()` from the `futures` crate.

---

## 40. Graceful Shutdown

```rust
use tokio::signal;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let work = async {
        loop {
            println!("working...");
            sleep(Duration::from_secs(1)).await;
        }
    };

    tokio::select! {
        _ = work => {},
        _ = signal::ctrl_c() => {
            println!("\nshutdown signal received, cleaning up...");
            // Drain in-flight requests, flush buffers, close connections
            sleep(Duration::from_millis(500)).await;
            println!("clean shutdown complete");
        }
    }
}
```

**Production pattern with `CancellationToken`** (for many tasks):

```rust
use tokio_util::sync::CancellationToken;

async fn worker(id: u64, token: CancellationToken) {
    loop {
        tokio::select! {
            _ = token.cancelled() => {
                println!("worker {} shutting down", id);
                return;
            }
            _ = tokio::time::sleep(Duration::from_secs(1)) => {
                println!("worker {} tick", id);
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let mut handles = Vec::new();
    for id in 0..3 {
        handles.push(tokio::spawn(worker(id, token.clone())));
    }

    tokio::signal::ctrl_c().await.unwrap();
    token.cancel(); // Signal all workers
    for h in handles { h.await.unwrap(); }
}
```

**Talking points**: `CancellationToken` is the canonical Tokio pattern for multi-task shutdown. Without it, killing a task mid-await loses in-flight work. Add a timeout (e.g., 30s) on the shutdown so you don't hang forever.

---

## 41. Implement a Future from Scratch

A future that completes after N polls:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

pub struct YieldNTimes {
    remaining: u32,
}

impl YieldNTimes {
    pub fn new(n: u32) -> Self { Self { remaining: n } }
}

impl Future for YieldNTimes {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.remaining == 0 {
            Poll::Ready(())
        } else {
            self.remaining -= 1;
            // Tell the executor to poll us again immediately
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    YieldNTimes::new(5).await;
    println!("done after 5 polls");
}
```

**A timer-based future** (more useful, builds on `tokio::time::Sleep`):

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

pub struct Delay {
    deadline: Instant,
    sleeper: Pin<Box<tokio::time::Sleep>>,
}

impl Delay {
    pub fn new(duration: Duration) -> Self {
        let deadline = Instant::now() + duration;
        Self {
            deadline,
            sleeper: Box::pin(tokio::time::sleep(duration)),
        }
    }
}

impl Future for Delay {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.deadline {
            return Poll::Ready(());
        }
        self.sleeper.as_mut().poll(cx)
    }
}
```

**Talking points**:
- `Pin<&mut Self>` — `Future` must be pinned because async state machines may have self-references.
- `cx.waker().wake_by_ref()` — tells the executor "schedule me again." Forgetting this causes the future to hang forever.
- `Poll::Pending` means "I'm not done; I've registered the waker for when I am."
- This is senior-level material. Even partial code shows depth.

---

## 42. Deadlock Detection / Avoidance

**The classic two-mutex deadlock**:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn deadlock_demo() {
    let a = Arc::new(Mutex::new(0));
    let b = Arc::new(Mutex::new(0));

    let (a1, b1) = (Arc::clone(&a), Arc::clone(&b));
    let t1 = thread::spawn(move || {
        let _ga = a1.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _gb = b1.lock().unwrap();    // deadlocks
    });

    let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
    let t2 = thread::spawn(move || {
        let _gb = b2.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _ga = a2.lock().unwrap();    // deadlocks
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

**Fixes**:

1. **Lock ordering** — always acquire in the same global order (e.g., by memory address or by ID):

```rust
fn lock_in_order<'a>(x: &'a Mutex<i32>, y: &'a Mutex<i32>)
    -> (std::sync::MutexGuard<'a, i32>, std::sync::MutexGuard<'a, i32>)
{
    let (first, second) = if (x as *const _) < (y as *const _) { (x, y) } else { (y, x) };
    let g1 = first.lock().unwrap();
    let g2 = second.lock().unwrap();
    (g1, g2)
}
```

2. **`try_lock` with backoff** — if you can't acquire both, release and retry:

```rust
use std::sync::TryLockError;

fn acquire_both<'a>(a: &'a Mutex<i32>, b: &'a Mutex<i32>) {
    loop {
        let ga = a.lock().unwrap();
        match b.try_lock() {
            Ok(gb) => return drop((ga, gb)),
            Err(TryLockError::WouldBlock) => {
                drop(ga);
                std::thread::yield_now();
            }
            Err(_) => panic!("poisoned"),
        }
    }
}
```

3. **`parking_lot::Mutex` with timeout** — bounded waits instead of indefinite block.

**Talking points**: Deadlock = (1) mutual exclusion + (2) hold-and-wait + (3) no preemption + (4) circular wait. Break any one. Lock ordering breaks (4); `try_lock` breaks (2). For complex graphs, mention `parking_lot::deadlock` detection or RAII guard ordering audits.

---

## 43. Channel-Based State Machine (Actor)

```rust
use tokio::sync::{mpsc, oneshot};

#[derive(Debug)]
pub enum AccountCommand {
    Deposit { amount: u64, reply: oneshot::Sender<u64> },
    Withdraw { amount: u64, reply: oneshot::Sender<Result<u64, String>> },
    GetBalance { reply: oneshot::Sender<u64> },
}

pub struct AccountHandle {
    tx: mpsc::Sender<AccountCommand>,
}

impl AccountHandle {
    pub fn new(initial: u64) -> Self {
        let (tx, mut rx) = mpsc::channel::<AccountCommand>(32);
        tokio::spawn(async move {
            let mut balance = initial;
            while let Some(cmd) = rx.recv().await {
                match cmd {
                    AccountCommand::Deposit { amount, reply } => {
                        balance += amount;
                        let _ = reply.send(balance);
                    }
                    AccountCommand::Withdraw { amount, reply } => {
                        if balance >= amount {
                            balance -= amount;
                            let _ = reply.send(Ok(balance));
                        } else {
                            let _ = reply.send(Err(format!("insufficient: have {}, need {}", balance, amount)));
                        }
                    }
                    AccountCommand::GetBalance { reply } => {
                        let _ = reply.send(balance);
                    }
                }
            }
        });
        Self { tx }
    }

    pub async fn deposit(&self, amount: u64) -> u64 {
        let (reply_tx, reply_rx) = oneshot::channel();
        self.tx.send(AccountCommand::Deposit { amount, reply: reply_tx }).await.unwrap();
        reply_rx.await.unwrap()
    }

    pub async fn withdraw(&self, amount: u64) -> Result<u64, String> {
        let (reply_tx, reply_rx) = oneshot::channel();
        self.tx.send(AccountCommand::Withdraw { amount, reply: reply_tx }).await.unwrap();
        reply_rx.await.unwrap()
    }

    pub async fn balance(&self) -> u64 {
        let (reply_tx, reply_rx) = oneshot::channel();
        self.tx.send(AccountCommand::GetBalance { reply: reply_tx }).await.unwrap();
        reply_rx.await.unwrap()
    }
}

#[tokio::main]
async fn main() {
    let account = AccountHandle::new(100);
    println!("balance: {}", account.balance().await);
    account.deposit(50).await;
    let result = account.withdraw(200).await;
    println!("withdraw 200: {:?}", result);
    println!("final: {}", account.balance().await);
}
```

**Talking points**:
- **No locks** — the actor task owns the state. Commands serialize through the channel.
- `mpsc` for commands, `oneshot` per-reply — the canonical pattern.
- This is the idiomatic Rust answer to "how do you share mutable state across tasks." **Strong signal.**
- For higher throughput: `tokio::sync::mpsc` is fast; for ultra-fast actor systems use `kanal` or `flume`.

---

# System Design — Solutions

These are discussion-shaped. Keep code minimal; structure your verbal answer.

## 44. URL Shortener

**Schema**:
```sql
CREATE TABLE urls (
    short_code VARCHAR(8) PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);
CREATE INDEX urls_long_url_idx ON urls(long_url); -- for dedup
```

**Code core**:
```rust
use base62;

pub fn encode_id(id: u64) -> String {
    base62::encode(id) // e.g., 125 -> "21"
}

pub fn shorten(long_url: &str, db: &mut HashMap<u64, String>) -> String {
    static NEXT_ID: std::sync::atomic::AtomicU64 = std::sync::atomic::AtomicU64::new(1);
    let id = NEXT_ID.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    db.insert(id, long_url.to_string());
    encode_id(id)
}
```

**Talking points**:
- ID generation: monotonic counter (single-DB), Snowflake (distributed), or hash-of-URL (idempotent).
- Storage: Postgres for source of truth; Redis cache for hot redirects.
- Hot keys: a viral link gets 10K req/sec — front with CDN-level cache.
- Custom slugs: extra `UNIQUE` constraint; collision handling.
- Analytics: write redirect events to Kafka, aggregate offline.

---

## 45. Distributed Counter

**Naive (broken under load)**: single `Postgres UPDATE counter SET v = v + 1`.

**Sharded**: N shards, each per-shard counter, periodic aggregation.

```rust
use std::sync::atomic::{AtomicU64, Ordering};

pub struct ShardedCounter {
    shards: Vec<AtomicU64>,
}

impl ShardedCounter {
    pub fn new(shard_count: usize) -> Self {
        Self { shards: (0..shard_count).map(|_| AtomicU64::new(0)).collect() }
    }

    pub fn increment(&self) {
        // Use a thread-local or hashed thread-id to pick a shard
        let shard = thread_id() % self.shards.len();
        self.shards[shard].fetch_add(1, Ordering::Relaxed);
    }

    pub fn total(&self) -> u64 {
        self.shards.iter().map(|s| s.load(Ordering::Relaxed)).sum()
    }
}

fn thread_id() -> usize {
    use std::thread;
    let id = thread::current().id();
    // hash or use as index — simplified here
    format!("{:?}", id).len()
}
```

**Talking points**:
- Increment: O(1), write-conflict-free.
- Read: O(N) sum across shards — eventually consistent.
- For multi-node: each node has local shards, async-aggregate to a central store every few seconds.
- For exact counts with high contention: HyperLogLog (approximate cardinality), Redis `INCR` with sharding.
- **CRDT angle**: G-Counter is the textbook CRDT for monotonic counters.

---

## 46. Idempotent Payment API

**The critical invariant**: at-most-once execution per `idempotency_key`.

```rust
use sqlx::PgPool;

#[derive(Debug)]
pub struct PaymentRequest {
    pub idempotency_key: String,
    pub from_account: i64,
    pub to_account: i64,
    pub amount_cents: i64,
}

pub async fn process_payment(
    pool: &PgPool,
    req: PaymentRequest,
) -> Result<i64, sqlx::Error> {
    let mut tx = pool.begin().await?;

    // 1. Check if already processed
    let existing: Option<(i64,)> = sqlx::query_as(
        "SELECT payment_id FROM idempotency_keys WHERE key = $1"
    ).bind(&req.idempotency_key).fetch_optional(&mut *tx).await?;

    if let Some((id,)) = existing {
        return Ok(id);
    }

    // 2. Insert payment + idempotency record in same transaction
    let payment_id: (i64,) = sqlx::query_as(
        "INSERT INTO payments (from_account, to_account, amount_cents) VALUES ($1, $2, $3) RETURNING id"
    ).bind(req.from_account).bind(req.to_account).bind(req.amount_cents)
        .fetch_one(&mut *tx).await?;

    sqlx::query(
        "INSERT INTO idempotency_keys (key, payment_id, created_at) VALUES ($1, $2, NOW())"
    ).bind(&req.idempotency_key).bind(payment_id.0).execute(&mut *tx).await?;

    // 3. Adjust balances (also in same tx)
    sqlx::query("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2")
        .bind(req.amount_cents).bind(req.from_account).execute(&mut *tx).await?;
    sqlx::query("UPDATE accounts SET balance_cents = balance_cents + $1 WHERE id = $2")
        .bind(req.amount_cents).bind(req.to_account).execute(&mut *tx).await?;

    tx.commit().await?;
    Ok(payment_id.0)
}
```

**Talking points**:
- Idempotency key + payment in **one transaction** — both succeed or both fail. No partial state.
- Use `INSERT ... ON CONFLICT DO NOTHING RETURNING` for a single round-trip alternative.
- TTL on idempotency keys (e.g., 24h) to prevent unbounded growth.
- Concurrency safety: PostgreSQL's `SERIALIZABLE` isolation, OR row-level lock on the accounts.
- For double-entry bookkeeping: insert `+amount` and `-amount` ledger entries — never `UPDATE` balances directly. **Mention this — it's the canonical fintech answer.**

---

## 47. Retry Policy with Exponential Backoff

```rust
use std::time::Duration;
use rand::Rng;

#[derive(Debug, Clone)]
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub base_delay_ms: u64,
    pub max_delay_ms: u64,
    pub jitter: bool,
}

impl Default for RetryPolicy {
    fn default() -> Self {
        Self { max_attempts: 5, base_delay_ms: 100, max_delay_ms: 30_000, jitter: true }
    }
}

#[derive(Debug)]
pub enum RetryError<E> {
    Retryable(E),
    Permanent(E),
}

pub async fn retry<F, Fut, T, E>(policy: RetryPolicy, mut op: F) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, RetryError<E>>>,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(v) => return Ok(v),
            Err(RetryError::Permanent(e)) => return Err(e),
            Err(RetryError::Retryable(e)) => {
                attempt += 1;
                if attempt >= policy.max_attempts {
                    return Err(e);
                }
                let backoff = (policy.base_delay_ms * (1u64 << attempt.min(20)))
                    .min(policy.max_delay_ms);
                let delay = if policy.jitter {
                    let mut rng = rand::thread_rng();
                    rng.gen_range(0..=backoff)
                } else {
                    backoff
                };
                tokio::time::sleep(Duration::from_millis(delay)).await;
            }
        }
    }
}
```

**Talking points**:
- **Jitter is mandatory** — otherwise N clients all retry at exactly the same moments (thundering herd).
- **Error classification matters**: 5xx is retryable, 4xx is not. Network timeout is retryable, "invalid request" is not.
- Pair with **circuit breaker** — after N consecutive failures, fail fast for a cooldown period.
- For payments: retry on transport errors only. **Never blindly retry a POST that may have already mutated state — use the idempotency key from #46.**

---

# Rust Conceptual Questions

Short answers you should be able to deliver fluently.

### 1. `String` vs `&str`?
- `String` is heap-owned, growable, UTF-8 — like `Vec<u8>` with UTF-8 invariant.
- `&str` is a borrowed view into UTF-8 bytes (a slice — pointer + length).
- Use `&str` for parameters (you don't need ownership). Use `String` for return values or when you need to grow.

### 2. `Rc<T>` vs `Arc<T>` vs `Box<T>`?
- `Box<T>` — single owner, heap-allocated. Free.
- `Rc<T>` — multiple owners, single-threaded reference count. Cheap.
- `Arc<T>` — multiple owners, atomic reference count, thread-safe. Slightly more expensive than `Rc`.

### 3. `Send + Sync`?
- `Send`: safe to **transfer ownership** of `T` across threads.
- `Sync`: safe to **share a reference** (`&T`) across threads.
- Most types are both automatically (marker traits). `Rc`/`Cell`/`RefCell` are neither. `Arc`/`Mutex` are both.

### 4. `Mutex<T>` vs `RwLock<T>`?
- `Mutex<T>`: one writer OR one reader at a time. Simple, fast.
- `RwLock<T>`: many readers OR one writer. Use when reads vastly outnumber writes.
- For very short critical sections, `Mutex` often wins despite `RwLock`'s theoretical advantage.

### 5. `impl Trait` vs `dyn Trait`?
- `impl Trait`: static dispatch, monomorphized at compile time. Faster (often inlined), larger binary.
- `dyn Trait`: dynamic dispatch via vtable. Slower (indirect call), smaller binary, supports heterogeneous collections like `Vec<Box<dyn Trait>>`.

### 6. What does `?` do?
- Unwraps a `Result::Ok` or returns the `Err` from the current function (after `From::from` conversion).
- `let x = foo()?;` desugars to roughly `let x = match foo() { Ok(v) => v, Err(e) => return Err(e.into()) };`.

### 7. Ownership in 60 seconds
- Every value has exactly one owner. When the owner goes out of scope, the value is dropped.
- You can borrow: shared (`&T`, many) OR mutable (`&mut T`, one). Never both at once.
- The borrow checker enforces this at compile time. Result: no use-after-free, no data races, no GC.

### 8. What is a lifetime?
- A compile-time annotation describing how long a reference is valid.
- Required when the compiler can't infer relationships (e.g., function returning a reference derived from multiple inputs).
- `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` — output lives at least as long as the shorter input.

### 9. `Pin` and why `Future`s need it?
- `Pin<P>` guarantees the pointee won't move in memory.
- Async functions are compiled to state machines that may contain **self-references** (e.g., a future borrowing a local variable across an `.await`).
- Moving such a state machine would invalidate those internal pointers. `Pin` prevents that.

### 10. `Vec<u8>` faster than `Vec<i32>`?
- `u8` is 1 byte, `i32` is 4. A `Vec<u8>` of N elements fits in 1/4 the cache lines of `Vec<i32>` of N elements.
- Iterating linearly hits fewer cache misses. For numeric data SIMD also vectorizes wider.

### 11. What does `#[derive(Clone)]` generate?
- An `impl Clone for T` that field-by-field clones each member. Requires every field to be `Clone`.
- For `Copy`, also generates `impl Copy` — shallow bitwise copy, no `clone()` semantics needed.

### 12. `iter()` vs `iter_mut()` vs `into_iter()`?
- `iter()` yields `&T` — borrows immutably.
- `iter_mut()` yields `&mut T` — borrows mutably.
- `into_iter()` yields `T` — consumes the collection. After this, you can't use the original.

### 13. `async fn` returns what type?
- Conceptually, it returns `impl Future<Output = T>`.
- The compiler generates an anonymous state-machine type. Every `.await` point becomes a state transition.

### 14. `drop(Arc<T>)`?
- Atomically decrements the strong reference count.
- If the count reaches zero, runs `T::drop()` and frees the allocation.
- If `Weak<T>` references still exist, the *control block* stays alive; only the inner `T` is dropped.

### 15. When to use `unsafe`?
- Calling FFI (C bindings).
- Dereferencing raw pointers (e.g., performance-critical lock-free data structures).
- Implementing fundamental abstractions (`Vec`, `Mutex` internals).
- Manual SIMD intrinsics.
- **Wrap `unsafe` in a safe API** and document the invariants you uphold. The Rustonomicon is the canonical reference.

---

## Final advice

- **Don't memorize**. Internalize the patterns. Interviewers will twist a problem mid-question to see if you understand or pattern-match.
- **Talk while you code.** Narrate your decisions: "I'm reaching for `BTreeMap` here because we need ordered iteration; if we only needed lookups I'd use `HashMap`."
- **Always discuss complexity** when you finish — time AND space.
- **Compile every solution before claiming done.** A non-compiling solution fails the round, even if logically correct.
- **For domain problems, mention `rust_decimal`.** It's the single biggest "I get fintech Rust" signal.
- **Cite production patterns**: "In production I'd add a metric here," "I'd put this behind a trait for testability," "I'd use `tracing` spans here." That's senior-level talk.

Good luck. You've got this.
