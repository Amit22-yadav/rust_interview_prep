# Rust Interview Prep — 9 PM IST Tomorrow

> **How to use this doc:** Read top-to-bottom once. Then re-read Section 6 in the morning, and skim the **One-line summary** at the end of each Q&A in the final hour before the call. Speak the answers out loud — interview-ready phrasing only works if your mouth has done it.

---

## Section 1 — Rust language Q&A

---

### Q1. Ownership

**Q:** Can you walk me through Rust's ownership model?

**A:** Sure. Rust's ownership is the mechanism that gives us memory safety without a garbage collector. The three rules are: every value has exactly one owner, when the owner goes out of scope the value is dropped, and ownership can be moved but not duplicated unless the type is `Copy`. So when I write `let a = String::from("hi"); let b = a;`, the heap buffer's ownership moves from `a` to `b`, and `a` is no longer usable — the compiler will reject any use of it. For small stack-only types like integers, `Copy` is implemented, so assignment copies bitwise instead of moving. The big win is that destruction is deterministic: I know exactly when `drop` runs, which is what makes RAII patterns like `MutexGuard` or file handles reliable.

**Follow-up trap:** *"What's the difference between Copy and Clone?"* — `Copy` is an implicit bitwise copy, opt-in via a marker trait, only allowed for types with no `Drop` and no heap ownership. `Clone` is explicit (`.clone()`) and can do arbitrary work, like deep-copying a `Vec`. Every `Copy` type is `Clone`, but not the reverse.

**One-line summary:** One owner, dropped at end of scope, moves transfer ownership — deterministic destruction is the payoff.

---

### Q2. Borrowing rules

**Q:** Explain the borrowing rules and why they exist.

**A:** Rust enforces a single-writer-or-many-readers rule at compile time. At any point in time, for a given piece of data, you can either have one `&mut T` reference or any number of `&T` references — never both. The reason isn't arbitrary; it's how Rust eliminates data races and iterator invalidation without a runtime check. If two parts of the code could mutate the same value, the compiler couldn't reason about aliasing, and you'd get the classic C++ "iterator invalidated by push_back" bugs. With this rule, the compiler can assume `&mut T` is unique, which also unlocks aggressive optimizations — similar to `restrict` in C, but checked.

**Follow-up trap:** *"What about Non-Lexical Lifetimes?"* — Modern Rust ends borrows at last-use, not end-of-scope. So `let r = &v; println!("{}", r); v.push(4);` compiles, because `r` is dead after the print. Pre-NLL Rust would've rejected it.

**One-line summary:** One mutable XOR many shared — eliminates aliasing bugs and unlocks optimizer assumptions.

---

### Q3. Lifetimes

**Q:** When do you need to annotate lifetimes, and what is `'static`?

**A:** Lifetimes are how the compiler tracks how long a reference is valid. You annotate them when the compiler can't figure out the relationship between input and output references on its own. The elision rules cover most function signatures — if there's one input reference, its lifetime is assumed for the output. You need explicit annotations when a function takes multiple references and returns one, or in structs that hold references. `'static` means the reference is valid for the entire program — string literals are `&'static str`, and any owned type with no borrowed data satisfies the `'static` *bound* even though it's not literally a reference. That second part trips people up: `T: 'static` doesn't mean "lives forever," it means "contains no references that could dangle," so `String: 'static` is true.

**Follow-up trap:** *"Does `'static` mean the value lives forever?"* — No. `T: 'static` as a bound means the type doesn't borrow anything with a shorter lifetime. A `String` is `'static` because it owns its buffer. `&'static str` *is* a reference that lives forever, but the bound is broader than that.

**One-line summary:** Lifetimes annotate reference relationships when elision fails; `'static` as a bound means "no non-static borrows inside."

---

### Q4. String vs &str

**Q:** When do you use `String` vs `&str`?

**A:** `&str` is a borrowed string slice — a pointer plus length into someone else's UTF-8 buffer. `String` is an owned, heap-allocated, growable buffer. The rule I follow: take `&str` in function parameters whenever you only need to read, because then callers can pass string literals, `&String`, or slices without allocating. Return `String` when you're producing new data the caller will own. For struct fields, default to `String` — holding a `&str` in a struct forces a lifetime annotation and usually isn't worth the friction unless you specifically need a zero-copy view. The mental model is: `&str` is to `String` what `&[T]` is to `Vec<T>`.

**Follow-up trap:** *"What about `Cow<str>`?"* — Use `Cow<'a, str>` when a function sometimes returns a borrowed slice and sometimes an owned modified string, like normalization that's a no-op for already-clean input. Avoids allocating in the common case.

**One-line summary:** `&str` for params, `String` for fields and returns; `Cow` when the answer is "sometimes borrowed, sometimes owned."

---

### Q5. Option and Result

**Q:** How do you handle `Option` and `Result`, and what does `?` do?

**A:** `Option<T>` is `Some(T)` or `None` — absence of a value. `Result<T, E>` is `Ok(T)` or `Err(E)` — operations that can fail. I match them with `match`, `if let`, or combinator chains like `map`, `and_then`, `unwrap_or`. The `?` operator is the workhorse: on a `Result`, it unwraps `Ok` or early-returns the `Err` from the current function. It also calls `From::from` on the error, so if your function returns `MyError` and you `?` a `std::io::Error`, the compiler inserts the conversion via `impl From<io::Error> for MyError`. That's how you compose errors across layers without explicit `match` everywhere. `?` also works on `Option` if your function returns `Option`.

**Follow-up trap:** *"What if I want to use `?` on `Option` inside a function returning `Result`?"* — Use `.ok_or(err)` to convert. `option.ok_or(MyError::NotFound)?`.

**One-line summary:** `?` early-returns the error and auto-converts via `From` — the entire error-handling story rests on that.

---

### Q6. Traits

**Q:** What are traits, and what's the difference between static and dynamic dispatch?

**A:** Traits are Rust's interface mechanism — they define a set of methods a type must implement. Unlike Go interfaces, trait satisfaction is *explicit*: you write `impl Trait for Type`, you don't get it implicitly by shape. Static dispatch happens with generics: `fn foo<T: Display>(x: T)` is monomorphized — the compiler generates a separate copy for each concrete `T`, so the call is a direct function call, often inlined. Dynamic dispatch happens with trait objects: `fn foo(x: &dyn Display)` — there's a vtable lookup at runtime, like a virtual method in C++. Static is faster and inlinable but inflates binary size; dynamic is one code path with an indirect call, useful when you need heterogeneous collections like `Vec<Box<dyn Shape>>`.

**Follow-up trap:** *"What's the cost of dynamic dispatch in practice?"* — One indirect call per method, plus loss of inlining. For hot loops it matters; for occasional polymorphism it's negligible. Don't reach for `dyn` to "save compile time" unless you've measured.

**One-line summary:** Traits are explicit interfaces; generics = monomorphized fast calls, `dyn` = vtable indirect calls.

---

### Q7. impl Trait vs dyn Trait

**Q:** When do you use `impl Trait` vs `dyn Trait`?

**A:** `impl Trait` in return position means "I return some single concrete type that implements this trait, but I don't want to name it" — common for iterators and closures. It's still static dispatch; the concrete type is fixed at compile time. `dyn Trait` is a trait object — different calls can return different concrete types, but there's a vtable cost. Rule of thumb: `impl Trait` when the type is always the same per call site (especially for `impl Iterator`, `impl Future`); `dyn Trait` when you genuinely need runtime polymorphism, like a plugin system or a heterogeneous collection. In argument position, `fn foo(x: impl Display)` is just syntactic sugar for `fn foo<T: Display>(x: T)`.

