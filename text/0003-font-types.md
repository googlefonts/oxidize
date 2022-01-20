- 2022-01-17
- status: proposal
- author: @cmyr
- PR: https://github.com/googlefonts/oxidize/pull/3

## A foundation for building font tooling in Rust


# Overview

We propose a Rust library (crate) for zero-allocation access and direct manipulation
of font tables and types, leveraging Rust’s procedural macro system. This
library is intended to be an opinion-free transparent representation of the types
in the specification, to serve as a foundational component for anything that
needs to read or write font files. An additional goal is to provide a common set
of types to represent common data formats (tags, glyph identifiers, etc) that
can be used throughout the broader ecosystem.


# Motivation

This is intended to answer some of the questions outlined in this repository's
README. In particular, we aim to determine whether the following goals are feasible:

- using macros to describe the shape of tables
- the same codegen tools should produce types suitable for both zero-allocation,
  read-only table definitions as well for creating and modifying new tables

## additional goals

- A simple, clean API for parsing: this should be as good as a hand-written API
  such as in [pinot][].
- A *reasonably* clean API for authoring: we don't expect this API to be used
  directly very often; most of the time there will likely be a higher-level
  compiler abstraction that will call into it. That said, it should be totally
  expressive (complete control over things like formats for subtables that
  support multiple formats) and should prioritize simplicity and efficiency over
  ergonomics.
- suitable for prototyping & extensibility, i.e. it should be easy to add a new
  table type for a project without needing to patch the core crate or maintain a
  fork.

# Design

The particulars of the design will be determined through this exploration, but a
few core ideas are clear. For the read-only case, the design is largely borrowed
directly from [pinot][], with the main difference being using macros for code
generation, and that approach has been successful for [fonttools-rs][].

## conversion to and from big-endian bytes

[Values used][data types] in the binary tables are big-endian and may not be aligned to
word boundaries, and so have to be converted to the appropriate scalars when
read (In this sense a font library can never be exactly 'zero-copy', although it
can be zero-allocation, and support random access.)

Scalar values that can be decoded (and later encoded) from raw font bytes will
be represented by a trait, something like:

```rust
trait FromFontBytes: Sized {
    /// Number of bytes required to represent this type
    const SIZE: usize = std::mem::size_of::<Self>();

    fn from_be_bytes(bytes: &[u8]) -> Result<Self, Error>;
}

impl FromFontBytes for u8 {
    fn from_be_bytes(bytes: &[u8]) -> Result<Self, Error> {
        bytes.first().copied().ok_or(Error::Eof)
    }
}
```

This trait can then also be derived for types that are entirely composed of
scalar values that implement this trait:

```rust
#[derive(Debug, Clone, FromFontBytes)]
#[repr(C)] // ensure field layout matches declaration order
struct AxisValueMap {
    from_coordinate: F2dot14,
    to_coordinate: F2dot14,
}
```

## zero-allocation views into bytes & random access

### `Buffer`

When reading font data, the library will perform no allocation, and instead
provide 'views' for accessing the different tables and their constituent
elements. This API will be based around a `Buffer` type, which is an abstraction
over some sequence of bytes. The actual storage of the those bytes is hidden
from the user, but it might be some bytes loaded from disk, a memory-mapped
file, or an owned buffer.

This type will need to have an explicit lifetime parameter, since in the case
where the bytes are not owned we need to ensure that the referenced data is
valid for as long as the `Buffer` exists.

The `Buffer` type will provide a simple API for reading raw scalars from the
underlying bytes, for creating new `Buffer` objects for sub-ranges of the bytes,
and for otherwise navigating the byte stream. This API will largely be adapted
from the existing implementation of [`pinot::Buffer`].

```rust
impl<'a> Buffer<'a> {
    /// Attempt to read a scalar of type `T` at the provided offset.
    fn read<T: FromFontBytes>(&self, offset: usize) -> Result<T, Error> {
        self.bytes().get(offset..).and_then(T::from_be_bytes).ok_or(Error::Eof)
    }
}
```

### `Slice`

The `Slice` type will be used to represent arrays of items in the byte stream,
allowing for random access. It will provide a safe interface for random access
by performing bounds checking, and ensuring (by construction) that the inner
buffer has the correct length for the number of elements.

```rust
// in buffer
fn slice<T: FromFontBytes>(&self, offset: usize, len: usize) -> Result<Slice<T>, Error> {
    let len_bytes = len * T::SIZE;
    let child_buf = self.slice_buf(offset..offset + len_bytes).ok_or(Error::Eof)?;
    Ok(Slice::from_buffer(child_buf, len))
}

impl<T: FromFontBytes> Slice<'_, T> {
    fn get(&self, idx: usize) -> Result<T, Error> {
        let offset = idx * T::SIZE;
        self.buffer.read(offset)
    }
}
```

### Schemas / mapping types

On top of all this, each table, subtable or other non-scalar type will be
represented as a wrapper around a `Buffer`, with methods on the type encoding
the types and offsets of various fields and subtables. This will be illustrated
in more detail below.

