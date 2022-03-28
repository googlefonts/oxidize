- 2022-03-10
- status: draft
- author: @cmyr
- PR: https://github.com/googlefonts/oxidize/pull/15

I have spent a bunch of time over the past month digging pretty deeply into a
tentative initial implementation of font parsing for the oxidize project.

This work has been focused on the zerocopy, read-only case.

status:

The work is currently very preliminary. I'm tempted to keep writing code in
order to flesh things out further, but I think it is a good time to step back
and solicit further input before deciding next steps.

## current design

The project [currently consists of three crates][explore-font-types]. [`font-types`][] contains the
basic data types defined in the specification, as well as a few helper types
(such as a `BigEndian<T>` wrapper type) that are foundational, and are used by
[`font-types-macro`][], which implements a procedural macro for describing
tables, records, flags, and other font data structures. Finally
[`font-tables`][] implements API for accessing and manipulating different tables
and records, which are defined using the macro.

### the macro

The macro uses rust-like syntax to define types. You can see real examples of
how it is used in the [font-tables::tables][tables] module, but I will include some
examples here as well. For instance, defining a simple record looks like:

```rust
LangSysRecord {
    /// 4-byte LangSysTag identifier
    lang_sys_tag: BigEndian<Tag>,
    /// Offset to LangSys table, from beginning of Script table
    lang_sys_offset: BigEndian<Offset16>,
}
```

This generates code that looks approximately like,

```rust
#[derive(Clone, Copy, Debug, zerocopy::FromBytes, zerocopy::Unaligned)]
#[repr(C)]
pub struct LangSysRecord {
    lang_sys_tag: BigEndian<Tag>,
    lang_sys_offset: BigEndian<Offset16>,
}

impl LangSysRecord {
    /// 4-byte LangSysTag identifier
    pub fn lang_sys_tag(&self) -> Tag {
        self.lang_sys_tag.get()
    }

    /// Offset to LangSys table, from beginning of Script table
    pub fn lang_sys_offset(&self) -> Offset16 {
        self.lang_sys_tag.get()
    }
}
```

This type can be mapped onto a pointer into font data; after a bounds check, all
field access is infallible, and compiles down to a read + a byte swap.

Slightly more complicated is defining tables, or records that include arrays.
For these types, we do not know the size of the type at compile time.

```rust
/// [Script Table](https://docs.microsoft.com/en-us/typography/opentype/spec/chapter2#script-table-and-language-system-record)
#[offset_host]
Script<'a> {
    /// Offset to default LangSys table, from beginning of Script table
    /// — may be NULL
    default_lang_sys_offset: BigEndian<Offset16>,
    /// Number of LangSysRecords for this script — excluding the
    /// default LangSys
    lang_sys_count: BigEndian<u16>,
    /// Array of LangSysRecords, listed alphabetically by LangSys tag
    #[count(lang_sys_count)]
    lang_sys_records: [LangSysRecord],
}
```

For these types, our generated code looks different; we essentially generate a
*view*, which is validated at runtime. If validation succeeds, field access is
again just a pointer read and a byte swap. The generated code here looks like,


```rust
pub struct Script<'a> {
    default_lang_sys_offset: LayoutVerified<'a [u8], BigEndian<Offset16>>,
    lang_sys_count: LayoutVerified<'a [u8], BigEndian<u16>>,
    lang_sys_records: LayoutVerified<'a [u8], [LangSysRecord]>,
}

impl<'a> Script<'a> {
    pub fn read(bytes: &'a [u8]) -> Option<Script<'a>> {
        let (default_lang_sys_offset, bytes) =
            LayoutVerified::<'a [u8], BigEndian<Offset16>>::new_unaligned_from_prefix(bytes)?;
        let (lang_sys_count, bytes) =
            LayoutVerified::<'a [u8], BigEndian<u16>>::new_unaligned_from_prefix(bytes)?;
        // resolved because this is used as the count for the next argument
        let resolved_lang_sys_count = lang_sys_count.get();
        let (lang_sys_records, bytes) =
            LayoutVerified::<'a [u8], [LangSysRecord]>::new_slice_unaligned_from_prefix(
            bytes,
            resolved_lang_sys_count as usize,
            )?;

        Some(Script { default_lang_sys_offset, lang_sys_count, lang_sys_records })
    }

    pub fn default_lang_sys_offset(&self) -> Offset16 {
        self.default_lang_sys_offset.get()
    }

    pub fn lang_sys_count(&self) -> u16 {
        self.lang_sys_count.get()
    }

    pub fn lang_sys_records(&self) -> &[LangSysRecord] {
        self.lang_sys_count.get()
    }
}
```