**Follow-up trap:** *"Can I return different concrete types from the same function with `impl Trait`?"* — No. Both branches of an `if` must return the same concrete type. If you need different types, you must box: `Box<dyn Trait>`.

**One-line summary:** `impl Trait` is anonymous static dispatch (one concrete type); `dyn Trait` is runtime polymorphism with a vtable.

---

### Q8. Trait objects and object safety

**Q:** Why `Box<dyn Trait>` and not `dyn Trait` directly? What's object safety?

**A:** `dyn Trait` is unsized — its size isn't known at compile time, because any type implementing the trait could be behind it. You can't put unsized types in variables directly; they have to live behind a pointer. So you see `&dyn Trait`, `Box<dyn Trait>`, `Arc<dyn Trait>`. Object safety is the set of rules a trait must satisfy to be used as a trait object: roughly, no generic methods, no `Self` by value in method signatures, no associated constants in method bodies. The reason is the vtable — it can't store an entry for `fn foo<T>()` because there'd be infinitely many `T`s.

**Follow-up trap:** *"How do you make a non-object-safe trait usable as `dyn`?"* — Either split it into a base trait that is object-safe and an extension trait with the generic methods, or use enum dispatch with the `enum_dispatch` crate.

**One-line summary:** `dyn Trait` is unsized so you need indirection; object safety = no generics or `Self`-by-value in methods.

---

### Q9. Closures

**Q:** Walk me through `Fn`, `FnMut`, `FnOnce`, and the `move` keyword.

**A:** Closures are anonymous functions that can capture from their environment. The three traits describe how they capture: `FnOnce` consumes captured values — can only be called once, like a closure that moves a `String` out and drops it. `FnMut` mutably borrows captures — can be called many times, can mutate state. `Fn` immutably borrows captures — callable many times, no mutation. The compiler picks the most permissive trait the body allows. The `move` keyword forces the closure to take ownership of its captures instead of borrowing — essential when you spawn the closure onto a thread or task, because the thread might outlive the original scope. The closure can still be `Fn` after `move` — `move` is about capture mode, not the calling trait.

**Follow-up trap:** *"If I `move` a String into a closure and only read it, is the closure `Fn` or `FnOnce`?"* — `Fn`. `move` decides ownership; trait is decided by usage. Reading `&self` of a captured `String` is fine for `Fn`.

**One-line summary:** `FnOnce` consumes, `FnMut` mutates, `Fn` reads; `move` is about capture mode, not call mode.

---

### Q10. Box vs Rc vs Arc

**Q:** When do you use `Box`, `Rc`, and `Arc`?

**A:** `Box<T>` is single-ownership heap allocation — use it for recursive types, large values you don't want on the stack, or trait objects. `Rc<T>` is reference-counted shared ownership, single-threaded only — clone bumps the count, drop decrements, zero count frees. `Arc<T>` is the atomic version, safe to share across threads — the counter uses atomic operations, so it's slightly slower than `Rc`. Default to `Box`. Reach for `Rc` when you have a graph or tree where multiple parents own a child. Reach for `Arc` when that shared ownership crosses thread boundaries — for example, sharing a config across worker tasks.

**Follow-up trap:** *"Why not just always use `Arc`?"* — Atomic ops are more expensive than non-atomic; `Rc` is measurably faster in single-threaded code. Also, neither gives you mutation by itself — you need `RefCell` or `Mutex` inside.

**One-line summary:** `Box` = unique heap, `Rc` = shared single-thread, `Arc` = shared cross-thread.

---

### Q11. RefCell vs Mutex vs RwLock

**Q:** Explain interior mutability — `RefCell`, `Mutex`, `RwLock`.

**A:** Interior mutability lets you mutate through a shared reference, which normally the borrow checker forbids. `RefCell<T>` moves the borrow check to runtime — `.borrow()` and `.borrow_mut()` panic if you violate the rules, single-threaded only. `Mutex<T>` is the thread-safe version with a lock: `.lock()` blocks until the mutex is free and returns a guard. `RwLock<T>` is like `Mutex` but allows many readers or one writer — use it when reads dominate writes and the critical section is long enough that contention matters. For short critical sections, `Mutex` is usually faster because `RwLock` has more bookkeeping.

**Follow-up trap:** *"When does `RefCell` panic?"* — When you call `.borrow_mut()` while a `borrow()` guard is alive, or vice versa. The error is "already borrowed" — it's a runtime mirror of the compile-time borrow rule.

**One-line summary:** `RefCell` = runtime-checked, single-thread; `Mutex` = exclusive lock; `RwLock` = many-readers-or-one-writer.

---

### Q12. Rc<RefCell> vs Arc<Mutex>

**Q:** Compare `Rc<RefCell<T>>` and `Arc<Mutex<T>>` — when do you use each?

**A:** `Rc<RefCell<T>>` is the single-threaded shared-mutable pattern — graph structures, observer patterns, anywhere you'd reach for shared mutable state inside one thread. Cheap, no atomics, no locks, just refcount and runtime borrow check. `Arc<Mutex<T>>` is the same pattern across threads — workers sharing a counter, async tasks sharing a cache. Atomic refcount plus locking, so it's noticeably more expensive. In the matching engine, for example, I deliberately *avoid* `Arc<Mutex<_>>` on the hot path because the lock acquisition itself blows the latency budget; I use channels instead and let one thread own the book.

**Follow-up trap:** *"What about cycles with `Rc`?"* — `Rc` cycles leak. Break them with `Weak<T>` for the back-pointer — `Weak` doesn't count toward the refcount, so the cycle can be dropped.

**One-line summary:** `Rc<RefCell>` single-thread shared-mutable; `Arc<Mutex>` cross-thread; both leak on cycles — use `Weak` to break them.

---

### Q13. Send and Sync

**Q:** What do `Send` and `Sync` mean? Why is `Rc` not `Send`?

**A:** They're auto-traits the compiler uses to track thread safety. `Send` means a value can be moved to another thread. `Sync` means a value can be shared between threads by reference — equivalently, `&T: Send`. Most types are both. `Rc<T>` is `!Send` because its refcount is non-atomic — if two threads cloned an `Rc` simultaneously, you'd race on the counter and get a use-after-free. `Arc<T>` is `Send + Sync` precisely because the counter is atomic. `RefCell<T>` is `!Sync` because two threads holding `&RefCell` could both call `borrow_mut` and bypass the borrow check, since it isn't thread-safe. The whole point of these traits is the compiler refuses to compile `thread::spawn` with a `!Send` capture.

**Follow-up trap:** *"Can a type be `Sync` but not `Send`?"* — Yes, rare. `MutexGuard` on some platforms is `Sync` (sharing the guard reference is fine) but `!Send` (the OS mutex must be released on the same thread that acquired it).

**One-line summary:** `Send` = movable across threads, `Sync` = shareable by reference; `Rc` is `!Send` because non-atomic refcount.

---

### Q14. Why does this fail to compile?

**Q:** `let mut v = vec![1,2,3]; let r = &v[0]; v.push(4); println!("{}", r);` — why does the borrow checker reject this?

