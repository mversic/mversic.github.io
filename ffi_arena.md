# FFI Arena: Is UniFFI still the Top Dog?
_Can we objectively tell which framework is the best?_

## Contestants

This article compares the following FFI frameworks:

* [uniffi](https://github.com/mozilla/uniffi-rs)
* [boltffi](https://github.com/boltffi/boltffi)
* [safer_ffi](https://github.com/getditto/safer_ffi)
* [Diplomat](https://github.com/rust-diplomat/diplomat)
* [Interoptopus](https://github.com/ralfbiedert/interoptopus)
* [CO3](https://github.com/mversic/co3)

## Syntactic vs semantic type resolution

We'll begin with a simple example:

```rust
pub type Buffer = [u8];

#[cfg_attr(feature = "uniffi", uniffi::export)]
#[cfg_attr(feature = "boltffi", boltffi::export)]
#[cfg_attr(feature = "interoptopus", interoptopus::ffi)]
#[cfg_attr(feature = "safer_ffi", safer_ffi::ffi_export)]
pub fn identity(input: &Buffer) -> &Buffer {
    input
}

#[cfg(feature = "diplomat")]
#[diplomat::bridge]
pub mod diplomat_case {
    use super::Buffer;

    pub fn identity(input: &Buffer) -> &Buffer {
        super::identity(input)
    }
}

#[cfg(feature = "co3")]
co3::ffi! {
    #![unsafe(export("C"))]

    fn identity(input: &Buffer) -> &Buffer;
}
```

**All fail except `CO3`**.

This example is the main thing that sets `CO3` apart from all other FFI frameworks. Where all frameworks resort to syntactical type resolution (based on the name of the type), **`CO3` resolves types semantically** (using the Rust compiler). It's important that this sinks in.

## Beating the drum

It doesn't matter if other frameworks are able to produce another syntactical rule under which this example compiles. Their design is structurally weaker and there'll always be cases that won't compile. Closing this gap would require a complete restructuring of the framwork's type resolution.

So how's semantic type resolution even possible? Well, I talked about this previously in the [release article](https://mversic.github.io/co3/). In short, `CO3` categorizes Rust types according to their relevant properties (with [`rust-spec`](https://github.com/mversic/rust-spec)) and then dispatches implementations based on those properties with [`disjoint_impls`](https://github.com/mversic/disjoint_impls).

How `CO3` does its magic boils down to the following simplified example:

```rust
use disjoint_impls::disjoint_impls;
use rust_spec::RustSpec;

disjoint_impls! {
    pub trait ReprC {
        type CType: CType;
    }

    impl<'a, R: ReprC> ReprC for &'a R
    where
        R: RustSpec<Size: Thin>,
    {
        type CType = Ref<'a, R::CType>;
    }

    impl<'a, R: ReprC + ?Sized> ReprC for &'a R
    where
        R: RustSpec<Size = MetaSized<SliceLike>>,
    {
        type CType = WideRef<'a, R::CType>;
    }
}
```

Implementing `RustSpec` describes a type in an n-dimensional space whose axes are properties such as `Layout`, `Size`, etc. By placing mutually exclusive bounds on those associated types, the `ReprC` implementations for `&R` each target a disjoint region of that space.

Expanding on this principle, `CO3` can precisely target any specific type family at compile-time.

## House of Cards

As a consequence of the afformentioned type-resolution design, other compared frameworks also struggle with several other things. Without going into too much detail:

* **Niche optimization** — `CO3` passes `Option<bool>` in one byte, others keep the discriminant tag.
* **Foreign writes through `&mut T`** — By default, `CO3` checks a foreign write for invalid values.
* **Interior mutability** — `CO3` alone accepts `&UnsafeCell<u32>` as an exported function parameter.
* **Custom DSTs** — `CO3` alone supports custom struct definitions with a DST tail. Others reject it

The full comparison can be found [here](https://github.com/mversic/co3/blob/main/comparisons/README.md).

## Binding existing C symbols

Of the six frameworks, **only `CO3` can bind functions from an existing C library**.

`CO3` does this through the same conversion API that is used for exports. You declare the Rust signature in an `ffi!` import and name the C symbol with `#[symbol_name]`. `CO3` handles the argument and return value conversions. Following the previous example that would look simply as follows:

```rust
pub type Buffer = [u8];

#[cfg(feature = "co3")]
co3::ffi! {
    #![unsafe(extern("C"))]    // Notice the switch to `extern`

    fn identity(input: &Buffer) -> &Buffer;
}
```

I would be remiss not to mention that `Interoptopus` also lets Rust call foreign code through its [plugin interface](https://interoptopus.rs/constructs/reverse-interop/). With `plugin!`, you define the interface in Rust and load a foreign plugin that implements it. This, however, does not enable import of existing C bindings.

## Stable Intermediate Representation

SIR stands between the bindings exported by `CO3` and foreign binding generators. It is a specification that tells each generator how to **rebuild the exact exported Rust API** in its target language.

`CO3` lowers exported Rust declarations to a stable ABI while preserving all relevant type-system information. Each language community can **write its own generator independently** by following the [SIR specification](https://github.com/mversic/co3/blob/main/SIR.md).

**Rust-to-Rust FFI already works.** Implementing a C header generator and binding generators for other languages is still future work.

## Why should you care?

I can keep pointing out advantages of using `CO3`, but so what?

Personally, what brought me to `Rust` as an engineer, is the certain aura about the language and its aesthetic. Particularly, what moves me to the edge of my chair are examples of compile-time type-system mastery that go beyond what is conceivable.

I want to believe that this is the heart of this community. We don't want solutions that work in some narrow context. Rather, we take particular joy of doing things in the **best way imaginable**.

_Imagination is the true frontier_
