# Advanced Rust Interview Questions (Hindi)

Senior positions aur deep technical discussions ke liye advanced Rust interview questions ka collection.

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

### 1. Unsafe Rust mein paanch cheezein kya kar sakte ho jo safe Rust mein nahi kar sakte?

**Jawab:**
1. Raw pointers dereference karna (`*const T`, `*mut T`)
2. Unsafe functions ya methods call karna
3. Mutable static variables access ya modify karna
4. Unsafe traits implement karna
5. Union fields access karna

```rust
unsafe {
    // 1. Raw pointer dereference
    let ptr: *const i32 = &10;
    println!("{}", *ptr);

    // 2. Unsafe function call
    unsafe fn dangerous() {}
    dangerous();

    // 3. Mutable static access
    static mut COUNTER: i32 = 0;
    COUNTER += 1;

    // 4. Unsafe trait implement (jaise Send, Sync manually)
    // 5. Union fields access
    union MyUnion { i: i32, f: f32 }
    let u = MyUnion { i: 42 };
    println!("{}", u.i);
}
```

**Samjhein:**
- `unsafe` block compiler ko batata hai ki tum responsibility le rahe ho
- Unsafe ka matlab ye nahi ki code galat hai, balki compiler verify nahi kar sakta
- Unsafe code ko safe abstraction mein wrap karna best practice hai

---

### 2. `*const T` aur `*mut T` mein kya antar hai?

**Jawab:**
- `*const T`: Raw pointer reading ke liye (immutable)
- `*mut T`: Raw pointer reading aur writing ke liye (mutable)

Dono safely create ho sakte hain, par dereference karne ke liye `unsafe` chahiye.

```rust
fn main() {
    let mut value = 42;

    // Raw pointers banana safe hai
    let const_ptr: *const i32 = &value;
    let mut_ptr: *mut i32 = &mut value;

    unsafe {
        // Dereference karne ke liye unsafe chahiye
        println!("const: {}", *const_ptr);
        *mut_ptr = 100;
        println!("mut: {}", *mut_ptr);
    }
}
```

**Key Points:**
- Raw pointers borrow checker bypass karte hain
- Null ho sakte hain
- Dangling ho sakte hain
- Aliasing rules follow nahi karte

---

### 3. "Unsafe boundaries" ka concept aur safe abstractions kaise design karein unsafe code ke upar?

