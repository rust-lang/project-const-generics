# ✨ Shiny future story: Array split first method

Shiny future of [status quo story](../status_quo/split_first.md)

Barbara is working on her project. She has the idea to write a `split_first` function that will allow her to split out the first item from a fixed-length array; naturally, the array must be non-empty. It looks something like this:

```rust
// Note: this method has an implied where clause that `N - 1` evaluates without 
// erroring because `N - 1` is in the function signature
fn split_first<T, const N: usize>(arr: [T; N]) -> (T; [T; N - 1]) {
    // ...
    let tail: [T; N - 1] = // ...
    (head, tail)
}
```

Next she wants to write a function that uses `split_first`:

```rust
fn some_method<const N: usize>(arr: [u8; N]) {
    let (first, rest) = split_first(arr);
    for i in rest {
        // ...
    }
}
```

The compiler gives her a compile error:
```
error: the constant expression `N - 1` is not known to evaluate without underflowing
2 |     let (first, rest) = split_first(arr);
  |                         ^^^^^^^^^^^ `N - 1` not known to be evaluatable without underflowing
note: required by this expression in `split_first`'s function signature
5 |     fn split_first<T, const N: usize>(arr: [T; N]) -> (T; [T; N - 1]) {
  |                                                               ^^^^^
help: add a where clause to `some_method`
  | fn some_method<const N: usize>(arr: [u8; N]) where { N > 0; }
```

Barbara hits the 'quick fix' button in her IDE and it inserts the where clause for her- she immediately
gets a compile error at another spot because she was calling `some_method` with an empty array:

```
error: the constant `0` is not greater than `0`
22 |     some_method([])
info: `0` must be greater than `0` because of this where clause
  | fn some_method<const N: usize>(arr: [u8; N]) where { N > 0; }
  |                                                    ----------
```

Barbara has no more compile errors, even the following code is compiling:
```rust
fn some_other_method<const N: usize>(arr: [u8; N]) where { N > 1; } {
    // ...
    let (first, rest) = split_first(arr);
    // ...
}
```