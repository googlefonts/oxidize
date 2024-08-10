# Mr B goes to Vartown

Or, a brief history of font compilers. Or, why fontmake. Or, It happened at ATypI.

Written in 2023, mistakes are @rsheeter's fault. Anything useful came from someone else.

What happened between 1976 and 2013 is underdocumented. PRs welcome.

The pseudonym Mr B is used because @rsheeter thinks its funny to refer to Behdad Esfahbod. YMMV.

* [ATypI 1976](#atypi-1976)
* [1989 Multiple Master](#1989-multiple-master)
* [The 90's, TrueType GX](#the-90s-truetype-gx)
* [1994 Skia Font](#1994-skia-font)
* [2000's](#2000s)
* [2004](#2004)
* [Late 2013 to summer 2014](#late-2013-to-summer-2014)
* [2014 to summer 2015](#2014-to-summer-2015)
* [ATypI 2014](#atypi-2014)
* [2015](#2015)
* [ATypI 2015](#atypi-2015)
* [2016](#2016)
* [ATypI 2016](#atypi-2016)
* [2017..2022](#20172022)
* [2023](#2023)

## ATypI 1976

In "Font Technology: Methods and Tools" Peter Karow mentions that in 1976 there was discussion of
the IKARUS program having a new trick: it can interpolate. This is the earliest written mention
of what ultimately becomes variable fonts the author has identified. However, it is ahead of time
as opposed to on the client.

## 1989 Multiple Master

Adobe demonstrates Multiple Master (MM) fonts.
They interpolate from corner masters and do not encode the default.
Multiple Master instantiation is done at runtime instead of ahead of time, although [Adobe Type Master (ATM)](https://en.wikipedia.org/wiki/Adobe_Type_Manager) offered ahead-of-time export.

ATM is very popular at this time. It hooks font-related system calls and is thus able to make MM
Just Work in a variety of apps, albeit with limited capability. A few apps, most notably Illustrator
and Photoshop, shipped with some degree of direct support - sliders but no auto optical sizing? - 
but overall user application support never really took off. Font making applications did support MM
in the mid-late 90s, but initially it was a proprietary format with an in-house font editor.

The Adobe Type team is excited about optical size, but the primary business reason to invest in MM is
the ability to substitute Adobe Sans and Serif when generating or rendering a PS or PDF file that would otherwise end up
missing a font. You have the "correct" font, perhaps not allowed to be embedded, and generate a good
substitution, with similar metrics, cap height, weight, slope, etc metadata. 
Or you have a PDF that doesn't contain a fully embedded font, but has that metadata, so Adobe Acrobat generates suitable instances to render it.
Adobe Sans goes first, derived from Myriad, and shipped in "Super ATM" in '90 or '91. Adobe Serif shows up in the next version.

Two major versions of Multiple Master were shipped, which we'll creatively call first and second:

* v1
   * Corner masters only.
   * Linear interpolation.
   * Optical size support, though no optical size font shipped until [Minion](https://en.wikipedia.org/wiki/Minion_(typeface)) (in '92).
   * You have to define all the corners so you need 2^N masters.
   * To keep things practical there is a hard coded limit of 4 axes, requiring 16 masters.
   * Piecewise linear remapping of coords, roughly avar1.
* v2
   * Adds support non-corner masters, referred to as "intermediate masters".
   * Notably used for [Adobe Jenson](https://en.wikipedia.org/wiki/Adobe_Jenson), shipped in '96 with weight and optical size.

At some point MM also added support for fencing off portions of the design space, notably used to keep
users away from the small, bold, condensed corner of Kepler (weight, width, optical size).

The MM format was abandoned during the move to OpenType, though the concept of a runtime variable font lived on.

## The 90s, TrueType GX!

Mike Reed, of [Skia](https://news.ycombinator.com/item?id=16146132#16147723) fame, looks at Multiple 
Master v1 and has a key insight, effectively inventing modern variable fonts: We shouldn't _have_ to start
at a corner, we should pick some point and encode deltas from it.
Like multiple master, TrueType GX fonts are instantiated at runtime.

It falls out of this approach that you can have lots of axes, the 4 axis limit is shattered!
This is variable fonts as announced at ATypI 2016. We just took a 20-ish year break.

For context, Apple owns it's platform. This enables Apple to invent and ship a format.

At about the same time TrueType - as an upgrade from bitmap fonts - is being sold as a disk space
win for printers. GX joins the fun, throw out your box of floppies and use this 40KB font that
supports all point sizes.

As was the style at the time, Apple doesn't open source any of this. The documentation is described
as poor. This probably contributes to the delay in today's variable fonts shipping.

References:

1. https://en.wikipedia.org/wiki/QuickDraw_GX#TrueType_GX
1. https://www.monotype.com/resources/articles/part-1-from-truetype-gx-to-variable-fonts
1. https://www.monotype.com/resources/expertise/truetype-gx-variable-fonts 

## 1994 Skia Font

[Skia](https://en.wikipedia.org/wiki/Skia_(typeface)) ([specimen](https://www.axis-praxis.org/specimens/skia)) is the first significant variable font
to ship. Characterized as a "QuickDraw GX" font, it has weight and width variation, demonstrated
complete with a slider UI in https://vimeo.com/120047887.

David Berlow was contracted by Apple to make fonts. Lightly paraphrased, multiple sources agree that
There's no single designer who knows the ins and outs of variable font design better than Berlow,
he's lived it since the 90's.

Berlow's famous animated lizard, shipped in [Zycon](https://www.axis-praxis.org/specimens/zycon),
is born in the mid-90's.

## 2000's

FreeType reverse engineers TrueType GX.

## 2004

[Superpolator](https://superpolator.com) is released in 2004.
Under the hood it uses the python MutatorMath library.

## Late 2013 to summer 2014

Adam Twardoch proposes the resurrection of Multiple Master to the W3C Webfonts Working group
[thread](https://lists.w3.org/Archives/Public/public-webfonts-wg/2013Jun/0024.html), observing
that the dynamic context of the web may benefit more than application UI or print. 

Google is working on Noto phase 3. So far we're collecting binaries.
It's not what would traditionally be viewed as open source because we don't have the
"source" in the Apache 2 sense of the preferred form for making modifications.

> "Source" form shall mean the preferred form for making modifications,
> including but not limited to software source code, documentation source, and configuration files.
>
> https://www.apache.org/licenses/LICENSE-2.0

This bothers Mr B. We need the souces to be properly open source and to do cool things
like variable fonts. Provision of sources becomes part of Noto phase 3.

## 2014 to summer 2015

A List Apart publishes [Variable Fonts for Responsive Design](https://alistapart.com/blog/post/variable-fonts-for-responsive-design/).

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

FontTools has a lot of the hard parts, and is open source, Mr B decides to build on that. fontmake is born! It knows how to build variable fonts from inception.

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
      * Before defcon we used RoboFab (https://github.com/robotools/robofab)
   * [ufoLib2](https://github.com/fonttools/ufoLib2) is created to provide a more compiler-friendly UFO accessor
      * Built on top of ufoLib
* Dependency excitement
   * Python 2 vs 3 is in full swing, library support is inconsistent
   * UFO 3 is new, not everything supports it yet

To build these new fangled variable font things the `.designspace` file and [designspaceLib](https://fonttools.readthedocs.io/en/latest/designspaceLib/index.html)
(by @letterror) are born. It is merged into FontTools to mitigate dependency excitement and to retain the
idea that FontTools can do all the basic font things in your life.

Variable fonts can be built but it's painful. You run fontmake to produce ttf-interpolatable files then manually run
varLib on them. Only a single axis of variation is supported so varLib is relatively simple.

@simoncozens [favorite part](https://simoncozens.github.io/compiling-variable-fonts/#merge)
of font compilation has arrived!

MutatorMath is used to produce instances.

## ATypI 2014

In the fall of 2014 several key things are made public at ATypI Barcelona:

* Mr B announces the intent to ship many of the key pieces you'd need for an open source compiler [slides](https://github.com/behdad/slippy/blob/master/fonttools/fonttools_slides.py)
* [MutatorMath](https://github.com/LettError/MutatorMath) goes open source
* Adobe announces AFDKO will become open source

Mutator math gives you everything you need to compute instances from a variable font. Now that's public and open source ... what if we did the interpolation on the client? - this is essentially what todays variable fonts are.

## 2015

fontmake begins to come together. It can directly build variable fonts, you don't need a multi-stage manual pipeline.

@brawer implements [gvar](https://learn.microsoft.com/en-us/typography/opentype/spec/gvar) in FontTools.

## ATypI 2015

Mr B pitches this cool idea he has about client-side interpolation. Fonts that vary. We could call them variable fonts perhaps?

Exploratory work, such as [glyphs2gx.py](https://github.com/behdad/playground/blob/master/glyphs2gx/glyphs2gx.py) starts to be published.

Mr B wishes he could have cool features in the font format, [OpenType BE](https://docs.google.com/document/d/1nwrs6_A0cyWPxiypxVMT06PvM8kr3hAQCD4CK2o5qI8/edit#heading=h.1kmpml9m1vxz). Some of them are actively in progress in 2023, https://github.com/harfbuzz/boring-expansion-spec.

## 2016

fontmake learns to directly create variable fonts from source. varLib learns to compute the deltas for multi-axis variable fonts, which means fontmake
can build such fonts.

That means we can build variable fonts, but there are pain points.

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

Mr B presents [slides](https://github.com/behdad/slippy/blob/master/fonttools2016/fonttools2016_slides.py) about the state
of the open source build pipeline. The focus is Noto but what's being built is general purpose.

## 2017..2022

The size and complexity of variable fonts keeps increasing. fontmake keeps working. Users wish it was faster and engineers wish it was simpler.

Many optimizations are applied to fontmake. Bits are cythonized, things happen in memory, ... users still wish it was faster.

MutatorMath begat https://github.com/LettError/ufoProcessor in 2017.

## 2023

We're still making fonts that are larger and more complex. fontmake still works. Users still wish it was faster.

Not to worry, we'll just [rewrite in Rust](https://github.com/googlefonts/oxidize):

* [fontc](https://github.com/googlefonts/fontc) aspires to be the next decades font compiler
* [skrifa](https://docs.rs/skrifa/latest/skrifa/) aspires to replace FreeType
* etc
