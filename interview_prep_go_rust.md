# Backend Engineer Interview Prep — Go & Rust

**For: Amit Kumar Yadav**
**Target roles:** Backend Engineer (Go/Rust), Blockchain Engineer, Protocol Engineer

---

## How to use this guide

1. Read each question.
2. Try to answer out loud before reading the answer.
3. If you can't explain it in 60 seconds, mark it for re-study.
4. Aim to know all **Tier 1** cold. **Tier 2** with confidence. **Tier 3** is bonus.

---

# PART 1 — GO LANGUAGE CONCEPTS

## TIER 1 — Asked in ~80% of Go interviews

### Q1. What's the difference between a goroutine and an OS thread?

**Answer:**
A goroutine is a lightweight, user-space thread managed by the Go runtime, not the OS.

- OS threads typically have 1–2 MB stack and are heavy to create. Context switching goes through the kernel.
- Goroutines start with a 2 KB stack that grows/shrinks dynamically. The Go runtime multiplexes many goroutines (M) onto fewer OS threads (N) — the M:N scheduling model.
- You can spawn millions of goroutines on one machine. Spawning a million OS threads would crash the system.

The Go scheduler uses a model called **GMP**: G = Goroutine, M = Machine (OS thread), P = Processor (logical context, count set by GOMAXPROCS). The scheduler does work-stealing across P's, so blocked goroutines don't starve the others.

---

### Q2. Channels — unbuffered vs. buffered, and what happens when you close one?

**Answer:**
- **Unbuffered channel** (`make(chan int)`): send and receive happen synchronously. Sender blocks until a receiver is ready. Used for synchronization.
- **Buffered channel** (`make(chan int, 5)`): send blocks only when the buffer is full. Receive blocks only when the buffer is empty. Used for queueing.

**Closing rules:**
- Sending on a closed channel → **panic**.
- Receiving from a closed channel → returns the zero value, and the second return value (`v, ok := <-ch`) is `false`.
- Closing an already-closed channel → **panic**.
- Only the sender should close a channel, never the receiver.
- You can `range` over a channel; the loop exits when the channel is closed.

---

### Q3. sync.Mutex vs sync.RWMutex — when to use which?

**Answer:**
- `sync.Mutex`: only one goroutine can hold the lock at a time (exclusive). Use when reads and writes are roughly equal frequency.
- `sync.RWMutex`: multiple readers can hold `RLock()` simultaneously, but `Lock()` (writer) is exclusive. Use when reads vastly outnumber writes — e.g., a cache that's read 1000x for every write.

**Gotcha:** RWMutex has higher overhead than Mutex. If reads and writes are similar in frequency, plain Mutex is faster.

---

### Q4. What is the `context` package and why do we use it?

**Answer:**
`context` is Go's standard mechanism for **cancellation, deadlines, and request-scoped values** across goroutine boundaries.

- `context.Background()` — root context, never cancelled.
- `context.WithCancel(parent)` — returns a child context and a `cancel()` function. Calling `cancel()` propagates cancellation to all derived contexts.
- `context.WithTimeout(parent, 5*time.Second)` — auto-cancels after the duration.
- `context.WithDeadline(parent, time)` — auto-cancels at the deadline.

**Why it matters:** propagate cancellation through call chains (HTTP handler → DB query → external API). When client disconnects, you stop all downstream work.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()
    result, err := db.QueryContext(ctx, "...")
    // ...
}
```

---

### Q5. How does Go handle errors? What's `errors.Is`, `errors.As`, and `%w`?

**Answer:**
Go uses explicit error values, not exceptions. Every function that can fail returns `(result, error)`.

**Error wrapping (Go 1.13+):**
```go
if err != nil {
    return fmt.Errorf("failed to fetch user: %w", err)
}
```
`%w` wraps the error so it can be unwrapped later.

- `errors.Is(err, target)` — checks if `err` (or anything it wraps) is the same as `target`. Used for sentinel errors like `io.EOF`, `sql.ErrNoRows`.
- `errors.As(err, &target)` — checks if `err` (or anything it wraps) can be assigned to `target`. Used for typed errors:
```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

---

### Q6. What does `defer` do and what's the gotcha with arguments?