**A:** Because `r` is a shared borrow into `v`'s buffer, and `v.push(4)` needs a `&mut self`. The borrow rule says you can't have a shared borrow alive while you take a mutable borrow. The deeper reason — and this is what interviewers want — is that `push` can reallocate. If `v`'s capacity is full, `push` allocates a new buffer and frees the old one. If `r` were allowed to stay alive across that, it would now be a dangling pointer into freed memory. So the borrow checker isn't just enforcing a syntactic rule; it's catching a real use-after-free. To fix: clone the value into `r`, or restructure so the read happens after the push.

**Follow-up trap:** *"What if you know the Vec won't reallocate?"* — The borrow checker doesn't know that, and isn't supposed to. You can use `unsafe` and raw pointers if you've proven it, but the standard solution is to copy out: `let r = v[0];` (since `i32: Copy`).

**One-line summary:** Shared borrow + `push` = potential use-after-free if reallocation; borrow checker prevents it.

---

### Q15. Iterators

**Q:** Why are Rust iterators "zero-cost"? Walk me through the key adapters.

**A:** Iterators are lazy — `map`, `filter`, `take` don't do any work until you call a consumer like `collect`, `sum`, `fold`, or use a `for` loop. Each adapter wraps the previous one into a new struct implementing `Iterator`. Because everything is generics and the structs are small, the compiler inlines the whole chain and produces code equivalent to a hand-written loop — often byte-identical. That's the "zero-cost abstraction" claim, and you can verify it with `cargo asm` or godbolt. Key adapters: `map` transforms, `filter` keeps matching items, `flat_map` flattens nested iterators, `zip` pairs two iterators, `enumerate` adds indices. Consumers: `collect` into any collection, `fold` for reductions, `sum`/`product`, `any`/`all`.

**Follow-up trap:** *"What does `collect` return into?"* — Any type implementing `FromIterator`. You usually annotate: `let v: Vec<_> = iter.collect()` or `iter.collect::<Vec<_>>()`. `collect::<Result<Vec<_>, _>>()` is a powerful trick: it short-circuits on the first `Err`.

**One-line summary:** Lazy, inlined to hand-loop quality, chained via small generic structs; consumers like `collect` drive the pipeline.

---

### Q16. Error handling (thiserror, anyhow, From)

**Q:** How do you structure errors in Rust — `thiserror`, `anyhow`, custom enums?

**A:** Two patterns depending on whether it's a library or a binary. For libraries, I define a typed error enum and derive `thiserror::Error` — it gives me `Display`, `Error`, and `#[from]` for automatic `From` impls. Callers can match on variants. For binaries or the application layer, I use `anyhow::Result<T>` which is essentially `Result<T, Box<dyn Error + Send + Sync>>` with great ergonomics — `?` works everywhere, you can add context with `.context("doing X")`, and you don't have to define an enum. Rule: `thiserror` at API boundaries where callers need to match; `anyhow` in main, request handlers, and tests where you just want to bubble errors with context. The `From` trait is the glue — once `impl From<IoError> for MyError` exists, `?` on an `io::Result` works inside a function returning `MyError`.

**Follow-up trap:** *"Isn't `anyhow` slower than typed errors?"* — Marginally — it boxes the error. But errors are the cold path; nobody returns errors at 100K/sec. Worry about it after profiling.

**One-line summary:** `thiserror` for library APIs (matchable enums), `anyhow` for app code (context + ergonomic propagation), `From` is the conversion glue for `?`.

---

### Q17. unsafe

**Q:** When do you reach for `unsafe`, and what guarantees do you owe the compiler?

**A:** `unsafe` is needed when you're doing something the type system can't verify — FFI, raw pointer manipulation, calling functions marked `unsafe`, implementing traits like `Send` manually, accessing union fields. You're telling the compiler "I've checked this; trust me." The contract is the list of invariants in the Nomicon: no dangling references, no aliased `&mut`, no reads of uninitialized memory, no data races, no breaking type invariants. The discipline is to encapsulate `unsafe` behind a safe API — a single `unsafe` block inside a function whose signature is safe, with comments explaining *why* the invariants hold. `Vec` is a good example: its internals are full of `unsafe`, but the public API is fully safe.

**Follow-up trap:** *"What's Undefined Behavior in Rust?"* — Same kinds as C: dereferencing null/dangling pointers, data races, unaligned access, breaking aliasing rules with raw pointers. Once UB happens, the program is meaningless — the compiler is allowed to assume it won't, so optimizations can do anything.

**One-line summary:** `unsafe` opts out of compiler checks at specific spots; you owe the Nomicon invariants and should wrap it in a safe API.

---

### Q18. Generics vs trait objects — monomorphization tradeoffs

**Q:** What are the tradeoffs of generics vs trait objects?

**A:** Generics monomorphize — the compiler generates a copy of the function for each concrete type used. Upside: zero dispatch cost, full inlining, max optimization. Downsides: longer compile times, bigger binaries, and every type pollutes the public API. Trait objects don't monomorphize — one copy of the code, indirect calls through a vtable. Upside: small binary, faster compiles, runtime polymorphism. Downside: can't inline across the call, ~one extra indirection per method. In the matching engine I lean generic because hot-path call costs dominate; in plugin or config-loading code I'm fine with `dyn`. Rule of thumb: generic on the hot path, `dyn` everywhere else.

**Follow-up trap:** *"What if I'm getting huge binary blowup from generics?"* — Two patterns: (1) inner-function trick — define a generic outer that calls a non-generic inner doing the actual work, so only the small wrapper monomorphizes; (2) convert the call to `dyn` if it's not hot.

**One-line summary:** Generics = fast + inlined + bigger binary; `dyn` = small + indirect; profile to choose.

---

### Q19. Drop and RAII

**Q:** Explain `Drop` and why deterministic destruction matters.

**A:** `Drop` is the trait whose `drop` method runs when a value goes out of scope. Rust calls it automatically — you don't manually free. This is RAII: resources are tied to object lifetime, so file handles close when the `File` goes out of scope, mutex guards release when the guard drops, allocations free when the `Box` drops. Deterministic destruction matters because it makes resource handling a structural concern, not a discipline concern. In a GC language, you don't know when a file gets closed; in Rust you do, by reading the code. That's also what makes scope-based locking sound: `let g = mutex.lock(); ...; ` — the lock is held exactly until `g` goes out of scope, no chance of forgetting to unlock.

**Follow-up trap:** *"Can you call `drop` manually?"* — Yes, via `std::mem::drop(value)`, which just moves the value into a function that immediately ends its scope. You cannot call `value.drop()` directly — the compiler forbids it to prevent double-drop.

**One-line summary:** Drop runs deterministically at end-of-scope; RAII ties resources to lifetimes, making leaks and forgotten releases structurally hard.

---

### Q20. Panic vs Result

**Q:** When do you panic vs return a `Result`?

**A:** Panic is for *bugs* — invariants the programmer violated, conditions that should never happen in correct code. `Result` is for *expected failures* — things the environment can do to you: I/O errors, parse errors, network failures, missing keys. So: indexing past a `Vec`'s length panics (your bug); `vec.get(i)` returns `Option` (you anticipated absence). In production servers, panics in a worker shouldn't crash the whole process — Tokio catches panics in spawned tasks by default, and you can use `catch_unwind` at boundaries. "Unwind safety" is the concept that some types (like `Mutex`) get poisoned on panic to prevent observing broken invariants. Don't use panic as error handling; it's not faster and it's not idiomatic.

