# Rust Interview Questions (Hindi)

Rust interviews mein frequently puche jaane wale questions ka comprehensive collection, basic se advanced level tak.

---

## Table of Contents
1. [Basic Questions](#basic-questions)
2. [Ownership & Borrowing](#ownership--borrowing)
3. [Lifetimes](#lifetimes)
4. [Traits & Generics](#traits--generics)
5. [Error Handling](#error-handling)
6. [Concurrency](#concurrency)
7. [Memory Management](#memory-management)
8. [Smart Pointers](#smart-pointers)
9. [Closures & Iterators](#closures--iterators)
10. [Macros](#macros)
11. [Async/Await](#asyncawait)
12. [Coding Questions](#coding-questions)

---

## Basic Questions

### 1. Rust kya hai aur iski main features kya hain?

**Jawab:** Rust ek systems programming language hai jo safety, speed, aur concurrency par focus karti hai. Main features:
- **Memory safety** bina garbage collection ke
- **Zero-cost abstractions**
- **Ownership system** memory management ke liye
- **Fearless concurrency**
- **Pattern matching**
- **Type inference**
- **Minimal runtime**

---

### 2. Rust mein `String` aur `&str` mein kya antar hai?

| Feature | `String` | `&str` |
|---------|----------|--------|
| Storage | Heap pe allocated | Stack, heap, ya static ho sakta hai |
| Mutability | Mutable (grow ho sakta hai) | Immutable |
| Ownership | Owned type | Borrowed reference |
| Size | Dynamic | Creation pe fixed |

```rust
fn main() {
    let s1: &str = "Hello";           // String slice (static)
    let s2: String = String::from("World"); // Owned String
    let s3: &str = &s2;               // String ko &str ki tarah borrow kiya
}
```

**Samjhein:**
- `&str` ek reference hai jo kisi existing string data ko point karta hai
- `String` apna data own karta hai aur modify ho sakta hai
- `&str` se `String` banane ke liye `.to_string()` ya `String::from()` use karo
- `String` se `&str` lene ke liye simply `&` lagao

---

### 3. `Copy` aur `Clone` traits mein kya antar hai?

**Jawab:**
- **`Copy`**: Implicit, bitwise copy. Types simple honi chahiye (stack pe stored). Examples: integers, booleans, chars.
- **`Clone`**: Explicit, deep copy. Heap allocation involve ho sakti hai. `.clone()` method call karna padta hai.

```rust
// Copy types - implicit copy
let x = 5;
let y = x;  // x abhi bhi valid hai

// Clone types - explicit clone
let s1 = String::from("hello");
let s2 = s1.clone();  // clone() use karna padega
// s1 abhi bhi valid hai kyunki humne clone kiya
```

**Key Points:**
- `Copy` wale types automatically copy hote hain assignment pe
- `Clone` wale types mein manually `.clone()` likhna padta hai
- Agar type `Copy` implement karti hai to wo `Clone` bhi implement karegi

---

### 4. `Vec`, Array, aur Slice mein kya differences hain?

| Feature | Array `[T; N]` | Vec `Vec<T>` | Slice `&[T]` |
|---------|----------------|--------------|--------------|
| Size | Compile time pe fixed | Dynamic | Dynamic view |
| Storage | Stack | Heap | Contiguous memory ka reference |
| Ownership | Owned | Owned | Borrowed |

```rust
fn main() {
    let arr: [i32; 3] = [1, 2, 3];      // Fixed-size array
    let vec: Vec<i32> = vec![1, 2, 3];   // Dynamic vector
    let slice: &[i32] = &arr[0..2];      // Slice (view)
}
```

**Samjhein:**
- Array ka size compile time pe pata hona chahiye
- Vec runtime pe grow/shrink ho sakta hai
- Slice kisi bhi contiguous memory ka view deta hai

---

### 5. `?` operator kya hai?

**Jawab:** `?` operator error propagation ke liye use hota hai. Ye `Result` ya `Option` types ko unwrap karta hai, aur error/None hone pe early return kar deta hai.

```rust
fn read_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?;  // Error pe early return
    Ok(content)
}

// Ye equivalent hai:
fn read_file_verbose(path: &str) -> Result<String, std::io::Error> {
    let content = match std::fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    Ok(content)
}
```

**Key Points:**
- `?` sirf `Result` ya `Option` return karne wale functions mein use ho sakta hai
- Error automatically propagate ho jata hai
- Code concise aur readable ho jata hai

---

## Ownership & Borrowing

### 6. Rust mein ownership ke rules explain karo.

**Jawab:** Rust mein teen ownership rules hain:
1. Har value ka exactly ek owner hota hai
2. Ek time pe sirf ek hi owner ho sakta hai
3. Jab owner scope se bahar jata hai, value drop ho jati hai

```rust
fn main() {
    let s1 = String::from("hello");  // s1 String ka owner hai
    let s2 = s1;                      // Ownership s2 ko move ho gaya
    // println!("{}", s1);           // Error! s1 ab valid nahi
    println!("{}", s2);              // OK
}
```

**Samjhein:**
- Ye rules memory leaks aur dangling pointers prevent karte hain
- Move hone ke baad original variable use nahi ho sakta
- Primitive types (i32, bool, etc.) copy hote hain, move nahi

---

### 7. Borrowing ke rules kya hain?

**Jawab:**
1. Ya to **ek mutable reference** ho sakta hai **YA** kitne bhi immutable references
2. References hamesha valid hone chahiye (no dangling references)

```rust
fn main() {
    let mut s = String::from("hello");

    // Multiple immutable borrows - OK
    let r1 = &s;
    let r2 = &s;
    println!("{} {}", r1, r2);

    // Immutable borrows khatam hone ke baad mutable borrow - OK
    let r3 = &mut s;
    r3.push_str(" world");

    // Mutable aur immutable ek saath nahi ho sakte
    // let r4 = &s;      // Error agar r3 abhi bhi use ho raha ho
}
```

**Key Points:**
- Ye rules data races compile time pe prevent karte hain
- Compiler ensure karta hai ki koi conflict na ho
- NLL (Non-Lexical Lifetimes) ki wajah se borrows jaldi khatam ho sakte hain

---

### 8. `&` aur `&mut` mein kya antar hai?

**Jawab:**
- `&` ek **immutable reference** create karta hai (sirf read access)
- `&mut` ek **mutable reference** create karta hai (read aur write access)

```rust
fn main() {
    let mut value = 10;

    let r1 = &value;      // Immutable borrow
    println!("{}", r1);   // Read kar sakte hain

    let r2 = &mut value;  // Mutable borrow
    *r2 = 20;             // Modify kar sakte hain
}
```

**Samjhein:**
- `&` se data read ho sakta hai, modify nahi
- `&mut` se data read aur modify dono ho sakte hain
- Ek time pe ek hi `&mut` ho sakta hai

---

### 9. Rust mein move kya hota hai?

**Jawab:** Move ownership ko ek variable se dusre mein transfer karta hai. Move ke baad original variable invalid ho jata hai.

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = v1;          // v1 v2 mein move ho gaya
    // println!("{:?}", v1);  // Error: v1 move ho chuka hai
    println!("{:?}", v2);    // OK
}

fn take_ownership(v: Vec<i32>) {
    println!("{:?}", v);
}

fn main() {
    let v = vec![1, 2, 3];
    take_ownership(v);      // v function mein move ho gaya
    // println!("{:?}", v); // Error: v move ho chuka hai
}
```

**Key Points:**
- Move heap-allocated data ke liye hota hai
- Stack data (Copy types) copy hota hai, move nahi
- Move se double-free errors prevent hote hain

---

## Lifetimes

### 10. Rust mein lifetimes kya hain?

**Jawab:** Lifetimes Rust ka tarika hai ensure karne ka ki references jab tak use ho rahe hain tab tak valid rahein. Ye dangling references prevent karte hain.

```rust
// Lifetime annotation 'a compiler ko batata hai ki dono inputs aur output
// kam se kam 'a tak live rahenge
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("long");
    let s2 = String::from("short");
    let result = longest(&s1, &s2);
    println!("Longest: {}", result);
}
```

**Samjhein:**
- Lifetimes compile-time concept hai, runtime pe koi cost nahi
- `'a` ek lifetime parameter hai
- Compiler ensure karta hai ki references valid rahein

---

### 11. `'static` lifetime kya hai?

**Jawab:** `'static` ka matlab hai ki reference program ki puri duration tak live rah sakta hai.

```rust
// String literals ki 'static lifetime hoti hai
let s: &'static str = "Hello, world!";

// Constants bhi 'static hote hain
static HELLO: &str = "Hello";

// Owned data 'static bounds satisfy kar sakta hai (koi references nahi)
fn needs_static<T: 'static>(value: T) {}
needs_static(String::from("owned")); // OK - String owned hai, koi references nahi
```

**Key Points:**
- String literals hamesha `'static` hote hain
- `'static` bound ka matlab hai type mein koi non-static references nahi hain
- Owned types `'static` bound satisfy kar sakte hain

---

### 12. Lifetime elision rules explain karo.

**Jawab:** Rust ke teen lifetime elision rules hain jo explicit annotations likhne se bachate hain:

1. Har input reference ko apni lifetime parameter milti hai
2. Agar exactly ek input lifetime ho, wo sab output lifetimes ko assign hoti hai
3. Agar `&self` ya `&mut self` ho, self ki lifetime sab outputs ko assign hoti hai

```rust
// Rule 2 apply - input lifetime output ko assign
fn first_word(s: &str) -> &str {
    // Same as: fn first_word<'a>(s: &'a str) -> &'a str
    &s[..1]
}

// Rule 3 apply - self ki lifetime output ko assign
impl MyStruct {
    fn get_name(&self) -> &str {
        // Same as: fn get_name<'a>(&'a self) -> &'a str
        &self.name
    }
}
```

**Samjhein:**
- Compiler automatically lifetimes infer kar leta hai common cases mein
- Ambiguous cases mein explicit annotations zaroori hain

---

## Traits & Generics

### 13. `impl Trait` aur `dyn Trait` mein kya antar hai?

**Jawab:**
- **`impl Trait`** (static dispatch): Compiler har type ke liye specific code generate karta hai. Zero runtime cost.
- **`dyn Trait`** (dynamic dispatch): Runtime pe vtable use hota hai. Thoda runtime overhead par heterogeneous collections allow karta hai.

```rust
// Static dispatch - compile time pe monomorphized
fn print_static(item: impl Display) {
    println!("{}", item);
}

// Dynamic dispatch - runtime pe vtable use
fn print_dynamic(item: &dyn Display) {
    println!("{}", item);
}

// dyn heterogeneous collections allow karta hai
let items: Vec<Box<dyn Display>> = vec![
    Box::new(42),
    Box::new("hello"),
];
```

**Key Points:**
- `impl Trait` faster hai (inlining possible)
- `dyn Trait` flexible hai (different types ek collection mein)
- `impl Trait` return type mein single concrete type represent karta hai

---

### 14. Trait object kya hai?

**Jawab:** Trait object dynamic dispatch ka tarika hai. Isme data ka pointer aur vtable ka pointer hota hai.

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;
struct Cat;

impl Animal for Dog {
    fn speak(&self) { println!("Bhow!"); }
}

impl Animal for Cat {
    fn speak(&self) { println!("Meow!"); }
}

fn main() {
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog),
        Box::new(Cat),
    ];

    for animal in animals {
        animal.speak();
    }
}
```

**Samjhein:**
- `dyn Trait` trait object banata hai
- Different types ek collection mein rakh sakte ho
- Runtime pe correct method call hota hai

---

### 15. Associated types traits mein kya hote hain?

**Jawab:** Associated types placeholder types hain traits ke andar jo implementors ko specify karne padte hain.

```rust
trait Iterator {
    type Item;  // Associated type
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;  // Associated type specify kiya

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        Some(self.count)
    }
}
```

**Key Points:**
- Associated types implementation mein define hote hain
- Generics se different - ek type ke liye ek hi implementation
- Code cleaner hota hai generic parameters se

---

### 16. Associated types aur generics mein kya antar hai?

**Jawab:**
- **Generics**: Multiple implementations allow karte hain different types ke liye
- **Associated types**: Ek type ke liye sirf ek implementation

```rust
// Generics ke saath - multiple implementations possible
trait ConvertTo<T> {
    fn convert(&self) -> T;
}

impl ConvertTo<i32> for String { /* ... */ }
impl ConvertTo<f64> for String { /* ... */ }

// Associated type ke saath - sirf ek implementation
trait IntoIterator {
    type Item;
    type IntoIter;
    fn into_iter(self) -> Self::IntoIter;
}
// Vec<T> ka sirf EK IntoIterator implementation ho sakta hai
```

**Samjhein:**
- Generics jab multiple conversions/implementations chahiye
- Associated types jab ek-to-one relationship ho

---

### 17. Trait bounds aur `where` clause explain karo.

**Jawab:** Trait bounds generic types ko constrain karte hain specific traits implement karne wale types tak.

```rust
// Inline trait bounds
fn print_debug<T: Debug>(item: T) {
    println!("{:?}", item);
}

// Multiple bounds
fn compare<T: PartialOrd + Debug>(a: T, b: T) {
    println!("{:?} vs {:?}", a, b);
}

// Where clause - complex bounds ke liye cleaner
fn complex_function<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}
```

**Key Points:**
- `+` se multiple traits combine hote hain
- `where` clause readability improve karta hai
- Bounds ensure karte hain ki required methods available hain

---

## Error Handling

### 18. `Result` aur `Option` mein kya antar hai?

**Jawab:**
- **`Option<T>`**: Optional value represent karta hai. `Some(T)` ya `None`.
- **`Result<T, E>`**: Success ya failure represent karta hai. `Ok(T)` ya `Err(E)`.

```rust
// Option - optional values ke liye
fn find_user(id: u32) -> Option<User> {
    if id == 1 {
        Some(User { name: "Amit".into() })
    } else {
        None
    }
}

// Result - fail ho sakne wale operations ke liye
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Zero se divide nahi kar sakte".into())
    } else {
        Ok(a / b)
    }
}
```

**Key Points:**
- `Option` jab value ho ya na ho
- `Result` jab operation fail ho sakta ho with error info
- Dono `match`, `if let`, ya methods se handle hote hain

---

### 19. `unwrap()`, `expect()`, aur `?` mein kya antar hai?

**Jawab:**
- **`unwrap()`**: Error pe generic message ke saath panic
- **`expect(msg)`**: Error pe custom message ke saath panic
- **`?`**: Error ko caller ko propagate karta hai (panic nahi)

```rust
fn main() {
    let result: Result<i32, &str> = Ok(42);

    // unwrap - Err pe panic
    let value = result.unwrap();

    // expect - custom message ke saath panic
    let value = result.expect("Value lene mein fail");
}

// ? operator - error propagate karta hai
fn process() -> Result<i32, &str> {
    let result: Result<i32, &str> = Ok(42);
    let value = result?;  // Err pe early return
    Ok(value * 2)
}
```

**Samjhein:**
- `unwrap`/`expect` testing aur prototyping mein useful
- Production code mein `?` prefer karo
- `?` graceful error handling allow karta hai

---

### 20. Custom error types kaise banate hain?

**Jawab:** Enums use karo `std::error::Error` trait implementation ke saath.

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum MyError {
    NotFound(String),
    InvalidInput(String),
    IoError(std::io::Error),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::NotFound(msg) => write!(f, "Nahi mila: {}", msg),
            MyError::InvalidInput(msg) => write!(f, "Galat input: {}", msg),
            MyError::IoError(e) => write!(f, "IO error: {}", e),
        }
    }
}

impl Error for MyError {}

// thiserror crate use karo (recommended)
use thiserror::Error;

#[derive(Error, Debug)]
enum MyError {
    #[error("Nahi mila: {0}")]
    NotFound(String),
    #[error("Galat input: {0}")]
    InvalidInput(String),
    #[error("IO error")]
    IoError(#[from] std::io::Error),
}
```

---

## Concurrency

### 21. Rust thread safety kaise ensure karta hai?

**Jawab:** Rust do marker traits use karta hai:
- **`Send`**: Type threads ke beech transfer ho sakti hai
- **`Sync`**: Type threads ke beech share ho sakti hai (references ke through)

```rust
use std::thread;
use std::sync::Arc;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    let handles: Vec<_> = (0..3).map(|i| {
        let data = Arc::clone(&data);
        thread::spawn(move || {
            println!("Thread {}: {:?}", i, data);
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

**Key Points:**
- Compiler check karta hai ki types thread-safe hain
- `Send` nahi hai: `Rc<T>`
- `Sync` nahi hai: `RefCell<T>`
- `Arc` thread-safe `Rc` hai

---

### 22. `Rc` aur `Arc` mein kya antar hai?

**Jawab:**
- **`Rc<T>`** (Reference Counted): Single-threaded reference counting. `Send` ya `Sync` nahi.
- **`Arc<T>`** (Atomic Reference Counted): Thread-safe reference counting. Atomic operations use karta hai.

```rust
use std::rc::Rc;
use std::sync::Arc;

// Rc - sirf single thread
let rc = Rc::new(5);
let rc2 = Rc::clone(&rc);

// Arc - thread-safe
let arc = Arc::new(5);
let arc2 = Arc::clone(&arc);

// Arc dusre threads mein bhi ja sakta hai
std::thread::spawn(move || {
    println!("{}", arc2);
});
```

**Samjhein:**
- Single-threaded code mein `Rc` use karo (faster)
- Multi-threaded code mein `Arc` use karo (safe)
- `Arc` ka overhead hai atomic operations ki wajah se

---

### 23. `Mutex` kya hai aur kaise use hota hai?

**Jawab:** `Mutex` mutual exclusion provide karta hai, ek time pe sirf ek thread data access kar sakti hai.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

**Key Points:**
- `lock()` se exclusive access milta hai
- Lock scope khatam hone pe automatically release
- `Arc<Mutex<T>>` common pattern hai threads mein sharing ke liye

---

### 24. `RwLock` kya hai aur `Mutex` se kab better hai?

**Jawab:** `RwLock` multiple readers YA ek writer allow karta hai. Use karo jab reads zyada frequent hain writes se.

```rust
use std::sync::RwLock;

let lock = RwLock::new(5);

// Multiple readers allowed
{
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
    println!("{} {}", r1, r2);
}

// Sirf ek writer
{
    let mut w = lock.write().unwrap();
    *w += 1;
}
```

**Samjhein:**
- Read-heavy workloads mein better performance
- Write lock exclusive hai
- Read locks concurrent ho sakte hain

---

### 25. Rust mein channels explain karo.

**Jawab:** Channels threads ke beech message passing provide karte hain. `mpsc` = multiple producer, single consumer.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // Multiple producers ke liye clone
    let tx1 = tx.clone();

    thread::spawn(move || {
        tx.send("thread 1 se").unwrap();
    });

    thread::spawn(move || {
        tx1.send("thread 2 se").unwrap();
    });

    // Sab messages receive karo
    for received in rx {
        println!("Mila: {}", received);
    }
}
```

**Key Points:**
- `send()` message bhejta hai
- `recv()` blocking receive
- `try_recv()` non-blocking receive
- Channel ownership-based hai (safe)

---

## Memory Management

### 26. Rust mein RAII kya hai?

**Jawab:** RAII (Resource Acquisition Is Initialization) ka matlab hai resources initialization pe acquire hote hain aur object scope se bahar jane pe release hote hain.

```rust
fn main() {
    {
        let file = File::create("test.txt").unwrap();
        // file yahan open hai
        file.write_all(b"Hello").unwrap();
    } // file automatically yahan close ho gaya (Drop call hua)

    {
        let mutex = Mutex::new(0);
        let guard = mutex.lock().unwrap();
        // mutex locked hai
    } // mutex automatically yahan unlock ho gaya
}
```

**Samjhein:**
- Resources automatically cleanup hote hain
- Memory leaks prevent hote hain
- Explicit cleanup code ki zaroorat nahi

---

### 27. `Drop` trait kya hai?

**Jawab:** `Drop` trait cleanup behavior define karta hai jab value scope se bahar jati hai.

```rust
struct CustomResource {
    name: String,
}

impl Drop for CustomResource {
    fn drop(&mut self) {
        println!("Resource drop ho raha: {}", self.name);
    }
}

fn main() {
    let r1 = CustomResource { name: "Resource 1".into() };
    let r2 = CustomResource { name: "Resource 2".into() };
    println!("Resources create ho gaye");
} // Prints: Resource 2 drop, phir Resource 1 (reverse order)

// Manual drop
fn main() {
    let r = CustomResource { name: "Early drop".into() };
    drop(r);  // Abhi drop ho jayega
    println!("Drop ke baad");
}
```

**Key Points:**
- `Drop` automatically call hota hai scope end pe
- Reverse order mein drop hota hai
- `drop()` function se manually bhi drop kar sakte ho

---

### 28. Rust mein stack aur heap allocation mein kya antar hai?

**Jawab:**
- **Stack**: Fixed-size data, fast allocation/deallocation, automatic cleanup
- **Heap**: Dynamic-size data, slower, explicit allocation zaroor

```rust
fn main() {
    // Stack pe allocated
    let x: i32 = 5;
    let arr: [i32; 3] = [1, 2, 3];

    // Heap pe allocated
    let s: String = String::from("hello");
    let v: Vec<i32> = vec![1, 2, 3];
    let b: Box<i32> = Box::new(5);
}
```

**Samjhein:**
- Stack fast hai, size fixed hona chahiye
- Heap dynamic hai, pointer through access
- Rust ownership system heap memory manage karta hai

---

## Smart Pointers

### 29. `Box<T>` kya hai aur kab use karein?

**Jawab:** `Box<T>` heap allocation ke liye smart pointer hai. Use cases:
- Recursive types (compile time pe known size chahiye)
- Large data jo ownership transfer bina copy ke karna ho
- Trait objects

```rust
// Recursive type - Box chahiye
enum List {
    Cons(i32, Box<List>),
    Nil,
}

// Trait object
fn get_speaker() -> Box<dyn Speak> {
    Box::new(Dog)
}

// Large data transfer
fn transfer_large_data(data: Box<[u8; 1000000]>) {
    // Ownership transfer bina copying ke
}
```

**Key Points:**
- Single owner, heap allocation
- Deref trait implement karta hai
- Recursive types ke liye essential

---

### 30. `Cow` (Copy on Write) kya hai?

**Jawab:** `Cow` smart pointer hai jo borrowed ya owned data hold kar sakta hai. Clone sirf mutation zaroorat pe hota hai.

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains(' ') {
        // Modify karna hai - owned return
        Cow::Owned(input.replace(' ', "_"))
    } else {
        // Modification nahi - borrowed return
        Cow::Borrowed(input)
    }
}

fn main() {
    let s1 = process("hello");        // Borrowed, koi allocation nahi
    let s2 = process("hello world");  // Owned, allocated
}
```

**Samjhein:**
- Lazy cloning - sirf zaroorat pe clone
- Performance optimization ke liye useful
- `Borrowed` ya `Owned` variants

---

### 31. `RefCell<T>` aur interior mutability explain karo.

**Jawab:** `RefCell<T>` interior mutability provide karta hai - immutable references ke saath bhi data mutate kar sakte ho. Borrow checking runtime pe hoti hai.

```rust
use std::cell::RefCell;

struct Counter {
    value: RefCell<i32>,
}

impl Counter {
    fn increment(&self) {  // Note: &self, &mut self nahi
        *self.value.borrow_mut() += 1;
    }

    fn get(&self) -> i32 {
        *self.value.borrow()
    }
}

fn main() {
    let counter = Counter { value: RefCell::new(0) };
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());  // 2
}
```

**Key Points:**
- Runtime borrow checking (panic ho sakta hai)
- Interior mutability pattern
- Single-threaded use ke liye

---

### 32. `Cell` aur `RefCell` mein kya antar hai?

**Jawab:**
- **`Cell<T>`**: `Copy` types ke liye. Values directly get/set. No runtime borrow checking.
- **`RefCell<T>`**: Kisi bhi type ke liye. References return karta hai. Runtime borrow checking (panic ho sakta hai).

```rust
use std::cell::{Cell, RefCell};

let cell: Cell<i32> = Cell::new(5);
cell.set(10);
let value = cell.get();  // Copy

let refcell: RefCell<String> = RefCell::new("hello".into());
refcell.borrow_mut().push_str(" world");
let borrowed = refcell.borrow();  // Reference
```

**Samjhein:**
- `Cell` simpler hai, Copy types ke liye
- `RefCell` complex types ke liye, references deta hai
- `RefCell` mein borrow rules runtime pe check

---

## Closures & Iterators

### 33. Teen closure traits kya hain: `Fn`, `FnMut`, `FnOnce`?

**Jawab:**
- **`FnOnce`**: Captured variables ki ownership leta hai. Sirf ek baar call ho sakta hai.
- **`FnMut`**: Captured variables mutably borrow karta hai. Multiple times call ho sakta hai.
- **`Fn`**: Captured variables immutably borrow karta hai. Multiple times call ho sakta hai.

```rust
// FnOnce - ownership leta hai
let s = String::from("hello");
let consume = move || {
    drop(s);  // s closure mein move ho gaya
};
consume();
// consume();  // Error: dobara call nahi kar sakte

// FnMut - mutate karta hai
let mut count = 0;
let mut increment = || {
    count += 1;
};
increment();
increment();

// Fn - immutable access
let x = 5;
let print = || println!("{}", x);
print();
print();
```

**Key Points:**
- `FnOnce` sabse restrictive
- `Fn` sabse flexible (multiple calls)
- Compiler automatically decide karta hai

---

### 34. `iter()`, `iter_mut()`, aur `into_iter()` mein kya antar hai?

**Jawab:**
- **`iter()`**: Elements borrow karta hai (`&T`)
- **`iter_mut()`**: Elements mutably borrow karta hai (`&mut T`)
- **`into_iter()`**: Elements ki ownership leta hai (`T`)

```rust
let mut vec = vec![1, 2, 3];

// iter() - borrow
for x in vec.iter() {
    println!("{}", x);  // x &i32 hai
}

// iter_mut() - mutable borrow
for x in vec.iter_mut() {
    *x *= 2;  // x &mut i32 hai
}

// into_iter() - ownership
for x in vec.into_iter() {
    println!("{}", x);  // x i32 hai
}
// vec ab accessible nahi
```

**Samjhein:**
- `iter()` collection ko preserve karta hai
- `iter_mut()` modification allow karta hai
- `into_iter()` collection consume kar leta hai

---

### 35. Iterator adaptors aur consumers explain karo.

**Jawab:**
- **Adaptors**: Iterators transform karte hain, lazy evaluation (map, filter, take, skip)
- **Consumers**: Iterators consume karte hain, values produce karte hain (collect, sum, count, for_each)

```rust
let numbers = vec![1, 2, 3, 4, 5];

// Adaptors lazy hain - consume hone tak kuch nahi hota
let doubled = numbers.iter()
    .map(|x| x * 2)
    .filter(|x| *x > 4);

// Consumer evaluation trigger karta hai
let result: Vec<_> = doubled.collect();  // [6, 8, 10]

// Chaining
let sum: i32 = (1..=100)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
```

**Key Points:**
- Adaptors chain ho sakte hain
- Consumer end pe chahiye
- Lazy evaluation efficient hai

---

## Macros

### 36. Declarative aur procedural macros mein kya antar hai?

**Jawab:**
- **Declarative macros** (`macro_rules!`): Code pe pattern matching
- **Procedural macros**: Token streams pe operate karte hain, zyada powerful

```rust
// Declarative macro
macro_rules! say_hello {
    () => {
        println!("Namaste!");
    };
    ($name:expr) => {
        println!("Namaste, {}!", $name);
    };
}

say_hello!();           // Namaste!
say_hello!("Duniya");   // Namaste, Duniya!

// Procedural macro (derive)
#[derive(Debug, Clone)]  // Ye procedural macros hain
struct MyStruct {
    field: i32,
}
```

**Samjhein:**
- `macro_rules!` simple cases ke liye
- Procedural macros complex transformations ke liye
- Derive macros automatically traits implement karte hain

---

### 37. Procedural macros ke types kya hain?

**Jawab:**
1. **Derive macros**: `#[derive(MacroName)]` se implementations add karte hain
2. **Attribute macros**: Custom attributes `#[macro_name]`
3. **Function-like macros**: Functions ki tarah call hote hain `macro_name!(...)`

```rust
// Derive macro
#[derive(Serialize, Deserialize)]
struct User { name: String }

// Attribute macro
#[route(GET, "/")]
fn index() {}

// Function-like macro
let sql = sql!(SELECT * FROM users WHERE id = 1);
```

---

## Async/Await

### 38. Rust mein async/await kaise kaam karta hai?

**Jawab:** `async` functions `Future` return karti hain jo executor (jaise tokio) ke through poll hota hai completion tak.

```rust
use tokio;

async fn fetch_data() -> String {
    // Simulated async operation
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    "Data".to_string()
}

#[tokio::main]
async fn main() {
    let data = fetch_data().await;
    println!("{}", data);
}
```

**Key Points:**
- `async fn` Future return karta hai
- `.await` Future ke completion ka wait karta hai
- Runtime (tokio, async-std) zaroor hai execution ke liye

---

### 39. `Pin` kya hai aur kyun zaroor hai?

**Jawab:** `Pin` data ko memory mein move hone se rokta hai. Self-referential async futures ke liye zaroor hai.

```rust
use std::pin::Pin;
use std::future::Future;

// Common use - futures pin karna
async fn example() {
    let future = async { 42 };
    let pinned = Box::pin(future);
    // pinned ab poll ho sakta hai
}

// Pin move hone se rokta hai
fn poll_future(future: Pin<&mut dyn Future<Output = i32>>) {
    // future bahar move nahi ho sakta
}
```

**Samjhein:**
- Async futures self-referential ho sakte hain
- Move hone se internal pointers invalid ho jayenge
- `Pin` ye safety ensure karta hai

---

### 40. `tokio::spawn` aur `async` blocks mein kya antar hai?

**Jawab:**
- **`async` blocks**: Future create karte hain, current task pe run hota hai
- **`tokio::spawn`**: Naya task create karta hai jo concurrently run hota hai

```rust
use tokio;

#[tokio::main]
async fn main() {
    // Sequential - ek ke baad ek
    let a = async { 1 }.await;
    let b = async { 2 }.await;

    // Concurrent - alag tasks spawn karo
    let handle1 = tokio::spawn(async { 1 });
    let handle2 = tokio::spawn(async { 2 });

    let (a, b) = (handle1.await.unwrap(), handle2.await.unwrap());

    // Ya join! use karo concurrent futures ke liye same task pe
    let (x, y) = tokio::join!(
        async { 1 },
        async { 2 }
    );
}
```

**Samjhein:**
- `spawn` parallel execution ke liye
- `join!` concurrent futures same task pe
- `await` sequential execution ke liye

---

## Coding Questions

### 41. String reverse karne ka function implement karo.

```rust
fn reverse_string(s: &str) -> String {
    s.chars().rev().collect()
}

// In-place Vec<char> ke liye
fn reverse_in_place(chars: &mut [char]) {
    let len = chars.len();
    for i in 0..len / 2 {
        chars.swap(i, len - 1 - i);
    }
}
```

---

### 42. Basic linked list implement karo.

```rust
type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    data: T,
    next: Link<T>,
}

struct LinkedList<T> {
    head: Link<T>,
}

impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList { head: None }
    }

    fn push(&mut self, data: T) {
        let new_node = Box::new(Node {
            data,
            next: self.head.take(),
        });
        self.head = Some(new_node);
    }

    fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.data
        })
    }
}
```

---

### 43. Thread-safe counter implement karo.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

struct Counter {
    count: Mutex<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Mutex::new(0) }
    }

    fn increment(&self) {
        *self.count.lock().unwrap() += 1;
    }

    fn get(&self) -> i32 {
        *self.count.lock().unwrap()
    }
}