**Answer:**
`defer` schedules a function to run when the enclosing function returns. Multiple defers execute in **LIFO order** (last in, first out).

**Gotcha — arguments are evaluated at defer time, not execution time:**
```go
func main() {
    i := 10
    defer fmt.Println(i)  // prints 10, NOT 20
    i = 20
}
```

**Gotcha — defer in a loop:**
```go
for i := 0; i < 1000; i++ {
    f, _ := os.Open(files[i])
    defer f.Close()  // BAD: defers don't run until function returns
}
```
Defers stack up; the file handles aren't closed until the loop's enclosing function exits. Fix: move the loop body into its own function.

---

### Q7. Slices — what is the slice header, and what's the gotcha with `append`?

**Answer:**
A slice is a struct with three fields: pointer to underlying array, length, capacity.

```go
type sliceHeader struct {
    Data unsafe.Pointer
    Len  int
    Cap  int
}
```

When you `append` to a slice:
- If `len < cap`, append writes in-place and returns a slice with `len + 1`.
- If `len == cap`, Go allocates a new larger backing array (typically 2x), copies data, returns a new slice pointing to it.

**Gotcha — slice aliasing:**
```go
a := []int{1, 2, 3, 4, 5}
b := a[:3]            // b shares the same backing array as a
b = append(b, 99)     // if cap allows, this overwrites a[3]!
fmt.Println(a)        // [1 2 3 99 5] — surprise!
```

---

### Q8. Why is a Go map not safe for concurrent use?

**Answer:**
Go's built-in `map` is not designed for concurrent reads + writes. If two goroutines write simultaneously, the runtime detects this and **panics** with "concurrent map writes" (this is a fatal error, not a recoverable one).

**Solutions:**
- `sync.Mutex` around the map.
- `sync.Map` — built for the pattern "write once, read many" or "many goroutines touching mostly disjoint keys."
- A channel-based goroutine that owns the map.

**Multiple concurrent reads (no writes) are safe** — but the moment you add a write, you need synchronization.

---

## TIER 2 — Asked in ~50% of Go interviews

### Q9. How do Go interfaces work? What's implicit satisfaction?

**Answer:**
A type **implements** an interface just by having all its methods — no `implements` keyword needed.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type File struct{}
func (f *File) Read(p []byte) (int, error) { return 0, nil }
// File now implements Reader automatically.
```

**The empty interface `interface{}` (Go 1.18+ alias: `any`)** matches any type. Used for generic containers, JSON parsing, etc. Type-assert to get the concrete value:
```go
var x interface{} = 42
if n, ok := x.(int); ok {
    fmt.Println(n)
}
```

**Type switch:**
```go
switch v := x.(type) {
case int: ...
case string: ...
default: ...
}
```

---

### Q10. Pointer receivers vs. value receivers — when to use which?

**Answer:**
- **Pointer receiver** (`func (t *T) Foo()`): can modify the receiver. Avoids copying large structs.
- **Value receiver** (`func (t T) Foo()`): operates on a copy. Safe for concurrent use.

**Rules of thumb:**
1. If the method modifies the receiver → pointer.
2. If the struct is large → pointer (avoid copying).
3. If the type contains `sync.Mutex` or similar → pointer (mutexes shouldn't be copied).
4. Be consistent across a type — don't mix pointer and value receivers on the same type.

---

### Q11. Why does Go have garbage collection? What kind of GC does Go use?

**Answer:**
Go uses a **concurrent, tri-color mark-and-sweep, non-generational** garbage collector.

- **Concurrent:** GC runs alongside your goroutines; STW (stop-the-world) pauses are typically under 1 ms.
- **Tri-color marking:** objects are colored white (unreachable), gray (reachable, scanning), or black (reachable, scanned).
- **Non-generational:** unlike Java's G1, Go doesn't separate young vs. old objects. This simplifies the runtime but means very allocation-heavy programs can have higher GC overhead.

**Tuning:** `GOGC` env var (default 100) controls how aggressively GC runs. `GOMEMLIMIT` (Go 1.19+) caps total heap size.

---

### Q12. What is the Go memory model? Why do you need sync, not just atomics?

**Answer:**
The Go memory model defines **happens-before** relationships — when a write in one goroutine is guaranteed to be visible to a read in another.

Without synchronization, the compiler and CPU can reorder reads/writes for performance. Two goroutines touching the same variable without sync can see stale values — even if writes are "atomic" at the hardware level.

```go
var ready bool
var data int