The funny part here is the [`LayoutVerified`][] type. Provided by the
[zerocopy][] crate, this is essentially a pointer + a length, that is used to
validate bounds and alignment of memory. In our `read` method, for each field we
iteratively read chunks from the front of the input slice. If the slice is too
short, we return `None`, and if construction succeeds then all access to fields
is infallible.

It might be possible to make this slightly more efficient by replacing
`LayoutVerified` with hand-written bounds checks, but for now I prefer the
ergonomics of this approach.

#### #[attributes]

Here we start to see the first 'attributes': these are various declarations that
occur in brackets, preceded by a pound sign, like: `#[..]`. In this example the
entire table has the `#[offset_host]` annotation, and the `lang_sys_records`
field has the `#[count()]` annotation. The first of these tells the macro that
this table is used as the base for some offsets (which means we need to hold on
to our source data, and implement methods to resolve offsets) and the second
annotation tells the macro that length of the `lang_sys_records` field is
determined by the value of the `lang_sys_count` field.

There are other attributes available; the exact set will surely continue to
evolve. Overall, though, this pattern lets us express the majority of the
patterns I have encountered so far. As a fallback, we can represent certain
fields as just byte slices, and then hand-write code for accessing them at
runtime.

#### version enums

One final thing that the macro currently does is generate enums for tables that
have multiple version/formats. Let's use coverage tables as an example:

```rust
    CoverageFormat1<'a> {
        /// Format identifier — format = 1
        coverage_format: BigEndian<u16>,
        /// Number of glyphs in the glyph array
        glyph_count: BigEndian<u16>,
        /// Array of glyph IDs — in numerical order
        #[count(glyph_count)]
        glyph_array: [BigEndian<u16>],
    }

    CoverageFormat2<'a> {
        /// Format identifier — format = 2
        coverage_format: BigEndian<u16>,
        /// Number of RangeRecords
        range_count: BigEndian<u16>,
        /// Array of glyph ranges — ordered by startGlyphID.
        #[count(range_count)]
        range_records: [RangeRecord],
    }

    #[format(u16)]
    enum CoverageTable<'a> {
        #[version(1)]
        Format1(CoverageFormat1<'a>),
        #[version(2)]
        Format2(CoverageFormat2<'a>),
    }
```

The relevant part is the enum declaration at the bottom. This generates an enum
that is initialized with a slice of bytes; it reads a single `u16` (specified by
the `format` attribute) and then based on that value it generates one of the
variants based on the specified `version` attribute.

#### preliminary codegen

Writing all of these macro declarations is still a lot of work, so there is a
preliminary step to help out: this step takes as input [text files][] containing
tables and records copied directly from [the spec][], with a few added
annotations. This tool generates the vast majority of the code for the table
definitions, which are then cleaned up manually.


## Findings & conclusions

This approach has worked very well for generating types suitable for shaping,
but is not trivially adaptable to the compilation & modification case.

### codegen is good

I am very happy with the approach of doing lots of automatic code generation; in
the majority of cases I have been able to express the basic logic of a given
table without much difficulty, and in cases where a table had some particularly
tricky representation, I've been able to fall back on manual processing: for
instance in the glyf table, the exact size and location of the
[flags, xCoordinates, and yCoordinates][simple glyph] arrays can only be
determined by examining them. In this case, we just [store the raw bytes][glyph raw]
for these arrays together, and [interpret them during parsing][glyph interpret].

I believe there is room to add additional functionality to the codegen tools:
for instance we could annotate offsets to include their source and target type
(e.g., `Offset16<ScriptList, Script>`) and in certain cases we could
automatically generate getters for specific offset targets.

In any case I think that the strength of the codegen approach will become more
clear when we start to really dig into the mutation/creation side of things: if
we are able to reuse the same table declarations to generate things like
builders, it will save us a bunch of time and energy.