**Jawab:** Unsafe boundaries ka matlab hai unsafe code ko safe API ke andar encapsulate karna. Unsafe implementation details hidden rehte hain, aur public interface Rust ki safety guarantees maintain karta hai.

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
            self.grow();  // Internally unsafe
        }
        unsafe {
            // SAFETY: Humne abhi capacity ensure ki hai
            std::ptr::write(self.ptr.add(self.len), value);
        }
        self.len += 1;
    }

    pub fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            unsafe {
                // SAFETY: index bounds-checked hai
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

// Users sirf safe methods se interact karte hain
fn main() {
    let mut v = SafeVec::new();
    v.push(1);
    v.push(2);
    println!("{:?}", v.get(0));  // Safe!
}
```

**Samjhein:**
- Public API safe honi chahiye
- Unsafe code internal implementation mein
- SAFETY comments explain karein ki unsafe kyun correct hai

---

### 4. `MaybeUninit<T>` kya hai aur kab use karein?

**Jawab:** `MaybeUninit<T>` potentially uninitialized memory represent karta hai. Use karo jab values incrementally initialize karni hon ya C ke saath interface karna ho.

```rust
use std::mem::MaybeUninit;

// Array element by element initialize karna
fn create_array() -> [String; 3] {
    let mut arr: [MaybeUninit<String>; 3] = unsafe {
        MaybeUninit::uninit().assume_init()
    };

    for (i, elem) in arr.iter_mut().enumerate() {
        elem.write(format!("Element {}", i));
    }

    // SAFETY: Sab elements ab initialized hain
    unsafe {
        std::mem::transmute::<_, [String; 3]>(arr)
    }
}

// Better approach array::from_fn ke saath (Rust 1.63+)
fn create_array_safe() -> [String; 3] {
    std::array::from_fn(|i| format!("Element {}", i))
}
```

**Key Points:**
- Uninitialized memory read karna UB (undefined behavior) hai
- `MaybeUninit` safe wrapper provide karta hai
- FFI mein commonly use hota hai

---

### 5. `Send` aur `Sync` manually implement karne ke rules kya hain?

**Jawab:**
- `Send`: Ownership threads ke beech safely transfer ho sakti hai
- `Sync`: References threads ke beech safely share ho sakte hain (`T: Sync` iff `&T: Send`)

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

// Custom type manual Send/Sync ke saath
struct MyCell<T> {
    value: UnsafeCell<T>,
    readers: AtomicUsize,
}

// SAFETY: Hum atomic operations ke through thread-safe access guarantee karte hain
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

**Common types jo Send/Sync NAHI hain:**
- `Rc<T>`: Not Send, Not Sync (use `Arc` instead)
- `RefCell<T>`: Send (if T: Send), Not Sync
- `*const T`, `*mut T`: Not Send, Not Sync by default

---

## Advanced Lifetimes

### 6. Higher-Ranked Trait Bounds (HRTBs) `for<'a>` ke saath explain karo.

**Jawab:** HRTBs express karte hain ki ek trait bound *sab* possible lifetimes ke liye hold hona chahiye, sirf specific lifetime ke liye nahi.

```rust
// HRTB ke bina - specific lifetime
fn call_with_ref<'a, F>(f: F)
where
    F: Fn(&'a str),
{
    let s = String::from("hello");
    // f(&s);  // Error: 'a local lifetime se match nahi karta
}

// HRTB ke saath - ANY lifetime ke liye kaam karta hai
fn call_with_ref_hrtb<F>(f: F)
where
    F: for<'a> Fn(&'a str),  // F SAB lifetimes ke liye kaam karta hai
{
    let s = String::from("hello");
    f(&s);  // OK!
}

fn main() {
    call_with_ref_hrtb(|s| println!("{}", s));
}
```

**Real-world example:** `Fn` traits internally HRTBs use karte hain:
```rust
// Ye signature:
fn takes_closure<F: Fn(&str)>(f: F) {}

// Actually ye hai:
fn takes_closure_explicit<F: for<'a> Fn(&'a str)>(f: F) {}
```

**Samjhein:**
- `for<'a>` matlab "har possible lifetime 'a ke liye"
- Closures mein commonly zaroorat hoti hai
- Compiler usually infer kar leta hai

---

### 7. Lifetime variance kya hai aur code ko kaise affect karta hai?

**Jawab:** Variance describe karta hai ki lifetimes kaise substitute ho sakte hain:
- **Covariant**: Longer lifetime shorter ke liye substitute ho sakti hai (`&'a T` 'a mein covariant hai)
- **Contravariant**: Shorter lifetime longer ke liye substitute ho sakti hai (rare, function arguments mein)
- **Invariant**: Koi substitution nahi ho sakti (`&'a mut T` T mein invariant hai)

```rust
// Covariance - 'static wahan use ho sakta hai jahan 'a expected hai
fn covariant<'a>(s: &'a str) -> &'a str { s }
let static_str: &'static str = "hello";
let result: &str = covariant(static_str);  // OK: 'static -> 'a

// Invariance mutable references ke saath
fn invariant<'a>(s: &'a mut Vec<&'a str>) {}
// &mut Vec<&'static str> nahi pass kar sakte jahan &mut Vec<&'a str> expected ho

// Invariance kyun matter karta hai:
fn evil<'a, 'b>(v: &'a mut Vec<&'b str>, s: &'b str) {
    // Agar ye allowed hota:
    // v.push(s);  // Short-lived &'b str push karo
    // v 'b se zyada jee sakta hai, dangling references create hote
}
```

**Samjhein:**
- Covariance common hai references mein
- Invariance mutable references mein safety ke liye
- Compiler automatically handle karta hai

---

### 8. `'a: 'b` aur `'b: 'a` mein kya antar hai?

**Jawab:** `'a: 'b` ka matlab lifetime `'a` outlives (ya equal hai) lifetime `'b`.

```rust
// 'a: 'b matlab 'a kam se kam 'b jitni live karti hai
fn example<'a, 'b>(x: &'a str, y: &'b str) -> &'b str
where
    'a: 'b,  // 'a outlives 'b
{
    x  // OK: x required 'b se zyada live karti hai
}

// Common pattern: shorter lifetime ke saath reference return
struct Context<'a> {
    data: &'a str,
}

impl<'a> Context<'a> {
    fn get_data<'b>(&'b self) -> &'b str
    where
        'a: 'b,  // data self ke borrow se zyada live karti hai
    {
        self.data
    }
}
```

**Samjhein:**
- `'a: 'b` read karo "a outlives b"
- Longer lifetime shorter satisfy kar sakti hai
- Compiler usually infer kar leta hai

---

### 9. `impl` blocks mein lifetime elision rules kya hain?

**Jawab:** `impl` blocks mein, `&self` ya `&mut self` ki lifetime automatically output references ko assign hoti hai.

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
    // Note: &self se tied reference return, 'a se nahi
    fn current(&self) -> &str {
        &self.input[..1]
    }

    // Explicit jab data ki lifetime return karni ho, self ki nahi
    fn remaining(&self) -> &'a str {
        self.input
    }
}
```

**Samjhein:**
- Methods mein `&self` ki lifetime usually output ko milti hai
- Kabhi kabhi explicit annotations zaroor hain
- Struct lifetime aur method lifetime different ho sakte hain

---

### 10. Self-referential structs Rust mein kaise handle karein?

**Jawab:** Self-referential structs challenging hain kyunki moves internal pointers invalid kar dete hain. Solutions:

```rust
// Option 1: References ki jagah indices use karo
struct SelfRefWithIndex {
    data: Vec<String>,
    current_index: usize,
}

