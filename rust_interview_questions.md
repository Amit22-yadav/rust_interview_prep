# Rust Interview Questions

A comprehensive collection of frequently asked Rust interview questions, from basic to advanced levels.

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

### 1. What is Rust and what are its main features?
**Answer:** Rust is a systems programming language focused on safety, speed, and concurrency. Main features include:
- **Memory safety** without garbage collection
- **Zero-cost abstractions**
- **Ownership system** for memory management
- **Fearless concurrency**
- **Pattern matching**
- **Type inference**
- **Minimal runtime**

---

### 2. What is the difference between `String` and `&str` in Rust?

| Feature | `String` | `&str` |
|---------|----------|--------|
| Storage | Heap-allocated | Can be stack, heap, or static |
| Mutability | Mutable (growable) | Immutable |
| Ownership | Owned type | Borrowed reference |
| Size | Dynamic | Fixed at creation |

```rust
fn main() {
    let s1: &str = "Hello";           // String slice (static)
    let s2: String = String::from("World"); // Owned String
    let s3: &str = &s2;               // Borrow String as &str
}
```

---

### 3. What is the difference between `Copy` and `Clone` traits?

**Answer:**
- **`Copy`**: Implicit, bitwise copy. Types must be simple (stored entirely on stack). Examples: integers, booleans, chars.
- **`Clone`**: Explicit, deep copy. Can involve heap allocation. Must call `.clone()` method.

```rust
// Copy types - implicit copy
let x = 5;
let y = x;  // x is still valid

// Clone types - explicit clone
let s1 = String::from("hello");
let s2 = s1.clone();  // Must use clone()
// s1 is still valid because we cloned
```

---

### 4. What are the differences between `Vec`, Array, and Slice?

| Feature | Array `[T; N]` | Vec `Vec<T>` | Slice `&[T]` |
|---------|----------------|--------------|--------------|
| Size | Fixed at compile time | Dynamic | Dynamic view |
| Storage | Stack | Heap | Reference to contiguous memory |
| Ownership | Owned | Owned | Borrowed |

```rust
fn main() {
    let arr: [i32; 3] = [1, 2, 3];      // Fixed-size array
    let vec: Vec<i32> = vec![1, 2, 3];   // Dynamic vector
    let slice: &[i32] = &arr[0..2];      // Slice (view)
}
```

---

### 5. What is the `?` operator?

**Answer:** The `?` operator is used for error propagation. It unwraps `Result` or `Option` types, returning early if there's an error/None.

```rust
fn read_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?;  // Returns early on error
    Ok(content)
}

// Equivalent to:
fn read_file_verbose(path: &str) -> Result<String, std::io::Error> {
    let content = match std::fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    Ok(content)
}
```

---

## Ownership & Borrowing

### 6. Explain the ownership rules in Rust.

**Answer:** Rust has three ownership rules:
1. Each value has exactly one owner
2. There can only be one owner at a time
3. When the owner goes out of scope, the value is dropped

```rust
fn main() {
    let s1 = String::from("hello");  // s1 owns the String
    let s2 = s1;                      // Ownership moves to s2
    // println!("{}", s1);           // Error! s1 is no longer valid
    println!("{}", s2);              // OK
}
```

---

### 7. What are the borrowing rules?

**Answer:**
1. You can have **either** one mutable reference **OR** any number of immutable references
2. References must always be valid (no dangling references)

```rust
fn main() {
    let mut s = String::from("hello");

    // Multiple immutable borrows - OK
    let r1 = &s;
    let r2 = &s;
    println!("{} {}", r1, r2);

    // Mutable borrow after immutable borrows are done - OK
    let r3 = &mut s;
    r3.push_str(" world");

    // Cannot have mutable and immutable at same time
    // let r4 = &s;      // Error if r3 is still in use
}
```

---

### 8. What is the difference between `&` and `&mut`?

**Answer:**
- `&` creates an **immutable reference** (read-only access)
- `&mut` creates a **mutable reference** (read and write access)

```rust
fn main() {
    let mut value = 10;

    let r1 = &value;      // Immutable borrow
    println!("{}", r1);   // Can read

    let r2 = &mut value;  // Mutable borrow
    *r2 = 20;             // Can modify
}
```

---

### 9. What is a move in Rust?

**Answer:** A move transfers ownership from one variable to another. After a move, the original variable becomes invalid.

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = v1;          // v1 is moved to v2
    // println!("{:?}", v1);  // Error: v1 has been moved
    println!("{:?}", v2);    // OK
}