## Procedural Macros

The crate will rely heavily on procedural macros to generate the code for
representing and accessing font tables. As an illustration, defining a simple
table might look something like,


```rust
font_types!(
    SegmentMaps {
       axis_value_maps: CountedSlice(u16, AxisValueMap),
    }

    Avar {
        version_major: u16,
        version_minor: u16,
        Padding(u16),
        segment_maps: CountedSlice(u16, SegmentMaps),
    }
);
```

This would generate code looking something like,

```rust
struct SegmentMaps<'a>(Buffer<'a>);

struct Avar<'a>(Buffer<'a>);

impl<'a> SegmentMaps<'a> {
    fn axis_value_maps(&self) -> Slice<'a, AxisValueMap> { .. }
}

impl<'a> Avar<'a> {
    fn version_major(&self) -> Result<u16, Error> {
        self.buffer.read(0)
    }

    fn version_major(&self) -> Result<u16, Error> {
        self.buffer.read(2)
    }

    fn segment_maps(&self) -> Result<Slice<'a, SegmentMaps>, Error> {
        let len: u16 = self.buffer.read(6);
        self.buffer.slice(8, len)
    }
}
```

In any case, the basic idea here is that you describe the ‘shape’ of a table
using standard Rust types, and then add annotations that provide additional
hints as to how the table data should be interpreted. The `CountedSlice`
annotation, for instance, instructs the macro to read a count from the buffer at
this location, and use that as the length of the subsequent field.

## Mutation & creation

So far, we have focused principally on the read-only case. For the case where we
are creating or mutating fonts, things are a bit less clear.

Currently, I can see two (perhaps two-and-a-half) approaches: explicit builders,
and plain-old-data-structures.

In either case, we will have a trait for encoding bytes, which will work
similarly the trait for decoding:

```rust
//something like:
trait ToFontBytes: FromFontBytes {
    // signature tbd
    fn write(&self, to_buf: &mut [u8]) -> Result<(), Error>;
}
```

## Explicit builders / constructors

In this model, types would all be immutable, and the same types described above
would be the types constructed when creating a font. Each construction of a
non-scalar type would allocate storage for that type, and would encode the bytes
at construction time. After construction, they would be identical to the types
used during parsing. This might look something like:

```rust
fn make_avar() -> Avar<'static> {
    // this is just trying to get a feel for possible API, we have lots
    // of choice around how we create
    let axis_values: Slice<_> = [(-1.0, -1.0), (0., 0.), (0.12, 0.14)]
        .into_iter()
        .map(AxisValueMap::from_floats)
        .collect();

    let segment_maps = Slice::from([axis_values]);
    let version_major = 1;
    let version_minor = 0;
    Avar::new(version_major, version_minor, segment_maps)
}
```

This doesn't feel too bad: it can use common Rust idioms and patterns, and is
not exceptionally verbose. It does have some drawbacks, though: when creating
objects, they need to be fully described, which makes it verbose, and there
isn't real mutation; you cannot create a `Slice` and then add or remove items
afterwards.

It also requires some intermediate allocation: for instance in this example
storage is allocated for the first `Slice`, those bytes are copied into the
second `Slice`, and then the whole thing is finally copied into the final table.

There are various ways we could reduce the amount of copying, but they all come
at the cost of *some* added API complexity.

> **Q**: At what granularity should we think about copying costs? For instance,
> I suspect that we do not want to copy bytes from tables that are unmodified.
> Do we want to extend this to not copying bytes from individual subtables that
> are unmodified, if other parts of a table are? Entries in subtables?

The basic issue here is that if our main types are all just wrappers around raw
bytes, we expect all of the components of a given object to be encoded
contiguously; there is no way to say that "this subtable over here is in
allocation A, and that one is reused from allocation B"; and each case where we
want to handle that pattern will require some new type. At the very least we can
imagine a special type for constructing a top-level font object (a header + a
list of tables), where each table has its own `Buffer` object that may reuse
storage from elsewhere; and we can then stream those bytes to disk or over the
network without any additional storage. Doing this for each individual table,
however, adds additional complexity.

## Plain old data structures

Another very compelling alternative is to just use plain old Rust data
structures.

In this world, our macros generate *two* types for each 'shape' that we define.
One of these is the type that we've already seen, (methods for accessing data
directly from underlying bytes) but the other one is just a plain old Rust
struct.

We can bikeshed names later; for the purpose of this illustration let's adopt
the pattern of naming these new types objects `${TYPE}Owned` where `${TYPE}` is
the name of the previously defined object. So continuing our `avar` example, we
get:

```rust
// identical to above
struct Avar<'a>(..);
struct SegmentMaps<'a>(..);

struct AvarOwned {
    pub version_major: u16,
    pub version_minor: u16,
    pub segment_maps: Vec<SegmentMapsOwned>,
}

struct SegmentMapsOwned(pub Vec<AxisValueMap>);
```

All of these 'Owned' types would be plain Rust structs (and likely enums)
generated by the macros, and they would also get generated code for writing them
back out to bytes.

