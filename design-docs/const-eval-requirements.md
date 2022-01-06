# Const eval requirements

For this to work, const operations have to be deterministic and
must not depend on any external state,
at least when they are used in the type system.

Using floats during CTFE is fully determinstic. So using
them inside of the type system is fine. CTFE can however
produce different results than what would happen on real hardware,
but this is not a concern for const generics.

Other sources of non-determinism are allocations. This non-determinism
must however not be observed during const-evaluation (TODO: link to const-eval).
Any references used in a constant are considered equal if their targets are equal, which is also determistic. (ref [val-trees](https://github.com/rust-lang/rust/issues/72396))