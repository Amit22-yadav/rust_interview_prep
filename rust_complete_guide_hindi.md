# Rust Complete Learning Guide - Basic se Advanced (Hindi)

Rust programming language seekhne ke liye ek complete guide jo basic se advanced topics tak cover karti hai, clear explanations aur examples ke saath.

---

## Table of Contents

### Part 1: Basics (शुरुआत)
1. [Rust kya hai?](#rust-kya-hai)
2. [Variables aur Mutability](#variables-aur-mutability)
3. [Data Types](#data-types)
4. [Functions](#functions)
5. [Control Flow](#control-flow)

### Part 2: Core Concepts (मुख्य अवधारणाएं)
6. [Ownership](#ownership)
7. [Borrowing aur References](#borrowing-aur-references)
8. [Slices](#slices)
9. [Structs](#structs)
10. [Enums aur Pattern Matching](#enums-aur-pattern-matching)
11. [Modules](#modules)

### Part 3: Intermediate (मध्यवर्ती)
12. [Error Handling](#error-handling)
13. [Generics](#generics)
14. [Traits](#traits)
15. [Lifetimes](#lifetimes)
16. [Closures](#closures)
17. [Iterators](#iterators)
18. [Collections](#collections)

### Part 4: Advanced (उन्नत)
19. [Smart Pointers](#smart-pointers)
20. [Concurrency aur Threads](#concurrency-aur-threads)
21. [Async/Await](#asyncawait)
22. [Unsafe Rust](#unsafe-rust)
23. [Macros](#macros)

---

# Part 1: Basics (शुरुआत)

## Rust kya hai?

### Rust Kya Hai?
Rust ek **systems programming language** hai jo teen cheezon par focus karti hai:
- **Safety (सुरक्षा)**: Garbage collector ke bina memory safety
- **Speed (गति)**: Zero-cost abstractions aur efficient code
- **Concurrency (समवर्तिता)**: Bina dar ke concurrent programming

### Rust Kyun Seekhein?
1. **Memory Safety**: No null pointers, no dangling references, no buffer overflows
2. **Performance**: C/C++ jitni fast with high-level abstractions
3. **Modern Tooling**: Cargo (package manager), rustfmt (formatter), clippy (linter)
4. **Growing Ecosystem**: Web (Actix, Rocket), Systems (Tokio), WebAssembly
5. **Industry Adoption**: Mozilla, Microsoft, Google, Amazon, Discord use karte hain

### Hello World
```rust
fn main() {
    println!("Hello, World!");
}
```

**Explanation:**
- `fn main()` - Har Rust program ka entry point
- `println!` - Console par print karne ke liye macro (note: `!` lagta hai)
- Statements semicolon `;` ke saath end hoti hain

---

## Variables aur Mutability

### Default mein Immutable
Rust mein variables **by default immutable** hote hain. Matlab ek baar value assign karne ke baad, usse change nahi kar sakte.

```rust
fn main() {
    let x = 5;
    // x = 6;  // Error! Immutable variable ko dubara assign nahi kar sakte
    println!("x = {}", x);
}
```

### Variable ko Mutable banana
`mut` keyword use karke variable ko mutable bana sakte hain:

```rust
fn main() {
    let mut x = 5;
    println!("x = {}", x);  // x = 5

    x = 6;  // Ab ye allowed hai
    println!("x = {}", x);  // x = 6
}
```

### Constants (स्थिरांक)
Constants hamesha immutable hote hain aur type annotation zaroori hai:

```rust
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159;

fn main() {
    println!("Max points: {}", MAX_POINTS);
}
```

**`const` aur `let` mein antar:**
- `const` ko type annotation dena zaroori hai
- `const` global scope mein declare ho sakta hai
- `const` ki value compile time pe pata honi chahiye
- `const` kabhi `mut` nahi ho sakta

### Shadowing
Ek hi naam se naya variable declare kar sakte ho, jo pehle wale ko "shadow" kar deta hai:

```rust
fn main() {
    let x = 5;
    let x = x + 1;      // Ab x = 6
    let x = x * 2;      // Ab x = 12

    println!("x = {}", x);  // x = 12
}
```

**Shadowing kyun use karein?**

1. Variable ka type change karna ho:
```rust
let spaces = "   ";        // &str type
let spaces = spaces.len(); // usize type - same naam, different type!
```

2. Value transform karna ho par naam same rakhna ho:
```rust
let age = "25";
let age: u32 = age.parse().unwrap();  // Ab age ek number hai
```

---

## Data Types

Rust ek **statically typed** language hai - compile time pe sabhi types pata hone chahiye.

### Scalar Types (एकल मान)

#### 1. Integers (पूर्णांक)
| Size | Signed | Unsigned |
|------|--------|----------|
| 8-bit | `i8` | `u8` |
| 16-bit | `i16` | `u16` |
| 32-bit | `i32` | `u32` |
| 64-bit | `i64` | `u64` |
| 128-bit | `i128` | `u128` |
| arch | `isize` | `usize` |

```rust
fn main() {
    let a: i32 = -42;     // Signed (negative ho sakta hai)
    let b: u32 = 42;      // Unsigned (sirf positive)
    let c = 98_222;       // Underscore readability ke liye
    let d = 0xff;         // Hexadecimal
    let e = 0o77;         // Octal
    let f = 0b1111_0000;  // Binary
    let g = b'A';         // Byte (sirf u8)
}
```

#### 2. Floating-Point Numbers (दशमलव संख्या)
```rust
fn main() {
    let x = 2.0;      // f64 (default)
    let y: f32 = 3.0; // f32

    // Operations
    let sum = 5.0 + 10.0;           // Jod
    let difference = 95.5 - 4.3;    // Ghata
    let product = 4.0 * 30.0;       // Guna
    let quotient = 56.7 / 32.2;     // Bhag
    let remainder = 43.0 % 5.0;     // Shesh
}
```

#### 3. Boolean
```rust
fn main() {
    let t = true;
    let f: bool = false;

    if t {
        println!("Ye sach hai!");
    }
}
```

#### 4. Character (अक्षर)
```rust
fn main() {
    let c = 'z';
    let heart = '❤';
    let emoji = '😀';

    // char 4 bytes ka hota hai (Unicode scalar value)
    println!("char ka size: {} bytes", std::mem::size_of::<char>());
}
```

### Compound Types (मिश्रित प्रकार)

#### 1. Tuples
Fixed-length collection jisme different types ki values ho sakti hain:

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);

    // Destructuring
    let (x, y, z) = tup;
    println!("y = {}", y);  // 6.4

    // Index se access
    let five_hundred = tup.0;
    let six_point_four = tup.1;
    let one = tup.2;
}
```

#### 2. Arrays
Fixed-length collection jisme same type ki values hoti hain:

```rust
fn main() {
    // Type annotation: [type; length]
    let arr: [i32; 5] = [1, 2, 3, 4, 5];

    // Same value se initialize
    let zeros = [0; 5];  // [0, 0, 0, 0, 0]

    // Elements access
    let first = arr[0];
    let second = arr[1];

    // Array ki length
    println!("Length: {}", arr.len());
}
```

### Strings

#### `String` vs `&str`

| Feature | `String` | `&str` |
|---------|----------|--------|
| Storage | Heap mein | Kahin bhi ho sakta |
| Ownership | Owned | Borrowed |
| Mutability | Mutable ho sakta | Immutable |
| Size | Dynamic | Fixed |

```rust
fn main() {
    // String literal - &str type (binary mein stored)
    let s1: &str = "Hello";

    // String - heap allocated, owned
    let s2: String = String::from("Hello");
    let s3: String = "Hello".to_string();

    // Conversion
    let s4: &str = &s2;              // String to &str (borrow)
    let s5: String = s1.to_owned();  // &str to String

    // Mutable String
    let mut s = String::from("Hello");
    s.push_str(", World!");  // String append
    s.push('!');             // Char append
    println!("{}", s);       // Hello, World!!
}
```

---

## Functions

### Basic Function Syntax
```rust
fn main() {
    println!("Main se hello!");
    another_function();
}

fn another_function() {
    println!("Dusre function se hello!");
}
```

### Parameters
```rust
fn main() {
    greet("Amit");
    print_sum(5, 3);
}

fn greet(name: &str) {
    println!("Namaste, {}!", name);
}

fn print_sum(x: i32, y: i32) {
    println!("{} + {} = {}", x, y, x + y);
}
```

### Return Values
```rust
fn main() {
    let result = add(5, 3);
    println!("5 + 3 = {}", result);

    let doubled = double(7);
    println!("7 * 2 = {}", doubled);
}

// Explicit return type -> ke saath
fn add(x: i32, y: i32) -> i32 {
    return x + y;  // Explicit return
}

// Implicit return (last expression pe semicolon nahi)
fn double(x: i32) -> i32 {
    x * 2  // No semicolon = ye return value hai
}
```

### Expressions vs Statements
```rust
fn main() {
    // Statement - action perform karta hai, value return nahi karta
    let x = 5;  // Ye ek statement hai

    // Expression - evaluate hoke value deta hai
    let y = {
        let x = 3;
        x + 1  // No semicolon - ye expression hai jo 4 return karta hai
    };

    println!("y = {}", y);  // y = 4
}
```

---

## Control Flow

### if Expressions
```rust
fn main() {
    let number = 7;

    if number < 5 {
        println!("5 se kam");
    } else if number == 5 {
        println!("5 ke barabar");
    } else {
        println!("5 se zyada");
    }

    // if as an expression
    let condition = true;
    let value = if condition { 5 } else { 6 };
    println!("value = {}", value);  // 5
}
```

### Loops

#### `loop` - Infinite loop
```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;  // Loop se value return
        }
    };

    println!("Result: {}", result);  // 20
}
```

#### `while` - Conditional loop
```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);
        number -= 1;
    }

    println!("LIFTOFF!");
}
```

#### `for` - Collection pe iterate
```rust
fn main() {
    // Array pe iterate
    let arr = [10, 20, 30, 40, 50];
    for element in arr {
        println!("Value: {}", element);
    }

    // Range (end exclusive)
    for number in 1..4 {
        println!("{}", number);  // 1, 2, 3
    }

    // Range (end inclusive)
    for number in 1..=4 {
        println!("{}", number);  // 1, 2, 3, 4
    }

    // Reverse range
    for number in (1..4).rev() {
        println!("{}", number);  // 3, 2, 1
    }
}
```

### Match Expression
```rust
fn main() {
    let number = 3;

    match number {
        1 => println!("Ek"),
        2 => println!("Do"),
        3 => println!("Teen"),
        4 | 5 => println!("Char ya Paanch"),  // Multiple patterns
        6..=10 => println!("Chhah se Das"),   // Range pattern
        _ => println!("Kuch aur"),            // Default case
    }

    // Match se value return
    let result = match number {
        1 => "ek",
        2 => "do",
        _ => "kuch aur",
    };
}
```

---

# Part 2: Core Concepts (मुख्य अवधारणाएं)

## Ownership

Ownership Rust ka sabse unique feature hai jo garbage collector ke bina memory safety ensure karta hai.

### Ownership ke Teen Niyam
1. **Har value ka exactly ek owner hota hai**
2. **Ek time pe sirf ek hi owner ho sakta hai**
3. **Jab owner scope se bahar jata hai, value drop ho jati hai**

### Ownership Samajhein
```rust
fn main() {
    // Rule 1: s1 String ka owner hai
    let s1 = String::from("hello");

    // Rule 2: Ownership s2 ko transfer ho gaya, s1 ab invalid hai
    let s2 = s1;

    // println!("{}", s1);  // Error! s1 ab valid nahi
    println!("{}", s2);     // Ye theek hai
}
```

### Ownership Kyun Zaroori Hai?
Samjho ki memory kaise kaam karti hai:

```
Stack (fast, fixed size)          Heap (slow, dynamic size)
+------------------+              +------------------+
| s1: ptr ---------|------------->| "hello"          |
|     len: 5       |              +------------------+
|     capacity: 5  |
+------------------+
```

Agar `s1` aur `s2` dono same memory ko point karein:
- Jab `s1` scope se bahar jaye, memory free ho jayegi
- Jab `s2` scope se bahar jaye, wahi memory dobara free karne ki koshish = **double free error!**

Rust iska solution deta hai ownership **move** karke.

### Copy vs Move

**Copy Types** (stack pe stored, copy karna sasta):
- Saare integers (`i32`, `u64`, etc.)
- `bool`
- `char`
- `f32`, `f64`
- Copy types ke Tuples: `(i32, i32)`

```rust
fn main() {
    let x = 5;
    let y = x;  // Copy hota hai, move nahi
    println!("x = {}, y = {}", x, y);  // Dono valid!
}
```

**Move Types** (heap pe stored ya complex):
- `String`
- `Vec<T>`
- Jo bhi type `Copy` implement nahi karta

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // Move hota hai, copy nahi
    // println!("{}", s1);  // Error! s1 move ho gaya
}
```

### Clone - Deep Copy
```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // Deep copy

    println!("s1 = {}, s2 = {}", s1, s2);  // Dono valid!
}
```

### Ownership aur Functions
```rust
fn main() {
    let s = String::from("hello");
    takes_ownership(s);      // s function mein move ho gaya
    // println!("{}", s);    // Error! s ab valid nahi

    let x = 5;
    makes_copy(x);           // x copy hota hai (i32 Copy hai)
    println!("x = {}", x);   // x abhi bhi valid hai
}

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}  // some_string scope se bahar jata hai aur drop hota hai

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}  // some_integer scope se bahar jata hai, kuch special nahi hota
```

### Ownership Return Karna
```rust
fn main() {
    let s1 = gives_ownership();  // Function s1 ko ownership deta hai

    let s2 = String::from("hello");
    let s3 = takes_and_gives_back(s2);  // s2 move hua, s3 ko return hua

    // s2 invalid hai, s3 valid hai
}

fn gives_ownership() -> String {
    let s = String::from("hello");
    s  // Ownership return hoti hai
}

fn takes_and_gives_back(s: String) -> String {
    s  // Ownership leke wapas deta hai
}
```

---

## Borrowing aur References

Borrowing se tum ownership liye bina value use kar sakte ho.

### Immutable References (`&`)
```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);  // s1 ko borrow kiya

    println!("'{}' ki length {} hai", s1, len);  // s1 abhi bhi valid!
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s scope se bahar jata hai, par data drop nahi hota (borrowed tha)
```

### Mutable References (`&mut`)
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);  // Mutable reference pass kiya

    println!("{}", s);  // "hello, world"
}

fn change(s: &mut String) {
    s.push_str(", world");
}
```

### Borrowing ke Niyam
1. **Ya to EK mutable reference ho sakta hai YA kitne bhi immutable references**
2. **References hamesha valid hone chahiye**

```rust
fn main() {
    let mut s = String::from("hello");

    // Multiple immutable references - OK
    let r1 = &s;
    let r2 = &s;
    println!("{} aur {}", r1, r2);

    // r1 aur r2 ka use ho gaya, ab mutable reference le sakte hain
    let r3 = &mut s;
    println!("{}", r3);

    // Ye KAAM NAHI karega:
    // let r1 = &s;
    // let r2 = &mut s;  // Error! Mutable nahi ho sakta jab immutable hai
}
```

### Dangling References - Rust Prevent Karta Hai
```rust
fn main() {
    // let reference = dangle();  // Compile nahi hoga
}

// Ye function dangling reference banata
// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s  // Error! s drop ho jayega, reference invalid hoga
// }

// Solution: Owned value return karo
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // Ownership move out ho jata hai
}
```

---

## Slices

Slices se tum collection mein contiguous sequence of elements ko reference kar sakte ho.

### String Slices (`&str`)
```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];   // "hello"
    let world = &s[6..11];  // "world"

    // Shorthand
    let hello = &s[..5];    // Start se
    let world = &s[6..];    // End tak
    let whole = &s[..];     // Puri string

    println!("{} {}", hello, world);
}
```

### Array Slices
```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];

    let slice: &[i32] = &arr[1..3];  // [2, 3]

    println!("{:?}", slice);

    // Function ko slice pass karo
    print_slice(&arr[..]);
}

fn print_slice(slice: &[i32]) {
    for item in slice {
        println!("{}", item);
    }
}
```

---

## Structs

Structs se tum related values ko group karke custom data types bana sakte ho.

### Struct Define aur Instantiate Karna
```rust
// Struct define karo
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    // Instance create karo
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("user123"),
        active: true,
        sign_in_count: 1,
    };

    // Fields access karo
    println!("Email: {}", user1.email);

    // Mutable instance
    let mut user2 = User {
        email: String::from("another@example.com"),
        username: String::from("another123"),
        active: true,
        sign_in_count: 1,
    };
    user2.email = String::from("newemail@example.com");
}
```

### Struct Update Syntax
```rust
fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("user123"),
        active: true,
        sign_in_count: 1,
    };

    // user1 se kuch values lekar user2 banao
    let user2 = User {
        email: String::from("different@example.com"),
        ..user1  // Baaki fields user1 se
    };
}
```

### Methods on Structs
```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Method - &self leta hai
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // Method with parameters
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }

    // Associated function (no self) - static method jaisa
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 10, height: 40 };

    println!("Area: {}", rect1.area());
    println!("rect2 aa sakta hai: {}", rect1.can_hold(&rect2));

    // Associated function call
    let square = Rectangle::square(20);
}
```

---

## Enums aur Pattern Matching

### Basic Enums
```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let dir = Direction::North;

    match dir {
        Direction::North => println!("Uttar ja rahe hain!"),
        Direction::South => println!("Dakshin ja rahe hain!"),
        Direction::East => println!("Purv ja rahe hain!"),
        Direction::West => println!("Pashchim ja rahe hain!"),
    }
}
```

### Enums with Data
```rust
enum Message {
    Quit,                       // Koi data nahi
    Move { x: i32, y: i32 },   // Named fields (struct jaisa)
    Write(String),              // Single value
    ChangeColor(i32, i32, i32), // Multiple values (tuple jaisa)
}