// goroutine 1
data = 42
ready = true   // might be reordered before data = 42!

// goroutine 2
if ready {
    fmt.Println(data)  // might print 0!
}
```

Use `sync/atomic`, `sync.Mutex`, channels, or `sync.Once` to establish happens-before.

---

## TIER 3 — Asked in 20% of senior interviews

### Q13. Walk through the GMP scheduler.

**Answer:**
- **G** (Goroutine): your `go func()` calls.
- **M** (Machine): an OS thread.
- **P** (Processor): a logical execution context. There are `GOMAXPROCS` of them.

Each P has a local run queue of G's. M needs a P to execute G's. When a goroutine makes a blocking syscall, the M detaches from P, and another M picks up that P to keep work going.

**Work stealing:** if a P's local queue is empty, it steals half of another P's queue. Keeps cores busy.

---

### Q14. What's the cost of `cgo`?

**Answer:**
Each call from Go into C (or vice versa) costs **~100–200 ns of overhead** plus context-switch-like work. Reasons:
- Goroutines run on small stacks; C code expects normal OS thread stacks. Switching stacks is expensive.
- The Go scheduler can't preempt code running in C.
- Race detector and GC don't see into C code.

Avoid frequent small cgo calls in hot paths. Batch the work.

---

## COMMON GO TRICK QUESTIONS

### TQ1. What does this print?

```go
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()
}
time.Sleep(time.Second)
```

**Answer:**
- **Go < 1.22:** prints `3 3 3` (or some permutation). The closure captures `i` by reference; all goroutines see the final value.
- **Go ≥ 1.22:** prints `0 1 2` in some order. The loop variable was redefined as per-iteration scope.

**Fix for older Go:**
```go
for i := 0; i < 3; i++ {
    i := i  // shadow with a per-iteration copy
    go func() { fmt.Println(i) }()
}
```

---

### TQ2. What does this print?

```go
func main() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
}
```

**Answer:** Prints `2 1 0`. Defers execute LIFO; arguments evaluated at defer time.

---

### TQ3. Why does this leak goroutines?

```go
func leaky() {
    ch := make(chan int)
    go func() {
        ch <- 42  // blocks forever because no one receives
    }()
}
```

**Answer:** The goroutine blocks on the unbuffered send. The channel is never received from, the goroutine never exits, and the GC can't collect it (the goroutine itself holds a reference to `ch`).

---

# PART 2 — RUST LANGUAGE CONCEPTS

## TIER 1 — Asked in ~90% of Rust interviews

### Q1. Explain ownership in Rust.

**Answer:**
Rust has three core ownership rules:
1. Each value has a single **owner**.
2. When the owner goes out of scope, the value is dropped (memory freed deterministically — no GC).
3. Ownership can be **moved** to a new owner, but the old owner becomes invalid.

```rust
let s1 = String::from("hello");
let s2 = s1;            // ownership moves from s1 to s2
println!("{}", s1);     // ERROR: s1 is no longer valid
```

This is what gives Rust **memory safety without garbage collection**. The compiler enforces these rules at compile time.

For types that are cheap to copy (`i32`, `bool`, `char`, etc.), the `Copy` trait is implemented; assignment copies instead of moves. `String`, `Vec<T>` are NOT `Copy` because they own heap data.

---

### Q2. Explain borrowing and the borrow checker rules.

**Answer:**
You can borrow a value instead of taking ownership:
- `&T` — immutable (shared) reference. Many allowed at once.
- `&mut T` — mutable (exclusive) reference. Only one at a time.

The rule: **at any time, you can have either one mutable reference OR any number of immutable references — but not both.**

```rust
let mut v = vec![1, 2, 3];
let r1 = &v;        // immutable borrow
let r2 = &v;        // OK — multiple immutable borrows allowed
let r3 = &mut v;    // ERROR: cannot borrow as mutable while immutable borrows exist
println!("{}", r1);
```

**Why?** This rule eliminates data races at compile time. You can't have one thread reading while another writes, because the borrow checker won't let you take both kinds of references simultaneously.

---

### Q3. What are lifetimes? When do you need to annotate them?

**Answer:**
A lifetime is a compile-time annotation that tells the compiler **how long a reference is valid**.

Most of the time, the compiler infers lifetimes (lifetime elision). You need explicit lifetimes when:
- A function returns a reference, and the compiler can't infer which input it borrows from:
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```
- A struct holds a reference:
```rust
struct Wrapper<'a> {
    inner: &'a str,
}
```

