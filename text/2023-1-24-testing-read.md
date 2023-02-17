## Testing strategies for font parsing

Along with the safety guarantees provided by the Rust language, we also aim to 
provide robust testing support to ensure that we agree with the 
[specification](https://learn.microsoft.com/en-us/typography/opentype/spec/),
match the outputs of existing high quality implementations such as [FreeType](https://freetype.org/)
and [HarfBuzz](https://github.com/harfbuzz/harfbuzz), and avoid regressions.

### read-fonts: low level parsing

The [read-fonts](https://github.com/googlefonts/fontations/tree/main/read-fonts) crate consists primarily
of generated code and is limited to parsing so the tests are generally scoped to ensuring correct
extraction of typed values from binary font data. It contains inline unit tests for each table and subtable. 

These are generally in one of two forms:
* Scraped from examples in the specification such as this (`GPOS`) `SinglePosFormat2` subtable
[example](https://learn.microsoft.com/en-us/typography/opentype/spec/gpos#example-3-singleposformat2-subtable)
which is embedded in our [test data](https://github.com/googlefonts/fontations/blob/12fe04eb7083faaf8972720629496c0ca9a4b6c5/read-fonts/src/tests/test_data.rs#L17).
* Hand written and designed to test against a [curated collection of fonts](https://github.com/googlefonts/fontations/tree/main/resources/test_fonts). An example of this for the `STAT` table can be found [here](https://github.com/googlefonts/fontations/blob/12fe04eb7083faaf8972720629496c0ca9a4b6c5/read-fonts/src/tables/stat.rs#L13).

### skrifa: metadata processing and loading/scaling of glyph outlines

Unlike `read-fonts`, the `skrifa` crate provides additional processing of the parsed data and the
goal is to match the output of FreeType for metadata, metrics and loaded outlines.

#### Glyph loading/scaling

The testing strategy for this library is to use a pinned version of FreeType (currently 2.12.0) to generate
a text file containing the outline data in both raw (contours, points and tags as defined in
[FT_Outline](https://freetype.org/freetype2/docs/reference/ft2-outline_processing.html#ft_outline)) and 
decomposed path segment forms.

Each outline in the text file is represented by the pattern:
```
glyph <glyph-id> <font-size> <hint-mode>
coords <coords>
points <points>
contours <contours>
tags <tags>
m|l|q|c <points>
-
```
With the following values:
* `glyph-id`: the glyph indentifier
* `font-size`: size in pixels per em. A size of 0 means unscaled
* `hint-mode`: one of `none`, `full`, `light`, or `light-subpixel`
* `coords`: space separated list of normalized design coordinates, one per axis
* `points`: space separated list of points in `x, y` format
* `contours` and `tags`: space separated list of integers representing contour end point 
    indices and tag bits, respectively
* `m`, `l`, `q`, `c`: path commands for `move to`, `line to`, `quad to` and `cubic to`, 
    respectively

The pattern ends with a single `-`.

While testing, `skrifa` will read this text file and the associated font, loading each
specified glyph according to the parameters and compare the outline data. This is done 
for both raw outlines and paths, testing both our loading/scaling code and our path
decomposition code.

#### Metadata and metrics

The exact formats haven't been determined yet, but we aim to do similar FreeType extraction for
metadata and metrics. The specific concerns here are character map selection, reading localized
strings and extracting font and glyph metrics.

#### Exemplars

Our collection of test fonts and glyphs is currently small. Our plan to grow this is to write
a tool to generate a collection of exemplar glyphs that contain "interesting" characteristics
that exercise the more complicated parts of the loading process. In particular, we are interested
in glyphs that:
* Contain multiple levels of composite nesting
* Make use of the available component transform modes (`WE_HAVE_A_SCALE`, `WE_HAVE_AN_X_AND_Y_SCALE`, `WE_HAVE_A_TWO_BY_TWO`)
* Apply a scale to a component offset (`SCALED_COMPONENT_OFFSET`)
* Specify the `USE_MY_METRICS` flag to override base metrics with those of a component

This will be done with a python script to collect the appropriate glyphs from a large set of 
fonts. These will be pared down to the interesting cases and added to our test font resources.