**Follow-up trap:** *"What's `?` vs `unwrap()` in production code?"* — `?` propagates. `unwrap()` panics on `Err`. `unwrap` is fine in tests, examples, and code paths you've proven can't fail; otherwise prefer `?` or `expect("reason")` so the panic message documents the assumption.

**One-line summary:** Panic = bugs / unrecoverable; `Result` = expected failures; `unwrap` is for tests and proofs, not real code.

---

### Bonus Q21. Modules and visibility

**Q:** How does Rust's module and visibility system work?

**A:** Modules form a tree rooted at the crate root (`lib.rs` or `main.rs`). Items are private by default; `pub` exposes them to the parent. `pub(crate)` exposes within the crate only, `pub(super)` to the immediate parent module, `pub(in path)` to a specific ancestor. The `use` keyword brings paths into scope. The big idea: privacy is per-module, not per-type — siblings in the same module can see each other's private fields, which is why "newtype with private field" is a valid encapsulation pattern.

**One-line summary:** Modules form a tree; private by default; `pub(crate)` is the workhorse for crate-internal APIs.

---

### Bonus Q22. Macros — declarative vs procedural

**Q:** Difference between `macro_rules!` and procedural macros?

**A:** `macro_rules!` is pattern-matching on token trees — great for repetitive code, DSL-ish things like `vec!` or `println!`. Procedural macros are Rust programs that run at compile time, take a `TokenStream`, and produce a `TokenStream`. Three kinds: derive macros (`#[derive(Serialize)]`), attribute macros (`#[tokio::main]`), and function-like macros (`sqlx::query!`). Procedural macros are way more powerful but live in a separate crate and have heavier compile cost.

**One-line summary:** `macro_rules!` = token-tree patterns; procedural macros = compile-time Rust programs producing code.

---

## Section 2 — Async Rust + Tokio

---

### Q1. What is a Future?

**Q:** What is a `Future` and what does `.await` actually do?

**A:** A `Future` is a value representing a computation that will eventually produce a result. It's not running by itself — it's inert until polled. The trait has one method, `poll`, which returns `Poll::Ready(value)` or `Poll::Pending`. `.await` is syntactic sugar that calls `poll` in a loop: if the future returns `Pending`, the surrounding async function yields back to the runtime, and registers a `Waker` so the runtime knows when to poll again — typically when a syscall returns, a timer fires, or a channel has a message. When the future returns `Ready`, `.await` extracts the value and execution continues. The key model: futures are state machines the compiler generates from your `async fn` body, and the runtime drives them by calling `poll`.

**Follow-up trap:** *"What happens if you never `.await` a future?"* — Nothing. It's inert. You get a compiler warning ("futures do nothing unless awaited"). This is opposite to JavaScript Promises, which start running immediately.

**One-line summary:** A `Future` is a poll-driven state machine; `.await` suspends until `poll` returns `Ready`, with a `Waker` to resume.

---

### Q2. Pin and self-referential futures

**Q:** Why is `Pin` necessary for futures? Keep it high-level.

**A:** When the compiler converts an `async fn` into a state machine, the generated struct can hold references to its own fields. For example, `let x = String::from("hi"); foo(&x).await;` becomes a struct containing both `x` and a pointer to `x`. If that struct gets moved in memory, the internal pointer would dangle. `Pin<P>` is a wrapper that says "the value behind this pointer will not move again." Once a future is pinned, the runtime can poll it safely knowing its self-references are valid. Most users never write `Pin` manually — `Box::pin(future)` and `tokio::spawn` handle it. The mental model is: pinning is a *promise* about future moves, not a runtime check.

**Follow-up trap:** *"Can all types be pinned?"* — Yes, but only `!Unpin` types actually need the guarantee. Most types implement `Unpin` automatically and can be moved freely even when pinned. Self-referential futures generated by `async` are `!Unpin`.

**One-line summary:** Async state machines can be self-referential; `Pin` promises the value won't move so internal pointers stay valid.

---

### Q3. tokio::spawn — Send + 'static

**Q:** Why does `tokio::spawn` require `Send + 'static`?

**A:** `'static` because the spawned task might outlive the caller — Tokio runs it on its own scheduler, and there's no way to tie its lifetime to a local scope. So the task can't borrow stack data; it must own everything or hold `'static` references. `Send` because Tokio's default scheduler is multi-threaded with work-stealing — a task that started on thread A can be resumed on thread B, so it has to be safe to move across threads. If you want to keep a `!Send` value (like `Rc`) across `.await` points, you need `tokio::task::spawn_local` on a `LocalSet`, which runs on a single thread.

**Follow-up trap:** *"What's a common Send error and how do you fix it?"* — Holding an `Rc` across `.await` makes the future `!Send`. Either swap to `Arc`, or scope the `Rc` so it's dropped before the `.await` — `{ let r = Rc::new(...); use_it(&r); } async_thing.await;`.

**One-line summary:** `'static` because tasks outlive the caller; `Send` because the multi-thread scheduler can migrate tasks.

---

### Q4. std::sync::Mutex across .await

**Q:** When does holding a `std::sync::Mutex` across `.await` cause a deadlock?

**A:** `std::sync::Mutex` is a blocking lock — `.lock()` parks the OS thread until the mutex is free. If you hold its guard across an `.await`, you suspend the task while still holding the lock. The Tokio worker thread is now free to pick up another task — possibly one that also wants this lock. That second task calls `.lock()`, the OS thread blocks, and now the *worker thread itself* is stuck. The first task can't resume because the worker is blocked. Result: deadlock, or at minimum, a starved worker. The fix is `tokio::sync::Mutex`, whose `.lock().await` yields cooperatively. Rule: never hold a blocking lock across `.await`. Even better, restructure so the critical section ends before the await — drop the guard explicitly.

**Follow-up trap:** *"Is `tokio::sync::Mutex` always the right answer?"* — No. It's slower than `std::sync::Mutex` for short non-await critical sections. Prefer `std::sync::Mutex` when the lock is held briefly without awaiting; reach for `tokio::sync::Mutex` only when you must await while holding the lock.

**One-line summary:** `std::Mutex` across `.await` can deadlock the worker; use `tokio::Mutex` for that, or drop the guard before awaiting.

---

### Q5. mpsc — bounded vs unbounded

**Q:** `mpsc::channel` bounded vs unbounded — when do you pick which?

**A:** Bounded almost always. A bounded channel has a fixed capacity; when full, `send` awaits — that's backpressure, automatic. Unbounded channels grow forever; if the producer outpaces the consumer, you OOM. The only time I use unbounded is when I can prove the producer rate is bounded by some external factor — for example, one message per incoming connection, with a connection limit. For everything else, bounded with a tuned capacity. In the matching engine, the order ingress channel is bounded; if it fills, we apply backpressure to the network layer rather than balloon memory.

**Follow-up trap:** *"What's a good capacity?"* — Big enough to absorb burst, small enough that backpressure kicks in before memory matters. Often a few thousand for high-throughput systems. Benchmark.

**One-line summary:** Bounded by default for backpressure; unbounded only when the producer is structurally rate-limited.

---

### Q6. oneshot, broadcast, watch

**Q:** What's each for?

**A:** `oneshot` — send exactly one value from one producer to one consumer; perfect for request/response patterns or signaling completion. `broadcast` — multi-producer, multi-consumer, every consumer sees every message; bounded ring buffer, slow consumers can lag and lose messages. `watch` — single-producer, multi-consumer, consumers only see the *latest* value; ideal for config updates or shutdown signals where intermediate values don't matter.