// Option 2: Pin + unsafe use karo
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

// Option 3: ouroboros ya self_cell crate use karo
// Ye safe abstractions provide karte hain self-referential structs ke liye
```

**Samjhein:**
- Self-referential structs Rust mein naturally difficult hain
- Pin prevent karta hai move hone se
- External crates safe solutions dete hain

---

## Advanced Traits

### 11. Trait objects ke liye object safety rules explain karo.

**Jawab:** Ek trait object-safe hai agar wo `dyn Trait` ki tarah use ho sake. Rules:

1. Methods `Self` return nahi kar sakte
2. Methods mein generic type parameters nahi ho sakte
3. Trait ko `Self: Sized` require nahi karna chahiye

```rust
// Object-safe trait
trait ObjectSafe {
    fn method(&self);
    fn method_with_default(&self) {}
}

// Object-safe NAHI
trait NotObjectSafe {
    fn returns_self(&self) -> Self;           // Self return karta hai
    fn generic<T>(&self, t: T);               // Generic parameter
    fn sized_only(&self) where Self: Sized;   // Sized bound (par exclude ho sakta hai)
}

// Trait ko partially object-safe banana
trait PartiallyObjectSafe {
    fn object_safe_method(&self);

    // Sized bound se trait object se exclude karo
    fn not_object_safe(&self) -> Self where Self: Sized;
}

// Ab dyn PartiallyObjectSafe kaam karta hai, par not_object_safe() call nahi kar sakte
fn use_trait_object(obj: &dyn PartiallyObjectSafe) {
    obj.object_safe_method();  // OK
    // obj.not_object_safe();  // dyn pe available nahi
}
```

**Samjhein:**
- `Self` return karna object safety break karta hai
- Generics bhi break karte hain
- `where Self: Sized` se methods exclude ho sakte hain

---

### 12. Blanket implementations kya hain?

**Jawab:** Blanket implementations ek trait ko sab types ke liye implement karte hain jo certain bounds satisfy karein.

```rust
// Standard library example: impl<T: Display> ToString for T
use std::fmt::Display;

trait Printable {
    fn print(&self);
}

// Sab Display types ke liye blanket implementation
impl<T: Display> Printable for T {
    fn print(&self) {
        println!("{}", self);
    }
}

fn main() {
    42.print();           // i32 ke liye kaam karta hai
    "hello".print();      // &str ke liye kaam karta hai
    3.14f64.print();      // f64 ke liye kaam karta hai
}

// Dusra example: From/Into
// impl<T, U> Into<U> for T where U: From<T>
// Agar tum From implement karo, Into free mein milta hai!
```

**Samjhein:**
- Blanket implementations powerful hain
- Standard library mein common hain
- Automatic trait implementations provide karte hain

---

### 13. `Deref` aur `DerefMut` traits aur deref coercion explain karo.

**Jawab:** `Deref` dereference operator `*` customize karne deta hai. Deref coercion automatically references convert karta hai.

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
    // &MyBox<String> -> &String (Deref se)
    // &String -> &str (String ki Deref se)
}
```

**Deref coercion rules:**
- `&T` to `&U` jab `T: Deref<Target=U>`
- `&mut T` to `&mut U` jab `T: DerefMut<Target=U>`
- `&mut T` to `&U` jab `T: Deref<Target=U>` (mutable to immutable)

---

### 14. `PhantomData` marker kya hai aur kab use hota hai?

**Jawab:** `PhantomData<T>` zero-sized type hai jo compiler ko batata hai ki act karo jaise T own kar rahe ho, variance aur drop checking affect karta hai.

```rust
use std::marker::PhantomData;

// Use case 1: Data store kiye bina ownership indicate karna
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<&'a T>,  // Lifetime relationship indicate karta hai
}

// Use case 2: Unused type parameter
struct TypedId<T> {
    id: u64,
    _marker: PhantomData<T>,  // T stored nahi par type ka part hai
}

type UserId = TypedId<User>;
type PostId = TypedId<Post>;

// Use case 3: Drop check ke liye ownership indicate karna
struct OwnedPtr<T> {
    ptr: *mut T,
    _owns: PhantomData<T>,  // Compiler jaanta hai hum T own karte hain drop ke liye
}

impl<T> Drop for OwnedPtr<T> {
    fn drop(&mut self) {
        unsafe {
            std::ptr::drop_in_place(self.ptr);
        }
    }
}
```

**Samjhein:**
- Runtime pe zero cost
- Compiler ko type information deta hai
- Variance aur drop checking ke liye zaroori

---

### 15. `AsRef`, `Borrow`, aur `Deref` mein antar explain karo.

**Jawab:**

| Trait | Purpose | Use Case |
|-------|---------|----------|
| `Deref` | Smart pointer dereference | Automatic coercion, `*x` |
| `AsRef` | Cheap reference conversion | Generic functions jo different types accept karein |
| `Borrow` | Hash/eq consistency ke saath borrowing | HashMap/HashSet keys |

