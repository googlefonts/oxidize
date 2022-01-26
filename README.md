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
   * HarfBuzz and FreeType have *years* of performance optimizations. A replacement that is significantly slower will not land.
1. Read/write use (compilers, utilities)
   * Performance is less critical here. We don't want to be *slow*, but we don't need the level of performance obsession shaping needs either.

For read-only users we may need to support readonly mmap access. Crates like zerocopy and the like suggest this is possible. Establishing a clean pattern here should be tackled early. https://github.com/googlefonts/oxidize/pull/3 proposes a path forward.

### Offer memory safety for text rendering 

Chrome and Microsoft both observe that:

1. 70% of high severity security bugs are memory safety issues ([Chrome](https://www.chromium.org/Home/chromium-security/memory-safety), [Microsoft](https://msrc-blog.microsoft.com/2019/07/18/we-need-a-safer-systems-programming-language/))
1. Rust _may_ help
   * If it can [play nice with C++](https://security.googleblog.com/2021/09/an-update-on-memory-safety-in-chrome.html)
   * For Fonts we can, if necessary, limit ourselves to stable C ABI

A safe text stack is desirable for Chrome, Android, and more. At time of writing we observe a semi-steady stream of fuzzer issues. It seems plausible a Rust rewrite can stop the bleeding. This is promising enough it's well worth giving it a try!

HarfBuzz is the primary target. Some parts of FreeType used in Chrome and Android may also need to be replaced. If we do both the stream of fuzzer issues _should_ stop.

### A productive codebase

We seek to provide a developer friendly codebase. For example, exploration of new structs for something COLRv1 should be readily achievable by forking and hacking around. That means the code needs to be simpler than today's C++ and as close as possible to Python level development velocity. To achieve this we will seek to establish an efficient pattern to support our two primary usage patterns. This will require that we:

   * Stay DRY
      * Don't hand-write things twice, once for high performance readonly access and once for mutable access
      * Don't hand-write per-field parsing (struct.blah = read type of blah, etc)
         * It's OK to *occasionally* hand-write, particularly for some of the more obtuse parts, but it should be the exception not the rule 
         * Try to continue the HarfBuzz and FontTools style of declaring the shape of things and letting automation take over wherever possible
      * Automated authoring, such as by macro or build time tooling is acceptable
   * Minimize unsafe code
   * Invest in testing

### Prefer incremental delivery, early production use

We will _prefer_ to:

* Make incremental progress, with each milestone being used in production, rather than spend years with parallel increasingly divergent implementations.
* Test the Rust implementation against the non-Rust implementation for both correctness and performance
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

## Definitions

### zerocopy

The direct reading/writing of scalars and, wherever possible, aggregates (structs and slices) through reinterpretation of pointers to raw bytes in memory.

## References

* https://security.googleblog.com/2021/09/an-update-on-memory-safety-in-chrome.html offers commentary on the feasibility of having Rust-like safety in C++
* https://pngquant.org/rust.html offers an interesting example of a Rust migration of a small library
* http://dtrace.org/blogs/bmc/2018/09/28/the-relative-performance-of-c-and-rust/ another interesting example