This has the advantage of being a trivially easy API without any visible
fanciness. You would describe your tables using simple rust types, using `std`
APIs to mutate values, modify collections, or otherwise manipulate them. When
you were happy with your representation, you would encode once, and then have
the valid binary representation of your table (and you could still reuse the
data structure, for instance modifying it further and writing out a different
version).

The downside of this is that there are now two types for each non-scalar
structure in the spec; and additionally you are holding on to two
representations of the data. In practice I don't mind this too much. One way to
think about this is that these data structures are basically just another form
of builder, that happen to be reusable and that maintain some intermediate
state.

### relationships between these two types

Initially, it may be useful to think of these two sets of types as basically
independent of one another, but ultimately we will want some mechanisms for
moving between them.

I think for the time being we can keep this simple, though: we can have simple
equivalents of the `From` and `Into` traits. If we have an existing table that
we want to modify, we can convert it into the decoded form, modify it
arbitrarily, and then re-encode it.

## Fancier options

My personal current preference is for the second option of creating two versions
of each table; my hunch is that this will end up being a much simpler API in the
general case. In most cases, the user will be using one of these two sets of
types, and in the advanced cases (e.g. server-side subsetting, instancing
variable fonts) it still shouldn't be that complex.

If it turns out that this approach does not meet our performance
needs in certain special circumstances, we can consider having additional API
for these on a per-case basis.

## Extensibility

A final element of the design I would like to highlight is extensibility. The
basic idea here is that tables would not need to all be defined in a single
canonical place. If a user wanted to add support for a new, non-standard table,
all of the macros required to describe it would be available publicly, and they
would only need to write out and annotate the structs.

# Prior art

There are a number of existing projects that I think are relevant to this work,
and which encourage me as to its feasibility. The first of these is
[fonttools-rs][], and its components
otspec and otspec-macros. This is a set of crates by Simon Cozens that are
focused on the compilation case (data is stored in native rust types and
converted to raw bytes during serialization and deserialization) but it uses a
system of macros to generate that IO code, to good effect. The second of these
is Chad Brokow’s [pinot][], which is an existing
zero-copy OpenType parser intended to be used for shaping.

Between these two projects, the majority of the goals i have for this project
have already been demonstrated to be feasible; the challenge here will be
largely to figure out a way to combine them in a sensible and ergonomic way.

Outside of Rust, I think it will also be important to spend some quality time
with [HarfBuzz][], to make sure there are not requirements of that use-case that I
am overlooking; and of course [fonttools][] is the current choice for font
compilation, and related tasks.


# Schedule and time allocation

The initial goal for this work is to determine whether it is feasible at all.
With that in mind, I think we can break the project up into three phases, where
we can abandon the work if any particular phase is a failure:


1. macros, zero-copy types: demonstrate that we can use macros to generate code
   for interpreting and accessing values from a small handful of tables,
   including some tricky tables like GPOS/GSUB. 4-6 weeks.
2. mutability: decide on a strategy for mutability, and implement it for that
   same handful of tables. 6-10 weeks. It is possible that as we work on
   mutability we will decide that we need to revisit how we structure our
   code-gen code: for instance if we need to generate multiple types for each
   table, a procedural macro may not be the best choice.
3. Implement all core tables: ideally this is mostly just a matter of typing,
   but it might still be quite a lot of typing. I’ll feel more comfortable
   giving this a time estimate after I’ve finished the previous items, but if I
   had to guess I would say ~12 weeks.


# Questions

There are a few remaining things I am currently unsure about:

- what tables have the weirdest edge cases / pose the biggest potential
  challenges, either for reading or writing? For instance serializing `loca` may
  depends on contents of `glyf`?
- how should the library handle errors during parsing? Is there any utility in
  reporting the locations of errors? Do we assume good data? If we try to handle
  errors ‘correctly’ then likely the entire API surface will need to return
  Result types, and that might get exhausting.
- is there a DRY method for generating methods that access various fields on
  various tables? How much variation is there likely to be in terms of the API
  requirements of different tables?
- How extensively do we want to be using concrete types, in place of things like
  type aliases/raw scalars? For instance a u16 can be used to represent a large
  number of possible things in the spec, from a glyph identifier to an
  F2DOT14 to an Offset16, an FWORD, etc etc. My inclination would be to have
  newtypes for all distinct data types, but that might seem cumbersome to some
  folks.
- Certain collections are heterogeneous, such as the different subtable types in a
  layout table. Can we generate nice code that represents these types using
  enums where appropriate?



[fonttools-rs]: https://github.com/simoncozens/fonttools-rs
[pinot]: https://github.com/dfrg/pinot
[`pinot::Buffer`]: https://github.com/dfrg/pinot/blob/7001393e7ef824ca99a7ed60a7592a9950a88230/src/parse/mod.rs#L16
[fonttools]: https://fonttools.readthedocs.io/en/latest/index.html
[HarfBuzz]: https://harfbuzz.github.io
[data types]: https://docs.microsoft.com/en-us/typography/opentype/spec/otff#data-types
