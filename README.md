# oxidize

Wherein we contemplate moving shaping, rasterization, font compilation, and general font inspection and manipulation from Python & C++ to Rust.

## Why

We currently rely C++ for anything high performance, e.g. shaping or subsetting, and Python for general purpose manipulation. The results are exactly what you'd expect:

* Python code (fonttools, fontmake, nanoemoji, etc) have high development velocity, relatively fast developer rampup, and runtime safety, slow runtime
   * It's SO slow that running it in production at scale causes us great pain. Queues back up, we burn memory and CPU, and it just doesn't fit our infra all that well.
* C++ code (HarfBuzz) has fast runtime, very slow development velocity, very slow developer rampup, and any safety it might have is due to extensive fuzzing
   * It's entirely normal to make a seemingly safe change and then see a stream of fuzzer bugs
   * Best thing ever in production, blindingly fast and low resource cost, fits infrastructure nicely. But scary due to lack of safety.

Some logic ends up implemented in both languages. For example, when it became apparent the Python subsetter was too slow for both runtime serving and future projects like [progressive font enrichment](https://www.w3.org/TR/PFE-evaluation/) the [hb-subset](goo.gl/Qy3Eqc) project was born.

Rust appears to offer us the ability to implement fast, safer code with development velocity in between Python and C++. This is a very appealing offer.

## Goals

### Support both our primary usage patterns

1. Performance-critical read-only users (shaping, rasterization)
1. Read/write use (compilers, utilities)

For read-only users we may need to support readonly mmap access. Crates like zerocopy and the like suggest this is possible. Establishing a clean pattern here should be tackled early ([poc exploration](https://github.com/rsheeter/rust_read_font)).

### Provide a safe, productive, codebase

We seek to provide a developer friendly codebase. For example, exploration of new structs for something COLRv1 should be readily achievable by forking and hacking around. That means the code needs to be simpler than todays C++ and as close as possible to Python level development velocity. To achieve this we will seek to establish an efficient pattern to support our two primary usage patterns. This will require that we:

   * Stay DRY
      * Don't hand-write things twice, once for high performance readonly access and once for mutable access
      * Don't hand-write per-field parsing (struct.blah = read type of blah, etc)
         * It's OK to *occasionally* hand-write, particularly for some of the more obtuse parts, but it should be the exception not the rule 
      * Automated authoring, such as by macro or build time tooling is acceptable
   * Minimize unsafe code
   * Invest in testing

### Prefer incremental delivery, early production use

We will _prefer_ to:

* Make incremental progress, with each milestone being used in production, rather than spend years with parallel increasingly divergent implementations.
* Test the Rust implementation against the non-Rust implementation
   * This has been very successful to enable replacement of Python subsetting with hb-subset
* Fail-fast, if rebuilding in Rust is infeasible we would like to know ASAP
   * Given the results of projects like piet-gpu and swash we currently believe the effort very feasible

## Getting started

_TODO: document first steps, milestones ... or use issues/milestones in this repo_

Current guess at first steps (very open to debate):

1. Explore how to support font I/O for both primary use cases cleanly, demonstrate a solution
   * If Swash could shape using our shiny new toys that would be a very nice demonstration
3. Build next-gen fontmake, converting parts to Rust and enabling parallelism
   * Guessing we would either do a good interchange format ([flatfont](https://github.com/googlefonts/flatfont)?) + ninja OR a Rust based executor using something like Rayon
