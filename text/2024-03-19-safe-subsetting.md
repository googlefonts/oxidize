# Memory Safe Subsetting and Instancing

Notes from @garretrieger visit to SVL, discussion with @qxliu76, @rsheeter, @rictic, and @cmyr.

## Goal

Start thinking seriously about moving subsetting and instancing to Rust.

## Context

### Prior Art, Python & C++

We have existing implementations in Python ([FontTools subset](https://fonttools.readthedocs.io/en/latest/subset/)) and C++ ([hb-subset](https://harfbuzz.github.io/harfbuzz-hb-subset.html)).

In general FontTools is considered the gold standard, with some exceptions where hb-subset is ahead.

### Prior Art, Rust

@cmyr wrote a toy subsetter to demonstrate feasibility. See:

* https://github.com/googlefonts/fontations/tree/subset for the source
   * Updated to run against latest fontations
* [2022-07-04-subsetting-experiments.md](./2022-07-04-subsetting-experiments.md) for write-up of results

### fontations vs HarfBuzz read/write access

fontations has two distinct crates:

1. [read-fonts](https://crates.io/crates/read-fonts)
   * zerocopy data access to a font provided as a slice of bytes
1. [write-fonts](https://crates.io/crates/write-fonts)
   * Owned types for modification
   * Idiomatic data access, A with an offset to B is just an A that has-a B

Read and write types are produced by code generation. This is done because Rust enforces 1 mutable accessor and prefers directed graphs. Pointer webs into shared buffers tend to produce unsafe code. See [read vs write types](https://docs.rs/write-fonts/latest/write_fonts/#write-versus-read-types).

Read types can be converted to write types, such as via [`to_owned_table`](https://docs.rs/write-fonts/latest/write_fonts/from_obj/trait.ToOwnedTable.html#tymethod.to_owned_table) in [this example](https://docs.rs/write-fonts/latest/write_fonts/#examples).

HarfBuzz sanitizes objects then assumes future accesses are all OK, no offset reaches outside the active table.

read-fonts validates as we read. For example, [gdef](https://github.com/googlefonts/fontations/blob/d158188cf47ad5372e4ddad1d6598190834636a4/read-fonts/generated/generated_gdef.rs#L52).

   * Validation is meant to be fast
   * If we get to the end of validate then this table - not it's sub-tables - is valid
      * That is, we have the bytes for all fields
   * Reading is bounds-checked
      * This could change if profiling reveals it to be problemmatic

### Complexity clamps

Fuzzers are very good at building very compact overlapping datastructures that expand borderline endlessly. This gives behavior like a nice small font that takes seconds to walk or shape.

HarfBuzz guards against this by tracking complexity and bailing out.

Fuzzing will likely very rapidly find similar issues in fontations.

> [!NOTE] 
> We will need this in Rust also, BEFORE Chrome go-live


#### Writing binary data with write-fonts

Producing binary data with write-fonts generally has 2 or 3 steps:

1. Optional, use a higher level type or function to product the lower level type
   * For example, [`CoverageTableBuilder`](https://docs.rs/write-fonts/latest/write_fonts/tables/layout/struct.CoverageTableBuilder.html)
1. Populating a [`FontBuilder`](https://docs.rs/write-fonts/latest/write_fonts/struct.FontBuilder.html) with tables
1. Calling [`FontBuilder::build`](https://docs.rs/write-fonts/latest/write_fonts/struct.FontBuilder.html#method.build) to assemble a final binary

Every object gets it's own buffer.

> [!NOTE] 
> This may prove too slow for some steps in subsetting and instancing

### Packing

HarfBuzz packing is fairly well documented:

   * https://github.com/harfbuzz/harfbuzz/blob/main/docs/repacker.md
   * https://github.com/harfbuzz/harfbuzz/blob/main/docs/serializer.md
      * Incrementally and efficiently fill a buffer of only modestly more than the output size while removing duplicates by treating the buffer as two stacks, one for work in progress and one for completed objects
   * TODO: public link to @garretrieger slide deck

HarfBuzz process is:

1. Try a naive pass (pack in order of encounter)
1. If ^ fails, repack
   1. Topological sort and see if that is enough
   1. Proceed to more aggressive interventions

The heart of the Rust implementation is [`dump_table`](https://github.com/googlefonts/fontations/blob/d158188cf47ad5372e4ddad1d6598190834636a4/write-fonts/src/write.rs#L52):

```rust
pub fn dump_table<T: FontWrite + Validate>(table: &T) -> Result<Vec<u8>, Error>
			table.validate()?;
```

[`pack_objects`](https://github.com/googlefonts/fontations/blob/d158188cf47ad5372e4ddad1d6598190834636a4/write-fonts/src/graph.rs#L303) packs the graph, heavily based on the HarfBuzz implementation.

> [!IMPORTANT]  
> The Rust implementation is behind HarfBuzz on a couple of points and needs catchup

## Performance goals

hb-subset is the current king of the hill for speed, running 250x+ as fast as the FontTools python subsetter. We need to come close to matching this, even at cost of introducing unsafe though of course we would prefer to be fully safe.

@garretrieger has done some profiling of hb-subset and observes that:

* Most of the time is spent in core datastructure operations
* Building Coverage and ClassDef is very time consuming
   * @cmyr's exploration in Rust found the same
   * For example, in one hb-subset profile 30% of the time was coverage, specifically manipulating bitsets and hashmaps

### Fast toys

HarfBuzz has domain-specific tuned sets. These seem likely to be worth porting, specifically:

* Source (each successive file builds on the prior)
   * https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-bit-set.hh
   * https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-bit-set-invertible.hh 
   * https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-set.hh
   * https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-algs.hh
      * Esp fasthash functions
      * Note also issues with int hashing mentioned [here](https://github.com/harfbuzz/harfbuzz/blob/3d455998bf2d48b333522e3f2bd72e720b5d7d5c/src/hb-algs.hh#L358)
* Tests
   * https://github.com/harfbuzz/harfbuzz/blob/main/src/test-set.cc 
   * https://github.com/harfbuzz/harfbuzz/blob/main/test/api/test-set.c 

## Make it go

### Where does the code go?

* Subsetting and instancing should be in a single new crate in fontations, raising immediate pressing questions about picking a Norse name
* Shaping should be in Skrifa, it's the library we point most people who just want to "use" a font at

### Shared test suites

Ideally we would like to have something akin to a conformance test suite for HarfBuzz, FontTools and Fontations implementations of subsetting. This is intended to be modelled off the current HarfBuzz test suite, likely hoisted out into it's own repository, similar to how https://github.com/harfbuzz/harfbuzz-monster-fonts is setup.

How does the current HarfBuzz test suite work?

* https://github.com/harfbuzz/harfbuzz/tree/main/test/subset
   * Consider https://github.com/harfbuzz/harfbuzz/blob/main/test/subset/data/tests/cmap.tests
   * PROFILES is command line flags, it would be helpful if we used the same ones [we'll need binary and lib entrypoints]
   * Run the crossproduct of font, profile, subsets
   * Compare each with an expected file which is in  https://github.com/harfbuzz/harfbuzz/tree/main/test/subset/data/expected
   * **We'll need expected per compiler**
   * https://github.com/harfbuzz/harfbuzz/blob/main/test/subset/generate-expected-outputs.py updates the expected files provided ttx matches
      * Note that this means ttx equivalent but binary different expected files
   * By default FontTools should assumed the gold standard
      * But not always, so we need the ability to specify it, likely defaulting to FontTools
* Today, if you modify subsetting
   * Modify code
   * Add a new test
   * Generate expected outputs
   * Commit it all together

@garretrieger prefers to keep the test data in the harfbuzz org. Once hoisted out one might imagine HarfBuzz referring to a specific version so a new test can be added and then HarfBuzz updates and implements.

Bonus sharing opportunity: repacker

* https://github.com/harfbuzz/harfbuzz/tree/main/test/subset/data/repack_tests
* Subset [this font] to [these codepoints] and confirm success


> [!NOTE]  
> C++ and Python largely match flags. Rust should continue this.

### Fuzzing

* https://github.com/harfbuzz/harfbuzz/tree/main/test/fuzzing has seeds
* Fuzzer interprets incoming bytes as arguments *and* binary font
* Similar input model to https://github.com/google/woff2/blob/master/src/convert_woff2ttf_fuzzer_new_entry.cc 

oss-fuzz refs

* https://github.com/google/oss-fuzz/blob/master/projects/woff2/project.yaml 
* https://github.com/google/oss-fuzz/tree/master/projects/harfbuzz 

https://github.com/googlefonts/fontations/issues/420 covers fontations fuzzing.

