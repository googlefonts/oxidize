# Productionizing HarfRuzz

Rough plan for bringing the [HarfRuzz](https://github.com/harfbuzz/harfruzz)
crate up to production quality.

The phases below are listed in logical order but much of the work can be
done in parallel (i.e. adding tables to `read-fonts` while bringing the repo
up to date).

## 1. Bring in updates from RustyBuzz

Since the fork, we have fallen behind [RustyBuzz](https://github.com/harfbuzz/rustybuzz)
which has had several updates that track changes to HarfBuzz.

There are two possible approaches:
* Replay the new commits over the current HarfRuzz tree
  * Likely to have some gnarly merge conflicts
  * Simon tried this and gave up
* Refresh the HarfRuzz tree with RustyBuzz current and reapply
  the Fontations changes

My current inclination is to do the latter as it is less likely to lead to
(perhaps unknown) broken code.

## 2. Bring in updates from HarfBuzz

RustyBuzz main matches HarfBuzz 10.1 but the current release of HarfBuzz is
11.0. 

## 3. Add missing AAT tables to `read-fonts`

We're currently missing the following.

* [trak](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6trak.html) -
  the tracking table
* [morx](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6morx.html) -
  the extended glyph metamorphosis table
* [kerx](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6kerx.html) - 
  the extended kerning table
* [kern](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6kern.html) -
  the kerning table
  * We already have partial support for the simplest format

We only need parsing for these tables as the complex logic to drive them
already exists in HarfRuzz.

## 4. Fully remove the `ttf-parser` dependency

And replace with `read-fonts` equivalents.

* Read classes and mark coverage from our `GDEF` table
* Replace references to remaining AAT tables listed above
* Replace `ttf_parser::GlyphId` with `font_types::GlyphId`
* Use our user to normalized variation coordinate conversion
* Use `read-fonts` for computing glyph metrics

Some of this work might suggest adding new APIs to `read-fonts`. For example
glyph metrics probably has some overlap with code currently in `skrifa` and
it would make sense to share that.

## 5. Reconsider naming and repo structure

RustyBuzz applied an ["uglification"](https://github.com/harfbuzz/rustybuzz/pull/99)
pass to make the codebase as close as possible to HarfBuzz to facilitate
easier updates. This included both module and type names.

I suspect that we can retain the easy-to-update situation without exactly copying HarfBuzz
names. For example, it should be simple for any developer applying an update
to associate `hb_ot_apply_context_t` with `OtApplyContext`.

Similarly, we can group, `aat_*.rs` and `ot_*.rs` files into `aat` and `ot`
modules, respectively.

## 6. Reshape the API

The current [RustyBuzz API](https://docs.rs/rustybuzz/latest/rustybuzz/) makes
it exceptionally difficult to cache data that is relatively expensive to
compute. Specifically, the [`Face`](https://docs.rs/rustybuzz/latest/rustybuzz/struct.Face.html)
type combines computed, heap allocated data structures with font data that
is bound to a lifetime. This makes it impossible to store these objects in a
cache without either use of `unsafe` or propagating the font data lifetime up to
the application entry point.

I propose splitting the cached data into a separate type (`ShapingFont` or
`ShapingData`, bikesheding to commence later) in a similar manner to what we've
done for the [`HintingInstance`](https://docs.rs/skrifa/latest/skrifa/outline/struct.HintingInstance.html)
in `skrifa`. This would require the user to maintain the association between a
`FontRef` and cached `ShapingFont` data but that requirement has not been an
overbearing burden for `skrifa` users.

RustyBuzz already has a separate type for [`ShapePlan`](https://docs.rs/rustybuzz/latest/rustybuzz/struct.ShapePlan.html)
which exposes cached state for a particular script/language/feature
configuration, so this separationg is not unprecedented and already represents
a break with the HarfBuzz API.

## 7. Performance tuning

According to the [README](https://github.com/harfbuzz/rustybuzz?tab=readme-ov-file#performance),
RustyBuzz is currently 1.5-2x slower than HarfBuzz. In order to offer a
production quality alternative, we must bridge that gap.

Profiling is key but there are a few known places where effort can be focused:

* More usage of the set digest bloom filter
  * This can be maintained for an entire buffer to potentially reject full
    lookups
  * Compute digest for mark coverage sets in `GDEF`
* Avoid redundant parsing
  * We currently reparse subtables for every glyph while processing a lookup
  * There's already a cache to avoid digging through the lookup structure
    but we should also build a local cache for fully parsed subtables
* Consider precomputing the "ignored" state of a glyph according to lookup
  properties
  * This was a big win for swash but Behdad [determined](https://github.com/harfbuzz/harfbuzz/issues/3221)
    it wasn't a huge deal for HarfBuzz. Rust overhead (bounds checking) is
    probably a factor here

There will undoubtedly be other opportunities as we begin measurement.

## 8. Testing

A substantial chunk of the HarfBuzz test suite has already been [ported](https://github.com/harfbuzz/harfruzz/tree/main/tests).
This gives us some strong assurance that the shaper is correct, at least
in areas of the code that are known to be tricky.

## 9. Future improvements