fn main() {
    let msg = Message::Move { x: 10, y: 20 };

    match msg {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Write(text) => println!("Text: {}", text),
        Message::ChangeColor(r, g, b) => println!("Color: {}, {}, {}", r, g, b),
    }
}
```

### Option Enum
Rust mein null nahi hota. Instead, `Option<T>` use hota hai:

```rust
// Rust standard library mein define hai:
// enum Option<T> {
//     Some(T),
//     None,
// }

fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // Dono cases handle karne padenge
    match some_number {
        Some(n) => println!("Number mila: {}", n),
        None => println!("Koi number nahi"),
    }

    // Useful methods
    let x = some_number.unwrap();           // Value lo ya panic karo
    let y = no_number.unwrap_or(0);         // Value lo ya default
    let z = some_number.map(|n| n * 2);     // Transform agar Some ho
}
```

### if let - Concise Pattern Matching
```rust
fn main() {
    let some_value: Option<i32> = Some(3);

    // Single pattern ke liye match ki jagah:
    if let Some(value) = some_value {
        println!("Value mili: {}", value);
    }

    // else ke saath
    if let Some(value) = some_value {
        println!("Value mili: {}", value);
    } else {
        println!("Koi value nahi");
    }
}
```

---

## Modules

### Modules Create Karna
```rust
// lib.rs ya main.rs mein
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {
            println!("Waitlist mein add ho gaya");
        }

        fn seat_at_table() {
            println!("Table pe baitha diya");
        }
    }

    mod serving {
        fn take_order() {}
        fn serve_order() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

### `use` Keyword
```rust
use std::collections::HashMap;
use std::io::{self, Write};  // Multiple imports
use std::fmt::Result as FmtResult;  // Rename

fn main() {
    let mut map = HashMap::new();
    map.insert("key", "value");
}
```

---

# Part 3: Intermediate (मध्यवर्ती)

## Error Handling

### Unrecoverable Errors with `panic!`
```rust
fn main() {
    panic!("Program crash ho gaya!");

    // Ya invalid operation se panic
    let v = vec![1, 2, 3];
    v[99];  // Ye panic karega
}
```

### Recoverable Errors with `Result`
```rust
use std::fs::File;
use std::io::{self, Read};

fn main() {
    // Result enum: Ok(T) ya Err(E)
    let file_result = File::open("hello.txt");

    let file = match file_result {
        Ok(file) => file,
        Err(error) => {
            println!("File open karne mein error: {:?}", error);
            return;
        }
    };
}
```

### Shortcuts: `unwrap` aur `expect`
```rust
fn main() {
    // unwrap - Err pe panic with default message
    let file = File::open("hello.txt").unwrap();

    // expect - Err pe panic with custom message
    let file = File::open("hello.txt")
        .expect("hello.txt open karne mein fail");
}
```

### `?` Operator se Error Propagate Karna
```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("hello.txt")?;  // Error pe early return
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}

// Aur chota chaining ke saath
fn read_username_from_file_v2() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("hello.txt")?.read_to_string(&mut username)?;
    Ok(username)
}

// Sabse chota std function use karke
fn read_username_from_file_v3() -> Result<String, io::Error> {
    std::fs::read_to_string("hello.txt")
}
```

---

## Generics

Generics se code likhte ho jo multiple types ke saath kaam kare.

### Generic Functions
```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("Sabse bada: {}", largest(&numbers));

    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Sabse bada: {}", largest(&chars));
}
```

### Generic Structs
```rust
struct Point<T> {
    x: T,
    y: T,
}

// x aur y ke liye different types
struct Point2<T, U> {
    x: T,
    y: U,
}

fn main() {
    let integer_point = Point { x: 5, y: 10 };
    let float_point = Point { x: 1.0, y: 4.0 };
    let mixed_point = Point2 { x: 5, y: 4.0 };
}
```

### Generic Methods
```rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// Sirf specific type ke liye implementation
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

---

## Traits

Traits shared behavior define karte hain types ke across (interfaces jaisa).

### Traits Define Karna
```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation
    fn summarize_author(&self) -> String {
        String::from("(Anonymous)")
    }
}
```

### Traits Implement Karna
```rust
struct NewsArticle {
    headline: String,
    author: String,
    content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.headline, self.author)
    }
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.content)
    }

    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