```rust
use std::borrow::Borrow;

// AsRef - cheap conversion, koi semantic guarantee nahi
fn print_path<P: AsRef<std::path::Path>>(path: P) {
    println!("{:?}", path.as_ref());
}
print_path("file.txt");           // &str -> &Path
print_path(String::from("x"));    // String -> &Path

// Borrow - hash/eq consistency guarantee karta hai
fn get_from_map<Q: ?Sized>(map: &HashMap<String, i32>, key: &Q) -> Option<&i32>
where
    String: Borrow<Q>,
    Q: Hash + Eq,
{
    map.get(key)
}
// &str se lookup kar sakte hain even though key String hai
let value = get_from_map(&map, "key");

// Deref - automatic coercion chain
let s = String::from("hello");
let r: &str = &s;  // String derefs to str
```

**Samjhein:**
- `Deref` automatic coercion ke liye
- `AsRef` flexible function parameters ke liye
- `Borrow` collections mein lookup ke liye

---

## Type System Deep Dive

### 16. Rust ke type coercion rules explain karo.

**Jawab:** Rust limited cases mein implicit coercions perform karta hai:

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

// 5. Closure to fn pointer (agar koi captures nahi)
let closure = |x: i32| x + 1;
let fn_ptr: fn(i32) -> i32 = closure;

// 6. Never type coercion
fn diverges() -> ! { panic!() }
let x: i32 = if true { 5 } else { diverges() };  // ! kisi bhi type mein coerce
```

**Samjhein:**
- Coercions implicit hoti hain
- Limited cases mein hi hoti hain
- Explicit conversions ke liye `as`, `From`/`Into` use karo

---

### 17. `!` (never) type kya hai?

**Jawab:** Never type `!` un computations ko represent karta hai jo kabhi complete nahi hote. Ye kisi bhi type mein coerce ho sakta hai.

```rust
// Functions jo kabhi return nahi karte
fn infinite_loop() -> ! {
    loop {}
}

fn always_panics() -> ! {
    panic!("boom");
}

// Match arms mein use
fn unwrap_or_die<T>(opt: Option<T>) -> T {
    match opt {
        Some(v) => v,
        None => panic!("was None"),  // panic! returns !
    }
}

// Enums mein useful
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// Infallible conversion
type Infallible = !;  // std::convert::Infallible
impl<T> From<!> for T {
    fn from(never: !) -> T {
        match never {}  // Never type pe empty match
    }
}
```

**Samjhein:**
- `!` diverging functions ke liye
- Kisi bhi type mein coerce ho sakta hai
- `match` mein useful hai

---

### 18. `Sized` aur `?Sized` mein antar explain karo.

**Jawab:**
- `Sized`: Type ki compile time pe known size hai (default bound)
- `?Sized`: Type sized ho bhi sakti hai nahi bhi (DST - dynamically sized type)

```rust
// Implicitly Sized
fn takes_sized<T>(t: T) {}  // T: Sized implied

// Explicitly ?Sized DSTs accept karne ke liye
fn takes_unsized<T: ?Sized>(t: &T) {}  // &str, &[i32], &dyn Trait le sakta hai

// DST examples
let s: &str = "hello";         // str unsized hai
let slice: &[i32] = &[1, 2];   // [i32] unsized hai
let obj: &dyn Display = &42;    // dyn Display unsized hai

// DST field ke saath struct (last field hona chahiye)
struct MySlice<T: ?Sized> {
    len: usize,
    data: T,  // Last field hona chahiye
}

// Unsized struct banana
let boxed: Box<MySlice<[i32]>> = todo!();
```

**Samjhein:**
- Most types `Sized` hain
- `str`, `[T]`, `dyn Trait` unsized hain
- DSTs sirf references/pointers ke through accessible

---

### 19. Const generics kya hain aur kaise use hote hain?

**Jawab:** Const generics types ko constant values se parameterize karne dete hain, sirf types se nahi.

```rust
// Generic size ke saath array
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

// Array size pe generic function
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

**Samjhein:**
- Compile-time constants types mein
- Arrays ke liye especially useful
- Zero runtime overhead

---

### 20. GATs (Generic Associated Types) explain karo.

**Jawab:** GATs associated types ko apne generic parameters rakhne dete hain.

```rust
// GATs ke bina - lending iterator express nahi kar sakte
trait LendingIterator {
    type Item<'a> where Self: 'a;  // Lifetime parameter ke saath GAT

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

// Type parameter ke saath GAT
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

**Samjhein:**
- GATs flexible associated types dete hain
- Lending iterators enable karte hain
- Rust 1.65+ mein stable

---

## Memory Layout & Optimization

### 21. Struct memory layout aur padding explain karo.

**Jawab:** Rust structs mein alignment ke liye padding hoti hai. Control ke liye `#[repr(C)]` ya `#[repr(packed)]` use karo.

