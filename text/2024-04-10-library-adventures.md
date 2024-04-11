# Adventures in automated library building

_transcribed from a doc by @simoncozens_

In order to generate new small caps families (and test the gftools-builder’s small cap builder capability), I identified all Latin families in the library with a smcp feature. There were 101 families, forming a sample of around 6% of the GF library. From this sample:

* **82** had known upstream repositories; 29 did not
* **27** of that 82 (33%) could be built successfully using gftools-builder

Of the those which could not be built automatically:

* 13 had multiple families in the same repo, with sources in separate subdirectories, so the build process could not be automatically discovered.
* 5 others **could have been** built successfully if the source location was made explicit (e.g. they had sources in a directory called “source” or “Sources” and we were looking in “sources”)
* 11 had source formats we do support, yet **could not** be built because they had a weird or unclear build process. (SIL fonts, Adobe Source Sans/Serif, etc.)
* 4 had source formats that we don’t support building from (SFD/VFB/VFJ).
* 4 were just binaries or TTXes.
* 12 **could not be built** because of glyphsLib bugs or misfeatures: 
   * 1 could not be built successfully because it was Glyphs 2 sources, and [glyphsLib gets their axis count wrong](https://github.com/googlefonts/glyphsLib/issues/990); 
   * 6 could not be built because [glyphsLib changed the way it reads the Axis Mappings parameter and now everything is upside down](https://github.com/googlefonts/glyphsLib/issues/993) (once of which further could not be built because it uses Glyphs 3 variable feature syntax, which we don’t (and won’t) support); 
   * 4 could not be built because [we now erase open corners, but we don’t do so in an interpolation-aware way](https://github.com/kosmynkab/Brygada-1918/issues/13). 
   * 1 could not be built because it has a weird smart components setup that we don’t support but Glyphs does (repeated masters). 
* 1 could not be built due to a bug in ufo2ft’s variable feature building approach.
* 4 could not be built because of problems in the upstream sources (incompatible masters, open contours, etc)
* 1 was broken and I made a PR to fix it and it works now. 🙂


In all cases where the fonts could be successfully built, small caps families could be automatically derived from sources with one minor problem - in one family, the resulting font’s filename was not what it should have been. (Family name in name table was fine though.)

One philosophical question which arises from the top 30 which did not build (unusual repo structure, unable to automatically find source files, etc.): If we want buildable upstreams for all GF library fonts, to what degree should we be massaging the upstream repository structure and build processes - forking our own versions if necessary to make things fit our conventions - to make this happen?