### Traits as Parameters
```rust
// impl Trait use karke
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// Trait bound syntax (same hai)
fn notify_v2<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// Multiple trait bounds
fn notify_v3(item: &(impl Summary + Display)) {}

// Where clause for cleaner syntax
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    0
}
```

---

## Lifetimes

Lifetimes ensure karte hain ki references jab tak chahiye tab tak valid rahein.

### Problem Jo Lifetimes Solve Karti Hai
```rust
// Ye compile nahi hoga - return value ki lifetime kya hogi?
// fn longest(x: &str, y: &str) -> &str {
//     if x.len() > y.len() { x } else { y }
// }
```

### Lifetime Annotations
```rust
// 'a ek lifetime parameter hai
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("lambi string");
    let result;

    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
        println!("Sabse lamba: {}", result);  // Is scope mein use karna padega
    }
    // println!("{}", result);  // Error! string2 scope se bahar hai
}
```

### Lifetime Elision Rules
Rust ye rules apply karta hai lifetimes infer karne ke liye:

1. Har input reference ko apni lifetime milti hai
2. Agar exactly ek input lifetime ho, wo sabhi outputs ko assign hoti hai
3. Agar `&self` ya `&mut self` ho, self ki lifetime outputs ko assign hoti hai

```rust
// Ye dono same hain:
fn first_word(s: &str) -> &str { /* ... */ }
fn first_word<'a>(s: &'a str) -> &'a str { /* ... */ }
```