**One-line summary:** `oneshot` = one-shot RPC; `broadcast` = fan-out with possible lag; `watch` = latest-value pub/sub.

---

### Q7. select! and cancellation safety

**Q:** What does `tokio::select!` do, and what's the cancellation-safety gotcha?

**A:** `select!` races multiple async branches and runs the body of whichever completes first. The crucial thing: the *other* branches are *cancelled* — their futures are dropped mid-execution. The gotcha is cancellation safety: a future is cancellation-safe if dropping it part-way leaves state consistent. Not all futures are — for example, `AsyncReadExt::read` from some implementations isn't cancel-safe because it may have consumed bytes from the kernel buffer before the cancel. The standard pattern is to only put cancel-safe futures in `select!`, or to structure work so the "losing" branch's partial state is recoverable. Tokio's docs label each method's cancellation safety. When unsure, do the work outside `select!` and use channels to surface the result.

**Follow-up trap:** *"Is `recv()` on an `mpsc::Receiver` cancellation-safe?"* — Yes. If it's cancelled, no message is consumed.

**One-line summary:** `select!` races branches and drops losers mid-flight; only use cancel-safe futures, or partial state will leak.

---

### Q8. spawn vs spawn_blocking

**Q:** When do you use `tokio::spawn` vs `tokio::task::spawn_blocking`?

**A:** `spawn` is for async tasks — they suspend at `.await` points and share Tokio's worker threads. `spawn_blocking` is for synchronous, CPU-bound or blocking-I/O work — it runs on a separate, larger thread pool (default 512 threads) so it doesn't starve async workers. Rule: any code that blocks for more than ~10–100µs without yielding belongs in `spawn_blocking` — file I/O without an async wrapper, CPU-heavy hashing or compression, calls into C libraries that block. Putting blocking work on a Tokio worker is the single most common async perf bug: one slow handler stalls every other task scheduled on that worker.

**Follow-up trap:** *"Can I `.await` inside `spawn_blocking`?"* — No, the task is sync. If you need to bridge back, use `tokio::runtime::Handle::current().block_on()` carefully, or send results via a channel.

**One-line summary:** `spawn` for async; `spawn_blocking` for sync/CPU/blocking-I/O — never block a Tokio worker.

---

### Q9. Tokio scheduler

**Q:** How does Tokio's scheduler work at a high level?

**A:** Tokio's multi-threaded runtime is an M:N work-stealing scheduler. M green tasks scheduled across N worker threads, typically one per CPU core. Each worker has a local FIFO run queue plus access to a global queue and the other workers' queues. When a worker's local queue is empty, it steals from a peer. Tasks are state machines; a worker calls `poll` on the task at the head of its queue. If the future returns `Pending`, the task is parked and re-enqueued when its `Waker` is signaled — for example, when an I/O syscall is ready, surfaced via epoll/kqueue/io_uring through Mio. The single-threaded variant (`current_thread`) is simpler — one queue, one thread — useful when you want determinism or `!Send` tasks.

**Follow-up trap:** *"What's a fairness issue with work-stealing?"* — A task that never yields (no `.await`) hogs its worker. Tokio injects yield budgets so cooperatively-scheduled tasks share the worker; CPU-heavy work should call `tokio::task::yield_now().await` periodically or move to `spawn_blocking`.

**One-line summary:** M:N work-stealing across worker threads, each polling task state machines; I/O readiness from epoll/kqueue.

---

### Q10. Backpressure

**Q:** How do you design an async pipeline that doesn't OOM under load?

**A:** Backpressure end-to-end. Every stage uses a bounded channel; when a downstream stage is slow, its input channel fills, and the upstream `send().await` suspends. That suspension propagates back to the network read or the source, which stops accepting new work. The system's throughput becomes pinned to the slowest stage, but no stage's queue grows unbounded. The wrong pattern is unbounded channels or `spawn`-per-message — both let memory grow without bound. In the matching engine, the ingress channel is bounded, so when the engine is in a hot spell, the network layer's `send` blocks, and either the client gets a flow-control signal or the connection back-pressures naturally via TCP.

**Follow-up trap:** *"What if you can't block the source — say, a UDP feed?"* — Then you must drop. Use `try_send` and explicitly drop or sample; record the drop count as a metric.

**One-line summary:** Bounded channels at every stage so slow consumers push back; never `spawn`-per-message or use unbounded queues.

---

### Q11. async fn vs impl Future

**Q:** Difference between `async fn foo() -> T` and `fn foo() -> impl Future<Output = T>`?

**A:** They're nearly equivalent — the `async fn` desugars to a function returning an anonymous `impl Future`. The differences in practice: with `impl Future`, you can write the body synchronously and return a hand-built future, which is useful for combinator-style code or when you need to capture by `move` explicitly. Also, `async fn` in traits had limitations until recently; on stable Rust today, `async fn` in traits works, but for max compatibility you might still see `fn ... -> impl Future + Send` written manually. Functionally though, callers `.await` either the same way.

**One-line summary:** `async fn` desugars to `fn -> impl Future`; same thing, modulo a few historical trait-syntax edges.

---

### Q12. Async deadlock patterns

**Q:** What are common deadlock patterns in async Rust?

**A:** Four common ones. (1) Holding `std::sync::Mutex` across `.await` — the worker blocks, deadlock. (2) Two tasks each waiting on the other's `oneshot` — classic mutual wait, same as threads. (3) Bounded channel where the producer and consumer are the same task — producer `send().await` blocks because the channel is full, but the consumer can't run because the producer is blocked; usually caused by mistakenly making one task do both. (4) `block_on` inside an async context — calling `Handle::block_on` from a Tokio worker tries to drive the runtime from itself and deadlocks. Fix discipline: short critical sections, never block in async, never reenter the runtime.

**One-line summary:** Blocking locks across await, mutual oneshot waits, self-loop on bounded channels, and `block_on` inside async — those are the four.

---

## Section 3 — Matching Engine Deep-Dive

> Speak these in first person, conversational pace. Don't recite — sound like someone who actually built it.

---

### Q1. Walk me through the architecture end-to-end.

**My answer:** At the top, the gateway accepts orders over a network protocol and pushes them into a bounded Tokio mpsc channel. On the other side of that channel is a single dedicated thread — the engine core — which owns the order book. The core runs a loop: pull the next command (new order, cancel, modify) off the channel, apply it deterministically, generate any resulting events — trades, book updates, ack — and push those events into a separate outbound channel. The outbound side is fanned out by a publisher task to subscribers: market data feed, persistence, risk. The key architectural choice is that the engine core is single-threaded and lock-free internally — no `Mutex`, no `Arc<RefCell>` on the hot path. All concurrency happens at the I/O boundary via channels. That gives me predictable tail latency and lets me reason about correctness without worrying about race conditions inside the book.

---

### Q2. Why single-threaded? What are the tradeoffs?

**My answer:** A matching engine has to be deterministic — given the same input sequence, you always produce the same trades. The cleanest way to guarantee that is one thread owning the book, processing one event at a time in order. Sharded multi-threaded engines do exist — typically sharded by symbol — but within a symbol you're back to a single thread, because price-time priority requires a global order on events. The tradeoff I'm accepting is that a single symbol's throughput is capped by one core's speed. I picked single-threaded because at 700K+ orders/sec on one core, I'm not the bottleneck — the network and persistence are. If I needed more, I'd shard *across* symbols, not parallelize *within* one. NASDAQ, LMAX, CME all do something similar — LMAX famously runs the matching engine on a single thread because the determinism win outweighs the parallelism loss.

