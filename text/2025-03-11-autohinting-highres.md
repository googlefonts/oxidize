 # The value of autohinting on high resolution displays

 See [autohinting](2025-01-27-autohinting.md) for an explanation of hinting and
 autohinting. In short, hinting is used to modify outlines in order to increase
 contrast and readability when those outlines are converted (rasterized) to
 images for display. This document provides a discussion on the relationship
 between autohinting and pixel density.

 ## Why disable autohinting at all?

As might be ascertained from the autohinting document linked above, the cost
of generating and applying hints at runtime can be significant in both time
and space. For complex variable fonts like [Roboto Flex](https://fonts.google.com/specimen/Roboto+Flex),
processing variations takes substantial time which hides some of the autohinting
cost and the overhead is roughly 100%. For the simpler, non-variable
[Roboto](https://fonts.google.com/specimen/Roboto), the time to render an
autohinted glyph can be as much as **10 times** that of an unhinted glyph.

Given this cost, it's worth considering whether there's some pixel density at
which autohinting can be disabled without sacrificing quality.

## Pixel density and text rendering



## A visual exploration

We'll take a look at the lowercase _e_ from the [Roboto](https://fonts.google.com/specimen/Roboto)
font at a reasonable body text size of 14 for this example. The following table
shows renderings of this character, unhinted and hinted, for various pixel
densities.

> These pixel density qualifiers are taken from _Table 1_ at the 
 [Android pixel density documentation](https://developer.android.com/training/multiscreen/screendensities#TaskProvideAltBmp).

|  | 160 PPI (baseline) | 240 PPI (high) | 320 PPI (extra high) | 480 PPI (extra-extra high) |
|--|---------|---------|---------|---------|
| Unhinted | ![14px unhinted](images/e_14_unhinted.png) | ![21px unhinted](images/e_21_unhinted.png) | ![28px unhinted](images/e_28_unhinted.png) | ![42px unhinted](images/e_42_unhinted.png) |
| Hinted | ![14px hinted](images/e_14_hinted.png) | ![21px hinted](images/e_21_hinted.png) | ![28px hinted](images/e_28_hinted.png) | ![42px hinted](images/e_42_hinted.png) |

It's quite obvious that the _baseline_ density (or anything less) really requires
hinting. The lower stem of the [counter](https://fonts.google.com/knowledge/glossary/counter)
in the unhinted rendering has very little contrast leading to a washed out
appearance and the character will appear blurry at actual size. Conversely,
the _extra-extra high_ density receives no benefit from hinting. Arguably,
the unhinted version better represents the original intent of the font designer.

The _high_ and _extra high_ densities are not as clear cut. The unhinted
renderings both have sufficient contrast. Whether hinting is required at these
densities is perhaps a matter of taste.