```rust
use std::mem::{size_of, align_of};

// Default layout - compiler fields reorder kar sakta hai
struct Default {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes
    c: u8,   // 1 byte
}
// Size: 24 bytes (padding ke saath alignment ke liye)

// C-compatible layout
#[repr(C)]
struct CLayout {
    a: u8,   // 1 byte + 7 padding
    b: u64,  // 8 bytes
    c: u8,   // 1 byte + 7 padding
}
// Size: 24 bytes (fields declared order mein)

// Packed - koi padding nahi
#[repr(packed)]
struct Packed {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes
    c: u8,   // 1 byte
}
// Size: 10 bytes (koi padding nahi, unaligned access)

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

**Samjhein:**
- Default layout mein compiler optimize kar sakta hai
- `repr(C)` FFI ke liye zaroor hai
- `repr(packed)` size kam karta hai par slow ho sakta hai

---

### 22. Niche optimization kya hai?

**Jawab:** Rust "niches" (invalid bit patterns) types mein use karta hai enum discriminants store karne ke liye bina extra space ke.

```rust
use std::mem::size_of;

// Option<&T> null pointer niche use karta hai
// &T kabhi null nahi ho sakta, to None 0x0 use karta hai
assert_eq!(size_of::<&i32>(), size_of::<Option<&i32>>());

// NonZero types mein niches hain
use std::num::NonZeroU64;
assert_eq!(size_of::<NonZeroU64>(), size_of::<Option<NonZeroU64>>());

// bool sirf 0 aur 1 use karta hai, 2-255 niches hain
assert_eq!(size_of::<bool>(), size_of::<Option<bool>>());

// Custom types niches ke saath
#[repr(u8)]
enum Status {
    Active = 0,
    Inactive = 1,
    // Values 2-255 niches hain
}
assert_eq!(size_of::<Status>(), size_of::<Option<Status>>());

// Koi niche nahi - extra byte chahiye
struct NoNiche {
    value: u8,
}
assert_eq!(size_of::<NoNiche>(), 1);
assert_eq!(size_of::<Option<NoNiche>>(), 2);  // Discriminant chahiye
```

**Samjhein:**
- Option<&T> free hai space mein
- NonZero types same benefit dete hain
- Compiler automatically optimize karta hai

---

### 23. `Box<[T]>` aur `Vec<T>` mein antar explain karo.

**Jawab:**

| Feature | `Box<[T]>` | `Vec<T>` |
|---------|------------|----------|
| Size | 2 words (ptr + len) | 3 words (ptr + len + capacity) |
| Growable | Nahi | Haan |
| Shrink | Fixed | Shrink ho sakta hai |
| Conversion | Vec se | Box<[T]> mein convert ho sakta hai |

```rust
use std::mem::size_of;

fn main() {
    // Vec mein capacity hai
    let vec: Vec<i32> = vec![1, 2, 3];
    println!("Vec size: {} bytes", size_of::<Vec<i32>>());  // 24 bytes

    // Box<[T]> mein capacity nahi
    let boxed: Box<[i32]> = vec.into_boxed_slice();
    println!("Box<[i32]> size: {} bytes", size_of::<Box<[i32]>>());  // 16 bytes

    // Wapas convert (reallocate ho sakta hai)
    let vec_again: Vec<i32> = boxed.into_vec();
}
```

**Samjhein:**
- `Box<[T]>` fixed-size slice ke liye
- Memory footprint kam hai `Vec` se
- Grow nahi kar sakta

---

### 24. Zero-cost abstraction kya hai aur Rust kaise achieve karta hai?

**Jawab:** Zero-cost abstractions ka runtime overhead nahi hota hand-written low-level code se compare mein. Rust achieve karta hai:

1. **Monomorphization**: Generics specific types ke liye compile hote hain
2. **Inlining**: Choti functions inline ho jati hain
3. **No runtime overhead**: Iterators loops mein compile hote hain

```rust
// Iterator chain simple loop mein compile hoti hai
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * 2)
    .sum();

// Ye roughly compile hota hai:
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

// Specific versions generate hote hain:
// add::<i32>(i32, i32) -> i32
// add::<f64>(f64, f64) -> f64
// Koi vtable nahi, koi dynamic dispatch nahi
```

**Samjhein:**
- High-level code low-level performance deta hai
- Compiler optimizations strong hain
- Ye Rust ka core principle hai

---

## Advanced Concurrency

### 25. Atomics mein memory ordering explain karo.

**Jawab:** Memory ordering control karti hai ki atomic operations threads ke beech kaise synchronize hote hain.

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};

// Relaxed - koi synchronization nahi, sirf atomic
let counter = AtomicUsize::new(0);
counter.fetch_add(1, Ordering::Relaxed);

// Acquire/Release - specific data synchronize
let data = AtomicUsize::new(0);
let ready = AtomicBool::new(false);

// Thread 1: Producer
data.store(42, Ordering::Relaxed);
ready.store(true, Ordering::Release);  // Release: pehle ke sab writes visible

// Thread 2: Consumer
while !ready.load(Ordering::Acquire) {}  // Acquire: Release ke pehle ke sab writes dekhe
let value = data.load(Ordering::Relaxed);  // 42 dekhna guaranteed

// SeqCst - sabse strong, total global ordering
let x = AtomicUsize::new(0);
x.store(1, Ordering::SeqCst);  // Sab threads order pe agree

// AcqRel - Acquire aur Release dono (read-modify-write ke liye)
let val = counter.fetch_add(1, Ordering::AcqRel);
```