`'static` is a special lifetime — the reference lives for the entire program. String literals are `&'static str`.

---

### Q4. What's the difference between `String` and `&str`?

**Answer:**
- `String`: a heap-allocated, owned, growable UTF-8 string. Has a pointer, length, and capacity.
- `&str`: a string slice. A "fat pointer" — pointer + length only, no ownership, no capacity. Points into a `String` or a static literal.

```rust
let owned: String = String::from("hello");
let borrowed: &str = &owned;           // borrowing the String as a slice
let literal: &'static str = "hello";   // string literal — &str baked into binary
```

**Rule of thumb:**
- Function parameters: take `&str` (more flexible — accepts both literals and `&String`).
- Returned/stored values: return `String` if you allocate, `&str` if you borrow.

---

### Q5. Explain `Option<T>` and `Result<T, E>`. What's the `?` operator?

**Answer:**
- `Option<T>` = `Some(T)` or `None`. Used when a value might be absent (replaces null).
- `Result<T, E>` = `Ok(T)` or `Err(E)`. Used when an operation might fail.

The `?` operator unwraps the success case or returns the error from the current function:
```rust
fn read_username() -> Result<String, io::Error> {
    let mut f = File::open("name.txt")?;  // returns Err if open fails
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

**Important:** `?` calls `From::from` on the error, allowing automatic conversion between error types. This is why `?` works across different error types if you have a `From` impl.

**Don't use `unwrap()` in production code** — it panics on `None`/`Err`. Use `?`, `match`, or `unwrap_or_else`.

---

### Q6. Traits — what are trait objects, and why do we need `Box<dyn Trait>`?

**Answer:**
A trait defines shared behavior. You can implement it for multiple types:
```rust
trait Animal {
    fn speak(&self);
}
impl Animal for Dog { fn speak(&self) { println!("woof"); } }
impl Animal for Cat { fn speak(&self) { println!("meow"); } }
```

**Static dispatch** (monomorphization) — compiler generates a separate function per concrete type:
```rust
fn make_speak<T: Animal>(a: T) { a.speak(); }
```

**Dynamic dispatch** via trait objects — the runtime looks up which `speak` to call:
```rust
fn make_speak(a: &dyn Animal) { a.speak(); }
```

Why `Box<dyn Trait>` instead of `dyn Trait`? Because `dyn Trait` is unsized (the compiler doesn't know how big it is at compile time). You need a pointer to it: `&dyn Trait`, `Box<dyn Trait>`, `Arc<dyn Trait>`.

---

### Q7. `Rc<T>` vs `Arc<T>` vs `RefCell<T>` vs `Mutex<T>` — when to use which?

**Answer:**

| Type | Threading | Ownership | Mutability |
|---|---|---|---|
| `Rc<T>` | Single-threaded only | Shared (ref counted) | Immutable |
| `Arc<T>` | Multi-threaded | Shared (atomic ref counted) | Immutable |
| `RefCell<T>` | Single-threaded only | Single owner | Interior mutability (runtime borrow checking) |
| `Mutex<T>` | Multi-threaded | Single owner | Interior mutability (lock-based) |
| `RwLock<T>` | Multi-threaded | Single owner | Interior mutability (many readers OR one writer) |

**Common combinations:**
- `Rc<RefCell<T>>` — single-threaded shared mutable state (e.g., a graph node).
- `Arc<Mutex<T>>` — multi-threaded shared mutable state. The bread-and-butter of concurrent Rust.
- `Arc<RwLock<T>>` — multi-threaded shared mutable state with read-heavy workload.

**Why can't `Rc` cross threads?** `Rc`'s ref counter is not atomic. The compiler enforces this by marking `Rc` as `!Send`. `Arc` uses atomic increments — safe across threads.

---

### Q8. What is interior mutability? Why does it exist?

**Answer:**
Interior mutability is the pattern of **mutating data through a shared (immutable) reference** — done safely.

Normally `&self` means you can't mutate. But sometimes you need to:
- Caching computed results.
- Reference counting (`Rc` increments a counter on `clone()` which takes `&self`).
- Locks (`Mutex::lock(&self)` returns mutable access through a guard).

Types providing interior mutability:
- `Cell<T>` — for `Copy` types. Get/set the value, no references inside.
- `RefCell<T>` — runtime-checked borrow rules. Panics if you violate them at runtime.
- `Mutex<T>`, `RwLock<T>` — for concurrent code.

---

## TIER 2 — Asked in ~60% of Rust interviews

### Q9. How do iterators work? Why are they "zero-cost"?

**Answer:**
Iterators in Rust are lazy — calling `.iter().map(...).filter(...)` doesn't compute anything until you consume them with `.collect()`, `.sum()`, `for` loop, etc.

**Zero-cost** means: the compiler inlines and optimizes iterator chains so aggressively that they compile down to the same assembly as a hand-written `for` loop. No vtables, no heap allocations, no function call overhead.

```rust
let sum: i32 = (1..=100).filter(|x| x % 2 == 0).sum();
// Compiles to a tight loop equivalent to:
// let mut sum = 0; for i in 1..=100 { if i % 2 == 0 { sum += i; } }
```

Key traits:
- `Iterator` — has `next() -> Option<Self::Item>`.
- `IntoIterator` — types convertible into iterators (e.g., `Vec<T>`).
- `FromIterator` — types buildable from iterators (used by `collect()`).

---

### Q10. Closures — what are `Fn`, `FnMut`, `FnOnce`?

**Answer:**
Closures capture variables from their environment. The three traits describe how:

- `Fn` — captures by reference. Can be called many times. No mutation of captured vars.
- `FnMut` — captures by mutable reference. Can mutate captured vars.
- `FnOnce` — captures by move. Can only be called once (consumes captured vars).

The compiler infers which trait based on how the closure body uses its captures. `move` keyword forces capture by value:

```rust
let s = String::from("hi");
let closure = move || println!("{}", s);  // takes ownership of s
```

Use `move` when spawning threads (the closure outlives the calling scope).

---

### Q11. Async Rust — what's a `Future`? How does `.await` work?

**Answer:**
A `Future` is a value representing a computation that may not be ready yet. The `Future` trait:
```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

