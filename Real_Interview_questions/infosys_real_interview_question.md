# Infosys — Real Interview Questions (Actually Asked)

Is file mein wo Rust questions track kiye jaate hain jo **Infosys ke actual interview** mein puche gaye. Ye interview code-heavy tha — Rust Playground par live code debug karwaya gaya aur Teams chat mein snippet bheje gaye. Neeche har question ka detailed + interview-ready answer hai.

**Company/Round:** Infosys · Rust Backend/Systems · Live coding (Rust Playground + Teams)
**Date:** 2026-07-17

---

## Index

| # | Topic | Link |
|---|-------|------|
| Q1 | Move semantics bug — `process(s)` used after move | [Jump](#q1-move-semantics-bug--use-of-moved-value) |
| Q2 | Concurrency + lifetime bug — `Rc/RefCell` in `tokio::spawn`, `find_longest`, `Cache<T>` | [Jump](#q2-concurrency--lifetime-bug--rcrefcell-in-tokiospawn) |
| Q3 | `#[repr(C)]`, `#[repr(packed)]`, `#[repr(align(N))]` | [Jump](#q3-repr-attributes--reprc-reprpacked-repralign) |
| Q4 | Inline assembly optimization (`asm!`) | [Jump](#q4-inline-assembly-optimization-asm) |
| Q5 | How compiler generates a state machine for `async fn` | [Jump](#q5-async--how-the-compiler-generates-a-state-machine) |
| Q6 | `panic!` — unwind vs abort | [Jump](#q6-panic--unwind-vs-abort) |
| Q7 | Rollback an individual transaction (savepoints) | [Jump](#q7-rollback-an-individual-transaction) |
| Q8 | Increment a lock-free data structure (CAS / FAA) | [Jump](#q8-increment-a-lock-free-data-structure) |

---

## Q1. Move semantics bug — "use of moved value"

**Interviewer ne ye code diya (Teams chat mein):**

```rust
fn process(name: String) {
    println!("{name}");
}

fn main() {
    let s = String::from("Rust");
    process(s);          // ownership of `s` MOVED here
    println!("{s}");     // ❌ ERROR: borrow of moved value: `s`
}
```

**Issue kya hai?**
- `process(name: String)` argument ko **by value** leta hai → poori ownership function mein chali jaati hai.
- `process(s)` call pe `s` ka data `process` ke andar **move** ho gaya. Function ke end pe `name` scope se bahar → value **drop/deallocate**.
- Ab `main` mein `println!("{s}")` ek **moved (dead) value** ko access kar raha hai → compile error: `borrow of moved value: s`.

**Fix 1 — Pass by reference (idiomatic, best):** ownership diye bina borrow karo.

```rust
fn process(name: &str) {     // &str reference leta hai
    println!("{name}");
}

fn main() {
    let s = String::from("Rust");
    process(&s);             // reference pass — no move
    println!("{s}");         // ✅ safe, `s` abhi bhi owner hai
}
```

**Fix 2 — Clone (jab function ko sach mein owned `String` chahiye):**

```rust
fn process(name: String) {
    println!("{name}");
}

fn main() {
    let s = String::from("Rust");
    process(s.clone());      // deep copy pass, original intact
    println!("{s}");         // ✅ safe
}
```

**Interview one-liner:** "`process(s)` moves ownership into the function where it's dropped at scope end; using `s` afterwards is use-of-moved-value. Fix: borrow with `&s` (idiomatic), ya `s.clone()` agar function ko ownership chahiye."

---

## Q2. Concurrency + lifetime bug — `Rc/RefCell` in `tokio::spawn`

**Interviewer ka buggy code (Rust Playground par, edition 2024):**

```rust
use std::collections::HashMap;
use std::rc::Rc;
use std::cell::RefCell;

struct Cache<T> {
    data: HashMap<String, T>,
    access_count: RefCell<HashMap<String, usize>>,
}

impl<T: Clone> Cache<T> {
    fn new() -> Self {
        Cache {
            data: HashMap::new(),
            access_count: RefCell::new(HashMap::new()),
        }
    }

    fn get(&self, key: &str) -> Option<T> {
        let mut counts = self.access_count.borrow_mut();
        *counts.entry(key.to_string()).or_insert(0) += 1;
        self.data.get(key).cloned()
    }

    fn insert(&mut self, key: String, value: T) {
        self.data.insert(key, value);
    }
}

async fn process_items(items: Vec<String>) -> Vec<String> {
    let cache = Rc::new(RefCell::new(Cache::new()));   // ❌ Rc/RefCell not Send
    let mut handles = vec![];

    for item in items {
        let cache_clone = Rc::clone(&cache);
        let handle = tokio::spawn(async move {         // ❌ needs Send
            let mut cache = cache_clone.borrow_mut();
            cache.insert(item.clone(), item.len());
            item.to_uppercase()
        });
        handles.push(handle);
    }

    let mut results = vec![];
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// ❌ lifetime bug: y is not tied to 'a
fn find_longest<'a>(x: &'a str, y: &str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Teen bugs the

**Bug 1 — `Rc<RefCell<T>>` ko `tokio::spawn` mein move karna (concurrency violation).**
- `tokio::spawn` ka future **`Send`** hona zaroori hai, kyunki task multi-threaded runtime par kisi bhi thread par schedule ho sakta hai.
- `Rc` reference count **non-atomically** track karta hai aur `RefCell` thread-safe nahi → dono **`!Send` + `!Sync`**. Compiler reject kar deta hai.
- **Fix:** `Rc<RefCell<T>>` → **`Arc<Mutex<T>>`**. `Arc` = atomic reference counting, `Mutex` = thread-safe interior mutability.

**Bug 2 — `find_longest` lifetime mismatch.**
- Signature `fn find_longest<'a>(x: &'a str, y: &str) -> &'a str` mein sirf `x` aur return `'a` se bandhe hain. Agar function `y` return kare, compiler guarantee nahi kar sakta ki `y` utni der jeeta hai.
- **Fix:** dono inputs ko same lifetime do → `fn find_longest<'a>(x: &'a str, y: &'a str) -> &'a str`.

**Bug 3 — `Cache<T>` generic `Clone` bound + borrow consistency.**
- `get` mein `.cloned()` tabhi chalega jab `T: Clone` ho — struct/impl par bound chahiye.
- `get(&self)` interior mutability se `access_count` ko mutate kar raha tha (`RefCell`) — clean karke `&mut self` bana do ya external `Mutex` ke through mutate karo.

### Fully corrected code

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Debug)]
struct Cache<T> {
    data: HashMap<String, T>,
    access_count: HashMap<String, usize>,
}

impl<T: Clone> Cache<T> {          // Fix 3: T: Clone bound
    fn new() -> Self {
        Cache {
            data: HashMap::new(),
            access_count: HashMap::new(),
        }
    }

    fn get(&mut self, key: &str) -> Option<T> {   // &mut self, no RefCell needed
        let count = self.access_count.entry(key.to_string()).or_insert(0);
        *count += 1;
        self.data.get(key).cloned()
    }

    fn insert(&mut self, key: String, value: T) {
        self.data.insert(key, value);
    }
}

// Fix 1: Arc<Mutex> so the future is Send + Sync
async fn process_items(items: Vec<String>) -> Vec<String> {
    let cache = Arc::new(Mutex::new(Cache::<usize>::new()));
    let mut handles = vec![];

    for item in items {
        let cache_clone = Arc::clone(&cache);
        let handle = tokio::spawn(async move {
            let mut guard = cache_clone.lock().unwrap();
            let len = item.len();
            guard.insert(item.clone(), len);
            item.to_uppercase()
        });
        handles.push(handle);
    }

    let mut results = vec![];
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// Fix 2: both inputs share output lifetime 'a
fn find_longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

#[tokio::main]
async fn main() {
    let mut cache = Cache::new();
    cache.insert("key1".to_string(), 42);
    let value = cache.get("key1");
    cache.insert("key2".to_string(), 100);
    println!("Value: {:?}", value);          // Value: Some(42)

    let s1 = String::from("long string");
    let s2 = "short";
    let result = find_longest(&s1, s2);
    println!("Longest: {}", result);         // Longest: long string

    let items = vec!["rust".to_string(), "tokio".to_string()];
    let upper = process_items(items).await;
    println!("Processed: {:?}", upper);      // Processed: ["RUST", "TOKIO"]
}
```

**Interview one-liner:** "Original code `Rc`/`RefCell` ko `tokio::spawn` mein move kar raha tha — ye `!Send` hain isliye multi-threaded runtime pe compiler reject karta hai; `Arc<Mutex<T>>` se fix. Plus `find_longest` mein dono inputs ko same lifetime `'a` se bandhna pada taaki returned reference valid rahe, aur `Cache<T>` par `T: Clone` bound zaroori tha."

---

## Q3. `repr` attributes — `#[repr(C)]`, `#[repr(packed)]`, `#[repr(align(N))]`

Ye attributes compiler ko batate hain ki struct/enum ko memory mein **kaise lay out** karna hai. FFI, hardware, aur network protocols ke liye critical.

### 1. `#[repr(C)]` — Compatibility layout
- **Default Rust:** compiler fields ko reorder kar sakta hai (padding minimize karne ke liye) → layout **unstable**, C ko safely pass nahi kar sakte.
- **`#[repr(C)]`:** fields ko **exactly declared order** mein, C compiler ke alignment rules ke saath layout karta hai → **stable, predictable**.
- **Use:** FFI (C libraries ke saath data pass karna), network packets, file formats.

```rust
#[repr(C)]
struct MyCStruct {
    a: u8,      // 1 byte
    // 3 bytes padding here (align `b` to 4-byte boundary)
    b: u32,     // 4 bytes
}   // total 8 bytes
```

### 2. `#[repr(packed)]` — Zero-padding layout
- Saara alignment padding **strip** kar deta hai → fields tightly squeezed.
- **Use:** memory tight ho, hardware registers, network headers, binary disk structures jahan har byte matter karta hai.
- ⚠️ **Warning:** packed struct ke fields access karna **unaligned memory access** cause kar sakta hai (performance penalty ya kuch architectures par crash). Isliye packed field ka **reference lena `unsafe`** hai.

```rust
#[repr(packed)]
struct PackedStruct {
    a: u8,      // 1 byte
    b: u32,     // 4 bytes — starts at index 1, NO padding
}   // total 5 bytes (instead of 8)
```

### 3. `#[repr(align(N))]` — Custom alignment
- Struct ke starting address ko **N ka multiple** force karta hai (N = power of 2: 2, 4, 8, 64...).
- **Use:** CPU **cache-line alignment** (typically 64 bytes) taaki multi-threaded code mein **false sharing** na ho; SIMD / DMA alignment requirements.

```rust
#[repr(align(64))]
struct CacheLineAligned {
    data: [u8; 10],
}   // data sirf 10 bytes, but total size padded to 64, address divisible by 64
```

### Summary table

| Attribute | Field ordering | Padding | Core use case |
|-----------|----------------|---------|---------------|
| `#[repr(C)]` | Preserved (C-like) | Standard C padding | FFI / interop with C |
| `#[repr(packed)]` | Preserved | None (squeezed) | Hardware regs / network protocols |
| `#[repr(align(N))]` | Preserved | Forces min alignment to N | SIMD / cache-line alignment |

**Interview one-liner:** "`repr(C)` = stable C-compatible layout for FFI; `repr(packed)` = all padding removed to save bytes (but unaligned access risk, `&` to fields is unsafe); `repr(align(N))` = force address to an N-byte boundary, e.g. cache-line align to avoid false sharing."

---

## Q4. Inline assembly optimization (`asm!`)

Inline assembly (`core::arch::asm!` / `std::arch::asm!`) se hum architecture-specific CPU instructions **directly** Rust functions mein likh sakte hain, jab compiler ka auto-vectorization / optimization enough nahi hota.

### Anatomy of an optimized `asm!` block

```rust
use std::arch::asm;

pub fn fast_add_scaled(a: u64, b: u64) -> u64 {
    let mut result;
    unsafe {
        asm!(
            // 1. Instruction: result = a + b*4 in one LEA
            "lea {res}, [{x} + {y} * 4]",

            // 2. Bind registers to Rust variables
            x = in(reg) a,
            y = in(reg) b,
            res = lateout(reg) result,

            // 3. Tell compiler about side effects
            options(nostack, pure, readonly),
        );
    }
    result
}
```

### Key optimization strategies

**A. Architecture-specific instructions** — compiler jo miss karta hai:
- **Bit manipulation:** `LZCNT` (leading-zero count), `POPCNT` (population count), `BSR` — poore bitwise loops ko ek instruction mein replace.
- **Math hacks:** `LEA` (Load Effective Address) x86 par addition + shift-multiply ek hi cycle mein.

**B. `options(...)` — compiler ko freedom do** (biggest lever):
- `nostack` — assembly stack use nahi karti → compiler stack frame setup skip kar sakta hai.
- `pure` — block ke side effects nahi, output sirf inputs par depend → result use na ho to **dead-code elimination** ho sakta hai.
- `readonly` — assembly sirf memory read karti hai → compiler variables ko registers mein cache rakh sakta hai (RAM se reload nahi).

**C. `lateout` for register reuse:**
- `out(reg)` compiler ko naya register maangne bolta hai.
- `lateout(reg)` = "output compute karne ke baad main input registers nahi padhunga" → compiler **input register ko output ke liye reuse** kar sakta hai → register pressure kam.

### Safety guardrails (warna de-optimization ya corruption)
- **Stack pointer manually kabhi mat change karo** (`RSP`/`ESP`/`EBP`) — Rust ki unwinding aur local variable referencing toot jaayegi.
- **Flags clobber flag karo** — agar assembly CPU status flags (zero/carry) modify karti hai to `options(preserves_flags)` sahi lagao (ya lagao mat, taaki compiler ko pata rahe).
- **Prefer `asm!` over external `.s` + FFI** — inline assembly LLVM directly caller function mein inline kar sakta hai → function-call overhead **zero**.

**Interview one-liner:** "`asm!` se specialized instructions (`LZCNT`, `LEA`) invoke karke compiler ki limitations bypass karte hain. `options` (`nostack`, `pure`, `readonly`) + `lateout` register reuse se max performance milti hai, aur LLVM ise caller mein inline karke call overhead khatam kar deta hai."

---

## Q5. `async` — how the compiler generates a state machine

Jab tum `async fn` ya `async` block likhte ho, compiler use ek **zero-cost state machine** mein transform kar deta hai jo `Future` trait implement karti hai.

### 1. Core transformation: `async` → `Future`

```rust
async fn fetch_data() -> String {
    let raw = download().await;     // yield point 1
    let parsed = parse(raw).await;  // yield point 2
    parsed
}
```

Compiler ise ek synchronous function bana deta hai jo anonymous `Future` return karta hai:

```rust
fn fetch_data() -> impl Future<Output = String> {
    // compiler-generated state machine struct
    FetchDataStateMachine::new()
}
```

### 2. State struct generation
Compiler ek hidden enum/struct banata hai jo store karta hai:
1. **Current execution state** (function abhi kahan paused hai).
2. Koi bhi **local variable jo `.await` boundaries ke across persist** karna hai.

```rust
enum FetchDataStateMachine {
    Start,
    WaitingOnDownload { download_future: DownloadFuture },
    WaitingOnParse    { parse_future: ParseFuture },
    Done,
}
```

### 3. `Future::poll` — execution driver
Poll ke andar ek **giant `match`** state ko aage badhata hai:

```rust
impl Future for FetchDataStateMachine {
    type Output = String;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            match *self {
                State::Start => {
                    let fut = download();
                    *self = State::WaitingOnDownload { download_future: fut };
                }
                State::WaitingOnDownload { ref mut download_future } => {
                    match Pin::new(download_future).poll(cx) {
                        Poll::Ready(raw) => {
                            let fut = parse(raw);
                            *self = State::WaitingOnParse { parse_future: fut };
                        }
                        Poll::Pending => return Poll::Pending,   // yield to executor
                    }
                }
                State::WaitingOnParse { ref mut parse_future } => {
                    match Pin::new(parse_future).poll(cx) {
                        Poll::Ready(parsed) => {
                            *self = State::Done;
                            return Poll::Ready(parsed);
                        }
                        Poll::Pending => return Poll::Pending,   // yield to executor
                    }
                }
                State::Done => panic!("Future polled after completion"),
            }
        }
    }
}
```

### 4. Key optimizations
- **Zero-cost & memory footprint:** layout compile-time par decide hota hai. Allocated memory = **largest state variant ka size**. Default **stack-allocated** (unless `Box::pin`). JS/C# ki tarah heap par dynamically allocate nahi hota.
- **`Pin` kyun zaroori hai:** local variables `.await` ke across borrow ho sakte hain → state machine **self-referential pointers** rakh sakti hai. Agar struct memory mein move ho jaaye to ye internal pointers toot jaayenge. `Pin` guarantee karta hai ki state machine apni fixed memory address par rahegi.

**Interview one-liner:** "Compiler `async fn` ko ek zero-cost, stack-allocated state-machine enum mein badal deta hai — variants un locals ko store karte hain jo `.await` ke across zinda rehte hain. Executor `Future::poll` call karta hai; ek giant `match` states ke beech transition karta hai. `Pin` isliye chahiye kyunki state machine self-referential ho sakti hai aur move hone par pointers break ho jaate."

---

## Q6. `panic!` — unwind vs abort

`panic!` ek **unrecoverable error** signal karta hai. Cleanup ke do strategies hain: **unwinding** (default) aur **aborting**.

### 1. Unwinding (default)
- Runtime call stack ko **backward walk** karta hai, frame by frame.
- Har live object ke liye **destructor (`Drop`)** call karta hai → heap memory, file handles, sockets cleanly free.
- **Catchable:** `std::panic::catch_unwind` se panic higher level pe intercept ho sakta hai — web servers (ek failing request poore server ko crash na kare) aur test runners isko use karte hain.
- **Trade-off:** binary mein extra **unwind tables** → binary size badi.

### 2. Aborting (`panic = "abort"` in `Cargo.toml`)
- Panic hote hi execution **turant stop**, OS poore process ko instantly terminate karta hai.
- **Koi `Drop` nahi chalta** — OS raw memory reclaim karta hai.
- **Uncatchable:** `catch_unwind` intercept nahi kar sakta (koi stack traversal hi nahi).
- **Trade-off:** unwind tables nahi chahiye → **smaller binary, faster compile**. Embedded, low-memory, Wasm ke liye ideal.

### Quick comparison

| Feature | Unwinding (`panic = "unwind"`) | Aborting (`panic = "abort"`) |
|---------|-------------------------------|------------------------------|
| Destructors (`Drop`) | Sabhi stack objects ke liye chalte hain | Skipped completely |
| Process termination | Sirf panicking **thread** | **Poora process** instantly |
| Recoverability | `catch_unwind` se catchable | Uncatchable |
| Binary size | Bada (unwind tables) | Chhota, optimized |
| Best for | General apps, web servers, tests | Embedded, low-memory, Wasm |

**Interview one-liner:** "Unwinding (default) stack ko backward walk karke `Drop` destructors chalata hai aur `catch_unwind` se catchable hai. Aborting cleanup skip karke OS ke through poore process ko instantly maar deta hai — uncatchable, par smaller binary + faster compile."

---

## Q7. Rollback an individual transaction

**Question:** poore session/transaction ko abort kiye bina ek **individual (partial) transaction** kaise rollback karein?

**Answer — Savepoints:**
- Transaction block ke andar ek checkpoint rakho: `SAVEPOINT name;`
- Error aane par: `ROLLBACK TO name;` — ye sirf **us savepoint ke baad wali operations** ko undo karta hai; baaki transaction valid rehta hai aur continue kar sakta hai.
- Successfully done ho to `RELEASE SAVEPOINT name;` se savepoint clear kar do.

```sql
BEGIN;
    INSERT INTO accounts (id, bal) VALUES (1, 100);

    SAVEPOINT sp1;                         -- checkpoint
    UPDATE accounts SET bal = bal - 50 WHERE id = 1;
    -- kuch galat hua:
    ROLLBACK TO sp1;                       -- sirf UPDATE undo, INSERT bacha rehta hai

    INSERT INTO accounts (id, bal) VALUES (2, 200);
COMMIT;                                    -- INSERTs commit ho jaate hain
```

**Interview one-liner:** "Individual transaction rollback ke liye **savepoints** use karo — `SAVEPOINT name` se checkpoint, error par `ROLLBACK TO name` jo sirf us savepoint ke baad ka kaam undo karta hai, baaki transaction intact rehta hai."

---

## Q8. Increment a lock-free data structure

**Question:** lock-free data structure mein value **atomically increment** kaise karte ho (bina `Mutex`)?

Do primary techniques — dono lock-free hain kyunki hardware-level atomic instructions par depend karti hain:

### 1. Fetch-and-Add (FAA) — simple counter ke liye best
- CPU ek **single atomic read-modify-write instruction** mein value increment karta hai — koi loop overhead nahi.
- Rust: `AtomicUsize::fetch_add`.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

fn bump() {
    // atomic +1, previous value return karta hai
    COUNTER.fetch_add(1, Ordering::SeqCst);
}
```

### 2. Compare-And-Swap (CAS) loop — complex/conditional update ke liye
- Jab increment kisi **logic par depend** kare, to loop:
  1. current value read karo,
  2. new value compute karo,
  3. `compare_exchange` se write **tabhi karo agar value change nahi hui**;
  4. agar beech mein kisi aur thread ne modify kar diya → loop **retry**.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn increment_if_even(counter: &AtomicUsize) {
    let mut current = counter.load(Ordering::Acquire);
    loop {
        if current % 2 != 0 {
            return;                              // condition fail — koi update nahi
        }
        let new = current + 1;
        match counter.compare_exchange(
            current, new,
            Ordering::AcqRel, Ordering::Acquire,
        ) {
            Ok(_) => break,                      // success
            Err(actual) => current = actual,     // koi aur thread jeet gaya → retry
        }
    }
}
```

**Interview one-liner:** "Lock-free increment do tareeke se: simple counter par **Fetch-and-Add** (`fetch_add`) — ek atomic instruction, no loop. Conditional/complex update par **CAS loop** (`compare_exchange`) — read → compute → swap-only-if-unchanged, warna retry. Dono `Mutex` ke bina hardware atomics use karte hain."

---
