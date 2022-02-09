# Why compile time evaluation?

This section is, in many ways, the most important. It aims to clarify the reasoning about any future decisions about const generics and const evaluation.

## Why do we want to evaluate things at compile time?

People already precompute certain data structures, lookup tables, and so on, at compile time.
For example the crate `regex` could be able to compile a regex at compile time,
improving the performance when using `regex` and even removing the need for runtime allocations.
Const evaluation is also very useful to assert correctness requirements at compile time, instead
of using a runtime panic.

Without good const evaluation support, is often easier to just use a `build.rs` file for this or compute it externally and generate the result outside of the ordinary compilation process. This is both worse for the user, as they essentially have to learn yet another
meta language, and also negatively impacts the compilation speed and incremental compilation. If that isn't possible or sensible, one often just requires the computation to happen at runtime, negatively impacting startup performance.

Computing things outside without using const eval makes it at lot more cumbersome to use and
prevents the input for that computation to simply be a generic or function parameter.

## Why const generics?

Const evaluation by itself is good enough unless it should influence the type system. 

This is needed when computing the layout of types, most often by using arrays.
Using arrays instead of dynamic allocations or slices can improve performance, reduces the dynamic memory footprint
and can remove runtime checks and implicit invariants.

Another reason is to be generic over configurations or to move certain conditions to compile time. Consider [image handles](https://github.com/EmbarkStudios/rust-gpu/blob/1431c18b9db70feafc64e5096a64e5fefffbed18/crates/spirv-std/src/image.rs#L31) in [`rust-gpu`](https://github.com/EmbarkStudios/rust-gpu) or TODO: example for state machines.

## What are our goals?

### Consistent: "just add `const`"

Writing code that works at compile time should at most require manually adding the needed `const` annotations at suggested places (or even automatically using `rustfix` or IDE support)
until everything compiles. One should not be required to avoid large parts of the language to do this.
Adding and using const parameters should be straight forward without weird restrictions or a lot required knowledge.
Using const generic items from different libraries should work without issues.

### Approachable: "what is `[u8; my_expr]:,` and why is it needed"

Reading or writing code which can be used at compile time should ideally not be more complex
than writing other code. Requiring a lot of weird trait bounds or other hacks should be avoided as much as possible.
Supporting compile time evaluation should not worsen the experience of users, especially for beginners.

### Reliable: "if it passes `cargo check`, it works"

Const evaluation failures should be detected as early as possible, even when using `cargo check`. When writing library code, incorrect constants
should cause warnings and errors whenever possible.