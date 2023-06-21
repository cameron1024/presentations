# Error handling in Rust

- Different types of errors
- When to panic
- When to `.unwrap()`
- Libraries and best practices

---

# Types of error

There are 2 types of error

## Expected errors

If this happens, it's not your fault
 - network erorrs
 - failed to parse json
 - invalid config file

## Unexpected errors

If this happens, it's your fault:
 - you thought an `Option` would be `Some` at this point, but it was `None`
 - you expected that a `Vec` would have elements, but it didn't
 - you expected that a config file would be correctly formatted

---

# When is it OK to panic?

---

# When is it OK to panic?

It's not

**Well-behaved programs do not panic.**

---

# So why does `.unwrap()` exist if it's never OK to panic?

Because sometimes you are **certain** that a `Result` is `Ok` (or an `Option` is `Some`).

So certain that, if it ever is `Err` or `None`, you'd consider it a bug.

There are many ways to write buggy code in Rust. `.unwrap()` is just one of many.

`.unwrap()` is how you tell the compiler: "I promise this is fine"

## Pet peeve about `.expect()`

Some people use `.expect("reason why this is fine")` instead.

This is better **if and only if** the message actually conveys some extra information:

 - GOOD - `some_fn().expect("at this point, init() has been called, so it is safe to call some_fn()")`
 - BAD - `some_fn().expect("some_fn() should be safe, panic otherwise")`

## Config file example

You should expect that the config file may be invalid.

Panicking if it is invalid is buggy behaviour, instead, propagate an error, and handle it in the caller

---

# `.unwrap()` is for developers and compilers

It requires that the error implements `Debug`, not `Display` - a user should never see it

For the compiler, to convert a `Result<T, E>` to a `T` requires handling the case where it is an error by either:
 - panicking
 - triggering UB (e.g. with `.unwrap_unchecked()`)
 - looping forever

Panicking is the "lesser evil"

---

# So should we ban `.unwrap()`? It sounds dangerous...

No

By the same logic, we'd have to ban **any other function that could cause us to write a buggy program**.

Panicking is always a bug, but we never write code with the assumption that it must be bug-free all the time (other than specialized circumstances).

Panicking is never good, but not so bad we should ban `.unwrap()`.

---

# So what should my error type look like?

It depends what you expect will happen with your error.

If your error is just going to be used to print a pretty error message, use a type-erased error type.

If your error is going to be inspected by the caller, use a structured error type (i.e. a regular struct or enum with an `Error` impl).

---

# Type-erased errors

An error type that isn't designed to be inspected. E.g.:

 - `Box<dyn std::error::Error>`
 - `anyhow::Error`
 - `color_eyre::Report`

I prefer `Report` because:
 - it is thread safe (i.e. `Send + Sync + 'static`)
 - it makes pretty error messages
 - allows adding lots of custom context (e.g. suggestions, extra info, etc)

`anyhow::Error` is similar (`eyre` is a fork of `anyhow`)

`Box<dyn Error>` is just a bit worse, but fundamentally similar (plus you probably want `Box<dyn Error + Send + Sync + 'static>` which is just awkward to type)

---

# Structured errors

You do **not** need a library

```rust
#[derive(Debug, Clone)]
enum Error {
    SomethingWentWrong {
        some_data: i32,
    },
    SomethingElse {
        user_id: String,
    }
}

impl std::fmt::Display for Error { /* ... */ }
impl std::error::Error for Error { /* ... */ }

impl From<Something> for Error { /* ... */ }
impl From<SomethingElse> for Error { /* ... */ }
```
This is a great error type:
 - it tells consumers of the API about the failure modes of the function
 - it allows consumers to handle each case
 - it provides the implmentations that an error type should have (`Debug`, `Clone`, `Display` and `Error` - you may also want `PartialEq`)
 - has `From` impls to allow it to work with `?`

---

# `String` is a bad error type

```rust
fn foo() -> Result<(), String> {
    if bar_error {
        return Err("bar was an error".into());
    }

    if baz_error {
        let baz_error_value: i32 = get_baz_error();
        return Err(format!("baz had an error, the value was {baz_error_value}"));
    }

    Ok(())
}
```
This function can fail in one of 2 ways, wow, that sounds like the perfect scenario for an enum:
```rust
enum Error {
    Bar,
    Baz(i32),
}
```
This is better because:
 - it is more precise
 - it avoids unnecessary heap allocations
 - it can be just as informative to users with a good `Display` impl

---

# What about `thiserror`?

`thiserror` is a macro that rewrites your enum to have the nice impls.

It does not appear in the API of your error type - it simply does the rewriting (so it can be an entirely crate-local implementation detail)

In high quality libraries, it is considered slightly bad practice to use it, since it adds compile-time overhead to all users of your crate:
 - proc macros usually depend on `syn` and `quote` - these are big crates that are slow to compile (`syn` is essentially a parser for the full Rust syntax)
 - proc macros have to be compiled (and executed) before other parts of the compiler, hurting parallelization significantly
 - widely used libraries (especially low level ones) should implement the error types by hand to avoid imposing this cost unnecessarily

This is not a concern for us - we are always going to be building with proc macros (thanks substrate), so the extra cost of `thiserror` is essentially zero, and it is very useful.
