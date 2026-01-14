# Complete Rust Learning Guide - Basic to Advanced

A comprehensive guide to learn Rust programming from basics to advanced topics with clear explanations and examples.

---

## Table of Contents

### Part 1: Basics
1. [Introduction to Rust](#introduction-to-rust)
2. [Variables and Mutability](#variables-and-mutability)
3. [Data Types](#data-types)
4. [Functions](#functions)
5. [Control Flow](#control-flow)
6. [Comments](#comments)

### Part 2: Core Concepts
7. [Ownership](#ownership)
8. [Borrowing and References](#borrowing-and-references)
9. [Slices](#slices)
10. [Structs](#structs)
11. [Enums and Pattern Matching](#enums-and-pattern-matching)
12. [Modules and Packages](#modules-and-packages)

### Part 3: Intermediate
13. [Error Handling](#error-handling)
14. [Generics](#generics)
15. [Traits](#traits)
16. [Lifetimes](#lifetimes)
17. [Closures](#closures)
18. [Iterators](#iterators)
19. [Collections](#collections)

### Part 4: Advanced
20. [Smart Pointers](#smart-pointers)
21. [Concurrency and Threads](#concurrency-and-threads)
22. [Async/Await](#asyncawait)
23. [Unsafe Rust](#unsafe-rust)
24. [Macros](#macros)
25. [FFI (Foreign Function Interface)](#ffi-foreign-function-interface)

---

# Part 1: Basics

## Introduction to Rust

### What is Rust?
Rust is a systems programming language focused on three goals:
- **Safety**: Memory safety without garbage collection
- **Speed**: Zero-cost abstractions and efficient code
- **Concurrency**: Fearless concurrent programming

### Why Learn Rust?
1. **Memory Safety**: No null pointers, no dangling references, no buffer overflows
2. **Performance**: As fast as C/C++ with high-level abstractions
3. **Modern Tooling**: Cargo (package manager), rustfmt (formatter), clippy (linter)
4. **Growing Ecosystem**: Web (Actix, Rocket), Systems (Tokio), WebAssembly
5. **Industry Adoption**: Used by Mozilla, Microsoft, Google, Amazon, Discord

### Hello World
```rust
fn main() {
    println!("Hello, World!");
}
```

**Explanation:**
- `fn main()` - Entry point of every Rust program
- `println!` - Macro for printing to console (note the `!`)
- Statements end with semicolons `;`

---

## Variables and Mutability

### Immutable by Default
In Rust, variables are **immutable by default**. This means once you assign a value, you cannot change it.

```rust
fn main() {
    let x = 5;
    // x = 6;  // Error! Cannot assign twice to immutable variable
    println!("x = {}", x);
}
```

### Making Variables Mutable
Use `mut` keyword to make a variable mutable:

```rust
fn main() {
    let mut x = 5;
    println!("x = {}", x);  // x = 5

    x = 6;  // This is allowed now
    println!("x = {}", x);  // x = 6
}
```

### Constants
Constants are always immutable and must have type annotation:

```rust
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159;

fn main() {
    println!("Max points: {}", MAX_POINTS);
}
```

**Difference between `const` and `let`:**
- `const` must have type annotation
- `const` can be declared in global scope
- `const` value must be known at compile time
- `const` cannot be `mut`

### Shadowing
You can declare a new variable with the same name, which "shadows" the previous one:

```rust
fn main() {
    let x = 5;
    let x = x + 1;      // x is now 6
    let x = x * 2;      // x is now 12

    println!("x = {}", x);  // x = 12
}
```

**Why use shadowing?**
1. Change the type of a variable:
```rust
let spaces = "   ";        // &str type
let spaces = spaces.len(); // usize type - same name, different type!
```

2. Transform a value while keeping the name:
```rust
let age = "25";
let age: u32 = age.parse().unwrap();  // Now age is a number
```

---

## Data Types

Rust is a **statically typed** language - all types must be known at compile time.

### Scalar Types

#### 1. Integers
| Length | Signed | Unsigned |
|--------|--------|----------|
| 8-bit | `i8` | `u8` |
| 16-bit | `i16` | `u16` |
| 32-bit | `i32` | `u32` |
| 64-bit | `i64` | `u64` |
| 128-bit | `i128` | `u128` |
| arch | `isize` | `usize` |

```rust
fn main() {
    let a: i32 = -42;     // Signed (can be negative)
    let b: u32 = 42;      // Unsigned (only positive)
    let c = 98_222;       // Underscores for readability
    let d = 0xff;         // Hexadecimal
    let e = 0o77;         // Octal
    let f = 0b1111_0000;  // Binary
    let g = b'A';         // Byte (u8 only)
}
```

#### 2. Floating-Point Numbers
```rust
fn main() {
    let x = 2.0;      // f64 (default)
    let y: f32 = 3.0; // f32

    // Operations
    let sum = 5.0 + 10.0;
    let difference = 95.5 - 4.3;
    let product = 4.0 * 30.0;
    let quotient = 56.7 / 32.2;
    let remainder = 43.0 % 5.0;
}
```

#### 3. Boolean
```rust
fn main() {
    let t = true;
    let f: bool = false;

    if t {
        println!("It's true!");
    }
}
```

#### 4. Character
```rust
fn main() {
    let c = 'z';
    let heart = '❤';
    let emoji = '😀';

    // char is 4 bytes (Unicode scalar value)
    println!("Size of char: {} bytes", std::mem::size_of::<char>());
}
```

### Compound Types

#### 1. Tuples
Fixed-length collection of values with different types:

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);

    // Destructuring
    let (x, y, z) = tup;
    println!("y = {}", y);  // 6.4

    // Access by index
    let five_hundred = tup.0;
    let six_point_four = tup.1;
    let one = tup.2;
}
```

#### 2. Arrays
Fixed-length collection of values with same type:

```rust
fn main() {
    // Type annotation: [type; length]
    let arr: [i32; 5] = [1, 2, 3, 4, 5];

    // Initialize with same value
    let zeros = [0; 5];  // [0, 0, 0, 0, 0]

    // Access elements
    let first = arr[0];
    let second = arr[1];

    // Array length
    println!("Length: {}", arr.len());
}
```

### Strings

#### `String` vs `&str`

| Feature | `String` | `&str` |
|---------|----------|--------|
| Storage | Heap | Can be anywhere |
| Ownership | Owned | Borrowed |
| Mutability | Can be mutable | Immutable |
| Size | Dynamic | Fixed |

```rust
fn main() {
    // String literal - &str type (stored in binary)
    let s1: &str = "Hello";

    // String - heap allocated, owned
    let s2: String = String::from("Hello");
    let s3: String = "Hello".to_string();

    // Converting between them
    let s4: &str = &s2;           // String to &str (borrow)
    let s5: String = s1.to_owned(); // &str to String

    // Mutable String
    let mut s = String::from("Hello");
    s.push_str(", World!");  // Append string
    s.push('!');             // Append char
    println!("{}", s);       // Hello, World!!
}
```

---

## Functions

### Basic Function Syntax
```rust
fn main() {
    println!("Hello from main!");
    another_function();
}

fn another_function() {
    println!("Hello from another function!");
}
```

### Parameters
```rust
fn main() {
    greet("Alice");
    print_sum(5, 3);
}

fn greet(name: &str) {
    println!("Hello, {}!", name);
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

// Explicit return type with ->
fn add(x: i32, y: i32) -> i32 {
    return x + y;  // Explicit return
}

// Implicit return (no semicolon on last expression)
fn double(x: i32) -> i32 {
    x * 2  // No semicolon = this is the return value
}
```

### Expressions vs Statements
```rust
fn main() {
    // Statement - performs action, doesn't return value
    let x = 5;  // This is a statement

    // Expression - evaluates to a value
    let y = {
        let x = 3;
        x + 1  // No semicolon - this is an expression that returns 4
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
        println!("Less than 5");
    } else if number == 5 {
        println!("Equal to 5");
    } else {
        println!("Greater than 5");
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
            break counter * 2;  // Return value from loop
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

#### `for` - Iterating over collections
```rust
fn main() {
    // Iterate over array
    let arr = [10, 20, 30, 40, 50];
    for element in arr {
        println!("Value: {}", element);
    }

    // Range (exclusive end)
    for number in 1..4 {
        println!("{}", number);  // 1, 2, 3
    }

    // Range (inclusive end)
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
        1 => println!("One"),
        2 => println!("Two"),
        3 => println!("Three"),
        4 | 5 => println!("Four or Five"),  // Multiple patterns
        6..=10 => println!("Six to Ten"),   // Range pattern
        _ => println!("Something else"),    // Default case
    }

    // Match with return value
    let result = match number {
        1 => "one",
        2 => "two",
        _ => "other",
    };
}
```

---

## Comments

```rust
// This is a single-line comment

/* This is a
   multi-line comment */

/// This is a documentation comment for the following item
/// It supports **Markdown** formatting
fn documented_function() {
    //! This is a documentation comment for the enclosing item
}
```

---

# Part 2: Core Concepts

## Ownership

Ownership is Rust's most unique feature and enables memory safety without garbage collection.

### Three Rules of Ownership
1. **Each value has exactly one owner**
2. **There can only be one owner at a time**
3. **When the owner goes out of scope, the value is dropped**

### Understanding Ownership
```rust
fn main() {
    // Rule 1: s1 owns the String
    let s1 = String::from("hello");

    // Rule 2: Ownership moves to s2, s1 is now invalid
    let s2 = s1;

    // println!("{}", s1);  // Error! s1 no longer valid
    println!("{}", s2);     // Works fine
}
```

### Why Does Ownership Exist?
Consider how memory works:

```
Stack (fast, fixed size)          Heap (slower, dynamic size)
+------------------+              +------------------+
| s1: ptr ---------|------------->| "hello"          |
|     len: 5       |              +------------------+
|     capacity: 5  |
+------------------+
```

If both `s1` and `s2` pointed to the same memory:
- When `s1` goes out of scope, memory is freed
- When `s2` goes out of scope, it tries to free same memory = **double free error!**

Rust solves this by **moving** ownership.

### Copy vs Move

**Copy Types** (stored on stack, cheap to copy):
- All integers (`i32`, `u64`, etc.)
- `bool`
- `char`
- `f32`, `f64`
- Tuples of Copy types: `(i32, i32)`

```rust
fn main() {
    let x = 5;
    let y = x;  // Copy, not move
    println!("x = {}, y = {}", x, y);  // Both valid!
}
```

**Move Types** (stored on heap or complex):
- `String`
- `Vec<T>`
- Any type that doesn't implement `Copy`

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // Move, not copy
    // println!("{}", s1);  // Error! s1 was moved
}
```

### Clone - Deep Copy
```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // Deep copy

    println!("s1 = {}, s2 = {}", s1, s2);  // Both valid!
}
```

### Ownership and Functions
```rust
fn main() {
    let s = String::from("hello");
    takes_ownership(s);      // s is moved into function
    // println!("{}", s);    // Error! s is no longer valid

    let x = 5;
    makes_copy(x);           // x is copied (i32 is Copy)
    println!("x = {}", x);   // x is still valid
}

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}  // some_string goes out of scope and is dropped

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}  // some_integer goes out of scope, nothing special happens
```

### Returning Ownership
```rust
fn main() {
    let s1 = gives_ownership();  // Function returns ownership to s1

    let s2 = String::from("hello");
    let s3 = takes_and_gives_back(s2);  // s2 moved in, returned to s3

    // s2 is invalid, s3 is valid
}

fn gives_ownership() -> String {
    let s = String::from("hello");
    s  // Ownership is returned
}

fn takes_and_gives_back(s: String) -> String {
    s  // Takes and returns ownership
}
```

---

## Borrowing and References

Borrowing lets you use a value without taking ownership.

### Immutable References (`&`)
```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);  // Borrow s1

    println!("Length of '{}' is {}", s1, len);  // s1 still valid!
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s goes out of scope, but doesn't drop data (it's borrowed)
```

### Mutable References (`&mut`)
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);  // Pass mutable reference

    println!("{}", s);  // "hello, world"
}

fn change(s: &mut String) {
    s.push_str(", world");
}
```

### Borrowing Rules
1. **You can have either ONE mutable reference OR any number of immutable references**
2. **References must always be valid**

```rust
fn main() {
    let mut s = String::from("hello");

    // Multiple immutable references - OK
    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);

    // After r1 and r2 are done being used, we can have a mutable reference
    let r3 = &mut s;
    println!("{}", r3);

    // This would NOT work:
    // let r1 = &s;
    // let r2 = &mut s;  // Error! Can't have mutable while immutable exists
}
```

### Dangling References - Prevented by Rust
```rust
fn main() {
    // let reference = dangle();  // Would not compile
}

// This function would create a dangling reference
// fn dangle() -> &String {
//     let s = String::from("hello");
//     &s  // Error! s is dropped, reference would be invalid
// }

// Solution: Return owned value
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // Ownership is moved out
}
```

---

## Slices

Slices let you reference a contiguous sequence of elements in a collection.

### String Slices (`&str`)
```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];   // "hello"
    let world = &s[6..11];  // "world"

    // Shorthand
    let hello = &s[..5];    // Start from beginning
    let world = &s[6..];    // Go to end
    let whole = &s[..];     // Entire string

    println!("{} {}", hello, world);
}
```

### Why Slices are Safe
```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);

    // s.clear();  // Error! Can't mutate while borrowed

    println!("First word: {}", word);
}
```

### Array Slices
```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];

    let slice: &[i32] = &arr[1..3];  // [2, 3]

    println!("{:?}", slice);

    // Pass slice to function
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

Structs let you create custom data types by grouping related values.

### Defining and Instantiating Structs
```rust
// Define a struct
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    // Create an instance
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("user123"),
        active: true,
        sign_in_count: 1,
    };

    // Access fields
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

    // Create user2 with some values from user1
    let user2 = User {
        email: String::from("different@example.com"),
        ..user1  // Fill remaining fields from user1
    };
}
```

### Tuple Structs
```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);

    // Access by index
    println!("Red: {}", black.0);
}
```

### Unit-Like Structs
```rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```

### Methods on Structs
```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Method - takes &self
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // Method with parameters
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }

    // Associated function (no self) - like static method
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
    println!("Can hold rect2: {}", rect1.can_hold(&rect2));

    // Call associated function
    let square = Rectangle::square(20);
}
```

### Deriving Traits
```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();

    println!("{:?}", p1);           // Debug format
    println!("{:#?}", p1);          // Pretty Debug format
    println!("Equal: {}", p1 == p2); // PartialEq
}
```

---

## Enums and Pattern Matching

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
        Direction::North => println!("Going north!"),
        Direction::South => println!("Going south!"),
        Direction::East => println!("Going east!"),
        Direction::West => println!("Going west!"),
    }
}
```

### Enums with Data
```rust
enum Message {
    Quit,                       // No data
    Move { x: i32, y: i32 },   // Named fields (like struct)
    Write(String),              // Single value
    ChangeColor(i32, i32, i32), // Multiple values (like tuple)
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

### Methods on Enums
```rust
impl Message {
    fn call(&self) {
        match self {
            Message::Write(text) => println!("Writing: {}", text),
            _ => println!("Other message"),
        }
    }
}
```

### Option Enum
Rust has no null. Instead, use `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}

fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // Must handle both cases
    match some_number {
        Some(n) => println!("Got number: {}", n),
        None => println!("No number"),
    }

    // Useful methods
    let x = some_number.unwrap();           // Get value or panic
    let y = no_number.unwrap_or(0);         // Get value or default
    let z = some_number.map(|n| n * 2);     // Transform if Some
}
```

### if let - Concise Pattern Matching
```rust
fn main() {
    let some_value: Option<i32> = Some(3);

    // Instead of match for single pattern:
    if let Some(value) = some_value {
        println!("Got value: {}", value);
    }

    // With else
    if let Some(value) = some_value {
        println!("Got value: {}", value);
    } else {
        println!("No value");
    }
}
```

### while let
```rust
fn main() {
    let mut stack = vec![1, 2, 3];

    while let Some(top) = stack.pop() {
        println!("Popped: {}", top);
    }
}
```

---

## Modules and Packages

### Creating Modules
```rust
// In lib.rs or main.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {
            println!("Adding to waitlist");
        }

        fn seat_at_table() {
            println!("Seating at table");
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

### The `use` Keyword
```rust
use std::collections::HashMap;
use std::io::{self, Write};  // Multiple imports
use std::fmt::Result as FmtResult;  // Rename

fn main() {
    let mut map = HashMap::new();
    map.insert("key", "value");
}
```

### Re-exporting with `pub use`
```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

// Now external code can use: library::hosting::add_to_waitlist()
```

### Separating Modules into Files
```
src/
├── main.rs
├── front_of_house.rs
└── front_of_house/
    └── hosting.rs
```

```rust
// main.rs
mod front_of_house;
use front_of_house::hosting;

fn main() {
    hosting::add_to_waitlist();
}

// front_of_house.rs
pub mod hosting;

// front_of_house/hosting.rs
pub fn add_to_waitlist() {}
```

---

# Part 3: Intermediate

## Error Handling

### Unrecoverable Errors with `panic!`
```rust
fn main() {
    panic!("crash and burn");

    // Or causing panic through invalid operations
    let v = vec![1, 2, 3];
    v[99];  // This will panic
}
```

### Recoverable Errors with `Result`
```rust
use std::fs::File;
use std::io::{self, Read};

fn main() {
    // Result enum: Ok(T) or Err(E)
    let file_result = File::open("hello.txt");

    let file = match file_result {
        Ok(file) => file,
        Err(error) => {
            println!("Error opening file: {:?}", error);
            return;
        }
    };
}
```

### Shortcuts: `unwrap` and `expect`
```rust
fn main() {
    // unwrap - panic with default message if Err
    let file = File::open("hello.txt").unwrap();

    // expect - panic with custom message if Err
    let file = File::open("hello.txt")
        .expect("Failed to open hello.txt");
}
```

### Propagating Errors with `?`
```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("hello.txt")?;  // Returns Err early if error
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}

// Even shorter with chaining
fn read_username_from_file_v2() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("hello.txt")?.read_to_string(&mut username)?;
    Ok(username)
}

// Shortest using std function
fn read_username_from_file_v3() -> Result<String, io::Error> {
    std::fs::read_to_string("hello.txt")
}
```

### Custom Error Types
```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    IoError(io::Error),
    ParseError(String),
    NotFound,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::IoError(e) => write!(f, "IO error: {}", e),
            AppError::ParseError(s) => write!(f, "Parse error: {}", s),
            AppError::NotFound => write!(f, "Not found"),
        }
    }
}

impl std::error::Error for AppError {}
```

---

## Generics

Generics allow writing code that works with multiple types.

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
    println!("Largest: {}", largest(&numbers));

    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Largest: {}", largest(&chars));
}
```

### Generic Structs
```rust
struct Point<T> {
    x: T,
    y: T,
}

// Different types for x and y
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

// Implementation only for specific type
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### Generic Enums
```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

---

## Traits

Traits define shared behavior across types (similar to interfaces).

### Defining Traits
```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation
    fn summarize_author(&self) -> String {
        String::from("(Anonymous)")
    }
}
```

### Implementing Traits
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
// Using impl Trait
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// Trait bound syntax (equivalent)
fn notify_v2<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}

// Multiple trait bounds
fn notify_v3(item: &(impl Summary + Display)) {}

fn notify_v4<T: Summary + Display>(item: &T) {}

// Where clauses for cleaner syntax
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
    0
}
```

### Returning Traits
```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("user"),
        content: String::from("Hello!"),
    }
}
```

### Trait Objects (Dynamic Dispatch)
```rust
// Static dispatch (impl Trait) - faster
fn static_dispatch(item: &impl Summary) {}

// Dynamic dispatch (dyn Trait) - more flexible
fn dynamic_dispatch(item: &dyn Summary) {}

fn main() {
    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(NewsArticle { /* ... */ }),
        Box::new(Tweet { /* ... */ }),
    ];

    for item in items {
        println!("{}", item.summarize());
    }
}
```

---

## Lifetimes

Lifetimes ensure that references are valid for as long as they're needed.

### The Problem Lifetimes Solve
```rust
// This won't compile - which lifetime should the return value have?
// fn longest(x: &str, y: &str) -> &str {
//     if x.len() > y.len() { x } else { y }
// }
```

### Lifetime Annotations
```rust
// 'a is a lifetime parameter
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("long string");
    let result;

    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
        println!("Longest: {}", result);  // Must use result within this scope
    }
    // println!("{}", result);  // Error! string2 is out of scope
}
```

### Lifetime Elision Rules
Rust applies these rules to infer lifetimes:

1. Each input reference gets its own lifetime
2. If there's exactly one input lifetime, it's assigned to all outputs
3. If there's `&self` or `&mut self`, self's lifetime is assigned to outputs

```rust
// These are equivalent:
fn first_word(s: &str) -> &str { /* ... */ }
fn first_word<'a>(s: &'a str) -> &'a str { /* ... */ }
```

### Lifetimes in Structs
```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();

    let excerpt = ImportantExcerpt {
        part: first_sentence,
    };
}
```

### Static Lifetime
```rust
// Lives for entire program duration
let s: &'static str = "I have a static lifetime.";

// String literals are always 'static
```

---

## Closures

Closures are anonymous functions that can capture their environment.

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

### Capturing the Environment
```rust
fn main() {
    let x = 4;

    // Closure captures x by reference
    let equal_to_x = |z| z == x;

    println!("{}", equal_to_x(4));  // true
}
```

### Three Ways to Capture
```rust
fn main() {
    let s = String::from("hello");

    // 1. By reference (Fn) - borrows immutably
    let print_s = || println!("{}", s);
    print_s();
    println!("{}", s);  // s still valid

    // 2. By mutable reference (FnMut) - borrows mutably
    let mut count = 0;
    let mut increment = || {
        count += 1;
    };
    increment();
    increment();
    println!("Count: {}", count);  // 2

    // 3. By value (FnOnce) - takes ownership
    let s2 = String::from("world");
    let consume = move || {
        println!("{}", s2);
    };
    consume();
    // println!("{}", s2);  // Error! s2 was moved
}
```

### Closure Traits
- `Fn`: Borrows values immutably
- `FnMut`: Borrows values mutably
- `FnOnce`: Takes ownership (can only be called once)

```rust
fn apply<F>(f: F)
where
    F: FnOnce(),
{
    f();
}

fn apply_ref<F>(f: F)
where
    F: Fn(),
{
    f();
    f();  // Can call multiple times
}
```

---

## Iterators

Iterators process sequences of elements lazily.

### Creating Iterators
```rust
fn main() {
    let v = vec![1, 2, 3];

    // iter() - borrows elements
    for val in v.iter() {
        println!("{}", val);
    }

    // iter_mut() - mutable borrows
    let mut v2 = vec![1, 2, 3];
    for val in v2.iter_mut() {
        *val *= 2;
    }

    // into_iter() - takes ownership
    for val in v.into_iter() {
        println!("{}", val);
    }
    // v is no longer valid
}
```

### Iterator Adaptors
```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    // map - transform elements
    let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();

    // filter - keep elements matching predicate
    let evens: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();

    // chain adaptors
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
    let first = v.iter().next();
    let found = v.iter().find(|x| **x == 3);
}
```

### Creating Custom Iterators
```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let counter = Counter::new();
    for num in counter {
        println!("{}", num);  // 1, 2, 3, 4, 5
    }
}
```

---

## Collections

### Vec<T> - Dynamic Array
```rust
fn main() {
    // Creating vectors
    let v1: Vec<i32> = Vec::new();
    let v2 = vec![1, 2, 3];

    // Adding elements
    let mut v = Vec::new();
    v.push(5);
    v.push(6);
    v.push(7);

    // Reading elements
    let third: &i32 = &v[2];
    let third: Option<&i32> = v.get(2);  // Safer

    // Iterating
    for i in &v {
        println!("{}", i);
    }

    // Mutating while iterating
    for i in &mut v {
        *i += 10;
    }

    // Removing elements
    let last = v.pop();  // Returns Option<T>
}
```

### HashMap<K, V> - Key-Value Store
```rust
use std::collections::HashMap;

fn main() {
    // Creating
    let mut scores = HashMap::new();

    // Inserting
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    // Accessing
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);  // Returns Option<&V>

    // Iterating
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }

    // Update only if key doesn't exist
    scores.entry(String::from("Blue")).or_insert(50);

    // Update based on old value
    let text = "hello world wonderful world";
    let mut word_count = HashMap::new();
    for word in text.split_whitespace() {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }
}
```

### HashSet<T> - Unique Values
```rust
use std::collections::HashSet;

fn main() {
    let mut set = HashSet::new();

    set.insert(1);
    set.insert(2);
    set.insert(2);  // Ignored, already exists

    println!("Contains 1: {}", set.contains(&1));

    // Set operations
    let set1: HashSet<i32> = [1, 2, 3].iter().cloned().collect();
    let set2: HashSet<i32> = [2, 3, 4].iter().cloned().collect();

    let union: HashSet<_> = set1.union(&set2).collect();
    let intersection: HashSet<_> = set1.intersection(&set2).collect();
    let difference: HashSet<_> = set1.difference(&set2).collect();
}
```

---

# Part 4: Advanced

## Smart Pointers

Smart pointers are data structures that act like pointers but have additional metadata and capabilities.

### Box<T> - Heap Allocation
```rust
fn main() {
    // Store data on heap
    let b = Box::new(5);
    println!("b = {}", b);

    // Recursive types need Box
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

    // Borrow rules checked at runtime
    {
        let mut borrowed = data.borrow_mut();
        *borrowed += 1;
    }

    println!("Data: {:?}", data.borrow());  // 6
}
```

### Combining Rc and RefCell
```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let shared_data = Rc::new(RefCell::new(vec![1, 2, 3]));

    let a = Rc::clone(&shared_data);
    let b = Rc::clone(&shared_data);

    a.borrow_mut().push(4);
    b.borrow_mut().push(5);

    println!("{:?}", shared_data.borrow());  // [1, 2, 3, 4, 5]
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

## Concurrency and Threads

### Creating Threads
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

    handle.join().unwrap();  // Wait for thread to finish
}
```

### Moving Data to Threads
```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    // move keyword transfers ownership to thread
    let handle = thread::spawn(move || {
        println!("Vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

### Message Passing with Channels
```rust
use std::sync::mpsc;  // Multiple producer, single consumer
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

### Multiple Producers
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for i in 0..3 {
        let tx_clone = tx.clone();
        thread::spawn(move || {
            tx_clone.send(format!("Message from thread {}", i)).unwrap();
        });
    }

    drop(tx);  // Close original sender

    for received in rx {
        println!("Got: {}", received);
    }
}
```

### Mutex for Shared State
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

### RwLock for Multiple Readers
```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);

    // Multiple readers allowed
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        println!("Readers: {} and {}", *r1, *r2);
    }

    // Only one writer
    {
        let mut w = lock.write().unwrap();
        *w += 1;
    }
}
```

---

## Async/Await

### Basic Async Function
```rust
async fn hello() {
    println!("Hello, async!");
}

// Async functions return a Future
// You need a runtime to execute them

#[tokio::main]  // Using tokio runtime
async fn main() {
    hello().await;  // .await executes the future
}
```

### Understanding async/await
```rust
use tokio::time::{sleep, Duration};

async fn fetch_data() -> String {
    sleep(Duration::from_secs(2)).await;
    String::from("Data fetched!")
}

async fn process_data() {
    println!("Starting to fetch data...");
    let data = fetch_data().await;  // Waits here, but doesn't block thread
    println!("Got: {}", data);
}

#[tokio::main]
async fn main() {
    process_data().await;
}
```

### Running Tasks Concurrently
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
    // Run concurrently
    tokio::join!(task1(), task2());

    // Or spawn as separate tasks
    let handle1 = tokio::spawn(task1());
    let handle2 = tokio::spawn(task2());

    handle1.await.unwrap();
    handle2.await.unwrap();
}
```

### Async Streams
```rust
use tokio_stream::StreamExt;

async fn process_stream() {
    let mut stream = tokio_stream::iter(vec![1, 2, 3, 4, 5]);

    while let Some(value) = stream.next().await {
        println!("Got: {}", value);
    }
}
```

---

## Unsafe Rust

### What You Can Do in Unsafe
1. Dereference raw pointers
2. Call unsafe functions
3. Access or modify mutable static variables
4. Implement unsafe traits
5. Access union fields

### Raw Pointers
```rust
fn main() {
    let mut num = 5;

    // Creating raw pointers is safe
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    // Dereferencing requires unsafe
    unsafe {
        println!("r1: {}", *r1);
        println!("r2: {}", *r2);
    }
}
```

### Unsafe Functions
```rust
unsafe fn dangerous() {
    println!("This is dangerous!");
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

### Mutable Static Variables
```rust
static mut COUNTER: u32 = 0;

fn add_to_counter(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_counter(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
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
        println!("Hello!");
    };
}

// Macro with argument
macro_rules! create_function {
    ($func_name:ident) => {
        fn $func_name() {
            println!("Function {:?} called", stringify!($func_name));
        }
    };
}

// Macro with multiple patterns
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

### Procedural Macros (Overview)
```rust
// Derive macro - adds implementations
#[derive(Debug, Clone)]
struct Point {
    x: i32,
    y: i32,
}

// Attribute macro - modifies item
// #[route(GET, "/")]
// fn index() {}

// Function-like macro
// let sql = sql!(SELECT * FROM users);
```

---

## FFI (Foreign Function Interface)

### Calling C from Rust
```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3: {}", abs(-3));
    }
}
```

### Exposing Rust to C
```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Called from C!");
}

// Compile with: cargo build --release
// Link from C: gcc -o main main.c -L./target/release -lmylib
```

### Using CString
```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn puts(s: *const c_char) -> i32;
}

fn main() {
    let c_string = CString::new("Hello from Rust!").unwrap();

    unsafe {
        puts(c_string.as_ptr());
    }
}
```

---

## Summary

### Rust's Key Principles
1. **Memory Safety** - No null pointers, no dangling references
2. **Zero-Cost Abstractions** - High-level code compiles to efficient machine code
3. **Fearless Concurrency** - Compiler prevents data races
4. **Ownership System** - Clear rules for memory management

### When to Use What

| Problem | Solution |
|---------|----------|
| Store single value on heap | `Box<T>` |
| Share data in single thread | `Rc<T>` |
| Share data across threads | `Arc<T>` |
| Interior mutability | `RefCell<T>` or `Cell<T>` |
| Thread-safe mutability | `Mutex<T>` or `RwLock<T>` |
| Optional value | `Option<T>` |
| Fallible operation | `Result<T, E>` |

### Learning Path
1. **Basics**: Variables, functions, control flow
2. **Core Concepts**: Ownership, borrowing, lifetimes
3. **Intermediate**: Error handling, traits, generics
4. **Advanced**: Async, unsafe, macros

Good luck on your Rust journey!