fn take_ownership(v: Vec<i32>) {
    println!("{:?}", v);
}

fn main() {
    let v = vec![1, 2, 3];
    take_ownership(v);      // v is moved into function
    // println!("{:?}", v); // Error: v has been moved
}
```

---

## Lifetimes

### 10. What are lifetimes in Rust?

**Answer:** Lifetimes are Rust's way of ensuring that references are valid for as long as they're used. They prevent dangling references.

```rust
// Lifetime annotation 'a tells compiler both inputs and output live at least as long as 'a
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

---

### 11. What is the `'static` lifetime?

**Answer:** `'static` means the reference can live for the entire duration of the program.

```rust
// String literals have 'static lifetime
let s: &'static str = "Hello, world!";

// Constants are 'static
static HELLO: &str = "Hello";

// Owned data can satisfy 'static bounds (no references)
fn needs_static<T: 'static>(value: T) {}
needs_static(String::from("owned")); // OK - String is owned, no references
```

---

### 12. Explain lifetime elision rules.

**Answer:** Rust has three lifetime elision rules that allow omitting explicit lifetime annotations:

1. Each input reference gets its own lifetime parameter
2. If there's exactly one input lifetime, it's assigned to all output lifetimes
3. If there's `&self` or `&mut self`, the lifetime of self is assigned to all output lifetimes

```rust
// Rule 2 applied - input lifetime assigned to output
fn first_word(s: &str) -> &str {
    // Same as: fn first_word<'a>(s: &'a str) -> &'a str
    &s[..1]
}

// Rule 3 applied - self's lifetime assigned to output
impl MyStruct {
    fn get_name(&self) -> &str {
        // Same as: fn get_name<'a>(&'a self) -> &'a str
        &self.name
    }
}
```

---

## Traits & Generics

### 13. What is the difference between `impl Trait` and `dyn Trait`?

**Answer:**
- **`impl Trait`** (static dispatch): Compiler generates specific code for each type. Zero runtime cost.
- **`dyn Trait`** (dynamic dispatch): Uses vtable at runtime. Has small runtime overhead but allows heterogeneous collections.

```rust
// Static dispatch - monomorphized at compile time
fn print_static(item: impl Display) {
    println!("{}", item);
}

// Dynamic dispatch - uses vtable at runtime
fn print_dynamic(item: &dyn Display) {
    println!("{}", item);
}

// dyn allows heterogeneous collections
let items: Vec<Box<dyn Display>> = vec![
    Box::new(42),
    Box::new("hello"),
];
```

---

### 14. What is a trait object?

**Answer:** A trait object is a way to use dynamic dispatch. It consists of a pointer to data and a pointer to a vtable.

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;
struct Cat;

impl Animal for Dog {
    fn speak(&self) { println!("Woof!"); }
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

---

### 15. What are associated types in traits?

**Answer:** Associated types are placeholder types within a trait that implementors must specify.

```rust
trait Iterator {
    type Item;  // Associated type
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;  // Specify the associated type

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        Some(self.count)
    }
}
```

---

### 16. What is the difference between associated types and generics?

**Answer:**
- **Generics**: Allow multiple implementations for different types
- **Associated types**: Only one implementation per type

```rust
// With generics - can implement multiple times
trait ConvertTo<T> {
    fn convert(&self) -> T;
}

impl ConvertTo<i32> for String { /* ... */ }
impl ConvertTo<f64> for String { /* ... */ }

// With associated type - only one implementation
trait IntoIterator {
    type Item;
    type IntoIter;
    fn into_iter(self) -> Self::IntoIter;
}
// Vec<T> can only have ONE IntoIterator implementation
```

---

### 17. Explain trait bounds and the `where` clause.

**Answer:** Trait bounds constrain generic types to those implementing specific traits.

```rust
// Inline trait bounds
fn print_debug<T: Debug>(item: T) {
    println!("{:?}", item);
}

// Multiple bounds
fn compare<T: PartialOrd + Debug>(a: T, b: T) {
    println!("{:?} vs {:?}", a, b);
}

// Where clause - cleaner for complex bounds
fn complex_function<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}
```

---

## Error Handling

### 18. What is the difference between `Result` and `Option`?

**Answer:**
- **`Option<T>`**: Represents optional value. `Some(T)` or `None`.
- **`Result<T, E>`**: Represents success or failure. `Ok(T)` or `Err(E)`.

```rust
// Option - for optional values
fn find_user(id: u32) -> Option<User> {
    if id == 1 {
        Some(User { name: "Alice".into() })
    } else {
        None
    }
}

// Result - for operations that can fail
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Cannot divide by zero".into())
    } else {
        Ok(a / b)
    }
}
```

---

### 19. What is the difference between `unwrap()`, `expect()`, and `?`?

**Answer:**
- **`unwrap()`**: Panics with generic message on error
- **`expect(msg)`**: Panics with custom message on error
- **`?`**: Propagates error to caller (doesn't panic)

```rust
fn main() {
    let result: Result<i32, &str> = Ok(42);

    // unwrap - panics on Err
    let value = result.unwrap();

    // expect - panics with custom message
    let value = result.expect("Failed to get value");
}

// ? operator - propagates error
fn process() -> Result<i32, &str> {
    let result: Result<i32, &str> = Ok(42);
    let value = result?;  // Returns Err early if error
    Ok(value * 2)
}
```

---

### 20. How do you create custom error types?

**Answer:** Use enums with `std::error::Error` trait implementation.

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
            MyError::NotFound(msg) => write!(f, "Not found: {}", msg),
            MyError::InvalidInput(msg) => write!(f, "Invalid input: {}", msg),
            MyError::IoError(e) => write!(f, "IO error: {}", e),
        }
    }
}