### Static Lifetime
```rust
// Pure program duration tak live karti hai
let s: &'static str = "Meri static lifetime hai.";

// String literals hamesha 'static hote hain
```

---

## Closures

Closures anonymous functions hain jo apne environment ko capture kar sakte hain.

### Basic Closure Syntax
```rust
fn main() {
    // Full syntax
    let add_one = |x: i32| -> i32 { x + 1 };

    // Simplified (type inference)
    let add_two = |x| x + 2;

    // Multiple parameters
    let add = |a, b| a + b;

    println!("{}", add_one(5));  // 6
    println!("{}", add_two(5));  // 7
    println!("{}", add(2, 3));   // 5
}
```

### Environment Capture Karna
```rust
fn main() {
    let x = 4;

    // Closure x ko reference se capture karta hai
    let equal_to_x = |z| z == x;

    println!("{}", equal_to_x(4));  // true
}
```

### Teen Tarike se Capture
```rust
fn main() {
    let s = String::from("hello");

    // 1. Reference se (Fn) - immutable borrow
    let print_s = || println!("{}", s);
    print_s();
    println!("{}", s);  // s abhi bhi valid

    // 2. Mutable reference se (FnMut) - mutable borrow
    let mut count = 0;
    let mut increment = || {
        count += 1;
    };
    increment();
    increment();
    println!("Count: {}", count);  // 2

    // 3. Value se (FnOnce) - ownership leti hai
    let s2 = String::from("world");
    let consume = move || {
        println!("{}", s2);
    };
    consume();
    // println!("{}", s2);  // Error! s2 move ho gaya
}
```

