# Skill tree for const generics features

```skill-tree
[[group]]
name = "min_cg"
label = "MIN CONST GENERICS"
description = ["Minimum subset of const generics only allowing `const N: {integer} | bool | char` and fully concrete constants"]
href = "https://github.com/rust-lang/rust/pull/79135"
items = []

[[group]]
name = "valtree"
label = "VALTREES"
description = ["Use valtrees in all type level constants"]
items = [
    { label = "Design of structural equality is worked out" },
    { label = "Valtrees are implemented" },
]

[[group]]
name = "min_defaults"
label = "CONCRETE CONST DEFAULTS"
description = ["Permit concrete defaults on traits/adts"]
items = []
requires = ["min_cg"]

[[group]]
name = "generic_defaults"
label = "GENERIC CONST DEFAULTS"
description = ["Permit generic defaults on traits/adts"]
items = [
    { label = "Disallow usage of forward declared params via param env" },
    { label = "Decide when a generic default is considered well formed" },
]
requires = ["min_defaults"]

[[group]]
name = "param_tys"
label = "MORE CONST PARAM TYPES"
description = ["Permit more types to be as the type of a const generic"]
items = [
    { label = "A `Constable` that that signifies a type is valid to use as a const param type" },
    { label = "Generic const param types i.e. `T: Constable, const N: T`" },
    { label = "Consider `impl&lt;const N: usize&gt; Trait&lt;{ Some(N) }&gt; for ()` to constrain `N`" },
]
requires = ["valtree", "min_cg"]

[[group]]
name = "exhaustiveness"
label = "IMPL EXHAUSTIVENESS"
description = ["Allow separate impls to be used to fulfill `for&lt;const N: usize&gt; (): Trait&lt;N&gt;`"]
items = []
requires = ["param_tys"]

[[group]]
name = "min_generic_consts"
label = "MIN GENERIC CONSTANTS"
description = ["Allow type level constants to be generic i.e. `N + 1`"]
items = [
    { label = "Add a where clause that requires a given expression to be evaluatable e.g. `where evaluatable { N - 1 }`" },
    { label = "Unused substs make it hard to tell if a const is concrete or generic and breaks unsize coercion in some cases" },
    { label = "Don't eagerly error when evaluating constants during selection" },
]
requires = ["min_cg"]

[[group]]
name = "generic_consts"
label = "GENERIC CONSTANTS"
description = ["Allow type level constants but better ✨"]
items = [
    { label = "Allow writing a where clause that requires a condition to hold e.g. `where { N &gt; 2 }`" },
    { label = "Some things can always evaluate and should not need a where clause e.g. `{ N == M }` `{ N / 1 }`" },
    { label = "If possible `where { N &gt; 0 }` should imply `N - 1` is evaluatable" },
]
requires = ["min_generic_consts"]

[[group]]
name = "assoc_const_bounds"
label = "ASSOCIATED CONST BOUNDS"
description = ["Permit where `where T: Trait&lt;ASSOC = 10&gt;`"]
items = []
requires = ["min_generic_consts"]

[[group]]
name = "arg_infer"
label = "GENERIC ARG CONST INFER"
description = ["Permit using `_` for generic args"]
href = "https://github.com/rust-lang/rust/pull/83484"
items = []
requires = ["min_cg"]
```
