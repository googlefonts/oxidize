# Migrate the glyph loop to Rust

Tl;dr @rsheeter wants to move the fontmake glyph loop to Rust. 

* [Problem statement](#problem-statement)
* [Proposal](#proposal)
* [Context](#context)
   * [How do you make a variable font anyway?](#how-do-you-make-a-variable-font-anyway)
   * [How do you do it faster?](#how-do-you-build-faster)
* [References](#references)

## Problem statement
We have builds that take tens of minutes or even hours with humans waiting on them (https://github.com/googlefonts/oxidize/pull/22).
This makes font development projects slower and more expensive.

To make font development faster and cheaper, make all font builds at least an order (preferably two) faster

   * A two order speedup would make edit/build/test loops possible where today you take rest of the day off
   * Bonus points for using less resources; today large builds exhaust github runners

Prior art around moving subsetting from Python to C++ suggests running two orders faster while consuming an order less resources is plausible.

For maximum impact, do it horizontally, impacting as close as possible to all Google Fonts projects without having to touch them one by one to alter their bespoke build processes. If we must touch them, we should move them to a consistent build structure.

Bias to heavy lifting done in Rust as part of Oxidize. A layer of Python orchestration is acceptable.

Bias to incremental progress, no big bang integration.
Writing a new compiler in isolation while everyone continues to use fontmake, and then abruptly switching projects over
(big bang integration) delays impact and increases risk.
An approach that incrementally improves fontmake has significant appeal.

## Proposal

Migrate the core for each glyph loop
([varLib._add_gvar](https://github.com/fonttools/fonttools/blob/455158f2bfd9eae2135a5f85bb96549a545cac82/Lib/fontTools/varLib/__init__.py#L222))
in fontmake from Python to Rust.

* Expose Rust to Python.
* Let Rust handle the parallelism.
* Build directly from UFO sources, optionally passing through some intermediary representation.
* Do NOT use the static TTFs at master locations to build glyph tables.
  * the static TTFs can still exist, but their glyph work should be brought as close as possible to zero (empty glyphs?)

The next step is to get approval from the fontmake owners to make such changes. @anthrotype basically :D

## Context

### How do you make a variable font anyway

Complicating this, the core approach we take to build a variable font is excitingly indirect (https://simoncozens.github.io/compiling-variable-fonts/):

   1. Generate static TTFs at master locations
   1. Merge those TTFs to create a final VF

As far as the author can tell there is no requirement for the statics to ever exist, we do it this way purely because
it was a relatively simple way to get fontmake to support VF. In the end we should go from sources → final VF without this step.

A concrete example may help, consider the N in [Josefin Slab](https://fonts.google.com/specimen/Josefin+Slab)
(https://github.com/TypeNetwork/Josefinslab) upright:

* [sources/JosefinSlab-Thin.ufo/glyphs/N_.glif](https://github.com/davelab6/josefinslab/blob/master/sources/JosefinSlab-Thin.ufo/glyphs/N_.glif) defines the curve at thin (N)
* [sources/JosefinSlab-Bold.ufo/glyphs/N_.glif](https://github.com/davelab6/josefinslab/blob/master/sources/JosefinSlab-Bold.ufo/glyphs/N_.glif) defines the curve at bold (N)
* [sources/JosefinSlab.designspace](https://github.com/davelab6/josefinslab/blob/master/sources/JosefinSlab.designspace) defines how the parts fit together
   * JosefinSlab-Thin.ufo is at weight 14
   * JosefinSlab-Bold.ufo is at weight 83
   * Font weights are mapped onto the space defined by the UFO
      * Default 100 maps to 14
      * 300 maps to 28, 400 to 41, etc
         * These aren't where you'd land by lerp, we need to watch how we set up interpolation

### How do you do it faster

#25 suggests that build time scales significantly with two things:

1. The amount of per glyph work, #glyphs * #masters, the primary predictor of build time
   * Makes sense, we do that work glyph by glyph ([varLib._add_gvar](https://github.com/fonttools/fonttools/blob/455158f2bfd9eae2135a5f85bb96549a545cac82/Lib/fontTools/varLib/__init__.py#L222)) and the per glyph work scales directly based on how many masters you have to worry about
1. The complexity of layout, which for complex scripts can be a very significant factor

Paraphrasing things Simon Cozens’ has claimed for some time, you do three things:

1. Do the work faster, such as by doing it in Rust instead of Python
1. Do the work concurrently wherever possible
1. Compute the final result as directly as possible
   * Don't do things like build a binary font for each master

Since we convert glyphs files to UFO lets focus on how a UFO should build (the actual code should abstract some of the concepts):

1. Concurrently process each glyph, plus the feature file(s)
   1. For Josefin Slab, process all N_.glif files together to produce the glyph and varstore you need for just N
      1. `f(all N_.gif, JosefinSlab.designspace) → glyf,gvar for the N`
   1. Merge all the feature files and compile them
      1. Shouldn't need final outlines to compute features
      1. `f(groups.plist, kerning.plist, features.fea, glyphs/contents.plist) → layout table(s)`
1. After all glyphs are done, merge per-glyph parts to form final glyf,gvar
   1. This needn't wait on feature file compilation, though it likely could without that much harm
1. Merge the final glyph parts and layout, with some care to ensure gids align

That gets you the most expensive parts of the font compiled directly to VF with significant parallelism.

**OK, but I need it in fontmake. Incrementally.**
Two major paths to incrementally march fontmake toward the end goal come to mind:

1. Orchestrate processes; serial Python → ninja → Rust
   * Break fontmake into processes, run those processes with ninja
   * Migrate processes to Rust incrementally, potentially playing with the execution graph along the way
1. Rust Python modules; expose Rust, such as via PyO3
   * Under the hood Rust can take advantage of parallelism
   * We can avoid writing things to disk for handoff as we must with ninja

Both options should work. For example, we could:

1. Implement compilation of individual variable glyphs  from {glif files for glyph} in Rust
   * Optionally introducing an intermediary abstraction, with an eye to eventually compiling directly from glyphs instead converting glyphs to UFO first
   * At a glance the trickiest part appears to be implementing IUP optimization
1. Implement parallel compilation of many glyphs at once in Rust
1. Alter the fontmake master compilation process to (initially opt-in):
   * Exclusively produce blank or placeholder glyphs in the per-master TTF files
   * Build the final VF
   * Invoke Rust to compile the final variable glyphs directly into the final VF

Switching feature compilation would be similar. Building a full Rust feature compiler is a larger chunk of work with lower impact so doing glyphs first makes sense.

The author believes this suggests it is very tractable to incrementally improve fontmake while also building up the parts to ultimately have a complete Rust compiler. 

## References

1. [Simon already did it](https://github.com/simoncozens/rust-font-tools/blob/723a47c6b92dc2dfcdcb558627f7549edb26d13b/fonticulus/src/buildbasic.rs#L94-L154.)
   * We may wish to rewrite against Oxidize and Norad to avoid having multiple definitions of core font structures and UFO parsing