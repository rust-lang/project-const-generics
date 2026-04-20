# The plan for `min_generic_const_args`

> [!NOTE]  
> This document was written back in 2024 and has not been updated since.
> It may contain dated/incorrect information.

## Where does `min_const_generics` fall short

The `min_const_generics` feature was stabilized [at the end of 2020](https://github.com/rust-lang/rust/pull/79135) with a few significant limitations. These limitations are explained nicely in WithoutBoats' blog post titled [Shipping Const Generics in 2020](https://without.boats/blog/shipping-const-generics/) and is reccomended reading before continuing.

The focus of this document is on allowing more complex expressions involving generic parameters to be used as const arguments, "No complex expressions based on generic types or consts" as titled in boats' blog post.

This document intends to explain:
- Why it is hard to allow arbitrary expressions involving generic parameters to be used in the type system
- What design the const generics project group intends to pursue in order to allow more complex expressions involving generic parameters.
- How the proposed design avoids pitfalls encountered by previous experimentation in this area.

## Why does `generic_const_exprs` not work

Allowing complex expressions involving generic parameters has been experimented with under the `generic_const_exprs` feature gate for the past few years. In short it allows writing code such as `[u8; <T as Trait>::ASSOC]` or `where Foo<{ N + 1 }>: Other<M>`; arbitrary expressions are supported as const arguments regardless of whether they involve generic parameters or not.

### Background implementation details

Currently const arguments (even under `min_const_generics`) are implemented by desugaring the const argument to a const item (called anonymous constants) and then referring to it. 
For example under `min_const_generics`:
```rust
fn foo<T: Trait>(arg: T) -> [T; MY_CONST + 2] {
    /* ... */
}

// gets desugared to:

const ANON: usize = MY_CONST + 2;
fn foo<T: Trait>(arg: T) -> [T; ANON] {
    /* ... */
}
```

The implementation strategy for `generic_const_exprs` is that the implicitly created `ANON` has the same generics and where clauses defined on it as `foo`. Additionally we also implicitly create `terminates { ... }` where clauses for any const arguments used in the signature (more on this later).

The previous example with the `generic_const_exprs` feature gate enabled would be desugared like so:
```rust
#![feature(generic_const_exprs)]

fn foo<T: Trait>(arg: T) -> [T; MY_CONST + 2] {
    /* ... */
}

// gets desugared to:

const ANON<T: Trait>: usize = MY_CONST + 2
where
    terminates { ANON::<T> };
fn foo<T: Trait>(arg: T) -> [T; ANON]
where
    terminates { ANON::<T> }
{
    /* ... */
}
```

With all that covered we can begin to talk about the issues with the current design of `generic_const_exprs`.

---

### Const arguments that fail to evaluate

In Rust not all expressions are guaranteed to terminate and return a value, they can loop forever, panic, encounter UB, etc. This introduces some complexity for const generics as we must decide on how to handle a const argument such as `Foo<{ usize::MAX + 1 }>` or `Foo<{ loop {} }>`.

While the previous examples are relatively easy to determine that they will panic and loop forever respectively, it is not so easy in the general case. This is quite literally the halting problem.

A conservative analysis is, in general, possible in order to guarantee termination of functions (a lot of functional programming languages implement this). Implementing this would require another function modifier, e.g. `const terminates fn foo() -> usize;` and generally complicate the language significantly.

The (stabilized) `min_const_generics` feature handles this by requiring any complex expressions to not use generic parameters, this means it is always possible to *try* to evaluate the expression. If it fails to evaluate then we error, otherwise we have a value.

As `generic_const_exprs` supports arbitrary expressions that *do* use generic parameters, such as `Foo<{ N + 1 }>`, it is no longer always possible to evaluate a const argument as it may involve generic parameters that are only known after monomorphization.

Given the example of `Foo<{ N + 1 }>` it would panic when `usize::MAX` is provided as the value of `N`. Some ways this could potentially be handled, one of which was already mentioned, would be:
- Introduce a `terminates` function modifier and have an addition function that is guaranteed to terminate if the type system can prove `N < usize::MAX`
- Post monomorphization errors for when generic arguments are provided to `N` that would happen to cause it to fail to evaluate
- Require the caller to prove that `{ N + 1 }` will terminate

The last option is what `generic_const_exprs` implements, to give a concrete example:
```rust
#![feature(generic_const_exprs)]

fn foo<const N: usize>() -> [u8; N] {
    /* ... */
}

fn bar<const N: usize>()
where
    terminates { N + 1 },
{
    // fine, caller has checked that `{ N + 1}` will terminate
    foo::<{ N + 1 }>();
}

fn baz() {
    // fine, `{ 10 + 1 }` terminates
    bar::<10>();    
    
    // ERROR: `{ { usize::MAX } + 1 }` will panic
    bar::<{ usize::MAX }>();
}

fn bar_caller_2<const N: usize>()
where
    terminates { { N * 2 } + 1 },
{
    // fine, caller has checked that `{ { N * 2 } + 1 }` will terminate
    bar::<{ N * 2 }>();
}

fn main() {
    baz();
    
    // fine, `{ { 10 * 2 } + 1 }` terminates
    bar_caller_2::<10>();
    
    // ERROR: `{ { { usize::MAX - 100} * 2 } + 1 }` will panic
    bar_caller_2::<{ usize::MAX - 100 }>();
}
```

Sidenote: In the actual implementation these bounds are not written as they are in the example. I opted to use `terminates { ... }` syntax as it is significantly more readable than what is actually implemented.

These `terminates` bounds are a huge design issue with the current feature as they require putting large amounts of implementation details of the function in its public API. I would not expect us to want to stabilize anything with this solution. 

### Unrecoverable CTFE errors

In general the trait system should be able to fail in arbitrary ways without causing compilation to fail. This requirement comes from a few places:
- Coherence evaluates where clauses and if they fail to hold use that information to consider impls to not overlap.
- During type checking/trait solving we often speculatively try some inference constraints and if they succeed great, otherwise we try other inference constraints

In these cases we can wind up evaluating generic constants with generic arguments that would cause evaluation failures. For example the following example:

```rust
trait Trait<const N: usize> {
    type Assoc;
}
impl<const N: usize> Trait<N> for [u8; N - 1]
where
    terminates { N - 1 }
{
    type Assoc = u32;
}
impl Trait<0> for [u8; 0] {
    type Assoc = u64;
}
```

In this example coherence attempts to check whether the two impls are overlapping or not. In order to do this we wind up inferring `N=0` and then trying to prove all of the where clauses on the generic impl.

The generic impl has only one where clause:`terminates { N - 1 }`, with generic arguments applied this is actually `terminates { 0 - 1 }`. Evaluating `0 - 1` panics which emits a *hard error* by the const eval machinery.

As hard errors are unrecoverable we wind up erroring on this code with the message that evaluating `0 - 1` failed even though in theory this ought to succeed compilation.

While `terminates` bounds were used in this example as it is a very simple demonstration of the issue, they are not strictly required to demonstrate this and so removing `terminates` bounds would not solve this.

Changing const eval to not emit a hard error when failing to evaluate constants is undesirable as the current setup prevents a lot of potential issues that could arise from forgetting to emit an error in other cases. 

Any design for generic const arguments needs to be able to avoid evaluating constants that might not succeed or else there will be weird behaviour and error messages in coherence and trait solving.

### Checking equality of generic consts causes query cycles

The type system representation of constants must be able to represent the actual expressions written not just a usage of an anon const. This is so that two separate `{ N + 1 }`  expressions are considered to be equal.

For example the following code should compile even though `N + 1` is written in two different places and would result in two diferent anon consts:
```rust
#![feature(generic_const_exprs)]

fn foo<const N: usize>(a: Bar<{ N + 1 }>)
where
    Bar<{ N + 1 }>: Trait,
{
    baz(a)
}

fn baz<T: Trait>(_: T) { }
```

The way that `generic_const_exprs` handles this is by looking at the THIR (post-typeck hir) of anon consts and then lowering it to a new AST specifically for type level expressions.

The requirement that we first type check the anon const before being able to represent its normalized form in the type system causes a lot of query cycle isues when anon consts are used in where clauses.

These cycles tend to look something like the following:
- Try to type check an anon const in a where clause
- Type checking that anon const requires doing trait solving
- Trait solving tries to use a where clause containing the original anon const
- We try to type check the original anon const again

A concrete example of this kind of thing would be the following code:
```rust
#![feature(generic_const_exprs)]

trait Trait<const N: usize> {
    const ASSOC: usize;
}

fn foo<T, U, V, const N: usize>()
where
    T: Trait<{ N + 1 }>,
    T: Trait<{ <T as Trait<{ N + 1 }>>::ASSOC }>,
{
    
}
```

In this example we might encounter a cycle like so:
- We try to type check `{ <T as Trait<{ N + 1 }>>::ASSOC }`
- This requires proving `T: Trait<{ N + 1 }>`
- There are 2 possible where clauses that from the trait solvers POV could be used:
    - `T: Trait<{ N + 1 }>` 
    - `T: Trait<{ <T as Trait<{ N + 1 }>>::ASSOC }>`
        - From the trait solvers POV `{ <T as Trait<{ N + 1 }>>::ASSOC }` is potentially normalizeable to `{ N + 1 }` in which case this *would* be a valid way of proving `T: Trait<{ N + 1 }>`
- Trying to determine if `{ N + 1 }` is equal to `{ <T as Trait<{ N + 1 }>>::ASSOC }` requires type checking both anon consts. As we are already type checking `{ N + 1 }` we have reached a cycle.

Attempting to represent type system constants through a lowering step *after* type checking therefore seems like a dead end to me as this is code that we should support ([equivalent code written with associated types](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=c15faa442ac33404cb63c1a84d272e45) compiles fine). 

### Unused generic parameters

The current desugaring can easily result in an anon const having more generics defined on it than are actually used by the body. Take the following example:

```rust
struct Foo<T, U, const N: usize> {
    arr: [u64; N + 1],
    ...
}

// desugars to:

const ANON<T, U, const N: usize>: usize = { N + 1 }
where
    terminates { ANON::<T, U, N> };
struct Foo<T, U, const N: usize>
where
    terminates { ANON::<T, U, N> }
{
    arr: [u64; ANON::<T, U, N>],
    ...
}
```

The type `[u64; ANON::<T, U, N>]` "uses" all of the generic parameters in scope which causes a number of issues:
- All generic parameters on the struct are invariant due to them all being used in a const alias
- Causes issues with object safety as an anon const would be considered to "use" the `Self` type
- Breaks with defaulted generic parameters as the anon const is considered to use all parameters even forward declared parameters

### Evaluating constants with where clauses that do not hold

Const eval should generally not be run on bodies that do not type check as doing so can easily cause ICEs. Attempting to change const eval to support running on bodies that do not type check would introduce a large amount of complexity to const eval machinery and is undesirable.

Unfortunately the current implementation of `generic_const_exprs` does not check that where clauses on anon consts hold before evaluating them. This results in const eval ICEing sometimes when using complex expressions with generic parameters in where clauses.

In theory it would be simple to ensure that the where clauses hold *before* evaluating a constant. In practice this would drastically worsen the previous issue about query cycles to the point where evaluating any anon const in a where clause would cause a query cycle.

This is caused by the fact that anon consts are desugared to having all of the where clauses in scope (similar to how they have more generics than is strictly necessary). This means that an anon const inside of a where clause, would be desugared to having that same where clause. Example:
```rust
fn foo<T, const N: usize>()
where
    T: Trait<{ N + 1 }>,
{}

// desugars to:

const ANON<T, const N: usize>: usize = { N + 1 }
where
    T: Trait<ANON::<T, N>>,
    terminates { ANON::<T, N> };
fn foo<T, const N: usize>()
where
    T: Trait<{ ANON::<T, N> }>,
    terminates { ANON::<T, N> },
```

Attempting to evaluate `ANON::<concrete, concrete>` would first require proving `terminates { ANON::<concrete, concrete> }`, which is proven by evaluating `ANON::<concrete, concrete>`. 

Any design for supporting complex expressions involving generic parameters needs to be able to only evaluate constants when they are wellformed.

## Can we fix `generic_const_exprs`?

Having highlighted the core issues with the current implementation of the feature it's worth trying to figure out if we can just solve these and then stabilize it. To summarize the requirements above we would wind up with:
1. How do we represent type system consts without requiring a  hir body that first need to be type checked
2. How to make sure constants only use the "right" amount of generic parameters
3. How do we make sure its possible that constants be checked for wellformedness before evaluating them
4. How to avoid evaluating constants that might not terminate
    - I am combining the two headers of "how to handle `N + 1` where `N=usize::MAX`" and "how to avoid hitting unrecoverable CTFE errors" as these wind up being incredibly similar problems

To start with we would have to stop using the current desugaring with creating const items- it's fundamentally incompatible with wanting to represent type system constants without requiring type checking a body first.

Instead we need to have a new kind of AST for representing expressions in the type system. There are a number of concerns here:
- We would have an entire second way of type checking expressions
- How do we handle evaluating this AST
- What expressions do we support? method calls such as `a.foo(N)` would add a significant amount of complexity to the type system which is undesirable, so we cant just say "all of them".

Regardless, this *would* solve issues 1, 2, and 3. The new desugaring would result in only syntactically used generic parameters being present in the AST. Additionally the minimal required where clauses to hold for the const argument to be wellformed can be derived from the AST.

For the next issue (4) I can see a couple possible solutions:
- Simply accept that the type system will be buggy and error on your code for little to no reason, and you have to work around it
- Introduce a simplistic termination checker to rust used by `generic_const_exprs` and only permit calls to functions that are statically known to terminate

Simply accepting a buggy type system implementation seems out of the question to me. It is not a good look for the language to decide to stabilize something with bugs without any plan for how they could ever be resolved.

As an additional point, if we accepted the buggy behaviour we would still need to decide how to handle `N + 1` as an argument when a concrete value of `usize::MAX` was provided for `N`. Post monomorphization errors are an "obvious" solution to this that *would* work from a technical standpoint.

Introducing a termination checker is a *significant* undertaking that would take a long time to fully work out the design of- let alone have it in a stabilizable state. I imagine this would take many years and also significantly increase the complexity of the language.

In conclusion while there is potentially a path here to having everything work, there is a large number of open questions and issues with those potential solutions. Regardless of what is decided it would potentially take years to get something into a stabilizable state, and that assumes that this will actually work out.

While `generic_const_exprs` itself might be a rather long way off, it should be possible to carve out a subset of the feature that *is* possible to get working nicely in a much shorter timeframe, e.g. a `min_generic_const_args` feature.

## What is `min_generic_const_args`

On stable there are limitations to what a const generic parameter can have as its argument, concretely this is:
- A single parameter `{ N }`
- An arbitrary expression that does not mention any generic parameters, e.g. `{ foo() + 10 / 2 }`

The `min_generic_const_args` will extend this to also support:
- A usage of an associated constant with arbitrary generic parameters, e.g. `<T as Trait<N>>::ASSOC` so long as the definition is marked `#[type_system_constant]`.

Similar to how uses of const parameters does not require explicit braces, the same is true of uses of associated constants. `Foo<<T as Trait>::ASSOC>` is legal without requiring it to be written as `FOO<{ <T as Trait>::ASSOC }>`. (maybe, might get cut, but it seems nice)

In order for an associated constant to be usable with generic parameters as an argument to a const generic parameter it must be marked with `#[type_system_constant]` on the trait definition. This attribute will require all trait implementations to specify a valid const generic argument as the value of the associated constant instead of an arbitrary unrestricted expression.

Example:
```rust
trait Trait {
    const ASSOC: usize;
}

impl<const N: usize> Trait for Foo<N> {
    const ASSOC: usize = N;
}

fn foo<const N: usize>() {}

fn bar<T: Trait>() {
    // ERROR: `Trait::ASSOC` is not marked as `#[type_system_constant]` and so
    // cannot be used in the type system as a const argument.
    foo::<<T as Trait>::ASSOC>();
}
```

If trait definition was rewritten to to use the `#[type_system_constant]` attribute then this would compile:
```rust
trait Trait {
    #[type_system_constant]
    const ASSOC: usize;
}
```

