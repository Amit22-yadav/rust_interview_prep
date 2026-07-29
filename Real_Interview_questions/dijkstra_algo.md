# Dijkstra's Algorithm — Explanation, Working & Rust Implementation

Single-source shortest path on a weighted graph with **non-negative** edge weights. This file covers what the algorithm is, how it works step by step, why it's correct, its complexity, and a complete, tested Rust implementation with interview-ready design notes.

---

## Index

| # | Section | Link |
|---|---------|------|
| 1 | What problem does it solve? | [Jump](#1-what-problem-does-it-solve) |
| 2 | Core idea and intuition | [Jump](#2-core-idea-and-intuition) |
| 3 | The algorithm, step by step | [Jump](#3-the-algorithm-step-by-step) |
| 4 | Worked example (dry run) | [Jump](#4-worked-example--dry-run) |
| 5 | Why it's correct | [Jump](#5-why-its-correct--the-greedy-proof) |
| 6 | Why non-negative weights are required | [Jump](#6-why-non-negative-weights-are-required) |
| 7 | Time & space complexity | [Jump](#7-time--space-complexity) |
| 8 | Rust implementation | [Jump](#8-rust-implementation) |
| 9 | Rust-specific design notes | [Jump](#9-rust-specific-design-notes-interview-gold) |
| 10 | Dijkstra vs other algorithms | [Jump](#10-dijkstra-vs-other-shortest-path-algorithms) |
| 11 | Common interview follow-ups | [Jump](#11-common-interview-follow-ups) |

---

## 1. What problem does it solve?

Given a weighted graph and a **start node**, find the **shortest (minimum total weight) path** from that start node to **every other reachable node**.

- **Input:** a graph `G = (V, E)` where each edge has a non-negative weight, plus a source node `s`.
- **Output:** `dist[v]` = the minimum total weight of any path from `s` to `v`, for every `v`. Usually also a `predecessor` map so you can reconstruct the actual path, not just its cost.

**Real-world uses:** GPS/route planning, network packet routing (OSPF and IS-IS are Dijkstra-based), game pathfinding, cheapest-flight problems, dependency resolution with costs.

---

## 2. Core idea and intuition

Dijkstra is a **greedy** algorithm built on one observation:

> The unvisited node with the **smallest known tentative distance** can never be improved later, so its distance is final.

The intuition: imagine water flooding out from the source at uniform speed along every edge simultaneously. The moment water reaches a node, that's the shortest path to it — nothing arriving later can be shorter. Dijkstra simulates that flood, processing nodes in increasing order of distance from the source.

Two sets of nodes exist conceptually:
- **Settled / finalized** — shortest distance is known for certain.
- **Frontier / tentative** — reachable, but we may still find a cheaper route.

Each step moves the cheapest frontier node into the settled set, then **relaxes** its outgoing edges.

**Relaxation** is the single operation at the heart of it:

```
if dist[u] + weight(u, v) < dist[v]:
    dist[v] = dist[u] + weight(u, v)
    predecessor[v] = u
```

In words: "if going to `v` *via* `u` is cheaper than the best route to `v` I currently know, take that route instead."

---

## 3. The algorithm, step by step

```
1. dist[start] = 0;  dist[everything else] = ∞
2. Push (0, start) into a min-priority-queue
3. While the queue is not empty:
     a. Pop the node `u` with the SMALLEST tentative distance
     b. If this popped distance > dist[u], it's a stale entry → skip it
     c. For each neighbor `v` of `u` with edge weight `w`:
          candidate = dist[u] + w
          if candidate < dist[v]:
              dist[v] = candidate
              predecessor[v] = u
              push (candidate, v) into the queue
4. Return dist and predecessor
```

**Step 3b — the stale-entry check — is the part people get wrong.** A standard binary heap has no efficient "decrease-key" operation. So instead of updating a node's priority in place, we simply **push a new, better entry** and leave the old one behind. That means a node can appear in the heap several times with different distances. When we pop one, we check whether it's outdated:

```rust
if current_dist > best_known { continue; }   // stale — already found something better
```

This is called the **lazy deletion** variant. It's what almost every practical implementation does, and it's the standard approach in Rust because `BinaryHeap` has no `decrease_key`.

---

## 4. Worked example — dry run

Using the exact graph from the implementation below:

```
        4
   A ───────► B
   │          │
 1 │          │ 5
   ▼    2     ▼
   C ───────► B
   │
 8 │
   ▼
   D                B ──5──► D
```

Edges: `A→B (4)`, `A→C (1)`, `C→B (2)`, `B→D (5)`, `C→D (8)`

| Step | Heap (min first) | Popped | Action | `dist` after |
|------|------------------|--------|--------|--------------|
| init | `(0,A)` | — | — | `A:0` |
| 1 | `(0,A)` | `(0,A)` | Relax A→B: 0+4=4 ✅<br>Relax A→C: 0+1=1 ✅ | `A:0, B:4, C:1` |
| 2 | `(1,C), (4,B)` | `(1,C)` | Relax C→B: 1+2=**3** < 4 ✅<br>Relax C→D: 1+8=9 ✅ | `A:0, B:3, C:1, D:9` |
| 3 | `(3,B), (4,B), (9,D)` | `(3,B)` | Relax B→D: 3+5=**8** < 9 ✅ | `A:0, B:3, C:1, D:8` |
| 4 | `(4,B), (8,D), (9,D)` | `(4,B)` | **STALE** — 4 > dist[B]=3 → skip | unchanged |
| 5 | `(8,D), (9,D)` | `(8,D)` | D has no outgoing edges | unchanged |
| 6 | `(9,D)` | `(9,D)` | **STALE** — 9 > dist[D]=8 → skip | unchanged |
| 7 | *(empty)* | — | done | **`A:0, B:3, C:1, D:8`** |

**Final answer:**
- `A → A` = 0
- `A → C` = 1, path `A → C`
- `A → B` = 3, path `A → C → B` *(cheaper than the direct A→B edge of 4!)*
- `A → D` = 8, path `A → C → B → D`

Notice steps 4 and 6 — those are exactly the stale entries the lazy-deletion check discards. And notice that the direct edge `A→B` with weight 4 loses to the two-hop route `A→C→B` costing 3. That's the whole point of relaxation.

**Path reconstruction** walks the `predecessor` map backwards from the goal:

```
predecessor = { C:A, B:C, D:B }

D → B → C → A     (walk backwards)
A → C → B → D     (reverse it)
```

---

## 5. Why it's correct — the greedy proof

**Claim:** when a node `u` is popped from the priority queue with the minimum tentative distance, `dist[u]` is already the true shortest distance.

**Proof sketch (by contradiction):**

Suppose `u` is popped with `dist[u]`, but a genuinely shorter path `P` to `u` exists with total weight `< dist[u]`.

`P` starts at `s` (which is settled) and ends at `u` (not yet settled). So somewhere along `P` there must be a first edge crossing from a settled node `x` to an unsettled node `y`.

- Because `x` is settled, `dist[x]` is final and correct, and we already relaxed the edge `x→y`. So `dist[y] ≤ dist[x] + w(x,y)` = the true cost of `P`'s prefix up to `y`.
- Because the remaining part of `P` from `y` to `u` has **non-negative** weight, cost of `P` ≥ cost of `P`'s prefix up to `y` ≥ `dist[y]`.
- We assumed cost of `P` < `dist[u]`, so `dist[y] ≤ cost(P) < dist[u]`.
- But that means `y` had a smaller tentative distance than `u` — so the priority queue would have popped `y` before `u`. **Contradiction.**

Therefore no such shorter path exists, and `dist[u]` is final. ∎

**The load-bearing step is the third bullet:** it relies on the remaining path having non-negative weight. That's precisely where negative edges break everything.

---

## 6. Why non-negative weights are required

If negative edges exist, a node can be finalized and *then* a cheaper route to it appears — but Dijkstra never revisits settled nodes, so it returns a wrong answer.

```
       2
  A ───────► B
  │          │
5 │          │ -10
  ▼          ▼
  C ◄────────┘
```

Edges: `A→B (2)`, `A→C (5)`, `B→C (-10)`

Dijkstra's trace:
1. Pop `A` (0). Relax: `dist[B]=2`, `dist[C]=5`.
2. Pop `B` (2) — smallest. Relax `B→C`: `2 + (-10) = -8 < 5` → `dist[C] = -8`.
3. Pop `C`… but with the *lazy* variant it may already have been popped at 5 and settled.

In the classic "settled set" formulation, `C` is finalized at 5 before `B→C` is ever relaxed → **wrong answer of 5 instead of −8**.

> **Interview note:** the *lazy* implementation below happens to be more forgiving because it re-pushes on improvement — but it is still **not** a correct negative-weight algorithm. It can degrade to exponential time and it cannot detect negative cycles (where "shortest path" is undefined, since you can loop forever getting cheaper). Never claim Dijkstra handles negative weights.

**Use instead:**
- **Bellman-Ford** — handles negative edges, detects negative cycles. `O(V·E)`.
- **Johnson's algorithm** — all-pairs with negative edges; reweights via Bellman-Ford, then runs Dijkstra from each node.
- **SPFA** — a queue-based Bellman-Ford optimization; good average case, bad worst case.

---

## 7. Time & space complexity

Let `V` = number of vertices, `E` = number of edges.

| Priority queue | Time | Notes |
|---|---|---|
| **Binary heap (lazy deletion)** | **`O((V + E) log V)`** ≈ `O(E log V)` | What the Rust code below does. Heap can hold up to `E` entries. |
| Binary heap + decrease-key | `O((V + E) log V)` | Same asymptotics, smaller heap; needs an index-tracking heap. |
| Fibonacci heap | `O(E + V log V)` | Better in theory, worse constants — rarely worth it in practice. |
| Unsorted array / linear scan | `O(V²)` | **Better for dense graphs** where `E ≈ V²`. |

**Deriving `O(E log V)`:** each edge relaxation may push one entry (`E` pushes), and each pop is `O(log)` of the heap size. With lazy deletion the heap holds `O(E)` entries, so it's technically `O(E log E)` — but `E ≤ V²` means `log E ≤ 2 log V`, so `O(E log E) = O(E log V)`.

**Space:** `O(V + E)` — the adjacency list is `O(V + E)`, and `dist`, `predecessor`, and the heap are each `O(V)` to `O(E)`.

**Practical guidance:** use a binary heap for **sparse** graphs (`E ≪ V²`, the common case — road networks, social graphs). Use the simple `O(V²)` array scan for **dense** graphs.

---

## 8. Rust implementation

```rust
// Dijkstra's shortest-path algorithm in Rust.
//
// Design notes (things worth saying out loud in an interview):
// - BinaryHeap is a MAX-heap by default, so we wrap distances in `Reverse`
//   to turn it into a min-heap (we always want to pop the smallest distance).
// - We use a HashMap for adjacency + distances rather than a fixed-size
//   Vec/array because node IDs here are arbitrary strings; in a
//   performance-critical version you'd intern nodes as u32 indices into
//   a Vec instead, avoiding hashing overhead entirely.
// - No `unsafe` needed — the ownership model handles this cleanly.

use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashMap};

/// A weighted, directed graph represented as an adjacency list.
struct Graph {
    // node -> list of (neighbor, edge_weight)
    adjacency: HashMap<String, Vec<(String, u32)>>,
}

impl Graph {
    fn new() -> Self {
        Graph {
            adjacency: HashMap::new(),
        }
    }

    /// Adds a directed edge. Call twice (both directions) for an
    /// undirected graph.
    fn add_edge(&mut self, from: &str, to: &str, weight: u32) {
        self.adjacency
            .entry(from.to_string())
            .or_insert_with(Vec::new)
            .push((to.to_string(), weight));

        // Ensure the destination node exists in the map even with no
        // outgoing edges, so it's reachable in iteration.
        self.adjacency.entry(to.to_string()).or_insert_with(Vec::new);
    }

    /// Returns the shortest distance from `start` to every reachable node,
    /// plus a `predecessor` map you can walk backwards to reconstruct
    /// the actual path.
    fn dijkstra(&self, start: &str) -> (HashMap<String, u32>, HashMap<String, String>) {
        let mut dist: HashMap<String, u32> = HashMap::new();
        let mut predecessor: HashMap<String, String> = HashMap::new();
        let mut heap: BinaryHeap<Reverse<(u32, String)>> = BinaryHeap::new();

        dist.insert(start.to_string(), 0);
        heap.push(Reverse((0, start.to_string())));

        while let Some(Reverse((current_dist, node))) = heap.pop() {
            // Stale entry check: we may have pushed this node multiple
            // times with different distances before it was finalized.
            // If we've already found a shorter path, skip this pop.
            if let Some(&best_known) = dist.get(&node) {
                if current_dist > best_known {
                    continue;
                }
            }

            if let Some(neighbors) = self.adjacency.get(&node) {
                for (neighbor, weight) in neighbors {
                    let candidate_dist = current_dist + weight;
                    let is_shorter = dist
                        .get(neighbor)
                        .map_or(true, |&existing| candidate_dist < existing);

                    if is_shorter {
                        dist.insert(neighbor.clone(), candidate_dist);
                        predecessor.insert(neighbor.clone(), node.clone());
                        heap.push(Reverse((candidate_dist, neighbor.clone())));
                    }
                }
            }
        }

        (dist, predecessor)
    }

    /// Reconstructs the path from `start` to `goal` using the predecessor
    /// map returned by `dijkstra`. Returns None if unreachable.
    fn reconstruct_path(
        predecessor: &HashMap<String, String>,
        start: &str,
        goal: &str,
    ) -> Option<Vec<String>> {
        if start == goal {
            return Some(vec![start.to_string()]);
        }

        let mut path = vec![goal.to_string()];
        let mut current = goal;

        while let Some(prev) = predecessor.get(current) {
            path.push(prev.clone());
            if prev == start {
                path.reverse();
                return Some(path);
            }
            current = prev;
        }

        None // goal was never reached
    }
}

fn main() {
    let mut graph = Graph::new();
    graph.add_edge("A", "B", 4);
    graph.add_edge("A", "C", 1);
    graph.add_edge("C", "B", 2);
    graph.add_edge("B", "D", 5);
    graph.add_edge("C", "D", 8);

    let (distances, predecessor) = graph.dijkstra("A");

    println!("Shortest distances from A:");
    let mut nodes: Vec<&String> = distances.keys().collect();
    nodes.sort();
    for node in nodes {
        println!("  {} -> {}", node, distances[node]);
    }

    if let Some(path) = Graph::reconstruct_path(&predecessor, "A", "D") {
        println!("Path A -> D: {}", path.join(" -> "));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_shortest_distance() {
        let mut graph = Graph::new();
        graph.add_edge("A", "B", 4);
        graph.add_edge("A", "C", 1);
        graph.add_edge("C", "B", 2);

        let (dist, _) = graph.dijkstra("A");
        // A -> C -> B (1 + 2 = 3) is shorter than A -> B directly (4)
        assert_eq!(dist["B"], 3);
    }

    #[test]
    fn reconstructs_correct_path() {
        let mut graph = Graph::new();
        graph.add_edge("A", "B", 4);
        graph.add_edge("A", "C", 1);
        graph.add_edge("C", "B", 2);

        let (_, predecessor) = graph.dijkstra("A");
        let path = Graph::reconstruct_path(&predecessor, "A", "B").unwrap();
        assert_eq!(path, vec!["A", "C", "B"]);
    }

    #[test]
    fn unreachable_node_returns_none() {
        let mut graph = Graph::new();
        graph.add_edge("A", "B", 1);
        graph.add_edge("C", "D", 1); // disconnected component

        let (_, predecessor) = graph.dijkstra("A");
        assert!(Graph::reconstruct_path(&predecessor, "A", "D").is_none());
    }
}
```

**Expected output:**

```
Shortest distances from A:
  A -> 0
  B -> 3
  C -> 1
  D -> 8
Path A -> D: A -> C -> B -> D
```

---

## 9. Rust-specific design notes (interview gold)

These are the points that turn "I implemented Dijkstra" into "I understand Rust." Each maps to a real language feature an interviewer can probe.

### 9.1 `BinaryHeap` is a max-heap — `Reverse` flips it

`std::collections::BinaryHeap` pops the **largest** element. Dijkstra needs the **smallest**. `std::cmp::Reverse` is a zero-cost newtype whose `Ord` impl inverts the comparison:

```rust
let mut heap: BinaryHeap<Reverse<(u32, String)>> = BinaryHeap::new();
heap.push(Reverse((0, start.to_string())));
while let Some(Reverse((current_dist, node))) = heap.pop() { … }
```

Note the destructuring `Some(Reverse((d, n)))` — pattern matching unwraps the newtype and the tuple in one step, no manual `.0` access.

**Why a tuple `(u32, String)` works:** tuples derive `Ord` **lexicographically** — it compares `u32` first, and only falls back to comparing the `String` on ties. Since the distance is the first element, the heap orders by distance exactly as needed. Getting the field order wrong (`(String, u32)`) would sort by node name instead — a classic bug.

**The alternative — a custom `Ord` impl.** Worth mentioning as the "textbook Rust" version:

```rust
#[derive(Copy, Clone, Eq, PartialEq)]
struct State { cost: u32, position: usize }

impl Ord for State {
    fn cmp(&self, other: &Self) -> Ordering {
        // Flipped: other.cmp(self) makes the max-heap behave as a min-heap
        other.cost.cmp(&self.cost)
            .then_with(|| self.position.cmp(&other.position))
    }
}
impl PartialOrd for State {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> { Some(self.cmp(other)) }
}
```

`Reverse` is cleaner and less error-prone; the custom impl gives finer tie-breaking control. Either is a fine answer — knowing *both* exist is the strong answer.

### 9.2 The stale-entry check replaces `decrease_key`

Textbook Dijkstra calls `decrease_key(v, new_dist)` to update a node's priority in place. `BinaryHeap` **has no such operation** — it can't locate an arbitrary element in `O(log n)` without an auxiliary index map.

So we use **lazy deletion**: push a duplicate entry with the better distance and discard outdated pops.

```rust
if let Some(&best_known) = dist.get(&node) {
    if current_dist > best_known {
        continue;   // stale entry — a better path was already found
    }
}
```

This is a genuine algorithmic trade-off worth naming: the heap grows to `O(E)` instead of `O(V)`, but every operation stays `O(log)` and the code stays simple and allocation-light. It's what `std`'s own `BinaryHeap` docs recommend.

### 9.3 `entry().or_insert_with()` — idiomatic map insertion

```rust
self.adjacency
    .entry(from.to_string())
    .or_insert_with(Vec::new)
    .push((to.to_string(), weight));
```

The Entry API does a **single hash lookup** for the "get or create" pattern. The naive version hashes twice:

```rust
// ❌ two lookups, and fights the borrow checker
if !self.adjacency.contains_key(from) {
    self.adjacency.insert(from.to_string(), Vec::new());
}
self.adjacency.get_mut(from).unwrap().push(…);
```

`or_insert_with(Vec::new)` takes a closure, so the `Vec` is only constructed when actually needed — versus `or_insert(Vec::new())` which eagerly builds one every call. For `Vec::new()` this is free (no allocation until first push), but the habit matters for expensive defaults. In modern Rust you can also write `.or_default()`.

The second line is subtle and worth pointing out:

```rust
self.adjacency.entry(to.to_string()).or_insert_with(Vec::new);
```

This registers the **destination** node even when it has no outgoing edges. Without it, a sink node like `D` never appears as a key, and iterating the graph would silently miss it.

### 9.4 `map_or` for the "no entry means infinity" case

```rust
let is_shorter = dist
    .get(neighbor)
    .map_or(true, |&existing| candidate_dist < existing);
```

Instead of initializing every node to `u32::MAX`, an **absent key means unreachable/infinite**. `map_or(true, …)` reads as: "if there's no known distance, any candidate is an improvement; otherwise compare."

This avoids a pre-pass over all nodes and sidesteps the overflow bug of `u32::MAX + weight`. Good to call out — `dist[u] + w` overflowing when `dist[u]` is a sentinel `MAX` is a real bug in naive implementations.

### 9.5 Ownership: why the `.clone()` calls are here and how to remove them

```rust
dist.insert(neighbor.clone(), candidate_dist);
predecessor.insert(neighbor.clone(), node.clone());
heap.push(Reverse((candidate_dist, neighbor.clone())));
```

`neighbor: &String` is borrowed from `self.adjacency`, but `dist`, `predecessor`, and `heap` all need **owned** `String`s. Since three separate structures each store a copy, cloning is genuinely necessary here — it's not "cloning to silence the borrow checker."

**But it is the main performance cost**, and the fix is the standard one:

```rust
// Intern node names once, then work entirely with indices
struct Graph {
    names: Vec<String>,                    // index -> name
    ids: HashMap<String, usize>,           // name  -> index
    adjacency: Vec<Vec<(usize, u32)>>,     // index -> [(neighbor_idx, weight)]
}

fn dijkstra(&self, start: usize) -> (Vec<u32>, Vec<usize>) {
    let mut dist = vec![u32::MAX; self.adjacency.len()];
    let mut prev = vec![usize::MAX; self.adjacency.len()];
    let mut heap = BinaryHeap::new();
    // … `usize` is Copy — zero clones, zero hashing, cache-friendly Vec indexing
}
```

Saying *"in production I'd intern nodes as `u32` indices into a `Vec`"* — which the code comment already does — signals you know the `String`-keyed version is a readability choice, not the fast path. `Vec` indexing is `O(1)` with no hashing and far better cache locality than `HashMap`.

### 9.6 Returning a `predecessor` map instead of paths

`dijkstra` returns `(dist, predecessor)` rather than building every path. The predecessor map is `O(V)` space and lets you reconstruct **any** path on demand; materializing all paths up front would be `O(V²)` in the worst case. Standard shortest-path-tree design.

### 9.7 `reconstruct_path` is an associated function, not a method

```rust
fn reconstruct_path(predecessor: &HashMap<String, String>, …) -> Option<Vec<String>>
```

No `self` parameter — it operates purely on the returned map and doesn't need the graph. Called as `Graph::reconstruct_path(…)`. Correct API design: don't take `&self` you don't use.

The `Option<Vec<String>>` return encodes "unreachable" **in the type system** rather than returning an empty `Vec` or a sentinel. The caller is forced to handle it.

### 9.8 The termination guarantee

`while let Some(…) = heap.pop()` terminates because a node is only re-pushed when its distance **strictly decreases** (`candidate_dist < existing`). With non-negative weights, distances are bounded below by 0, so there can only be finitely many improvements. No visited-set needed for termination — the strict inequality does the work.

---

## 10. Dijkstra vs other shortest-path algorithms

| Algorithm | Handles | Complexity | Use when |
|---|---|---|---|
| **BFS** | Unweighted (all weights = 1) | `O(V + E)` | All edges cost the same — don't use Dijkstra, BFS is strictly faster |
| **Dijkstra** | Non-negative weights, single source | `O(E log V)` | The default for weighted single-source |
| **Bellman-Ford** | **Negative** weights; detects negative cycles | `O(V·E)` | Negative edges exist, e.g. currency arbitrage |
| **Floyd-Warshall** | All-pairs, negative weights OK | `O(V³)` | Need every pair, small dense graph |
| **Johnson's** | All-pairs, negative weights OK | `O(V·E log V)` | Need every pair, large **sparse** graph |
| **A\*** | Non-negative + an admissible heuristic | `O(E log V)` worst, much faster in practice | Single target and you have a good distance estimate (e.g. Euclidean on a map) |
| **0-1 BFS** | Weights only 0 or 1 | `O(V + E)` | Deque instead of a heap — beats Dijkstra |

**The relationship worth naming:** A\* **is** Dijkstra with a heuristic `h(n)` added to the priority. Set `h(n) = 0` and A\* degenerates exactly into Dijkstra. And Dijkstra on a graph where every weight is 1 explores nodes in the same order as BFS.

---

## 11. Common interview follow-ups

**Q: Why is `BinaryHeap` a max-heap, and how do you get a min-heap?**
Wrap in `std::cmp::Reverse`, or write a custom `Ord` that flips the comparison. `Reverse` is zero-cost and preferred.

**Q: Why can a node be pushed to the heap multiple times?**
Because `BinaryHeap` has no `decrease_key`. We push improved entries and skip stale pops — lazy deletion. The heap holds `O(E)` entries rather than `O(V)`.

**Q: How do you detect that a node is unreachable?**
It never appears as a key in `dist`. In the index-based version, `dist[v] == u32::MAX`.

**Q: How would you find the shortest path to just one target?**
Break out of the loop as soon as you pop the goal node — its distance is final at that moment. Better still, use **A\*** with an admissible heuristic, or **bidirectional Dijkstra** (search from both ends and meet in the middle, roughly halving the explored area).

**Q: What if edges have weights that could overflow?**
Use `u64`/`i64`, or `checked_add` / `saturating_add`. This is a real hazard when using `u32::MAX` as an infinity sentinel — `MAX + weight` wraps. The absent-key approach in this implementation avoids it entirely.

**Q: How do you handle an undirected graph?**
Call `add_edge` twice, once in each direction. The doc comment on `add_edge` says exactly this.

**Q: Can you make this generic over the node type?**
Yes — `Graph<N: Eq + Hash + Clone>` with `adjacency: HashMap<N, Vec<(N, u32)>>`. You'd also want the weight generic over something like `Copy + Ord + Add<Output = W>` + a zero value, which is roughly what the `petgraph` crate does.

**Q: Which crate would you use in production?**
**`petgraph`** — mature, well-tested graph library with `petgraph::algo::dijkstra`, plus A\*, Bellman-Ford, and Floyd-Warshall. Hand-rolling is for interviews and for cases where you need a specialized representation.

**Q: How would you parallelize it?**
Dijkstra is inherently sequential (the greedy step needs the global minimum). Parallel alternatives: **Δ-stepping**, which buckets nodes by distance range and processes each bucket in parallel, or run independent Dijkstras from many sources concurrently with `rayon` for all-pairs.

---

## Quick Revision Sheet

| Concept | The one thing to remember |
|---|---|
| **What it does** | Single-source shortest paths, non-negative weights only |
| **Core operation** | Relaxation: `if dist[u] + w < dist[v] { dist[v] = dist[u] + w }` |
| **Greedy invariant** | The unvisited node with the smallest tentative distance is already final |
| **Data structure** | Min-priority-queue → `BinaryHeap<Reverse<(cost, node)>>` in Rust |
| **Complexity** | `O((V + E) log V)` with a binary heap; `O(V²)` with an array (better if dense) |
| **Stale entries** | No `decrease_key` → push duplicates, skip pops where `popped > dist[node]` |
| **Negative weights** | ❌ Breaks the proof — use Bellman-Ford |
| **Path reconstruction** | Keep a `predecessor` map, walk backwards from goal, reverse |
| **`Reverse`** | Turns Rust's max-heap into a min-heap at zero cost |
| **Tuple ordering** | Tuples compare lexicographically → put `cost` first |
| **Perf upgrade** | Intern nodes as `usize` indices into a `Vec`; kills hashing + cloning |
| **Relation to A\*** | A\* = Dijkstra + heuristic; `h(n) = 0` gives you Dijkstra back |
| **Relation to BFS** | Dijkstra with all weights = 1 explores in BFS order (but BFS is faster) |
| **Production crate** | `petgraph` |
