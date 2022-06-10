# Experiments with parallelising font compilation

Take from @simoncozens https://github.com/googlefonts/oxidize/issues/29 report.

We've discussed the idea of parallelising the build. I've done some experiments with this last week and again today, and here's what I found.

## What do we mean by parallelising?

When we talk about parallelising the build process, we mean using multiple threads or processes to run parts of the build at the same time - for example, building a set of master TTFs from master UFOs in parallel. There are two ways to approach parallelisation. The obvious first approach is apply parallization in Python within the compiler process; ie. where we have things like

```python
    for source in designSpaceDoc.sources:
        otfs.append(
            compileOTF(
                ufo=source.font,
```
(https://github.com/googlefonts/ufo2ft/blob/2886fa4b363d1245c3c1a745d397f625c5facaa0/Lib/ufo2ft/__init__.py#L436-L439)

to use either [multiprocessing](https://docs.python.org/3/library/multiprocessing.html) or [threading](https://docs.python.org/3/library/threading.html) to parallelise the loop.

However, this quickly fails. Threading doesn't work because Python has a Global Interpreter Lock and even within the context of threading only runs one piece of code at a time(!), and multiprocessing requires all relevant data to be pickled and unpicked across the process boundary, which, when we're dealing with UFOs in memory, is a serious PITA.

So instead the approach I've been looking at is using [ninja](https://ninja-build.org) to orchestrate the parallelisation of discrete tasks within the build process.

## First experiment: gftools-builder-ninja

 I first tried this with the [gftools.builder.ninja](https://github.com/googlefonts/gftools/pull/569/) module, which works well in the gftools-builder context, because in that situation, there are a large number of fairly large macro-tasks that need to be organised, and a large number of output artefacts, and generating these artefacts do map well to command-line tasks. We build OTFs, TTFs, web fonts and variable fonts out of multiple sources, so we can ask the existing fontmake compiler and other command line tools to first build the master UFOs, then the master TTFs, variable fonts, auto-hinting, WOFF compression, and so on.

Here is an example of orchestrating a gftools-builder run with the ninja module:
![ninja](https://user-images.githubusercontent.com/106728/173144711-9ff082a4-1370-46b6-9603-fb0ee0c27122.png)

This gave a speedup of around 3x over the equivalent gftools-builder run.

Encouraging, but obviously specialised to the gftools-builder workflow. Can we do better?

## Second experiment: fontninja

The challenge of extending this approach to the current compiler architecture is that the "discrete tasks" are scattered across (and share data between) methods in fontmake, ufo2ft, fontTools.varLib, cu2qu and elsewhere. The whole toolchain is written with the understanding that Python methods are going to be passing data around; to orchestrate these operations with a make-like tool requires them to be called as individual command-line tasks. So the operations need to be broken into their components and a command-line wrapper (together with any data loading and saving) wrapped around each one. 

So, the second experiment does just that. [fontninja](https://github.com/simoncozens/fontninja) is a sketch of a replacement for fontmake which uses the existing toolchain components but breaks them into discrete operations. The sketch had two immediate goals: 1) compile a single UFO to TTF, and 2) compile a variable TTF from a designspace file.

### What discrete operations are available?

To understand how fontninja works, you need to understand [what happens when you build a variable font with fontmake](https://simoncozens.github.io/compiling-variable-fonts/). Briefly the steps required are:

* Preprocess each UFO:
  * Where any glyphs contain both components and contours, decompose the components. (This is because TTF glyf table entries are either component glyphs or contour glyphs but not both.)
  * Remove overlaps.
  * Convert cubic curves to quadratic curves. **Important Note**: a cubic segment converts to *one or more* quadratic segments, so when compiling a variable font, curves must be converted across *all UFOs at once* so that the converted curves have the same number of segments. This is quite a bottleneck!
  * Write out a feature file making kerning and anchor attachment explicit.
* Compile each UFO into a basic binary TTF.
* Add the features from the feature file generated above.
* Merge all TTFs together into a variable font.

(This is not the only way of building variable fonts - you can also read all UFOs at once and build one TTF - but it's the way we have right now and the point is to use the existing toolchain as much as we can.)

fontninja contains "plumbing" commands (`fontninja --plumbing <command> <inputs> <output>`) for [each of the steps above](https://github.com/simoncozens/fontninja/tree/main/Lib/fontninja/plumbing). It also has a fontmake-style build front-end which writes a ninja file which in turn calls these plumbing commands. Here's what the ninja orchestration looks like:

![graph](https://user-images.githubusercontent.com/106728/173148695-a724491a-9c83-4112-a918-2f8b18849688.png)

Note that by wrapping each of these steps into a separate command-line utility, we are required to load data at the start of each step and save it at the end. For example, the "decompose mixed glyphs" step reads a UFO, decomposes the glyphs, and writes out the UFO to disk again; the next step reads the UFO, ...

It should not be a surprise that this serialisation and deserialisation overhead means that `fontninja`, despite the parallelisation, is actually slower than `fontmake`:

```
fontmake -o variable -m NotoSans.designspace: 1:32.79 total
fontninja -o variable -m NotoSans.designspace: 1:47.49 total
```

But only a little slower! But with our new architecture, do we have room to improve?

### Potential Rustification

We certainly do! Because we have now split the compilation of a font into discrete, encapsulated steps, we can now look at rewriting those steps in a faster language (Rust, obviously):

* [`ufo-decompose`](https://github.com/simoncozens/rust-font-tools/tree/main/ufo-decompose) can replace the Python `DecomposeComponentsFilter`.
* [`ufo-cu2qu`](https://github.com/simoncozens/rust-font-tools/tree/main/ufo-cu2qu) can replace the cu2qu step.
* [`fonticulus`](https://github.com/simoncozens/rust-font-tools/tree/main/fonticulus) can build variable fonts but it can't handle OpenType layout, but we *can* also use it to replace the "compile a UFO into a basic binary TTF" step.

Using all of these Rustified operations (using the `--rust` flag in fontninja to select them) speeds up the build to 67 seconds, so we're now faster than fontmake, but only marginally.

### Another gain

Of those 67 seconds, the vast majority are spent in the final step, which is calling `fontTools.varLib.merge`. Of *that*, 50% is spend optimizing the gvar table by calculating [inferred variation deltas](https://docs.microsoft.com/en-us/typography/opentype/spec/gvar#inferred-deltas-for-un-referenced-point-numbers). This is an optimization for production builds that we can just turn off, because it so happens that we already have a [Rust command line utility](https://github.com/simoncozens/rust-font-tools/blob/main/fonttools-cli/src/bin/ttf-optimize-gvar.rs) to perform this optimization step separately (which I literally wrote in preparation for this very day).

Final result:

* fontmake: 1 minute 32 seconds.
* fontninja with Python operations: 1 minute 47 seconds.
* fontninja with Rust operations: 1 minute 7 seconds.
* fontninja with Rust operations and the gvar optimization turned off: 24 seconds.