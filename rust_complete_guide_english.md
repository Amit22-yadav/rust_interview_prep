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

Rust is a **systems programming language** designed to provide memory safety, concurrency, and performance. Unlike languages like Python or JavaScript that use garbage collection, or C/C++ that give you manual memory control (with all its dangers), Rust uses a unique **ownership system** that the compiler checks at compile time.

**The Three Pillars of Rust:**

1. **Memory Safety Without Garbage Collection**
   - In languages like Java or Go, a garbage collector periodically scans memory to free unused data. This causes unpredictable pauses.
   - In C/C++, you manually allocate and free memory, leading to bugs like use-after-free, double-free, and memory leaks.
   - Rust's ownership system tracks who "owns" each piece of data. When the owner goes out of scope, memory is automatically freed. No garbage collector needed, no manual memory management required.

2. **Zero-Cost Abstractions**
   - "Zero-cost" means you don't pay a runtime performance penalty for using high-level features.
   - When you use iterators, closures, or generics in Rust, the compiler optimizes them to be as fast as hand-written low-level code.
   - This is unlike languages where abstractions add overhead (virtual function calls, boxing, etc.).

3. **Fearless Concurrency**
   - Concurrent programming is notoriously difficult because of data races—when two threads access the same data and at least one modifies it.
   - Rust's type system prevents data races at compile time. If your code compiles, it's free from data races.
   - This means you can write multi-threaded code confidently.

### Why Learn Rust?

**1. Memory Safety Without Runtime Cost**
Traditional memory bugs cause 70% of security vulnerabilities in systems software (according to Microsoft and Google). Rust eliminates:
- **Null pointer dereferences**: Rust has no null. Instead, it uses `Option<T>` which forces you to handle the "no value" case.
- **Buffer overflows**: Array bounds are checked, preventing out-of-bounds access.
- **Use-after-free**: The ownership system ensures data isn't accessed after being freed.
- **Data races**: The type system prevents concurrent mutable access.

**2. Performance**
Rust compiles to native machine code, just like C/C++. Benchmarks show Rust performing comparably to C in most scenarios. It achieves this through:
- No garbage collector pauses
- No runtime overhead
- Compile-time optimizations
- Direct memory access when needed

**3. Modern Developer Experience**
- **Cargo**: Package manager and build system that handles dependencies, building, testing, and documentation
- **rustfmt**: Automatic code formatter ensuring consistent style
- **clippy**: Linter that catches common mistakes and suggests improvements
- **Excellent error messages**: Rust's compiler errors are famously helpful, often suggesting exactly how to fix issues

**4. Growing Industry Adoption**
- **Mozilla**: Created Rust; uses it in Firefox (Servo browser engine)
- **Microsoft**: Rewriting Windows components in Rust for security
- **Google**: Using Rust in Android and Chromium
- **Amazon**: AWS services like Firecracker (powers Lambda and Fargate)
- **Discord**: Rewrote their Read States service from Go to Rust for performance
- **Cloudflare**: Uses Rust for performance-critical edge computing
- **Linux Kernel**: Rust is now an officially supported language for kernel development

### Hello World

```rust
fn main() {
    println!("Hello, World!");
}
```

**Deep Explanation:**

- **`fn main()`**: This is the entry point of every Rust program. When you run a Rust executable, execution begins at `main()`. The `fn` keyword declares a function. Unlike some languages, `main` doesn't return a value by default (though it can return `Result` for error handling).

- **`println!`**: Notice the exclamation mark `!`. This indicates `println!` is a **macro**, not a function. Why a macro?
  - Macros can accept a variable number of arguments: `println!("{} + {} = {}", 1, 2, 3)`
  - They perform compile-time string formatting, which is more efficient
  - They can do things functions can't, like accessing the file/line number for debugging

- **`"Hello, World!"`**: This is a **string literal** with type `&str` (a string slice). String literals are embedded directly in the compiled binary and are immutable.

- **Semicolons `;`**: Rust is an **expression-based language**. Statements end with semicolons. Removing the semicolon from the last line in a block makes it an expression that returns a value (more on this later).

---

## Variables and Mutability

### Why Immutability by Default?

In Rust, variables are **immutable by default**. This is a deliberate design choice for several reasons:

1. **Prevents Accidental Mutation**: In large codebases, it's easy to accidentally modify a variable you didn't intend to. Immutability by default catches this at compile time.

2. **Easier to Reason About**: When you see `let x = 5`, you know `x` will always be 5 in its scope. You don't need to trace through code to see if it changed.

3. **Thread Safety**: Immutable data can be safely shared between threads without synchronization. If data can't change, there can't be data races.

4. **Compiler Optimizations**: The compiler can make more aggressive optimizations when it knows a value won't change.

```rust
fn main() {
    let x = 5;
    // x = 6;  // Error! Cannot assign twice to immutable variable
    println!("x = {}", x);
}
```

**What happens when you try to mutate an immutable variable?**
The compiler gives you a clear error:
```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:3:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     x = 6;
  |     ^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut x = 5;
  |         +++
```

### Making Variables Mutable

When you need to change a value, explicitly mark it with `mut`:

```rust
fn main() {
    let mut x = 5;
    println!("x = {}", x);  // x = 5

    x = 6;  // This is allowed now
    println!("x = {}", x);  // x = 6
}
```

**The `mut` keyword communicates intent**: When you see `let mut x`, you immediately know this variable will be modified somewhere in the code. This makes code self-documenting.

**Important distinction**: `mut` allows you to change the **value** stored in the variable, but not its **type**. You can change `5` to `6`, but you can't change an integer to a string.

### Constants

Constants differ from immutable variables in fundamental ways:

```rust
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159;
const APP_NAME: &str = "My Application";

fn main() {
    println!("Max points: {}", MAX_POINTS);
}
```

**Key Differences Between `const` and `let`:**

| Feature | `const` | `let` |
|---------|---------|-------|
| Type annotation | Required | Optional (inferred) |
| Scope | Can be global | Only within functions/blocks |
| Evaluation time | Must be known at compile time | Can be computed at runtime |
| Mutability | Never mutable | Can use `mut` |
| Memory | Inlined at each use site | Stored in a memory location |

**When to use `const`:**
- Configuration values that never change: `const MAX_CONNECTIONS: u32 = 100;`
- Mathematical constants: `const PI: f64 = 3.14159265358979;`
- Values that need to be available at compile time

**The underscore in numbers (`100_000`)**: Rust allows underscores in numeric literals for readability. `100_000` is the same as `100000`, but easier to read. You can place underscores anywhere: `1_000_000`, `0xFF_FF_FF`, `0b1111_0000`.

### Shadowing: Reusing Variable Names

Shadowing lets you declare a new variable with the same name as a previous one:

```rust
fn main() {
    let x = 5;         // x is 5
    let x = x + 1;     // NEW variable x is 6 (previous x is shadowed)
    let x = x * 2;     // NEW variable x is 12

    println!("x = {}", x);  // x = 12
}
```

**How Shadowing Works:**
Each `let x` creates a **completely new variable** that happens to have the same name. The previous variable still exists in memory but is no longer accessible by that name—it's "shadowed."

**Shadowing vs Mutation—Key Differences:**

```rust
// Shadowing - creates new variable, can change type
let spaces = "   ";        // &str type
let spaces = spaces.len(); // usize type - this is a NEW variable!

// Mutation - modifies existing variable, same type required
let mut spaces = "   ";
// spaces = spaces.len();  // ERROR! Can't change type with mut
```

**Why Shadowing is Useful:**

1. **Type Transformation**: Convert data while keeping a meaningful name
   ```rust
   let user_input = "42";           // String from user
   let user_input: i32 = user_input.parse().unwrap();  // Now it's an integer
   ```

2. **Refining Data**: Process data in steps without needing new names
   ```rust
   let data = get_raw_data();       // Raw bytes
   let data = parse_data(data);     // Parsed structure
   let data = validate_data(data);  // Validated structure
   ```

3. **Scoped Shadowing**: Shadow within inner scopes
   ```rust
   let x = 5;
   {
       let x = x * 2;  // x is 10 inside this block
       println!("Inner x: {}", x);  // 10
   }
   println!("Outer x: {}", x);  // 5 - original x unchanged!
   ```

---

## Data Types

### Understanding Rust's Type System

Rust is a **statically typed** language, meaning the compiler must know the type of every variable at compile time. This enables:

1. **Compile-time error detection**: Type mismatches are caught before your code runs
2. **Performance**: No runtime type checking overhead
3. **Better tooling**: IDEs can provide accurate autocomplete and error highlighting
4. **Self-documenting code**: Types serve as documentation

Rust often **infers** types from context, so you don't always need explicit annotations:
```rust
let x = 5;        // Compiler infers i32
let y = 5.0;      // Compiler infers f64
let z = "hello";  // Compiler infers &str
```

### Scalar Types

Scalar types represent a single value. Rust has four primary scalar types.

#### 1. Integers

**Understanding Integer Types:**

| Length | Signed | Unsigned | Range (Signed) | Range (Unsigned) |
|--------|--------|----------|----------------|------------------|
| 8-bit | `i8` | `u8` | -128 to 127 | 0 to 255 |
| 16-bit | `i16` | `u16` | -32,768 to 32,767 | 0 to 65,535 |
| 32-bit | `i32` | `u32` | -2.1 billion to 2.1 billion | 0 to 4.2 billion |
| 64-bit | `i64` | `u64` | ±9.2 quintillion | 0 to 18.4 quintillion |
| 128-bit | `i128` | `u128` | Enormous range | Enormous range |
| arch | `isize` | `usize` | Depends on architecture | Depends on architecture |

**Signed vs Unsigned:**
- **Signed (`i`)**: Can store negative and positive numbers. Uses one bit for the sign.
- **Unsigned (`u`)**: Only positive numbers. All bits used for value, so double the positive range.

**Architecture-dependent types (`isize`/`usize`):**
- On 32-bit systems: same as i32/u32
- On 64-bit systems: same as i64/u64
- `usize` is used for indexing collections (array indices must be non-negative)

```rust
fn main() {
    let a: i32 = -42;     // Signed: can be negative
    let b: u32 = 42;      // Unsigned: only positive (or zero)
    let c = 98_222;       // Underscores for readability (same as 98222)
    let d = 0xff;         // Hexadecimal: 255 in decimal
    let e = 0o77;         // Octal: 63 in decimal
    let f = 0b1111_0000;  // Binary: 240 in decimal
    let g = b'A';         // Byte literal: ASCII value of 'A' (65), type is u8
}
```

**Integer Overflow:**
In debug mode, Rust panics on integer overflow. In release mode, it wraps around (e.g., 255u8 + 1 = 0). For explicit behavior:
```rust
let x: u8 = 255;
x.wrapping_add(1);   // Returns 0, wraps around
x.checked_add(1);    // Returns None (Option<u8>)
x.saturating_add(1); // Returns 255 (stays at max)
x.overflowing_add(1);// Returns (0, true) - value and overflow flag
```

#### 2. Floating-Point Numbers

Rust has two floating-point types following IEEE-754 standard:
- **`f64`**: 64-bit, double precision (default)
- **`f32`**: 32-bit, single precision

```rust
fn main() {
    let x = 2.0;      // f64 by default (more precision)
    let y: f32 = 3.0; // f32 (less memory, less precision)

    // Arithmetic operations
    let sum = 5.0 + 10.0;        // 15.0
    let difference = 95.5 - 4.3; // 91.2
    let product = 4.0 * 30.0;    // 120.0
    let quotient = 56.7 / 32.2;  // 1.76...
    let remainder = 43.0 % 5.0;  // 3.0 (modulo)
}
```

**Why f64 is the default:**
- Modern CPUs handle f64 nearly as fast as f32
- f64 has roughly 15 decimal digits of precision vs f32's 7
- Avoids subtle precision bugs in most calculations

**Floating-point pitfalls:**
```rust
// Floating-point numbers can't represent all decimals exactly
let x = 0.1 + 0.2;
println!("{}", x);  // 0.30000000000000004, not exactly 0.3!

// For financial calculations, use integer cents or a decimal library
```

#### 3. Boolean

The boolean type has exactly two values: `true` and `false`. It's one byte in size.

```rust
fn main() {
    let t = true;
    let f: bool = false;

    // Booleans are primarily used in conditionals
    if t {
        println!("It's true!");
    }

    // Boolean operations
    let and = true && false;  // false (both must be true)
    let or = true || false;   // true (at least one true)
    let not = !true;          // false (negation)
}
```

**Why booleans matter:** Rust requires conditions in `if`, `while`, etc. to be explicitly boolean. Unlike C or JavaScript, `if 1` or `if "hello"` won't work—you must use `if 1 != 0` or `if !string.is_empty()`.

#### 4. Character

Rust's `char` type represents a **Unicode Scalar Value**, not just ASCII.

```rust
fn main() {
    let c = 'z';
    let heart = '❤';
    let emoji = '😀';
    let chinese = '中';

    // char is always 4 bytes (32 bits) to hold any Unicode character
    println!("Size of char: {} bytes", std::mem::size_of::<char>()); // 4
}
```

**Important distinctions:**
- `char` uses single quotes: `'a'`
- String/str uses double quotes: `"a"`
- A `char` is NOT the same as a byte—it's a Unicode code point
- Some visual "characters" (like some emojis) are actually multiple Unicode scalars

### Compound Types

Compound types group multiple values into one type.

#### 1. Tuples

Tuples group values of **different types** with a **fixed length**.