### The shaping/read-only case

To start with the positives: I am very happy with how this approach works in the
read-only case. We are able to generate code that performs most bounds checking
at compile time; in general runtime field access is exactly a read and a byte
swap. In some cases we aren't generating perfect assembly, but this is
addressable (the specific problems I found were with certain functions not being
marked as being inlineable in the upstream crate, and I think we can expect this to be
addressed.)

Although I am hesitant to make a firm assessment without doing some more
real-world measurement and benchmarking, my hunch is that we should be able to
create a memory-safe API and user-friendly API with roughly equivalent
performance to existing C++ implementations.

### zerocopy mutability

Adding mutability to this design poses challenges. Fundamentally, the problem
comes down to Rust's memory safety guarantees. Specifically, Rust forbids
aliasing mutable references. In the zerocopy case, we would want to be able to
have mutable references to multiple tables at the same time. These tables share
the same underlying storage, so we cannot just pass out multiple mutable
references to that storage to our multiple tables.

The zerocopy crate *is* designed with this in mind: this is the reason for the
API of `LayoutVerified`, which can read off the front of a slice: under the
covers, it is splitting the pointer, ensuring that it has exclusive access. This
pattern may cover *some* of our needs, but not all of them, and even where it
was feasible it would add significant complexity. With the top-level
tables, for instance, we could verify when loading the font that the reported
offsets for the tables were all disjoint, and then we could split the underlying
data into chunks, each of which would be independently mutable; we could either
do this explicitly (allocating an array of disjoint slices) or via some other
mechanism, similar to a `RefCell`, where we would keep track at runtime of which regions
were currently borrowed.

This would work (although it would add significant complexity) in cases where we
had foreknowledge of the position and size of an object, but I do not believe
this could work when dealing with tables that host offsets, or at least not in
the general case. The problem with offsets is that they are not verified ahead
of time, and they often point to objects that do not have a known size. This
means there is really no way to know exactly how much memory a table at a given
offset is going to access. While it is possible to imagine complicated runtime
schemes for tracking this, it would definitely require allocating additional
data structures, and would significantly degrade the ergonomics of the API, and
significantly increase the learning curve.

I've been thinking about this problem quite a bit, and my opinion it is evidence
of a fundamental tension in the design goals of this project.

#### deciding on goals

I believe that we currently have two conflicting goals. One of the rationale for
this project is concern about the difficulty of getting developers up to speed
on HarfBuzz, and how hard it is to add features there without accidentally
introducing bugs. There is a hope that Rust's safety guarantees can give us a
code base that is easier to modify and hack on; however there is *also* a desire
to end up with code that is more-or-less equivalent to the existing C++.

The basic tension is this: the ergonomics and safety advantages that Rust offers
work by explicitly preventing certain behaviour, such as the aliasing of mutable
variables. We cannot translate idiomatic C++ directly into Rust and get safety;
we get safety by ruling out certain patterns.

I think it is really worth stepping back and considering our goals. We
would like to have a tool for authoring & manipulating font files, and we would
like it to be a) fast and b) easy to use. This is something that we can achieve,
but I do not think the result is going to look like it might look in C++. My
feeling is that too much of our thinking (or at least my thinking!) so far has
been focused on porting an existing design to Rust, and that this may be
misguided.

#### Subsetting is special

A major focus of (and motivation for) this work has been hb-subset. I think this
has influenced our thinking in important ways. I'm not entirely confident in my
analysis here, so please correct any bad assumptions. With that said:

- My sense is that the design of hb-subset is largely path-dependent on the
  existing design of HarfBuzz. This is just an intuition. What this means to me,
  though, is that if we were designing a subsetter from scratch, we might make
  different choices.
- Subsetting really feels like a special case. To the best of my understanding
  it is used almost exclusively(?) by Google Fonts, and at a much larger scale.
- I think that trying to address the *general* case in a way that also natively
  supports the subsetting case might be a mistake. In particular I worry that
  this dramatically constrains our solution space, and will lead to a design
  that is seriously compromised in the general case.
