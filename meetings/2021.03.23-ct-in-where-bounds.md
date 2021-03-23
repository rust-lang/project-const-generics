# cg meeting 2021-03-23 ([zulip thread](https://rust-lang.zulipchat.com/#narrow/stream/260443-project-const-generics/topic/meeting.202021-03-23))

## Consts in `where`-bounds

```rust
fn foo<T: Trait>() where [u8; <T as Trait>::ASSOC]: OtherTrait {}
```

Help, our `where`-bounds don't hold:

### ICE example

(first discussed in https://rust-lang.zulipchat.com/#narrow/stream/260443-project-const-generics/topic/anon.20const.20in.20where.20bounds )

Core example is:

```rust
trait Foo {
    type Assoc;
}
const fn bar<T>()
where
    T: Foo<Assoc = T>,
{
    ...
}
fn fun1<T>
where
    T: Foo<Assoc = T>,
    [u8; bar::<T>()]: Sized,
{}
impl Foo for i32 {
    type Assoc = u32;
}
fn main() {
    fun1::<i32>();
}
```

* The problem here is that when you call `fun1::<i32>` you wind up with this unevaluated constant `bar::<T>()`.
* We type-check that constant when type-checking `fun1` and determine that the source code is well-typed, under the assumption that `T: Foo<Assoc = T>`
* But in the body of `main` we get two distinct things to prove:
    * `i32: Foo<Assoc = i32>` — unprovable
    * `[u8; bar::<i32>()]: Sized` — no longer well typed
* The current code considers them independently and may well ICE when evaluating the second one (not probably in this particular example) because it assumed that the body of a constant is well typed when evaluating.

### Possible solutions

- don't assume substituted anon consts to be well typed
    - dangerous, may hide actual bugs
- change typeck to not evaluate constants before proving there `where`-bounds
    - difficult to implement

:shrug:
