# ✨ Shiny future story: Image type in rust-gpu
Barbara is working on rust-gpu. In that project, she has a struct `Image` that represents GPU images. There are a number of constant parameters allowing this type to be heavily customized in a number of ways. In some cases, helper methods are only available for images with particular formats. She writes the struct declaration:

```rust
struct Image<
    SampledType: SampleType<FORMAT>,
    const DIM: Dimensionality,
    const DEPTH: ImageDepth,
    const ARRAYED: Arrayed,
    const MULTISAMPLED: Multisampled,
    const SAMPLED: Sampled,
    const FORMAT: ImageFormat,
    const ACCESS_QUALIFIER: Option<AccessQualifier>,
>(PhantomData<SampledType>);
```

Barbara gets a few compile errors about her types used as a const param not implementing `StructuralEq` so she derives that and moves on.
She then wants to write some methods that only work for images in some specific formats:

```rust
impl<
    SampledType: SampleType<FORMAT>,
    const DIM: Dimensionality,
    const DEPTH: ImageDepth,
    const ARRAYED: Arrayed,
    const MULTISAMPLED: Multisampled,
    const SAMPLED: Sampled,
    const FORMAT: ImageFormat,
    const ACCESS_QUALIFIER: Option<AccessQualifier>,
> Image<SampledType, DIM, DEPTH, ARRAYED, MULTISAMPLED, SAMPLED, FORMAT, ACCESS_QUALIFIER> {
    pub fn example_method(/* snip */)
    where
        { some_condition(DEPTH, MULTISAMPLED) }
    { /* snip */ }
}

const fn some_condition(a: ImageDepth, m: Multisampled) -> bool {
    match (a, m) {
        /* snip */
    }
}
```

This compiles just fine and Barbara moves on to more complicated things 