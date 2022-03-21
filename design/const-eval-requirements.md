# ❗🔙 Restrictions on const evaluation

For generic constants in the type system to be sound, const evaluation must
not be able to differentiate between values considered equal by the type system.
Const evaluation must also be fully deterministic and must not depend on any external state
when used in the type system.

## Floats

Using floats during CTFE is fully determinstic. So using
them inside of the type system is fine. CTFE can however
produce different results than what would happen on real hardware,
but this is not a concern for const generics.

## Dealing with references and allocations

Other sources of non-determinism are allocations. This non-determinism
must however not be observed during const-evaluation (TODO: link to const-eval).

Any references used in a constant are considered equal if their targets are equal, which is also determistic.
This is needed by [valtrees](./valtrees.html). The specific design of valtrees also adds some other
additional constraints to const evaluation which are listed on its page.
