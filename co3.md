---
layout: article
title: "CO3: Toward the Optimal FFI"
permalink: /co3/
---
# [CO3](https://github.com/mversic/co3): Toward the Optimal FFI

_When I first started thinking about FFI more than three years ago, I was riding boda-bodas through the streets of Nairobi._

<details class="article-toc" markdown="block">
<summary>On this page</summary>

* Contents
{:toc}

</details>

## Minimalism as the Guiding Aesthetic

At the time, I was working at Soramitsu, a company developing its own blockchain ledger. Not that I cared much about the field, but it provided me with the opportunity to write Rust while making a living.

It was my long-standing desire to enter the space of systems programming, and I knew I would never trust myself enough to write C++. More importantly, in my eyes, the code written in Rust held so much more elegance compared to its well-established precursors. The aesthetic of expression was always one of the primary drives in my life.

## Pay Attention to Minute Particulars

_For Art and Science cannot exist but in minutely organized Particulars, And not in generalizing Demonstrations of the Rational Power._\
— William Blake, [*Jerusalem*, Chapter 3, plate 55](https://blake.lib.asu.edu/html/jerusalem.html)

Imagine someone had told you there was work underway to add an extension to the Rust programming language that would make it possible to use **generics in FFI**. Not only that—imagine you were told it would be possible to **directly export your Rust code** across the FFI boundary without ever "feeling" the boundary. Strong claims, you would say. Moreover, who is this person, and why should you even listen to them?

I know I can't provide you with any satisfying credentials to my name, but I can let the excellence of the work speak for itself. I hope that, by the end of this article, I will have justified a strong claim that this *is not yet another FFI framework*, but a general model that **unifies all attempts to solve the bidirectional interfacing** and mapping of types between Rust and the C ABI. Specifically, via the same declaration and conversion model, not all of which are discussed in the article, `CO3` covers:

- **Imports and exports** of functions, statics, methods, and opaque types, with configurable ABIs and conditional compilation.
- **The Rust type space**: native sized and dynamically sized types, including those with `#[repr(Rust)]` layout.
- **Representation and validity**: size and alignment, niches, trap values, pointee validation, and custom invariants.
- **Value-passing semantics**: borrowing, cloning, ownership transfer, `#[soft]` conversions with mutable writeback.
- **Polymorphism**: static monomorphization and runtime tag dispatch, with customizable erasure and tag placement.
- **C ABI adaptation**: configurable symbol naming, compound-value unpacking, and configurable failure handling.

### 1. Abstractions Are Zero-Cost by Default

The library was sculpted around the following rules:

- You mustn't concern yourself with how to make your code **fit the framework**.
- You write only and exclusively **pure Rust code**, then export it when ready.
- The syntax of the framework must draw inspiration from *existing Rust idioms*.

Let's see how that looks in practice:

```rust
use co3::{ReprC, ffi};
use rust_spec::RustSpec;

type Header = [u32; 4];

#[derive(RustSpec, ReprC)]
#[repr(C)]
struct Packet<T> {
    header: [u32; 4],
    payload: T,
}

impl<T> Packet<T> {
    fn header(&self) -> &Header {
        &self.header
    }
}

// Declaration of exports
ffi! {
    #![unsafe(export("C"))]

    impl Packet<u32> {
        // `export_name` is inferred
        fn header(&self) -> &Header;
    }
}
```

### 2. The Type Mapping Space Is Complete

Using `CO3` must introduce as little friction as possible:

- **Each and every native Rust type** (with some caveats) must be usable in FFI.
- The framework must make it possible to declare **extern (i.e. opaque) types**.

```rust
use co3::{ReprC, ffi};
use rust_spec::RustSpec;

// An ordinary `repr(Rust)` struct
#[derive(Clone, Copy, RustSpec, ReprC)]
struct Options {
    verify_checksum: bool,
    max_payload_len: usize,
}

// A custom slice-tailed DST
#[derive(RustSpec, ReprC)]
#[repr(C)]
struct Packet<T> {
    id: u32,
    payload: [T],
}

// Declaration of imports
ffi! {
    #![unsafe(extern("C"))]

    // Opaque extern type
    type Context;

    #[symbol_name = "lib_context"]
    fn context() -> &'static Context;

    #[symbol_name = "lib_next_packet"]
    fn next_packet(context: &Context) -> &Packet<u8>;

    #[symbol_name = "lib_inspect"]
    fn inspect(
        packet: &Packet<u8>,
        context: &Context,
        options: Options,
    ) -> bool;
}

fn main() {
    let context = context();
    let packet = next_packet(context);
    let accepted = inspect(
        packet,
        context,
        Options {
            verify_checksum: true,
            max_payload_len: 4096,
        },
    );

    println!("packet accepted: {accepted}");
}
```

### 3. Soundness Is Not a Compromise

As far as `CO3` is concerned, the default API **MUST** enforce all soundness guarantees within its control, even at the expense of the zero-cost abstraction principle. The big caveat is that this guarantee rests on the assumption that unsafe FFI declarations accurately describe the foreign implementation. Opt-in conversion modes make the costs of preserving soundness, and any additional guarantees, explicit. **Therefore:**

- Values are lowered to robust C-compatible representations and checked for trap values (including pointees).
- **Ownership transfer is opt-in**: owned values are borrowed and cloned on import unless marked `move`.
- References that cannot be safely converted in place require an explicit `#[soft]` opt-in.

```rust
use co3::ffi;

fn toggle(value: &mut bool) {
    *value = !*value;
}

fn round_trip(bytes: Vec<u8>) -> Vec<u8> {
    bytes
}

ffi! {
    #![unsafe(export("C"))]

    // Convert `&mut bool` through temporary local storage and synchronize changes after the call.
    //
    // If passed directly, a mutable reference to a non-robust pointee could be set to a trap value
    // by a non-conforming caller. That is why this conversion—to ensure soundness—violates the
    // zero-cost principle. `#[soft]` opts into temporary storage, cloning, and writeback while
    // preserving value validity, but not pointer identity.
    fn toggle(#[soft] value: &mut bool);

    // Both the argument and return value transfer ownership.
    //
    // Without moving the argument, `CO3` assumes the caller has not passed ownership, decodes
    // the argument as a reference `&[u8]`, and clones it into memory managed by the local allocator.
    //
    // Without moving the return value, the export would fail to compile
    // (it's not possible to return a reference to temporary storage).
    move fn round_trip(move bytes: Vec<u8>) -> Vec<u8>;
}
```

## The Frontier of Language

_The evolution of language is the evolution of reality._\
— Terence McKenna, [*Shamanology*](https://www.organism.earth/library/document/shamanology)

As everyone knows, Rust generics do not exist at the C ABI.

`CO3`, therefore, provides a language extension, implemented as a procedural macro, with a highly ergonomic vocabulary that supports both **static parameter monomorphization** and **runtime tag-dispatched parameters**. The following are simple examples of these language extensions that allow both static monomorphization and runtime dynamic dispatch across the FFI boundary:

### 1. Runtime-Tagged Dispatch

As the true aficionados of FFI among you know, C APIs commonly use runtime-tagged dispatch: a function receives both an erased handle to a value and a tag that identifies its concrete type. On import, CO3 passes the erased representation together with its tag. On the export side, this desugars to a match on the tag and a corresponding reinterpretation to the type identified by the tag.

The crazy part is that the dispatch is **checked at compile time** and exposed through an **entirely safe Rust interface**. Declare it with `<dyn(TagType) T = ErasedType>` and select valid choices with `where use<T, ..> @ (<ConcreteTy1, ..> | ..)`:

```rust
use co3::ffi;

trait Decoder {
    fn byte_len(&self) -> usize;
}

ffi! {
    #![unsafe(extern("C"))]
    #![symbol_prefix = "image"]

    #[tag(u8, unsafe(1))]
    type PngDecoder;

    #[tag(u8, unsafe(2))]
    type JpegDecoder;

    fn png_decoder() -> &'static PngDecoder;
    fn jpeg_decoder() -> &'static JpegDecoder;

    // `T` is a runtime tag-dispatched parameter erased as `c_void` behind the receiver pointer.
    // A custom erased representation can be chosen with the `<dyn(u8) T = ErasedType>` syntax.
    impl<dyn(u8) T> Decoder for T
    where
        // Constrain the space of concrete types available for selecting `T`.
        use<T> @ (<PngDecoder> | <JpegDecoder>),
    {
        // By default, dynamic parameter tag is injected at the beginning of the ABI argument list.
        //
        // For tighter control, the `<dyn T>::TAG` can be placed explicitly at the desired position:
        //     `fn byte_len(tag: `<dyn T>::TAG`, &self)`
        #[symbol_name = "image_decoder_byte_len"]
        fn byte_len(&self) -> usize;
    }
}

fn main() {
    let png = png_decoder();
    let jpeg = jpeg_decoder();

    println!("PNG bytes: {}", png.byte_len());
    println!("JPEG bytes: {}", jpeg.byte_len());
}
```

### 2. Interpolating Static Parameters

An ordinary parameter such as `<T>` undergoes a regular static dispatch. Every selected type produces a monomorphized wrapper and therefore needs **a distinct C symbol**.

By default, `CO3` generates a unique name based on the defined symbol naming scheme, which I find is only potentially useful on exports. Existing C libraries rarely follow the default naming scheme, so a parameter can be interpolated manually into an explicit symbol via `#[symbol_name = "MyName{T}"]` as follows:

```rust
use co3::ffi;

struct Crc32;
struct XxHash3;

ffi! {
    #![unsafe(extern("C"))]

    // Define the symbol fragment associated with each algorithm.
    #![symbol_fragments {
        Crc32 = "crc32",
        XxHash3 = "xxh3",
    }]

    #[symbol_name = "image_hash_{Algorithm}"]
    fn hash<Algorithm>(bytes: &[u8]) -> u64
    where
        use<Algorithm> @ (<Crc32> | <XxHash3>);
}

fn main() {
    let image = std::fs::read("image.png").unwrap();

    let checksum = hash::<Crc32>(&image);
    let fast_hash = hash::<XxHash3>(&image);

    println!("CRC-32: {checksum:08x}");
    println!("XXH3: {fast_hash:016x}");
}
```

## When Your Hogwarts Letter Never Comes

_… but you are a wizard._

The crux of the solution underpinning this library became evident to me after a month or two of wrestling with the problem space. However, I had completely underestimated the effort required to discover its sharp outlines.

The idea lay dormant for two years before I finally decided to commit myself (*my time, my resources, and, most importantly, my life*) to implementing it. One year and three existential crises later, I was ready to present my work.

To bring it to life, I had to solve two precursor problems, which I have published as their own crates as well:

### 1. [disjoint_impls](https://crates.io/crates/disjoint_impls)

As it stands, even now, Rust's trait solver can't figure out that impls don't overlap when disjoint over associated types. Initially, I dealt with this shortcoming through obfuscated trait magic written out in source code… **BY HAND**. At times, I would find myself looking at this incomprehensible mush, just trying to catch that glimmer of an unravelling thread of understanding.

I had to write a proc-macro. As most of you must still remember, back in the day, LLMs were not as useful, and writing proc-macros was always a drag. I had the crate released in under 3 months of lax side work. However, arriving at the *complete solution* would take me several more months of dedicated work.

```rust
use disjoint_impls::disjoint_impls;

pub trait Dispatch {
    type Group;
}

disjoint_impls! {
    pub trait Dispatched {}

    impl<T: Dispatch<Group = u32>> Dispatched for T {}
    impl<T: Dispatch<Group = i32>> Dispatched for T {}
}
```

### 2. [rust-spec](https://crates.io/crates/rust-spec)

Next, I had to invent a compile-time language, expressed through associated types, capable of describing the type categories relevant to FFI. After many iterations, I found that Rust types could be classified along the following five primary public axes:

```rust
unsafe trait RustSpec {
    type Layout;    // Stable | Unstable
    type Size;      // Sized<Zero | Gt<Zero>> | MetaSized<SliceLike | DynTraitLike> | ExternTypeLike
    type Alignment; // One | Gt<One>
    type Trap;      // Robust | NonRobust
    type Niche;     // WithoutNiche | WithNiche<Stable | Unstable>
}
```

I would then use the categorization to precisely describe the type's lowering into its C-compatible representation.

## Ready for the Real World

Looking at it as a whole, `CO3` should **not be considered a prototype or a proof of concept**. Even if there are a few edges to smooth out, as there are with any piece of software, you will find it is a heavily tested, **ready-for-real-world-use** library. All it needs now is **wider adoption**.

If you are interested in what using `CO3` looks like in practice, I refer you to [rs-odbc](https://github.com/mversic/rs-odbc), which showcases the full-blown expressive power of this crate in a concrete, useful, and working library.

### What Follows?

`CO3` is a big, but not the only, part of a broader FFI ecosystem. Generating C headers remains the domain of `cbindgen` and `cheadergen` and integrating them with `CO3` is the natural next step.

I want to give a special thanks to Quan Hao Ng, whom I mentored as part of the `Linux Foundation Mentorship` as he implemented the associated-type support in `cbindgen` this integration requires. Unfortunately, that work has remained unattended by the maintainer for years despite repeated follow-ups.

## The Eschaton

_... in the heart of a universe prolonged along its axis of complexity, there exists a divine center of convergence ... the point Omega._\
— Pierre Teilhard de Chardin, [*Life and the Planets*](https://www.organism.earth/library/document/life-and-the-planets)

This work has taken incredible sacrifices, not least of which was more than a year of unemployment and all the stress related to living without an income. During this time, I witnessed firsthand AI technology advance rapidly from a preschool level to a PhD level, not only as a coder but as a deeply nuanced thinker. We can now indeed feel the tug of singularity.

Standing at this point in time and history, we are all finally coming to terms with what the full power of gradient descent, given sufficient scale and data, really is. As shocking and unbelievable as it is, don't fool yourself into thinking this is not happening or is somewhere far in the future. Extrapolate the trendlines now, **learn what OOM is**, and try to truly comprehend how an exponential takeoff is produced through compounding progress. For all practical purposes and human affairs, **the singularity is just around the corner**.

The transformation that is underway right now will reshape the very essence of what it means to be a human. Therefore, this is a good point to ask ourselves: Did our lives make sense? Did we make the best choices? Did we make the right offerings and the required sacrifices? **What is it that matters to us the most?**

## Gratitude

Before we close, I want to express my **deepest and most humble gratitude** to my wife for all the support and patience and for bringing colour and deeper meaning to my life.

Further, I want to express appreciation for my sister, my mother, my late father, and my family and friends for all the wonderful, and sometimes not-so-wonderful, times we had together. Furthermore, I am grateful to my teachers, my neighbours, my employers and business associates, even my foes and enemies, and, in general, anyone I have ever crossed paths with.

A truly creative work draws inspiration from and can only come from **the totality of one's lived experience**.

### When We Free Ourselves

_We are not freed into a void, we are freed into the dimension in which art is an obligation._\
— Terence McKenna, [*Dreaming Awake at the End of Time*](https://www.organism.earth/library/document/dreaming-awake-at-the-end-of-time)

As all of us are aware, while living inside an economy, it is difficult to provide meaningful work of great depth and beauty that is devoid of all financial incentives. If you appreciate the effort and find the work, if not foundational, at least valuable, consider **providing a small donation**.

#### The Fine Print

For legal reasons, I am obliged to make the following painfully explicit: the library is **freely available under its open-source license** regardless of whether you provide financial support.

Any payment is **entirely voluntary**, is not payment for the development of the software, and **does not entitle the donor** to support, maintenance, consulting, feature development, additional licensing rights, advertising, priority treatment, or any other goods or services. A donation **does not create any obligation** to continue developing, maintaining, or supporting the library.
