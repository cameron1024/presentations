# Rust Best Practices

 - How to set up a new Rust project
 - Common pitfalls
 - Tooling

Slides available at https://github.com/cameron1024/presentations

---

# Starting a new Rust project

```
cargo new my-cool-project
```
or
```
cargo new --lib my-cool-project
```
What's the difference?
 - `src/main.rs` vs. `src/lib.rs`
 - `Cargo.lock`

---

# Library/Binary split

Split your crate into a library and a binary (imagine crate name `foobar`)

```
├── Cargo.lock
├── Cargo.toml
└── src
    ├── bin
    │   └── my_binary.rs
    └── lib.rs
```
```rust
// src/lib.rs
pub fn do_the_thing() {
  println!("I'm doing the thing");
}

// src/bin/my_binary.rs
fn main() {
  foobar::do_the_thing();
}
```

---

# Advantages of this approach

 - improves code reusability
 - can be used as a dependency by other crates (even pushed to crates.io)
 - integration testing/doctests - only available for libraries
 - trains you to "think like a library author"

## Integration tests

Found in `<crate-root>/tests` (not `<crate-root>/src/tests`)

```rust
// tests/my_integration_test.rs
#[test]
fn it_shouldnt_panic() {
    foobar::do_the_thing();
    // crate::do_the_thing() wouldn't work here
}
```
Only have access to the crate's public API

---

# Doctests

