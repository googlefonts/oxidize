
Behdad answering which tables are weirdest
   * format is non-uniform
   * CFF != glyf != G* != simple struct tables
   * Layout ends up with either a builder or owned versions you compile
   * Vs glyf or cff you compile glyphs and then glue together

   * Can't model in a uniform way, too inconsistent; each table gets it's own model

   * CFF is truly compiling code, instructions w/offsets w/variable length offset
      * can simplify by padding offsets (5-6 pointers at top of struct)
      * could do multipass, e.g. assume size then spin until stable
      * subroutinizer adds considerable complexity
         * don't need for the web, but if you ship on devices
         * simple/greedy algorithm for this exists

   * complexity groups {glyf, CFF, layout, simpler ones like name or cmap or fvar can be modelled, varstore [but small]}
      * COLRv1 has the novelty of pointer to middle of array with length
      * https://github.com/harfbuzz/harfbuzz/blob/main/docs/repacker.md

   * Behdad will write a taxonomy of the types of data structures, https://github.com/googlefonts/oxidize/issues/7

   * Types
      * HBUNINT16, etc act very much like U32<BigEndian> in zerocopy
         * provides equals, assignment, etc
      * struct SingleSubstFormat2 is a nice simple example

Behdad error handling

   * DO have to handle errors
   * result types everywhere could become tiresome
   * HarfBuzz doesn't report errors, just best effort at shaping (safely)
      * rejecting the entire font in shaping isn't a helpful result; prefer forgiveness
   * Instead of endless error checks we validate when reading from an offset
      * potentially rewriting the table to make the offset 0
      * We resolve reads from offset 0 to a null instance
      * So no null pointers
   * This ofc doesn't apply to compilation
   * hb-subset is a filter, not a general purpose table builder
      * compilation is a bit different

Garret on building subsets

   * given a source font and a description of which parts of it we want
      * the subset plan
   * guess size and allocate a buffer, preferring to be slightly too big
   * on a per table basis (mostly)
      * start writing table into buffer

For offset graphs, e.g. layout

   * `[head ][    ][tail]`
      * start writing an object, e.g. a subtable in a lookup, any struct or thing packed together
         * a thing that is accessed by offset
      * objects in tail can refer to objects previously written to tail by offset
      * we're writing tail to head
   * we also have an object table
      * tracks references
         * the object in tail has a null offset, all resolved in object table
      * by hash, so a second reference to identical bytes deduplicates
   * if we determine we don't want the object in head we can revert, just dump it
   * head has a stack of objects that we pop off to push into tail or discard
   * at the end we know all the *actual* offsets and can push them into the things in tail
      * we may then observe we have illegal [too large] offsets
      * if so, we invoke repack https://github.com/harfbuzz/harfbuzz/blob/main/docs/repacker.md
         * operates on object graph
         * move things, duplicate shared nodes, etc
   * subsetter closes over the things reachable from the initial request
      * e.g. subset to "field", which could activate the "fi" ligature so keep that glyph too
      * expand glyphset until doing so stops expanding the set
      * doesn't account for shaper-specific logic
         * Google Fonts accounts for this by expanding the input set before calling subsetter
         ^ unpublished but we could
      * Raph noted you could fuzz this
         * shape on whole font, shape on subset font, is same?
   * Between Behdad and Garret get the part upstream of the repacker documented, https://github.com/googlefonts/oxidize/issues/8
   * TODO: Garret volunteers to give a talk on the repacker

Accelerator

   * caching layer on top of the zerocopy structs
   * hb-ot-layout-gsubgpos.h
      * hb_ot_layout_lookup_accelerator_t
      * allocates to speed up lookup
      * keep a pointer to coverage tables
         * a set of gids, CoverageFormat1: sorted array of gid or CoverageFormat2: sorted array of ranges
            * bsearch on this is lethal to performance
         * digest speeds this process up 3x
            * key insight: often lookups only apply to a few glyphs (e.g. just digits or just devanagari, 10s or 100s out of 1000s)
               ~10 step bsearch is still quite a lot of ops
            * hb-set-digest.hh is a fixed-width bloom filter on gids
               * a 64bit int can look at six bits of gid and capture all patterns used
               * look at middle six bits first, then lower, then top
               * in ~10 int ops we can match a coverage table that only handles a small # glyphs
               * in some situations it's full and always passes you through
   * harfbuzz does *not* do things like cache result of kern resolution
      * basically it eschews anything that enthusiastically consumes memory

In HarfBuzz we can't declare structures with variable sized fields; some gets left behind

   * e.g. ChainContext

hb-open-type.hh types are all BE, no alignment required

   * defined in terms of 1uint8_t[]`
      * so a struct of them can be reinterpret_cast
   * transparent endian conversion, you can read/write and pretty much ignore that it's not "really" a uint16_t/uint32_t/etc
   * Tag is just an HBUINT32

   * Offset has two variants, one for fields that permit null and one for others
      * one null means "I don't have one"
      * one null means "it starts *right here*"
      * See struct OffsetTo
      * operator + w/a Base gives you the target (because it's an offset from the base)
      * here we see get_null and get_crap (depending on const'ness)
         * Common Region for Access Protection

Colin suggests access modulo length could allow eliding the bounds check while ensuring valid offset

Arrays in spec are great fun and may store length, length+1, or length-1

   * HB has distinct types, e.g. ArrayOfP1 or ArrayOfM1
   * ex GSUB4 or packed deltas in varstore
   * https://docs.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas

Colin asked about what I'll paraphrase as high-memory use accelerators

   * tl;dr they could make sense, Garret notes cmap for CJK as an example
   * Raph notes we could precompute a perfect has for cmap lookup

Safety issues

   * Raph suggests we might want a compiler flag to bypass unsafe entirely, even at cost of performance
   * Garret notes exploding computational complexity is also an issue; tracking op counts matters
   * Chad notes not propagating errors would mean we can't replace ot-sanitize
      * Result everywhere might be OK, how detailed is an interesting question
   * Behdad notes shaping has *very* few memory allocating sites so error handling is very simple
      * less true of subsetting, which checks and gives up vs giving a best effort result

Behdad overview of HB loading

   * create an hb_blob_t from, typically, an mmap
   * create a face from that
   * accesss tables from the face, e.g. face->ot.table.gsub
      * thread-safe
      * may generate a safe copy if a borked item

Garret notes hb-subset scales with size of output vs pyftsubset that scales with size of input

   * See [Faster Horse](goo.gl/Qy3Eqc) design doc

PDF use case, generation of variable font instances also important

   * Raph notes there may be a case to start from a representation other than the raw font bytes

Chad notes early exploration suggests we can have substantially safe code

   * can't reinterpret_cast w/o unsafe but ... maybe that's almost it

View vs Transmute

   * View can be safe, which is a BIG plus
   * Transmute is more HB-like
   * Really need to try it and see