`Poll` is either `Ready(value)` or `Pending`.

**An async function returns `impl Future`:**
```rust
async fn fetch() -> String { ... }
// equivalent to:
fn fetch() -> impl Future<Output = String> { ... }
```

`.await` polls the future. If `Pending`, it yields control back to the runtime (e.g., Tokio), which can run other futures. When the future is ready (e.g., I/O complete, timer fired), it's polled again.

**Key insight:** futures are inert. They do nothing until polled by a runtime. This is why you need `tokio::spawn` or `block_on` to actually run an async function.

---

### Q12. Tokio's `tokio::sync::Mutex` vs `std::sync::Mutex` — why does it matter?

**Answer:**
**Never hold a `std::sync::Mutex` across an `.await`.**

When you `.await` while holding `std::sync::Mutex`, the lock holds across the suspension. If the runtime schedules a different task on the same OS thread, and that task tries to acquire the same mutex — **deadlock**.

`tokio::sync::Mutex` is async-aware: `.lock().await` yields to the runtime instead of blocking. Safe to hold across `.await`.

**Rule of thumb:**
- Lock guards that don't cross `.await` boundaries: `std::sync::Mutex` (faster).
- Lock guards that must cross `.await`: `tokio::sync::Mutex`.

---

### Q13. `Send` and `Sync` — what do they mean?

**Answer:**
- `Send`: a type is `Send` if it's safe to **transfer ownership** to another thread. (`Arc<T>` is Send; `Rc<T>` is not.)
- `Sync`: a type is `Sync` if it's safe to **share a reference** across threads (`&T` can be sent between threads). (`Mutex<T>` is Sync; `Cell<T>` is not.)

Almost all types are auto-derived `Send + Sync` if their fields are. Exceptions:
- `Rc<T>` — `!Send, !Sync` (non-atomic ref counting).
- `RefCell<T>` — `!Sync` (single-threaded borrow checking).
- Raw pointers `*const T`, `*mut T` — `!Send, !Sync` by default.