Doctests are a special kind of integration test (needs `///` not `//` but slides doesn't render it 😢)

```rust
// lib.rs

// Doubles a number
//
// ```rust 
// # use my_crate::double;
// let x = double(123);
// assert_eq!(x, 246);
// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

 - runs on `cargo test`
 - rendered in the output of `cargo doc`
 - docs **cannot** get out of sync with code
 - easier to understand the API (particularly more complex APIs)
 - can run in different modes:
   - compile and run (default)
   - just typeck/borrowck
   - expect compile fail
   - ignore

Complex logic is fine - `diesel` runs an SQL database in doctests

---

# Cargo Workspaces

Workspaces allow you to group related crates

Shared `Cargo.lock`

```toml
# Cargo.toml

[workspace]
members = [
    "foo",
    "bar",
]

[workspace.dependencies]
tracing = "0.1"
tokio = "1"

# ----------------------
# foo/Cargo.toml
[dependencies]
tracing = { workspace = true }
tokio = { workspace = true }

# ----------------------
# bar/Cargo.toml
[dependencies]
tracing = { workspace = true }
```

---

# Tooling

 - rustc, cargo, rustup
 - clippy
 - trybuild/integration tests
 - proptest/kani
 - fun stuff (bacon, nextest, )

---

# clippy

Clippy is the de-facto linter

Behaviour can be customized with attributes in `lib.rs`

```rust
#![allow(clippy::explicit_deref_methods)]
#![warn(clippy::perf)]
#![deny(
    clippy::integer_arithmetic,
    clippy::disallowed_methods,
)]
```
Extra config goes in `clippy.toml`
```toml
disallowed-methods = [
    { path = "std::env::var", reason = "don't use environment variables" },
    { path = "std::collections::HashMap", reason = "use BTreeMap instead" },
]
```
Can be invoked by running `cargo clippy`, but rust-analyzer will pick it up (modulo editor/config)

---

# Trybuild

Sometimes you want to check that "this code doesn't compile"

Particularly useful when writing macros, but also generally useful

```rust
// <crate-root>/tests/ui/some_test.rs
// exact path doesn't matter
use my_cool_crate::NotThreadSafeDataStructure;

fn main() {
    let not_thread_safe = NotThreadSafeDataStructure::new();

    std::thread::spawn(move || {
        not_thread_safe.do_cool_thing();  
    });
}
```
Captures stderr in a file called `/path/to/test/file.stderr`

Can set `TRYBUILD=overwrite` to replace the old stderr snapshots

---

# Proptest

Property testing library

```rust
#[proptest]
fn some_test(
    arbitrary_string: String,
    #[strategy(0..100)] x: i32,
    #[strategy(100..200)] y: i32,
) {
    assert!(arbitrary_string.len() >= 0);
    assert!(x < y);
}
```
Types which implement `Strategy<Value = T>` can be used to generate values of type `T`

Types which implement `Arbitrary` have a "canonical strategy"

Note: this code snippet uses a helper library called `test_strategy` - working on getting this upstreamed

---

# Kani

Kani is a model-checker for Rust

On the surface, very similar to proptest

```rust
fn is_even(x: u64) -> bool {
    x % 2 == 0 
}

#[kani::proof]
fn x_is_even_or_x_plus_1_is_even() {
    let x: u64 = kani::any();
    assert!(is_even(x) || is_even(x + 1));
}
```

`kani::any()` is a "symbolic value" 

SAT solver for "is there any possible bit pattern that can cause this program to panic"

Not always the right tool for the job, useful for small, mission-critical logic

Struggles with big loops

---

# Fun stuff

## Bacon

CLI to run tests/clippy in the background

## Nextest

Better `cargo test`

 - Looks prettier
 - Better parallel scheduler
 - More features (e.g. sharding, automatic retry of flakes, dangling process detection)

---

# Simplicity

Rust is a powerful language, but you don't have to use all its features

The cleverer your code, the harder to understand (I hear this can be an issue in Scala 😅)

## Which is "better"

```rust
fn foo(s: &str) {
  println!("the string is {s}");
}
```
OR
```rust
fn foo<S: AsRef<str>(s: S) {
  let s = s.as_ref();
  println!("the string is {s}");
}
```

---

# Simplicity (cont.)

Do you really need...

 - a trait + struct that implements the trait?
 - a higher-order function?
 - generics (where dynamic dispatch would work)?
 - async?
 - lifetimes?
 - to define a macro?

Generics (including lifetimes) and async are also "contagious" - be extra careful with them

---

# Unsafe

You do not need unsafe - `#![deny(unsafe_code)]` in all crates

It is not "just writing C", it is much worse

## Undefined behaviour is Very Bad

```rust
fn always_returns_true(x: u8) -> bool {
    x < 120 || x == 120 || x > 120
}

fn main() {
    // create a `u8` from uninitialized memory
    let x: u8 = unsafe { MaybeUninit::uninit().assume_init() };
    assert!(always_returns_true(x));
}
```

---

# Unsafe (cont.)

## Undefined Behaviour can time travel

What is the output of this program?

```rust
fn main() {
    println!("this bit is safe, right...");

    unsafe { unreachable_unchecked() };

    println!("we never reach here");
}
```
---

# Keep module APIs small

Test at the module level

Module structure is entirely decoupled from file structure

Privacy is enforced at the module level

 - `pub(super)` makes something visible in the parent module
 - `pub(crate)` makes something potentially visible in the whole crate 

---

# You usually don't need to create a trait

```rust
trait FooCalculator {
    fn calculate_foo(&self, foo: Foo) -> Bar;
}

struct DefaultFooCalculator;

impl FooCalculator for DefaultFooCalculator {
    fn calculate_foo(&self, foo: Foo) -> Bar {
        todo!()
    }
}

struct HasAFooCalculator<C: FooCalculator> {
   calculator: C 
}
```

## Function pointer

```rust
struct HasAFooCalculator {
    calculator: fn(Foo) -> Bar,
}
```

## Function object

```rust
struct HasAFooCalculator {
    calculator: Box<dyn Fn(Foo) -> Bar>,
}
```

---

# Generics have a cost

Trait objects are "slower":

 - not always true
 - not always relevant

## Trait objects do not "infect" with generics

```rust
struct Foo 
    inner: Box<dyn Trait>,
}
```

## Faster to compile

No monomorphization - less work for LLVM (the slow bit)

## Generics interfere with type inference

```rust
fn foo(x: impl Into<String>) { /* ... */ }

fn main() {
    // what is the type of `x`? Can't even fix with `.into::<Type>()`
    let x = "hello".into();
    foo(x);
}
```

---

# Newtype pattern

Probably already seen this:
```rust
struct Email(String);

fn send_email(email: Email, message: String) { /* ... */ }
```
Avoids confusion, helps catch refactoring errors

Can be tedious, consider `microtype` crate (shameless self plug)

```rust
microtype! {
    #[derive(Debug, Clone, ...)]
    String {
        Email,
        Username,
        // ...
    },

    #[secret]
    String {
        Password,
    }
}
```
Generates:
 - `From`/`Into` impls
 - `Deref` impls (allowing `email.some_string_method()`)
 - `serde(transparent)`
 - `AsRef` impls


---

# Very quick error handling

## Panic when "it would be a bug if it happened"

When you know about invariants that the compiler doesn't

If possible, encode this invariant in the type system

Not always possible (or sometimes possible but very awkward)

## Use `color_eyre::eyre::Report` for "I don't care" errors

Errors that aren't meant to be handled

Just want a pretty message printed and then exit

`Result` is better than panic because it **gives the caller the choice**

## Use `thiserror` for errors you may want to handle

Highest quality, but highest effort

Act as documentation for "the possible error states of the function"

Any errors at a public API boundary that we care about

Doesn't appear in type signature

---

# Questions?
