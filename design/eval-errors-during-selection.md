# ❗🔄 Silence evaluation errors during selection

Given an impl like

```rust
trait Trait<const N: usize> {
    type Assoc;
}
impl<const N: usize> Trait<N> for [u8; N - 1] {
    type Assoc = u32;
}
impl<const N: usize> Trait<0> for [u8; 0] {
    type Assoc = u64;
}
```
This would cause an error during coherence because we fail to evaluate `N - 1` when using `0` for `N`.
The same issue also exists during candidate selection, checking whether `[u8; 0]: Trait<0>` holds
will cause the compiler to check whether the `[u8; N - 1]: Trait<N>` impl applies. For this the compiler
first tries to unify the two `TraitRef`s, unifying `N - 1` - after substituting `0` for `N` - with `0`.
Note that this unification happens before we ever consider any `where`-clauses.

### Can we avoid silent CTFE errors?

**tl;dr: no**

Changing CTFE to potentially not emit errors is not ideal and could be a new source of bugs.
Everything else being equal, we should avoid this.

Instead of using const evaluation failures to discard candidates, we could instead use conditions,
rewriting the above example to
```rust
struct Condition<const B: bool>;
trait IsTrue {}
impl IsTrue for Condition<true> {}

trait Trait<const N: usize> {
    type Assoc;
}
impl<const N: usize> Trait<N> for [u8; N - 1]
where
    Condition<{ N > 0 }: IsTrue,
{
    type Assoc = u32;
}
impl<const N: usize> Trait<0> for [u8; 0] {
    type Assoc = u64;
}
```

This would make it difficult for the compiler to check that these impls are [exhaustive](./exhaustiveness.md)
if that is something we want to add. **TODO: this isn't necessarily true, elaborate here**

It also might not even be enough to avoid incorrect errors during candidate selection.
We start candidate selection by trying to unify the the `TraitRef`s of the impl and the obligation.
Checking whether `[u8; 0]: Trait<N>` holds therefore tries to unify `[u8; ?0 - 1]: Trait<?0>` with `[u8; 0]: Trait<0>`
which - unless we explicitly avoid it - will cause us to evaluate `0 - 1` and emit an error. 

This issue is also prevalent for generic constants in `where`-clauses.

```rust
trait Foo {}
impl<const N: usize> Foo for [u8; N]
where
    Condition<{ N > 0 }>: IsTrue,
    [u8; N - 1]: Foo,
{}
```
We would have to somehow force the compiler to first check `Condition<{ N > 0 }>: IsTrue` before checking `[u8; N - 1]: Foo`.
While we could add some special predicate kind for "this has to evaluate to `true`", we still have the same issue in case
we have multiple such bounds which some dependencies between them. Another option would be to check predicates in a defined order,
e.g. in order of appearance. That is difficult to fit into the current query system and opens up a lot of difficult questions.

**TODO**: Refer to cyclic dependencies in where clauses here, that would also theoreticall be solved by relying on a syntactic order
and should go more in-depth than this.

We therefore believe that the option for CTFE errors to be silent is needed.

### Undefined behavior during CTFE

Whether the compiler detects undefined behavior during const evaluation is not part
of our stability guarantees, so we **must not** consider some candidate to not apply due to UB by itself.

We can therefore either only error if the impl would apply unless UB were to occur or always emit an error.
Always emitting an error when encountering UB seems desirable, even if it happens during selection.

### Reaching `#[const_eval_limit]`

When reaching the interpreter step limit, e.g. by using `loop {}`, it is unclear whether the code
would compile after some additional steps. We therefore have to treat this in the same way as UB.
Unlike UB, rejecting a candidate because the `#[const_eval_limit]` is reached would even be unsound,
as a different crate could raise that limit and would therefore consider the candidate to hold.

### Possible designs

We're going to refer to the initial example of this document. All of the possible designs here rely on
our ability to have silent CTFE errors and only differ in the way that they are used.

#### Rely on errors for disjointness

Do we want to consider impls for `[u8; N - 1]` and `[u8; 0]` to be disjoint by themselves?

Given that we have to silence the CTFE errors anyways, we could reject candidates if they contain a constant which fails to evaluate.
If we do this, the following change can break code:
```rust
// original version:
const fn my_computation(n: usize) -> usize {
    if some_edge_case(n) {
        unimplemented!()
    } else {
        whatever(n)
    }
}
// updated version:
const fn my_computation(n: usize) -> usize {
    if some_edge_case(n) {
        deal_with_edge_case(n)
    } else {
        whatever(n)
    }
}
```
It generally seems desirable to not consider such a change to be breaking. Such a case actually causing issues seems quite unlikely,
though it definitely may happen that people write overlapping impls relying on `some_edge_case` precisely because `my_computation` panics in
that case.

#### Require explicit bounds

We could also require a `Condition<{ N > 0 }>: IsTrue` bound for this to compile,
potentially adding some syntactic sugar or special predicate for this.

This adds some additional boilerplate to impls, though it does reduce the
amount of implicit behavior the user has to understand. It also prevents any unintentional breakage
when fixing buggy functions which would cause them to terminate successfully instead of panicking.
For types like `[u8; N / 2 - 1]` the correct boolean condition to prevent this from panicking isn't immediately obvious however.
This could introduce some errors. To avoid these we could add a lint, recommending the correct boolean condition for a given computation where possible.

If a candidate contains a CTFE failure but would otherwise hold, we can either consider that candidate to be ambiguous or emit a hard
error. The exact impact of making candidates ambiguous isn't yet clear and needs further consideration.

#### Require explicit bounds for CTFE failures inside of functions

Changing the body of a constant used in the type system will be a breaking change as it may can prevent unification.
It might therefore be sensible that errors originating directly in the anonymous constant - or transparent named constant - 
cause the candidate to be rejected while errors originating in functions - or opaque named constants - would require an explicit bound.

## Status

**Idle**: We can already work on this but it's also fine to wait until `feature(generic_const_exprs)` is
closer to stable.