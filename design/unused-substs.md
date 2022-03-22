# ❗🔙 🔄 Unused substs

This is one of the core issues wrt generic constants in the type system.
With `feature(generic_const_exprs)` we would like `foo::<{ N + 3 }>` to be valid code.
The current implementation gives anonymous constants all of the generics and where clauses of the parent item:
```rust
fn bar<const N: usize, T: Trait>() {
    foo::<{ N + 1 }>();
}
```
is desugared to
```rust
fn bar<const N: usize, T: Trait>() {
    const ANON_CONST<const N: usize, T: Trait>: usize = N + 1;
    foo::<ANON_CONST::<N, T>>();
}
```

This causes a lot of issues.

#### Forcing all type parameters to be invariant

```rust
struct Foo<'a, T, const N: usize> {
    field: [u8; N + 1],
    by_ref: &'a T,
}
```
is desugared to
```rust
const ANON_CONST<'a, T, const N: usize>: usize = N + 1;
struct Foo<'a, T, const N: usize> {
    field: [u8; ANON_CONST::<'a, T, N>],
    by_ref: &'a T,
}
```
Generic arguments of anonymous constants are forced to be invariant. This
causes `'a` and `T` be invariant even though they aren't actually used.

#### Unsize coercions ([#78369](https://github.com/rust-lang/rust/issues/78369))

```rust
struct P<T: ?Sized>([u8; 1 + 4], T);

fn main() {
    let x: Box<P<[u8; 0]>> = Box::new(P(Default::default(), [0; 0]));
    let _: Box<P<[u8]>> = x;
}
```
This compiles on stable but fails to compile if we provide `T` to the anonymous constant `1 + 4` as
we now cannot unsize `T` anymore.

#### Self referential closures ([#85665](https://github.com/rust-lang/rust/issues/85665))

```rust
fn foo<F: FnOnce([u8; N - 1]), const N: usize>(_: F) {}

fn main() {
    foo(|arg: [u8; 5]| ());
}
```
The anonymous constant `N - 1` has `F` as a generic parameter. This means that the function signature of `F` contains
`F` itself, causing an error.

## How this might get fixed

As this is one of the core issues of const generics, we thought about quite a few potential solutions to this issue.

### Completely replacing `ty::ConstKind::Unevaluated`

By never representing anonymous constants as `ty::ConstKind::Unevaluated` and instead directly converting them into some abstract
representation, we can avoid these issues, for example:

```rust
fn foo<F: FnOnce([u8; N - 1]), const N: usize>(_: F) {}
// We could represent `N - 1` as `ty::ConstKind::Expr(ty::Expr::Sub(N, 1))`,
// meaning that we don't mention `F` at all.
```

This means we directly embed the fully expanded [AbstractConst](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_trait_selection/traits/const_evaluatable/struct.AbstractConst.html) directly as a variant of `ty::ConstKind`.

- this has to happen incredibly early, can't really use typeck to get the abstract representation, add some special "mini typeck" which
directly converts `hir` into the `ty::ConstKind::Expr` representation.
    - what if that mini typeck disagrees with the "correct typeck"
    - do we still want to typeck anon consts or do we only use the "mini typeck"
    - the "mini typeck" will only allow a fairly arbitrary and small subset
        - the more we allow, the higher our maintenance
        - will be fairly untested, so there could be a lot of bugs and bad diagnostics here
        - though, even when not using the mini typeck, we still have to restrict expressions which can
            be used in generic anonymous constants
    - the "mini typeck" can't be used for concrete anonymous constants
        - constants which don't explicitly mention a generic param keep getting typechecked without any generics in scope.