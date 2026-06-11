# 🦀 Rust + Backend Interview Preparation
### Mid-Level Backend / Systems Developer
> Covers: Rust Core | Backend Systems | Design Patterns | Kafka | REST & gRPC | Docker & Kubernetes

---

# TABLE OF CONTENTS

## Rust Core
1. [Ownership & Borrowing](#1-ownership--borrowing)
2. [Generics](#2-generics)
3. [Traits](#3-traits)
4. [Lifetimes](#4-lifetimes)
5. [&str vs String](#5-str-vs-string)
6. [Smart Pointers](#6-smart-pointers)
7. [Tokio & Rayon](#7-tokio--rayon)
8. [Pin & Futures](#8-pin--futures)
9. [Sync vs Async](#9-sync-vs-async)
10. [Multithreading & Channels](#10-multithreading--channels)
11. [Iterators](#11-iterators)
12. [Error Handling](#12-error-handling)
13. [Closures & Function Pointers](#13-closures--function-pointers)
14. [Enums & Pattern Matching](#14-enums--pattern-matching)
15. [Collections](#15-collections)

## Design Patterns
16. [Builder Pattern](#16-builder-pattern)
17. [Newtype Pattern](#17-newtype-pattern)
18. [Strategy Pattern](#18-strategy-pattern)
19. [Observer Pattern](#19-observer-pattern)
20. [State Pattern](#20-state-pattern)
21. [Singleton Pattern](#21-singleton-pattern)
22. [RAII Pattern](#22-raii-pattern)

## Backend Systems
23. [Kafka & Message Queues](#23-kafka--message-queues)
24. [REST APIs & gRPC](#24-rest-apis--grpc)
25. [Docker & Kubernetes](#25-docker--kubernetes)

---

# RUST CORE

---

## 1. Ownership & Borrowing

**Q: What is Ownership and Borrowing in Rust?**

Ownership and borrowing are Rust's way of managing memory without a garbage collector — and once it clicks, it's one of the things that makes Rust really elegant.

### Ownership — Three Simple Rules

- Every value in Rust has **one owner** — one variable that owns it.
- When that owner goes out of scope, the value is **automatically dropped** (freed from memory).
- There can only be **one owner at a time** — if you assign a value to another variable, ownership *moves* to the new one, and the old variable is no longer valid.

```rust
let s1 = String::from("hello");
let s2 = s1;  // ownership moves to s2
// println!("{}", s1);  // ERROR: s1 is no longer valid
```

### Borrowing — Two Kinds

Borrowing lets other parts of your code use a value without giving up ownership. Instead of moving the value, you pass a reference using the `&` symbol.

- **Shared borrow (`&T`)** — multiple parts of code can read the value at the same time, but nobody can modify it.
- **Mutable borrow (`&mut T`)** — one part of code can read and modify the value, but while that's happening, nobody else can access it at all.

```rust
let mut s = String::from("hello");

let r1 = &s;      // shared borrow — OK
let r2 = &s;      // another shared borrow — OK
// let r3 = &mut s; // ERROR: can't have mutable + shared borrow at same time

let r3 = &mut s;  // mutable borrow — OK when no other borrows exist
r3.push_str(", world");
```

> 💡 **Key Point:** This eliminates data races, dangling pointers, and use-after-free bugs at compile time — without a garbage collector.

---

## 2. Generics

**Q: What are Generics in Rust?**

Generics are a way to write one piece of code that works for multiple types — instead of writing the same function or struct over and over for each type, you write it once and let the compiler figure out the specific types when it's actually used. Think of it like a template.

### Basic Example

```rust
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

// Works for i32, f64, char — any type that supports comparison
let result = largest(10, 20);       // i32
let result = largest(3.5, 2.1);    // f64
```

### Where Generics Work

- **Functions** — like the example above
- **Structs** — a `Stack<T>` can be a stack of integers, strings, whatever
- **Enums** — `Option<T>` and `Result<T, E>` are both generic enums in the standard library

### Trait Bounds

When you write `<T>`, Rust doesn't know anything about that type. If you want to do something with it — like compare it, print it, or add it — you tell Rust what that type is capable of using trait bounds.

```rust
fn print_item<T: std::fmt::Display>(item: T) {
    println!("{}", item);  // T must support Display
}

// Multiple bounds
fn process<T: Clone + std::fmt::Debug>(item: T) {
    let copy = item.clone();
    println!("{:?}", copy);
}
```

### Zero-Cost Abstraction — Monomorphization

Generics in Rust are **zero cost at runtime**. The compiler uses **monomorphization** — it looks at every place you used your generic function, sees what concrete types were passed in, and generates a separate optimized version for each one. No boxing, no virtual dispatch, no overhead.

> 💡 **Key Point:** Write once, works for many types, and costs you nothing at runtime.

---

## 3. Traits

**Q: What are Traits in Rust?**

Traits are Rust's way of defining **shared behavior**. A trait says — any type that implements me, must be able to do these things. It's similar to interfaces in Java or Go, but more powerful and flexible.

### Defining and Implementing a Trait

```rust
trait Greet {
    fn hello(&self) -> String;
}

struct English;
struct Spanish;

impl Greet for English {
    fn hello(&self) -> String { String::from("Hello!") }
}

impl Greet for Spanish {
    fn hello(&self) -> String { String::from("Hola!") }
}
```

### Key Features

- **Default implementations** — provide a default body inside the trait; types can override if needed.
- **Trait bounds** — use `<T: Greet>` to restrict a generic to only types that implement a certain trait.
- **`impl Trait` syntax** — `fn do_something(item: impl Greet)` — cleaner for simple cases.
- **Dynamic dispatch with `dyn Trait`** — `Box<dyn Greet>` decides at runtime which implementation to use.

```rust
// Static dispatch (compile time) — zero cost
fn greet_static<T: Greet>(g: T) { println!("{}", g.hello()); }

// Dynamic dispatch (runtime) — flexible, slight overhead
fn greet_dynamic(g: &dyn Greet) { println!("{}", g.hello()); }
```

### Common Standard Library Traits

| Trait | Purpose |
|-------|---------|
| `Clone` / `Copy` | Duplicating values |
| `Debug` / `Display` | Printing and formatting |
| `Iterator` | Anything that can be looped over |
| `From` / `Into` | Type conversions |
| `PartialOrd` / `Ord` | Comparison and ordering |

```rust
#[derive(Debug, Clone)]  // auto-implement common traits
struct Point { x: f64, y: f64 }
```

> 💡 **Key Point:** Traits are how Rust achieves flexible, reusable code while staying fully type-safe and mostly resolved at compile time.

---

## 4. Lifetimes

**Q: What are Lifetimes in Rust?**

Lifetimes are Rust's way of making sure that **references never outlive the data they point to**. When you take a reference, you're pointing to something you don't own. Lifetimes track how long that reference is valid and make sure you never use it after the original data is gone.

### The Problem — Dangling References

```rust
// This would compile in C++ and crash at runtime
// In Rust, the compiler REJECTS this:
fn bad() -> &String {
    let s = String::from("hello");
    &s  // ERROR: s is dropped here, reference is dangling!
}
```

### Lifetime Annotations

Most of the time the compiler figures out lifetimes on its own (lifetime elision). In certain situations — especially functions that take and return references — you need to be explicit.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
// 'a means: the returned reference is valid as long as BOTH x and y are valid
```

You're not telling the compiler *how long* something lives — you're describing the **relationships** between lifetimes.

### Lifetimes in Structs

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,  // struct cannot outlive the string it borrows
}
```

### The 'static Lifetime

`'static` means the reference lives for the entire duration of the program. String literals are `'static` because they're baked into the binary.

```rust
let s: &'static str = "I live for the entire program";
```

> 💡 **Key Point:** Lifetimes prevent dangling references at compile time. You're describing relationships between lifetimes, not how long things live.

---

## 5. &str vs String

**Q: What is the difference between &str and String?**

Both represent text, but they're fundamentally different in terms of who owns the data, where it lives, and how flexible it is.

### &str — The Borrowed String Slice

- Just a **reference to string data that lives somewhere else** — doesn't own the data
- **Fixed size** — can't grow or shrink it
- **Read only** — can't modify it
- **Very lightweight** — just a pointer and a length under the hood
- String literals like `"hello"` are `&str` with a `'static` lifetime

### String — The Owned String

- **Heap-allocated, growable, owned** string — like a `Vec<u8>` under the hood
- **Growable** — you can push characters, append strings, etc.
- **Modifiable** — you have full control
- **Owns its data** — when it goes out of scope, memory is freed automatically

| | `&str` | `String` |
|--|--------|----------|
| Ownership | Borrowed | Owned |
| Location | Stack / binary / heap | Heap |
| Mutable | No | Yes |
| Growable | No | Yes |
| Cost | Very cheap (ptr + len) | Heap allocation |

### Practical Rule — When to Use Which

```rust
// Use &str for function parameters — more flexible
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

let name = String::from("Amit");
greet(&name);    // String coerces to &str automatically (deref coercion)
greet("Amit");   // &str literal — also works

// Use String when you need to own, build, or modify
fn build_greeting(name: &str) -> String {
    format!("Hello, {}!", name)  // returns owned String
}
```

> 💡 **Key Point:** Use `&str` to read, use `String` to own. `&str` is a lightweight window into string data; `String` is a fully owned, heap-allocated string you can modify.

---

## 6. Smart Pointers

**Q: What are Smart Pointers in Rust?**

Smart pointers are data structures that act like pointers but come with extra capabilities — like automatic memory management, reference counting, or interior mutability. They go beyond a simple reference by owning the data they point to and doing extra work behind the scenes.

### Box\<T\> — Heap Allocation

The simplest smart pointer. Puts a value on the heap instead of the stack.

- Used when data is too large for the stack
- Essential for recursive types like trees and linked lists
- Used for trait objects: `Box<dyn Trait>`

```rust
let b = Box::new(5);  // 5 is now on the heap
// Memory freed automatically when b goes out of scope
```

### Rc\<T\> — Reference Counting (Single-threaded)

Enables multiple owners of the same data in single-threaded code. Keeps a count of owners; frees data when count hits zero.

```rust
use std::rc::Rc;
let a = Rc::new(5);
let b = Rc::clone(&a);  // count = 2
// Both a and b dropped -> count = 0 -> memory freed
```

### Arc\<T\> — Atomic Reference Counting (Multi-threaded)

Same as `Rc<T>` but thread-safe. Use when sharing data across threads.

### RefCell\<T\> — Interior Mutability

Moves borrow checking from compile time to runtime. Allows mutation even through an immutable reference.

```rust
use std::cell::RefCell;
let data = RefCell::new(5);
*data.borrow_mut() += 1;  // mutate even through immutable binding
```

### Arc\<T\> + Mutex\<T\> — Shared Mutable State Across Threads

```rust
use std::sync::{Arc, Mutex};
let counter = Arc::new(Mutex::new(0));
let clone = Arc::clone(&counter);
// In another thread:
let mut val = clone.lock().unwrap();
*val += 1;  // lock auto-released when val drops
```

### Quick Reference

| Smart Pointer | What it does | Thread-safe? |
|---------------|-------------|--------------|
| `Box<T>` | Heap allocation, single owner | Yes |
| `Rc<T>` | Multiple owners, single thread | No |
| `Arc<T>` | Multiple owners, multi-thread | Yes |
| `RefCell<T>` | Interior mutability, runtime borrow check | No |
| `Mutex<T>` | Safe mutation across threads | Yes |

> 💡 **Key Point:** Smart pointers handle cases where simple ownership isn't enough — multiple owners, shared mutation, and cross-thread data access.

---

## 7. Tokio & Rayon

**Q: What are Tokio and Rayon?**

Tokio and Rayon both deal with doing multiple things at once, but they solve completely different problems.

### Tokio — Async Runtime for I/O Bound Work

Rust has `async/await` built into the language, but it doesn't have a built-in runtime to execute async code. Tokio is that runtime — the engine that drives your async code.

- Designed for **I/O bound tasks** — HTTP requests, database calls, file reads, network operations
- Instead of blocking a thread while waiting, Tokio suspends the task and lets the thread do other work
- A small number of threads can handle **thousands of concurrent connections**
- Foundation for web frameworks like **Axum** and **Actix**

```rust
#[tokio::main]
async fn main() {
    // Run two async tasks concurrently
    let (user, orders) = tokio::join!(
        fetch_user(),    // doesn't block the thread
        fetch_orders()   // runs at the same time
    );
}
```

### Rayon — Data Parallelism for CPU Bound Work

Rayon is about splitting heavy computation across multiple CPU cores. Just swap your normal iterator for a parallel one:

```rust
use rayon::prelude::*;

let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8];

// Normal — sequential
let result: Vec<_> = numbers.iter().map(|x| x * 2).collect();

// Parallel — uses all CPU cores, just change iter() to par_iter()
let result: Vec<_> = numbers.par_iter().map(|x| x * 2).collect();
```

### Key Difference

| | Tokio | Rayon |
|--|-------|-------|
| Best for | I/O bound work | CPU bound work |
| Core idea | Async, non-blocking | Parallel threads |
| Use case | Web servers, DB calls, APIs | Data processing, number crunching |
| Threads | Few threads, many tasks | Many threads, heavy computation |

> 💡 **Key Point:** Web server handling 10,000 requests? Tokio. Processing a million records? Rayon. Need both? Use them together.

---

## 8. Pin & Futures

**Q: What are Pin and Futures in Rust?**

### Futures — Lazy Computations

When you write `async fn` in Rust, it doesn't execute immediately. Instead it returns a **Future** — a value that represents a computation that hasn't finished yet.

```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
// Poll::Ready(value) — done, here's the result
// Poll::Pending     — not done yet, come back later
```

The runtime (Tokio) repeatedly polls your future until it returns `Ready`. While it's `Pending`, Tokio goes off and does other work. When you write `await`, the compiler generates a **state machine** that saves where you left off so execution can be paused and resumed.

### The Problem — Self-Referential Structs

When an async function is paused, all its local variables get bundled into a struct by the compiler. Sometimes a Future holds a reference to its own data inside that struct:

```rust
async fn example() {
    let data = vec![1, 2, 3];
    let reference = &data;      // points to data in the SAME struct
    some_async_call().await;    // paused here — both saved in struct
    println!("{:?}", reference);
}
```

If this struct is **moved** to a different memory location, the internal pointer (`reference`) now points to the old location — it's dangling. This is the self-referential struct problem.

### Pin — The Solution

`Pin<P>` is a wrapper that guarantees a value **won't be moved in memory**. Once something is pinned, Rust guarantees it stays at the same memory address for its entire lifetime.

```rust
// poll takes Pin<&mut Self> — not just &mut Self
// "I'll only let you poll this if you promise it won't move"
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;

// In practice — tokio::pin! macro
let my_future = some_async_fn();
tokio::pin!(my_future);  // pins to stack so you can poll manually
```

> 💡 **Key Point:** Futures are how Rust represents async work (lazy state machines polled by a runtime). Pin is what makes it safe to pause and resume them without corrupting memory.

---

## 9. Sync vs Async

**Q: What is Sync and Async in Rust?**

### Sync (Synchronous) — The Default

Synchronous code runs one step at a time, in order. When you call a function, your program stops and waits for it to finish before moving to the next line.

```rust
fn read_file() -> String {
    let content = std::fs::read_to_string("file.txt").unwrap(); // blocks here
    content  // only reaches here after file is fully read
}
```

### Async (Asynchronous) — Non-Blocking

Instead of blocking the thread while waiting, async suspends the current task and lets the thread go do other work until the result is ready.

```rust
async fn read_file() -> String {
    // suspends, doesn't block — thread is free to do other work
    let content = tokio::fs::read_to_string("file.txt").await.unwrap();
    content
}
```

### The Restaurant Analogy

- **Sync** — a waiter takes your order, then stands at the kitchen window doing nothing until your food is ready. Every other table is ignored.
- **Async** — a waiter takes your order, puts it in, then serves other tables while the kitchen works. Same number of waiters, handles way more tables.

### Key Differences

| | Sync | Async |
|--|------|-------|
| Execution | Blocking — waits for each step | Non-blocking — suspends and resumes |
| Thread usage | One task per thread | Many tasks per thread |
| Best for | CPU heavy work, simple scripts | I/O heavy work, web servers, APIs |
| Complexity | Simple and straightforward | More complex, needs a runtime |
| Syntax | Normal functions | `async fn` + `.await` |

### Critical Rules

- **Async is lazy** — just calling an async function does nothing. You must `.await` it.
- **Don't block inside async** — use `tokio::time::sleep` not `std::thread::sleep`
- **spawn_blocking** — for CPU heavy work inside async, move it off the async thread pool

```rust
// WRONG — blocks the async thread
async fn bad() { std::thread::sleep(Duration::from_secs(1)); }

// RIGHT — suspends, doesn't block
async fn good() { tokio::time::sleep(Duration::from_secs(1)).await; }

// Concurrent tasks
let (user, orders) = tokio::join!(fetch_user(), fetch_orders());
```

> 💡 **Key Point:** Sync blocks and waits. Async suspends and multitasks. Async lets a small number of threads handle massive concurrent workloads.

---

## 10. Multithreading & Channels

**Q: What is Multithreading and Channels in Rust?**

### Rust's Concurrency Guarantee — Fearless Concurrency

In most languages, multithreading bugs like data races and shared state corruption are runtime problems. In Rust, **the compiler catches all of that at compile time**. If your multithreaded code compiles, it's guaranteed to be free of data races.

### Spawning Threads

```rust
use std::thread;

let handle = thread::spawn(|| {
    println!("Hello from a new thread!");
});

handle.join().unwrap();  // wait for thread to finish
```

### Moving Data Into Threads

You can't freely pass references to other threads. The solution is to move ownership into the thread:

```rust
let data = vec![1, 2, 3];
thread::spawn(move || {  // move transfers ownership into the thread
    println!("{:?}", data);  // now safe
});
```

### Shared Mutable State — Arc + Mutex

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..5 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut val = counter.lock().unwrap();  // only one thread at a time
        *val += 1;
    });  // lock auto-released when val drops
    handles.push(handle);
}

for handle in handles { handle.join().unwrap(); }
println!("Final: {}", *counter.lock().unwrap());  // 5
```

### Channels — Message Passing Between Threads

Instead of sharing memory, threads communicate by sending messages. `mpsc` = multiple producer, single consumer.

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send(String::from("hello from thread")).unwrap();
});

let message = rx.recv().unwrap();  // blocks until message arrives
println!("{}", message);

// Multiple producers
let tx2 = tx.clone();
thread::spawn(move || tx2.send("from thread 2").unwrap());
for msg in rx { println!("{}", msg); }
```

### Shared State vs Message Passing

| | Shared State (Arc+Mutex) | Message Passing (Channels) |
|--|--------------------------|---------------------------|
| Best for | Shared counters, caches | Pipelines, worker queues |
| Mental model | Shared whiteboard with a lock | Post office — send packages |
| Risk | Deadlocks if misused | Complex with many channels |

### OS Threads vs Tokio Tasks

| | OS Thread (`thread::spawn`) | Tokio Task (`tokio::spawn`) |
|--|-----------------------------|-----------------------------|
| Weight | Heavy — one per task | Lightweight — thousands possible |
| Best for | CPU intensive work | Async I/O work |
| Cost | MB of stack each | Tiny, shared thread pool |

> 💡 **Key Point:** Rust gives you shared state (Arc+Mutex) and message passing (channels). Whichever you use, the compiler guarantees no data races — that's the fearless concurrency promise.

---

## 11. Iterators

**Q: What are Iterators in Rust?**

An iterator is essentially **anything that can produce a sequence of values one at a time**. In Rust, an iterator is anything that implements the `Iterator` trait:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Every time you call `next()`, it gives you `Some(value)` for the next item, or `None` when it's done.

### Creating Iterators

```rust
let v = vec![1, 2, 3];

v.iter()       // iterates over &T  — borrows the values (read only)
v.iter_mut()   // iterates over &mut T — borrows mutably (can modify)
v.into_iter()  // iterates over T — takes ownership (consumes the vec)
```

### Iterator Adaptors

```rust
let v = vec![1, 2, 3, 4, 5, 6];

let result: Vec<i32> = v.iter()
    .filter(|&&x| x % 2 == 0)   // keep only even numbers
    .map(|&x| x * 10)            // multiply each by 10
    .collect();                  // gather results into a Vec

// result = [20, 40, 60]
```

Common adaptors:
- `map` — transform each element
- `filter` — keep elements matching a condition
- `take(n)` — take only the first n elements
- `skip(n)` — skip the first n elements
- `enumerate` — gives you `(index, value)` pairs
- `zip` — combine two iterators into pairs
- `flat_map` — map and flatten nested results
- `chain` — connect two iterators end to end

### Consumers — Actually Running the Iterator

Iterators are **lazy** — nothing runs until you consume the iterator:

```rust
let v = vec![1, 2, 3, 4, 5];

let doubled: Vec<_> = v.iter().map(|x| x * 2).collect();  // collect
let total: i32 = v.iter().sum();          // 15
let count = v.iter().filter(|&&x| x > 2).count();  // 3
let has_even = v.iter().any(|&x| x % 2 == 0);      // true
let all_positive = v.iter().all(|&x| x > 0);        // true
let first_even = v.iter().find(|&&x| x % 2 == 0);  // Some(2)
let product = v.iter().fold(1, |acc, &x| acc * x); // 120
```

### Custom Iterator

```rust
struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count <= 5 {
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let counter = Counter { count: 0 };
    for value in counter {
        println!("{}", value);  // prints 1 2 3 4 5
    }
}
```

Once you implement just `next()`, you get **all other iterator methods for free** — map, filter, sum, etc.

### for loop is just an iterator

```rust
// These are identical:
for x in &v { println!("{}", x); }

// same as:
let mut iter = v.iter();
while let Some(x) = iter.next() { println!("{}", x); }
```

> 💡 **Key Point:** Iterators are lazy sequences that compile down to the same assembly as hand-written loops — zero overhead. This is Rust's zero cost abstractions in action.

---

## 12. Error Handling

**Q: How does Error Handling work in Rust?**

Rust has **no exceptions**. Instead, errors are just values — returned explicitly using `Result` and `Option`. This forces you to handle errors at compile time, not discover them at runtime.

### Option\<T\> — For Values That Might Not Exist

```rust
enum Option<T> {
    Some(T),   // there is a value
    None,      // there is no value
}
```

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some(String::from("Amit")) } else { None }
}

// Safe ways to handle Option
match find_user(1) {
    Some(name) => println!("Found: {}", name),
    None => println!("User not found"),
}

// if let — cleaner for single case
if let Some(name) = find_user(1) {
    println!("Found: {}", name);
}

// unwrap_or — provide a default
let name = find_user(99).unwrap_or(String::from("Guest"));

// unwrap — gives value or PANICS if None — avoid in production!
let name = find_user(1).unwrap();
```

### Result\<T, E\> — For Operations That Can Fail

```rust
enum Result<T, E> {
    Ok(T),   // success — here's the value
    Err(E),  // failure — here's the error
}
```

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Cannot divide by zero"))
    } else {
        Ok(a / b)
    }
}

match divide(10.0, 2.0) {
    Ok(result) => println!("Result: {}", result),
    Err(e) => println!("Error: {}", e),
}

let result = divide(10.0, 0.0)
    .unwrap_or_else(|e| {
        println!("Caught error: {}", e);
        0.0
    });
```

### The ? Operator — Propagate Errors Cleanly

Instead of manually matching every Result, `?` says — if this is `Ok`, give me the value and continue. If it's `Err`, return the error immediately from this function.

```rust
use std::fs::File;
use std::io::{self, Read};

// Without ? — verbose
fn read_file_verbose(path: &str) -> Result<String, io::Error> {
    let file = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    // ...
}

// With ? — clean and readable
fn read_file(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;   // if error, return immediately
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

### Custom Error Types

```rust
#[derive(Debug)]
enum AppError {
    DatabaseError(String),
    NotFound(String),
    ValidationError(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::DatabaseError(msg) => write!(f, "DB Error: {}", msg),
            AppError::NotFound(msg) => write!(f, "Not Found: {}", msg),
            AppError::ValidationError(msg) => write!(f, "Validation: {}", msg),
        }
    }
}

fn get_user(id: u32) -> Result<String, AppError> {
    if id == 0 {
        return Err(AppError::ValidationError(String::from("ID cannot be zero")));
    }
    Ok(format!("User_{}", id))
}
```

> 💡 **Key Point:** In Rust, errors are values — you can't ignore them. The compiler forces you to handle every possible failure. No null pointer exceptions, no unhandled exceptions crashing your server at 3am.

---

## 13. Closures & Function Pointers

**Q: What are Closures in Rust?**

Closures are **anonymous functions that can capture variables from the surrounding scope** — like lambdas in other languages.

### Basic Syntax

```rust
// Regular function
fn add(x: i32, y: i32) -> i32 { x + y }

// Closure — same thing, shorter syntax
let add = |x, y| x + y;
println!("{}", add(2, 3));  // 5

let square = |x| x * x;
let multiply = |x: i32, y: i32| -> i32 { x * y };
```

### Closures Capture Their Environment

```rust
let threshold = 10;

// Closure captures 'threshold' from surrounding scope
let is_big = |x| x > threshold;

println!("{}", is_big(15));  // true
println!("{}", is_big(5));   // false
// A regular fn cannot do this
```

### Three Closure Traits — Fn, FnMut, FnOnce

**`FnOnce`** — takes ownership of captured values. Can only be called **once**.

```rust
let name = String::from("Amit");
let greet = move || {
    println!("Hello, {}", name);  // name is moved into closure
};
greet();  // works
// greet();  // ERROR — name already moved
```

**`FnMut`** — mutably borrows captured values. Can be called **multiple times** but needs `mut`.

```rust
let mut count = 0;
let mut increment = || {
    count += 1;
    println!("Count: {}", count);
};
increment();  // Count: 1
increment();  // Count: 2
```

**`Fn`** — immutably borrows captured values. Can be called **multiple times** freely.

```rust
let greeting = String::from("Hello");
let greet = |name| println!("{}, {}!", greeting, name);
greet("Amit");    // Hello, Amit!
greet("Rahul");   // Hello, Rahul!
```

### Closures with Iterators

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

let evens: Vec<i32> = numbers.iter()
    .filter(|&&x| x % 2 == 0)
    .cloned()
    .collect();

let multiplier = 3;
let tripled: Vec<i32> = numbers.iter()
    .map(|&x| x * multiplier)  // captures multiplier
    .collect();
```

### Passing Closures to Functions

```rust
fn filter_numbers(numbers: &[i32], predicate: impl Fn(i32) -> bool) -> Vec<i32> {
    numbers.iter()
        .filter(|&&x| predicate(x))
        .cloned()
        .collect()
}

let evens = filter_numbers(&numbers, |x| x % 2 == 0);
let big = filter_numbers(&numbers, |x| x > 3);
```

### Function Pointers

```rust
fn double(x: i32) -> i32 { x * 2 }
fn triple(x: i32) -> i32 { x * 3 }

let operation: fn(i32) -> i32 = double;
println!("{}", operation(5));  // 10
```

> 💡 **Key Point:** The three Fn traits give you precise control over how data is captured — by reference, mutably, or by ownership — and the compiler enforces the right one automatically.

---

## 14. Enums & Pattern Matching

**Q: What are Enums and Pattern Matching in Rust?**

Enums in Rust are **sum types** — a value that can be one of several different variants, where each variant can carry its own data.

### Enums with Data

```rust
enum Shape {
    Circle(f64),                         // radius
    Rectangle(f64, f64),                 // width, height
    Triangle { base: f64, height: f64 }, // named fields
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rectangle(w, h) => w * h,
        Shape::Triangle { base, height } => 0.5 * base * height,
    }
}
```

### match — Exhaustive Pattern Matching

`match` must handle every possible case — the compiler rejects your code if you miss one:

```rust
enum Status {
    Active,
    Inactive,
    Banned { reason: String },
    Pending(u32),
}

fn describe(status: &Status) -> String {
    match status {
        Status::Active => String::from("User is active"),
        Status::Inactive => String::from("User is inactive"),
        Status::Banned { reason } => format!("Banned: {}", reason),
        Status::Pending(days) => format!("Active in {} days", days),
        // Forget any variant? Compiler error!
    }
}
```

### Match Guards & Multiple Patterns

```rust
let number = 7;
match number {
    n if n < 0 => println!("Negative"),
    0 => println!("Zero"),
    n if n % 2 == 0 => println!("Positive even"),
    _ => println!("Positive odd"),
}

let x = 3;
match x {
    1 | 2 => println!("one or two"),
    3..=5 => println!("three to five"),
    _ => println!("something else"),
}
```

### if let — When You Only Care About One Variant

```rust
// Without if let — verbose
match config {
    Some(value) => println!("Config: {}", value),
    None => {}
}

// With if let — clean
if let Some(value) = config {
    println!("Config: {}", value);
}
```

### while let — Loop Until Pattern Stops Matching

```rust
let mut stack = vec![1, 2, 3, 4, 5];

while let Some(top) = stack.pop() {
    println!("Popped: {}", top);
}
// Prints: 5, 4, 3, 2, 1
```

### Destructuring

```rust
struct Point { x: i32, y: i32 }
let p = Point { x: 3, y: 7 };

// Destructure in let
let Point { x, y } = p;

// Destructure in match
match p {
    Point { x: 0, y } => println!("On y-axis at {}", y),
    Point { x, y: 0 } => println!("On x-axis at {}", x),
    Point { x, y } => println!("At ({}, {})", x, y),
}

// Destructure tuples
let (a, b, c) = (1, 2, 3);
```

> 💡 **Key Point:** Rust's enums with exhaustive `match` guarantee you never miss handling a case. The compiler is your safety net.

---

## 15. Collections

**Q: What are the main Collections in Rust?**

### Vec\<T\> — The Growable Array

```rust
let mut v = vec![1, 2, 3];

// Adding
v.push(4);           // add to end
v.insert(0, 0);      // insert at index 0

// Removing
v.pop();             // remove and return last — Option<T>
v.remove(0);         // remove at index

// Accessing
let third = &v[2];   // panics if out of bounds
let third = v.get(2); // returns Option<&T> — safe

// Useful methods
v.len();
v.is_empty();
v.contains(&3);
v.sort();
v.dedup();
v.retain(|&x| x > 2);
```

### HashMap\<K, V\> — Key-Value Store

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, i32> = HashMap::new();

scores.insert(String::from("Amit"), 95);
scores.insert(String::from("Rahul"), 87);

// Accessing
let score = scores.get("Amit");  // Some(95)

// Insert only if key doesn't exist
scores.entry(String::from("New")).or_insert(0);

// Update existing value
let count = scores.entry(String::from("Amit")).or_insert(0);
*count += 5;

// Iterating
for (name, score) in &scores {
    println!("{}: {}", name, score);
}

scores.remove("Rahul");
```

### HashSet\<T\> — Unique Values Only

```rust
use std::collections::HashSet;

let mut set: HashSet<i32> = HashSet::new();
set.insert(1);
set.insert(2);
set.insert(2);  // duplicate — silently ignored

let a: HashSet<i32> = vec![1, 2, 3, 4].into_iter().collect();
let b: HashSet<i32> = vec![3, 4, 5, 6].into_iter().collect();

let union: HashSet<_> = a.union(&b).collect();
let intersection: HashSet<_> = a.intersection(&b).collect();  // {3, 4}
let difference: HashSet<_> = a.difference(&b).collect();      // {1, 2}
```

### Choosing the Right Collection

| Collection | Use when |
|------------|----------|
| `Vec<T>` | Ordered list, index access, push/pop |
| `HashMap<K,V>` | Key-value lookup, O(1) access by key |
| `HashSet<T>` | Unique values, fast membership check |
| `BTreeMap<K,V>` | Key-value but need sorted order |
| `VecDeque<T>` | Efficient push/pop from both ends |
| `BinaryHeap<T>` | Priority queue — always get max element |

> 💡 **Key Point:** The `entry` API on HashMap handles insert-if-not-exists or update-if-exists in a single, race-condition-free operation.

---

# DESIGN PATTERNS

---

## 16. Builder Pattern

**What is it:** A way to **construct complex objects step by step** instead of passing everything into one giant constructor.

**The Problem:** A struct with 8 fields — some required, some optional. Without a builder you must specify every field, easy to mix up order, breaks when struct changes.

```rust
// The final struct
struct Request {
    url: String,
    method: String,
    timeout: u32,
    retries: u32,
    body: Option<String>,
    auth_token: Option<String>,
}

// The builder
struct RequestBuilder {
    url: String,
    method: String,
    timeout: u32,
    retries: u32,
    body: Option<String>,
    auth_token: Option<String>,
}

impl RequestBuilder {
    fn new(url: &str) -> Self {
        RequestBuilder {
            url: url.to_string(),
            method: String::from("GET"),  // sensible defaults
            timeout: 30,
            retries: 3,
            body: None,
            auth_token: None,
        }
    }

    fn method(mut self, method: &str) -> Self {
        self.method = method.to_string();
        self  // return self for chaining
    }

    fn timeout(mut self, timeout: u32) -> Self {
        self.timeout = timeout;
        self
    }

    fn auth_token(mut self, token: &str) -> Self {
        self.auth_token = Some(token.to_string());
        self
    }

    fn build(self) -> Request {
        Request {
            url: self.url,
            method: self.method,
            timeout: self.timeout,
            retries: self.retries,
            body: self.body,
            auth_token: self.auth_token,
        }
    }
}

// Usage — clean and readable
let request = RequestBuilder::new("https://api.example.com")
    .method("POST")
    .timeout(60)
    .auth_token("Bearer xyz123")
    .build();
```

**Why `mut self` and returning `self`?** Each setter takes ownership, modifies it, and returns it — enabling **method chaining**.

**Key benefits:**
- Only set what you need — everything else gets a sensible default
- Readable — reads like plain English
- Extensible — add new fields without breaking existing call sites

> 💡 Seen everywhere in real Rust: `reqwest`, `tokio`, virtually every major crate uses Builder.

---

## 17. Newtype Pattern

**What is it:** Wrapping a primitive type in a struct to give it a **distinct identity** so the compiler can catch misuse.

**The Problem:**
```rust
fn create_user(user_id: u64, company_id: u64, role_id: u64) { }

// Easy to call it wrong — compiler won't catch this
create_user(company_id, user_id, role_id);  // swapped!
```

**The Solution:**
```rust
struct UserId(u64);
struct CompanyId(u64);
struct RoleId(u64);

fn create_user(user_id: UserId, company_id: CompanyId, role_id: RoleId) {
    println!("Creating user {}", user_id.0);  // access inner value with .0
}

create_user(uid, cid, rid);   // ✅ correct
create_user(cid, uid, rid);   // ❌ compiler error — wrong types!
```

**Zero cost at runtime** — the wrapper compiles away completely.

**Adding behavior:**
```rust
struct Meters(f64);
struct Kilograms(f64);

impl std::ops::Add for Meters {
    type Output = Meters;
    fn add(self, other: Meters) -> Meters { Meters(self.0 + other.0) }
}

// Can't accidentally add meters to kilograms — different types!
// let wrong = Meters(10.0) + Kilograms(5.0);  // ❌
```

> 💡 Real world: `UserId`, `OrderId`, `EmailAddress(String)`. NASA lost a spacecraft in 1999 due to a unit mixup this pattern would have prevented!

---

## 18. Strategy Pattern

**What is it:** Define a family of algorithms, put each in its own type, and make them **interchangeable at runtime**.

**The Problem:** Adding new payment methods means touching the same function forever — violates Open/Closed principle.

**The Solution:**
```rust
trait PaymentStrategy {
    fn pay(&self, amount: f64) -> bool;
    fn name(&self) -> &str;
}

struct CreditCard { card_number: String, cvv: String }
struct PayPal { email: String }
struct Crypto { wallet_address: String }

impl PaymentStrategy for CreditCard {
    fn pay(&self, amount: f64) -> bool {
        println!("Charging ${} to credit card", amount);
        true
    }
    fn name(&self) -> &str { "Credit Card" }
}

impl PaymentStrategy for PayPal {
    fn pay(&self, amount: f64) -> bool {
        println!("Sending ${} via PayPal to {}", amount, self.email);
        true
    }
    fn name(&self) -> &str { "PayPal" }
}

struct PaymentProcessor {
    strategy: Box<dyn PaymentStrategy>,
}

impl PaymentProcessor {
    fn new(strategy: Box<dyn PaymentStrategy>) -> Self {
        PaymentProcessor { strategy }
    }

    fn checkout(&self, amount: f64) {
        if self.strategy.pay(amount) {
            println!("Payment successful!");
        }
    }

    fn change_strategy(&mut self, strategy: Box<dyn PaymentStrategy>) {
        self.strategy = strategy;
    }
}

let mut processor = PaymentProcessor::new(Box::new(CreditCard {
    card_number: String::from("1234567890123456"),
    cvv: String::from("123"),
}));

processor.checkout(99.99);
processor.change_strategy(Box::new(PayPal { email: String::from("amit@example.com") }));
processor.checkout(49.99);
```

**Adding a new strategy** — just create a new struct and implement `PaymentStrategy`. Zero changes to `PaymentProcessor`.

> 💡 **Open/Closed Principle** — open for extension, closed for modification. Traits make this completely natural in Rust.

---

## 19. Observer Pattern

**What is it:** When something happens, **automatically notify other parts** that care about it — without tight coupling.

**The Problem:** `place_order` knows about every downstream system. Adding a new one means touching it every time.

**The Solution — Channels:**
```rust
use std::sync::mpsc;
use std::thread;

#[derive(Debug, Clone)]
struct OrderEvent {
    order_id: u64,
    customer_email: String,
    amount: f64,
}

// Email observer — runs in its own thread
let (email_tx, email_rx) = mpsc::channel::<OrderEvent>();
thread::spawn(move || {
    for event in email_rx {
        println!("📧 Sending confirmation email to {}", event.customer_email);
    }
});

// Inventory observer
let (inv_tx, inv_rx) = mpsc::channel::<OrderEvent>();
thread::spawn(move || {
    for event in inv_rx {
        println!("📦 Updating inventory for order {}", event.order_id);
    }
});

// Place an order — just send the event, don't care who handles it
let event = OrderEvent { order_id: 1001, customer_email: String::from("amit@example.com"), amount: 299.99 };
email_tx.send(event.clone()).unwrap();
inv_tx.send(event).unwrap();
```

**With Tokio broadcast for async (multiple consumers):**
```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<OrderEvent>(100);

let mut rx1 = tx.subscribe();  // email service
let mut rx2 = tx.subscribe();  // inventory service

tokio::spawn(async move {
    while let Ok(event) = rx1.recv().await {
        println!("Email: {:?}", event);
    }
});

tx.send(order_event).unwrap();  // all subscribers receive it
```

> 💡 Adding a new observer means just subscribing to the channel. Zero changes to existing code.

---

## 20. State Pattern

**What is it:** Model an object that **behaves differently depending on its state**, with clear rules about allowed transitions.

**The Solution — Enums:**
```rust
#[derive(Debug)]
enum OrderState {
    Pending,
    Confirmed { confirmation_id: String },
    Shipped { tracking_number: String },
    Delivered { delivered_at: String },
    Cancelled { reason: String },
}

struct Order {
    id: u64,
    state: OrderState,
}

impl Order {
    fn new(id: u64) -> Self {
        Order { id, state: OrderState::Pending }
    }

    fn confirm(mut self, confirmation_id: &str) -> Self {
        self.state = match self.state {
            OrderState::Pending => {
                println!("✅ Order {} confirmed!", self.id);
                OrderState::Confirmed { confirmation_id: confirmation_id.to_string() }
            }
            other => panic!("Cannot confirm order in state {:?}", other),
        };
        self
    }

    fn ship(mut self, tracking: &str) -> Self {
        self.state = match self.state {
            OrderState::Confirmed { .. } => {
                println!("🚚 Order {} shipped!", self.id);
                OrderState::Shipped { tracking_number: tracking.to_string() }
            }
            other => panic!("Cannot ship order in state {:?}", other),
        };
        self
    }

    fn cancel(mut self, reason: &str) -> Self {
        self.state = match self.state {
            OrderState::Pending | OrderState::Confirmed { .. } => {
                OrderState::Cancelled { reason: reason.to_string() }
            }
            other => panic!("Cannot cancel order in state {:?}", other),
        };
        self
    }
}

// Usage
let order = Order::new(1001)
    .confirm("CNF-ABC123")
    .ship("TRK-XYZ789");
```

**Why Rust's version is better than OOP:** The `match` statement forces you to handle every state — you can never forget a case. The compiler is your safety net.

> 💡 Invalid state transitions become impossible. Each state carries only the data relevant to it.

---

## 21. Singleton Pattern

**What is it:** Ensure a type has **exactly one instance** shared across the entire program.

**The Solution — OnceLock:**
```rust
use std::sync::OnceLock;

#[derive(Debug)]
struct Config {
    database_url: String,
    port: u16,
    debug_mode: bool,
}

impl Config {
    fn load() -> Self {
        Config {
            database_url: String::from("postgres://localhost/mydb"),
            port: 8080,
            debug_mode: false,
        }
    }
}

static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| Config::load())
}

fn main() {
    let config = get_config();
    println!("Port: {}", config.port);
    // Same instance every call — initialized only once
}
```

**With Mutex for mutable singleton:**
```rust
use std::sync::{OnceLock, Mutex};

static COUNTER: OnceLock<Mutex<u64>> = OnceLock::new();

fn get_counter() -> &'static Mutex<u64> {
    COUNTER.get_or_init(|| Mutex::new(0))
}

fn increment() {
    let mut count = get_counter().lock().unwrap();
    *count += 1;
}
```

**Honest take:** Most experienced Rust developers prefer **dependency injection** over global singletons — passing `Arc<Config>` explicitly is more testable. But for logging, global config, metrics — `OnceLock` is perfectly idiomatic.

> 💡 `OnceLock` guarantees thread-safe initialization — even if multiple threads call `get_or_init` simultaneously, the initialization runs exactly once.

---

## 22. RAII Pattern

**What is it:** **Resources are acquired when an object is created and automatically released when it goes out of scope** — no manual cleanup ever needed.

**The Problem in C:**
```c
FILE* f = fopen("data.txt", "r");
// ... what if an early return happens? file never closed!
fclose(f);  // manual cleanup — easy to forget
```

**The Solution — Rust's Drop trait:**
```rust
use std::fs::File;
use std::io::Write;

fn write_data() -> std::io::Result<()> {
    let mut file = File::create("output.txt")?;  // resource acquired
    file.write_all(b"Hello!")?;
    // No close() needed — file automatically closed when it goes out of scope
    Ok(())
}
```

**Implementing Drop on your own types:**
```rust
struct DatabaseConnection {
    connection_string: String,
}

impl DatabaseConnection {
    fn new(conn_str: &str) -> Self {
        println!("🔌 Opening connection to {}", conn_str);
        DatabaseConnection { connection_string: conn_str.to_string() }
    }
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        println!("🔒 Closing connection to {}", self.connection_string);
        // cleanup happens automatically
    }
}

fn main() {
    {
        let conn = DatabaseConnection::new("postgres://localhost/mydb");
        // use conn...
    }  // Drop called automatically — connection closed!
    println!("Connection is gone — no leak possible");
}
```

**RAII with Mutex locks:**
```rust
{
    let mut guard = data.lock().unwrap();  // lock acquired
    guard.push(4);
}  // MutexGuard's Drop is called — lock automatically released!
// No unlock() needed — impossible to forget
```

**Every major resource uses RAII:**

| Resource | Acquired | Released via Drop |
|----------|----------|------------------|
| File | `File::open()` | File closed |
| Mutex | `.lock()` | Lock released |
| Memory | `Box::new()` | Memory freed |
| DB connection | `Pool::get()` | Returned to pool |
| Network socket | `TcpStream::connect()` | Socket closed |

> 💡 **Key Point:** RAII makes resource leaks structurally impossible. You don't rely on developers remembering to clean up — the compiler enforces it through ownership.

---

# BACKEND SYSTEMS

---

## 23. Kafka & Message Queues

### What is Kafka and why do we use it?

Kafka is a **distributed event streaming platform** — it lets different parts of your application communicate by sending and receiving messages asynchronously.

Instead of every service talking directly to every other service (tight coupling), Kafka sits in the middle as a **buffer and communication backbone** — producers send messages to Kafka, consumers read from it at their own pace, completely independently.

**Real world use cases:** Processing payments, tracking user activity, syncing databases, sending notifications, building audit logs.

---

### Topics, Partitions, and Offsets

- **Topic** — a category or channel. Producers write to a topic, consumers read from it. e.g., `user-signups`, `payment-events`

- **Partition** — how Kafka scales. Each topic is split into multiple partitions — separate ordered logs running in parallel on different machines. Messages within a partition are always in order; no global ordering across partitions.

- **Offset** — the position of a message within a partition, like an index. Consumers track which offset they've read up to. If they crash and restart, they know exactly where to continue. This is what gives Kafka its reliability.

---

### Producers, Consumers, and Consumer Groups

- **Producer** — any service that writes messages into Kafka. Decides which topic to send to.

- **Consumer** — any service that reads messages from Kafka. Tracks its offset, processes at its own pace.

- **Consumer Group** — a set of consumers working together to consume a topic. Kafka automatically assigns each partition to exactly one consumer in the group — work is **distributed and parallelized**. If one consumer crashes, Kafka automatically **rebalances** — redistributing that partition to another consumer.

---

### At-least-once, At-most-once, and Exactly-once Delivery

- **At-most-once** — sent once, never retried. May lose messages. No duplicates. Only for metrics/logs where loss is acceptable.

- **At-least-once** — retried until acknowledged. Guaranteed delivery but may get **duplicates**. Most common. Requires **idempotent consumers**.

- **Exactly-once** — delivered and processed exactly one time. Supported via **idempotent producers** and **transactional APIs**. More complex, some performance cost.

> In most production systems, **at-least-once with idempotent consumers** is the sweet spot.

---

### What happens when a Kafka consumer goes down?

1. Each consumer sends **heartbeats** to the group coordinator
2. If heartbeat stops, Kafka waits for a timeout then marks consumer as dead
3. Kafka triggers a **rebalance** — redistributes partitions to remaining healthy consumers
4. Surviving consumers check committed offsets and pick up right where dead consumer left off
5. No messages are lost

The brief period between consumer dying and rebalance completing is called **rebalance lag**. Tune heartbeat and session timeout settings carefully in production.

---

### Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|--|-------|----------|
| Type | Distributed log | Traditional message queue |
| Message retention | Retained for configurable period | Deleted after consumption |
| Scale | Millions of messages/second | Moderate load |
| Consumers | Multiple independent consumer groups | Single consumer per message |
| Use case | Event streaming, audit logs, replay | Task queues, job distribution |
| Direction | Pull-based | Push-based |

**When to choose:**
- **Kafka** — need to replay events, multiple independent consumers, huge throughput
- **RabbitMQ** — simple job queues, request routing, moderate scale

---

## 24. REST APIs & gRPC

### What is REST?

REST (Representational State Transfer) is an architectural style for building APIs over HTTP.

**Core principles:**
- **Stateless** — every request contains all information needed. Server stores no session state. Easy to scale.
- **Resource based** — everything is a resource with a URL. `/users/123`, `/orders/456`
- **Uniform interface** — `GET` to read, `POST` to create, `PUT`/`PATCH` to update, `DELETE` to remove
- **Client-Server separation** — client and server are independent
- **Cacheable** — responses indicate whether they can be cached

---

### HTTP Status Codes

**2xx — Success**
- `200 OK` — request succeeded
- `201 Created` — resource created
- `204 No Content` — success, nothing to return (common for DELETE)

**3xx — Redirection**
- `301 Moved Permanently` — resource has new URL forever
- `304 Not Modified` — cached version still valid

**4xx — Client Errors**
- `400 Bad Request` — malformed or invalid request
- `401 Unauthorized` — not authenticated (not logged in)
- `403 Forbidden` — authenticated but not allowed
- `404 Not Found` — resource doesn't exist
- `409 Conflict` — conflict with current state (e.g. duplicate)
- `429 Too Many Requests` — rate limited

**5xx — Server Errors**
- `500 Internal Server Error` — something broke on our end
- `502 Bad Gateway` — upstream server returned invalid response
- `503 Service Unavailable` — server down or overloaded

> **401 vs 403:** 401 = not logged in. 403 = logged in but no permission. Very different things.

---

### Idempotency

Idempotency means — calling the same operation multiple times produces the **same result as calling it once**.

**Why it matters:** Networks are unreliable. A client sends a request, doesn't get a response, retries. If not idempotent, you could charge a customer twice, create duplicate records.

| Method | Idempotent? |
|--------|-------------|
| `GET` | ✅ Yes — just reading |
| `PUT` | ✅ Yes — replacing with same data |
| `DELETE` | ✅ Yes — deleting twice = still deleted |
| `POST` | ❌ No — calling twice creates two resources |

For `POST` requests needing safe retry — use an **idempotency key**. Client sends a unique key; server stores result against it. Same key comes in again → return stored result, don't process again.

---

### What is gRPC?

gRPC is a **remote procedure call framework** built by Google. Instead of resources and HTTP methods, gRPC lets you call functions on a remote server as if they were local.

Uses **HTTP/2** for transport and **Protocol Buffers (protobuf)** for serialization — faster and more efficient than REST+JSON.

| | REST | gRPC |
|--|------|------|
| Data format | JSON (human readable, verbose) | Protobuf (binary, compact, fast) |
| Communication | Request/response | Request/response + streaming |
| Contract | OpenAPI (loose) | `.proto` file (strict, generates code) |
| Browser support | Works everywhere | Needs gRPC-Web setup |
| Performance | Good | Excellent |

**When to use which:**
- **REST** — public APIs, browser clients, simplicity, human readability
- **gRPC** — internal microservice communication, high performance, streaming, when you control both client and server

> Most modern architectures: **REST for external APIs, gRPC for internal service-to-service communication.**

---

### API Versioning

Three main approaches:

**URL versioning** — `/v1/users`, `/v2/users`
- Most common, most explicit
- Easy to see, easy to route
- Standard in most companies

**Header versioning** — `API-Version: 2` header
- Keeps URLs clean
- Less visible, harder to test in browser

**Query parameter** — `/users?version=2`
- Easy to test
- Can feel messy

**Strategy for breaking changes:**
- Maintain old version while rolling out new one
- Give clients a deprecation timeline
- Never remove a version without notice
- Support at least two versions simultaneously

---

## 25. Docker & Kubernetes

### What is Docker?

Docker is a **containerization platform** — packages your application with everything it needs (code, runtime, libraries, config) into a single unit called a **container**.

**Problem it solves:** The classic "it works on my machine" problem. Before Docker, deploying meant making sure production had the exact right versions of everything. With Docker, you ship a container with everything built in — runs the same everywhere.

**Containers are lightweight** — unlike VMs which run a full OS, containers share the host OS kernel. Run dozens on the same machine with minimal overhead.

---

### Docker Image vs Container

- **Docker Image** — a **blueprint** (read-only template). Contains code, dependencies, config. Like a class. Doesn't run on its own.

- **Container** — a **running instance** of an image. Like an object from a class. Multiple containers from the same image. Each isolated — own filesystem, network, process space.

Built from a **Dockerfile** in **layers** — each instruction creates a layer, layers are cached. Only changed layers rebuild — makes builds fast.

---

### What is Kubernetes?

Docker runs containers. When you have hundreds of containers across multiple machines in production, you need something to **manage, orchestrate, and automate** all of that. That's Kubernetes (K8s).

**Kubernetes handles:**
- **Scheduling** — deciding which machine to run each container on
- **Auto-scaling** — spinning up more containers when traffic increases
- **Self-healing** — if a container crashes, K8s automatically restarts it
- **Load balancing** — distributing traffic across instances
- **Rolling updates** — deploying new versions with zero downtime
- **Service discovery** — letting containers find and talk to each other

> Docker is the shipping container. Kubernetes is the entire port logistics system managing thousands of containers.

---

### Core Kubernetes Components

**Pod** — the smallest deployable unit. Wraps one or more containers sharing the same network and storage. Pods are ephemeral — created and destroyed constantly. Never rely on a pod's IP address.

**Deployment** — manages pods. You declare "I want 3 replicas always running" and it makes it happen. Handles rolling updates — brings up new pods, takes down old ones gracefully.

**Service** — gives you a **stable network endpoint** to reach pods (since pod IPs change constantly). Load balances traffic across healthy pods. Types:
- `ClusterIP` — internal access only
- `NodePort` — external access on a port
- `LoadBalancer` — provisions a cloud load balancer

---

### Service vs Ingress

**Service** — transport layer (TCP/UDP). Gives a stable IP/port to reach pods. If you have 10 microservices with LoadBalancer Services, that's 10 separate load balancers — expensive and messy.

**Ingress** — HTTP/HTTPS layer on top. Single entry point with **routing rules** based on URL path or hostname:

```
api.myapp.com/users    → users-service
api.myapp.com/orders   → orders-service
api.myapp.com/payments → payments-service
```

All through a single Ingress with one load balancer. Also handles **SSL termination** — configure TLS certificate once, all internal services run plain HTTP.

---

### Container vs Virtual Machine

| | Container | Virtual Machine |
|--|-----------|-----------------|
| OS | Shares host OS kernel | Full OS per VM |
| Startup | Seconds | Minutes |
| Size | MBs | GBs |
| Overhead | Minimal | Significant |
| Isolation | Process-level | Full OS-level |
| Density | 10x more per machine | Heavier |

**In modern cloud:** Containers inside Kubernetes on VMs — infrastructure isolation of VMs + efficiency of containers together.

---

# FINAL CHECKLIST

| Topic | Category | Status |
|-------|----------|--------|
| Ownership & Borrowing | Rust Core | ✅ |
| Generics | Rust Core | ✅ |
| Traits | Rust Core | ✅ |
| Lifetimes | Rust Core | ✅ |
| &str vs String | Rust Core | ✅ |
| Smart Pointers | Rust Core | ✅ |
| Tokio & Rayon | Rust Core | ✅ |
| Pin & Futures | Rust Core | ✅ |
| Sync vs Async | Rust Core | ✅ |
| Multithreading & Channels | Rust Core | ✅ |
| Iterators | Rust Core | ✅ |
| Error Handling | Rust Core | ✅ |
| Closures & Function Pointers | Rust Core | ✅ |
| Enums & Pattern Matching | Rust Core | ✅ |
| Collections | Rust Core | ✅ |
| Builder Pattern | Design Patterns | ✅ |
| Newtype Pattern | Design Patterns | ✅ |
| Strategy Pattern | Design Patterns | ✅ |
| Observer Pattern | Design Patterns | ✅ |
| State Pattern | Design Patterns | ✅ |
| Singleton Pattern | Design Patterns | ✅ |
| RAII Pattern | Design Patterns | ✅ |
| Kafka & Message Queues | Backend | ✅ |
| REST APIs & gRPC | Backend | ✅ |
| Docker & Kubernetes | Backend | ✅ |

---

> 🦀 **Good luck with your Accenture and Persistent interviews!**
> You've got this — now go build something fast and safe.
