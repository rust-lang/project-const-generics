# Anon consts substs

Currently locked behind `feature(lazy_normalization_consts)` or `feature(const_generics)`, we do not supply anonymous constants
with their parents generics.

Doing so is needed if we want to ever support more complex generic expressions in anonymous constants.

## Known blockers

Unless said otherwise, "currently" means "with `feature(const_generics)` enabled" for the rest of this document.

### Unused substs

Anon consts currently inherit all of their parents generic parameters. This breaks code already compiling on stable and causes unexpected compilation failures:

[#78369](https://github.com/rust-lang/rust/issues/78369)
```rust
#![feature(lazy_normalization_consts)]
struct P<T: ?Sized>([u8; 1 + 4], T);

fn main() {
    let x: Box<P<[u8; 0]>> = Box::new(P(Default::default(), [0; 0]));
    let _: Box<P<[u8]>> = x; //~ ERROR mismatched types
}
```
const occurs check (with [#81351](https://github.com/rust-lang/rust/pull/81351)):
```rust
#![feature(lazy_normalization_consts)]

fn bind<const N: usize>(value: [u8; N]) -> [u8; 3 + 4] {
    todo!()
}

fn main() {
    let mut arr = Default::default();
    arr = bind(arr); //~ ERROR mismatched type
}
```

#### Status

Discussed in the following meetings: [2021.02.09](../meetings/2021.02.09-lazy-norm.md), [2021.02.16](../meetings/2021.02.16-lazy-norm.md), [2021.03.09](../meetings/2021.03.09-unused-substs-impl.md)

A solution has been found and is currently getting implemented in [#83086](https://github.com/rust-lang/rust/pull/83086)

### Consts in where bounds can reference themselves

This causes query cycles for code that should compile and already compiles on stable

[#79356](https://github.com/rust-lang/rust/issues/79356)
```rust
#![feature(const_generics)]
#![allow(incomplete_features)]

struct Foo<T>(T);

fn test() where [u8; {
    let _: Foo<[u8; 3 + 4]>;
    3
}]: Sized {}
```
#### Status

Discussed in the following meetings: [2021.02.23](../meetings/2021.02.23-ct-in-where-bounds.md)

Not yet solved, potentially fixed by improving the query/trait system to deal with cycles. Maybe blocked on *chalk*. 

### Consts in where clauses are executed with inconsistent substs

We evaluate consts in where clauses without first proving the where clauses of the const itself, potentially causing ICE.

```rust
#![feature(const_generics, const_evaluatable_checked)]

trait Foo {
    type Assoc;
    const ASSOC: <Self as Foo>::Assoc;
}

impl Foo for i32 {
    type Assoc = i64;
    const ASSOC: i64 = 3;
}

const fn bar<T>(x: T) -> usize {
    std::mem::forget(x);
    3
}

fn foo<T>() where
    T: Foo<Assoc = T>,
    [u8; bar::<T>(<T as Foo>::ASSOC)]: Sized,
{}

fn main() {
    foo::<i32>();
}
```

#### Status

Discussed in the following meetings: [2021.03.23](../meetings/2021.03.23-ct-in-where-bounds.md)

Not yet solved, requires us to either change typeck to not evaluate constants without checking their
where clauses or to stop assuming ctfe to only deal with well typed MIR.









