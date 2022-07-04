- 2022-07-04
- status: draft
- author: @cmyr
- PR: https://github.com/googlefonts/oxidize/pull/31

# Subsetting experiments

Over the past month or so I have implemented a toy subsetter on top of my work
on rust font parsing & compilation. The intent of this work is to provide a
preliminary sanity check of the project: is it feasible, with the current
design, to achieve performance similar to that of [`hb-subset`][]?

## design

The current design of my rust font library (let's just call it 'oxidize' for the
sake of brevity) involves two distinct sets of types; zerocopy types for
parsing, and then owned, allocating types used for compilation. In general,
there is a one-to-one correspondence between these types. As an example, the
`layout::CoverageFormat1` zerocopy type wraps a read-only pointer to some data,
from which it provides access to an array of glyph ids; and the
`layout::compile::CoverageFormat1` type owns a `std::Vec<GlyphId>`, which can be
mutated.

Each zerocopy type can be converted into a corresponding compile type, and
compile types can be serialized back out into a buffer.

The subsetter is built on top of this; we parse the font, convert the GPOS table
to its owned representation, modify that owned table by applying the subset
plan, and then write it back out to disk.

By contrast, `hb-subset` works without the intermediate allocating types; the
zerocopy types write themselves out into a buffer directly.

The TL;DR here is that the oxidize implementation is doing a bunch more work,
and I do not expect it, in this form, to be as fast as `hb-subset`. That said,
there are advantages to this design:

- it uses the same types (and serialization code) as would be used for a
  general-purpose font compiler
- the code is (hopefully) easier to follow; all of the subsetting code
  itself just involves modifying various types in-place ([see example][subset-impl])
- as each table owns its data, and subsetting mutates the table in place
  (not writing to a shared buffer) multiple tables can be subset in
      parallel.

### note on a design more similar to `hb-subset`

The design I've chosen here makes clear trade-offs; in particular it is trading
off some speed in favour of improved ergonomics and robustness (it relies on
much less hand-written code).

If this trade-off ends up being unacceptable, it will always be possible to
write a more bespoke implementation, which would operate directly on the parse
types, and write out bytes directly, very similar to how `hb-subset` works. This
would be more work and more error-prone, but it is a viable escape hatch if
needed. In a final implementation, it would even be possible to do this
*selectively*, for instance by doing these manual implementations only for
GPOS/GSUB, and still using the compile types for less complicated tables.

## benchmark setup

The goal for this test was not to build a production subsetter; rather it was to
build the 'simplest credible comparison', that is to implement exactly as much
functionality as is necessary in order to get a ballpark sense of performance.

After chatting with @garret, we decided on a simple comparison: subsetting using
glyph ids, dropping all tables but GPOS, and testing with a font that doesn't
use complex (contextual) shaping rules. This saves us the work of implementing
subsetting for the complex tables, and also saves the work of having to compute
the glyph closure, since we drop all the tables that might be involved.

The invocation to `hb-subset` looks like,

```sh
> ./hb-subset LibreBodoni-Regular.ttf -o subset_hb.ttf --gids 5,10,22,50-400 \
  --drop-tables="*" --drop-tables-=GPOS -n 20
```

On the rust side, we have a simple binary that tries to emulate this behaviour:
it drops all tables but GPOS, which it subsets based on some set of glyph ids.

The rust binary is [avaialable here][subset_gpos], and the invocation looks
like:

```sh
> ./subset_gpos LibreBodoni-Regular.ttf -o subset_rs.ttf --gids 5,10,22,50-400 --runs 20
```

## results

> Initial disclaimer: running each of these commands produces *similar* binary output:
there are an equal number of subtables with an equal number of records,
equal-length coverage tables, etc. The output is not identical; for instance
there seems to be a difference in the mapping of old->new glyph ids (presumably
the result of off-by-ones in my impl somewhere) but I do not believe that these
differences are important for evaluating performance.

Timing the execution of the commands above, running each five times:

```
hb:
0.01s user 0.01s system 66% cpu 0.035 total
0.02s user 0.00s system 91% cpu 0.023 total
0.02s user 0.00s system 91% cpu 0.022 total
0.02s user 0.00s system 92% cpu 0.022 total
0.02s user 0.00s system 92% cpu 0.023 total
```

```
oxidize:
0.03s user 0.00s system 87% cpu 0.035 total
0.03s user 0.00s system 94% cpu 0.034 total
0.03s user 0.00s system 94% cpu 0.035 total
0.03s user 0.00s system 95% cpu 0.035 total
0.03s user 0.00s system 94% cpu 0.035 total
```

That is, in this *very loose* comparison, `hb-subset` is about 50% faster than our
implementation.

To get a better sense for what's going on, I ran both of these through a
profiler. In our implementation, 60% of our time is spent in the conversion from
our zerocopy types to their compiling partners, and then 40% is spent on
subsetting and then writing the result back out.

From what I can tell, `hb-subset` is still spending about 30% of its time
creating the subset 'plan', which doesn't even show up in the oxidize profile;
this implies to me that harfbuzz is doing a reasonable chunk of extra work, and
I would infer from that our relative performance is worse than the numbers
indicate.


## conclusions

I want to be extremely cautious about reading too much into these results, but I
am encouraged; at the very least, I think this shows that we are in the same
general ballpark, and I think it we can be highly confident we're not more
than 2-3x slower. Given that our implementation is very naive, this is cause for
cautious optimism: our implementation leaves room for optimization, and if we
were specifically concerned with performance in the context of responding to
network requests, there are more holistic design choices we could make
specifically to address that problem; for instance we could trivially use
`serde` to serialize the font's compile types, and use these instead of the raw
ttf; or we could have a long-running server process that could reuse the same
(parsed, owned) font objects for multiple requests.

## next steps

I am now reasonably happy with the overall shape of this project: there is a set
of high-performance parse types, and a separate set of types for compilation,
and in both cases performance seems excellent.

In the particulars, there remain various sharp edges & dirty corners. I am not
happy with the specifics of the implementation on the parsing side, and have
ideas for how to improve that. There is no error handling; there are limited
diagnostics; the structure of modules & crates could change; there are numerous
tasks of this sort that I have put aside as not being necessary for this
preliminary investigation.

From my position, then, the next step is largely *deciding on what we want*, and
in what order. Do we consider these results encouraging enough to move forward?
Do we want to think about shaping? Do we want to focus on compilation? From my
perspective a very useful and informative next step would be to focus on
compilation for OpenType layout tables, but other parties may have other goals.


[subset-impl]: https://github.com/cmyr/explore-font-types/blob/bdb23b5f4acbd4b0a1efbd7e756979a634d75d31/font-tables/src/tables/gpos.rs#L598
[subset_gpos]: https://github.com/cmyr/explore-font-types/blob/subset/font-tabl
[`hb-subset`]: https://harfbuzz.github.io/harfbuzz-hb-subset.html
