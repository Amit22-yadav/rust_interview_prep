# Actlogica Rust Backend Interview — Coding Question Bank

Targeted prep for the Product Engineer (Backend) role at Actlogica Solutions. Questions are grouped by category and ordered from most-likely to least-likely. **Solve in Rust**, idiomatic where possible.

For each problem: aim to (a) state the approach in 30 seconds, (b) discuss time/space complexity, (c) write clean, compiling code, (d) walk through one test case.

---

## How to use this list

- ⭐ **MUST DO** = the 6 highest-priority problems for this specific interview. Solve these cold, multiple times, before anything else.
- **Tier S** = practice until you can solve them cold. These are the canonical "Rust backend interview" problems.
- **Tier A** = solve at least once, understand the pattern.
- **Tier B** = skim, know the approach.
- **Domain** = financial/fintech problems aligned with Actlogica's wealth-management domain. **High signal at this company.**
- **Concurrency / Async** = Rust-specific async problems. Almost certain to be asked.

Time budget: **10–14 days, ~2 hrs/day** to get through Tier S + Tier A + Domain. Concurrency is non-negotiable.

---

## ⭐ TOP 6 — MUST DO FIRST

If you only have time for 6 problems, do these. They cover the most likely interview questions and exercise the patterns everything else builds on. **Practice each one 3 times, in a real editor, with `cargo run`.**

| # | Problem | Category | Why |
|---|---|---|---|
| 1 | LRU Cache | Tier S | The single most-asked Rust mid-level problem |
| 6 | Word Frequency Counter | Tier S | HashMap + iterators + scale discussion |
| 8 | Producer–Consumer with Bounded Channel | Tier S | The canonical Tokio question |
| 29 | VWAP from CSV | Domain | Direct domain fit for Actlogica's wealth-management work |
| 36 | Thread-Safe Counter | Concurrency | Mutex vs Atomic tradeoffs — concurrency fundamentals |
| 37 | Rate Limiter (Token Bucket) | Concurrency | Time + concurrency + production patterns |

Solutions for all 6 are in [solution.md](solution.md). Don't move on to other problems until you can solve these 6 from scratch in under 25 minutes each.

---

## Tier S — Must solve cold