impl Error for MyError {}

// Using thiserror crate (recommended)
use thiserror::Error;

#[derive(Error, Debug)]
enum MyError {
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    #[error("IO error")]
    IoError(#[from] std::io::Error),
}
```

---

## Concurrency

### 21. How does Rust ensure thread safety?

**Answer:** Rust uses two marker traits:
- **`Send`**: Type can be transferred between threads
- **`Sync`**: Type can be shared between threads (via references)

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

---

### 22. What is the difference between `Rc` and `Arc`?

**Answer:**
- **`Rc<T>`** (Reference Counted): Single-threaded reference counting. NOT `Send` or `Sync`.
- **`Arc<T>`** (Atomic Reference Counted): Thread-safe reference counting. Uses atomic operations.

```rust
use std::rc::Rc;
use std::sync::Arc;

// Rc - single thread only
let rc = Rc::new(5);
let rc2 = Rc::clone(&rc);

// Arc - thread-safe
let arc = Arc::new(5);
let arc2 = Arc::clone(&arc);

// Arc can be sent to other threads
std::thread::spawn(move || {
    println!("{}", arc2);
});
```

---

### 23. What is `Mutex` and how is it used?

**Answer:** `Mutex` provides mutual exclusion, allowing only one thread to access data at a time.

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

---

### 24. What is `RwLock` and when would you use it over `Mutex`?

**Answer:** `RwLock` allows multiple readers OR one writer. Use when reads are more frequent than writes.

```rust
use std::sync::RwLock;

let lock = RwLock::new(5);

// Multiple readers allowed
{
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
    println!("{} {}", r1, r2);
}

// Only one writer
{
    let mut w = lock.write().unwrap();
    *w += 1;
}
```

---

### 25. Explain channels in Rust.

**Answer:** Channels provide message passing between threads. `mpsc` = multiple producer, single consumer.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // Clone for multiple producers
    let tx1 = tx.clone();

    thread::spawn(move || {
        tx.send("from thread 1").unwrap();
    });

    thread::spawn(move || {
        tx1.send("from thread 2").unwrap();
    });

    // Receive all messages
    for received in rx {
        println!("Got: {}", received);
    }
}
```

---

## Memory Management

### 26. What is RAII in Rust?

**Answer:** RAII (Resource Acquisition Is Initialization) means resources are acquired during initialization and released when the object goes out of scope.

```rust
fn main() {
    {
        let file = File::create("test.txt").unwrap();
        // file is open here
        file.write_all(b"Hello").unwrap();
    } // file is automatically closed here (Drop is called)

    {
        let mutex = Mutex::new(0);
        let guard = mutex.lock().unwrap();
        // mutex is locked
    } // mutex is automatically unlocked here
}
```

---

### 27. What is the `Drop` trait?

**Answer:** `Drop` trait defines cleanup behavior when a value goes out of scope.

```rust
struct CustomResource {
    name: String,
}

impl Drop for CustomResource {
    fn drop(&mut self) {
        println!("Dropping resource: {}", self.name);
    }
}

fn main() {
    let r1 = CustomResource { name: "Resource 1".into() };
    let r2 = CustomResource { name: "Resource 2".into() };
    println!("Resources created");
} // Prints: Dropping resource: Resource 2, then Resource 1 (reverse order)

// Manual drop
fn main() {
    let r = CustomResource { name: "Early drop".into() };
    drop(r);  // Drops immediately
    println!("After drop");
}
```

---

### 28. What is the difference between stack and heap allocation in Rust?

**Answer:**
- **Stack**: Fixed-size data, fast allocation/deallocation, automatic cleanup
- **Heap**: Dynamic-size data, slower, requires explicit allocation

```rust
fn main() {
    // Stack allocated
    let x: i32 = 5;
    let arr: [i32; 3] = [1, 2, 3];

    // Heap allocated
    let s: String = String::from("hello");
    let v: Vec<i32> = vec![1, 2, 3];
    let b: Box<i32> = Box::new(5);
}
```

---

## Smart Pointers

### 29. What is `Box<T>` and when would you use it?

**Answer:** `Box<T>` is a smart pointer for heap allocation. Use cases:
- Recursive types (need known size at compile time)
- Large data you want to transfer ownership without copying
- Trait objects

```rust
// Recursive type - needs Box
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
    // Ownership transferred without copying
}
```

---

### 30. What is `Cow` (Copy on Write)?

**Answer:** `Cow` is a smart pointer that can hold either borrowed or owned data. Clones only when mutation is needed.

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains(' ') {
        // Need to modify - return owned
        Cow::Owned(input.replace(' ', "_"))
    } else {
        // No modification needed - return borrowed
        Cow::Borrowed(input)
    }
}

