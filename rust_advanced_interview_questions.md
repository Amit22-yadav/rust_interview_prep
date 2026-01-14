# Advanced Rust Interview Questions

A collection of advanced Rust interview questions for senior positions and deep technical discussions.

---

## Table of Contents
1. [Unsafe Rust](#unsafe-rust)
2. [Advanced Lifetimes](#advanced-lifetimes)
3. [Advanced Traits](#advanced-traits)
4. [Type System Deep Dive](#type-system-deep-dive)
5. [Memory Layout & Optimization](#memory-layout--optimization)
6. [Advanced Concurrency](#advanced-concurrency)
7. [Advanced Async](#advanced-async)
8. [FFI & Interoperability](#ffi--interoperability)
9. [Compiler Internals](#compiler-internals)
10. [Performance & Optimization](#performance--optimization)
11. [Design Patterns](#design-patterns)
12. [Advanced Coding Challenges](#advanced-coding-challenges)

---

## Unsafe Rust

### 1. What are the five things you can do in unsafe Rust that you can't do in safe Rust?

**Answer:**
1. Dereference raw pointers (`*const T`, `*mut T`)
2. Call unsafe functions or methods
3. Access or modify mutable static variables
4. Implement unsafe traits
5. Access fields of unions

```rust
unsafe {
    // 1. Dereference raw pointer
    let ptr: *const i32 = &10;
    println!("{}", *ptr);

    // 2. Call unsafe function
    unsafe fn dangerous() {}
    dangerous();

    // 3. Access mutable static
    static mut COUNTER: i32 = 0;
    COUNTER += 1;

    // 4. Implement unsafe trait (e.g., Send, Sync manually)
    // 5. Access union fields
    union MyUnion { i: i32, f: f32 }
    let u = MyUnion { i: 42 };
    println!("{}", u.i);
}
```

---

### 2. What is the difference between `*const T` and `*mut T`?

**Answer:**
- `*const T`: Raw pointer for reading (immutable)
- `*mut T`: Raw pointer for reading and writing (mutable)

Both can be created safely, but dereferencing requires `unsafe`.

```rust
fn main() {
    let mut value = 42;

    // Creating raw pointers is safe
    let const_ptr: *const i32 = &value;
    let mut_ptr: *mut i32 = &mut value;

    unsafe {
        // Dereferencing requires unsafe
        println!("const: {}", *const_ptr);
        *mut_ptr = 100;
        println!("mut: {}", *mut_ptr);
    }
}
```

---

### 3. Explain the concept of "unsafe boundaries" and how to design safe abstractions over unsafe code.

**Answer:** Unsafe boundaries mean encapsulating unsafe code within a safe API. The unsafe implementation details are hidden, and the public interface maintains Rust's safety guarantees.

```rust
pub struct SafeVec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

impl<T> SafeVec<T> {
    pub fn new() -> Self {
        SafeVec {
            ptr: std::ptr::null_mut(),
            len: 0,
            cap: 0,
        }
    }

    // Safe public API
    pub fn push(&mut self, value: T) {
        if self.len == self.cap {
            self.grow();  // Unsafe internally
        }
        unsafe {
            // SAFETY: We just ensured capacity
            std::ptr::write(self.ptr.add(self.len), value);
        }
        self.len += 1;
    }

    pub fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            unsafe {
                // SAFETY: index is bounds-checked
                Some(&*self.ptr.add(index))
            }
        } else {
            None
        }
    }

    fn grow(&mut self) {
        // ... unsafe allocation logic
    }
}

// Users interact only with safe methods
fn main() {
    let mut v = SafeVec::new();
    v.push(1);
    v.push(2);
    println!("{:?}", v.get(0));  // Safe!
}
```

---

### 4. What is `MaybeUninit<T>` and when would you use it?

**Answer:** `MaybeUninit<T>` represents potentially uninitialized memory. It's used when you need to initialize values incrementally or when interfacing with C.

```rust
use std::mem::MaybeUninit;

// Initializing an array element by element
fn create_array() -> [String; 3] {
    let mut arr: [MaybeUninit<String>; 3] = unsafe {
        MaybeUninit::uninit().assume_init()
    };

    for (i, elem) in arr.iter_mut().enumerate() {
        elem.write(format!("Element {}", i));
    }

    // SAFETY: All elements are now initialized
    unsafe {
        std::mem::transmute::<_, [String; 3]>(arr)
    }
}

// Better approach with array::from_fn (Rust 1.63+)
fn create_array_safe() -> [String; 3] {
    std::array::from_fn(|i| format!("Element {}", i))
}
```

---

### 5. What are the rules for implementing `Send` and `Sync` manually?

**Answer:**
- `Send`: Safe to transfer ownership between threads
- `Sync`: Safe to share references between threads (`T: Sync` iff `&T: Send`)

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

// Custom type with manual Send/Sync
struct MyCell<T> {
    value: UnsafeCell<T>,
    readers: AtomicUsize,
}

// SAFETY: We guarantee thread-safe access through atomic operations
unsafe impl<T: Send> Send for MyCell<T> {}
unsafe impl<T: Send> Sync for MyCell<T> {}

impl<T> MyCell<T> {
    fn new(value: T) -> Self {
        MyCell {
            value: UnsafeCell::new(value),
            readers: AtomicUsize::new(0),
        }
    }

    fn get(&self) -> &T {
        self.readers.fetch_add(1, Ordering::Acquire);
        unsafe { &*self.value.get() }
    }
}
```

**Common types that are NOT Send/Sync:**
- `Rc<T>`: Not Send, Not Sync (use `Arc` instead)
- `RefCell<T>`: Send (if T: Send), Not Sync
- `*const T`, `*mut T`: Not Send, Not Sync by default

---

## Advanced Lifetimes

### 6. Explain Higher-Ranked Trait Bounds (HRTBs) with `for<'a>`.

**Answer:** HRTBs allow expressing that a trait bound must hold for *all* possible lifetimes, not just a specific one.

```rust
// Without HRTB - specific lifetime
fn call_with_ref<'a, F>(f: F)
where
    F: Fn(&'a str),
{
    let s = String::from("hello");
    // f(&s);  // Error: 'a doesn't match local lifetime
}

// With HRTB - works for ANY lifetime
fn call_with_ref_hrtb<F>(f: F)
where
    F: for<'a> Fn(&'a str),  // F works for ALL lifetimes
{
    let s = String::from("hello");
    f(&s);  // OK!
}

fn main() {
    call_with_ref_hrtb(|s| println!("{}", s));
}
```

**Real-world example:** `Fn` traits use HRTBs internally:
```rust
// This signature:
fn takes_closure<F: Fn(&str)>(f: F) {}

// Is actually shorthand for:
fn takes_closure_explicit<F: for<'a> Fn(&'a str)>(f: F) {}
```

---

### 7. What is lifetime variance and how does it affect your code?

**Answer:** Variance describes how lifetimes can be substituted:
- **Covariant**: Can substitute longer lifetime for shorter (`&'a T` is covariant in `'a`)
- **Contravariant**: Can substitute shorter lifetime for longer (rare, in function arguments)
- **Invariant**: No substitution allowed (`&'a mut T` is invariant in `T`)

```rust
// Covariance - 'static can be used where 'a is expected
fn covariant<'a>(s: &'a str) -> &'a str { s }
let static_str: &'static str = "hello";
let result: &str = covariant(static_str);  // OK: 'static -> 'a

// Invariance with mutable references
fn invariant<'a>(s: &'a mut Vec<&'a str>) {}
// Cannot pass &mut Vec<&'static str> where &mut Vec<&'a str> expected

// Why invariance matters:
fn evil<'a, 'b>(v: &'a mut Vec<&'b str>, s: &'b str) {
    // If this were allowed:
    // v.push(s);  // Push short-lived &'b str
    // v could outlive 'b, creating dangling references
}
```

---

### 8. Explain the difference between `'a: 'b` and `'b: 'a`.

**Answer:** `'a: 'b` means lifetime `'a` outlives (or equals) lifetime `'b`.

```rust
// 'a: 'b means 'a lives at least as long as 'b
fn example<'a, 'b>(x: &'a str, y: &'b str) -> &'b str
where
    'a: 'b,  // 'a outlives 'b
{
    x  // OK: x lives longer than required 'b
}

// Common pattern: return reference with shorter lifetime
struct Context<'a> {
    data: &'a str,
}

impl<'a> Context<'a> {
    fn get_data<'b>(&'b self) -> &'b str
    where
        'a: 'b,  // data outlives the borrow of self
    {
        self.data
    }
}
```

---

### 9. What are lifetime elision rules in `impl` blocks?

**Answer:** In `impl` blocks, the lifetime of `&self` or `&mut self` is automatically assigned to output references.

```rust
struct Parser<'a> {
    input: &'a str,
}

impl<'a> Parser<'a> {
    // Elided: fn new(input: &'a str) -> Parser<'a>
    fn new(input: &str) -> Parser {
        Parser { input }
    }

    // Elided: fn parse<'b>(&'b self) -> &'b str
    // Note: returns reference tied to &self, not 'a
    fn current(&self) -> &str {
        &self.input[..1]
    }

    // Explicit when returning data's lifetime, not self's
    fn remaining(&self) -> &'a str {
        self.input
    }
}
```

---

### 10. How do you handle self-referential structs in Rust?

**Answer:** Self-referential structs are challenging because moves invalidate internal pointers. Solutions:

```rust
// Option 1: Use indices instead of references
struct SelfRefWithIndex {
    data: Vec<String>,
    current_index: usize,
}

// Option 2: Use Pin + unsafe
use std::pin::Pin;
use std::marker::PhantomPinned;

struct SelfRef {
    value: String,
    pointer: *const String,
    _pin: PhantomPinned,
}

impl SelfRef {
    fn new(value: String) -> Pin<Box<Self>> {
        let mut boxed = Box::new(SelfRef {
            value,
            pointer: std::ptr::null(),
            _pin: PhantomPinned,
        });
        let ptr = &boxed.value as *const String;
        boxed.pointer = ptr;
        Box::into_pin(boxed)
    }

    fn get_value(self: Pin<&Self>) -> &str {
        &self.value
    }

    fn get_pointer(self: Pin<&Self>) -> &String {
        unsafe { &*self.pointer }
    }
}

// Option 3: Use ouroboros or self_cell crate
// These provide safe abstractions for self-referential structs
```

---

## Advanced Traits

### 11. Explain object safety rules for trait objects.

**Answer:** A trait is object-safe if it can be used as `dyn Trait`. Rules:

1. Methods must not return `Self`
2. Methods must not have generic type parameters
3. The trait must not require `Self: Sized`

```rust
// Object-safe trait
trait ObjectSafe {
    fn method(&self);
    fn method_with_default(&self) {}
}

// NOT object-safe
trait NotObjectSafe {
    fn returns_self(&self) -> Self;           // Returns Self
    fn generic<T>(&self, t: T);               // Generic parameter
    fn sized_only(&self) where Self: Sized;   // Sized bound (but can be excluded)
}

// Making a trait partially object-safe
trait PartiallyObjectSafe {
    fn object_safe_method(&self);

    // Exclude from trait object with Sized bound
    fn not_object_safe(&self) -> Self where Self: Sized;
}

// Now dyn PartiallyObjectSafe works, but can't call not_object_safe()
fn use_trait_object(obj: &dyn PartiallyObjectSafe) {
    obj.object_safe_method();  // OK
    // obj.not_object_safe();  // Not available on dyn
}
```

---

### 12. What are blanket implementations?

**Answer:** Blanket implementations implement a trait for all types that satisfy certain bounds.

```rust
// Standard library example: impl<T: Display> ToString for T
use std::fmt::Display;

trait Printable {
    fn print(&self);
}

// Blanket implementation for all Display types
impl<T: Display> Printable for T {
    fn print(&self) {
        println!("{}", self);
    }
}

fn main() {
    42.print();           // Works for i32
    "hello".print();      // Works for &str
    3.14f64.print();      // Works for f64
}

// Another example: From/Into
// impl<T, U> Into<U> for T where U: From<T>
// If you implement From, you get Into for free!
```

---

### 13. Explain the `Deref` and `DerefMut` traits and deref coercion.

**Answer:** `Deref` allows customizing the dereference operator `*`. Deref coercion automatically converts references.

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

fn main() {
    let x = MyBox(String::from("hello"));

    // Explicit dereference
    let s: &String = &*x;

    // Deref coercion: &MyBox<String> -> &String -> &str
    fn takes_str(s: &str) {}
    takes_str(&x);  // Automatic coercion!

    // Coercion chain:
    // &MyBox<String> -> &String (via Deref)
    // &String -> &str (via String's Deref)
}
```

**Deref coercion rules:**
- `&T` to `&U` when `T: Deref<Target=U>`
- `&mut T` to `&mut U` when `T: DerefMut<Target=U>`
- `&mut T` to `&U` when `T: Deref<Target=U>` (mutable to immutable)

---

### 14. What is the `PhantomData` marker and when is it used?

**Answer:** `PhantomData<T>` is a zero-sized type that tells the compiler to act as if it owns `T`, affecting variance and drop checking.

```rust
use std::marker::PhantomData;

// Use case 1: Indicate ownership without storing data
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<&'a T>,  // Indicates lifetime relationship
}

// Use case 2: Unused type parameter
struct TypedId<T> {
    id: u64,
    _marker: PhantomData<T>,  // T isn't stored but is part of the type
}

type UserId = TypedId<User>;
type PostId = TypedId<Post>;

// Use case 3: Indicate ownership for drop check
struct OwnedPtr<T> {
    ptr: *mut T,
    _owns: PhantomData<T>,  // Compiler knows we own T for drop purposes
}

impl<T> Drop for OwnedPtr<T> {
    fn drop(&mut self) {
        unsafe {
            std::ptr::drop_in_place(self.ptr);
        }
    }
}
```

---

### 15. Explain the difference between `AsRef`, `Borrow`, and `Deref`.

**Answer:**

| Trait | Purpose | Use Case |
|-------|---------|----------|
| `Deref` | Smart pointer dereference | Automatic coercion, `*x` |
| `AsRef` | Cheap reference conversion | Generic functions accepting various types |
| `Borrow` | Borrowing with hash/eq consistency | HashMap/HashSet keys |

```rust
use std::borrow::Borrow;

// AsRef - cheap conversion, no semantic guarantees
fn print_path<P: AsRef<std::path::Path>>(path: P) {
    println!("{:?}", path.as_ref());
}
print_path("file.txt");           // &str -> &Path
print_path(String::from("x"));    // String -> &Path

// Borrow - guarantees hash/eq consistency
fn get_from_map<Q: ?Sized>(map: &HashMap<String, i32>, key: &Q) -> Option<&i32>
where
    String: Borrow<Q>,
    Q: Hash + Eq,
{
    map.get(key)
}
// Can lookup with &str even though key is String
let value = get_from_map(&map, "key");

// Deref - automatic coercion chain
let s = String::from("hello");
let r: &str = &s;  // String derefs to str
```

---

## Type System Deep Dive

### 16. Explain Rust's type coercion rules.

**Answer:** Rust performs implicit coercions in limited cases:

```rust
// 1. Deref coercion
let s = String::from("hello");
let r: &str = &s;  // &String -> &str

// 2. Pointer weakening
let x: &mut i32 = &mut 5;
let y: &i32 = x;  // &mut T -> &T

// 3. Unsizing coercion
let arr: [i32; 3] = [1, 2, 3];
let slice: &[i32] = &arr;  // [T; N] -> [T]

let concrete: Box<String> = Box::new("hi".into());
let trait_obj: Box<dyn Display> = concrete;  // T -> dyn Trait

// 4. Function pointer coercion
fn foo() {}
let f: fn() = foo;  // Named function -> fn pointer

// 5. Closure to fn pointer (if no captures)
let closure = |x: i32| x + 1;
let fn_ptr: fn(i32) -> i32 = closure;

// 6. Never type coercion
fn diverges() -> ! { panic!() }
let x: i32 = if true { 5 } else { diverges() };  // ! coerces to any type
```

---

### 17. What is the `!` (never) type?

**Answer:** The never type `!` represents computations that never complete. It coerces to any other type.

```rust
// Functions that never return
fn infinite_loop() -> ! {
    loop {}
}

fn always_panics() -> ! {
    panic!("boom");
}

// Used in match arms
fn unwrap_or_die<T>(opt: Option<T>) -> T {
    match opt {
        Some(v) => v,
        None => panic!("was None"),  // panic! returns !
    }
}

// Useful in enums
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// Infallible conversion
type Infallible = !;  // std::convert::Infallible
impl<T> From<!> for T {
    fn from(never: !) -> T {
        match never {}  // Empty match on never type
    }
}
```

---

### 18. Explain the difference between `Sized` and `?Sized`.

**Answer:**
- `Sized`: Type has known size at compile time (default bound)
- `?Sized`: Type may or may not be sized (DST - dynamically sized type)

```rust
// Implicitly Sized
fn takes_sized<T>(t: T) {}  // T: Sized implied

// Explicitly ?Sized to accept DSTs
fn takes_unsized<T: ?Sized>(t: &T) {}  // Can take &str, &[i32], &dyn Trait

// DST examples
let s: &str = "hello";         // str is unsized
let slice: &[i32] = &[1, 2];   // [i32] is unsized
let obj: &dyn Display = &42;    // dyn Display is unsized

// Structs with DST field (must be last)
struct MySlice<T: ?Sized> {
    len: usize,
    data: T,  // Must be last field
}

// Creating unsized struct
let boxed: Box<MySlice<[i32]>> = todo!();
```

---

### 19. What are const generics and how are they used?

**Answer:** Const generics allow types to be parameterized by constant values, not just types.

```rust
// Array with generic size
struct Array<T, const N: usize> {
    data: [T; N],
}

impl<T: Default + Copy, const N: usize> Array<T, N> {
    fn new() -> Self {
        Array { data: [T::default(); N] }
    }

    fn len(&self) -> usize {
        N  // Compile-time constant
    }
}

// Generic over array size
fn print_array<T: Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}

print_array([1, 2, 3]);      // N = 3
print_array([1, 2, 3, 4]);   // N = 4

// Const expressions
struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [[T; COLS]; ROWS],
}

impl<T: Default + Copy, const R: usize, const C: usize> Matrix<T, R, C> {
    fn transpose(self) -> Matrix<T, C, R> {
        todo!()
    }
}
```

---

### 20. Explain GATs (Generic Associated Types).

**Answer:** GATs allow associated types to have their own generic parameters.

```rust
// Without GATs - can't express lending iterator
trait LendingIterator {
    type Item<'a> where Self: 'a;  // GAT with lifetime parameter

    fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
}

// Implementation
struct WindowsMut<'t, T> {
    slice: &'t mut [T],
    start: usize,
}

impl<'t, T> LendingIterator for WindowsMut<'t, T> {
    type Item<'a> = &'a mut [T] where Self: 'a;

    fn next<'a>(&'a mut self) -> Option<Self::Item<'a>> {
        let window = self.slice.get_mut(self.start..self.start + 2)?;
        self.start += 1;
        Some(window)
    }
}

// GAT with type parameter
trait Container {
    type Item<T>;
    fn wrap<T>(&self, item: T) -> Self::Item<T>;
}

struct MyContainer;

impl Container for MyContainer {
    type Item<T> = Option<T>;

    fn wrap<T>(&self, item: T) -> Self::Item<T> {
        Some(item)
    }
}
```

---

## Memory Layout & Optimization

### 21. Explain struct memory layout and padding.

**Answer:** Rust structs have padding for alignment. Use `#[repr(C)]` or `#[repr(packed)]` for control.

```rust
use std::mem::{size_of, align_of};

// Default layout - compiler may reorder fields
struct Default {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes
    c: u8,   // 1 byte
}
// Size: 24 bytes (with padding for alignment)

// C-compatible layout
#[repr(C)]
struct CLayout {
    a: u8,   // 1 byte + 7 padding
    b: u64,  // 8 bytes
    c: u8,   // 1 byte + 7 padding
}
// Size: 24 bytes (fields in declared order)

// Packed - no padding
#[repr(packed)]
struct Packed {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes
    c: u8,   // 1 byte
}
// Size: 10 bytes (no padding, unaligned access)

// Optimized manual ordering
struct Optimized {
    b: u64,  // 8 bytes
    a: u8,   // 1 byte
    c: u8,   // 1 byte + 6 padding
}
// Size: 16 bytes

fn main() {
    println!("Default: {} bytes", size_of::<Default>());
    println!("CLayout: {} bytes", size_of::<CLayout>());
    println!("Packed: {} bytes", size_of::<Packed>());
    println!("Optimized: {} bytes", size_of::<Optimized>());
}
```

---

### 22. What is niche optimization?

**Answer:** Rust uses "niches" (invalid bit patterns) in types to store enum discriminants without extra space.

```rust
use std::mem::size_of;

// Option<&T> uses null pointer niche
// &T can never be null, so None uses 0x0
assert_eq!(size_of::<&i32>(), size_of::<Option<&i32>>());

// NonZero types have niches
use std::num::NonZeroU64;
assert_eq!(size_of::<NonZeroU64>(), size_of::<Option<NonZeroU64>>());

// bool only uses 0 and 1, values 2-255 are niches
assert_eq!(size_of::<bool>(), size_of::<Option<bool>>());

// Custom types with niches
#[repr(u8)]
enum Status {
    Active = 0,
    Inactive = 1,
    // Values 2-255 are niches
}
assert_eq!(size_of::<Status>(), size_of::<Option<Status>>());

// No niche - needs extra byte
struct NoNiche {
    value: u8,
}
assert_eq!(size_of::<NoNiche>(), 1);
assert_eq!(size_of::<Option<NoNiche>>(), 2);  // Needs discriminant
```

---

### 23. Explain the difference between `Box<[T]>` and `Vec<T>`.

**Answer:**

| Feature | `Box<[T]>` | `Vec<T>` |
|---------|------------|----------|
| Size | 2 words (ptr + len) | 3 words (ptr + len + capacity) |
| Growable | No | Yes |
| Shrink | Fixed | Can shrink |
| Conversion | From Vec | Can convert to Box<[T]> |

```rust
use std::mem::size_of;

fn main() {
    // Vec has capacity
    let vec: Vec<i32> = vec![1, 2, 3];
    println!("Vec size: {} bytes", size_of::<Vec<i32>>());  // 24 bytes

    // Box<[T]> has no capacity
    let boxed: Box<[i32]> = vec.into_boxed_slice();
    println!("Box<[i32]> size: {} bytes", size_of::<Box<[i32]>>());  // 16 bytes

    // Convert back (may reallocate)
    let vec_again: Vec<i32> = boxed.into_vec();
}
```

---

### 24. What is zero-cost abstraction and how does Rust achieve it?

**Answer:** Zero-cost abstractions have no runtime overhead compared to hand-written low-level code. Rust achieves this through:

1. **Monomorphization**: Generics are compiled to specific types
2. **Inlining**: Small functions are inlined
3. **No runtime overhead**: Iterators compile to loops

```rust
// Iterator chain compiles to simple loop
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * 2)
    .sum();

// Compiles to roughly:
let mut sum = 0;
for x in 0..1000 {
    if x % 2 == 0 {
        sum += x * 2;
    }
}

// Generic function
fn add<T: std::ops::Add<Output = T>>(a: T, b: T) -> T {
    a + b
}

// Generates specific versions:
// add::<i32>(i32, i32) -> i32
// add::<f64>(f64, f64) -> f64
// No vtable, no dynamic dispatch
```

---

## Advanced Concurrency

### 25. Explain memory ordering in atomics.

**Answer:** Memory ordering controls how atomic operations synchronize between threads.

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};

// Relaxed - no synchronization, just atomic
let counter = AtomicUsize::new(0);
counter.fetch_add(1, Ordering::Relaxed);

// Acquire/Release - synchronize specific data
let data = AtomicUsize::new(0);
let ready = AtomicBool::new(false);

// Thread 1: Producer
data.store(42, Ordering::Relaxed);
ready.store(true, Ordering::Release);  // Release: all writes before visible

// Thread 2: Consumer
while !ready.load(Ordering::Acquire) {}  // Acquire: see all writes before Release
let value = data.load(Ordering::Relaxed);  // Guaranteed to see 42

// SeqCst - strongest, total global ordering
let x = AtomicUsize::new(0);
x.store(1, Ordering::SeqCst);  // All threads agree on order

// AcqRel - both Acquire and Release (for read-modify-write)
let val = counter.fetch_add(1, Ordering::AcqRel);
```

**Ordering strength: Relaxed < Acquire/Release < AcqRel < SeqCst**

---

### 26. What is lock-free programming and how do you implement it in Rust?

**Answer:** Lock-free programming uses atomic operations instead of locks to achieve thread safety.

```rust
use std::sync::atomic::{AtomicPtr, AtomicUsize, Ordering};
use std::ptr;

// Lock-free stack (Treiber stack)
struct LockFreeStack<T> {
    head: AtomicPtr<Node<T>>,
}

struct Node<T> {
    data: T,
    next: *mut Node<T>,
}

impl<T> LockFreeStack<T> {
    fn new() -> Self {
        LockFreeStack {
            head: AtomicPtr::new(ptr::null_mut()),
        }
    }

    fn push(&self, data: T) {
        let new_node = Box::into_raw(Box::new(Node {
            data,
            next: ptr::null_mut(),
        }));

        loop {
            let old_head = self.head.load(Ordering::Acquire);
            unsafe { (*new_node).next = old_head; }

            // CAS: Compare-And-Swap
            if self.head.compare_exchange_weak(
                old_head,
                new_node,
                Ordering::Release,
                Ordering::Relaxed,
            ).is_ok() {
                break;
            }
        }
    }

    fn pop(&self) -> Option<T> {
        loop {
            let old_head = self.head.load(Ordering::Acquire);
            if old_head.is_null() {
                return None;
            }

            let next = unsafe { (*old_head).next };

            if self.head.compare_exchange_weak(
                old_head,
                next,
                Ordering::Release,
                Ordering::Relaxed,
            ).is_ok() {
                let data = unsafe { Box::from_raw(old_head).data };
                return Some(data);
            }
        }
    }
}
```

---

### 27. Explain `crossbeam` and when to use it over standard library.

**Answer:** Crossbeam provides advanced concurrency primitives beyond std.

```rust
use crossbeam::channel;
use crossbeam::scope;
use crossbeam::queue::ArrayQueue;

// Scoped threads - borrow local data safely
fn scoped_example() {
    let data = vec![1, 2, 3, 4, 5];

    crossbeam::scope(|s| {
        // Can borrow data without Arc
        s.spawn(|_| {
            println!("Thread 1: {:?}", &data[..2]);
        });
        s.spawn(|_| {
            println!("Thread 2: {:?}", &data[2..]);
        });
    }).unwrap();  // Threads joined here
}

// Better channels (bounded, select)
fn channel_example() {
    let (tx, rx) = channel::bounded::<i32>(10);

    // Select across multiple channels
    let (tx2, rx2) = channel::bounded::<String>(10);

    crossbeam::select! {
        recv(rx) -> msg => println!("Got int: {:?}", msg),
        recv(rx2) -> msg => println!("Got string: {:?}", msg),
        default => println!("No message"),
    }
}

// Lock-free data structures
fn queue_example() {
    let queue = ArrayQueue::new(100);
    queue.push(1).unwrap();
    let val = queue.pop();
}
```

---

### 28. What is `parking_lot` and why use it over `std::sync`?

**Answer:** `parking_lot` provides faster, more feature-rich synchronization primitives.

```rust
use parking_lot::{Mutex, RwLock, ReentrantMutex, FairMutex};

// Standard Mutex - faster than std
let mutex = Mutex::new(0);
{
    let mut guard = mutex.lock();  // No Result, no poisoning
    *guard += 1;
}

// Reentrant Mutex - same thread can lock multiple times
let reentrant = ReentrantMutex::new(0);
{
    let guard1 = reentrant.lock();
    let guard2 = reentrant.lock();  // OK! Same thread
}

// Fair Mutex - prevents starvation
let fair = FairMutex::new(0);

// RwLock with upgradeable reads
let rwlock = RwLock::new(0);
{
    let read_guard = rwlock.upgradable_read();
    // Can upgrade to write without releasing
    let mut write_guard = parking_lot::RwLockUpgradableReadGuard::upgrade(read_guard);
    *write_guard += 1;
}

// Advantages over std:
// - No poisoning (simpler API)
// - Smaller size (1 byte for Mutex)
// - Faster in most benchmarks
// - More features (fair locks, upgradeable reads)
```

---

## Advanced Async

### 29. Explain how Rust's async/await is implemented at a low level.

**Answer:** Async functions are transformed into state machines that implement `Future`.

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// This async function:
async fn example(x: i32) -> i32 {
    let a = async_op_1().await;
    let b = async_op_2(a).await;
    x + a + b
}

// Is roughly transformed to:
enum ExampleStateMachine {
    Start { x: i32 },
    WaitingOp1 { x: i32, fut: Op1Future },
    WaitingOp2 { x: i32, a: i32, fut: Op2Future },
    Done,
}

impl Future for ExampleStateMachine {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<i32> {
        loop {
            match self.get_mut() {
                Self::Start { x } => {
                    let fut = async_op_1();
                    *self = Self::WaitingOp1 { x: *x, fut };
                }
                Self::WaitingOp1 { x, fut } => {
                    match Pin::new(fut).poll(cx) {
                        Poll::Ready(a) => {
                            let fut = async_op_2(a);
                            *self = Self::WaitingOp2 { x: *x, a, fut };
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                Self::WaitingOp2 { x, a, fut } => {
                    match Pin::new(fut).poll(cx) {
                        Poll::Ready(b) => {
                            return Poll::Ready(*x + *a + b);
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                Self::Done => panic!("polled after completion"),
            }
        }
    }
}
```

---

### 30. What is `Pin` and why is it necessary for async?

**Answer:** `Pin` prevents moving data in memory. Async futures may be self-referential (containing pointers to themselves), so they must not be moved after creation.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Self-referential struct
struct SelfRef {
    value: String,
    ptr: *const String,  // Points to value
    _pin: PhantomPinned,
}

impl SelfRef {
    fn new(s: String) -> Pin<Box<Self>> {
        let mut boxed = Box::new(SelfRef {
            value: s,
            ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });

        // Set ptr to point to value
        let ptr = &boxed.value as *const String;
        boxed.ptr = ptr;

        // Pin it - prevents moving
        Box::into_pin(boxed)
    }

    fn value(self: Pin<&Self>) -> &str {
        &self.value
    }

    fn ptr_value(self: Pin<&Self>) -> &str {
        unsafe { &*self.ptr }
    }
}

// Without Pin, moving would invalidate ptr:
// let mut x = SelfRef::new("hello".into());
// let y = x;  // Move! ptr now points to invalid memory
```

---

### 31. Explain `Waker` and how it enables efficient async execution.

**Answer:** `Waker` notifies the executor when a future can make progress, avoiding busy-polling.

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::Duration;

struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

impl TimerFuture {
    fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        let state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = state.lock().unwrap();
            state.completed = true;
            if let Some(waker) = state.waker.take() {
                waker.wake();  // Wake the executor!
            }
        });

        TimerFuture { shared_state }
    }
}

impl Future for TimerFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.shared_state.lock().unwrap();

        if state.completed {
            Poll::Ready(())
        } else {
            // Store waker for later
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

---

### 32. What is the difference between `tokio` and `async-std`?

**Answer:**

| Feature | Tokio | async-std |
|---------|-------|-----------|
| Maturity | Older, more battle-tested | Newer |
| API Style | Custom types | Mirrors std API |
| Work Stealing | Yes (multi-thread) | Yes |
| Ecosystem | Larger | Smaller |
| IO Primitives | Own types | Matches std |

```rust
// Tokio
use tokio::fs::File;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let mut file = File::open("foo.txt").await.unwrap();
    let mut contents = String::new();
    file.read_to_string(&mut contents).await.unwrap();
}

// async-std - mirrors std API
use async_std::fs::File;
use async_std::io::ReadExt;

#[async_std::main]
async fn main() {
    let mut file = File::open("foo.txt").await.unwrap();
    let mut contents = String::new();
    file.read_to_string(&mut contents).await.unwrap();
}
```

---

## FFI & Interoperability

### 33. How do you call C functions from Rust?

**Answer:** Use `extern "C"` blocks and link to C libraries.

```rust
// Declare C functions
extern "C" {
    fn strlen(s: *const std::os::raw::c_char) -> usize;
    fn printf(format: *const std::os::raw::c_char, ...) -> i32;
}

fn main() {
    let s = std::ffi::CString::new("Hello").unwrap();

    unsafe {
        let len = strlen(s.as_ptr());
        println!("Length: {}", len);
    }
}

// Linking to library
#[link(name = "mylib")]
extern "C" {
    fn my_c_function(x: i32) -> i32;
}

// Using libc crate for standard C types
use libc::{c_int, c_char, size_t};

extern "C" {
    fn custom_func(buf: *mut c_char, len: size_t) -> c_int;
}
```

---

### 34. How do you expose Rust functions to C?

**Answer:** Use `#[no_mangle]` and `extern "C"`.

```rust
// Expose Rust function to C
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// For strings
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn rust_greet(name: *const c_char) -> *mut c_char {
    let c_str = unsafe {
        assert!(!name.is_null());
        CStr::from_ptr(name)
    };

    let name_str = c_str.to_str().unwrap_or("stranger");
    let greeting = format!("Hello, {}!", name_str);

    CString::new(greeting).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn rust_free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe { drop(CString::from_raw(s)); }
    }
}
```

**Header file (generated or manual):**
```c
// mylib.h
int32_t rust_add(int32_t a, int32_t b);
char* rust_greet(const char* name);
void rust_free_string(char* s);
```

---

### 35. What is `bindgen` and `cbindgen`?

**Answer:**
- **bindgen**: Generates Rust FFI bindings from C/C++ headers
- **cbindgen**: Generates C/C++ headers from Rust code

```rust
// build.rs using bindgen
fn main() {
    println!("cargo:rustc-link-lib=mylib");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks))
        .generate()
        .expect("Unable to generate bindings");

    bindings
        .write_to_file("src/bindings.rs")
        .expect("Couldn't write bindings!");
}

// cbindgen.toml
// [export]
// include = ["rust_add", "rust_greet"]
//
// Run: cbindgen --config cbindgen.toml --output mylib.h
```

---

## Compiler Internals

### 36. Explain the Rust compilation process.

**Answer:** Rust compilation has several stages:

1. **Lexing/Parsing**: Source → AST
2. **Macro Expansion**: Expand macros
3. **Name Resolution**: Resolve imports and paths
4. **HIR (High-Level IR)**: Type checking, trait resolution
5. **MIR (Mid-Level IR)**: Borrow checking, optimization
6. **LLVM IR**: Machine-independent optimization
7. **Machine Code**: Final binary

```
Source Code
    ↓
  Lexer → Tokens
    ↓
  Parser → AST
    ↓
  Macro Expansion → Expanded AST
    ↓
  Name Resolution → Resolved AST
    ↓
  Lowering → HIR
    ↓
  Type Checking → Typed HIR
    ↓
  Lowering → MIR
    ↓
  Borrow Checking → Verified MIR
    ↓
  Optimization → Optimized MIR
    ↓
  Code Generation → LLVM IR
    ↓
  LLVM Passes → Optimized LLVM IR
    ↓
  Machine Code → Binary
```

---

### 37. What is MIR and why is it important?

**Answer:** MIR (Mid-level Intermediate Representation) is a simplified representation used for:
- Borrow checking
- Optimization
- Code generation

```rust
// Source code
fn example(x: i32) -> i32 {
    let y = x + 1;
    if y > 0 { y } else { 0 }
}

// Simplified MIR representation
// bb0: {
//     _2 = _1 + const 1_i32;
//     switchInt(_2 > const 0_i32) -> [true: bb1, false: bb2]
// }
// bb1: {
//     _0 = _2;
//     return;
// }
// bb2: {
//     _0 = const 0_i32;
//     return;
// }

// View MIR with: cargo rustc -- -Z dump-mir=all
```

---

### 38. What are the different codegen units strategies?

**Answer:** Codegen units control parallel compilation and optimization.

```toml
# Cargo.toml
[profile.release]
codegen-units = 1        # Fewer units = better optimization, slower compile
lto = "fat"              # Link-time optimization
opt-level = 3            # Maximum optimization

[profile.dev]
codegen-units = 256      # More units = faster compile, less optimization
```

| Setting | Compile Speed | Runtime Speed | Binary Size |
|---------|---------------|---------------|-------------|
| codegen-units = 256 | Fast | Slower | Larger |
| codegen-units = 1 | Slow | Fastest | Smaller |
| LTO enabled | Very slow | Fastest | Smallest |

---

## Performance & Optimization

### 39. How do you profile and optimize Rust code?

**Answer:** Use various tools and techniques:

```rust
// 1. Benchmarking with criterion
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 | 1 => n,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, benchmark);
criterion_main!(benches);

// 2. CPU profiling with perf
// RUSTFLAGS="-C force-frame-pointers=yes" cargo build --release
// perf record ./target/release/myapp
// perf report

// 3. Memory profiling with heaptrack
// heaptrack ./target/release/myapp

// 4. Compiler hints
#[inline(always)]
fn hot_path() {}

#[inline(never)]
fn cold_path() {}

#[cold]
fn unlikely_path() {}

// 5. SIMD optimization
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn simd_add(a: &[f32; 4], b: &[f32; 4]) -> [f32; 4] {
    let va = _mm_loadu_ps(a.as_ptr());
    let vb = _mm_loadu_ps(b.as_ptr());
    let result = _mm_add_ps(va, vb);
    let mut out = [0.0f32; 4];
    _mm_storeu_ps(out.as_mut_ptr(), result);
    out
}
```

---

### 40. What are common performance pitfalls in Rust?

**Answer:**

```rust
// 1. Unnecessary allocations
// Bad
fn bad(v: Vec<i32>) -> Vec<i32> {
    v.iter().map(|x| x * 2).collect()  // New allocation
}
// Good
fn good(v: &mut Vec<i32>) {
    for x in v.iter_mut() { *x *= 2; }  // In-place
}

// 2. Bounds checking in loops
// Bad
fn sum_bad(v: &[i32]) -> i32 {
    let mut sum = 0;
    for i in 0..v.len() {
        sum += v[i];  // Bounds check each iteration
    }
    sum
}
// Good
fn sum_good(v: &[i32]) -> i32 {
    v.iter().sum()  // Iterator, no bounds checks
}

// 3. String concatenation
// Bad
let mut s = String::new();
for i in 0..1000 {
    s = s + &i.to_string();  // Reallocates each time
}
// Good
let mut s = String::with_capacity(4000);
for i in 0..1000 {
    use std::fmt::Write;
    write!(s, "{}", i).unwrap();
}

// 4. HashMap with bad hasher
use std::collections::HashMap;
use rustc_hash::FxHashMap;  // Faster for small keys

// 5. Unnecessary clones
// Bad
fn process(data: &String) {
    let owned = data.clone();  // Unnecessary
}
// Good
fn process(data: &str) {  // Accept slice
    // Work with reference
}

// 6. Blocking in async
// Bad
async fn bad_async() {
    std::thread::sleep(Duration::from_secs(1));  // Blocks executor!
}
// Good
async fn good_async() {
    tokio::time::sleep(Duration::from_secs(1)).await;  // Async sleep
}
```

---

## Design Patterns

### 41. Implement the Type State pattern.

**Answer:** Type state encodes state in the type system, making invalid states unrepresentable.

```rust
// States as zero-sized types
struct Draft;
struct PendingReview;
struct Published;

struct Post<State> {
    content: String,
    _state: std::marker::PhantomData<State>,
}

impl Post<Draft> {
    fn new() -> Self {
        Post {
            content: String::new(),
            _state: std::marker::PhantomData,
        }
    }

    fn add_content(&mut self, text: &str) {
        self.content.push_str(text);
    }

    fn request_review(self) -> Post<PendingReview> {
        Post {
            content: self.content,
            _state: std::marker::PhantomData,
        }
    }
}

impl Post<PendingReview> {
    fn approve(self) -> Post<Published> {
        Post {
            content: self.content,
            _state: std::marker::PhantomData,
        }
    }

    fn reject(self) -> Post<Draft> {
        Post {
            content: self.content,
            _state: std::marker::PhantomData,
        }
    }
}

impl Post<Published> {
    fn content(&self) -> &str {
        &self.content
    }
}

fn main() {
    let mut post = Post::new();
    post.add_content("Hello, world!");

    let post = post.request_review();
    // post.add_content("more");  // Compile error! Can't modify in review

    let post = post.approve();
    println!("{}", post.content());  // Only published posts have content()
}
```

---

### 42. Implement the Newtype pattern with `Deref`.

**Answer:**

```rust
use std::ops::{Deref, DerefMut};

// Newtype wrapper
struct Email(String);

impl Email {
    fn new(email: &str) -> Result<Self, &'static str> {
        if email.contains('@') {
            Ok(Email(email.to_string()))
        } else {
            Err("Invalid email")
        }
    }
}

// Deref for transparent access
impl Deref for Email {
    type Target = String;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

// Prevent arbitrary modification
// impl DerefMut for Email { ... }  // Intentionally not implemented

fn main() {
    let email = Email::new("user@example.com").unwrap();

    // Can use String methods via Deref
    println!("Length: {}", email.len());
    println!("Contains @: {}", email.contains('@'));

    // But can't create invalid email
    // email.0 = "invalid".to_string();  // Private field
}
```

---

### 43. Implement the Extension Trait pattern.

**Answer:**

```rust
// Extend existing types with new methods

trait StringExt {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len.saturating_sub(3)])
        }
    }

    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}

// Also works for external types
trait OptionExt<T> {
    fn unwrap_or_log(self, msg: &str) -> Option<T>;
}

impl<T> OptionExt<T> for Option<T> {
    fn unwrap_or_log(self, msg: &str) -> Option<T> {
        if self.is_none() {
            eprintln!("Warning: {}", msg);
        }
        self
    }
}

fn main() {
    let text = "Hello, World!";
    println!("{}", text.truncate_with_ellipsis(8));  // "Hello..."
    println!("{}", "   ".is_blank());  // true

    let opt: Option<i32> = None;
    opt.unwrap_or_log("Value was missing");
}
```

---

### 44. Implement a compile-time validated builder.

**Answer:**

```rust
use std::marker::PhantomData;

// Marker traits for required fields
trait ToSet {}
trait Set {}

struct Required;
struct Optional;
struct Provided;

impl ToSet for Required {}
impl Set for Provided {}

// Builder with type-level tracking
struct ServerBuilder<Host, Port> {
    host: Option<String>,
    port: Option<u16>,
    max_connections: u32,
    _marker: PhantomData<(Host, Port)>,
}

impl ServerBuilder<Required, Required> {
    fn new() -> Self {
        ServerBuilder {
            host: None,
            port: None,
            max_connections: 100,
            _marker: PhantomData,
        }
    }
}

impl<Port> ServerBuilder<Required, Port> {
    fn host(self, host: &str) -> ServerBuilder<Provided, Port> {
        ServerBuilder {
            host: Some(host.to_string()),
            port: self.port,
            max_connections: self.max_connections,
            _marker: PhantomData,
        }
    }
}

impl<Host> ServerBuilder<Host, Required> {
    fn port(self, port: u16) -> ServerBuilder<Host, Provided> {
        ServerBuilder {
            host: self.host,
            port: Some(port),
            max_connections: self.max_connections,
            _marker: PhantomData,
        }
    }
}

impl<Host, Port> ServerBuilder<Host, Port> {
    fn max_connections(mut self, max: u32) -> Self {
        self.max_connections = max;
        self
    }
}

// build() only available when all required fields are set
impl ServerBuilder<Provided, Provided> {
    fn build(self) -> Server {
        Server {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            max_connections: self.max_connections,
        }
    }
}

struct Server {
    host: String,
    port: u16,
    max_connections: u32,
}

fn main() {
    // Compiles - all required fields set
    let server = ServerBuilder::new()
        .host("localhost")
        .port(8080)
        .max_connections(1000)
        .build();

    // Won't compile - missing port
    // let server = ServerBuilder::new()
    //     .host("localhost")
    //     .build();  // Error: build() not available
}
```

---

## Advanced Coding Challenges

### 45. Implement a thread-safe lazy initialization.

```rust
use std::sync::OnceLock;

// Using OnceLock (std, Rust 1.70+)
static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        println!("Initializing config...");
        Config::load()
    })
}

// Manual implementation
use std::sync::atomic::{AtomicU8, Ordering};
use std::cell::UnsafeCell;

const UNINIT: u8 = 0;
const INITIALIZING: u8 = 1;
const INIT: u8 = 2;

struct Lazy<T, F = fn() -> T> {
    state: AtomicU8,
    value: UnsafeCell<Option<T>>,
    init: F,
}

unsafe impl<T: Send + Sync, F: Send> Sync for Lazy<T, F> {}

impl<T, F: Fn() -> T> Lazy<T, F> {
    const fn new(f: F) -> Self {
        Lazy {
            state: AtomicU8::new(UNINIT),
            value: UnsafeCell::new(None),
            init: f,
        }
    }

    fn get(&self) -> &T {
        if self.state.load(Ordering::Acquire) == INIT {
            return unsafe { (*self.value.get()).as_ref().unwrap() };
        }

        self.init_slow()
    }

    #[cold]
    fn init_slow(&self) -> &T {
        match self.state.compare_exchange(
            UNINIT,
            INITIALIZING,
            Ordering::Acquire,
            Ordering::Relaxed,
        ) {
            Ok(_) => {
                let value = (self.init)();
                unsafe { *self.value.get() = Some(value); }
                self.state.store(INIT, Ordering::Release);
            }
            Err(_) => {
                while self.state.load(Ordering::Acquire) != INIT {
                    std::hint::spin_loop();
                }
            }
        }
        unsafe { (*self.value.get()).as_ref().unwrap() }
    }
}
```

---

### 46. Implement an arena allocator.

```rust
use std::cell::RefCell;
use std::mem::MaybeUninit;

struct Arena<T, const N: usize> {
    storage: RefCell<ArenaInner<T, N>>,
}

struct ArenaInner<T, const N: usize> {
    data: [MaybeUninit<T>; N],
    len: usize,
}

impl<T, const N: usize> Arena<T, N> {
    fn new() -> Self {
        Arena {
            storage: RefCell::new(ArenaInner {
                data: unsafe { MaybeUninit::uninit().assume_init() },
                len: 0,
            }),
        }
    }

    fn alloc(&self, value: T) -> &T {
        let mut inner = self.storage.borrow_mut();

        if inner.len >= N {
            panic!("Arena full");
        }

        let index = inner.len;
        inner.data[index].write(value);
        inner.len += 1;

        // SAFETY: We just wrote to this index
        unsafe { inner.data[index].assume_init_ref() }
    }

    fn clear(&self) {
        let mut inner = self.storage.borrow_mut();

        for i in 0..inner.len {
            unsafe { inner.data[i].assume_init_drop(); }
        }
        inner.len = 0;
    }
}

impl<T, const N: usize> Drop for Arena<T, N> {
    fn drop(&mut self) {
        let inner = self.storage.get_mut();
        for i in 0..inner.len {
            unsafe { inner.data[i].assume_init_drop(); }
        }
    }
}

fn main() {
    let arena: Arena<String, 100> = Arena::new();

    let s1 = arena.alloc("Hello".to_string());
    let s2 = arena.alloc("World".to_string());

    println!("{} {}", s1, s2);
}
```

---

### 47. Implement a simple async executor.

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

struct Executor {
    queue: Arc<Mutex<VecDeque<Task>>>,
}

impl Executor {
    fn new() -> Self {
        Executor {
            queue: Arc::new(Mutex::new(VecDeque::new())),
        }
    }

    fn spawn(&self, future: impl Future<Output = ()> + Send + 'static) {
        self.queue.lock().unwrap().push_back(Box::pin(future));
    }

    fn run(&self) {
        loop {
            let task = self.queue.lock().unwrap().pop_front();

            match task {
                Some(mut task) => {
                    let waker = dummy_waker();
                    let mut cx = Context::from_waker(&waker);

                    if task.as_mut().poll(&mut cx).is_pending() {
                        self.queue.lock().unwrap().push_back(task);
                    }
                }
                None => break,
            }
        }
    }
}

fn dummy_waker() -> Waker {
    fn clone(_: *const ()) -> RawWaker { raw_waker() }
    fn wake(_: *const ()) {}
    fn wake_by_ref(_: *const ()) {}
    fn drop(_: *const ()) {}

    fn raw_waker() -> RawWaker {
        RawWaker::new(
            std::ptr::null(),
            &RawWakerVTable::new(clone, wake, wake_by_ref, drop),
        )
    }

    unsafe { Waker::from_raw(raw_waker()) }
}

fn main() {
    let executor = Executor::new();

    executor.spawn(async {
        println!("Task 1");
    });

    executor.spawn(async {
        println!("Task 2");
    });

    executor.run();
}
```

---

### 48. Implement a lock-free SPSC queue.

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

struct SpscQueue<T, const N: usize> {
    buffer: [UnsafeCell<Option<T>>; N],
    head: AtomicUsize,  // Consumer reads from here
    tail: AtomicUsize,  // Producer writes here
}

unsafe impl<T: Send, const N: usize> Send for SpscQueue<T, N> {}
unsafe impl<T: Send, const N: usize> Sync for SpscQueue<T, N> {}

impl<T, const N: usize> SpscQueue<T, N> {
    fn new() -> Self {
        SpscQueue {
            buffer: std::array::from_fn(|_| UnsafeCell::new(None)),
            head: AtomicUsize::new(0),
            tail: AtomicUsize::new(0),
        }
    }

    fn push(&self, value: T) -> Result<(), T> {
        let tail = self.tail.load(Ordering::Relaxed);
        let next_tail = (tail + 1) % N;

        if next_tail == self.head.load(Ordering::Acquire) {
            return Err(value);  // Queue full
        }

        unsafe {
            *self.buffer[tail].get() = Some(value);
        }

        self.tail.store(next_tail, Ordering::Release);
        Ok(())
    }

    fn pop(&self) -> Option<T> {
        let head = self.head.load(Ordering::Relaxed);

        if head == self.tail.load(Ordering::Acquire) {
            return None;  // Queue empty
        }

        let value = unsafe { (*self.buffer[head].get()).take() };

        self.head.store((head + 1) % N, Ordering::Release);
        value
    }
}

fn main() {
    use std::sync::Arc;
    use std::thread;

    let queue = Arc::new(SpscQueue::<i32, 16>::new());
    let producer = queue.clone();
    let consumer = queue.clone();

    let p = thread::spawn(move || {
        for i in 0..10 {
            while producer.push(i).is_err() {}
        }
    });

    let c = thread::spawn(move || {
        let mut received = Vec::new();
        while received.len() < 10 {
            if let Some(v) = consumer.pop() {
                received.push(v);
            }
        }
        received
    });

    p.join().unwrap();
    let result = c.join().unwrap();
    println!("Received: {:?}", result);
}
```

---

## Quick Reference: Advanced Topics Checklist

- [ ] Unsafe Rust and raw pointers
- [ ] `Send` and `Sync` manual implementation
- [ ] Higher-Ranked Trait Bounds (HRTBs)
- [ ] Lifetime variance (covariant, contravariant, invariant)
- [ ] Self-referential structs and `Pin`
- [ ] Object safety rules
- [ ] GATs (Generic Associated Types)
- [ ] Memory layout and niche optimization
- [ ] Atomic memory ordering
- [ ] Lock-free data structures
- [ ] Async internals (state machines, `Waker`)
- [ ] FFI and `extern "C"`
- [ ] Compiler stages (HIR, MIR, LLVM IR)
- [ ] Type state pattern
- [ ] Extension traits

---

## Tips for Advanced Rust Interviews

1. **Understand the "why"** - Know why Rust made certain design decisions
2. **Read the Rustonomicon** - Essential for understanding unsafe Rust
3. **Study real codebases** - tokio, crossbeam, rayon for advanced patterns
4. **Practice explaining trade-offs** - `Rc` vs `Arc`, `Mutex` vs `RwLock`
5. **Know the standard library** - Deep understanding of `Option`, `Result`, iterators
6. **Understand async deeply** - State machines, `Pin`, `Waker`, executors
7. **Be ready to write unsafe code** - And explain safety invariants

Good luck with your advanced Rust interviews!
