# ❔ Overly restrictive variance

Currently all generic parameters used by constants in the type system are constrained
to be invariant. Looking at this example, that is unnecessary:

```rust
struct UsesRef<'a> {
    actual_ref: &'a (),
    in_anon_const: [u8; std::mem::size_of::<&'a ()>()],
}
```
Here `'a` is forced to be invariant, as it is used in the anonymous constant, but
ideally it should be covariant.

While variance mostly doesn't matter for CTFE, it is also responsible for higher ranked
subtyping. Higher ranked types act different during candidate selection and can therefore
not simply be ignored. We currently have a future compatability lint for this in
[#56105](https://github.com/rust-lang/rust/issues/56105). Even once higher ranked lifetimes
can't influence candidate selection anymore, it is unclear whether there are any other issues
here, so this would still need some further consideration.

## Status

**Blocked**