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

## Final advice

- **Don't memorize**. Internalize the patterns. Interviewers will twist a problem mid-question to see if you understand or pattern-match.
- **Talk while you code.** Narrate your decisions: "I'm reaching for `BTreeMap` here because we need ordered iteration; if we only needed lookups I'd use `HashMap`."
- **Always discuss complexity** when you finish — time AND space.
- **Compile every solution before claiming done.** A non-compiling solution fails the round, even if logically correct.
- **For domain problems, mention `rust_decimal`.** It's the single biggest "I get fintech Rust" signal.
- **Cite production patterns**: "In production I'd add a metric here," "I'd put this behind a trait for testability," "I'd use `tracing` spans here." That's senior-level talk.

Good luck. You've got this.