- I would prefer to focus on the general case, and then try to figure out a more
  holistic approach to the subsetting problem. In practical terms, this means a
  general approach that does more transient allocation for subtables etc, and
  then a subsetting approach that (for instance) uses a long-running process to
  handle multiple subsetting requests, and reuses memory and other state between
  them.

## next steps

### a design for mutability

My main concern right now is figuring out a model for compilation & mutability.
For compilation, I'm currently imagining generating builder types, but I haven't
dug into this yet.

Offset resolution and rewriting will also be an important consideration. A
compiler author should be able to use these tools to do things like deduplicate
identical subtables, or to reorder tables as needed, which means we do need a
mechanism for rewriting offsets as needed. There are various options here; for
instance we could track offsets by ids, and then keep track of the location of
the offsets as we write tables into a shared buffer, rewriting them later as
needed. This will need more thought, experimentation, and discussion.

**status** *this is the next major objective; I expect to have something to
report in the first week or two of May*.

### evaluating current performance

It may also be useful to get some concrete numbers on the efficiency of the
current work-to-date. I have spent a bit of time looking at disassembly and am
generally confident that we should be able to achieve good results, but I
haven't *demonstrated* this.

I'm not sure this is strictly necessary at this point, but it might be helpful;
if performance is actually terrible for some unforeseen reason, we might decide
that our approach needs to be fundamentally rethought.

I've also resisted this because I'm not sure of exactly what we should measure.
In any case, it is something we will want to do at some point.

**status** *this is deferred, for now. Profiling will require us to write a
non-trivial amount of code on top of the generated code, and the exact details
of the generated code may change during the work on mutability; I would prefer
to keep things as slim as possible while codegen is being iterated on.

### further refinements

Finally, there are various additional features and functionality that it might
be worth exploring, such as improving type information related to offsets (for
instance encoding in the type of the offset what sort of table it should
produce, or what table it is indexed into.) In addition, I am *very curious*
about looking into migrating from doing codegen in a proc-macro to doing codegen
as a separate build script that outputs normal rust files that are then included
and used like any other. This has several advantages: it should reduce
compilation time, but it should also produce more readable code: currently we
generate lots of methods and type definitions that do not actually exist in any
of our source files. This makes it harder for users to understand what is going
on, and it also confuses tools like IDEs, which often cannot provide
autocomplete for these generated methods.

**status** *deferred: this is interesting, but is not on the critical path*

## To discuss

- Is it worth doing some real-world profiling now? For instance, we could pretty
  easily implement something like "iterate all glyphs and manually compute
  bounding boxes", and we could try and compare that against HarfBuzz as well as
  other Rust font parsing implementations.
- Do my thoughts about hb-subset make sense? Put another way: would we be happy
  with a result where subsetting is, say 50% slower than hb-subset (and I think
  I'm being very pessimistic with that number), but is easier to maintain and
  contribute to? This would still be orders of magnitude faster than, say,
  fonttools.





[`font-types`]: https://github.com/cmyr/explore-font-types/tree/main/font-types
[`font-types-macro`]: https://github.com/cmyr/explore-font-types/tree/main/font-types-macro
[`font-tables`]: https://github.com/cmyr/explore-font-types/tree/main/font-tables
[the spec]: https://docs.microsoft.com/en-us/typography/opentype/spec
[zerocopy]: https://docs.rs/zerocopy/latest/zerocopy/index.html
[`LayoutVerified`]: https://docs.rs/zerocopy/latest/zerocopy/struct.LayoutVerified.html
[explore-font-types]: https://github.com/cmyr/explore-font-types
[text files]: https://github.com/cmyr/explore-font-types/blob/main/resources/tables/maxp.txt
[tables]: https://github.com/cmyr/explore-font-types/tree/main/font-tables/src/tables
[simple glyph]: https://docs.microsoft.com/en-us/typography/opentype/spec/glyf#simple-glyph-description
[glyph raw]: https://github.com/cmyr/explore-font-types/blob/161f5d455b9b667a28ef2c8ad793003cc8b42a16/font-tables/src/tables/glyf.rs#L44-L45
[glyph interpret]: https://github.com/cmyr/explore-font-types/blob/161f5d455b9b667a28ef2c8ad793003cc8b42a16/font-tables/src/tables/glyf.rs#L275-L278