This restriction is required to ensure that implementors of `Trait` are not able to write arbitrary expressions as the value of the associated constant, instead a valid const generic argument must be written. For example:
```rust
impl<const N: usize> Trait for Foo<N> {
    // ERROR: `N + 1` is not a valid expression in the type system 
    const ASSOC: usize = N + 1;
}
```

## Does this actually fix the original problems?

Reusing the list of issues from earlier:

1. How do we represent type system consts without requiring a  hir body that first need to be type checked
2. How to make sure constants only use the "right" amount of generic parameters
3. How do we make sure its possible that constants be checked for wellformedness before evaluating them
4. How to avoid evaluating constants that might not terminate

The first problem is easy to solve with associated constants as they have an obvious representation, a `DefId` + list of generic args (the same as we do for associated types). This also does not introduce any of the complexities with trying to represent arbitrary expressions:
- Evaluating associated constants is trivial as the compiler already has the machinery for this
- There is no need to figure out how to type check or borrow check arbitrary type system expressions as associated constants have simple rules around wellformedness already present in the compiler, and they just inherently don't need borrow checking.

It is also trivial to tell what generic parameters a const argument uses as they are explicitly defined on the trait, the same as how associated types are handled in the compiler which has been shown to work adequately.

The requirement that all associated constants in the type system are defined as being equal to a valid const argument means that ensuring all constants in the type system terminate is as easy as it is under `min_const_generics`. An associated constant can either be assumed to terminate successfully or simply be evaluated and then error if that fails.

This also means that it is trivial to avoid evaluating constants that might not terminate as every const generic argument will have already been checked somewhere that it can be evaluated without issues, e.g. in the caller or in an associated const's definition.

Determining that const arguments are well formed before evaluating them is *also* trivial as we do not wind up with such recursive where clauses as all where clauses are explicitly written by the user on the `trait` or the associated const itself.

## Future extensions to `min_generic_const_args`

- Support struct/enum construction for when arbitrary types are allowed as const generics.
- Special case `size_of`/`align_of` to be permitted too?
- Experiment with permitting arbitrary expressions as described in [Can we fix generic_const_exprs](#can-we-fix-generic_const_exprs) or by allowing arbitrary expressions to be used for associated consts in the type system
- Introducing where clauses that allow bounding generic parameters, e.g. a hypothetical `where N is 1..`?