fn main() {
    let counter = Arc::new(Counter::new());
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..100 {
                counter.increment();
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", counter.get());  // 1000
}
```

---

### 44. Builder pattern implement karo.

```rust
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
}

#[derive(Default)]
struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: u32,
}

impl ServerBuilder {
    fn new() -> Self {
        ServerBuilder {
            host: "localhost".to_string(),
            port: 8080,
            max_connections: 100,
        }
    }

    fn host(mut self, host: &str) -> Self {
        self.host = host.to_string();
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn max_connections(mut self, max: u32) -> Self {
        self.max_connections = max;
        self
    }

    fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
        }
    }
}

fn main() {
    let server = ServerBuilder::new()
        .host("0.0.0.0")
        .port(3000)
        .max_connections(1000)
        .build();

    println!("{:?}", server);
}
```

---

### 45. Ye code kya print karega? (Ownership quiz)

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    // println!("{}", s1);  // Ye error dega
    println!("{}", s2);
}
```

**Jawab:** `hello` print hoga. `s1` `s2` mein move ho gaya, `s1` ab invalid hai. Sirf `s2` use ho sakta hai.

---

### 46. Ye code kya print karega? (Lifetime quiz)

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    // println!("{}", r);
}
```

**Jawab:** Ye code compile nahi hoga! `r` dangling reference hoga kyunki `x` scope se bahar ja chuka hai `r` use hone se pehle.

---

### 47. Is code ko fix karo:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

**Jawab:** Lifetime annotations add karo:
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

---

### 48. HashMap use karke simple cache implement karo.

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct Cache<K, V> {
    store: HashMap<K, V>,
}

impl<K: Eq + Hash, V: Clone> Cache<K, V> {
    fn new() -> Self {
        Cache { store: HashMap::new() }
    }

    fn get(&self, key: &K) -> Option<V> {
        self.store.get(key).cloned()
    }

    fn set(&mut self, key: K, value: V) {
        self.store.insert(key, value);
    }

    fn get_or_insert<F>(&mut self, key: K, f: F) -> V
    where
        F: FnOnce() -> V,
        K: Clone,
    {
        self.store.entry(key).or_insert_with(f).clone()
    }
}
```

