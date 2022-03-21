# ❔🔙 Functions as const parameters

**[project-const-generics#7](https://github.com/rust-lang/project-const-generics/issues/7)**

It would be nice to allow function pointers as const parameter types.

```rust
fn foo<const N: fn() -> usize>() {}
```

While they do have a sensible definition of structural equality at compile time,
comparing them is not deterministic at runtime. This behavior might be fairly surprising,
so it might be better to not allow this.

## Current status

**Planning/Blocked**: needs `generic_const_exprs` to be closer to stable before it makes sense to start working on this.
