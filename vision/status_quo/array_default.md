# 😱 Status quo story: Barbara wants to implement `Default` for all array lengths 

Barbara is working on `std`. She wants to implement `Default` for all array types. Currently, it is implemented for `N <= 32` using a macro ([link](https://doc.rust-lang.org/nightly/src/core/array/mod.rs.html#391))

She thinks "Ah, I can use min-const-generics for this!" and goes to write

```rust
impl<T, const N: usize> Default for [T; N]
where
    T: Default,
{
    fn default() -> Self {
        
    }

}
```

So far so good, but then she realizes she can't figure out what to write in the body. At first she tries:

```rust
impl<T, const N: usize> Default for [T; N]
where
    T: Default,
{
    fn default() -> Self {
        [T::default(); N]
    }

}
```

but this won't compile:

```
error[E0277]: the trait bound `T: Copy` is not satisfied
  --> src/lib.rs:10:9
   |
10 |         [T::default(); N]
   |         ^^^^^^^^^^^^^^^^^ the trait `Copy` is not implemented for `T`
   |
   = note: the `Copy` trait is required because the repeated element will be copied
help: consider further restricting this bound
   |
7  |     T: MyDefault + Copy,
   |                  ^^^^^^
```

"Ah," she realizes, "this would be cloning a single value of `T`, but I want to make `N` distinct values. How can I do that?"

She asks on Zulip and lcnr tells her that there is this [`map` function  defined on arrays](https://doc.rust-lang.org/std/primitive.array.html#method.map). She could write:


```rust
impl<T, const N: usize> Default for [T; N]
where
    T: Default,
{
    fn default() -> Self {
        [(); N].map(|()| T::default())
    }
}
```

"That code will build," lcnr warns her, "but we're not going to be able to ship it. Test it and see." Barbara runs the tests and finds she is getting a failure. The following test no longer builds:

```rust
fn generic<T>() -> [T; 0] {
    Default::default()
}
```

"Ah," she says, "I see that `Default` is implemented for any type `[T; 0]`, regardless of whether `T: Default`. That makes sense. Argh!"

Next she tries (this already relies on a nightly feature)
```rust
impl<T: Trait, const N: usize> Default for [T; N]
where
    T: Default,
    N != 0, // nightly feature!
{
    fn default() -> Self {
        [(); N].map(|()| T::default())
    }
}

impl<T> Trait for [T; 0] {
    // ...
}
```

While this does seem to compile, when trying to use it, it causes an unexpected error.

```rust
fn foo<T: Trait, const N: usize>() -> [u8; N] {
    <[T; N] as Trait>::ASSOC //~ ERROR `[T; N]` does not implement `Trait`
}
```

The compiler can't tell that `N == 0 || N != 0` is true for all possible `N`, so it can't infer `[T; N]: Trait` from `T: Trait`.

Frustrated, Barbara gives up and goes looking for another issue to fix.

Even worse, Barbara notices the same problem for `serde::Deserialize` and decides to
abandon backwards compatibility in favor of a brighter future.