When you `tokio::spawn(async move { ... })`, the future must be `Send` because the runtime may move it across threads.

---

## TIER 3 — Asked in 30% of senior Rust interviews

### Q14. When is `unsafe` Rust actually needed?

**Answer:**
- **FFI** — calling C functions. The compiler can't verify C's safety.
- **Raw pointers** — needed for some data structures (linked lists, lock-free structures).
- **Manual memory layout** — `repr(C)` structs, custom allocators.
- **Performance-critical hot paths** — when you can prove safety yourself but the compiler can't.

`unsafe` doesn't disable the borrow checker. It just unlocks a few specific superpowers (deref raw pointers, call unsafe fns, access mutable statics, implement unsafe traits). You're still responsible for upholding Rust's invariants.

---

### Q15. What's the difference between `impl Trait` and `dyn Trait`?

**Answer:**
- `impl Trait` (in return position) — static dispatch. Compiler picks a single concrete type at compile time. Zero runtime cost. But the type is fixed per call site.
- `dyn Trait` — dynamic dispatch via vtable. Choice happens at runtime. Heterogeneous collections (`Vec<Box<dyn Trait>>`).

```rust
fn make_iter() -> impl Iterator<Item = i32> {     // one specific iterator type
    (1..10).filter(|x| x % 2 == 0)
}

fn make_iter() -> Box<dyn Iterator<Item = i32>> { // any iterator, runtime chosen
    Box::new((1..10).filter(|x| x % 2 == 0))
}
```

---

## COMMON RUST TRICK QUESTIONS

### TQ1. Why doesn't this compile?

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("{}", first);
```

**Answer:** `&v[0]` is an immutable borrow of `v`. `v.push(4)` requires a mutable borrow. Rust forbids holding both simultaneously. Also: if `push` reallocates, `first` would dangle — the borrow checker prevents this entire class of bug.

**Fix:** read `first` before `push`, or clone the value: `let first = v[0]; v.push(4); println!("{}", first);`

---

### TQ2. Why isn't `String` `Copy`?

**Answer:** `String` owns heap-allocated data (a pointer, length, capacity). Copying it would mean either:
- A shallow copy → two strings pointing to the same heap memory → double-free on drop.
- A deep copy → expensive, surprising performance cliff.

Rust forces you to opt-in via `.clone()` so it's explicit and visible. If you want bitwise copy semantics, use `Copy` types like `i32`, `bool`, `&str`.

---

### TQ3. Why can't `Rc<T>` be sent across threads?

**Answer:** `Rc<T>`'s reference counter is a plain `usize`. If two threads both call `clone()` simultaneously, the counter could miss an increment, leading to use-after-free.

`Rc<T>` is marked `!Send` so the compiler rejects sending it across threads. Use `Arc<T>` which uses atomic increments.

---

### TQ4. What's the difference between these two?

```rust
fn foo<T: Display>(x: T) { ... }
fn foo(x: &dyn Display) { ... }
```

**Answer:**
- First is generic: monomorphized at compile time. One compiled version per `T` you call it with. Zero runtime cost but larger binary.
- Second uses a trait object: one compiled version, vtable lookup at runtime. Smaller binary but slight runtime cost.

---

# PART 3 — BLOCKCHAIN / PROTOCOL-SPECIFIC QUESTIONS

(For Web3, blockchain, and protocol engineer roles)

### Q1. What is PolyBFT? How does it differ from PBFT?

**Answer:**
PolyBFT is Polygon's BFT consensus engine used in Polygon Edge / Supernets / Hydragon. It's based on IBFT 2.0 with modifications:

- **PBFT** (classical Castro & Liskov): 3-phase commit (pre-prepare, prepare, commit). Requires `3F+1` validators for `F` faults.
- **IBFT** (Istanbul BFT): simpler 2-phase commit, used in Quorum / Polygon Edge.
- **PolyBFT** extends IBFT with: BLS signature aggregation (smaller commit messages), checkpointing to Ethereum mainnet, slashing for double-signing.

Quorum requirement: `ceil(2N/3)` — i.e., more than 2/3 of validators by voting power.

---

### Q2. What is double-signing and how do you detect it?

**Answer:**
Double-signing = a validator signs two different blocks at the same height/round. Indicates either malicious behavior or a critical bug (e.g., running two node instances with the same key).

Detection:
1. Maintain a record of all signed proposals per height/round.
2. When a new signed proposal arrives, check if the same validator has signed a different one at the same height/round.
3. If yes, package both signatures as evidence and submit a system transaction to the slashing contract.

The slashing contract then deducts a portion of the validator's stake.

---

### Q3. Walk through what EIPs you implemented in the Hydragon EVM upgrade.

**Answer:**

**EIP-4399 (PREVRANDAO):** Replaced the `DIFFICULTY` opcode (0x44). Post-merge, Ethereum has no proof-of-work difficulty. The same opcode now returns the previous block's RANDAO mix — a pseudo-random value contributed by validators. Smart contracts that relied on `DIFFICULTY` for randomness now get something better-distributed (though still not cryptographically secure).

**EIP-3855 (PUSH0):** Added a new opcode `PUSH0` (0x5F) that pushes the constant `0` onto the stack. Costs 2 gas. Previously, code had to use `PUSH1 0x00` (3 gas, 2 bytes). PUSH0 is 1 byte. Small but compounds across deployed contracts.

**EIP-3860 (Initcode size limit and metering):**
- Limits contract initcode size to 49152 bytes (2 × max code size).
- Adds a gas cost of 2 gas per 32-byte word of initcode.
- Prevents DoS via massive initcode payloads.

**EIP-3651 (Warm COINBASE):** Pre-Shanghai, the `COINBASE` address (block builder) was "cold" — first access cost 2600 gas. Now it's "warm" from the start — 100 gas. Makes MEV-related interactions cheaper.

---

### Q4. Substrate-specific: what's `T: Config` and why are pallets generic?

**Answer:**
Substrate's FRAME pallets are generic over a `Config` trait. This trait declares the types and constants the pallet needs:

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<...>;
    type WeightInfo: WeightInfo;
    type MaxValidators: Get<u32>;
}
```