**Ordering strength: Relaxed < Acquire/Release < AcqRel < SeqCst**

**Samjhein:**
- Relaxed sabse fast par koi guarantees nahi
- SeqCst sabse safe par slowest
- Acquire/Release common use case hai

---

### 26. Lock-free programming kya hai aur Rust mein kaise implement karein?

**Jawab:** Lock-free programming locks ki jagah atomic operations use karti hai thread safety ke liye.

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

**Samjhein:**
- Lock-free contention mein better perform karta hai
- CAS (Compare-And-Swap) core operation hai
- Complex implement karna hai

---

### 27. `crossbeam` explain karo aur standard library se kab better hai?

**Jawab:** Crossbeam advanced concurrency primitives provide karta hai std se zyada.

```rust
use crossbeam::channel;
use crossbeam::scope;
use crossbeam::queue::ArrayQueue;

// Scoped threads - local data safely borrow karo
fn scoped_example() {
    let data = vec![1, 2, 3, 4, 5];

    crossbeam::scope(|s| {
        // Arc ke bina data borrow kar sakte ho
        s.spawn(|_| {
            println!("Thread 1: {:?}", &data[..2]);
        });
        s.spawn(|_| {
            println!("Thread 2: {:?}", &data[2..]);
        });
    }).unwrap();  // Threads yahan join hote hain
}

// Better channels (bounded, select)
fn channel_example() {
    let (tx, rx) = channel::bounded::<i32>(10);

    // Multiple channels pe select
    let (tx2, rx2) = channel::bounded::<String>(10);

    crossbeam::select! {
        recv(rx) -> msg => println!("Int mila: {:?}", msg),
        recv(rx2) -> msg => println!("String mila: {:?}", msg),
        default => println!("Koi message nahi"),
    }
}

// Lock-free data structures
fn queue_example() {
    let queue = ArrayQueue::new(100);
    queue.push(1).unwrap();
    let val = queue.pop();
}
```

**Samjhein:**
- Scoped threads borrowing allow karte hain
- Select multiple channels pe
- Lock-free data structures built-in

---

### 28. `parking_lot` kya hai aur `std::sync` se kyun use karein?

**Jawab:** `parking_lot` faster, zyada feature-rich synchronization primitives provide karta hai.

```rust
use parking_lot::{Mutex, RwLock, ReentrantMutex, FairMutex};

// Standard Mutex - std se faster
let mutex = Mutex::new(0);
{
    let mut guard = mutex.lock();  // Koi Result nahi, koi poisoning nahi
    *guard += 1;
}

// Reentrant Mutex - same thread multiple times lock kar sakta hai
let reentrant = ReentrantMutex::new(0);
{
    let guard1 = reentrant.lock();
    let guard2 = reentrant.lock();  // OK! Same thread
}

// Fair Mutex - starvation prevent karta hai
let fair = FairMutex::new(0);

// RwLock upgradeable reads ke saath
let rwlock = RwLock::new(0);
{
    let read_guard = rwlock.upgradable_read();
    // Bina release kiye write mein upgrade
    let mut write_guard = parking_lot::RwLockUpgradableReadGuard::upgrade(read_guard);
    *write_guard += 1;
}

// std se advantages:
// - Koi poisoning nahi (simpler API)
// - Smaller size (Mutex ke liye 1 byte)
// - Most benchmarks mein faster
// - Zyada features (fair locks, upgradeable reads)
```

---

## Advanced Async

### 29. Rust ka async/await low level pe kaise implement hai explain karo.

**Jawab:** Async functions state machines mein transform hote hain jo `Future` implement karte hain.

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// Ye async function:
async fn example(x: i32) -> i32 {
    let a = async_op_1().await;
    let b = async_op_2(a).await;
    x + a + b
}

// Roughly ye transform hota hai:
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
                Self::Done => panic!("completion ke baad poll kiya"),
            }
        }
    }
}
```

**Samjhein:**
- Async fn Future return karta hai
- Future ek state machine hai
- Each await point ek state hai

---

### 30. `Pin` kya hai aur async ke liye kyun zaroor hai?

**Jawab:** `Pin` data ko memory mein move hone se rokta hai. Self-referential async futures ke liye zaroor hai.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Self-referential struct
struct SelfRef {
    value: String,
    ptr: *const String,  // value ko point karta hai
    _pin: PhantomPinned,
}

impl SelfRef {
    fn new(s: String) -> Pin<Box<Self>> {
        let mut boxed = Box::new(SelfRef {
            value: s,
            ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });

        // ptr ko value point karao
        let ptr = &boxed.value as *const String;
        boxed.ptr = ptr;

        // Pin karo - move prevent hoga
        Box::into_pin(boxed)
    }

    fn value(self: Pin<&Self>) -> &str {
        &self.value
    }

    fn ptr_value(self: Pin<&Self>) -> &String {
        unsafe { &*self.ptr }
    }
}

// Pin ke bina, move ptr invalidate kar deta:
// let mut x = SelfRef::new("hello".into());
// let y = x;  // Move! ptr ab invalid memory point karta hai
```

