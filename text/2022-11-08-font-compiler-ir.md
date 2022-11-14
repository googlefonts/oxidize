_Why_ is articulated in https://github.com/googlefonts/oxidize/blob/main/text/2022-11-14-why-ir.md.

- [Goals](#goals)
  - [Potential future goals](#potential-future-goals)
  - [Non-goals](#non-goals)
- [Basic structure](#basic-structure)
- [IR](#ir)
  - [Font sources](#font-sources)
  - [Sketch](#sketch)
  - [Change tracking](#change-tracking)
  - [Implementation notes](#implementation-notes)
  - [Test inputs](#test-inputs)
- [References](#references)

# Terminology

[Intermediate representation](https://en.wikipedia.org/wiki/Intermediate_representation), IR henceforth,
refers to a data structure used to represent source to facilitate further processing. In this document we
mean the sources a font authoring tool produces that we compile to a binary font.

# Goals

Provide a common format all current and anticipated font source formats can be converted into by a frontend so we can have a single backend compiler to produce
binary fonts.

The backend should be variable-first (no [merge](https://simoncozens.github.io/compiling-variable-fonts/#merge)). Once the user has a variable font they can
instance and subset it down as they see fit.

The design should facilitate parallel execution (see
[Simon’s explorations](https://github.com/googlefonts/oxidize/blob/main/text/2022-05-10-parallel-font-compile-experiments.md#what-discrete-operations-are-available))
and incremental compilation.

We should be able to optionally capture IR. IR should support human friendly (textual) and fast/compact (binary) representation (shamelessly copying ... I mean
inspired by ... [LLVM IR](https://llvm.org/docs/Reference.html#llvm-ir), hopefully leaning on serde).

This is very similar to the goals Behdad outlined for [Better-Engineered Font Formats](https://docs.google.com/document/d/1CDDSqHPdjAcvw3r2im5DfPc9XXFA04FCJoqdBUx2B2M/edit).

## Potential future goals

Enable translation from `sourceA → IR → sourceB` so we can modernise sources in an automated manner (credit @cmyr for this idea).

## Non-goals

Lossless `source → IR → source`.

# Basic structure

```
Source → IR → font binary
            ^ one ir2font implementation
       ^ many source2ir implementations
```

Source is one of

- .glyphs
- UFO (including many w/design space)
- .ttx
- nanoemoji config

In future, sources could additionally include:

- binary font
  - Some contend if we support ttx we effectively support binary fonts
- Unknown Future Design Format

For every source we will have a dedicated conversion to IR, e.g. `glyphs2ir`, `ufo2ir`, etc in https://github.com/googlefonts/fontmake-rs.

Let’s take `ufo2ir` as a concrete example, it would:

- Depend on a UFO parser, likely https://github.com/linebender/norad
  - Aside: norad uses quick-xml, we should try to stick to that elsewhere
- Depend on the IR library

# IR

## Font sources

Rough outline of common source structures

UFO:

- Designspace file maps UFO(s) into design space
- UFO font has layers
- Layers have glyphs

Glyphs:

- Font has glyphs
- Glyphs have one layer per master
- Plus user defined layers
- Plus "special" layers that have attributes
  - For example, "brace" layers to define intermediate masters for specific glyphs

## Sketch

Based on [ufo3](https://unifiedfontobject.org/versions/ufo3/), [babelfont-rs](https://github.com/simoncozens/babelfont-rs) and discussion in this repo. The
structures sketched here in pseudo-Rust (based on multiple reports a similar format was helpful for colr). We deliberately have no concept of "master", we
instead have a set of variable glyphs whose definition locations need not be identical. The examples are not exhaustive of all fields, they merely aspire to
convey the general idea of a variable first, sparse first, structure broken apart in alignment with units of work for a parallel backend:

```rust
// data common to the entire designspace
// written to disk in one file
// some of https://unifiedfontobject.org/versions/ufo3/fontinfo.plist/ 
struct Font
  upem
  axes: Vec<Axis>

// TODO: do we really need "discrete" axes? - designspace 5.0
enum AxisRange {
  Continuous(min, max, default)
  Discrete(Vec<f32>)  // e.g. Italic
}
struct Axis { tag, range: AxisRange, hidden }

// Roughly
type DesignSpaceLocation = HashMap<Tag, f32>

// the complete variable definition of a single glyph
// written to disk in one file per glyph
// components point to other glyphs, in time expand to variable components
// https://github.com/simoncozens/rust-font-tools/blob/main/babelfont-rs/src/glyph.rs 
// an obvious unit of work for parallel execution
// Note that this is sparse, not tied to masters, and the definition keys need not match glyph to glyph
struct Glyph
  name
  definitions: Map<DesignSpaceLocation, GlyphInstance>

// a specific instance of a glyph
// https://unifiedfontobject.org/versions/ufo3/glyphs/glif/
struct GlyphInstance
  advance
  outline: Vec<Shape>

enum Shape
  Contour(Contour)
  Component(Component)

// A set of features that need to be built together
// written to disk in one file per group
// encompasses both kerning and fea
// UFO https://unifiedfontobject.org/versions/ufo3/kerning.plist/ converts to fea
// fea files can in theory be torn apart, for example if a single fea file has a bunch of pairpos and some ligatures it could be split to enhance downstream parallelism.
// Behdad noted that kerning, accent, and cursive positioning come from outside fea. We generate fea, then compile. Ligatures are typically directly in fea. The GPOS merging is the main concern with instance + merge. Broadly, the feature file is the nastiest part of the whole compile.
// ^ suggests if we generate separate fea files presumably we can process them in parallel.
// Cosimo: matching anchors (e.g. glyph has a "top" anchor) to accent is done by undocumented (in UFO) convention, e.g. match "_top" mark to "top" anchor. All editors use this convention. We use this to create mark features in a fea AST which we then dump, insert into user fea (append or aligned with a magic comment [TODO: example of magic comment]), then compiled. It would be tricky to make the FontTools fea lib update layout given a fea, it wants to treat it as the complete definition.

TODO: how do we capture features(s) in a variable-first manner?
Just a Map<DesignSpaceLocation, FeaContent>?
```

## Change tracking

Every struct that is written to disk must support association of 0..N blocks to track changes to input to facilitate incremental compilation. The compiler
itself could be considered an input to ensure IR is not reused across compiler versions. Initially this could simply use `(size, mtime)` as rsync’s quick check
algorithm does. That may require elaboration to efficiently handle subsections of .glyphs files if we eventually want to incrementally compile them.

## Implementation Notes

Suggested phasing:

- Build ufo2ir for just glyphs and rudimentary font data (e.g. upem) for single UFOs
  - Norad + basic IR structures
- Build static glyphs
  - Create basic ir2font backend
- Extend ufo2ir to support designspace files + many UFOs
- Build variable glyphs
  - Enhance ir2font backend
- Include feature content
- Build rudimentary features

Suggested repository layout:

- https://github.com/googlefonts/fontmake-rs, a workspace initially containing:

  - ir
  - ufo2ir

TODO

- Elaborate if/how to wire into fontmake, e.g. should we takeover glyph compilation?
- Consider how this might fit Just’s font editor needs: sparse compile, watch capability

## Test inputs

1. Curated list of "interesting" families, https://github.com/google/fonts/issues/4772
1. An interesting set of Noto could be derived from oxidize#25
   - Linux-friendly way to create fontmake builds in [this gist](https://gist.github.com/rsheeter/9abfdce3aa8896056a4c6f8d65078793)

# References

- [The Case for a Font Compiler IR](https://github.com/googlefonts/oxidize/blob/main/text/2022-11-14-why-ir.md)- why we should have IR at all
- [Better Engineered Font Formats](https://docs.google.com/document/d/1CDDSqHPdjAcvw3r2im5DfPc9XXFA04FCJoqdBUx2B2M/edit)
- https://simoncozens.github.io/compiling-variable-fonts/
- https://github.com/simoncozens/rust-font-tools
  - Some parts may be reusable, though we would require any binary font access go through https://github.com/googlefonts/fontations
- https://unifiedfontobject.org/versions/ufo3/index.html
- Just’s notes on font editor needs (https://github.com/googlefonts/oxidize/issues/21)