```rust
fn main() {
    // Tuple with three different types
    let tup: (i32, f64, u8) = (500, 6.4, 1);

    // Destructuring: extract all values at once
    let (x, y, z) = tup;
    println!("y = {}", y);  // 6.4

    // Access by index (zero-based)
    let five_hundred = tup.0;    // First element
    let six_point_four = tup.1;  // Second element
    let one = tup.2;             // Third element
}
```

**When to use tuples:**
- Returning multiple values from a function
- Grouping related values temporarily
- When you need a quick, lightweight structure

```rust
// Function returning multiple values
fn get_dimensions() -> (u32, u32) {
    (1920, 1080)
}

let (width, height) = get_dimensions();
```

**Unit type `()`:** An empty tuple. Used when a function returns nothing meaningful. It's Rust's equivalent of `void` in C.

#### 2. Arrays

Arrays group values of the **same type** with a **fixed length known at compile time**.

```rust
fn main() {
    // Type annotation: [element_type; length]
    let arr: [i32; 5] = [1, 2, 3, 4, 5];

    // Initialize all elements with the same value
    let zeros = [0; 5];  // Creates [0, 0, 0, 0, 0]

    // Access elements by index
    let first = arr[0];   // 1
    let second = arr[1];  // 2

    // Length
    println!("Length: {}", arr.len());  // 5
}
```

**Arrays vs Vectors:**
- **Array `[T; N]`**: Fixed size, stored on **stack**, known at compile time
- **Vector `Vec<T>`**: Dynamic size, stored on **heap**, can grow/shrink

```rust
// Use arrays when:
let months: [&str; 12] = ["Jan", "Feb", "Mar", /* ... */]; // Size is fixed forever

// Use vectors when:
let mut numbers: Vec<i32> = Vec::new();
numbers.push(1);  // Can grow dynamically
```

**Bounds checking:** Rust checks array bounds at runtime. Accessing an invalid index causes a panic, not undefined behavior like in C.

### Strings: A Deep Dive

Strings in Rust can be confusing for newcomers because there are two main types.

#### `String` vs `&str` Explained

| Feature | `String` | `&str` |
|---------|----------|--------|
| **What is it** | Owned, growable string | Borrowed view into string data |
| **Storage** | Data on heap, metadata on stack | Just a pointer + length |
| **Ownership** | Owns its data | Borrows from somewhere else |
| **Mutability** | Can be `mut` and modified | Always immutable |
| **Size** | Dynamic (can grow) | Fixed (slice of existing data) |

**Think of it this way:**
- `String` is like owning a book—you can write in it, add pages, tear pages out
- `&str` is like borrowing a book—you can read it, but can't modify it

```rust
fn main() {
    // String literal - type is &str
    // Stored directly in the compiled binary, always immutable
    let s1: &str = "Hello";

    // String - heap allocated, owned
    // Created at runtime, can be modified
    let s2: String = String::from("Hello");
    let s3: String = "Hello".to_string();  // Same result

    // Converting: String → &str (cheap, just borrows)
    let s4: &str = &s2;  // s4 borrows s2's data

    // Converting: &str → String (allocates new memory)
    let s5: String = s1.to_owned();  // Copies data to heap

    // Mutable String operations
    let mut s = String::from("Hello");
    s.push_str(", World!");  // Append a string slice
    s.push('!');             // Append a single character
    println!("{}", s);       // "Hello, World!!"
}
```

**Memory layout visualization:**
```
String s2 = String::from("Hello")

Stack:                    Heap:
+------------+           +---+---+---+---+---+
| ptr   -----+---------> | H | e | l | l | o |
| len: 5     |           +---+---+---+---+---+
| capacity: 5|
+------------+

&str s1 = "Hello" (string literal)

Stack:                    Binary (read-only memory):
+------------+           +---+---+---+---+---+
| ptr   -----+---------> | H | e | l | l | o |
| len: 5     |           +---+---+---+---+---+
+------------+
```

**When to use which:**
- **`&str`**: Function parameters (accepts both `&String` and `&str`)
- **`String`**: When you need to own or modify the string
- **Rule of thumb**: Accept `&str`, return `String`

---

## Functions

### Understanding Functions in Rust

Functions are fundamental building blocks in Rust. They encapsulate reusable logic and help organize code.

**Key characteristics:**
- Declared with the `fn` keyword
- Use **snake_case** for function names (convention enforced by compiler warnings)
- Parameter types **must** be declared (no type inference for parameters)
- The function body is enclosed in curly braces `{}`

```rust
fn main() {
    println!("Hello from main!");
    another_function();
}

fn another_function() {
    println!("Hello from another function!");
}
```

**Function declaration order doesn't matter**: Unlike C, you can call a function before it's defined in the source file. Rust resolves all function definitions before executing.

### Parameters (Arguments)

Parameters are special variables that are part of a function's signature. When a function has parameters, you **must** provide values (arguments) for them.

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

**Why explicit type annotations?**
1. **Clarity**: Makes the function's contract explicit
2. **Better error messages**: Compiler knows exactly what types are expected
3. **No ambiguity**: Rust doesn't guess what type you meant

**Multiple parameters** are separated by commas, and each needs its own type annotation:
```rust
fn describe_person(name: &str, age: u32, height: f64) {
    println!("{} is {} years old and {} cm tall", name, age, height);
}
```

### Return Values

Functions can return values to the code that calls them. The return type is declared after an arrow `->`.

```rust
fn main() {
    let result = add(5, 3);
    println!("5 + 3 = {}", result);  // 8

    let doubled = double(7);
    println!("7 * 2 = {}", doubled);  // 14
}

// Explicit return with 'return' keyword
fn add(x: i32, y: i32) -> i32 {
    return x + y;
}

// Implicit return (idiomatic Rust) - no semicolon on last expression
fn double(x: i32) -> i32 {
    x * 2  // This expression's value is returned
}
```

**The semicolon rule is critical:**
```rust
fn five() -> i32 {
    5      // Returns 5 (no semicolon = expression)
}

fn five_broken() -> i32 {
    5;     // ERROR! Semicolon makes it a statement that returns ()
}
```

**Early returns** are useful for guard clauses:
```rust
fn divide(a: f64, b: f64) -> f64 {
    if b == 0.0 {
        return 0.0;  // Early return
    }
    a / b  // Normal return (implicit)
}
```

### Expressions vs Statements: A Critical Distinction

This is one of Rust's most important concepts. Understanding it unlocks idiomatic Rust code.

**Statements**: Perform an action, return nothing (`()`)
**Expressions**: Evaluate to a value

```rust
fn main() {
    // STATEMENT: performs action, returns () (unit type)
    let x = 5;  // 'let' is a statement

    // You can't do this because let doesn't return a value:
    // let y = (let z = 5);  // ERROR!

    // EXPRESSION: evaluates to a value
    let y = {
        let x = 3;
        x + 1  // NO SEMICOLON - this is an expression returning 4
    };
    println!("y = {}", y);  // y = 4

    // The block above is an expression that evaluates to 4
}
```

