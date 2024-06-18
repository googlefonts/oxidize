# Combatting Fuzzer Timeouts in Harfbuzz

This document captures my (@garretrieger) experience implementing mechanisms to prevent fuzzer timeouts in Harfbuzz.

## The Problem

The font format is complex and presents many opportunities for particular configurations of the data to result in extremely long processing
times. In many cases, this can be done with relatively small input font files. Since fuzzers explore all kinds of ways of encoding font
data, they will inevitably trigger these degenerate cases, ultimately resulting in timeouts.

Timeouts in the fuzzer can block it's progress so should be eliminated. Additionally timeouts are a potential security issue. If timeouts
aren't prevented then, malicious fonts could be crafted that cause the software processing fonts (such as browsers) to lock up when trying
to render them.

We spent a significant amount of time in Harfbuzz tracking down and fixing timeouts found by the fuzzer. This led to developing several
strategies to prevent broad classes of timeouts instead of fixing each timeout as a one-off.

## Main Types of Mechanisms that Can Create Timeouts

When processing fonts, we generally strive to have the processing time roughly proportional to the size of the input font file. For typical
fonts, this is generally true. However, fonts crafted to abuse various mechanisms in the font format can cause processing times to scale
much faster than linearly (e.g. exponentially) and result in timeouts for small inputs.

There are two broad classes of mechanisms that can cause non-linear scaling in processing times:

- **Class 1: Structural Explosion:** A common pattern throughout the font format is to have offsets which point to blocks of data
  (subtables). Subtables can, in turn, have offsets of their own. In almost all cases, the font specification allows subtables to overlap
  (partially and even completely). This means you can have an array of offsets which point to the same subtable object, effectively
  multiplying the bytes of the subtable at the cost of one 16-byte offset per copy. If this pattern is repeated for multiple levels, it
  creates an exponential explosion, causing the effective size of the input font to increase by orders of magnitude.

- **Class 2: Processing Explosion:** After interpreting the bytes of the input font, the structures are processed in various ways.
  In certain cases, the nature of the processing introduces additional opportunities for non-linear scaling of processing costs. The three
  most commonly seen cases are:

  - **Recursive mechanisms:** There are several instances where processing data contains some sort of mechanism which enables recursion.
    Typically this is not limited to being acyclic and thus creates the possibility of creating cycles and infinite loops during
    processing. The primary example is GSUB/GPOS lookups, which can recursively invoke other lookups via lookup ID.

  - **Range-based structures:** When encoding sets of things, it's common to have a subformat which allows the set to be specified as a
    list of ranges (start + end). This provides a mechanism to generate many set members for a relatively small encoding cost.

  - **Computation-based structures:** There are some types of data in the font which are treated as a program which is executed (the
    primary example is CFF and CFF2). Like with the recursive case, this provides the possibility of unbounded execution times.

## Mitigation Strategies

1. **Effective byte counting:** In Harfbuzz, all inputs are sanitized before being used for any processing. The sanitization process walks
   the complete structure of all tables in the font. During sanitization, we keep track of the effective number of bytes processed. If
   multiple offsets point to the same subtable bytes, we add the count of those bytes each time they are referenced. Sanitization enforces
   a maximum effective byte count which scales with the size of the inputs. If the limit is exceeded, the input is rejected, and no further
   processing is done. For reference, "Max ops" (max effective bytes) is computed
   [here](https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-sanitize.hh#L225). This approach pretty much eliminates all class 1
   timeouts. However, it will not catch class 2 timeouts since the sanitizer only checks the structure of the input and not the
   interpretation.

2. **Input validation:** In many cases, the input data for a particular sub-table type can be validated against constraints introduced
   elsewhere in the font file or external to the font file. This can effectively limit the maximum set sizes that can be encoded into a
   sub-table. This is particularly effective with range-based structures. Some of the typical validations we do:

   - Check that ranges are in sorted order and disjoint (when required by the spec). This prevents multiplying the processing times by
     overlapping the same range multiple times.

   - Limit the set members to the domain of the set. For example, when ranges represent Unicode codepoints, reject/ignore anything outside
     the set of valid Unicode code points. This substantially reduces the maximum set size. Likewise, for glyph IDs, limit the IDs to
     either the maximum glyph id (2^16) or the maximum glyph IDx in the font/subset.

   [Example validation during coverage subsetting](https://github.com/harfbuzz/harfbuzz/blob/main/src/OT/Layout/Common/Coverage.hh#L165):
   Here, we limit the glyphs in coverage to be less than or equal to the maximum glyph in the output subset.

   The most problematic tables where input validation is helpful are the range-based variants of cmap, Coverage, and ClassDef. This list is
   not exhaustive; anywhere you have the ability to encode things as ranges, this validation should be implemented.

3. **Operation Counting:** This works similarly to effective byte counting, but instead of counting bytes, we count some sort of operation
   that happens during processing structures susceptible to class 2 issues. For example, when processing lookups, we count the number of
   lookups visited and end processing when a configured limit is reached. In CFF/CFF2 processing, we limit the number of instructions that
   can be executed. In Harfbuzz, [hb-limits.h](https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-limits.hh) contains several
   configurable operation limits. This can be used as a starting point to find places where we've implemented operation counts.

4. **Cycle Detection:** when processing recursive structures it is sometimes possible to eliminate cycles (if it doesn't affect
   correctness) via cycle detection. For example harfbuzz does this during
   [composite glyph closure](https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-subset-plan.cc#L810). Cycle detection will allow the
   processing to not waste any time in cycles and can be used in conjunction with operation counting to catch other non-cycle cases.

## Future Improvements ##

One thing we currently don't do in Harfbuzz, which is causing problems, is to use a unified work counter across different parts of the
code. For example: we have one work counter when getting glyph outline from the glyf table. And we have a separate work counter in the
VARC table. But the two when combined currently can do too much work because we use separate counters.
