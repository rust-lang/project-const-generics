# ❗ Constraining generic parameters

Given an impl, the compiler has to be able to decide the generic arguments used by that impl.
Consider the following snippet:

```rust
struct Weird<const N: usize>;

impl<const A: usize, const B: usize> Weird<{ A + B }> {
    fn returns_a() -> usize {
        A
    }
}
```

When calling `Weird::<3>::returns_a()`, there is no way to restrict the generic parameters `A` or `B` so this has to error.
If a generic parameter is used by an injective expression, then we should allow this. The most relevant case here are
constructors:
```rust
struct UsesOption<const N: Option<usize>>;
impl<const N: usize> UsesOption<{ Some(N) }> {}
```
Here it is very clear which `N` we should use given `UsesOption::<{ Some(3) }>`.

## Current status

**Blocked**: Before making any decisions here we should first figure out [structural equality](./design/structural-equality.md)
