# ✨ Shiny future story: Barbara implements `Default` for all array lengths

Shiny future of [status quo story](../status_quo/array_default.md)

Barbara is working on `std`. She saw that the newest version of rustc has had some improvements to const generics
and decides to try implementing `Default` for all array lengths. She goes to write:

```rust
impl<T, const N: usize> Default for [T; N]
where
    T: Default,
{
    fn default() -> Self {
        /* snip */
    }
}
```

The code builds just fine but then she sees a test failing:
```rust
fn generic<T>() -> [T; 0] {
    Default::default()
}
```

"Ah," she says, "I see that Default is implemented for any type [T; 0], regardless of whether T: Default. That makes sense. Argh!"

Next she tries to write:
```rust
impl<T, const N: usize> Default for [T; N]
where
    T: Default,
    { N > 0 },
{
    fn default() -> Self {
        /* snip */
    }
}

impl<T> Default for [T; 0] {
    fn default() -> Self {
        []
    }
}
```

This compiles just fine and the test is passing. She decides to submit a PR where her reviewer asks her to 
add a test to make sure that `[T; N]: Default` holds when `T: Default` as this didn't used to work

```rust
fn exhaustive_default_impl<T: Default, const N: usize>() -> [T; N] {
    <[T; N] as Default>::default()
}
```

This test passes just fine, "yay const generics ✨" she says