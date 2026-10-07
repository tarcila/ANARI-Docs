# Spatial-field extensions

**Scope:** Specification wording and supporting review notes. These changes affect documentation only, not SDK or backend support.

## Organization and review series

Each extension is introduced in its own commit, starting with data centering. Shared research is separate: [VisRTX review](../research/visrtx-extension-ratification-review.md) and [anari-ospray investigation](../research/anari-ospray-spatial-fields.md).

Only actual field subtypes are sections in the spatial-field documentation tree. Parameters appear in each affected subtype's table with an extension link. Shared rules are inline below the first relevant table; subsequent subtypes reference those rules and specify their differences. All content is present in ordinary builds, without opt-in attributes. No C ABI declarations or registry tokens are added.

## Data centering

[Common rules and regular-field behavior](../../chapters/object_types/spatial_fields.txt#extension_spatial_field_data_centering) are inline below the regular-field table. The extension defines sample association, but placement and defaults are subtype-specific. Other subtype definitions may explicitly adopt the parameter. An absent, unknown, or unset value selects the subtype's documented default; it does not universally select `node`.

For `structuredRegular`, the default stays `node`. Cell samples are at `origin + (i + 0.5) * spacing`, with extent `origin + n * spacing`; one sample per axis suffices. Regular-field cubic filtering remains optional, using the subtype's kernel and extended boundary samples rather than a clamped continuous coordinate.

The inspected anari-ospray sources contained no standalone centering extension. AMR work is not evidence of an independent implementation of this contract. Neither ordinary NanoVDB nor unstructured sample association is changed by this extension.

Backend validation cases:

- Regular origin 10, spacing 2, three samples: node positions 10/12/14 with extent `[10,14]`; cell positions 11/13/15 with extent `[10,16]`.
- Cell values `[0,2,4]` on that grid: linear values at X=10/12/16 are 0/1/4, including the outer half cells.
- One cell sample per axis produces a nonempty constant field; the same dimensions are invalid in node mode.
- Check unknown-value fallback, unset/recommit, boundary filtering, volume and isosurface use, and introspection consistency.

## Structured rectilinear fields

[Specification source](../../chapters/extensions/spatial_field_structured_rectilinear.txt)

The base subtype is node-centered, with nearest/linear reconstruction and the scalar types already used by regular fields. It adopts the centering extension with default `node`: `n` node samples need `n` axis coordinates, while `n` cell samples need `n + 1` boundary coordinates. Coordinates must be finite and strictly increasing.

Placement and interpolation are subtype-specific. Positions first map through the nonuniform axis intervals to logical coordinates `u`, then to sample coordinates `q = u` for nodes or `q = u - 0.5` for cells. The regular-field `(x-origin)/spacing` formula and cubic kernel do not apply here.

For cell boundaries `[0,1,4]` and values `[0,1]`, the specified reconstruction gives 0.5 at X=1; interpolation between physical centers gives 0.25. The extension uses the logical-grid interpretation without prescribing VisRTX's lookup-table resolution. Agree numerical tolerances before treating a backend as conformant.

Additional validation cases:

- Node coordinates and values `[0,1,4]`: linear reconstruction at X=2 gives 2.
- Cell boundaries `[0,1,4]`, values `[0,1]`: linear gives 0.5 at X=1; nearest selects the higher sample there.
- Validate coordinate types/counts, finite strict ordering, one-cell dimensions, defaults, updates, and both volume/isosurface use.

## NanoVDB rectilinear fields

[Specification source](../../chapters/extensions/spatial_field_nanovdb_rectilinear.txt)

This adds a separate `nanovdbRectilinear` subtype and `KHR_SPATIAL_FIELD_NANOVDB_RECTILINEAR` extension. It extends the scope beyond the initially deferred item in the historical VisRTX review. Ordinary `nanovdb` behavior is unchanged, and supporting the new subtype does not require exposing the dense rectilinear subtype.

The parameter names remain `coordsX`, `coordsY`, and `coordsZ`. Defaults differ deliberately: dense rectilinear fields are node-centered, while NanoVDB rectilinear fields are cell-centered. The latter adopts the common centering extension to additionally allow `node`; absent/unknown/unset values select its own `cell` default.

The extension defines the mapping as follows:

- Explicit coordinates are the sole object-space placement/domain authority. They replace, rather than compose with, the embedded index-to-world map and `worldBBox` for this subtype.
- The inclusive active index bbox `[b,e]` defines sample counts `n=e-b+1` and index origins. Nonempty cell-centered axes require `n+1` boundaries; node-centered axes require `n` sample coordinates.
- Reuse the dense rectilinear logical mapping, clamp the local sample coordinate to `[0,n-1]`, then add `b` for tree lookup. A nonzero or negative index origin is not discarded.
- Logical lookups retain inactive stored values, constant tiles, and background values. Inactivity is not a zero/transparent mask inside the domain.
- A well-formed empty index bbox gives an empty domain. Required coordinate arrays remain typed and present, but their values/lengths are ignored. Malformed grids and invalid nonempty-grid coordinate inputs instead follow invalid-field diagnostics.

**The specified contract is not a claim of VisRTX conformance.** Its [schema](../../../VisRTX/devices/rtx/device/visrtx_spatial_field_nanovdb_rectilinear.json) describes object-space axes and a cell default, but its [field implementation](../../../VisRTX/devices/rtx/device/spatial_field/NvdbRectilinearField.cpp#L131) derives bounds from `worldBBox`, and its [sampler](../../../VisRTX/devices/rtx/device/spatial_field/NvdbRectilinearSamplerInline.h#L127) composes the embedded map with axis lookup tables. The new authority rule and coordinate-count validation need implementation work and independent review. NanoVDB serialized-format/grid-type support must also be reconciled with the existing NanoVDB contract rather than inferred from vendor shader paths.

Numeric and edge-case validation cases for this subtype:

| Input | Expected behavior |
|---|---|
| X index range `[-3,-2]`, cell boundaries `[0,1,4]`, values `[0,1]` | At X=1, local coordinate is 0.5 and continuous tree coordinate is -2.5: linear gives 0.5; nearest chooses index -2 and gives 1. |
| X index range `[7,9]`, node coordinates `[0,1,4]`, values `[0,1,4]` | At X=2, interpolate indices 8 and 9 to obtain 2; do not look up indices relative to zero. |
| Embedded map/world bbox far from axis endpoints | Rendering, isosurfaces, and reported containing-object bounds use the axis-defined domain; the header world bbox cannot cull it away. |
| Inactive voxel with nonzero stored value, a constant tile, and absent topology with nonzero background inside the index bbox | Use each tree lookup value, not zero; transfer functions may make it visible. |
| One logical sample per axis | Valid constant cell-centered field with two boundaries per axis; invalid node-centered dimensions. |
| Empty index bbox | Empty domain, no contribution; typed coordinate arrays may be empty. |
| Node/unknown/unset centering changes | Node uses `n` coordinates; unknown/unset restores cell behavior and requires `n+1`, with no automatic array resizing. |
| Activity changes that alter `indexBBox` | Revalidate coordinate lengths against the new counts and use the new lower index; do not silently retain the old offset. |

Samples in these examples are constant along the other two axes, with valid coordinate counts there. The extension defines the transform override and empty-grid behavior; these are not inherited implementation guarantees. Numerical tolerances remain an interoperability review topic.

## Build and validation

From this workspace:

```sh
make html SPECREMARK="spatial-field extension review"
make allchecks
```

Open `out/html/anspec.html`. No `EXTENSIONS` flags are needed. `SPECREMARK` avoids the Makefile's Git-based revision lookup in a Jujutsu workspace.

On this Nix host, the optimizer's `#!/bin/bash` interpreter path does not exist. Render with `make pdf GS_EXISTS= SPECREMARK="spatial-field extension review"`, then run `bash config/optimize-pdf out/pdf/anspec.pdf` to produce `out/pdf/anspec-optimized.pdf`.

Every commit prefix is checked with HTML generation, `make allchecks`, and heading/anchor/parameter-table checks. The complete series also receives a PDF build. These checks validate documents, not device conformance. Independent implementations, numerical tolerances, and application examples remain necessary review topics.