---

## Quick Reference: Common Interview Topics Checklist

- [ ] Ownership, borrowing, aur lifetimes
- [ ] `String` vs `&str`
- [ ] `Vec`, arrays, aur slices
- [ ] `Result` aur `Option` error handling
- [ ] Traits aur generics
- [ ] `Box`, `Rc`, `Arc`, `RefCell`
- [ ] Closures (`Fn`, `FnMut`, `FnOnce`)
- [ ] Iterators aur adaptors
- [ ] Concurrency (`Mutex`, `RwLock`, channels)
- [ ] `Send` aur `Sync` traits
- [ ] Pattern matching
- [ ] Macros basics
- [ ] Async/await fundamentals
- [ ] Memory layout (stack vs heap)
- [ ] Zero-cost abstractions

---

## Rust Interviews ke liye Tips

1. **Ownership deeply samjho** - Ye #1 topic hai Rust interviews mein
2. **Borrow checker errors explain karne ki practice karo** - Interviewers puchte hain "ye compile kyun nahi hota?"
3. **Standard library jaano** - `Option`, `Result`, `Vec`, `HashMap`, `Iterator` methods
4. **Smart pointers ka use samjho** - `Box` vs `Rc` vs `Arc` vs `RefCell` kab use karein
5. **Safe concurrent code likhna aaye** - `Arc<Mutex<T>>` patterns
6. **Static vs dynamic dispatch ka fark jaano** - `impl Trait` vs `dyn Trait`
7. **Paper/whiteboard pe code karne ki practice karo** - Bahut interviews mein compiler nahi hota

Rust interviews ke liye all the best!