**Everything can be an expression in Rust:**

```rust
// if is an expression
let number = if condition { 5 } else { 6 };

// match is an expression
let description = match x {
    1 => "one",
    2 => "two",
    _ => "other",
};

// Blocks are expressions
let result = {
    let a = 1;
    let b = 2;
    a + b  // Returns 3
};

// Function calls are expressions
let len = "hello".len();  // len() returns a value
```

**Why this matters:**
1. **Concise code**: No need for temporary variables
2. **Single expression of intent**: The value flows naturally
3. **Functional style**: Enables method chaining and functional patterns

```rust
// Without expression-based style
let mut result = 0;
if condition {
    result = 5;
} else {
    result = 6;
}

// With expression-based style (idiomatic)
let result = if condition { 5 } else { 6 };
```

---

## Control Flow

Control flow determines the order in which code executes. Rust provides several powerful constructs, and importantly, most of them are **expressions** that return values.

### if Expressions

The `if` expression allows you to branch code based on conditions.

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
}
```

**Key points about `if` in Rust:**

1. **Condition must be a `bool`**: Unlike C or JavaScript, Rust won't automatically convert numbers or other types to boolean.
   ```rust
   let number = 3;
   // if number { }     // ERROR! Expected bool, found integer
   if number != 0 { }   // Correct: explicit comparison
   ```

2. **`if` is an expression**: It returns a value, enabling concise code.
   ```rust
   let condition = true;
   let value = if condition { 5 } else { 6 };
   println!("value = {}", value);  // 5
   ```

3. **Both branches must return the same type**:
   ```rust
   // ERROR! Types don't match
   // let value = if condition { 5 } else { "six" };

   // Correct: both return i32
   let value = if condition { 5 } else { 6 };
   ```

4. **No parentheses required** around the condition (unlike C/Java).

### Loops

Rust provides three loop constructs, each with specific use cases.

#### `loop` - Infinite Loop with Control

`loop` creates an infinite loop that you control with `break` and `continue`.

```rust
fn main() {
    let mut counter = 0;

    // loop can return a value via break
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;  // Return value and exit loop
        }
    };

    println!("Result: {}", result);  // 20
}
```

**Why use `loop` over `while true`?**
- More explicit intent: "I want to loop forever until I break"
- Enables returning values from loops
- Compiler knows it's intentionally infinite

**Loop labels** for nested loops:
```rust
fn main() {
    let mut count = 0;
    'outer: loop {
        let mut remaining = 10;
        loop {
            if remaining == 0 {
                break;  // Breaks inner loop only
            }
            if count == 2 {
                break 'outer;  // Breaks outer loop
            }
            remaining -= 1;
        }
        count += 1;
    }
}
```

#### `while` - Conditional Loop

`while` loops while a condition is true. It's sugar for `loop { if !condition { break } ... }`.

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

**When to use `while`:**
- The number of iterations isn't known in advance
- You need to check a condition each iteration
- The condition might be false initially (zero iterations)

**`while` vs `loop`:**
- `while` can't return a value (always returns `()`)
- Use `while` when the condition is the natural exit point
- Use `loop` when you need more complex exit logic or return values

#### `for` - Iterating Over Collections

`for` is the most common loop in Rust. It's safe (no out-of-bounds) and idiomatic.

```rust
fn main() {
    let arr = [10, 20, 30, 40, 50];

    // Iterate over array elements
    for element in arr {
        println!("Value: {}", element);
    }

    // Range: 1..4 means 1, 2, 3 (exclusive end)
    for number in 1..4 {
        println!("{}", number);  // 1, 2, 3
    }

    // Range: 1..=4 means 1, 2, 3, 4 (inclusive end)
    for number in 1..=4 {
        println!("{}", number);  // 1, 2, 3, 4
    }

    // Reverse iteration
    for number in (1..4).rev() {
        println!("{}", number);  // 3, 2, 1
    }
}
```

**Why `for` is preferred:**
1. **Safety**: Can't accidentally go out of bounds
2. **Clarity**: The intent is clear—iterate over a sequence
3. **Performance**: Often optimized as well as manual indexing
4. **Ownership control**: Can choose to borrow or consume

```rust
let v = vec![1, 2, 3];

// Borrow elements (v still usable after)
for item in &v {
    println!("{}", item);
}

// Borrow mutably (can modify elements)
let mut v = vec![1, 2, 3];
for item in &mut v {
    *item *= 2;
}

// Take ownership (v consumed, no longer usable)
for item in v {
    println!("{}", item);
}
```

### Match Expression

`match` is Rust's powerful pattern matching construct. It's like `switch` in other languages but far more powerful.

```rust
fn main() {
    let number = 3;

    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 => println!("Three"),
        4 | 5 => println!("Four or Five"),  // OR pattern
        6..=10 => println!("Six to Ten"),   // Range pattern
        _ => println!("Something else"),    // Wildcard (default)
    }
}
```

**Key features of `match`:**

1. **Exhaustive**: Must cover all possible values
   ```rust
   // ERROR! Not all cases covered
   // match number { 1 => "one", 2 => "two" }

   // Correct: _ catches everything else
   match number { 1 => "one", 2 => "two", _ => "other" }
   ```

2. **Returns a value** (it's an expression):
   ```rust
   let description = match number {
       1 => "one",
       2 => "two",
       _ => "other",
   };
   ```

3. **Pattern guards** for additional conditions:
   ```rust
   match number {
       n if n < 0 => println!("Negative"),
       n if n > 100 => println!("Large"),
       _ => println!("Normal"),
   }
   ```

4. **Destructuring** complex types:
   ```rust
   let point = (3, 5);
   match point {
       (0, 0) => println!("Origin"),
       (x, 0) => println!("On x-axis at {}", x),
       (0, y) => println!("On y-axis at {}", y),
       (x, y) => println!("At ({}, {})", x, y),
   }
   ```

**`match` vs `if`:**
- Use `match` when comparing one value against multiple patterns
- Use `match` when you need to destructure data
- Use `if` for simple boolean conditions

---

## Comments

Comments are essential for documenting code. Rust has several comment styles, each with a specific purpose.

### Regular Comments

```rust
// Single-line comment - most common for inline explanations
let x = 5; // Can also appear at end of line

/*
 * Multi-line comment
 * Useful for longer explanations
 * or temporarily disabling code blocks
 */

/* Multi-line comments can also /* be nested */ unlike in C */
```

### Documentation Comments

Rust has first-class support for documentation that generates HTML docs.

```rust
/// Documentation for the FOLLOWING item (function, struct, etc.)
///
/// This supports **Markdown** formatting:
/// - Bullet points
/// - `code snippets`
/// - [Links](https://example.com)
///
/// # Examples
///
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
fn add(a: i32, b: i32) -> i32 {
    a + b
}

