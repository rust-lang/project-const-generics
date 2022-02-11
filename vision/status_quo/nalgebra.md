# 😱 Status quo: nalgebra

*a huge thanks to [Andreas Borgen Longva](https://github.com/Andlon) and [Sébastien Crozet](https://github.com/sebcrozet) for the help with figuring this out*

[nalgebra](https://nalgebra.org/) is a linear algebra library. At the core of that library is a type `struct Matrix<T, R, C, S>` where `T` is the components scalar type, `R` and `C` represents the number of rows and columns and `S` represents the type of the buffer containing the data.

Relevant for const generics are the parameters `R` and `C`. These are instantiated using one of the following types:
```rust
// For matrices of know size.
pub struct Const<const R: usize>;
// For matrices with a size only known at runtime.
pub struct Dynamic { value: usize }
```

The authors of nalgebra then introduce a type alias
```rust
pub struct ArrayStorage<T, const R: usize, const C: usize>(pub [[T; R]; C]);
/// A matrix of statically know size.
pub type SMatrix<T, const R: usize, const C: usize> =
    Matrix<T, Const<R>, Const<C>, ArrayStorage<T, R, C>>;
```

To deal with the lack of generic const expressions, they a trait for conversions from and to [`typenum`](https://crates.io/crates/typenum) for all `Const` up to size `127` ([source](https://github.com/dimforge/nalgebra/blob/39bb572557299a44093ea09daaff144fd6d9ea1f/src/base/dimension.rs#L273-L345)).

Whenever they now need some computation using `Const<N>`, they convert it to type nums, evaluate the computation using the trait system, and then convert the result back to some `Const<M>`.

## Disadvantages

While this mostly works fine, there are some disadvantages.

### Annoying `ToTypenum` bounds

Most notably this adds a lot of unnecessary bounds, consider the following impl:

```rust
impl<T, const R1: usize, const C1: usize, const R2: usize, const C2: usize>
    ReshapableStorage<T, Const<R1>, Const<C1>, Const<R2>, Const<C2>> for ArrayStorage<T, R1, C1>
where
    T: Scalar,
    Const<R1>: ToTypenum,
    Const<C1>: ToTypenum,
    Const<R2>: ToTypenum,
    Const<C2>: ToTypenum,
    <Const<R1> as ToTypenum>::Typenum: Mul<<Const<C1> as ToTypenum>::Typenum>,
    <Const<R2> as ToTypenum>::Typenum: Mul<
        <Const<C2> as ToTypenum>::Typenum,
        Output = typenum::Prod<
            <Const<R1> as ToTypenum>::Typenum,
            <Const<C1> as ToTypenum>::Typenum,
        >,
    >,
{
    type Output = ArrayStorage<T, R2, C2>;

    fn reshape_generic(self, _: Const<R2>, _: Const<C2>) -> Self::Output {
        unsafe {
            let data: [[T; R2]; C2] = mem::transmute_copy(&self.0);
            mem::forget(self.0);
            ArrayStorage(data)
        }
    }
}
```

### `ToTypenum` is only implemented up to fixed size

That's annoying. ✨

### Cannot use associated constants

It is currently also not possible to have the size of a matrix depend on associated constants:
```rust
trait MyDimensions {
   const ROWS: usize;
   const COLS: usize;
}

fn foo<Dims: MyDimensions>() {
    // Not possible!
    let matrix: SMatrix<f64, Dims::ROWS, Dims::COLS> = SMatrix::zeros();
}
```
While this can be avoided by going to back to `typenum` and using associated types, this adds a lot of unnecessary bounds and inpacts all of the code dealing with it.

## Wishlist

Ideally, `Matrix` could be changed to the following:

```rust
enum Dim {
    Const(usize),
    Dynamic,
}

struct Matrix<T, const R: Dim, const C: Dim, S> { ... }

type SMatrix<T, const R: usize, const C: usize> =
    Matrix<T, Dim::Const(R), Dim::Const(C), ArrayStorage<T, R, C>>;
```

For this to work well there have a bunch of requirements for const generics:

### User-defined types as const parameter types

We have to be able to use `Dim` as a const param type

### Consider injective expressions to bind generic params

With this change, `nalgebra` needs impls like the following

```rust
impl<T, const R: usize, const C: usize> for SMatrix<T, R, C> {
    // ...
}
```

For this impl to bind `R` and `C`, the expression `Dim::Const(N)` has to bind `N`.
This is sound as constructors are injective. It seems very desirable to at least
enable this for expressions using constructors.

### Merge partial impls to be exhaustive

By adding one trait impl impl for `Dim::Dynamic` and one for `Dim::Const(N)`, it should be enough to consider that trait to be implemented for all `Dim`.

Ideally, the compiler should figure this out by itself, or it can be emulated using specialization by manually adding an impl for all `Dim` which always gets overridden.

### Generic const expressions

For example when computing the [Kronecker product](https://en.wikipedia.org/wiki/Kronecker_product) which has the following simplified signature:
```rust
pub fn kronecker<T, const R1: Dim, const C1: Dim, const R2: Dim, const C2: Dim>(
    lhs: &Matrix<T, R1, C2>,
    rhs: &Matrix<T, R2, C2>,
) -> Matrix<T, R1 * R2, C1 * C2> {
    ...
}
```

For this generic const expressions have to be supported.

### const Trait implementations

For `R1 * R2` to work we need const trait impls, otherwise this
can be written using `mul_dim(R1, R2)` or something.

## `Default` for arrays

`nalgebra` currently has to work around `Default` not being implemented
for all arrays where `T: Default`.