---

### Q3. Why BTreeMap?

**My answer:** Price levels need to be ordered, and I need fast access to the best price on both sides. `BTreeMap<Price, FifoQueue<Order>>` gives me O(log n) insert at a new price level and O(1) — amortized via `first_entry`/`last_entry` — access to the best bid or ask. A `Vec` would be O(n) for inserting a new price in the middle. A skiplist would have similar asymptotics to `BTreeMap` but worse cache behavior and isn't in std. I'm also cache-friendly: `BTreeMap` is a B-tree, so each node holds many keys contiguously — much better than a binary tree of pointers. For the most common operation, which is *matching at the best price* — not inserting at a new level — I'm O(1) on the price lookup plus FIFO pop from the queue, which is also O(1). New price-level insertion is rarer and O(log n) is fine.

---

### Q4. Walk me through a buy limit order.

**My answer:** Order arrives on the ingress channel. Engine pulls it. First, validation — symbol exists, quantity is positive, price is within bands. Then matching: peek at the best ask. If the order's limit price is greater than or equal to the best ask, we have a match. Pop orders FIFO from the best-ask price level, decrement quantities, emit trade events. Continue walking up the ask side while the incoming order still has quantity and the next best ask is still at or below the limit price. If the incoming order is fully filled, we're done — emit fill event. If there's residual quantity, we insert it into the bid side at its limit price — find or create the `FifoQueue` at that `Price`, push to the back. Then emit any book-update events. Throughout, there are no allocations on the hot path because the price-level queues are pre-allocated FIFOs.

---

### Q5. Allocations on the hot path.

**My answer:** The two natural allocation points are: creating new price levels in the BTreeMap, and pushing into the per-level order queue. I addressed both. The per-level queues are `VecDeque`s with a tuned initial capacity, so steady-state pushes don't reallocate. For brand-new price levels, the `BTreeMap` insert does allocate a node, but new price levels are rare relative to matches at existing levels — most order flow concentrates near the top of book. I also keep orders themselves in a slab allocator — a `Vec<Order>` with free-list indices — so creating an order is `O(1)` index reuse rather than a heap allocation. Events going out of the engine are written into a pre-sized ring buffer and the publisher task reads from it. With those changes, criterion shows zero allocations on a sustained match-only workload, which is what gets us predictable latency.

---

### Q6. Input channel — what if it fills?

**My answer:** The ingress is a bounded `tokio::sync::mpsc` with a tuned capacity — order of low tens of thousands. If it fills, the gateway's `send().await` suspends, which propagates backpressure to the network read loop, which means the kernel TCP receive buffer fills, which means the client sees flow control. That's intentional. The alternative — unbounded queue — would just shift the OOM from the channel to memory writes during a burst. I'd rather have the engine signal "I'm saturated" by slowing acceptance than crash. We also emit a metric on channel occupancy so we can alarm before it saturates.

---

### Q7. Cancels and modifies — where's the order indexed?

**My answer:** Order ID lookup is via a separate `HashMap<OrderId, OrderLocation>` where `OrderLocation` is essentially a pointer into the slab — index plus a tag for side and price. Cancel: hash lookup, mark slot as cancelled, remove from the price-level queue. If the queue uses indices rather than removing from the middle of a `VecDeque`, the cancel is O(1); a naive `VecDeque::remove` would be O(n). I use the slab + linked-list-per-level pattern: each order slot has prev/next indices, so deletion is O(1). Modify is more nuanced — price changes lose priority, so a modify-price is really cancel-plus-new in price-time priority; modify-down-quantity preserves priority and is O(1). I emit a single ack event either way.

---

### Q8. Benchmarking — 728K orders/sec, was it sustained?

**My answer:** Sustained, measured via `criterion`. The harness pre-builds a representative order stream — a mix of new, partial-match, full-match, and cancel — and feeds it through the engine via the same channel path used in production, including the deserialize step. I run on a single isolated core with `taskset`, with hyperthreading disabled to avoid sibling-core noise. I don't claim any NUMA tuning because the workload fits on one socket. The 728K is the median throughput over a 30-second sustained run; bursts hit above 1M when the stream is biased toward matches at the top of book — fewer BTreeMap inserts. The key number for me is actually p99 latency per order, which I keep under a few microseconds.

---

### Q9. Crash recovery.

**My answer:** Event sourcing. Every command that mutates the book is appended to a write-ahead log before it's applied — synchronous fsync on a separate thread, batched by group commit so we don't fsync per order. To recover, replay the log into a fresh engine. To bound recovery time, snapshot the book periodically — serialize the in-memory state, atomically swap the snapshot file, and from then on only need to replay the log tail since the snapshot. There's a tradeoff between snapshot frequency, write amplification, and recovery RTO. I'd target snapshot every N minutes such that worst-case replay is under, say, 30 seconds.

---

### Q10. Scaling to multiple symbols.

**My answer:** Shard by symbol. Each symbol gets its own engine instance with its own thread and its own channels. The gateway routes orders to the right shard by hashing the symbol. Each shard is independent — no cross-symbol matching, which is fine for a CEX. This scales linearly in symbols across cores. Where it gets interesting is cross-symbol features — spread orders, basket orders — those need an extra coordination layer above the per-symbol engines, typically a separate process that decomposes a basket into per-symbol child orders. For pure single-symbol throughput, sharding is the answer; for cross-symbol semantics, you put the logic above the engine, not inside it.

---

### Q11. 10M orders/sec — next bottleneck.

**My answer:** At 10x, my single-thread engine becomes the bottleneck within a symbol; across symbols, the network and serialization layers become the bottleneck before the engines. Within a symbol, I'd push toward LMAX-style techniques: a single thread but with even tighter data layout — struct-of-arrays for orders, cache-line aligned, branchless matching where possible. Beyond that, you'd accept some loss of strict price-time priority within microsecond windows to get parallel pre-validation, which is a product decision, not just an engineering one. On the I/O side: kernel bypass (DPDK, io_uring with submission queue polling) and a binary wire format to keep deserialization cheap. But honestly at that throughput you start measuring power, not just code.

---

### Q12. Market data fanout.

**My answer:** The engine emits events into a single outbound channel — it never knows about subscribers. A separate publisher task consumes that channel and fans out: one path to the persistence layer, one to the market data multiplexer, one to risk. The market data multiplexer uses a `tokio::sync::broadcast` channel with a lagging policy — slow subscribers can fall behind and either lose messages with a gap event, or be disconnected and resnapshotted. The point is: the engine never blocks waiting for subscribers; if a downstream is slow, it's the publisher's queue that backs up, not the engine's. That isolation is critical — one slow consumer must not delay matching.

---

## Section 4 — Coding Curveballs

---

### Problem 1 — Simple price-time-priority order book

