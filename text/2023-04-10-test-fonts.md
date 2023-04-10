Initial list from from https://github.com/google/fonts/issues/4772, captured here so I can find it again.

**Families which take a long time to compile:**
- Roboto Serif, https://github.com/googlefonts/roboto-serif (~1hr with fontmake)
- Material Symbols, https://github.com/google/material-design-icons (~2-3hr with custom chain) and its private 1P version
- Google Sans (~15mins with fontmake)

**Families with multiple axes, manual hinting, UFO sources:**
- Roboto Flex, https://github.com/googlefonts/roboto-flex (about 12 axes in total)
- Amstelvar, https://github.com/googlefonts/amstelvar

**Colr v1 families:**
- Foldit, https://github.com/SophiaDesign/Foldit

**Complex GSUB/GPOS:**
- Gulzar, https://github.com/googlefonts/Gulzar
- Tiro Indigo project, https://github.com/TiroTypeworks/Indigo (eight families for complex scripts)
- STIX Math, https://github.com/stipub/stixfonts (fonts for Scientific, Technical, and Mathematical texts)

**Low Hanging fruit (static, no vf, no color):**
- Send Flowers, https://github.com/googlefonts/send-flowers (single font family)
- Castoro, [static font](https://github.com/TiroTypeworks/Castoro) (single font family but custom build)

**Normal VF families building from UFO/designspace file:**
- [Kumbh Sans](https://github.com/xconsau/KumbhSans) 2 axes, wght, YOPQ

**Normal VF families building from .glyphs file:**
- [Oswald](https://github.com/googlefonts/OswaldFont) 1 axis, wght
- [Dynapuff](https://github.com/googlefonts/dynapuff) 2 axes, weight, width
- [Texturina](https://github.com/Omnibus-Type/Texturina) (Roman/Italic family with 2 axes)
- [Fragment Mono](https://github.com/weiweihuanghuang/fragment-mono) (no kerning, some ligatures)
- [Readex Pro](https://github.com/ThomasJockin/readexpro) (simple arabic font)

**Several sub-families building from same source:**
- [Lexend](https://github.com/googlefonts/lexend)

**Noto:**

All sorts of fun stuff, probably most easily found via http://notofonts.github.io/noto.json