The runtime (the binary that becomes a blockchain) implements `Config` for each pallet it uses, choosing the concrete types. This makes pallets reusable across different chains with different parameters — same code, different MaxValidators, different reward formulas, etc.

---

### Q5. How does a matching engine work? Walk me through processing a limit order.

**Answer:**

**Data structure:** two sorted maps for the order book:
- `bids: BTreeMap<Price, FifoQueue<Order>>` — sorted descending by price (best bid first).
- `asks: BTreeMap<Price, FifoQueue<Order>>` — sorted ascending by price (best ask first).

**Processing a new buy limit order at price P:**
1. Look at the best ask (lowest sell price).
2. If best_ask ≤ P, match: dequeue the head of the FIFO at best_ask, generate a trade for the lesser of the two quantities, decrement both. If the resting order is fully filled, remove it.
3. Repeat until either the incoming order is fully filled OR the best ask > P.
4. If any quantity remains on the incoming order, insert it at price P in the bids book (FIFO at that price level).

**Price-time priority** = best price first; within same price, earliest order first.

**Throughput tricks:**
- Single-threaded engine with channels for input/output → no locks, predictable latency.
- Pre-allocate FIFO queues.
- Avoid heap allocation in hot path (use object pools).
- Benchmark with `criterion` crate.

---

# PART 4 — BEHAVIORAL / HR QUESTIONS

### B1. Tell me about a difficult bug you solved.

