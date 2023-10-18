# Me B goes to Vartown

Or, a brief history of font compilers. Or, why fontmake. Or, It happened at ATypI.

Written in 2023, mistakes are @rsheeter's fault. Anything useful came from someone else.

The pseudonym Mr B is used because @rsheeter thinks its funny. YMMV.

[TOC]

## ATypI 1976

In "Font Technology: Methods and Tools" Peter Karow mentions that in 1976 there was discussion of
the IKARUS program having a new trick: it can interpolate. This is the earliest written mention
of what ultimately becomes variable fonts the author has identified.

## Late 2013 to summer 2014

Google is working on Noto phase 3. So far we're collecting binaries.
It's not what would traditionally be viewed as open source because we don't have the
"source" in the Apache 2 sense of the preferred form for making modifications

> "Source" form shall mean the preferred form for making modifications,
> including but not limited to software source code, documentation source, and configuration files.
>
> https://www.apache.org/licenses/LICENSE-2.0

This bothers Mr B. We need the souces to be properly open source and to do cool things
like variable fonts. Provision of sources becomes part of Noto phase 3.

## 2014 to summer 2015

Mr B knows sources are coming! He's excited! But ... how do you _build_ those sources? We're going to
go from binaries with no sources to sources+binaries and no way to compile a binary from the sources.

We need a compiler! And the compiler needs to build variable fonts. So, what's available to make a compiler from?

* [AFDKO](https://github.com/adobe-type-tools/afdko) exists and has a CFF compiler and feature compiler
   * Free (might require a license agreement?)
     but not open source
* Authoring Tools read their respective formats and include compilers
   * Some may bundle the AFDKO to do some or all of the job
* [https://github.com/fonttools/fonttools](https://github.com/fonttools/fonttools) exists and can read/write
fonts
   * It doesn't read any source format
   * It is open source
* [fontforge](https://fontforge.org/) exists, is open source, and has a compiler
   * It is not noted for being easy to work with

FontTools has a lot of the hard parts, and is open source, Mr B decides to build on that. fontmake is born! It knows how to build
variable fonts from inception.

* Mr B writes glyphsLib to read `.glyphs` plist files
* Mr B teaches the system to build a TrueType outline based variable font
   * We need quadratics so [cu2qu](https://fonttools.readthedocs.io/en/latest/cu2qu/index.html) is born
   * [varLib](https://fonttools.readthedocs.io/en/latest/varLib/index.html) is born to figure out the low level representation of variations
* @jamesgk, working with Mr B, writes the first edition of fontmake
* @brawer writes [feaLib](https://fonttools.readthedocs.io/en/latest/feaLib/index.html) so we can compile .fea files in an open source stack

There are a few rough spots:

* A path ops library that runs on polygons not curves
* Reading [Unified Font Object](https://en.wikipedia.org/wiki/Unified_Font_Object)'s is slow
   * fontmake is using defcon which is built for use in editors
   * defcon uses ufoLib, the reference implementation of UFO by @typesupply
      * [ufoLib](https://fonttools.readthedocs.io/en/latest/ufoLib/index.html) lives on in FontTools
   * [ufoLib2](https://github.com/fonttools/ufoLib2) is created to provide a more compiler-friendly UFO accessor
      * Built on top of ufoLib
* Dependency excitement
   * Python 2 vs 3 is in full swing, library support is inconsistent
   * UFO 3 is new, not everything supports it yet

To build these new fangled variable font things the `.designspace` file and [designspaceLib](https://fonttools.readthedocs.io/en/latest/designspaceLib/index.html)
(by @letterror) are born. It is merged into FontTools to mitigate dependency excitement and to retain the
idea that FontTools can do all the basic font things in your life.

varLib is built to take compatible static fonts and merge them into a variable font because that lets you generate
a variable font from existing statics. fontmake builds on this. @simoncozens [favorite part](https://simoncozens.github.io/compiling-variable-fonts/#merge)
of font compilation is here!

## ATypI 2014

In the fall of 2014 several key things are made public at ATypI Barcelona:

* Mr B announces the intent to ship an open source compiler
* [MutatorMath](https://github.com/LettError/MutatorMath) goes open source
* Adobe announces AFDKO will become open source

Mutator math gives you everything you need to compute instances from a variable font. Now that's public and open source ... what if we
did the interpolation on the client? Variable fonts as we know them are born.

## ATypI 2015

Mr B pitches this cool idea he has about client-side interpolation. Fonts that vary. We could call them variable fonts perhaps?

Exploratory work, such as [glyphs2gx.py](https://github.com/behdad/playground/blob/master/glyphs2gx/glyphs2gx.py) starts to be published.

Mr B wishes he could have cool features in the font format, [OpenType BE](https://docs.google.com/document/d/1nwrs6_A0cyWPxiypxVMT06PvM8kr3hAQCD4CK2o5qI8/edit#heading=h.1kmpml9m1vxz). Some of them are actively in progress in 2023, https://github.com/harfbuzz/boring-expansion-spec.

## 2016

We can build variable fonts but there are pain points.

* @twardoch thinks it would be cool to directly write variable features (https://github.com/adobe-type-tools/afdko/issues/153)
* fontmake likes to write statics to disk before it merges them; it's slow
   * @anthrotype makes it happen in memory

Mr B writes [OpenType GX, an exploratory proposal](https://docs.google.com/document/d/1sDXjGjsOr40nPkatwuQBbreSzGDfg-8_ZBCh7xK9Bk4/edit#heading=h.1kmpml9m1vxz).
The impact of libraries and tools becoming available is noted:

> Interpolation has become common in recent years, thanks to tools like Superpolator / MutatorMath, as well as modern font editors like Glyphs and RoboFont

Microsoft, Adobe, Monotype, Google - henceforth The Cabal - meet in April to kick off a variable fonts program. They meet monthly until a September announcement.

## ATypI 2016

[The return of Multiple Master](https://typenetwork.com/articles/atypi-warsaw-2016-day-2-part-i-the-return-of-the-multiple-master)
steals the show! The Cabal presumably does a victory lap.

John Hudson writes the best known explanation of what the new fangled thing actually is in [Introducing OpenType Variable Fonts](https://medium.com/variable-fonts/https-medium-com-tiro-introducing-opentype-variable-fonts-12ba6cd2369).

## 2017..2022

The size and complexity of variable fonts keeps increasing. fontmake keeps working. Users wish it was faster and engineers wish it was simpler.

Many optimizations are applied to fontmake. Bits are cythonized, things happen in memory, ... users still wish it was faster.

## 2023

We're still making fonts that are larger and more complex. fontmake still works. Users still wish it was faster.

Not to worry, we'll just [rewrite in Rust](https://github.com/googlefonts/oxidize):

* [fontc](https://github.com/googlefonts/fontc) aspires to be the next decades font compiler
* [skrifa](https://docs.rs/skrifa/latest/skrifa/) aspires to replace FreeType
* etc