fn main() {
    let s1 = process("hello");        // Borrowed, no allocation
    let s2 = process("hello world");  // Owned, allocated
}
```

---

### 31. Explain `RefCell<T>` and interior mutability.

**Answer:** `RefCell<T>` provides interior mutability - mutating data even with immutable references. Borrow checking happens at runtime.

```rust
use std::cell::RefCell;

struct Counter {
    value: RefCell<i32>,
}

impl Counter {
    fn increment(&self) {  // Note: &self, not &mut self
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

---

### 32. What is the difference between `Cell` and `RefCell`?

**Answer:**
- **`Cell<T>`**: For `Copy` types. Get/set values directly. No runtime borrow checking.
- **`RefCell<T>`**: For any type. Returns references. Runtime borrow checking (can panic).

```rust
use std::cell::{Cell, RefCell};

let cell: Cell<i32> = Cell::new(5);
cell.set(10);
let value = cell.get();  // Copy

let refcell: RefCell<String> = RefCell::new("hello".into());
refcell.borrow_mut().push_str(" world");
let borrowed = refcell.borrow();  // Reference
```

---

## Closures & Iterators

### 33. What are the three closure traits: `Fn`, `FnMut`, `FnOnce`?

**Answer:**
- **`FnOnce`**: Takes ownership of captured variables. Can only be called once.
- **`FnMut`**: Mutably borrows captured variables. Can be called multiple times.
- **`Fn`**: Immutably borrows captured variables. Can be called multiple times.

```rust
// FnOnce - takes ownership
let s = String::from("hello");
let consume = move || {
    drop(s);  // s is moved into closure
};
consume();
// consume();  // Error: can't call twice

// FnMut - mutates
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

---

### 34. What is the difference between `iter()`, `iter_mut()`, and `into_iter()`?

**Answer:**
- **`iter()`**: Borrows elements (`&T`)
- **`iter_mut()`**: Mutably borrows elements (`&mut T`)
- **`into_iter()`**: Takes ownership of elements (`T`)

```rust
let mut vec = vec![1, 2, 3];

// iter() - borrows
for x in vec.iter() {
    println!("{}", x);  // x is &i32
}

// iter_mut() - mutable borrow
for x in vec.iter_mut() {
    *x *= 2;  // x is &mut i32
}

// into_iter() - ownership
for x in vec.into_iter() {
    println!("{}", x);  // x is i32
}
// vec is no longer accessible
```

---

### 35. Explain iterator adaptors and consumers.

**Answer:**
- **Adaptors**: Transform iterators, lazy evaluation (map, filter, take, skip)
- **Consumers**: Consume iterators, produce values (collect, sum, count, for_each)

```rust
let numbers = vec![1, 2, 3, 4, 5];

// Adaptors are lazy - nothing happens until consumed
let doubled = numbers.iter()
    .map(|x| x * 2)
    .filter(|x| *x > 4);

// Consumer triggers evaluation
let result: Vec<_> = doubled.collect();  // [6, 8, 10]

// Chaining
let sum: i32 = (1..=100)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
```

---

## Macros

### 36. What is the difference between declarative and procedural macros?

**Answer:**
- **Declarative macros** (`macro_rules!`): Pattern matching on code
- **Procedural macros**: Operate on token streams, more powerful

```rust
// Declarative macro
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

say_hello!();           // Hello!
say_hello!("World");    // Hello, World!

// Procedural macro (derive)
#[derive(Debug, Clone)]  // These are procedural macros
struct MyStruct {
    field: i32,
}
```

---

### 37. What are the types of procedural macros?

**Answer:**
1. **Derive macros**: Add implementations via `#[derive(MacroName)]`
2. **Attribute macros**: Custom attributes `#[macro_name]`
3. **Function-like macros**: Called like functions `macro_name!(...)`

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

### 38. How does async/await work in Rust?

**Answer:** `async` functions return a `Future` that must be polled to completion by an executor (like tokio).

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

---

### 39. What is `Pin` and why is it needed?

**Answer:** `Pin` prevents moving data in memory. Required for self-referential async futures.

```rust
use std::pin::Pin;
use std::future::Future;

// Most common use - pinning futures
async fn example() {
    let future = async { 42 };
    let pinned = Box::pin(future);
    // pinned can now be polled
}

// Pin prevents moving
fn poll_future(future: Pin<&mut dyn Future<Output = i32>>) {
    // future cannot be moved out
}
```

---

### 40. What is the difference between `tokio::spawn` and `async` blocks?

**Answer:**
- **`async` blocks**: Create a future, runs on current task
- **`tokio::spawn`**: Creates a new task that runs concurrently

```rust
use tokio;

#[tokio::main]
async fn main() {
    // Sequential - one after another
    let a = async { 1 }.await;
    let b = async { 2 }.await;

    // Concurrent - spawn separate tasks
    let handle1 = tokio::spawn(async { 1 });
    let handle2 = tokio::spawn(async { 2 });

    let (a, b) = (handle1.await.unwrap(), handle2.await.unwrap());

    // Or use join! for concurrent futures on same task
    let (x, y) = tokio::join!(
        async { 1 },
        async { 2 }
    );
}
```

---

## Coding Questions

### 41. Implement a function to reverse a string.

```rust
fn reverse_string(s: &str) -> String {
    s.chars().rev().collect()
}

// In-place for Vec<char>
fn reverse_in_place(chars: &mut [char]) {
    let len = chars.len();
    for i in 0..len / 2 {
        chars.swap(i, len - 1 - i);
    }
}
```

---

### 42. Implement a basic linked list.

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

### 43. Implement a thread-safe counter.

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

### 44. Implement the Builder pattern.

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

### 45. What will this code print? (Ownership quiz)

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    // println!("{}", s1);  // This would error
    println!("{}", s2);
}
```

**Answer:** Prints `hello`. `s1` is moved to `s2`, making `s1` invalid. Only `s2` can be used.

---

### 46. What will this code print? (Lifetime quiz)

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

**Answer:** This code won't compile! `r` would be a dangling reference because `x` goes out of scope before `r` is used.

---

### 47. Fix this code:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

**Answer:** Add lifetime annotations:
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

---

### 48. Implement a simple cache using HashMap.

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

- [ ] Ownership, borrowing, and lifetimes
- [ ] `String` vs `&str`
- [ ] `Vec`, arrays, and slices
- [ ] `Result` and `Option` error handling
- [ ] Traits and generics
- [ ] `Box`, `Rc`, `Arc`, `RefCell`
- [ ] Closures (`Fn`, `FnMut`, `FnOnce`)
- [ ] Iterators and adaptors
- [ ] Concurrency (`Mutex`, `RwLock`, channels)
- [ ] `Send` and `Sync` traits
- [ ] Pattern matching
- [ ] Macros basics
- [ ] Async/await fundamentals
- [ ] Memory layout (stack vs heap)
- [ ] Zero-cost abstractions

---

## Tips for Rust Interviews

1. **Understand ownership deeply** - This is the #1 topic in Rust interviews
2. **Practice explaining borrow checker errors** - Interviewers love to ask "why doesn't this compile?"
3. **Know the standard library** - `Option`, `Result`, `Vec`, `HashMap`, `Iterator` methods
4. **Understand when to use which smart pointer** - `Box` vs `Rc` vs `Arc` vs `RefCell`
5. **Be able to write safe concurrent code** - Using `Arc<Mutex<T>>` patterns
6. **Know the difference between static and dynamic dispatch** - `impl Trait` vs `dyn Trait`
7. **Practice coding on paper/whiteboard** - Many interviews don't have compiler access

Good luck with your Rust interviews!
