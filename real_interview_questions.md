# Real Interview Questions (Actually Asked)

Is file mein wo Rust interview questions track kiye jaate hain jo **actual interviews** mein puche gaye — date, company/round, aur detailed answer ke saath. Naye questions niche append karte raho.

---

## Index

| Date | Topics Covered | Link |
|------|----------------|------|
| 2026-06-04 | Drop order, ownership/borrowing, lifetimes, traits, String vs &str, futures, async, heap/Box, smart pointers, channels, Cell, error handling, Result vs Option, thread poisoning | [Jump](#interview--2026-06-04) |

---

## Interview — 2026-06-04

Topics: Drop trait ordering, ownership & borrowing, lifetimes, traits, `String` vs `&str`, futures, async programming, stack→heap, smart pointers, channels, `Cell`, error handling, `Result` vs `Option`, thread poisoning.

---

### Q1. Drop order — predict the output

```rust
use std::mem;

struct A(u8);

impl Drop for A {
    fn drop(&mut self) {
        println!("Dropping {}", self.0);
    }
}

fn main() {
    let a1 = A(3);
    let a2 = A(2);
    let a3 = A(1);

    {
        mem::drop(a3);
        mem::drop(a1);
    }
    mem::drop(a2);
}
```

**Output:**
```
Dropping 1
Dropping 3
Dropping 2
```

**Explanation:**
- `mem::drop(x)` value ki **ownership le leta hai** aur usse **wahin turant drop** kar deta hai (immediately, line par hi). Ye scope-end ka wait nahi karta.
- Isliye execution order exactly likhe gaye order mein chalti hai:
  1. `mem::drop(a3)` → `a3` has value `1` → prints `Dropping 1`
  2. `mem::drop(a1)` → `a1` has value `3` → prints `Dropping 3`
  3. `mem::drop(a2)` → `a2` has value `2` → prints `Dropping 2`
- **Trap**: Normally agar tum `mem::drop` na karte, to Rust variables ko **reverse declaration order** (LIFO — stack jaise) mein drop karta: `a3`(1), `a2`(2), `a1`(3). Yahan explicit `mem::drop` calls ne wo natural order override kar diya.
- `a1` aur `a3` already moved-and-dropped ho chuke hain, isliye scope/main end pe unhe dobara drop nahi kiya jaata (double-free nahi hota — Rust ka move semantics ensure karta hai).
- Inner `{ }` block yahan output order ko change nahi karta kyunki dono drops explicit hain.

---

### Q2. Ownership & Borrowing — append "hello" → "hello world" without a new variable, pass `&mut`

Bina koi naya variable banaye, existing `String` ko mutate karke append karo. `push_str` ek **mutable reference** (`&mut self`) leta hai.

```rust
fn append_world(s: &mut String) {   // mutable reference pass kiya
    s.push_str(" world");           // in-place mutate, no new variable
}

fn main() {
    let mut hello = String::from("hello");
    append_world(&mut hello);       // &mut se mutable borrow diya
    println!("{}", hello);          // hello world
}
```

**Ownership & Borrowing rules (interview answer):**
1. Har value ka exactly **ek owner** hota hai. Owner scope se bahar jaate hi value drop ho jaati hai.
2. Ownership **move** ho sakti hai (`let b = a;` ke baad `a` invalid). Copy types (`i32` etc.) move nahi, copy hote hain.
3. **Borrowing** = reference le lo bina ownership liye:
   - **Koi bhi number of immutable borrows** (`&T`) ek saath allowed.
   - **Ya** exactly **ek mutable borrow** (`&mut T`) — aur uss time koi aur borrow nahi.
   - Ye "shared XOR mutable" rule data races compile-time pe rokta hai.
4. Reference apne owner se zyada jee nahi sakta (no dangling references).

Inline version (still no extra variable):
```rust
fn main() {
    let mut hello = String::from("hello");
    hello.push_str(" world");   // method call &mut self automatically leta hai
    println!("{}", hello);
}
```

---

### Q3. Lifetimes — find the largest (longest) string

Function do string slices leta hai aur lambi waali return karta hai. Compiler ko batana padta hai ki returned reference kitni der valid hai → **lifetime annotation `'a`**.

```rust
// 'a: dono inputs aur output same lifetime share karte hain.
// Output utni der valid jab tak chhota lifetime waala input valid hai.
fn largest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Rust");
    let s2 = String::from("Programming");

    let result = largest(s1.as_str(), s2.as_str());
    println!("Largest string: {}", result);   // Programming
}
```

**Kyun lifetime chahiye?** Compiler ko nahi pata return hui reference `x` se aayi ya `y` se. Bina annotation ke wo guarantee nahi kar sakta ki returned reference dono inputs ke valid rehne tak valid hai. `'a` bolta hai: "output uss lifetime tak valid jo `x` aur `y` dono ke valid rehne ka **overlap** hai."

---

### Q4. Traits — full implementation

Trait = shared behavior (interface jaisa). Define karo, types ke liye implement karo, generics/`dyn` se use karo.

```rust
// 1. Trait define
trait Animal {
    fn name(&self) -> String;

    fn sound(&self) -> String;          // required method

    fn describe(&self) -> String {      // default method (override optional)
        format!("{} says {}", self.name(), self.sound())
    }
}

// 2. Types
struct Dog;
struct Cat;

// 3. Implement trait for types
impl Animal for Dog {
    fn name(&self) -> String { "Dog".to_string() }
    fn sound(&self) -> String { "Woof".to_string() }
}

impl Animal for Cat {
    fn name(&self) -> String { "Cat".to_string() }
    fn sound(&self) -> String { "Meow".to_string() }
    fn describe(&self) -> String {       // default override
        "A cat ignores you elegantly".to_string()
    }
}

// 4a. Static dispatch (generics + trait bound) — fast, monomorphized
fn print_sound<T: Animal>(animal: &T) {
    println!("{}", animal.sound());
}

// 4b. Dynamic dispatch (trait object) — heterogeneous collection
fn describe_all(animals: &[Box<dyn Animal>]) {
    for a in animals {
        println!("{}", a.describe());
    }
}

fn main() {
    let dog = Dog;
    let cat = Cat;

    print_sound(&dog);                  // Woof
    println!("{}", dog.describe());     // Dog says Woof

    let zoo: Vec<Box<dyn Animal>> = vec![Box::new(Dog), Box::new(Cat)];
    describe_all(&zoo);                 // Dog says Woof / A cat ignores you elegantly
}
```

**Key points:** required vs default methods, static dispatch (`impl`/`<T: Trait>`, zero-cost, monomorphized) vs dynamic dispatch (`dyn Trait` via vtable, runtime polymorphism, heterogeneous collections).

---

### Q5. `String` vs `&str` — difference with code

| Feature | `String` | `&str` |
|---------|----------|--------|
| Ownership | Owned | Borrowed (reference) |
| Memory | Heap-allocated, growable | View into existing data (no alloc) |
| Mutable? | Yes (`push`, `push_str`) | No (immutable slice) |
| Size | 3 words (ptr, len, cap) | 2 words (ptr, len) — fat pointer |
| Use when | Need to own/modify text | Just need to read/borrow text |

```rust
fn main() {
    // String — owned, heap, growable
    let mut owned: String = String::from("hello");
    owned.push_str(" world");           // can modify
    println!("{}", owned);              // hello world

    // &str — borrowed slice, immutable view
    let literal: &str = "I am a string literal";   // baked into binary
    let slice: &str = &owned[0..5];               // borrow part of String
    println!("{}", slice);              // hello

    // String -> &str (deref coercion / .as_str())
    print_str(&owned);                  // &String -> &str automatically
    print_str(literal);

    // &str -> String
    let back: String = literal.to_string();
}

fn print_str(s: &str) {                 // &str leke dono accept karta hai
    println!("len = {}", s.len());
}
```

**Interview line:** "`String` heap-owned growable buffer hai; `&str` ek immutable borrowed view (fat pointer = ptr + len). Function args mein `&str` prefer karo — String aur literal dono accept ho jaate hain via deref coercion."

---

### Q6. What are futures in Rust?

- **Future** ek value ko represent karta hai jo **abhi ready nahi, par eventually hogi** — "eventually a value" abstraction.
- `async fn` ek `Future` return karta hai. Future **lazy** hota hai — banane se kuch nahi hota; **poll** ya `.await` karne par hi kaam shuru hota hai.
- `Future` trait:
  ```rust
  pub trait Future {
      type Output;
      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
  }
  pub enum Poll<T> { Ready(T), Pending }
  ```
- **Polling model**: executor `poll()` call karta hai → future `Pending` (not ready) ya `Ready(value)` return karta hai. `Pending` ke saath future ek **Waker** register karta hai — "jab ready hu mujhe wake karo". Event hone par (I/O done, timer) Waker executor ko poke karta hai → dobara poll.
- Rust mein executor built-in **nahi** — runtime crate chahiye (**Tokio**, async-std, smol).

```rust
use std::future::Future;

fn make_future() -> impl Future<Output = u32> {
    async { 42 }                  // async block ek Future hai
}

#[tokio::main]
async fn main() {
    let fut = make_future();      // lazy — abhi kuch nahi chala
    let val = fut.await;          // yahan poll hota hai, value milti hai
    println!("{}", val);          // 42
}
```

---

### Q7. What is asynchronous programming?

- **Async programming** = **non-blocking concurrent** code likhna, **synchronous-looking syntax** (`async`/`await`) mein.
- Jab koi task **wait** karta hai (network, disk, DB), wo thread ko block nahi karta — task **suspend** ho jaata hai aur thread **doosre tasks** chalata hai. Wait khatam hone par task resume.
- **Threads vs Async**:
  - Threads = heavy (har thread 1–8 MB stack, kernel context switch), best for **CPU-bound**.
  - Async tasks = lightweight (chhota struct, no kernel switch), best for **I/O-bound** — hazaaron/lakhon concurrent connections.
- Cooperative scheduling: task `.await` pe **yield** karta hai (pre-emptive nahi).

```rust
use tokio::time::{sleep, Duration};

async fn fetch(id: u32) -> u32 {
    sleep(Duration::from_secs(1)).await;   // yahan thread block NAHI hota
    id * 10
}

#[tokio::main]
async fn main() {
    // dono ek saath chalenge (~1s total, ~2s nahi)
    let (a, b) = tokio::join!(fetch(1), fetch(2));
    println!("{} {}", a, b);               // 10 20
}
```

**One-liner:** "Async = ek thread pe blocking ke bina kai cheezein 'wait' karwana; I/O-bound concurrency ke liye ideal."

---

### Q8. Store an `i32` from stack to heap

By default `i32` **stack** par rehta hai. Heap par daalne ke liye `Box<T>` use karo — Box value ko heap pe allocate karta hai aur stack par sirf ek pointer rakhta hai.

```rust
fn main() {
    let on_stack: i32 = 5;          // stack par

    let on_heap: Box<i32> = Box::new(5);   // value heap par, pointer stack par

    println!("{}", on_heap);        // 5  (Display via deref)
    println!("{}", *on_heap + 1);   // 6  (* se deref karke value)

    // Box drop hote hi heap memory free (RAII)
}
```

**Interview point:** `Box<T>` sabse simple smart pointer hai — single owner, heap allocation, deref se inner value, drop par auto-free. Use cases: bade data ko heap pe rakhna, recursive types (`Box<Node>`), trait objects (`Box<dyn Trait>`).

---

### Q9. How smart pointers work (all smart pointers)

**Smart pointer** = ek struct jo pointer jaisa behave karta hai par **extra capabilities + metadata** rakhta hai. Usually `Deref` (pointer jaisa deref) aur `Drop` (auto cleanup) implement karta hai.

| Smart Pointer | Kaam | Ownership | Thread-safe? |
|---------------|------|-----------|--------------|
| `Box<T>` | Heap allocation, single owner | Single | — (T pe depend) |
| `Rc<T>` | Reference counting — **multiple owners** | Shared (immutable) | ❌ No (single-thread) |
| `Arc<T>` | Atomic Rc — multiple owners across threads | Shared | ✅ Yes |
| `RefCell<T>` | **Interior mutability**, borrow rules **runtime** pe check | Single | ❌ No |
| `Cell<T>` | Interior mutability via copy/replace (no references) | Single | ❌ No |
| `Mutex<T>` | Thread-safe interior mutability (lock) | — | ✅ Yes |
| `RwLock<T>` | Multiple readers OR one writer | — | ✅ Yes |
| `Weak<T>` | Non-owning ref (`Rc`/`Arc` ke saath) — **reference cycles** todne ke liye | Non-owning | matches Rc/Arc |

```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

fn main() {
    // Box — heap, single owner
    let b = Box::new(10);

    // Rc — multiple owners (single thread)
    let a = Rc::new(5);
    let a2 = Rc::clone(&a);                 // count = 2
    println!("count = {}", Rc::strong_count(&a));   // 2

    // Rc<RefCell<T>> — shared + mutable (single thread)
    let shared = Rc::new(RefCell::new(0));
    *shared.borrow_mut() += 1;              // runtime borrow check
    println!("{}", shared.borrow());        // 1

    // Arc<Mutex<T>> — shared + mutable across threads
    let counter = Arc::new(Mutex::new(0));
    *counter.lock().unwrap() += 1;
    println!("{}", *counter.lock().unwrap());   // 1
}
```

**Common patterns:** `Rc<RefCell<T>>` (single-thread shared mutable), `Arc<Mutex<T>>` (multi-thread shared mutable).

---

### Q10. Increment & decrement a value using a shared channel

Channel se message bhej kar ek shared counter ko modify karte hain. Yahan ek thread `Inc`/`Dec` commands bhejta hai, main thread receive karke counter update karta hai.

```rust
use std::sync::mpsc;
use std::thread;

enum Op {
    Inc,
    Dec,
}

fn main() {
    let (tx, rx) = mpsc::channel();

    // Producer thread — commands bhejta hai
    let handle = thread::spawn(move || {
        tx.send(Op::Inc).unwrap();
        tx.send(Op::Inc).unwrap();
        tx.send(Op::Inc).unwrap();
        tx.send(Op::Dec).unwrap();
        // tx yahan drop hota hai -> rx loop khatam ho jaayega
    });

    // Consumer (main) — receive karke counter update
    let mut counter = 0;
    for op in rx {                      // sender drop hone tak iterate
        match op {
            Op::Inc => counter += 1,
            Op::Dec => counter -= 1,
        }
        println!("Counter = {}", counter);
    }

    handle.join().unwrap();
    println!("Final = {}", counter);    // Final = 2
}
```

**Philosophy:** "Do not communicate by sharing memory; share memory by communicating." Counter sirf ek thread (consumer) own karta hai → koi lock/`Mutex` nahi chahiye, koi data race nahi.

**Alternative — `Arc<Mutex<i32>>` shared-state version** (jab interviewer channel ke bajaye shared mutable state maange):

```rust
use std::sync::{Arc, Mutex};

struct Counter {
    value: Arc<Mutex<i32>>,
}

impl Counter {
    fn new() -> Self {
        Self {
            value: Arc::new(Mutex::new(0)),
        }
    }

    fn increment(&self) {
        let mut val = self.value.lock().unwrap();   // lock pakda
        *val += 1;                                  // guard ke through mutate
    }                                               // guard drop -> lock release

    fn decrement(&self) {
        let mut val = self.value.lock().unwrap();
        *val -= 1;
    }

    fn get(&self) -> i32 {
        *self.value.lock().unwrap()
    }
}

fn main() {
    let counter = Counter::new();

    counter.increment();
    counter.increment();
    counter.decrement();

    println!("Counter = {}", counter.get());        // Counter = 1
}
```

**Channel vs `Arc<Mutex>` — kab kya:**
- **Channel (Q10)**: counter ek hi thread own karta hai, baaki threads sirf messages bhejte hain → koi lock nahi, deadlock nahi. "Share memory by communicating."
- **`Arc<Mutex>` (ye version)**: multiple threads **directly** shared value pe lock le-de kar mutate karte hain. `Arc` = shared ownership across threads, `Mutex` = ek time ek hi thread access kare.
- Note: `lock()` ka guard scope end pe **automatically release** hota hai (RAII). `lock().unwrap()` poisoned mutex pe panic karega — dekho [Q14 thread poisoning](#q14-thread-poisoning).

---

### Q11. What is `Cell`?

- `Cell<T>` **interior mutability** deta hai — yani ek value ko **immutable reference (`&self`) ke through bhi mutate** kar sakte ho. Normally Rust `&T` ke through mutation allow nahi karta.
- `Cell<T>` **references nahi deta** inner value ka — sirf **pura value copy/replace/get/set** karta hai. Isliye borrow rules break nahi hote, aur **runtime cost zero** (no borrow tracking).
- **Single-threaded only** (`!Sync`). Threads ke liye `Mutex`/`RwLock`/`Atomic`.

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<i32>,       // mutate kar sakte even with &self
}

fn main() {
    let c = Counter { count: Cell::new(0) };

    c.count.set(c.count.get() + 1);     // get + set
    c.count.set(c.count.get() + 1);
    println!("{}", c.count.get());      // 2

    let old = c.count.replace(100);     // replace, returns old
    println!("old={}, now={}", old, c.count.get());   // old=2, now=100
}
```

**`Cell` vs `RefCell`:**
- `Cell<T>` — `get`/`set`/`replace` se **value copy/move**, no references diye jaate, no runtime borrow checks. (T: `Copy` for `get`.)
- `RefCell<T>` — `borrow()`/`borrow_mut()` se **references** deta hai, borrow rules **runtime** pe check, violate karoge to **panic**.

---

### Q12. Error handling in Rust

Rust mein exceptions nahi — errors **values** hain. Do approaches:

1. **Recoverable errors** → `Result<T, E>`:
   ```rust
   use std::fs::File;

   fn open_file() -> Result<File, std::io::Error> {
       let f = File::open("config.txt")?;   // ? = error ho to early return
       Ok(f)
   }
   ```
   - `?` operator: `Ok(v)` → unwrap, `Err(e)` → function se `return Err(e.into())`.
   - `match`, `unwrap_or`, `unwrap_or_else`, `map_err` se handle karte hain.

2. **Unrecoverable errors** → `panic!` (program abort/unwind). Use `unwrap()`/`expect()` jab tum **certain** ho value hai (warna panic).

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("divide by zero".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10, 2) {
        Ok(v) => println!("Result: {}", v),     // Result: 5
        Err(e) => println!("Error: {}", e),
    }

    // ? operator chaining ke liye
    let r = divide(10, 0).unwrap_or(-1);
    println!("{}", r);                            // -1
}
```

**Best practices:** libraries `Result` return karein (caller decide kare); custom error types ya `Box<dyn Error>` / `thiserror`/`anyhow` use karo; `panic!` sirf truly unrecoverable cases mein.

---

### Q13. Difference between `Result` and `Option`

| | `Option<T>` | `Result<T, E>` |
|--|-------------|----------------|
| Represents | Value **ho bhi sakti hai, nahi bhi** | Operation **succeed ya fail** |
| Variants | `Some(T)` / `None` | `Ok(T)` / `Err(E)` |
| "Empty" case info | `None` — koi reason nahi | `Err(E)` — failure ka **reason** carry karta hai |
| Use when | Absence is normal/expected | Failure ka cause batana ho |

```rust
// Option — value ho bhi sakti hai nahi bhi
fn find(v: &[i32], target: i32) -> Option<usize> {
    v.iter().position(|&x| x == target)
}

// Result — fail ka reason chahiye
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse::<i32>()       // Ok(n) ya Err(reason)
}

fn main() {
    let v = vec![1, 2, 3];
    match find(&v, 2) {
        Some(i) => println!("found at {}", i),   // found at 1
        None => println!("not found"),
    }

    match parse("42") {
        Ok(n) => println!("parsed {}", n),       // parsed 42
        Err(e) => println!("error: {}", e),
    }
}
```

- Convert: `Option` → `Result` via `.ok_or(err)`; `Result` → `Option` via `.ok()`.
- **Rule of thumb:** "Value missing ho sakti hai" → `Option`. "Operation fail ho sakta hai (aur kyun janna hai)" → `Result`.

---

### Q14. Thread poisoning

- Jab koi thread `Mutex`/`RwLock` ka lock **hold karte hue panic** kar jaaye, to wo lock **"poisoned"** ho jaata hai.
- Reason: panic ke time protected data ek **inconsistent/half-updated state** mein ho sakta hai. Rust doosre threads ko warn karta hai ki "ye data shayad corrupt hai".
- Iske baad `lock()` call ek **`Err(PoisonError)`** return karta hai (panic nahi karta) — caller decide kare kya karna hai.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(0));

    // Thread lock pakad ke panic karega -> mutex poison
    let d = Arc::clone(&data);
    let _ = thread::spawn(move || {
        let mut g = d.lock().unwrap();
        *g += 1;
        panic!("boom while holding lock");   // poison!
    }).join();   // join returns Err

    // Ab lock() Err(PoisonError) dega
    match data.lock() {
        Ok(g) => println!("value = {}", *g),
        Err(poisoned) => {
            // Data recover karna ho to into_inner() se forcibly le sakte ho
            let g = poisoned.into_inner();
            println!("recovered (poisoned) value = {}", *g);   // 1
        }
    }
}
```

**Interview points:**
- Poisoning sirf `std::sync::Mutex`/`RwLock` mein (Tokio ke async mutex mein poisoning nahi).
- `lock().unwrap()` poisoned lock pe **panic** karega — isliye production mein `match`/`into_inner()` se handle karo.
- Purpose: panic ke baad shared state ki corruption ko silently use hone se rokna.

---
