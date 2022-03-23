# 📚 Design

For an explanation of how const generics currently works in the compiler
also check out the [rustc-dev-guide](https://rustc-dev-guide.rust-lang.org/constants.html).

Const generics has quite a few deep rooted issues and design challenges.
In this document we try to highlight some specific topics which are fairly
pervasive while working on this feature.
The goalis to more clearly understand the
current state of const generics and to help us figure out ways to
make progress. Without this the challenges we currently face seem
near insurmountable.

### 🔙 Backwards compatability

Rust was not design with const generics in mind, and we've also made
some unfortunate decisions even after work on const generics started.

Future extensions of const generics must not cause substantial breakage
to existing code, or if they do, this breakage has to be contained somehow.
A possible solution to many such changes are editions.

### ⚖️ No perfect solution

There isn't just one *perfect* version of const generics we are working towards.
Instead there are a lot of major and minor tradeoffs between learnability, expressiveness,
implementation complexity and many other factors. As we can't perfectly tell how
const generics will be used in the future, making these decisions can be especially
hard. For any such issues we should try to reach out to the wider community before
coming to a final conclusion.

### 🔄 The query system

Const generics mixes evaluation with typechecking which very quickly reaches the edge
of what is possible with the query system. It might make sense to look at these issues in
unison and consider whether there are some more fundamental changes to the compiler which
provide a better solution to them instead of [adding more hacks](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.WithOptConstParam.html).

### ❔ Optional extensions

There are a lot of things which are nice to have but not strictly necessary or maybe not even
desirable. This means that decisions which prevents them may still be the correct ones,
i.e. these are not strictly a blocking concern.
We should still try to find alternatives before blocking these though.

## Const parameter types ([`adt_const_params`](https://github.com/rust-lang/rust/issues/95174))

On stable, only integers, `bool`, and `char` is allowed as the type of const parameters.
We want to extend this to more types in the future.
```rust
struct NamedType<const NAME: &'static str> { ... }
```

Additionally, we want to support generic const parameter types and must not stabilize anything
which prevents the addition of that in the future.
```rust
struct WithGenericArray<T, const N: usize, const ARRAY: [T; N]> { ... }
```

This feature interacts with the following topics:

- [❗ Constraining generic parameters](./constraining-generic-parameters.md)
- [❗🔙 ⚖️ Structural equality](./structural-equality.md)
- [❗⚖️ Valid const parameter types](./valid-const-parameter-types.md)
- [❗ Valtrees](./valtrees.md)
- [❗⚖️🔄 Generic const parameter types](./generic-const-param-types.md)
- [❔ Exhaustiveness](./design/exhaustiveness.md)
- [❔🔙 Functions as const parameters](./functions-as-const-parameters.md)

## Generic constants in the type system ([`generic_const_exprs`](https://github.com/rust-lang/rust/issues/76560))

- [❗🔙 🔄 Unused substs](./unused-substs.md)
- ❗ Anonymous constants in binders
- [❗🔄 Silence evaluation errors during selection](./eval-errors-during-selection.md)
- ❗🔄 Self referential where clauses
- ❗🔄 Evaluation without first checking where-clauses
- ❔ Partially evaluating anonymous constants
### Unifying generic constants

- [❗🔙 Restrictions on const evaluation](./const-eval-requirements.md)
- [❗🔙 Leaking implementation details](./leaking-implementation-details.md)
- [❗🔙 Opaque and transparent associated constants](./opaque-and-transparent-assoc-consts.md)
- ❗ Splitting constants during unification
- [❗ Constraining generic parameters](./constraining-generic-parameters.md)
- ❔⚖️ Extending unification logic

### ❔ Const evaluatable bounds

- ❗ Using subtrees to fulfill bounds
- ❔ Discarding bounds and soundness
- ❔ Compile time conditionals
- ❔⚖️ Improving const evaluatability checks

ALL OF THIS

## Repeat length backcompatability lint ([`const_evaluatable_unchecked`](https://github.com/rust-lang/rust/issues/76200))

close this or wait until it's possible on stable.

## `Default` for arrays

possible solutions/combinations of features to allow this. Both in std and for arbitrary crates