**Samjhein:**
- Self-referential structs mein internal pointers hote hain
- Move se pointers invalid ho jate hain
- Pin guarantee deta hai ki move nahi hoga

---

### 31. `Waker` explain karo aur efficient async execution kaise enable karta hai.

**Jawab:** `Waker` executor ko notify karta hai jab future progress kar sakta hai, busy-polling avoid karta hai.

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
                waker.wake();  // Executor ko jaagao!
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
            // Waker baad ke liye store karo
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

**Samjhein:**
- Waker callback mechanism hai
- Future complete hone pe executor ko notify karta hai
- Efficient scheduling enable karta hai

---

### 32. `tokio` aur `async-std` mein kya antar hai?

**Jawab:**

| Feature | Tokio | async-std |
|---------|-------|-----------|
| Maturity | Older, zyada battle-tested | Newer |
| API Style | Custom types | std API mirror karta hai |
| Work Stealing | Haan (multi-thread) | Haan |
| Ecosystem | Larger | Smaller |
| IO Primitives | Own types | std se match |

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

// async-std - std API mirror karta hai
use async_std::fs::File;
use async_std::io::ReadExt;

#[async_std::main]
async fn main() {
    let mut file = File::open("foo.txt").await.unwrap();
    let mut contents = String::new();
    file.read_to_string(&mut contents).await.unwrap();
}
```

**Samjhein:**
- Tokio zyada mature aur widely used hai
- async-std std se familiar feel deta hai
- Dono capable runtimes hain

---

## FFI & Interoperability

### 33. Rust se C functions kaise call karein?

**Jawab:** `extern "C"` blocks use karo aur C libraries se link karo.

```rust
// C functions declare karo
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

// Library se link karna
#[link(name = "mylib")]
extern "C" {
    fn my_c_function(x: i32) -> i32;
}

// libc crate standard C types ke liye
use libc::{c_int, c_char, size_t};

extern "C" {
    fn custom_func(buf: *mut c_char, len: size_t) -> c_int;
}
```

**Samjhein:**
- `extern "C"` C ABI use karta hai
- `CString` null-terminated string banata hai
- Unsafe block zaroor hai C functions call karne ke liye

---

### 34. Rust functions ko C mein kaise expose karein?

**Jawab:** `#[no_mangle]` aur `extern "C"` use karo.

```rust
// Rust function C ko expose karo
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// Strings ke liye
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

**Header file (generated ya manual):**
```c
// mylib.h
int32_t rust_add(int32_t a, int32_t b);
char* rust_greet(const char* name);
void rust_free_string(char* s);
```

---

### 35. `bindgen` aur `cbindgen` kya hain?

**Jawab:**
- **bindgen**: C/C++ headers se Rust FFI bindings generate karta hai
- **cbindgen**: Rust code se C/C++ headers generate karta hai

```rust
// build.rs bindgen use karke
fn main() {
    println!("cargo:rustc-link-lib=mylib");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks))
        .generate()
        .expect("Bindings generate nahi ho sake");

    bindings
        .write_to_file("src/bindings.rs")
        .expect("Bindings likh nahi sake!");
}

// cbindgen.toml
// [export]
// include = ["rust_add", "rust_greet"]
//
// Run: cbindgen --config cbindgen.toml --output mylib.h
```

**Samjhein:**
- bindgen C libraries wrap karne ke liye
- cbindgen Rust libraries C ko expose karne ke liye
- Dono automation tools hain

---

## Compiler Internals

### 36. Rust compilation process explain karo.

**Jawab:** Rust compilation ke kई stages hain:

1. **Lexing/Parsing**: Source → AST
2. **Macro Expansion**: Macros expand karo
3. **Name Resolution**: Imports aur paths resolve karo
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

### 37. MIR kya hai aur kyun important hai?

**Jawab:** MIR (Mid-level Intermediate Representation) ek simplified representation hai jo use hoti hai:
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

// MIR dekho: cargo rustc -- -Z dump-mir=all
```

**Samjhein:**
- MIR simple hai HIR se
- Borrow checker MIR pe kaam karta hai
- Optimizations MIR level pe hoti hain

---

## Performance & Optimization

### 38. Rust code profile aur optimize kaise karein?

**Jawab:** Various tools aur techniques use karo:

```rust
// 1. Criterion se benchmarking
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

// 2. perf se CPU profiling
// RUSTFLAGS="-C force-frame-pointers=yes" cargo build --release
// perf record ./target/release/myapp
// perf report

// 3. heaptrack se Memory profiling
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

### 39. Rust mein common performance pitfalls kya hain?

**Jawab:**

```rust
// 1. Unnecessary allocations
// Galat
fn bad(v: Vec<i32>) -> Vec<i32> {
    v.iter().map(|x| x * 2).collect()  // Nayi allocation
}
// Sahi
fn good(v: &mut Vec<i32>) {
    for x in v.iter_mut() { *x *= 2; }  // In-place
}

// 2. Loops mein bounds checking
// Galat
fn sum_bad(v: &[i32]) -> i32 {
    let mut sum = 0;
    for i in 0..v.len() {
        sum += v[i];  // Har iteration pe bounds check
    }
    sum
}
// Sahi
fn sum_good(v: &[i32]) -> i32 {
    v.iter().sum()  // Iterator, koi bounds checks nahi
}

// 3. String concatenation
// Galat
let mut s = String::new();
for i in 0..1000 {
    s = s + &i.to_string();  // Har baar reallocate
}
// Sahi
let mut s = String::with_capacity(4000);
for i in 0..1000 {
    use std::fmt::Write;
    write!(s, "{}", i).unwrap();
}

// 4. Galat hasher ke saath HashMap
use std::collections::HashMap;
use rustc_hash::FxHashMap;  // Choti keys ke liye faster

// 5. Unnecessary clones
// Galat
fn process(data: &String) {
    let owned = data.clone();  // Unnecessary
}
// Sahi
fn process(data: &str) {  // Slice accept karo
    // Reference ke saath kaam karo
}

// 6. Async mein blocking
// Galat
async fn bad_async() {
    std::thread::sleep(Duration::from_secs(1));  // Executor block!
}
// Sahi
async fn good_async() {
    tokio::time::sleep(Duration::from_secs(1)).await;  // Async sleep
}
```

---

## Design Patterns

### 40. Type State pattern implement karo.

**Jawab:** Type state states ko type system mein encode karta hai, invalid states unrepresentable banata hai.

```rust
// States zero-sized types ki tarah
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
    // post.add_content("more");  // Compile error! Review mein modify nahi kar sakte

    let post = post.approve();
    println!("{}", post.content());  // Sirf published posts ka content()
}
```

**Samjhein:**
- States compile-time pe enforce hote hain
- Invalid transitions impossible hain
- Zero runtime cost

---

### 41. `Deref` ke saath Newtype pattern implement karo.

**Jawab:**

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

// Transparent access ke liye Deref
impl Deref for Email {
    type Target = String;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

// Arbitrary modification prevent karo
// impl DerefMut for Email { ... }  // Intentionally implement nahi kiya

fn main() {
    let email = Email::new("user@example.com").unwrap();

    // Deref se String methods use kar sakte hain
    println!("Length: {}", email.len());
    println!("@ hai: {}", email.contains('@'));

    // Par invalid email nahi bana sakte
    // email.0 = "invalid".to_string();  // Private field
}
```

**Samjhein:**
- Newtype type safety add karta hai
- Deref convenient access deta hai
- Internal invariants protect hote hain

---

### 42. Extension Trait pattern implement karo.

**Jawab:**

```rust
// Existing types ko naye methods se extend karo

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

// External types ke liye bhi kaam karta hai
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
    opt.unwrap_or_log("Value nahi thi");
}
```

---

## Advanced Coding Challenges

### 43. Thread-safe lazy initialization implement karo.

```rust
use std::sync::OnceLock;

// OnceLock use karke (std, Rust 1.70+)
static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        println!("Config initialize ho rahi hai...");
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

### 44. Simple async executor implement karo.

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

## Quick Reference: Advanced Topics Checklist

- [ ] Unsafe Rust aur raw pointers
- [ ] `Send` aur `Sync` manual implementation
- [ ] Higher-Ranked Trait Bounds (HRTBs)
- [ ] Lifetime variance (covariant, contravariant, invariant)
- [ ] Self-referential structs aur `Pin`
- [ ] Object safety rules
- [ ] GATs (Generic Associated Types)
- [ ] Memory layout aur niche optimization
- [ ] Atomic memory ordering
- [ ] Lock-free data structures
- [ ] Async internals (state machines, `Waker`)
- [ ] FFI aur `extern "C"`
- [ ] Compiler stages (HIR, MIR, LLVM IR)
- [ ] Type state pattern
- [ ] Extension traits

---

## Advanced Rust Interviews ke liye Tips

1. **"Kyun" samjho** - Rust ne kisi design decision kyun liya ye jaano
2. **Rustonomicon padho** - Unsafe Rust samajhne ke liye essential hai
3. **Real codebases study karo** - tokio, crossbeam, rayon advanced patterns ke liye
4. **Trade-offs explain karna aaye** - `Rc` vs `Arc`, `Mutex` vs `RwLock`
5. **Standard library deeply jaano** - `Option`, `Result`, iterators
6. **Async deeply samjho** - State machines, `Pin`, `Waker`, executors
7. **Unsafe code likhne ki practice karo** - Aur safety invariants explain karo

Advanced Rust interviews ke liye all the best!