//! Documentation for the ENCLOSING item (usually at top of file/module)
//!
//! This is typically used at the start of lib.rs or mod.rs to describe
//! the entire module or crate.
```

**Documentation sections by convention:**
- `# Examples` - Code examples (these are actually run as tests!)
- `# Panics` - When the function might panic
- `# Errors` - What errors might be returned
- `# Safety` - For unsafe functions, what invariants must be upheld

**Generate documentation:**
```bash
cargo doc --open  # Builds docs and opens in browser
```

**Why documentation comments matter:**
1. `cargo doc` generates beautiful HTML documentation
2. Code examples in `///` comments are **compiled and tested** by `cargo test`
3. IDEs show documentation on hover
4. Published crates automatically get documentation on docs.rs

---

# Part 2: Core Concepts

## Ownership

Ownership is **Rust's most distinctive feature** and the foundation of its memory safety guarantees. Understanding ownership is crucial—it's what makes Rust, Rust.

### The Problem Ownership Solves

In traditional systems programming:
- **Manual memory management (C/C++)**: You allocate and free memory manually. Mistakes lead to:
  - **Use-after-free**: Accessing memory that's been freed
  - **Double free**: Freeing the same memory twice
  - **Memory leaks**: Forgetting to free memory
  - **Dangling pointers**: Pointers to freed memory

- **Garbage collection (Java, Go, Python)**: Runtime tracks memory and frees it automatically. Downsides:
  - Unpredictable pauses when GC runs
  - Runtime overhead
  - Less control over memory layout

**Rust's solution**: The ownership system—compile-time rules that guarantee memory safety with zero runtime cost.

### The Three Rules of Ownership

These rules are enforced by the compiler:

1. **Each value in Rust has exactly one owner** (a variable)
2. **There can only be one owner at a time**
3. **When the owner goes out of scope, the value is dropped (freed)**

```rust
fn main() {
    // Rule 1: s1 is the owner of this String
    let s1 = String::from("hello");

    // Rule 2: Ownership MOVES to s2. s1 is no longer valid.
    let s2 = s1;

    // println!("{}", s1);  // ERROR! s1 was moved, it no longer owns anything
    println!("{}", s2);     // OK: s2 is now the owner

}  // Rule 3: s2 goes out of scope, String is dropped (memory freed)
```

### Memory Layout: Understanding What Happens

To truly understand ownership, you need to understand memory:

```
STACK (fast, fixed-size, automatic)     HEAP (slower, dynamic, manual)
┌──────────────────┐                   ┌─────────────────────┐
│ s1:              │                   │                     │
│   ptr ───────────┼──────────────────►│ h │ e │ l │ l │ o │
│   len: 5         │                   │                     │
│   capacity: 5    │                   └─────────────────────┘
└──────────────────┘

When we do: let s2 = s1;

STACK                                   HEAP
┌──────────────────┐                   ┌─────────────────────┐
│ s1: (invalid)    │                   │                     │
│   ptr ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─►│ h │ e │ l │ l │ o │
│   len: 5         │                   │                     │
│   capacity: 5    │                   └─────────────────────┘
├──────────────────┤                            ▲
│ s2:              │                            │
│   ptr ───────────┼────────────────────────────┘
│   len: 5         │
│   capacity: 5    │
└──────────────────┘
```

**Why move instead of copy?** If both s1 and s2 were valid:
1. When s1's scope ends → memory is freed
2. When s2's scope ends → same memory freed again = **CRASH** (double-free)

By invalidating s1, Rust ensures only one variable can free the memory.

### Copy vs Move: Which Types Do What?

**Copy Types** implement the `Copy` trait. Assignment copies the bits:
- Integers: `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`
- Floating point: `f32`, `f64`
- Boolean: `bool`
- Character: `char`
- Tuples (if all elements are Copy): `(i32, i32)`, `(bool, f64)`
- Arrays (if element type is Copy): `[i32; 5]`
- References: `&T` (but not `&mut T`)

```rust
fn main() {
    let x = 5;        // i32 is Copy
    let y = x;        // x is COPIED, not moved
    println!("x = {}, y = {}", x, y);  // Both valid!
}
```

**Why are these Copy?** They're simple values stored entirely on the stack. Copying them is just copying a few bytes—cheap and fast.

**Move Types** don't implement `Copy`. Assignment moves ownership:
- `String`
- `Vec<T>`
- `Box<T>`
- Any struct/enum unless you derive `Copy`
- Types containing heap data

```rust
fn main() {
    let s1 = String::from("hello");  // String is NOT Copy
    let s2 = s1;                      // s1 is MOVED to s2
    // println!("{}", s1);            // ERROR! s1 is invalid
    println!("{}", s2);               // OK
}
```

**Why aren't these Copy?** They contain pointers to heap data. A bitwise copy would create two owners of the same heap data—dangerous!

### Clone: Explicit Deep Copy

When you need a true copy of heap data, use `clone()`:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // Creates a complete copy on the heap

    println!("s1 = {}, s2 = {}", s1, s2);  // Both valid!
}
```

**Memory after clone:**
```
STACK                                   HEAP
┌──────────────────┐                   ┌─────────────────────┐
│ s1:              │                   │                     │
│   ptr ───────────┼──────────────────►│ h │ e │ l │ l │ o │
│   len: 5         │                   │                     │
│   capacity: 5    │                   └─────────────────────┘
├──────────────────┤
│ s2:              │                   ┌─────────────────────┐
│   ptr ───────────┼──────────────────►│ h │ e │ l │ l │ o │
│   len: 5         │                   │ (separate copy)     │
│   capacity: 5    │                   └─────────────────────┘
└──────────────────┘
```

**When to use `clone()`:**
- When you genuinely need two independent copies
- When ownership rules are inconvenient and performance isn't critical
- As a temporary fix while learning (but refactor later!)

**Warning**: `clone()` can be expensive for large data. Use it consciously.

### Ownership and Functions

Passing a value to a function transfers ownership, just like assignment:

```rust
fn main() {
    let s = String::from("hello");
    takes_ownership(s);       // s MOVED into function
    // println!("{}", s);     // ERROR! s is no longer valid

    let x = 5;
    makes_copy(x);            // x is COPIED (i32 implements Copy)
    println!("x = {}", x);    // OK! x is still valid
}

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}  // some_string goes out of scope, memory is freed

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}  // some_integer goes out of scope, nothing special (it was a copy)
```

### Returning Ownership

Functions can transfer ownership back to the caller:

```rust
fn main() {
    let s1 = gives_ownership();         // Function creates and returns String

    let s2 = String::from("hello");
    let s3 = takes_and_gives_back(s2);  // s2 moved in, new owner is s3

    // s2 is invalid (was moved)
    // s3 is valid (received ownership)
}

