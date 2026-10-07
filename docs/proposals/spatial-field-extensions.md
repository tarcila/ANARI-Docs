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

## Build and validation

From this workspace:

```sh
make html SPECREMARK="spatial-field extension review"
make allchecks
```

Open `out/html/anspec.html`. No `EXTENSIONS` flags are needed. `SPECREMARK` avoids the Makefile's Git-based revision lookup in a Jujutsu workspace.

On this Nix host, the optimizer's `#!/bin/bash` interpreter path does not exist. Render with `make pdf GS_EXISTS= SPECREMARK="spatial-field extension review"`, then run `bash config/optimize-pdf out/pdf/anspec.pdf` to produce `out/pdf/anspec-optimized.pdf`.

Every commit prefix is checked with HTML generation, `make allchecks`, and heading/anchor/parameter-table checks. The complete series also receives a PDF build. These checks validate documents, not device conformance. Independent implementations, numerical tolerances, and application examples remain necessary review topics.
