# ❗🔙 Opaque and transparent associated constants

As we want to be able to use associated constants in types, we have to look into them for unification.

```rust
trait Encode {
    const LENGTH: usize;
    fn encode(&self) -> [u8; Self::LENGTH];
}

struct Dummy<const N: usize>;
impl<const N: usize> for Dummy<N> {
    const LENGTH: usize = N + 1;
    fn encode(&self) -> [u8; Self::LENGTH] {
        [0; N + 1]
    }
}
```

For this to compile, we have to unify `Self::LENGTH` with `N + 1`. That means that we have to look into
this associated constant, i.e. we have to treat it as transparent.

Treating all associated constants causes some concerns however.

#### Additional stability requirements

Consider a library which has a public associated constant

```rust
use std::mem::size_of;
trait WithAssoc {
    const ASSOC: usize;
}

struct MyType<T>(Vec<T>);
impl<T> WithAssoc for MyType<T> {
    const ASSOC: usize = size_of::<Vec<T>>();
}
```

In a new minor version, they've cleaned up their code and changed the associated constant to `size_of::<Self>()`.
Without transparent associated constant, this can't break anything. If `ASSOC` is treated transparently,
this change is now theoretically breaking. 

#### Overly complex associated constants

Our abstract representation used during unification will not be able to represent arbitrary user code.
This means that there will be some - potentially already existing - associated constants we cannot convert
to this abstract representation.

It is unclear how much code is affected by this and what the right solution for this would be.
We should probably either silently keep these associated constants opaque or keep them opaque and emit a warning.

### Keeping some associated constants opaque

Still undecided how this should be handled. We can make transparent constants opt-in, potentially making associated constants
used in some other type of the trait impl transparent by default. We could also make constants transparent by default
and keep constants for which the conversion failed opaque, either emitting a warning or just silently.

## Status

**Blocked**: We should wait for `feature(generic_const_exprs)` to be closer to stable before thinking about this.