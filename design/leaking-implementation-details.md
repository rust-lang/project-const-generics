# ❗🔙 Leaking implementation details

When unifying generic constants, we have to avoid accidentally stabilizing
parts of our lowering.

```rust
fn foo<const B: bool>() -> [u8; if B { 3 } else { 7 }] {
    [0; match B {
        true => 3,
        false => 7,
    }]
}
```

If we allow `foo` to compile, we have to guarantee that we will continue supporting it
in the future. While this is fine if we intentionally allow it, it could also be allowed
by accident if we we use the same representation for `if` and `match` in the abstract representation for
generic constants.

Another issue here is the removal of noops, like redundant `as`-casts. This was an issue with a
previous implementation which relied on the MIR of generic constants for unification.

```rust
fn multiple_casts<const N: u8>() -> [u8; N as usize as usize] {
    [0; N as usize]
}
```

In general, the unification of generic constants has to add a new representation for expressions which can
be used in types. If we end up stabilizing `feature(generic_const_exprs)` with a suboptimal lifting from expressions
to that new representation, this will probably be quite difficult to fix while maintaining backwards compatability.