```rust
use std::collections::{BTreeMap, VecDeque};

pub type Price = u64;     // fixed-point to avoid float comparisons
pub type Qty = u64;
pub type OrderId = u64;

#[derive(Debug, Clone, Copy)]
pub enum Side { Buy, Sell }

#[derive(Debug, Clone)]
pub struct Order {
    pub id: OrderId,
    pub side: Side,
    pub price: Price,
    pub qty: Qty,
}

#[derive(Debug)]
pub struct Trade {
    pub taker: OrderId,
    pub maker: OrderId,
    pub price: Price,
    pub qty: Qty,
}

pub struct OrderBook {
    // For bids we want descending iteration over keys (highest first).
    // BTreeMap is ascending; we negate or use Reverse. Here we keep raw
    // and use last_key_value() to get the highest bid.
    bids: BTreeMap<Price, VecDeque<Order>>,
    asks: BTreeMap<Price, VecDeque<Order>>,
}

impl OrderBook {
    pub fn new() -> Self {
        Self { bids: BTreeMap::new(), asks: BTreeMap::new() }
    }

    pub fn best_bid(&self) -> Option<Price> {
        self.bids.keys().next_back().copied()
    }

    pub fn best_ask(&self) -> Option<Price> {
        self.asks.keys().next().copied()
    }

    /// Submit a limit order. Returns any trades plus the residual (if any)
    /// that was added to the book.
    pub fn submit(&mut self, mut order: Order) -> Vec<Trade> {
        let mut trades = Vec::new();

        match order.side {
            Side::Buy => self.match_against_asks(&mut order, &mut trades),
            Side::Sell => self.match_against_bids(&mut order, &mut trades),
        }

        if order.qty > 0 {
            let book = match order.side {
                Side::Buy => &mut self.bids,
                Side::Sell => &mut self.asks,
            };
            book.entry(order.price).or_insert_with(VecDeque::new).push_back(order);
        }
        trades
    }

    fn match_against_asks(&mut self, taker: &mut Order, out: &mut Vec<Trade>) {
        // Walk best ask upward while crossable.
        while taker.qty > 0 {
            let best = match self.asks.keys().next().copied() {
                Some(p) if p <= taker.price => p,
                _ => break, // no crossable level
            };
            let queue = self.asks.get_mut(&best).unwrap();
            while taker.qty > 0 {
                let Some(maker) = queue.front_mut() else { break };
                let traded = taker.qty.min(maker.qty);
                out.push(Trade { taker: taker.id, maker: maker.id, price: best, qty: traded });
                taker.qty -= traded;
                maker.qty -= traded;
                if maker.qty == 0 { queue.pop_front(); }
            }
            if queue.is_empty() { self.asks.remove(&best); }
        }
    }

    fn match_against_bids(&mut self, taker: &mut Order, out: &mut Vec<Trade>) {
        while taker.qty > 0 {
            let best = match self.bids.keys().next_back().copied() {
                Some(p) if p >= taker.price => p,
                _ => break,
            };
            let queue = self.bids.get_mut(&best).unwrap();
            while taker.qty > 0 {
                let Some(maker) = queue.front_mut() else { break };
                let traded = taker.qty.min(maker.qty);
                out.push(Trade { taker: taker.id, maker: maker.id, price: best, qty: traded });
                taker.qty -= traded;
                maker.qty -= traded;
                if maker.qty == 0 { queue.pop_front(); }
            }
            if queue.is_empty() { self.bids.remove(&best); }
        }
    }
}
```

**Key things to call out while writing this:**
- `Price` as integer (fixed-point), never `f64` — comparisons must be exact, no NaN, no rounding.
- `VecDeque` for FIFO at each price level — O(1) push_back / pop_front, preserves time priority.
- BTreeMap because we need ordered access to best price; HashMap would be wrong (unordered).
- Residual goes back on the book only if `qty > 0` after matching.
- For production: add an `OrderId -> (Side, Price, slab_index)` index for O(1) cancel.

---

### Problem 2 — Bounded SPSC ring buffer

```rust
use std::cell::UnsafeCell;
use std::mem::MaybeUninit;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

pub struct SpscRing<T> {
    buf: Box<[UnsafeCell<MaybeUninit<T>>]>,
    capacity: usize,                // power of two
    mask: usize,                    // capacity - 1
    head: AtomicUsize,              // producer writes here, consumer reads
    tail: AtomicUsize,              // consumer reads here, producer reads
}

// SAFETY: only one producer and one consumer access the data slots,
// and they never touch the same slot at the same time because head/tail
// gate access via the atomic counters.
unsafe impl<T: Send> Send for SpscRing<T> {}
unsafe impl<T: Send> Sync for SpscRing<T> {}

pub struct Producer<T> { ring: Arc<SpscRing<T>> }
pub struct Consumer<T> { ring: Arc<SpscRing<T>> }

pub fn channel<T>(capacity: usize) -> (Producer<T>, Consumer<T>) {
    assert!(capacity.is_power_of_two(), "capacity must be power of two");
    let buf = (0..capacity)
        .map(|_| UnsafeCell::new(MaybeUninit::uninit()))
        .collect::<Vec<_>>()
        .into_boxed_slice();
    let ring = Arc::new(SpscRing {
        buf,
        capacity,
        mask: capacity - 1,
        head: AtomicUsize::new(0),
        tail: AtomicUsize::new(0),
    });
    (Producer { ring: ring.clone() }, Consumer { ring })
}

impl<T> Producer<T> {
    pub fn push(&self, value: T) -> Result<(), T> {
        let head = self.ring.head.load(Ordering::Relaxed);     // only we write head
        let tail = self.ring.tail.load(Ordering::Acquire);     // see consumer's progress
        if head.wrapping_sub(tail) == self.ring.capacity {
            return Err(value);                                  // full
        }
        let slot = &self.ring.buf[head & self.ring.mask];
        // SAFETY: slot is exclusively ours until we publish head.
        unsafe { (*slot.get()).write(value); }
        self.ring.head.store(head.wrapping_add(1), Ordering::Release); // publish
        Ok(())
    }
}

impl<T> Consumer<T> {
    pub fn pop(&self) -> Option<T> {
        let tail = self.ring.tail.load(Ordering::Relaxed);     // only we write tail
        let head = self.ring.head.load(Ordering::Acquire);     // see producer's writes
        if head == tail {
            return None;                                        // empty
        }
        let slot = &self.ring.buf[tail & self.ring.mask];
        // SAFETY: producer published this slot via Release on head.
        let value = unsafe { (*slot.get()).assume_init_read() };
        self.ring.tail.store(tail.wrapping_add(1), Ordering::Release);
        Some(value)
    }
}
```

**Talking points:**
- **Why power-of-two:** `idx & mask` is faster than `idx % capacity` and works correctly with wrapping arithmetic on `usize`.
- **Memory ordering:**
  - Producer `Release` on `head.store` publishes the slot write to the consumer.
  - Consumer `Acquire` on `head.load` pairs with it — guarantees the slot write is visible.
  - Symmetric on `tail`: consumer's `Release` makes slot-freed visible to producer's `Acquire`.
  - The producer's own `head.load` is `Relaxed` because only the producer writes it (no cross-thread synchronization needed for its own value).
- **Why `UnsafeCell` + `MaybeUninit`:** slots are uninitialized until written; `UnsafeCell` is required for interior mutability through a shared reference.
- **What this avoids:** allocation per message, locks, syscalls. Sub-100ns push/pop on commodity hardware.

---

### Problem 3 — Spot the bug

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

pub async fn process_orders(orders: Arc<Mutex<Vec<Order>>>) {
    let guard = orders.lock().await;
    for order in guard.iter() {
        // pretend this calls an async function
        handle(order).await;
    }
}

async fn handle(_o: &Order) { /* ... */ }
```

**Bug:** Holding the `Mutex` guard across multiple `.await` points serializes the entire workload — every other task that wants the lock waits the full duration of all `handle` calls. Worse, if `handle` itself ever tries to lock the same mutex (directly or transitively), you deadlock. Also, you hold a `&Vec<Order>` borrow across awaits, meaning no one can mutate the vec — even though logically you might only need a snapshot.

**Fix:**

```rust
pub async fn process_orders(orders: Arc<Mutex<Vec<Order>>>) {
    // Take a snapshot under the lock, release the lock immediately.
    let snapshot: Vec<Order> = {
        let guard = orders.lock().await;
        guard.clone()
    };
    for order in &snapshot {
        handle(order).await;
    }
}
```

**Explain it like this in the room:** "The lock is held across awaits, which both kills concurrency and risks deadlock. The fix is to scope the critical section to just the read, drop the guard at the brace, and do the async work on the snapshot. If cloning is too expensive, swap to a lock-free design — like the channel-based pattern I used in the matching engine."

---

## Section 5 — STAR Behavioral

---

### Story 1 — Hardest Rust bug

**Pick:** Use a Rust matching-engine bug, not the Go/PolyBFT HasQuorum bug. Reason: the interview is Rust-focused; a Rust bug demonstrates Rust depth. Save HasQuorum for "tell me about your highest-impact work."

**S** — While benchmarking the matching engine, I saw p99 latency spike every few seconds even though average throughput was fine. Median was a few microseconds; p99 was hitting hundreds.

**T** — Find the source of the periodic jitter and bring p99 down to the predictable-tail-latency budget we'd promised.

**A** — I instrumented the hot path with timestamp samples around every major step and dumped them when an outlier fired. The spikes correlated with new price levels being inserted into the `BTreeMap` — specifically, the node allocations. I confirmed with `perf` and the `dhat` allocator profiler that the allocations during `BTreeMap` growth were the culprit, amplified by the fact that the system allocator's behavior under our pattern was occasionally slow. I moved orders into a slab — a pre-sized `Vec` with free-list indices — and replaced the per-level `VecDeque` with a doubly-linked list of slab indices. After that, only brand-new price-level creation hit `BTreeMap`, and the per-order work was allocation-free.

**R** — p99 dropped roughly an order of magnitude, sustained 700K+ orders/sec became achievable, and the criterion harness now asserts zero allocations on the steady-state path so we catch regressions.

---

### Story 2 — Why this architecture

**S** — We needed a CEX matching engine that could sustain high throughput with deterministic, microsecond-scale tail latency.

**T** — Design the concurrency model.

**A** — I evaluated three options: (1) lock-based shared engine with worker threads, (2) actor model with one engine actor per symbol, (3) classic LMAX-style single-threaded engine with channels at the I/O boundary. I went with option 3. Locks add unpredictability — lock contention bursts produce exactly the kind of tail-latency spikes we couldn't accept. Multi-actor per symbol shards nicely, but within a symbol you're back to one writer for determinism, so the per-symbol design is still single-threaded. The deciding factor was that LMAX disruptor and ITCH-style engines have proven this pattern works at far higher throughputs than we needed — there's no reason to take on the complexity of internal parallelism.

**R** — We hit 728K orders/sec sustained on a single core with predictable p99, and the codebase is dramatically simpler than the lock-based prototype I started with.

---

### Story 3 — Learned something fast

**S** — When I joined the Hydragon project I had to ramp on PolyBFT internals — a consensus protocol I'd not worked on before — and ship a validator slashing module within months.

**T** — Understand the consensus state machine well enough to safely insert slashing logic without breaking liveness or safety.

**A** — I started by reading the existing IBFT/PolyBFT code top-down: round changes, proposal validation, commit aggregation. I instrumented a local 4-node devnet and traced what happens to a validator during a double-sign. I read the Polygon Edge papers and the IBFT spec. Then I wrote the slashing detection as a pure function with unit tests before touching any consensus code, so I could verify the logic in isolation. Only then did I wire it into the system-transaction layer.

**R** — Five merged PRs, slashing module shipped, and as a side benefit I caught the `HasQuorum` quorum-math bug that was stalling small validator sets at 100% — fixed it because by then I understood the math well enough to spot it.

---

### Story 4 — Tradeoff I'd reconsider

**S** — Early in the matching engine I chose a `BTreeMap<Price, VecDeque<Order>>` for price levels.

**T** — Get to a working order book quickly so we could start benchmarking and tuning.

**A** — `BTreeMap` was the fastest way to get correct behavior on day one. As load increased, I found that `VecDeque::remove` for mid-queue cancels was O(n), which mattered for high-cancel-rate market makers. I migrated to a slab-plus-linked-list per price level for O(1) cancel.

**R** — Cancel latency dropped to constant time. Looking back, I'd probably have started with the slab design from day one — the BTreeMap part was right, but the `VecDeque` was a "good enough" choice that I had to redo. The lesson is to look one query pattern ahead, not just at insert/match. That said, I don't regret shipping the simple version first; it let us start benchmarking weeks earlier, and the eventual rewrite was informed by real data.

---

## Section 6 — Tomorrow Morning Checklist

Top 10 highest-leverage items to re-skim. Do these in this order.

1. **The three ownership rules + Copy vs Clone.** Be able to recite verbatim in 30 seconds.
2. **Borrowing rules + the `vec.push` invalidation example.** This question lands in 50% of Rust interviews.
3. **Send/Sync and why Rc is !Send.** Atomic vs non-atomic refcount — one-sentence answer.
4. **std::sync::Mutex vs tokio::sync::Mutex across .await.** The single most-asked async question.
5. **tokio::spawn requires Send + 'static — why.** Two sentences each, separately.
6. **Bounded vs unbounded mpsc + backpressure.** Always pick bounded; explain why.
7. **Your matching engine architecture diagram in your head.** Ingress channel → single-threaded core → outbound channel → fanout. Be able to draw it on a whiteboard in 60 seconds.
8. **Why BTreeMap, why single-threaded, where allocations were eliminated.** These three are the heart of your project section.
9. **The order-book code in Section 4 Problem 1.** Read it twice; you must be able to whiteboard the `submit` function.
10. **The SPSC memory ordering reasoning.** "Release publishes the slot write, Acquire on the other side sees it." If they probe atomics, this answer alone shows depth.

**Things to say if stuck:**
- "Let me think out loud about that…" (interviewers love this; silence is the enemy)
- "I haven't used that exact API, but I'd expect it works like X because…" (shows reasoning even without recall)
- "Honestly I'd reach for the docs / `cargo expand` to confirm — but my mental model is…"

**Things NOT to say:**
- "Rust is hard." (you're a Rust engineer — own it)
- "I just use `unwrap` everywhere." (even joking, no)
- "I don't really know async." (if asked something you don't know, narrow the scope: "I haven't used `select!` heavily, but I understand the cancellation model — here's how I'd reason about it…")
- "It's basically like Go." (interviewers in Rust shops hate this comparison)

**Last 30 minutes before the call:**
- Glass of water beside you, mute notifications.
- Do a 5-minute warm-up: explain ownership and borrow rules out loud to yourself.
- Have the architecture diagram open on a second screen.
- Breathe. You've shipped this stuff in production — you're not bluffing, you're recalling.

Good luck. You've got this.