**Story: Quorum bug in HasQuorum (PR #125)**

**Situation:** Hydragon's chain was occasionally stalling block production on testnets where the validator set dropped below 4 nodes.

**Task:** Identify why valid blocks were being rejected and restore liveness.

**Action:** I traced the IBFT consensus voting logic and found a special case in `HasQuorum`: when the validator set was smaller than 4, the code required 100% voting power to reach quorum. So if even one validator was offline, no blocks could be committed. Compared against the core IBFT spec — it expected the standard `ceil(2N/3)` formula in all cases. The special case was a defensive misimplementation. I removed it, ran the chain through testnet scenarios with various validator counts, and confirmed liveness was restored without breaking larger-set behavior.

**Result:** PR merged, block production stabilized across small testnets. Fix aligned the implementation with the upstream IBFT consensus spec.

---

### B2. Why are you leaving your current company?

**Answer:**
"I've spent four productive years at Antier across multiple chains — Substrate-based L1s, Sui Move contracts, and most recently a year of protocol work on Hydragon. I'm now looking for an environment where I can go deeper on Rust at scale, distributed systems, and ideally a single product I can grow with over time. Services work has given me breadth; I'm looking to build depth and longer-term ownership."

(Don't mention layoff unless directly asked. If asked: "Antier had a recent reduction. I'd been planning to explore options anyway — the timing just moved up.")

---

### B3. Tell me about a time you had to learn something new fast.

**Story:** When I moved from Substrate (pure Rust) to Hydragon (Go + Geth fork), I had to ramp on Go and on Geth's codebase in about 4 weeks before contributing real PRs. I focused on the consensus and EVM layers since those were my known territory in Rust/Substrate, then expanded outward to libp2p (which I'd touched at a basic level) and Geth's transaction-pool internals. The first PR I shipped was the EVM hard-fork upgrade — I learned the codebase by mapping each EIP requirement to its location in the Go-Ethereum execution layer.

---

### B4. Tell me about ownership of a project.

**Story (Sui Move contracts at HYDRO):** I owned the full smart contract suite for HYDRO token economy — 5 contracts (Token, Migration, Vesting, Staking, Rewards Distribution) deployed to Sui mainnet. End-to-end: design with product, implementation in Sui Move, internal review, deployment, post-deployment support. The migration contract was particularly sensitive — legacy token holders minting the new token with replay-safe accounting. Coordinating the migration window with stakeholders required clear documentation and a phased rollout.

---

# PART 5 — QUICK FLASH-CARD STYLE REVISION

Use this for the night before an interview. 30-second answers.

**Go:**
- Goroutine: lightweight thread, 2KB stack, M:N scheduled.
- Channel: typed conduit for goroutine communication. Unbuffered = sync, buffered = queue.
- Defer: LIFO order, args evaluated at defer time, runs on function return.
- Context: cancellation/timeout/deadlines propagated through goroutines.
- Error wrapping: `fmt.Errorf("%w", err)`, unwrap with `errors.Is` / `errors.As`.
- Slice header: pointer + len + cap. Append reallocates when len == cap.
- Map: not safe for concurrent writes (panics). Use sync.Mutex or sync.Map.
- Interface: implicitly satisfied. Empty interface = `any`. Type assert with `v, ok := x.(T)`.

**Rust:**
- Ownership: one owner, dropped at scope end, move on assignment.
- Borrow: `&T` shared (many), `&mut T` exclusive (one). Never both.
- Lifetime: how long a reference is valid. Usually inferred.
- `String` vs `&str`: owned heap vs. borrowed slice.
- `Option<T>` / `Result<T, E>`: nullability and fallibility. Use `?` to propagate.
- `Box<T>`: heap allocation. `Rc<T>`: single-threaded shared. `Arc<T>`: multi-threaded shared.
- `RefCell<T>`: runtime borrow checking. `Mutex<T>`: thread-safe interior mutability.
- `Send`: can move to another thread. `Sync`: can share &T across threads.
- Iterators: lazy, zero-cost when chained, consumed by `collect`/`sum`/etc.
- Async: `Future` is inert, polled by runtime, yields on `.await`.

**Blockchain:**
- PolyBFT: IBFT-based, BLS aggregation, quorum = ceil(2N/3).
- Double-signing: validator signs two blocks at same height → slashing.
- EIP-4399: DIFFICULTY → PREVRANDAO post-merge.
- EIP-3855: PUSH0 opcode (1 byte, 2 gas).
- EIP-3860: 49152-byte initcode limit + gas per word.
- EIP-3651: warm COINBASE (100 gas instead of 2600).
- Substrate `T: Config`: pallet generic over runtime-provided types.
- Order book: BTreeMap<Price, FIFO<Order>> per side. Match best-ask vs incoming buy.

---

**End of guide. Total: ~30 questions across Go, Rust, Blockchain, Behavioral.**

**Goal:** read once front-to-back, then drill weak areas. By interview day, every Tier 1 should feel automatic.