fn gives_ownership() -> String {
    let s = String::from("yours");
    s  // Ownership transferred to caller
}

fn takes_and_gives_back(s: String) -> String {
    s  // Takes ownership and gives it back
}
```

**The limitation**: This ownership dance is tedious. What if you want to use a value without giving up ownership? That's what **borrowing** solves (next section).

---

## Borrowing and References

Borrowing solves a fundamental problem: how do you let code use a value **without transferring ownership**? The answer is **references**.

### The Motivation for Borrowing

Without borrowing, this pattern is tedious:

```rust
fn main() {
    let s = String::from("hello");
    let (s, len) = calculate_length(s);  // Have to return s back!
    println!("Length of '{}' is {}", s, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)  // Return s so caller can use it again
}
```

**With borrowing**, we can "lend" the value temporarily:

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);  // Pass a REFERENCE, not ownership
    println!("Length of '{}' is {}", s1, len);  // s1 still valid!
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s (the reference) goes out of scope, but doesn't drop the data
```

### Immutable References (`&`)

An immutable reference lets you read data without owning it:

```rust
fn main() {
    let s = String::from("hello");

    let r1 = &s;  // r1 is a reference to s
    let r2 = &s;  // Multiple immutable references are OK

    println!("{} and {}", r1, r2);  // Can read through references
    println!("{}", s);               // Original still valid too
}
```

**What `&` means:**
- `&s` creates a reference to `s` (borrows `s`)
- `&String` is the type "reference to String"
- The reference points to `s` but doesn't own it

**Memory visualization:**
```
STACK                                   HEAP
┌──────────────────┐                   ┌─────────────────────┐
│ s:               │                   │                     │
│   ptr ───────────┼──────────────────►│ h │ e │ l │ l │ o │
│   len: 5         │                   │                     │
│   capacity: 5    │                   └─────────────────────┘
├──────────────────┤                            ▲
│ r1: ─────────────┼────────────────────────────┤ (points to same data)
├──────────────────┤                            │
│ r2: ─────────────┼────────────────────────────┘
└──────────────────┘
```

### Mutable References (`&mut`)

A mutable reference lets you modify borrowed data:

```rust
fn main() {
    let mut s = String::from("hello");  // s must be declared mut

    change(&mut s);  // Pass a mutable reference

    println!("{}", s);  // "hello, world"
}

fn change(s: &mut String) {
    s.push_str(", world");  // Can modify through &mut
}
```

**Requirements for `&mut`:**
1. The variable must be declared `mut`
2. You pass `&mut variable` to create the mutable reference
3. The function parameter type is `&mut Type`

### The Two Borrowing Rules

These rules prevent data races at compile time:

**Rule 1: You can have EITHER:**
- **Any number of immutable references (`&T`)**, OR
- **Exactly one mutable reference (`&mut T`)**

**But never both at the same time!**

```rust
fn main() {
    let mut s = String::from("hello");

    // ✓ Multiple immutable references (readers) - OK
    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used after this point

    // ✓ One mutable reference (writer) - OK (r1, r2 are done)
    let r3 = &mut s;
    println!("{}", r3);
}

// ✗ This will NOT compile:
fn broken() {
    let mut s = String::from("hello");

    let r1 = &s;      // Immutable borrow
    let r2 = &mut s;  // ERROR! Can't borrow as mutable while immutable borrow exists

    println!("{}", r1);  // r1 is still "alive" here
}
```

**Why this rule?**
- Multiple readers = safe (no one is changing data)
- One writer = safe (exclusive access)
- Readers + writer = **DATA RACE** (reader might see partially modified data)

**Rule 2: References must always be valid**

The compiler tracks lifetimes to ensure you never have a dangling reference.

### Non-Lexical Lifetimes (NLL)

Rust is smart about when references are "alive":

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // r1 and r2's last use is here - they're now "dead"

    // This works because r1 and r2 are no longer used
    let r3 = &mut s;
    println!("{}", r3);
}
```

The references `r1` and `r2` end at their last use, not at the end of the scope. This is called **Non-Lexical Lifetimes (NLL)**.

### Dangling References - Prevented by Rust

A dangling reference points to memory that's been freed. Rust prevents this:

```rust
// This will NOT compile:
fn dangle() -> &String {
    let s = String::from("hello");
    &s  // ERROR! s will be dropped, reference would be invalid
}

// Error message:
// this function's return type contains a borrowed value, but there is
// no value for it to be borrowed from
```

**The fix**: Return an owned value instead:

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // Move ownership to caller
}
```

### Reference Rules Summary

| Feature | `&T` (immutable ref) | `&mut T` (mutable ref) |
|---------|---------------------|----------------------|
| Can read data | ✓ | ✓ |
| Can modify data | ✗ | ✓ |
| Multiple at same time | ✓ (unlimited) | ✗ (only one) |
| Can coexist with `&T` | ✓ | ✗ |
| Prevents | Nothing | Prevents `&T` and other `&mut T` |

---

## Slices

Slices are **views into contiguous sequences** of data. They don't own data—they're references to a portion of a collection.

### What is a Slice?

A slice is a "fat pointer" containing:
1. A pointer to the first element
2. A length (number of elements)

```
String s = "hello world"

STACK                    HEAP
┌──────────────────┐    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ s: ptr ──────────┼───►│ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
│    len: 11       │    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
│    capacity: 11  │          ▲                   ▲
└──────────────────┘          │                   │
                              │                   │
Slice: &s[0..5] "hello"       │                   │
┌──────────────────┐          │                   │
│ ptr ─────────────┼──────────┘                   │
│ len: 5           │                              │
└──────────────────┘                              │
                                                  │
Slice: &s[6..11] "world"                          │
┌──────────────────┐                              │
│ ptr ─────────────┼──────────────────────────────┘
│ len: 5           │
└──────────────────┘
```

### String Slices (`&str`)

The most common slice type. String literals are actually slices!

```rust
fn main() {
    let s = String::from("hello world");

    // Range syntax: start..end (end is exclusive)
    let hello = &s[0..5];   // "hello" - characters 0,1,2,3,4
    let world = &s[6..11];  // "world" - characters 6,7,8,9,10

    // Shorthand syntax
    let hello = &s[..5];    // Same as &s[0..5] - start from beginning
    let world = &s[6..];    // Same as &s[6..11] - go to end
    let whole = &s[..];     // Same as &s[0..11] - entire string

    println!("{} {}", hello, world);  // "hello world"
}
```

**Important**: String slicing works on **byte indices**, not character indices. For ASCII this is fine, but multi-byte UTF-8 characters require care:

```rust
let s = "नमस्ते";  // Hindi "Namaste"
// let slice = &s[0..1];  // ERROR! Would split a UTF-8 character
let slice = &s[0..3];     // OK: "न" is 3 bytes
```

