# 😱 Status quo: Array split first method

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
error: the constant expression `N-1` is not known to be evaluatable
2 |     let (first, rest) = split_first(arr);
  |                         ^^^^^^^^^^^ `N-1` not known to be evaluatable
info: N may underflow
help: add a where clause to `some_method`
  | fn some_method<const N: usize>(arr: [u8; N]) where [(); {N - 1}]:
```

Barbara hits the 'quick fix' button in her IDE and it inserts the where clause for her- she immediately
gets a compile error at another spot because she was calling `some_method` with an empty array:

```
error: integer underflow evaluating constant expression
22 |     some_method([])
   |     ^^^^^^^^^^^^^^^ `0-1` is not evaluatable
info: `0-1` must be evaluatable because of this where clause
  | fn some_method<const N: usize>(arr: [u8; N]) where [(); { N - 1}]:
  |                                                    ---------------
```

She also gets a compile error at another spot with a `[(); { N - 2; }]:` where clause in scope
```rust
fn some_other_method<const N: usize>(arr: [u8; N]) where [(); { N - 2; }]: {
    // ...
    let (first, rest) = split_first(arr);
    // ...
}
```
```
error: the constant expression `N-1` is not known to be evaluatable
2 |     let (first, rest) = split_first(arr);
  |                         ^^^^^^^^^^^ `N-1` not known to be evaluatable
info: N may underflow
help: add a where clause to `some_method`
  | fn some_method<const N: usize>(arr: [u8; N]) where [(); { N - 2; }}:, [(); { N - 1; }];, {
```

"What!!! That's silly"- Barbara sighs, hitting the quick fix button and moving on

(rustc is not currently smart enough to know that `N - 2` being evaluatable implies `N - 1`)

## Alt Universe with post-mono errors

Barbara is working on her project. She has the idea to write a `split_first` function that will allow her to split out the first item from a fixed-length array; naturally, the array must be non-empty. It looks something like this:

```rust
// Note: this method has no implied where clause that `N - 1` evaluates
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

Everything seems fine when she runs `cargo check`. Then later she runs `cargo test` and sees a compilation error:

```
error: const evaluation error occurred
22 |    let tail: [T; N - 1] = // ...
   |                  ^^^^^ integer underflow, cannot subtract `1` from `0`
info: this const evaluation was required by `some_other_method`, which contains:
22 |     some_method([])
info: `some_method` contains:
22 |    let (first, rest) = split_first(arr);
info: `split_first` contains:
22 |    let tail: [T; N - 1] = // ...
```