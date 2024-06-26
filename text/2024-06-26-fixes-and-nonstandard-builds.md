# GF pre- and post-processing requirements

When producing fonts, the compiler is only one tool in the build chain.
This document explains what other tools are needed and why.

We split this into two areas: standard builds, which comprise the
majority of fonts in the GF library, and custom builds, which are a
minority but which still need supporting in a Rust toolchain.

## Standard builds

A standard build happens when `gftools-builder` is used with no custom
configuration file, or with a little "light" configuration: options
provided by the default `googlefonts` recipe provider.

The `googlefonts` recipe provider builds multiple targets, some of which
are for the benefit of the font designer - designers often like to
distribute OTF, TTF, and WOFF static fonts in their own repositories and
releases, alongside the VF - and some are for GF onboarding: the VF/VFs
in a variable font design, or autohinted statics in the case of a
non-variable design.

Google Fonts has a tight set of [requirements](https://googlefonts.github.io/gf-guide/) for a binary font
before it can be onboarded. We use `fontbakery` to check the fonts
against those requirements. Some of these requirements can be met by
altering the sources, but some cannot; and it is not always possible
to alter the sources (for example, if we are onboarding a non-commissioned
font where the upstream designer wants additional VF instances and we
want to avoid forking the source).

To help meet these requirements, after a font is compiled, the
`googlefonts` recipe provider runs the following post-production steps:

* `ttfautohint` (for a non-variable design)
* `gftools-fix-font`/`gftools-fix-family` (binary hotfixes)
* `gftools-gen-stat` (add a compliant `STAT` table)

### What does `gftools-fix-family` do?

`gftools-fix-family` applies fixes both to font binaries considered
independently and as part of a family (typically, Regular and Italic VFs
in the same family). Some of these fixes are considered "source fixes"
in that they are possible to apply to the sources, but others are not.

The core fixes are:

* Ensuring that the OS/2 version is set to 4.
* For an OFL font, setting `name` table entry 13 to the latest OFL
  description and entry 14 to the latest OFL URL. (This could be
  done at source level; I'm not sure why it isn't considered a source fix.)
* If the font is hinted, setting flag 3 on the `head` table; if the font is
  not hinted, adding a smooth `GASP` table and simple `prep` program. Both
  of these are needed to improve the appearance of the font on Windows.
* Adding a `name` table entry 25 for variable fonts. This is very GF
  specific in that it is constructed using the fallback names for axes
  in the GF axis registry.
* Ensures that GID 1 in a COLRv0 font has no outlines, to avoid a
  [rendering bug in Windows 10](https://github.com/googlefonts/gftools/issues/609). For COLRv1, run `maximum_color`.
* Ensures that `hhea.caretSlopeRise` and `hhea.caretSlopeRun` are
  correct: for a font with `post.italicAngle` != 0, `hhea.caretSlopeRise`
  should be upm, and `hhea.caretSlopeRun` should be `upm × tan -θ`.
* Reducing the named instances on the `fvar` table to the
  [set supported by Google Fonts](https://googlefonts.github.io/gf-guide/variable.html#fvar-instances).

The source fixes are:

* Removing [extraneous tables](https://github.com/googlefonts/gftools/blob/344853290c7c52672ef77b1f5347a31f999176d9/Lib/gftools/fix.py#L89-L101). (I'm not sure why this is considered
  a source fix, since most of them are added by compilers or editors)
* Rewriting the `name` table so that it conforms to
  [GF requirements](https://googlefonts.github.io/gf-guide/statics.html),
  particularly regarding subfamily name, typographic subfamily name, etc.
  Similar to `name` table 25 above, this is dependent on the axis registry.
* Setting `OS/2.fsType` to 0 and `OS/2.fsSelection` to `USE_TYPO_METRICS`
  plus the correct RIBBI bits; also setting `head.macStyle` to the correct
  RIBBI bits.
* Setting `OS/2.usWeightClass` to the correct value for the default weight
  style (VF) or static weight style, expected by the GF static instance.
  (This is needed if upstream interprets the CSS weight value / named
  instance progression differently to the way that GF does.)
* Fixes `post.italicAngle` to make it consistent with `hhea.caretSlopeRise`
  and `hhea.caretSlopeRun`. (This may seem inconsistent with the above
  core fix, but both are needed.)
* For fonts which already exist on GF, inheriting the vertical metrics
  from the production fonts; otherwise, setting the vertical metrics
  according to [our requirements](https://googlefonts.github.io/gf-guide/metrics.html). (The vertical metrics requirements
  are complex and are worth reading in detail.)

Many of these fixes can be applied automatically by an "opinionated
compiler" which contains multiple "profiles" containing expectations
for the deliverable (in our case, there would be a `googlefonts`
profile which embeds the knowledge about our requirements), and
overrides defaults to produce table values in compliance with this
profile.

### Compliant STAT tables

Google Fonts also has [complex requirements](https://googlefonts.github.io/gf-guide/variable.html#stat-table) for the `STAT` table, and
there is currently no way to specify the desired contents of this table
in font editors. This is why we have created the `gftools-gen-stat`
utility to add the `STAT` table. `gftools-gen-stat` either takes a YAML
description of the desired `STAT` table entries, or automatically
determines the desired entries based on the font and the data in the axis
registry. (I don't know why someone would specify a `STAT` table in 
YAML when it could be done automatically.)

Correct `STAT` tables need to be generated with the whole family in mind,
not just on generating a single binary, because `STAT` tables in Roman
and italic binaries need to link to one another.

## Custom builds

Some fonts still don't quite do what we want, even with the `googlefonts`
recipe provider and its fixes. For these we either use a custom recipe
provider or we specify the build steps individually. Builder2 has created
a language and a set of "operations" to specify the build process,
allowing fonts with custom requirements to be built up from multiple
operations.

### Noto

The most common custom build is the Noto project, which has a number of
unusual requirements, partly because different end-users require different
things from the deliverables:

* For the Google Fonts deliverable, we use `ufomerge` to merge in a Latin
  core from Noto Sans or Noto Serif at the source level before compiling.
  We apply the fixes above to help make these deliverables compliant with
  GF requirements.
* For the Android deliverable, we use `hb-subset` to slim the variable
  font to the range `wght=400:800` and drop any width axis.
* To support other end-users (Linux distros etc.), we also supply a
  matrix of
    - variable fonts, unhinted TTF, hinted TTF and OTF
    - full (+Latin) and standard (no Latin)

### Color fonts

Some of our fonts (Kalnia Glaze, Bitcount, SixtyFour Convergence, etc.)
use `paintcompiler`, a tool for adding a COLRv1 table programatically
from a Python script. For example, a designer can say "paint this
glyph according to a gradient which starts at the origin and ends at
the top right corner of the bounding box" without having to manually
calculate the values for each glyph (and recalculate them when the
outlines change). See for example [Kalnia Glaze paint definitions](https://github.com/fridamedrano/Kalnia-Glaze/blob/main/sources/paints.py).

This `paintcompiler` step is added to the end of the standard build
process by modifying the recipe produced by the `googlefonts` recipe
provider; again see the [Kalnia Glaze config file](https://github.com/fridamedrano/Kalnia-Glaze/blob/main/sources/config.yaml).

### "Many-from-one" fonts

In some cases we produce multiple fonts using variants derived
from a single source. The most complex example of this is the Playwrite
font, although other handwriting fonts (Edu AU Hand families etc) also
follow this model.

To take Playwrite as example, each of 56 language-specific Playwrite fonts
(and potentially another 56 Playwrite Guides fonts in the future) are
generated from a single source file, using a build plan orchestrated by
a gftools-builder recipe provider called `fontprimer`. `fontprimer` is
given information about how a variant is derived, and then determines
the build steps to make it.

To build Playwrite Argentina, we first build the main Playwrite VF
according to the googlefonts requirements above, and then:

* Use `fonttools varLib.instancer` to select particular values of the
  `YEXT`, `SPED` and `slnt` axes, to produce a partially-instanced VF.
* Perform a "deep" remapping of the `cmap` table such that, for example,
  `A` now maps to the cursive capital `A.cur`. ("Deep" remapping also
  replaces `A` with `A.cur` in layout rules.)
* Remap the GSUB layout tables such that layout rules in `locl` and
  `ccmp` are added to the `calt` feature. (A fix for Adobe InDesign.)
* Use `hb-subset` to remove non-essential glyphs. (In this case, we
  no longer use the standard non-cursive `A` glyph.)
* Fix the `post.italicAngle` post-subsetting. (Running `varLib.instancer`
  on a font with `slnt` axis does *not* produce an italic font.)
* Run `gftools-fix-font` again to update the name table / `hhea` table
  as appropriate.
* Use `gftools-rename-family` to the family to `Playwrite Argentina`.

A separate build target uses `fontprimer.guidelines` to modify the source
to add guidelines to each glyph, and then does the above steps to produce
a separate "Playwrite Argentina Guides" family.

Similar processes are used to build other handwriting families, although
the specific steps used to select a variant may differ. (For example,
we derive the variants of Edu AU Hand families by baking in different
stylistic sets instead of partially instancing and remapping.)

### Other challenging build situations

This is not an exhaustive list of the build requirements. See also,
for example:

* [BitCount](https://github.com/petrvanblokland/TYPETR-Bitcount), where multiple source UFOs are generated programatically
  as variants from a master UFO, and which also uses `paintcompiler`.
* [Noto Duployan](https://github.com/notofonts/duployan/), where the font has no sources in the normal
  sense but is built entirely from a Python script.
* [Noto Sans Math](https://github.com/notofonts/math), which has a `MATH` table. (Now part of the
  `glyphsLib`/`ufo2ft` build chain, but previously added manually
  with `ttx`)
* [Sawarabi Mincho](https://github.com/googlefonts/sawarabi-mincho/blob/main/sources/config.yaml), which has FontForge sources and uses `babelfont`
  to compile them to Glyphs and then TTF.
* Noto CJK, where we ingest binaries.