### Why Slices Make Code Safe

Slices tie together a reference and a length, preventing common bugs:

```rust
// Without slices (unsafe in C, impossible in Rust)
fn first_word_bad(s: &String) -> usize {
    // Returns index of first space
    // Problem: What if s changes? Index becomes invalid!
    for (i, &byte) in s.as_bytes().iter().enumerate() {
        if byte == b' ' {
            return i;
        }
    }
    s.len()
}

// With slices (safe!)
fn first_word(s: &str) -> &str {
    for (i, &byte) in s.as_bytes().iter().enumerate() {
        if byte == b' ' {
            return &s[0..i];
        }
    }
    &s[..]  // No space found, return whole string
}

fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);  // word borrows s

    // s.clear();  // ERROR! Can't mutate s while word borrows it

    println!("First word: {}", word);
}  // word goes out of scope, now s could be mutated
```

The slice **borrows** from the original data, so the compiler prevents you from modifying the data while the slice exists.

### Array Slices (`&[T]`)

Slices work for any contiguous collection, not just strings:

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];

    let slice: &[i32] = &arr[1..3];  // [2, 3]
    println!("{:?}", slice);

    // Slices work with vectors too
    let vec = vec![10, 20, 30, 40, 50];
    let vec_slice: &[i32] = &vec[2..4];  // [30, 40]

    // Functions that take slices are very flexible
    print_slice(&arr[..]);      // Pass entire array
    print_slice(&arr[1..4]);    // Pass portion of array
    print_slice(&vec[..]);      // Pass entire vector
}

fn print_slice(slice: &[i32]) {
    println!("Slice has {} elements:", slice.len());
    for item in slice {
        println!("  {}", item);
    }
}
```

### `&str` vs `&String`: Why `&str` is Preferred

Functions should accept `&str` instead of `&String`:

```rust
// Good: accepts &str
fn process(s: &str) {
    println!("{}", s);
}

// Less flexible: only accepts &String
fn process_string(s: &String) {
    println!("{}", s);
}

fn main() {
    let owned = String::from("hello");
    let literal = "world";

    // &str is more flexible
    process(&owned);    // &String coerces to &str
    process(literal);   // &str works directly
    process(&owned[0..3]);  // Slice works too

    // &String is restrictive
    process_string(&owned);  // Works
    // process_string(literal);     // ERROR! &str is not &String
    // process_string(&owned[0..3]); // ERROR! &str is not &String
}
```

**Rule of thumb**: Accept `&str` in function parameters. It accepts both owned strings (via coercion) and string slices.

---

## Structs

Structs let you create **custom data types** by grouping related values together. They're similar to classes in other languages but without inheritance.

### Why Use Structs?

Structs provide:
1. **Named fields** - Clear, self-documenting code
2. **Type safety** - Compiler ensures you use the right data
3. **Methods** - Functions tied to your data type
4. **Encapsulation** - Group related data and behavior

### Defining and Instantiating Structs

```rust
// Define a struct with named fields
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    // Create an instance - must provide all fields
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("user123"),
        active: true,
        sign_in_count: 1,
    };

    // Access fields with dot notation
    println!("Email: {}", user1.email);

    // Entire struct must be mutable to modify any field
    let mut user2 = User {
        email: String::from("another@example.com"),
        username: String::from("another123"),
        active: true,
        sign_in_count: 1,
    };
    user2.email = String::from("newemail@example.com");
}
```

**Note on mutability**: Rust doesn't allow marking individual fields as mutable. The entire instance is either mutable or not.

### Field Init Shorthand

When variable names match field names:

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,           // Same as email: email
        username,        // Same as username: username
        active: true,
        sign_in_count: 1,
    }
}
```

### Struct Update Syntax

Create a new struct using values from another:

```rust
fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("user123"),
        active: true,
        sign_in_count: 1,
    };

    // Create user2 with different email, rest from user1
    let user2 = User {
        email: String::from("different@example.com"),
        ..user1  // Fill remaining fields from user1
    };

    // WARNING: user1.username was MOVED to user2!
    // println!("{}", user1.username);  // ERROR! username was moved
    println!("{}", user1.active);        // OK! bool is Copy
}
```

**Important**: `..user1` **moves** non-Copy fields. After this, `user1.username` is invalid (but `active` and `sign_in_count` are fine because they're Copy types).

### Tuple Structs

Named tuples—useful when field names would be redundant:

```rust
struct Color(i32, i32, i32);   // RGB values
struct Point(i32, i32, i32);   // XYZ coordinates

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);

    // Access by index
    println!("Red component: {}", black.0);
    println!("X coordinate: {}", origin.0);

    // Even though both have (i32, i32, i32), they're different types!
    // let p: Point = black;  // ERROR! Color is not Point
}
```

**Why tuple structs?** They create distinct types. A `Color(255, 0, 0)` is not interchangeable with a `Point(255, 0, 0)` even though both contain three `i32` values.

### Unit-Like Structs

Structs with no fields—useful for implementing traits:

```rust
struct AlwaysEqual;

fn main() {
    let _subject = AlwaysEqual;
}

// Useful for marker types or implementing traits
impl SomeTrait for AlwaysEqual {
    // ...
}
```

### Methods: Functions Attached to Structs

The `impl` block defines methods for a struct:

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Method: first parameter is always some form of self
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // Method that takes ownership (rare but possible)
    fn destroy(self) {
        println!("Rectangle destroyed!");
        // self is dropped after this
    }

    // Method with mutable access
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }

    // Method with additional parameters
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }

    // Associated function (no self) - like static method
    // Called with Rectangle::square(10), not rect.square(10)
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let mut rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 10, height: 40 };

    println!("Area: {}", rect1.area());             // Method call
    println!("Can hold rect2: {}", rect1.can_hold(&rect2));

    rect1.double();  // Mutable method
    println!("Doubled area: {}", rect1.area());

    let square = Rectangle::square(20);  // Associated function call
}
```

**Understanding `self`:**
- `&self` - Immutable borrow (read only)
- `&mut self` - Mutable borrow (can modify)
- `self` - Takes ownership (consumes the struct)

**Automatic referencing**: Rust automatically adds `&`, `&mut`, or `*` when calling methods. `rect.area()` is equivalent to `(&rect).area()`.

### Multiple `impl` Blocks

You can split implementation across multiple blocks:

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

impl Rectangle {
    fn perimeter(&self) -> u32 {
        2 * (self.width + self.height)
    }
}
```

This is useful when implementing traits (covered later).

### Deriving Traits