### 1. LRU Cache ⭐ MUST DO
- **Problem**: Design and implement a Least Recently Used (LRU) cache with `get(key)` and `put(key, value)` in O(1).
- **Why asked**: The single most common Rust interview problem at the mid level. Tests `HashMap`, references, and (if you do it with linked list) `Rc<RefCell<T>>`.
- **Approach**: `HashMap<K, usize>` + `VecDeque<(K, V)>`, or `HashMap` + doubly linked list with `Rc<RefCell<Node>>`. In Rust, the `Vec`-of-indices approach is simpler and often what they want.
- **LeetCode**: [146. LRU Cache](https://leetcode.com/problems/lru-cache/)
- **Bonus**: [460. LFU Cache](https://leetcode.com/problems/lfu-cache/)
- **Solution**: [solution.md#1-lru-cache](solution.md#1-lru-cache)

### 2. Two Sum
- **Problem**: Given `nums` and `target`, return indices of two numbers that add to target.
- **Why asked**: Warmup. They're checking if you know `HashMap` and `enumerate()` in Rust.
- **Approach**: One-pass `HashMap<i32, usize>`.
- **LeetCode**: [1. Two Sum](https://leetcode.com/problems/two-sum/)
- **Solution**: [solution.md#2-two-sum](solution.md#2-two-sum)

### 3. Valid Parentheses
- **Problem**: Given a string of brackets, decide if they're balanced.
- **Why asked**: Stack-based parsing. Tests `Vec` as stack, `chars()`, pattern matching.
- **LeetCode**: [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)
- **Solution**: [solution.md#3-valid-parentheses](solution.md#3-valid-parentheses)

### 4. Merge Intervals
- **Problem**: Merge overlapping intervals.
- **Why asked**: Classic. Tests sorting with custom comparator, `Vec` mutation, edge cases.
- **LeetCode**: [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/)
- **Solution**: [solution.md#4-merge-intervals](solution.md#4-merge-intervals)

### 5. Top K Frequent Elements
- **Problem**: Return the `k` most frequent elements in an array.
- **Why asked**: Real-world (top traded symbols, hot portfolios). Tests `HashMap` + heap (`BinaryHeap`).
- **Approach**: Count with `HashMap`, then `BinaryHeap<(count, value)>` of size k, or bucket sort.
- **LeetCode**: [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/)
- **Solution**: [solution.md#5-top-k-frequent-elements](solution.md#5-top-k-frequent-elements)

### 6. Word Frequency Counter (Custom) ⭐ MUST DO
- **Problem**: Given a stream of strings, return the top N most frequent. Now scale it: what if the stream doesn't fit in memory?
- **Why asked**: Common in product-company interviews. Tests iterators, `HashMap`, and design-thinking for scale.
- **Approach**: `HashMap<String, u64>` for in-memory; for streaming, sketch a count-min sketch or external sort.
- **No LeetCode link** — practice on your own using `std::io::BufRead` reading from stdin.
- **Solution**: [solution.md#6-word-frequency-counter](solution.md#6-word-frequency-counter)

### 7. Binary Search Variants
- **Problem**: Implement binary search; then variants — first occurrence, last occurrence, insertion point.
- **Why asked**: Cornerstone algorithm. Tests off-by-one rigor.
- **LeetCode**:
  - [704. Binary Search](https://leetcode.com/problems/binary-search/)
  - [34. Find First and Last Position](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)
  - [35. Search Insert Position](https://leetcode.com/problems/search-insert-position/)
- **Solution**: [solution.md#7-binary-search-variants](solution.md#7-binary-search-variants)

### 8. Producer–Consumer with Bounded Channel ⭐ MUST DO
- **Problem**: Spawn N producers and M consumers communicating via a bounded async channel. Producers generate items, consumers process and aggregate.
- **Why asked**: The canonical Tokio interview question. Tests `tokio::sync::mpsc`, `tokio::spawn`, `JoinSet`, `select!`.
- **Approach**: `tokio::sync::mpsc::channel(capacity)`, multiple `tokio::spawn` for producers, single or multiple for consumers, `JoinSet` to await.
- **No LeetCode link** — code this from scratch in a `cargo new` project.
- **Solution**: [solution.md#8-producerconsumer-with-bounded-channel](solution.md#8-producerconsumer-with-bounded-channel)

---

## Tier A — Solve at least once

### 9. Longest Substring Without Repeating Characters
- **Approach**: Sliding window + `HashMap<char, usize>` for last seen index.
- **LeetCode**: [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)
- **Solution**: [solution.md#16-longest-substring-without-repeating-characters](solution.md#16-longest-substring-without-repeating-characters)

### 10. Group Anagrams
- **Approach**: Sort each string as the key into `HashMap<String, Vec<String>>`.
- **LeetCode**: [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- **Solution**: [solution.md#17-group-anagrams](solution.md#17-group-anagrams)

### 11. Product of Array Except Self
- **Approach**: Prefix and suffix products; no division.
- **LeetCode**: [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/)
- **Solution**: [solution.md#18-product-of-array-except-self](solution.md#18-product-of-array-except-self)

### 12. Maximum Subarray (Kadane's)
- **Approach**: Single-pass tracking running max.
- **LeetCode**: [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)
- **Solution**: [solution.md#19-maximum-subarray-kadanes](solution.md#19-maximum-subarray-kadanes)

### 13. Best Time to Buy and Sell Stock
- **Why asked**: **Domain-aligned** — Actlogica builds wealth/portfolio software. Expect a question framed as "given a series of prices, compute X."
- **LeetCode**:
  - [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
  - [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)
  - [123. Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)
- **Solution**: [solution.md#20-best-time-to-buy-and-sell-stock](solution.md#20-best-time-to-buy-and-sell-stock)

### 14. Implement a Trie (Prefix Tree)
- **Approach**: `struct TrieNode { children: HashMap<char, TrieNode>, is_end: bool }`.
- **LeetCode**: [208. Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/)
- **Solution**: [solution.md#21-implement-a-trie](solution.md#21-implement-a-trie)

### 15. Number of Islands (Graph DFS/BFS)
- **Approach**: DFS or BFS over grid, mark visited.
- **LeetCode**: [200. Number of Islands](https://leetcode.com/problems/number-of-islands/)
- **Solution**: [solution.md#22-number-of-islands](solution.md#22-number-of-islands)

### 16. Course Schedule (Topological Sort)
- **Approach**: Kahn's algorithm with in-degree counts.
- **LeetCode**: [207. Course Schedule](https://leetcode.com/problems/course-schedule/)
- **Solution**: [solution.md#23-course-schedule-topological-sort](solution.md#23-course-schedule-topological-sort)

### 17. Kth Largest Element
- **Approach**: `BinaryHeap` (min-heap of size k) or quickselect.
- **LeetCode**: [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)
- **Solution**: [solution.md#24-kth-largest-element](solution.md#24-kth-largest-element)

### 18. Merge K Sorted Lists
- **Approach**: Min-heap of (value, list-id, index).
- **LeetCode**: [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)
- **Solution**: [solution.md#25-merge-k-sorted-lists](solution.md#25-merge-k-sorted-lists)

### 19. Subsets / Permutations / Combinations
- **Approach**: Backtracking. Practice the template once.
- **LeetCode**:
  - [78. Subsets](https://leetcode.com/problems/subsets/)
  - [46. Permutations](https://leetcode.com/problems/permutations/)
  - [77. Combinations](https://leetcode.com/problems/combinations/)
- **Solution**: [solution.md#26-subsets--permutations--combinations](solution.md#26-subsets--permutations--combinations)

### 20. Coin Change (DP)
- **Approach**: 1D DP, `dp[i] = min coins to make amount i`.
- **LeetCode**: [322. Coin Change](https://leetcode.com/problems/coin-change/)
- **Solution**: [solution.md#27-coin-change-dp](solution.md#27-coin-change-dp)

---

## Tier B — Know the approach

### 21. Word Break
- **LeetCode**: [139. Word Break](https://leetcode.com/problems/word-break/)
- **Solution**: [solution.md#28-word-break](solution.md#28-word-break)

### 22. Longest Palindromic Substring
- **LeetCode**: [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)
- **Solution**: [solution.md#29-longest-palindromic-substring](solution.md#29-longest-palindromic-substring)

### 23. Rotate Image (90° matrix)
- **LeetCode**: [48. Rotate Image](https://leetcode.com/problems/rotate-image/)
- **Solution**: [solution.md#30-rotate-image](solution.md#30-rotate-image)

### 24. Spiral Matrix
- **LeetCode**: [54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix/)
- **Solution**: [solution.md#31-spiral-matrix](solution.md#31-spiral-matrix)

### 25. Set Matrix Zeroes
- **LeetCode**: [73. Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes/)
- **Solution**: [solution.md#32-set-matrix-zeroes](solution.md#32-set-matrix-zeroes)

### 26. Search in Rotated Sorted Array
- **LeetCode**: [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/)
- **Solution**: [solution.md#33-search-in-rotated-sorted-array](solution.md#33-search-in-rotated-sorted-array)

### 27. Linked List Cycle
- **Approach**: Floyd's tortoise and hare.
- **LeetCode**: [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) — *note: linked lists in Rust are awkward; this might come as a design discussion, not a coding ask.*
- **Solution**: [solution.md#34-linked-list-cycle](solution.md#34-linked-list-cycle)

### 28. Reverse Linked List
- **LeetCode**: [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) — same caveat as above.
- **Solution**: [solution.md#35-reverse-linked-list](solution.md#35-reverse-linked-list)

---

## Domain — Fintech / Wealth Management problems

These are aligned with Actlogica's products. Practice these specifically — they signal you understand their business.

### 29. Compute VWAP (Volume-Weighted Average Price) ⭐ MUST DO
- **Problem**: Given a list of trades `(symbol, price, quantity, timestamp)`, compute VWAP per symbol.
- **Approach**: `HashMap<String, (f64 sum_pv, u64 sum_v)>`, then `vwap = sum_pv / sum_v`.
- **Why asked**: Direct domain fit. Tests iterators, `HashMap`, floating-point handling.
- **Variation**: time-windowed VWAP (last 1 hour).
- **Solution**: [solution.md#9-vwap-volume-weighted-average-price](solution.md#9-vwap-volume-weighted-average-price)

### 30. Order Book Top of Book
- **Problem**: Given a stream of orders `(side, price, quantity)` and cancels, maintain the top-of-book (best bid, best ask).
- **Approach**: `BTreeMap<Reverse<OrderedFloat<f64>>, u64>` for bids, `BTreeMap<OrderedFloat<f64>, u64>` for asks.
- **Why asked**: **This is your matching-engine work in interview form.** If they ask this, you're winning.
- **Variation**: handle partial fills.
- **Solution**: [solution.md#10-order-book-top-of-book](solution.md#10-order-book-top-of-book)

### 31. Portfolio NAV Calculation
- **Problem**: Given holdings `(symbol, quantity)` and prices `(symbol, price, timestamp)`, compute portfolio NAV.
- **Approach**: Join the two streams, sum `qty * price`. Handle missing prices.
- **Why asked**: FinFlo and Alpha Portfolio both compute NAV. **High-signal question.**
- **Solution**: [solution.md#11-portfolio-nav](solution.md#11-portfolio-nav)

### 32. Reconciliation: Two-Source Diff
- **Problem**: Two sources report trades — your internal ledger and a broker feed. Both are `Vec<Trade>` (or streams). Find trades in one but not the other, with a tolerance for floating-point amounts.
- **Approach**: `HashMap<TradeKey, Trade>` for one side, iterate the other, compare. Or sort and merge-walk.
- **Why asked**: Reconciliation is a stated FinFlo feature. **Very high-signal.**
- **Solution**: [solution.md#12-reconciliation-two-source-diff](solution.md#12-reconciliation-two-source-diff)

### 33. Time-Series Aggregation
- **Problem**: Given a stream of price ticks `(symbol, price, timestamp)`, emit 1-minute OHLC (open, high, low, close) candles per symbol.
- **Approach**: `HashMap<(Symbol, MinuteBucket), Candle>`. Discuss late-arriving ticks.
- **Why asked**: Standard fintech aggregation question.
- **Solution**: [solution.md#13-time-series-ohlc-aggregation](solution.md#13-time-series-ohlc-aggregation)

### 34. Position Tracker
- **Problem**: Given a stream of executed trades `(symbol, side, quantity, price)`, maintain running positions per symbol with average cost basis. Handle buys, sells, and shorts.
- **Approach**: `HashMap<Symbol, Position { qty, avg_cost }>`. Update rules: buy → blend cost basis; sell → reduce quantity, realize P&L.
- **Why asked**: Bread-and-butter portfolio accounting. Tests careful state updates.
- **Solution**: [solution.md#14-position-tracker](solution.md#14-position-tracker)

### 35. Simple Interest Rate Calculator (with edge cases)
- **Problem**: Given principal, rate, time, and compounding frequency, compute final amount. Then add: handle midnight rollovers, day-count conventions (ACT/360, ACT/365, 30/360).
- **Why asked**: Actuarial-domain primer.
- **Solution**: [solution.md#15-interest-rate-calculator](solution.md#15-interest-rate-calculator)

---

## Concurrency & Async — Rust-specific

These are the **most likely Rust-specific live coding asks**. Practice in a real `cargo new` project.

### 36. Thread-Safe Counter ⭐ MUST DO
- **Problem**: A counter that N threads can increment safely. Compare `Arc<Mutex<u64>>` vs `Arc<AtomicU64>`. When is each correct?
- **Talking points**: Mutex has overhead but supports complex updates; `AtomicU64` is faster but only supports atomic primitives.
- **Solution**: [solution.md#36-thread-safe-counter](solution.md#36-thread-safe-counter)

### 37. Rate Limiter (Token Bucket) ⭐ MUST DO
- **Problem**: Implement a per-key rate limiter. `try_acquire(key) -> bool`.
- **Approach**: `HashMap<Key, TokenBucket>` behind a `Mutex` or `DashMap`; refill on each call based on elapsed time.
- **Variation**: distributed via Redis (discussion-only).
- **Solution**: [solution.md#37-rate-limiter-token-bucket](solution.md#37-rate-limiter-token-bucket)

### 38. Concurrent Bounded Queue with Backpressure
- **Problem**: Implement an async-aware bounded queue. Producers `await` when full; consumers `await` when empty.
- **Approach**: `tokio::sync::mpsc::channel(capacity)` — they may want you to discuss what it does internally.
- **Solution**: [solution.md#38-concurrent-bounded-queue](solution.md#38-concurrent-bounded-queue)

### 39. Fan-out / Fan-in with `tokio::spawn` + `JoinSet`
- **Problem**: Given a list of URLs, fetch all concurrently with a max in-flight of N, collect results.
- **Approach**: `JoinSet`, `Semaphore` for concurrency limit.
- **Solution**: [solution.md#39-fan-out--fan-in](solution.md#39-fan-out--fan-in)

### 40. Graceful Shutdown with `tokio::select!`
- **Problem**: A server loop that processes work but exits cleanly on Ctrl-C.
- **Approach**: `tokio::select! { _ = work_loop => ..., _ = signal::ctrl_c() => ... }`.
- **Solution**: [solution.md#40-graceful-shutdown](solution.md#40-graceful-shutdown)

### 41. Implement a Future from Scratch (advanced)
- **Problem**: Write a custom `Future` that resolves after N polls (or after a timer).
- **Why asked**: Senior screen. Tests `Pin`, `Context`, `Waker`. If asked, even partial credit shows depth.
- **Solution**: [solution.md#41-implement-a-future-from-scratch](solution.md#41-implement-a-future-from-scratch)

### 42. Dead-Lock Detection / Avoidance Discussion
- **Problem**: They show you two `Mutex`-locking functions that can deadlock. Identify and fix.
- **Approach**: Lock ordering, `try_lock`, splitting into smaller critical sections, `parking_lot::Mutex` with timeout.
- **Solution**: [solution.md#42-deadlock-detection--avoidance](solution.md#42-deadlock-detection--avoidance)

### 43. Channel-Based State Machine (Actor Pattern)
- **Problem**: Implement an "account service" actor: one task owns the state, receives commands `Deposit(amount)`, `Withdraw(amount)`, `GetBalance(reply)` via a channel.
- **Approach**: `tokio::sync::mpsc::channel` for commands, `tokio::sync::oneshot` for replies.
- **Why asked**: Idiomatic Rust concurrency without locks. **This is a strong "I know Rust" answer.**
- **Solution**: [solution.md#43-channel-based-state-machine-actor](solution.md#43-channel-based-state-machine-actor)

---

## System Design coding-adjacent

These come up as 20–30 min discussions, not full implementations. Prepare a structured answer for each.

### 44. Design a URL Shortener
- **LeetCode**: [535. Encode and Decode TinyURL](https://leetcode.com/problems/encode-and-decode-tinyurl/)
- **Talking points**: ID generation (base62, snowflake), storage (Redis + Postgres), cache layer, hot-key handling.
- **Solution**: [solution.md#44-url-shortener](solution.md#44-url-shortener)

### 45. Design a Distributed Counter
- **Approach**: Sharded counters per node, periodic aggregation. Discuss CRDTs.
- **Solution**: [solution.md#45-distributed-counter](solution.md#45-distributed-counter)

### 46. Design an Idempotent Payment API
- **Approach**: Idempotency keys, `INSERT ... ON CONFLICT` in Postgres, dedupe window. **Domain-relevant.**
- **Solution**: [solution.md#46-idempotent-payment-api](solution.md#46-idempotent-payment-api)

### 47. Design a Retry Policy
- **Approach**: Exponential backoff + jitter, max attempts, circuit breaker, error classification (retryable vs not).
- **Solution**: [solution.md#47-retry-policy-with-exponential-backoff](solution.md#47-retry-policy-with-exponential-backoff)

---

## Rust-specific conceptual questions (not coding, but expect them)

These are likely during the technical round between problems. Have crisp answers.

**Answers**: [solution.md#rust-conceptual-questions](solution.md#rust-conceptual-questions)


1. **What's the difference between `String` and `&str`?**
2. **When would you use `Rc<T>` vs `Arc<T>` vs `Box<T>`?**
3. **What does `Send + Sync` mean?**
4. **`Mutex<T>` vs `RwLock<T>` — when each?**
5. **`impl Trait` vs `dyn Trait` — what's the runtime difference?**
6. **What does `?` do? What does it desugar to?**
7. **Explain ownership in 60 seconds.**
8. **What is a lifetime? When do you need to annotate one?**
9. **What is `Pin` and why do `Future`s need it?**
10. **Why is `Vec<u8>` faster than `Vec<i32>` for some operations?** (cache-line, alignment)
11. **What does `#[derive(Clone)]` actually generate?**
12. **What's the difference between `iter()`, `iter_mut()`, and `into_iter()`?**
13. **`async fn` returns what type, conceptually?**
14. **What happens when you `drop` an `Arc<T>`?**
15. **When would you use `unsafe`? Give a concrete example.**

---

## Practice plan (10 days)

| Day | Focus | Problems |
|-----|-------|----------|
| 1 | Tier S basics | #1 LRU, #2 Two Sum, #3 Valid Parens |
| 2 | Tier S continued | #4 Merge Intervals, #5 Top K, #6 Word Frequency |
| 3 | Binary search + async | #7 Binary Search variants, #8 Producer-Consumer |
| 4 | Domain | #29 VWAP, #30 Order Book, #31 NAV |
| 5 | Domain | #32 Reconciliation, #33 OHLC, #34 Position Tracker |
| 6 | Concurrency | #36 Counter, #37 Rate Limiter, #38 Bounded Queue |
| 7 | Concurrency | #39 Fan-out, #40 Graceful Shutdown, #43 Actor pattern |
| 8 | Tier A sweep | Pick 5 from #9–#20 |
| 9 | System design + concepts | #44–#47 + conceptual Q&A |
| 10 | Mock interview | Full simulation, light review |

---

## Setup tips

- Use a fresh `cargo new --bin actlogica-prep` repo. One module per problem (`mod lru_cache;` etc.).
- Run each solution with `cargo test` — write at least one unit test per problem.
- For async problems use `[tokio::main]` and `tokio = { version = "1", features = ["full"] }`.
- For the order-book / NAV / reconciliation problems, use `rust_decimal` (not `f64`) — financial code uses decimals to avoid rounding errors. **Saying "I'd use `rust_decimal` here for accurate money arithmetic" is a strong signal.**

---

## Final notes

- **Don't memorize solutions.** Memorize patterns. Interviewers vary the problem slightly.
- **Talk while you code.** Silence reads as panic. Narrate: "I'm reaching for a HashMap because we need O(1) lookup..."
- **Compile before claiming done.** A solution that doesn't compile fails the round, even if the logic is correct.
- **Time matters.** Tier S problems should be done in 20 min. If you're stuck at 30, ask for a hint — that's a positive signal, not negative.
- **Always discuss complexity** at the end. Big-O is the standard closer.

Good luck.
