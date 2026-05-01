# Proposal: A New Unified Font API for `read-fonts`

## 1. Introduction
This document proposes a new high-level `Font` API for the `read-fonts` crate. While `read-fonts` remains a `no_std` compatible library, this new API introduces an owned-data model to bridge the gap between low-level SFNT parsing and the high-performance requirements of modern shaping and platform-native font handles.

## 2. Current State: The `FontRef` Model
The current architecture is defined by its efficiency and low overhead:
* **Zero-Allocation Entry Point:** `FontRef` provides a view into font data without requiring a heap. This is ideal for lightweight metadata queries or environments where an allocator is unavailable.
* **Table-Centric Design:** Access is managed through the `TableProvider` trait, which provides typed views into raw tables. 
* **Stateless Lookups:** `FontRef` does not cache table offsets. Each table request performs a **binary search** on the `TableDirectory`.
* **The "Loose Association" Problem:** External caches are currently stored separately from font data and reassociated at use-time. This lack of a hard link prevents the safe use of `unsafe` optimizations, as we cannot guarantee the bytes haven't changed between validation and use.

## 3. The Catalysts for a New API

### A. The "Sanitize Once, Read Unsafely" Model
To optimize hot paths in shaping and rendering, we are moving toward a model where we validate (sanitize) table data once and then use `unsafe` to bypass bounds checks.
* **Safety Mandate:** To avoid Undefined Behavior, the validation state must be strictly bound to immutable data. By relaxing the "no-allocation" restriction for this specific API, we can own the data (e.g., via `Arc<[u8]>`), ensuring that the sanitized state and the underlying bytes are inseparable.

### B. Discontiguous Data (Platform Handles)
Platform APIs like CoreText often provide font data as a collection of individual table buffers.
* **The Limitation:** `FontRef` requires a single contiguous "blob." The new API will support "discontiguous" storage, allowing native platform handles to be treated as a single logical font without the overhead of re-stitching buffers.

## 4. Proposed Design: The Owned, Lazy Font API

### 1. Persistence of `FontRef`
`FontRef` will remain a core part of the crate. It is still the preferred tool for simple, low-frequency direct table access where the overhead of a binary search is negligible and zero-allocation is a priority.

### 2. High-Level Format Abstraction
Unlike `FontRef`, which is strictly an OpenType parser, the new API provides a unified interface for **Legacy Formats** (Type 1 and pure CFF).
* **Unified API:** Instead of virtualizing these formats into "fake" tables, the API provides a high-level surface for metrics, names, charmaps, and scaling that works consistently across all supported formats.

### 3. Lazy, Stateful Performance
The new `Font` type employs **Lazy Sanitization**:
* **On-Demand:** Tables are only sanitized upon their first request.
* **Memoized:** Once a table is validated, its pointer/offset and "verified" status are cached within the `Font` object. Subsequent accesses are $O(1)$ and utilize optimized `unsafe` reads.

### 4. The `FontInstance` Context
The `FontInstance` acts as the shared operational handle for a font. It encapsulates all parameters necessary for shaping and scaling:
* **Variation Settings:** Pre-calculated normalized coordinates for variable fonts.
* **Size:** The target PPEM for scaling operations.
* **Hinting Configuration:** User preferences and state for grid-fitting.
* **Caching:** Preserving variation region scalars can significantly improve performance of rendering "monster" variable fonts such as Roboto Flex.

## 5. Ecosystem Impact: Unified Identity
This architecture positions the new `Font` and `FontInstance` types as the shared "standard currency" across the `fontations` ecosystem.

* **Convergence:** Skrifa and HarfRust will utilize these shared types, eliminating redundant, crate-specific wrappers.
* **Single Configuration:** A `FontInstance` is configured once and then passed directly to both the shaper (for layout) and the scaler (for outlines), ensuring perfect synchronization of metrics and features.
* **Safety by Construction:** By moving the safety boundary from the call site to the object construction, the entire pipeline becomes memory-safe even while utilizing extreme performance optimizations.

## 6. Caveats
There are a few things worth considering about the new API:
* **Enumerated Table Set**: To ensure safety and enable multi-threaded usage, we must represent metadata as a discrete field for each table. This limits us to a set of "known" tables and bloats the amount of the code required. The `TableDirectory` type is still available as an escape hatch for the former and the latter is handled by generating most of the per-table code with a Python script.
* **Allocation**: We're forced to allocate to ensure that the font data and validation metadata are tightly bound. In practice, every sophisticated client of both Skrifa and HarfRust allocates resources when instantiating a font so this change has no practical impact, aside from offending the author's sensibilities.
* **Cross-crate Interop**: To achieve the "Unified Identity" mentioned above, both Skrifa and HarfRust need to attach data to the types defined in `read-fonts`. This means that we need some mechanism to enable this in the public API. The prototype provides some functions that are hidden in an `interop` module which is hidden from both documentation generation and rust-analyzer.
* **Once Dependency**: This all relies heavily on lazy, threadsafe initialization and thus requires a new `read-fonts` dependency on the `once_cell` crate for `no_std` compilations. This is already available in Chrome third_party.

## 7. Prior Art
Some will find this design remakably similar to the HarfBuzz representation of a font, mainly `hb_blob_t`, `hb_font_t` and `hb_face_t`. This isn't accidental: that API is both proven _and_ is the form we intend to match for HarfRust integration with Chrome.
As a side benefit, this also more closely matches Skia's `SkTypeface` and `SkScalerContext` architecture.

## 8. Prototype
A draft PR implementing a minimal version of this proposal is available at [googlefonts/fontations#1852](https://github.com/googlefonts/fontations/pull/1852).

## 9. Conclusion
The introduction of a stateful, owned `Font` API complements the existing `FontRef` by providing a path for high-performance, platform-integrated typography. It shifts the burden of validation and lifetime management from the user to the library, creating a more robust and cohesive Rust typography stack.