The `#[derive]` attribute auto-implements common traits:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();  // Clone trait

    println!("{:?}", p1);     // Debug: Point { x: 1, y: 2 }
    println!("{:#?}", p1);    // Pretty Debug (multi-line)
    println!("Equal: {}", p1 == p2);  // PartialEq trait
}
```

**Common derivable traits:**
- `Debug` - Enable `{:?}` formatting
- `Clone` - Enable `.clone()` method
- `Copy` - Enable implicit copying (for simple types)
- `PartialEq` - Enable `==` and `!=` comparisons
- `Eq` - Marker for full equality (requires `PartialEq`)
- `PartialOrd` - Enable `<`, `>`, `<=`, `>=`
- `Ord` - Enable total ordering (requires `Eq`)
- `Hash` - Enable use as HashMap key
- `Default` - Enable `Type::default()` constructor

---

## Enums and Pattern Matching

Enums (enumerations) let you define a type by listing its possible **variants**. Combined with pattern matching, they're one of Rust's most powerful features.

### Understanding Enums

An enum represents "one of several possibilities":

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let dir = Direction::North;

    // Must handle all variants
    match dir {
        Direction::North => println!("Going north!"),
        Direction::South => println!("Going south!"),
        Direction::East => println!("Going east!"),
        Direction::West => println!("Going west!"),
    }
}
```

**Key insight**: A value of type `Direction` can only be **one** of these four variants. The compiler ensures you handle all possibilities.

### Enums with Associated Data

Unlike C enums, Rust enums can hold data:

```rust
enum Message {
    Quit,                         // No data
    Move { x: i32, y: i32 },     // Named fields (like struct)
    Write(String),                // Single value (like tuple struct)
    ChangeColor(i32, i32, i32),  // Multiple values (like tuple)
}

fn main() {
    // Each variant holds different data
    let m1 = Message::Quit;
    let m2 = Message::Move { x: 10, y: 20 };
    let m3 = Message::Write(String::from("hello"));
    let m4 = Message::ChangeColor(255, 0, 0);

    // Pattern matching extracts the data
    match m2 {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Write(text) => println!("Text: {}", text),
        Message::ChangeColor(r, g, b) => println!("RGB: {}, {}, {}", r, g, b),
    }
}
```

**Why this is powerful**: A single type can represent multiple shapes of data. A `Message` could be any of these forms, and pattern matching lets you handle each appropriately.

### Methods on Enums

Enums can have methods just like structs:

```rust
impl Message {
    fn call(&self) {
        match self {
            Message::Quit => println!("Quitting..."),
            Message::Move { x, y } => println!("Moving to ({}, {})", x, y),
            Message::Write(text) => println!("Writing: {}", text),
            Message::ChangeColor(r, g, b) => {
                println!("Changing color to RGB({}, {}, {})", r, g, b)
            }
        }
    }
}

fn main() {
    let msg = Message::Write(String::from("hello"));
    msg.call();  // "Writing: hello"
}
```

### The `Option<T>` Enum: Rust's Answer to Null

Rust has **no null**. Instead, it uses `Option<T>` to represent "maybe a value":

```rust
// This is built into the standard library:
enum Option<T> {
    Some(T),  // Contains a value of type T
    None,     // No value
}
```

**Why no null is better:**
- In languages with null, any variable could be null—you have to check everywhere
- In Rust, if a value might not exist, it MUST be `Option<T>`
- The compiler forces you to handle the None case

```rust
fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // MUST handle both cases - compiler enforces this
    match some_number {
        Some(n) => println!("Got number: {}", n),
        None => println!("No number"),
    }

    // You can't use Option<i32> as i32 directly
    // let sum = some_number + 5;  // ERROR! Can't add Option<i32> + i32

    // Must extract the value first
    let sum = some_number.unwrap() + 5;  // OK, but panics if None
}
```

### Common `Option` Methods

```rust
fn main() {
    let some: Option<i32> = Some(5);
    let none: Option<i32> = None;

    // unwrap: Get value or panic
    let x = some.unwrap();          // 5
    // let y = none.unwrap();       // PANIC!

    // expect: Like unwrap but with custom panic message
    let x = some.expect("should have value");

    // unwrap_or: Get value or use default
    let x = none.unwrap_or(0);      // 0

    // unwrap_or_else: Get value or compute default
    let x = none.unwrap_or_else(|| expensive_computation());

    // map: Transform the inner value if Some
    let doubled: Option<i32> = some.map(|n| n * 2);  // Some(10)
    let doubled: Option<i32> = none.map(|n| n * 2);  // None

    // and_then: Chain operations that return Option
    let result: Option<i32> = some
        .and_then(|n| if n > 0 { Some(n * 2) } else { None });

    // is_some / is_none: Check without consuming
    if some.is_some() {
        println!("Has value!");
    }
}
```

### `if let`: Concise Pattern Matching

When you only care about one pattern, use `if let`:

```rust
fn main() {
    let some_value: Option<i32> = Some(3);

    // Verbose match for single pattern
    match some_value {
        Some(value) => println!("Got: {}", value),
        None => {}  // Do nothing
    }

    // Concise if let
    if let Some(value) = some_value {
        println!("Got: {}", value);
    }

    // With else for the "otherwise" case
    if let Some(value) = some_value {
        println!("Got: {}", value);
    } else {
        println!("Nothing");
    }
}
```

**Use `if let` when:**
- You only care about one pattern
- You want to avoid the boilerplate of `match` with `_ => {}`

### `while let`: Loop While Pattern Matches

```rust
fn main() {
    let mut stack = vec![1, 2, 3];

    // Pop until empty
    while let Some(top) = stack.pop() {
        println!("Popped: {}", top);
    }
    // Output: 3, 2, 1

    println!("Stack is now empty");
}
```

### The `Result<T, E>` Enum: Error Handling

For operations that can fail, Rust uses `Result`:

```rust
enum Result<T, E> {
    Ok(T),   // Success with value of type T
    Err(E),  // Failure with error of type E
}

fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(result) => println!("Result: {}", result),
        Err(e) => println!("Error: {}", e),
    }
}
```

(Result is covered more in the Error Handling section.)

### Pattern Matching Deep Dive

Patterns are powerful. Here are more capabilities:

```rust
fn main() {
    let x = 5;

    match x {
        // Multiple patterns with |
        1 | 2 => println!("one or two"),

        // Range patterns
        3..=5 => println!("three through five"),

        // Guards: additional conditions
        n if n % 2 == 0 => println!("even"),

        // Catch-all (must be last)
        _ => println!("something else"),
    }

    // Destructuring tuples
    let point = (3, -5);
    match point {
        (0, 0) => println!("origin"),
        (x, 0) => println!("on x-axis at {}", x),
        (0, y) => println!("on y-axis at {}", y),
        (x, y) => println!("at ({}, {})", x, y),
    }

    // Ignoring values
    let numbers = (1, 2, 3, 4, 5);
    match numbers {
        (first, _, third, _, fifth) => {
            println!("first: {}, third: {}, fifth: {}", first, third, fifth)
        }
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