---

## Iterators

Iterators lazily sequences of elements process karte hain.

### Iterators Create Karna
```rust
fn main() {
    let v = vec![1, 2, 3];

    // iter() - elements borrow karta hai
    for val in v.iter() {
        println!("{}", val);
    }

    // iter_mut() - mutable borrow
    let mut v2 = vec![1, 2, 3];
    for val in v2.iter_mut() {
        *val *= 2;
    }

    // into_iter() - ownership leta hai
    for val in v.into_iter() {
        println!("{}", val);
    }
    // v ab valid nahi
}
```

### Iterator Adaptors
```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    // map - elements transform karo
    let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();

    // filter - matching elements rakho
    let evens: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();

    // Chain adaptors
    let result: Vec<i32> = v.iter()
        .filter(|x| *x % 2 == 0)
        .map(|x| x * 2)
        .collect();
}
```

### Consuming Adaptors
```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let sum: i32 = v.iter().sum();
    let product: i32 = v.iter().product();
    let count = v.iter().count();
    let max = v.iter().max();
    let min = v.iter().min();
    let any_even = v.iter().any(|x| x % 2 == 0);
    let all_positive = v.iter().all(|x| *x > 0);
}
```

---

## Collections

### Vec<T> - Dynamic Array
```rust
fn main() {
    // Vectors create karna
    let v1: Vec<i32> = Vec::new();
    let v2 = vec![1, 2, 3];

    // Elements add karna
    let mut v = Vec::new();
    v.push(5);
    v.push(6);
    v.push(7);

    // Elements read karna
    let third: &i32 = &v[2];
    let third: Option<&i32> = v.get(2);  // Safer

    // Iterate karna
    for i in &v {
        println!("{}", i);
    }

    // Mutate karna iterate karte waqt
    for i in &mut v {
        *i += 10;
    }

    // Elements remove karna
    let last = v.pop();  // Option<T> return karta hai
}
```

