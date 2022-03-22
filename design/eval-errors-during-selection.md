# ❗🔄 Silence evaluation errors during selection

Given an impl like

```rust
trait Trait<const N: usize> {
    type Assoc;
}
impl<const N: usize> Trait<N> for [u8; N - 1] {
    type Assoc = u32;
}
impl<const N: usize> Trait<0> for [u8; 0] {
    type Assoc = u64;
}
```
This would cause an error during coherence because we fail to evaluate `N - 1` when using `0` for `N`.

### Can we avoid silent CTFE errors?

Changing CTFE to potentially not emit errors is not ideal and could be a new source of bugs.
Everything else being equal, we should avoid this.

Instead of using const evaluation failures to discard candidates, we could instead use conditions,
rewriting the above example to
```rust
struct Condition<const B: bool>;
trait IsTrue {}
impl IsTrue for Condition<true> {}

trait Trait<const N: usize> {
    type Assoc;
}
impl<const N: usize> Trait<N> for [u8; N - 1]
where
    Condition<{ N > 0 }: IsTrue,
{
    type Assoc = u32;
}
impl<const N: usize> Trait<0> for [u8; 0] {
    type Assoc = u64;
}
```
This would make it very hard for the compiler to check that these impls are [exhaustive](./exhaustiveness.md).

Additionally, this may not be enough to completely avoid silent CTFE errors, as TODO: consts in where clauses.

### Undefined behavior during CTFE

Whether the compiler detects undefined behavior during const evaluation is not part
of our stability guarantees, so we **must not** consider some candidate to not apply due to UB by itself.

It generally seems desirable to always have error when encountering UB, even if it happens during selection.

## Status

**Idle**: We can already work on this but it's also fine to wait until `feature(generic_const_exprs)` is
closer to stable.