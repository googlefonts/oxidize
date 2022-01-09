# oxidize

## Why

We currently rely C++ for anything high performance, e.g. shaping or subsetting, and Python for general purpose manipulation. The results are exactly what you'd expect:

* Python code (fonttools, fontmake, nanoemoji, etc) have high development velocity, relatively fast developer rampup, and runtime safety, slow runtime
* C++ code (HarfBuzz) has fast runtime, very slow development velocity, very slow developer rampup, and any safety it might have is due to extensive fuzzing
   * It's entirely normal to make a seemingly safe change and then see a stream of fuzzer bugs

Some logic ends up implemented in both languages.

Rust appears to offer us the ability to implement fast, safer code with development velocity in between Python and C++. This is a very appealing offer.

## Goals

1. We want to move shaping (HarfBuzz) and font manipulation (compilation, random inspection, arbitrary manipulations of existing fonts) to Rust.
   * To support shaping we likely want to support readonly mmap access.
   * Crates like zerocopy and the like suggest this is possible. Establishing a clean pattern here should be tackled early ([poc exploration](https://github.com/rsheeter/rust_read_font))
1. We do NOT want:
   * To hand-write things twice, once for high performance readonly access and once for mutable
   * To hand-write per-field parsing (struct.blah = read type of blah, etc)

