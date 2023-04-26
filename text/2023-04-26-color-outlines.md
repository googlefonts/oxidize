# API for color outlines

In order to support [COLRv1] (and v0) fonts, the
[skrifa](https://github.com/googlefonts/fontations/tree/main/skrifa)
crate requires an API to expose a more sophisticated set of operations than
are handled by the current [`Pen`](https://docs.rs/font-types/0.1.4/font_types/trait.Pen.html)
trait that is used to generate basic outlines.

## Proposal

The color [specification][COLRv1] defines a directed acyclic graph of "paint"
nodes with each describing an operation such as transform, clip or fill. An
implementation is expected to traverse the graph in depth first order applying
those operations in a sequence. While this is a reasonable representation for
storage and analysis, exposing this data structure directly through a trait
based callback interface is cumbersome.

The proposed API would flatten the graph structure into a linear sequence of
drawing commands, making use of stack semantics to capture the graph structure
where necessary. The set of commands required to represent the graph is fairly
small and maps nicely to most graphics APIs which already represent layering
with a state stack.

## Interface

This is a rough outline of the proposed interface which would introduce a new
`ColorPen` trait for accessing color outlines.

### Supporting types

Basic types that are used by the new `ColorPen` trait.

```rust
/// Extents of a rectangular region.
pub struct BoundingBox;

/// Affine transformation matrix.
pub struct Transform;

/// 32-bit RGBA unpremultiplied color.
pub struct Color;

/// Enumeration containing data for solid or gradient fills.
pub enum Brush<'a>;

/// Contains a glyph path for clips or fills. Offers a method that can feed
/// path commands to a Pen.
pub struct Path<'a>;
```

### Scaler methods

The `Scaler` type will gain two new methods:

```rust
fn has_color_outlines(&self) -> bool
```
Simply returns true if a `COLR` table is present.

```rust
fn color_outline(
    &self, 
    glyph_id: GlyphId, 
    palette_fn: impl Fn(u16) -> Option<Color>, 
    pen: &mut impl ColorPen
) -> Result<(), Error>
```
Loads a color outline for the specified glyph and invokes the appropriate
callbacks on the pen.

The `palette_fn` parameter is used to specify the mapping from palette
color index to a color value. This can be used to implement overrides and
a helper will also be provided for constructing this function for basic
palette support.

### The `ColorPen` trait

This trait contains methods that are implemented by the caller to access
the sequence of operations necessary to render a color outline.

> Note: When provided below, the code in block quotes demonstrates how these
operations might roughly be implemented for a Skia based renderer (assuming
a local variable `canvas` of type `SkCanvas`)

#### Methods

```rust
fn bounds(&mut self, bounds: BoundingBox)
```
This is guaranteed to be the first method called on the pen to allow an
appropriately sized surface to be allocated if necessary. It will pass the
value of the `ClipBox` if available, otherwise a bounding box computed
during evaluation of the paint graph.

In accordance with the spec, if the paint graph is determined to be
unbounded, no methods on the pen will be called and the `color_outline`
method will return an error.

```rust
fn push_transform(&mut self, transform: Transform)
```
Push a transform onto the state stack.

> `canvas->save();`<br>
`canvas->concat(TransformToSkMatrix(transform));`

```rust
fn pop_transform(&mut self)
```
Pop a transform from the state stack.

> `canvas->restore();`<br>

```rust
fn push_clip(&mut self, glyph_id: Glyph_id, path: &Path)
```
Push a clip path onto the state stack.

> `SkPath clipPath = PathToSkPath(path);`<br>
`canvas->save();`<br>
`canvas->clip(clipPath, true);`

```rust
fn pop_clip(&mut self)
```
Pop a clip path from the state stack.

> `canvas->restore();`<br>

```rust
fn push_layer(&mut self, mode: Option<CompositeMode>)
```
Push a new layer to the state stack, optionally with a composite mode.

If `mode` is not `None`:
> `SkPaint blendModePaint;`<br>
`blendModePaint.setBlendMode(CompositeModeToSkBlendMode(mode.unwrap());`<br>
`canvas->saveLayer(nullptr, &blendModePaint);`

Otherwise:
> `canvas->saveLayer(nullptr, nullptr);`<br>

```rust
fn pop_layer(&mut self)
```
Pop a layer from the state stack.

> `canvas->restore();`<br>

```rust
fn fill(&mut self, brush: &Brush);
```
Fill the current clip with the given brush.

> `canvas->drawPaint(BrushToSkPaint(brush));`<br>

```rust
fn fill_path(
    &mut self,
    glyph_id: GlyphId,
    path: &Path,
    brush: &Brush,
    brush_transform: Option<Transform>
)
```
Fill the specified path with the given brush. Note that this is an optimization
for the pattern `Clip Transform* Fill` in the graph. The intermediate
transforms are captured in the optional `brush_transform` parameter.

Assuming `brush_transform` is `None`:
> `SkPath fillPath = PathToSkPath(path);`<br>
`SkPaint fillPaint = BrushToSkPaint(brush);`<br>
`canvas->drawPath(fillPath, fillPaint);`<br>

Otherwise, use the standard clip, transform, fill code.

## Questions

### Should transforms be absolute?

The transform stack is generally unnecessary after flattening the graph. We can
remove the `push_transform`/`pop_transform` methods and provide absolute 
transforms to the methods where they are required.

### How much processing on gradient fills?

Skia does a significant amount of
[processing](https://skia.googlesource.com/skia/+/265a074c910a0a4ac29176bca0a34615247f10f7/src/ports/SkFontHost_FreeType_common.cpp#597)
on gradient fills to match the expectations of the relevant shaders. The
applied transformations appear to produce equivalent brushes with color stops
and other parameters adjusted to account for various constraints. If these
adjustments are applicable to (or at least do not hurt) other renderers, then
it makes sense to apply them in our code as well.

If this is done, the `BrushToSkPaint` function in the examples above should
be mostly trivial.

[COLRv1]: https://github.com/googlefonts/colr-gradients-spec