### HashMap<K, V> - Key-Value Store
```rust
use std::collections::HashMap;

fn main() {
    // Create karna
    let mut scores = HashMap::new();

    // Insert karna
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    // Access karna
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);  // Option<&V> return

    // Iterate karna
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }

    // Sirf update karo agar key nahi hai
    scores.entry(String::from("Blue")).or_insert(50);

    // Old value ke basis pe update
    let text = "hello world wonderful world";
    let mut word_count = HashMap::new();
    for word in text.split_whitespace() {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }
}
```

---

# Part 4: Advanced (उन्नत)

## Smart Pointers

Smart pointers data structures hain jo pointers ki tarah kaam karte hain par extra metadata aur capabilities rakhte hain.

### Box<T> - Heap Allocation
```rust
fn main() {
    // Data heap pe store karo
    let b = Box::new(5);
    println!("b = {}", b);

    // Recursive types ko Box chahiye
    enum List {
        Cons(i32, Box<List>),
        Nil,
    }

    use List::{Cons, Nil};
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
}
```

### Rc<T> - Reference Counting
```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(5);
    println!("Count: {}", Rc::strong_count(&a));  // 1

    let b = Rc::clone(&a);
    println!("Count: {}", Rc::strong_count(&a));  // 2

    {
        let c = Rc::clone(&a);
        println!("Count: {}", Rc::strong_count(&a));  // 3
    }

    println!("Count: {}", Rc::strong_count(&a));  // 2
}
```

