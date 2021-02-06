# Generic const param types

We want to support the types of const parameters
to depend on other generic parameters.
```rust
fn foo<const LEN: usize, const ARR: [u8; LEN]>() -> [u8; LEN] {
    ARR
}
```

This is currently forbidden during name resolution.

Probably the biggest blocker is type-checking const arguments
for generic parameters. This currently uses [a hack][WithOptConstParam]
to supply the `DefId` of the corresponding const parameter.

Now, let's look at the following:
```rust
fn foo<const N: usize, const M: [u8; N]>() {}

fn bar() {
    foo::<3, { [0; 3] }>();
}
```
Here the expected type of `{ [0; 3] }` should be `[u8; 3]`. With the
current approach it is `[u8; N]` instead. To fix this we would have to
supply the affected queries the expected type of the const argument itself
or use a `(DefId, SubstsRef<'tcx>)` pair instead.

Doing so isn't trivial because we have to worry about accidentially
ending up with different types for the same const argument which would
break stuff. 

Potential dangers include partially resolved types and projections.

[WithOptConstParam]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.WithOptConstParam.html