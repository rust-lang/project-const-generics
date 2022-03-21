# ❔ Exhaustiveness

Whether we want to be able to unify different partial impls to consider them exhaustive.

```rust
trait ForBool<const B: bool> {}

impl ForBool<true> for u8 {}
impl ForBool<false> for u8 {}
// Does `for<const B: bool> u8: ForBool<B>` hold?
```

## Status

**Idle**: While not blocked, it may make sense to wait until we are able to [constrain generic parameters](./constraining-generic-parameters.html).