### RefCell<T> - Interior Mutability
```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);

    // Borrow rules runtime pe check hote hain
    {
        let mut borrowed = data.borrow_mut();
        *borrowed += 1;
    }

    println!("Data: {:?}", data.borrow());  // 6
}
```

### Arc<T> - Thread-Safe Reference Counting
```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("Thread {}: {:?}", i, data);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

---

## Concurrency aur Threads

### Threads Create Karna
```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("Spawned thread: {}", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("Main thread: {}", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();  // Thread finish hone ka wait karo
}
```

### Data ko Threads mein Move Karna
```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    // move keyword ownership thread ko transfer karta hai
    let handle = thread::spawn(move || {
        println!("Vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

### Channels se Message Passing
```rust
use std::sync::mpsc;  // Multiple producer, single consumer
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("namaste");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Mila: {}", received);
}
```

### Mutex se Shared State
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

    println!("Result: {}", *counter.lock().unwrap());  // 10
}
```

---

## Async/Await

### Basic Async Function
```rust
async fn hello() {
    println!("Hello, async!");
}

// Async functions Future return karti hain
// Execute karne ke liye runtime chahiye

#[tokio::main]  // Tokio runtime use kar rahe hain
async fn main() {
    hello().await;  // .await Future execute karta hai
}
```

### async/await Samajhein
```rust
use tokio::time::{sleep, Duration};

async fn fetch_data() -> String {
    sleep(Duration::from_secs(2)).await;
    String::from("Data fetch ho gaya!")
}

async fn process_data() {
    println!("Data fetch shuru...");
    let data = fetch_data().await;  // Yahan wait karta hai, thread block nahi hota
    println!("Mila: {}", data);
}

#[tokio::main]
async fn main() {
    process_data().await;
}
```

### Tasks Concurrently Run Karna
```rust
use tokio::time::{sleep, Duration};

async fn task1() {
    sleep(Duration::from_secs(1)).await;
    println!("Task 1 complete");
}

async fn task2() {
    sleep(Duration::from_secs(2)).await;
    println!("Task 2 complete");
}

#[tokio::main]
async fn main() {
    // Concurrently run karo
    tokio::join!(task1(), task2());

    // Ya separate tasks spawn karo
    let handle1 = tokio::spawn(task1());
    let handle2 = tokio::spawn(task2());

    handle1.await.unwrap();
    handle2.await.unwrap();
}
```

---

## Unsafe Rust

### Unsafe mein Kya Kar Sakte Ho
1. Raw pointers dereference karo
2. Unsafe functions call karo
3. Mutable static variables access ya modify karo
4. Unsafe traits implement karo
5. Union fields access karo

### Raw Pointers
```rust
fn main() {
    let mut num = 5;

    // Raw pointers create karna safe hai
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    // Dereference karne ke liye unsafe chahiye
    unsafe {
        println!("r1: {}", *r1);
        println!("r2: {}", *r2);
    }
}
```

### Unsafe Functions
```rust
unsafe fn dangerous() {
    println!("Ye dangerous hai!");
}

fn main() {
    unsafe {
        dangerous();
    }
}
```

### Safe Abstraction over Unsafe Code
```rust
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

---

## Macros

### Declarative Macros (macro_rules!)
```rust
// Simple macro
macro_rules! say_hello {
    () => {
        println!("Namaste!");
    };
}

// Argument ke saath macro
macro_rules! create_function {
    ($func_name:ident) => {
        fn $func_name() {
            println!("Function {:?} call hua", stringify!($func_name));
        }
    };
}

// Multiple patterns ke saath macro
macro_rules! print_result {
    ($expression:expr) => {
        println!("{:?} = {:?}", stringify!($expression), $expression);
    };
}

fn main() {
    say_hello!();

    create_function!(foo);
    foo();

    print_result!(1 + 2);  // "1 + 2" = 3
}
```

### Repetition in Macros
```rust
macro_rules! vec_of_strings {
    ($($element:expr),*) => {
        {
            let mut v = Vec::new();
            $(
                v.push(String::from($element));
            )*
            v
        }
    };
}

fn main() {
    let v = vec_of_strings!["a", "b", "c"];
    println!("{:?}", v);
}
```

---

## Summary

### Rust ke Key Principles
1. **Memory Safety** - No null pointers, no dangling references
2. **Zero-Cost Abstractions** - High-level code efficient machine code mein compile hota hai
3. **Fearless Concurrency** - Compiler data races prevent karta hai
4. **Ownership System** - Memory management ke clear rules

### Kab Kya Use Karein

| Problem | Solution |
|---------|----------|
| Single value heap pe store | `Box<T>` |
| Single thread mein data share | `Rc<T>` |
| Threads ke beech data share | `Arc<T>` |
| Interior mutability | `RefCell<T>` ya `Cell<T>` |
| Thread-safe mutability | `Mutex<T>` ya `RwLock<T>` |
| Optional value | `Option<T>` |
| Fail ho sakne wala operation | `Result<T, E>` |

### Learning Path
1. **Basics**: Variables, functions, control flow
2. **Core Concepts**: Ownership, borrowing, lifetimes
3. **Intermediate**: Error handling, traits, generics
4. **Advanced**: Async, unsafe, macros

Rust seekhne mein all the best!
