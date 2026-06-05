# Rust Interview Preparation

A comprehensive repository for Rust interview preparation containing learning guides, interview questions, algorithms, and practical programs.

---

## Contents

### Learning Guides

| File | Description |
|------|-------------|
| [rust_complete_guide_english.md](rust_complete_guide_english.md) | Complete Rust learning guide from basics to advanced (English) |
| [rust_complete_guide_hindi.md](rust_complete_guide_hindi.md) | Complete Rust learning guide from basics to advanced (Hindi) |
| [Rust_basics.md](Rust_basics.md) | Rust fundamentals with examples (English) |
| [RustBasics_Hindi.md](RustBasics_Hindi.md) | Rust fundamentals with examples (Hindi) |

### Interview Questions

| File | Description |
|------|-------------|
| [rust_interview_questions.md](rust_interview_questions.md) | 48 frequently asked Rust interview questions (English) |
| [rust_interview_questions_hindi.md](rust_interview_questions_hindi.md) | 48 frequently asked Rust interview questions (Hindi) |
| [rust_advanced_interview_questions.md](rust_advanced_interview_questions.md) | Advanced Rust interview questions for senior positions (English) |
| [rust_advanced_interview_questions_hindi.md](rust_advanced_interview_questions_hindi.md) | Advanced Rust interview questions for senior positions (Hindi) |
| [real_interview_questions.md](real_interview_questions.md) | Questions actually asked in real interviews, tracked by date with detailed answers |

### Algorithms & Programs

| File | Description |
|------|-------------|
| [sorting_and_searching_algo.md](sorting_and_searching_algo.md) | Sorting and searching algorithms in Rust |
| [programs.md](programs.md) | Common programming problems and solutions |

---

## Topics Covered

### Basics
- Variables, Mutability, and Shadowing
- Data Types (Scalars, Compounds, Strings)
- Functions and Control Flow
- Arrays, Vectors, and Slices

### Core Concepts
- Ownership and Move Semantics
- Borrowing and References
- Lifetimes
- Structs and Enums
- Pattern Matching
- Modules and Packages

### Intermediate
- Error Handling (Result, Option)
- Generics
- Traits and Trait Objects
- Closures
- Iterators
- Collections (Vec, HashMap, HashSet)

### Advanced
- Smart Pointers (Box, Rc, Arc, RefCell)
- Concurrency (Threads, Mutex, Channels)
- Async/Await and Tokio
- Unsafe Rust
- Macros (Declarative and Procedural)
- FFI (Foreign Function Interface)
- Memory Layout and Optimization
- Design Patterns (Type State, Builder, Newtype)

---

## How to Use This Repository

1. **If you're new to Rust:**
   - Start with [rust_complete_guide_english.md](rust_complete_guide_english.md) or [rust_complete_guide_hindi.md](rust_complete_guide_hindi.md)
   - Practice with [programs.md](programs.md)

2. **If you're preparing for interviews:**
   - Review [rust_interview_questions.md](rust_interview_questions.md) for common questions
   - Study [rust_advanced_interview_questions.md](rust_advanced_interview_questions.md) for senior positions
   - Practice algorithms from [sorting_and_searching_algo.md](sorting_and_searching_algo.md)

3. **If you prefer Hindi:**
   - Use the `_hindi.md` versions of all files

---

## Quick Reference

### Key Rust Concepts for Interviews

```
Ownership Rules:
1. Each value has exactly one owner
2. Only one owner at a time
3. Value is dropped when owner goes out of scope

Borrowing Rules:
1. One mutable reference OR any number of immutable references
2. References must always be valid

Smart Pointer Usage:
- Box<T>      → Heap allocation, single owner
- Rc<T>       → Multiple owners, single thread
- Arc<T>      → Multiple owners, multi-thread
- RefCell<T>  → Interior mutability, runtime borrow check
```

---

## Contributing

Feel free to add more questions, algorithms, or improve existing content. All contributions are welcome!

---

## Resources

- [The Rust Programming Language Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://github.com/rust-lang/rustlings)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/)

---

Good luck with your Rust interviews!
