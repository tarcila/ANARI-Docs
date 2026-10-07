# VisRTX RTX extensions: ANARI standardization review

## Recommendation

Take **structured rectilinear fields, explicit field centering, and field ROI clipping** to the ANARI Working Group first. Prepare **ray-buffer cameras and geometry back-face culling** as a second small group after resolving the contract issues below. Develop **frame accumulation telemetry** and a **MaterialX interoperability profile** separately. Do not propose the complete VisRTX renderer interfaces, CUDA plumbing, or experimental SDF/neural interfaces for unchanged promotion.

These are recommendations for **optional standardized extensions**, not requirements that every ANARI device implement them, and not claims that any proposal is ready for a ratification vote. New `KHR_*` names below are illustrative working names, not registered names. The specification distinguishes vendor experimentation from ratified `KHR` extensions; its style guide distinguishes multi-author, unratified `EXT` extensions as well. [S1, S2]

## Scope and evidence

Reviewed the local RTX device schemas, README, selected CPU/GPU implementation paths, and relevant test sources against this ANARI-Docs workspace:

- ANARI-Docs: clean working copy `b328e262`, based on `7481cd02`; specification source identifies version 1.2 in `Makefile:106–107`.
- VisRTX: clean working copy `5c782f19`, based on `7ec68acf`.
- Inventory: **17 local extension JSON files** (15 `VISRTX_*`, two `NV_*`), plus **four imported `EXT_SAMPLER_COMPRESSED_*` dependencies** and additional frame/device/renderer parameters. MDL, MaterialX, and neural extension advertisement is build-conditional. [R1]
- This is a source-based assessment, **not runtime validation or a current Khronos approval-status audit**. No GPU tests or documentation builds were run. Recommendations about independent implementations and tests are proposed readiness criteria, not asserted Khronos procedural requirements.

Citation shorthand: **R** paths are under `../VisRTX/devices/rtx/`; **S** paths are relative to this documentation workspace. Source references at the end are clickable. Line numbers refer to the inspected snapshots.

## Complete disposition

| Existing surface | Disposition | Reason / necessary change |
|---|---|---|
| `VISRTX_SPATIAL_FIELD_STRUCTURED_RECTILINEAR` | **First submission** | Fills a real representation gap; specify coordinate counts, centering, reconstruction, and bounds. |
| `VISRTX_SPATIAL_FIELD_DATA_CENTERING` | **First submission** | Portable scientific-data semantics; preserve existing node-centered defaults and specify cell reconstruction. |
| `VISRTX_SPATIAL_FIELD_REGION_OF_INTEREST` | **First submission** | Small, useful clipping contract shared by volume rendering and isosurfaces. |
| `VISRTX_CAMERA_RAY_BUFFER` | **Next submission** | Useful application-supplied camera rays without a GPU-specific API; resolve depth and camera-extension interactions. |
| `VISRTX_TRIANGLE_BACK_FACE_CULLING` | **Next submission, after repair** | Portable visibility control, but currently inconsistent across renderers. |
| Frame `numSamples`, `nextFrameReset` | **Design a narrow companion to accumulation** | Useful telemetry; exact sampling, synchronization, and reset-prediction semantics need work. |
| `VISRTX_MATERIAL_MATERIALX` | **Strategic, profile-first proposal** | Portable document interchange potential; do not standardize the MDL translation backend. |
| `VISRTX_SPATIAL_FIELD_NANOVDB_RECTILINEAR` | **Defer; reconsider composition** | External axes and NanoVDB metadata create competing coordinate/bounds authorities. |
| `VISRTX_GEOMETRY_SDF` | **Redesign before submission** | Analytic implicit geometry is useful; current binary ABI and tracing/noise controls are not a clean interchange contract. |
| `VISRTX_GEOMETRY_NEURAL` | **Keep experimental** | Schema does not describe the implemented interface; implementation readiness problems remain. |
| `VISRTX_MATERIAL_MDL` | **Keep language-specific for now** | Could support a future optional MDL interoperability extension, but needs a language/profile/resource contract and implementer interest. |
| `NV_ARRAY_CUDA`, `NV_FRAME_BUFFERS_CUDA` | **Keep CUDA-specific** | Useful interoperability, not backend-neutral memory semantics. |
| `VISRTX_RENDERER_DEFAULT`, `FAST`, `INTERACTIVE`, `QUALITY`, `DEBUG` | **Do not promote as bundles** | `default` already exists in ANARI; algorithms and most tuning knobs are intentionally implementation-specific. Extract only independently useful feature contracts. |
| `EXT_SAMPLER_COMPRESSED_IMAGE2D`, `EXT_SAMPLER_COMPRESSED_FORMAT_BC123`, `BC45`, `BC67` | **Separate existing-EXT promotion track** | Already referenced as imported EXT capabilities, not newly invented VisRTX interfaces. Missing from this local spec's extension index; reconcile their external specification/status before proposing promotion. |
| Renderer `cutPlane` | **Separate clipping design** | Potentially useful, but not equivalent to object-space field ROI; define ray classes, coordinate space, scope, and capping. |
| Other renderer/device controls | **Mostly retain implementation-specific** | See below; avoid mandating OptiX algorithms, packaging, or device selection. |

## 1. Structured rectilinear spatial fields

**Suggested working name:** `KHR_SPATIAL_FIELD_STRUCTURED_RECTILINEAR`.

ANARI currently describes regular, NanoVDB, and unstructured fields, but no structured rectilinear subtype. Encoding a rectilinear grid as unstructured cells is possible but needlessly expands topology and coordinates. A subtype using `data: ARRAY3D` plus `coordsX/Y/Z: ARRAY1D<FLOAT32>` expresses the data directly. VisRTX has both a schema and a concrete sampler. [S3, R2]

**Proposed minimum contract:**

- Required scalar `data` and three finite, strictly increasing coordinate arrays in object space.
- For data dimensions `(nx, ny, nz)`, node-centered coordinate lengths are `(nx, ny, nz)`; cell-centered lengths are `(nx+1, ny+1, nz+1)` and describe cell boundaries. These relationships are already assumed in `StructuredRectilinearSamplerInline.h:52–59`.
- Bounds come from the first and last coordinates. State minimum dimension sizes and invalid-input behavior explicitly.
- Start with `nearest` and `linear`; do not inherit cubic support implicitly from another subtype.
- Specify reconstruction mathematically, especially whether cell-centered linear interpolation occurs in warped index space or between physical cell centers. Those interpretations differ on nonuniform grids.
- Keep ROI orthogonal; coordinate its applicability with the centering and ROI proposals rather than making either an accidental requirement of every rectilinear implementation.

**Readiness issues:** The JSON does not state coordinate-length relationships or monotonicity. The field's axis-building loop checks element type and LUT construction but does not compare axis counts with data dimensions (`StructuredRectilinearField.cpp:180–214`). `RectilinearLUT.h:90` fixes the lookup resolution at 8192; that is an implementation choice, not the interpolation definition. Cross-device tests should include strongly nonuniform spacing and analytic ramps to expose approximation error. Align the supported scalar-type list deliberately with `structuredRegular` rather than assuming the two lists already match. [R2, S3]

## 2. Explicit node/cell data centering

**Suggested working name:** `KHR_SPATIAL_FIELD_DATA_CENTERING`.

The existing `structuredRegular` specification explicitly requires vertex-centered values and bounds `origin + (dims - 1) * spacing`. VisRTX adds `dataCentering = "node" | "cell"`, changes regular-grid extent to `dims * spacing` for cell data, and changes texture-coordinate offset so the samples lie at cell centers. This is a small, renderer-independent addition. Unstructured fields already distinguish `vertex.data` from `cell.data`, so the broader concept is not new to ANARI. [S3, R3]

**Proposed contract:**

- Preserve `"node"` when the parameter is absent on existing structured regular fields.
- For positive spacing, node sample `i` lies at `origin + i * spacing`; cell sample `i` lies at `origin + (i + 0.5) * spacing`, with outer extent `origin + dims * spacing`.
- Define filtering and boundary reconstruction; “cell-centered” alone does not mean piecewise constant when `filter="linear"`.
- Specify allowed dimensions, spacing, invalid strings, and the interaction with isosurfaces and bounds.
- Coordinate rectilinear support with proposal 1. Do not reinterpret the already separate `vertex.data`/`cell.data` interface on unstructured fields.

**NanoVDB caveat:** VisRTX defaults NanoVDB to `"cell"` and regular fields to `"node"`. Do not impose one global default by renaming the extension. The NanoVDB contract already derives extent from its serialized grid header; its centering/mapping rules need a separate compatibility decision. A first revision limited to dense structured fields is a reasonable way to avoid changing existing NanoVDB interpretation. [R3, S3]

## 3. Object-space field ROI clipping

**Suggested working name:** `KHR_SPATIAL_FIELD_REGION_OF_INTEREST`.

Retain the small interface: optional `roi: FLOAT32_BOX3`, in the field's object coordinates; absence means no additional clipping. This is more portable than a renderer algorithm or CUDA interface. The implementation clips both volume ray intervals and isosurface marching intervals against the ROI. [R4]

**Decisions to settle before submission:**

1. Applicability to supported field subtypes, and how support is queried without promising unsupported types.
2. Empty, disjoint, degenerate, non-finite, and inverted boxes; recommend defining empty/disjoint intersections as no contribution.
3. Whether a clipped field boundary generates a cap. Recommend **no synthetic isosurface cap** for this minimal extension.
4. Whether reconstruction near the boundary may read underlying samples outside the ROI. Recommend clipping the domain, not rewriting the underlying scalar data to zero.
5. Whether bounds properties must be tight or may remain conservative. VisRTX's regular-field `bounds()` remains untrimmed; this is **not automatically a correctness bug**, since the traversal path clips later. The standard must make the observable expectation clear.
6. Interaction with instances: transform the field and its ROI together, not a world-space box shared accidentally by every instance.

The README still says “two optional parameters” although the schema defines one box parameter. Fix that documentation drift. The implementation's use of floating-point extremes internally for an unset ROI need not be standardized or replaced; the API contract can simply define parameter absence. [R4, R3]

## 4. Application-supplied ray-buffer camera

**Suggested working name:** `KHR_CAMERA_RAY_BUFFER`.

This is a strong candidate for unusual projections, calibrated cameras, and application-defined per-pixel rays. The interface is ordinary ANARI arrays, not CUDA pointers: optional `ray.org`/`ray.dir` vector arrays and `ray.tmin`/`ray.tmax` scalar arrays matching frame dimensions. VisRTX normalizes directions and has a focused test covering perspective parity, interval clipping, origin offsets, and mismatched dimensions. [R5]

**Required contract work:**

- Specify world-space origins/directions, frame indexing, defaults, normalization, and the units of `tmin/tmax` after normalization.
- Resolve **depth semantics**: ANARI describes distance to the camera; the VisRTX test expects distance from each supplied ray origin. State the subtype-specific rule explicitly. [S4, R5]
- Define zero/non-finite directions, invalid intervals, mismatched dimensions, and updates/recommits. VisRTX currently collapses reversed intervals with `fmaxf` and makes all rays miss for size mismatch; decide which behavior is normative rather than inheriting it accidentally.
- Explicitly exclude or define `imageRegion`, stereo, camera transforms/motion, shutter, and depth-of-field interactions. Existing camera text says all cameras accept common parameters; a new subtype needs explicit exceptions, not just a README note. [S5]
- Keep a first version to deterministic per-pixel primary rays. Renderer sampling can still refine lighting; it cannot invent a subpixel ray distribution absent from the inputs.

## 5. Geometry back-face culling

**Suggested working name:** `KHR_GEOMETRY_BACK_FACE_CULLING` — broader than the existing “TRIANGLE” name because the schema also covers quads.

Keep a per-geometry boolean `cullBackfaces`, default false. The current schema promises culling **in primary visibility**, not removing a surface from all light transport. Specify winding, mirrored instance transforms, and effects on depth/ID channels. Do not conflate camera-only culling with material sidedness or shadow/reflection visibility. [R6]

**Concrete implementation blocker:** `fast` calls `ray::cullbackFaces()` in its primary any-hit program, and `interactive` calls it in shading any-hit. `quality`'s shading any-hit only calls `cullCutPlane()`; it does not use the per-geometry culling helper. The separate renderer parameter `cullTriangleBackfaces` sets a global OptiX ray flag and does not repair the missing per-geometry contract. Resolve support reporting/behavior and add renderer-specific tests before citing VisRTX as a consistent reference implementation. [R6]

Prefer one geometry-level standard control. If global renderer culling is retained as a vendor option, define its precedence explicitly.

## 6. Frame telemetry, not a new accumulation mechanism

ANARI already has `KHR_FRAME_ACCUMULATION` and `refinementProgress`. VisRTX adds `numSamples: INT32` and `nextFrameReset: BOOL`; the latter can flush committed-object work during a property query. [S4, R7]

Propose a **separately discoverable companion** to accumulation instead of silently making new properties mandatory for every existing implementation advertising `KHR_FRAME_ACCUMULATION`.

- Prioritize a precisely defined sample-count property, with completed-frame versus in-flight semantics, treatment of adaptive/checkerboard sampling, accumulation-disabled behavior, reset behavior, and overflow. Minimum-per-pixel count is more defensible than an undefined “samples” count; VisRTX's README already describes a lower bound under checkerboarding.
- Discuss `sampleLimit` and `pixelSamples` only for a renderer capability that can honor such budgets; do not assume all ANARI algorithms are Monte Carlo integrators.
- Defer `nextFrameReset` until its synchronization and prediction guarantees are clear. It must not imply certainty about later commits. A completed-frame accumulation-generation/reset indicator is an alternative design, **not an existing VisRTX feature**.
- Do not standardize checkerboarding, the frame-counter implementation, or the side effect of flushing the entire commit buffer.

## 7. MaterialX: valuable, but standardize a profile

A material document interface is strategically useful beyond a fixed PBR parameter table. VisRTX provides `source`, `sourceType = documentFile | documentInline`, and `materialName`; it also implements `materialNames` and `textureInputs` properties and input overrides. The implementation derives from the MDL path, but that is **not** a necessary part of a MaterialX ANARI contract. [R8]

A credible optional MaterialX extension needs:

- A versioned MaterialX/node-definition profile and a declared set of supported surface models, rather than a promise to execute arbitrary documents.
- Material selection and deterministic ambiguity/error behavior.
- ANARI parameter/sampler/attribute mappings, texture coordinates, normal conventions, units, and color-space rules.
- Resource-resolution rules for file and inline documents, including relative assets; lifetime/recommit behavior; missing-resource and unsupported-node diagnostics.
- Discovery semantics and lifetimes for returned string lists. Both implemented properties need an explicit contract; merely mentioning `materialNames` in a parameter description is insufficient.
- Tests on another backend that does not reuse VisRTX's MDL transcoder.

Prefer a bounded surface profile first; keep device search paths, distribution discovery, generated MDL, and compiler-cache details implementation-specific. MDL itself could also be an optional language-interoperability extension if other implementers want it; being implemented through an NVIDIA SDK is not, by itself, proof that standardization is impossible. It is simply not the highest-priority general ANARI gap here.

## 8. Features to defer or extract carefully

### NanoVDB rectilinear mapping

Do not promote `nanovdbRectilinear` unchanged. Decide whether external axes override or compose with the embedded transform, how arrays relate to a possibly nonzero index bounding box, and how background/inactive values behave. In the current implementation, `m_bounds` comes from `worldBBox()` while coordinate LUTs are built separately from the supplied axes; axis lengths are not checked against grid dimensions in that loop. This needs explicit semantics and tests before serving as a reference. Consider a coordinate-mapping extension, but do not force a generalized mapping abstraction merely to avoid one subtype. [R9]

### SDF and neural geometry

SDF describes `primitive.sdf` as raw bytes containing 72-byte C++ structs. A binary representation is not inherently forbidden—NanoVDB already uses one—but a new format needs a stable layout, discriminants, alignment/endianness rules, versioning, and validation. Separate mathematical primitives/blending from sphere-tracing policy and camera-distance-dependent displacement. Evaluate an analytic spatial field consumed by existing `isosurface` geometry as an alternative. The schema says the default march count is 128; code uses 16. [R10, S6]

Neural geometry's schema exposes only `sourceType`, described as a layer count, whereas code reads `n_layers`, `layer_N`, `aabb_min/max`, and `threshold`. Also, `finalize()` uploads `m_aabb`, but inspection found no assignment from those bounds parameters; shader bounds are populated separately. Treat this as a source-level correctness concern needing a runtime regression test, not a verified prediction of a particular GPU failure. A future proposal needs a model format, inference/precision semantics, bounds, and an introspectable contract—not a promise to standardize OptiX cooperative-vector machinery. [R11]

### Renderer bundles and unadvertised extras

ANARI already deliberately leaves renderer subtypes/algorithms implementation-specific and requires a usable `default` renderer. Standardization should describe results or application intent, not mandate VisRTX's `fast`/`quality`/`interactive` scheduling. [S7]

- **Already covered:** boolean denoising, background color/image, ambient light, material emission, frame accumulation, and light primary visibility are present in the local spec. Improvements to emissive-light sampling, MIS, and visible-light proxies are implementation work, not automatically new API extensions. [S4, S7, S8]
- **Potential follow-on:** renderer clipping planes. VisRTX exposes one `cutPlane`; distinguish global clipping from field ROI and specify camera/secondary/shadow/volume behavior and cap semantics before expanding to multiple planes. [R12]
- **Lower priority:** `denoiseStart` could become a scheduling feature once sample-budget semantics exist. `denoiseMode` selects buffers passed to a particular denoising setup, so it is less compelling as a universal requirement. The JSON default is `colorAlbedoNormal`, while `Renderer.cpp:179` defaults to `color`; reconcile before reusing this contract. [R12]
- **Retain as policy:** firefly filtering, checkerboarding, AO heuristics, shadow sampling rates, ray-depth budgets, light sampling limits/falloff, debug methods, and similar implementation knobs unless a separate multi-implementation use case emerges.
- **Do not rubber-stamp:** `premultiplyBackground` needs to be assessed as part of a complete frame color/alpha compositing contract, not standardized as an isolated switch.
- **Device plumbing:** GPU ordinal selection, `forceInit`, MDL engine pointers, and MaterialX/MDL search paths are not priorities for portable scene-description semantics. [R13]

### CUDA and compressed textures

Keep `NV_ARRAY_CUDA` and `NV_FRAME_BUFFERS_CUDA` usable as explicit CUDA interoperability capabilities. A broader memory-interoperability effort must specify ownership, lifetime, device/context identity, synchronization, accessibility, and mapping—not just remove `NV` from a name. A CUDA-specific standardized extension is conceptually possible, but it would remain CUDA-specific. [R14]

The four compressed-texture EXT dependencies deserve a separate promotion assessment. Reuse their existing contracts and settle format/size/block layout, color-space, and support-query behavior with their authors. This review confirms VisRTX advertises and implements BC-format sampling, **not** the approval status or cross-vendor conformance of those external extensions. [R1, R15]

## Submission and validation plan

1. Draft three independent, coordinated proposals: **rectilinear fields**, **centering**, **ROI**. Show dependencies and subtype applicability without bundling unrelated capabilities.
2. Prepare **ray-buffer camera** and **back-face culling** proposals after the identified contract inconsistencies are resolved. Start **frame telemetry** and **MaterialX profile** discussions in parallel, not as blockers for the field proposals.
3. For each proposal, provide motivation and application examples, normative parameter/property tables, defaults, validity rules, extension interactions, error behavior, and exact capability-query semantics. Update relevant ANARI-Docs chapters and `appendices/extension_index.txt`; update SDK schemas/query support too. Update the XML registry if the proposal introduces API declarations or tokens; do not imply every string-valued object extension necessarily requires new C ABI.
4. Supply executable tests: regular/rectilinear analytic ramps, node/cell bounds and interpolation, invalid coordinate arrays, ROI on volumes and isosurfaces through instances, ray-origin depth and camera interactions, and winding/negative-scale culling across renderers. Extend accumulation tests with checkerboarding, reset, and `ANARI_WAIT` cases.
5. Seek an independent non-VisRTX implementation and application usage. Publish expected numeric tolerances rather than demanding identical stochastic images.
6. Before proposing ratification, reconcile schema/README/runtime behavior and test capability reporting under optional build flags. Follow repository contribution/CLA requirements and human review of AI-assisted text. [S9]

**Verification performed:** parsed all 17 local JSON files; inspected the declared dependency list and optional-generation logic; compared candidate surfaces with the local extension index and normative chapters; traced selected sampling/intersection/property implementations; inspected ray-buffer and MaterialX test availability. No dedicated field-centering/rectilinear/ROI/culling/SDF/neural references were found by targeted searches of `apps/tests/api/*.cpp`; this is not proof that no coverage exists elsewhere. Tests were not executed. No implementation or normative specification files were modified by this review.

## Sources

- **S1:** [ANARI extension model](../../chapters/api_design_choices.txt#L60), especially lines 76–93.
- **S2:** [Extension naming / KHR vs EXT](../../style/extensions.txt#L110), lines 110–125.
- **S3:** [Spatial fields](../../chapters/object_types/spatial_fields.txt), regular lines 17–48, NanoVDB 58–82, unstructured 85 onward.
- **S4:** [Frame accumulation, depth, properties](../../chapters/object_types/frame.txt), lines 45–50, 85–88, 109–116.
- **S5:** [Camera common parameters and motion](../../chapters/object_types/cameras.txt#L17), lines 17–79.
- **S6:** [Isosurface geometry](../../chapters/object_types/geometries.txt#L184), lines 184–206.
- **S7:** [Renderer model and common extensions](../../chapters/object_types.txt#L22), lines 22–69.
- **S8:** [Material emission](../../chapters/object_types/materials.txt#L76); [light primary visibility](../../chapters/object_types/lights.txt#L19); [extension index](../../appendices/extension_index.txt).
- **S9:** [Contribution and AI-assisted contribution requirements](../../CONTRIBUTING.adoc).
- **R1:** [Device dependencies](../../../VisRTX/devices/rtx/device/visrtx_device.json.in); [conditional advertisement](../../../VisRTX/devices/rtx/device/CMakeLists.txt#L410), lines 410–441.
- **R2:** [Rectilinear schema](../../../VisRTX/devices/rtx/device/visrtx_spatial_field_structured_rectilinear.json); [field implementation](../../../VisRTX/devices/rtx/device/spatial_field/StructuredRectilinearField.cpp#L180); [sampler semantics](../../../VisRTX/devices/rtx/device/spatial_field/StructuredRectilinearSamplerInline.h#L52); [LUT construction](../../../VisRTX/devices/rtx/device/spatial_field/RectilinearLUT.h#L44).
- **R3:** [Centering schema](../../../VisRTX/devices/rtx/device/visrtx_spatial_field_data_centering.json); [regular-field parameters and bounds](../../../VisRTX/devices/rtx/device/spatial_field/StructuredRegularField.cpp#L97), lines 97–110, 181–194; [sample positioning](../../../VisRTX/devices/rtx/device/spatial_field/StructuredRegularSamplerInline.h#L46).
- **R4:** [ROI schema](../../../VisRTX/devices/rtx/device/visrtx_spatial_field_region_of_interest.json); [volume clipping](../../../VisRTX/devices/rtx/device/geometry/Intersectors_ptx.cu#L219); [isosurface clipping](../../../VisRTX/devices/rtx/device/geometry/Intersectors_ptx.cu#L1093); [README](../../../VisRTX/devices/rtx/README.md#L120).
- **R5:** [Ray-buffer schema](../../../VisRTX/devices/rtx/device/visrtx_camera_ray_buffer.json); [validation](../../../VisRTX/devices/rtx/device/camera/RayBuffer.cpp#L42); [ray generation](../../../VisRTX/devices/rtx/device/gpu/cameraCreateRay.h#L75); [tests](../../../VisRTX/devices/rtx/apps/tests/api/TestRayBufferCamera.cpp#L247), especially origin/depth at lines 289–312.
- **R6:** [Culling schema](../../../VisRTX/devices/rtx/device/visrtx_triangle_back_face_culling.json); [geometry-level helper](../../../VisRTX/devices/rtx/device/gpu/populateHit.h#L372); [fast](../../../VisRTX/devices/rtx/device/renderer/Fast_ptx.cu#L79); [interactive](../../../VisRTX/devices/rtx/device/renderer/Interactive_ptx.cu#L425); [quality](../../../VisRTX/devices/rtx/device/renderer/Quality_ptx.cu#L535); [global flag](../../../VisRTX/devices/rtx/device/gpu/intersectRay.h#L80).
- **R7:** [Frame properties](../../../VisRTX/devices/rtx/device/frame/Frame.cu#L607), lines 607–635; [sample accounting](../../../VisRTX/devices/rtx/device/frame/Frame.cu#L754); [README telemetry](../../../VisRTX/devices/rtx/README.md#L218).
- **R8:** [MaterialX schema](../../../VisRTX/devices/rtx/device/visrtx_material_materialx.json); [MaterialX implementation and properties](../../../VisRTX/devices/rtx/device/material/MaterialX.cpp), notably lines 55, 144–181, 316–345; [MDL schema](../../../VisRTX/devices/rtx/device/visrtx_material_mdl.json); tests under `R/apps/tests/api/TestMaterialX*.cpp`.
- **R9:** [NanoVDB rectilinear schema](../../../VisRTX/devices/rtx/device/visrtx_spatial_field_nanovdb_rectilinear.json); [bounds and axes](../../../VisRTX/devices/rtx/device/spatial_field/NvdbRectilinearField.cpp#L131), lines 131–184, 197–203.
- **R10:** [SDF schema](../../../VisRTX/devices/rtx/device/visrtx_geometry_sdf.json); [SDF parameters](../../../VisRTX/devices/rtx/device/geometry/SDF.cpp#L81), especially lines 87–88.
- **R11:** [Neural schema](../../../VisRTX/devices/rtx/device/visrtx_geometry_neural.json); [Neural implementation](../../../VisRTX/devices/rtx/device/geometry/Neural.cpp#L17), lines 17–36, 65–80; [bounds member](../../../VisRTX/devices/rtx/device/geometry/Neural.h#L29).
- **R12:** [Renderer parameters](../../../VisRTX/devices/rtx/device/renderer/Renderer.cpp#L168), lines 168–206; [quality schema](../../../VisRTX/devices/rtx/device/visrtx_renderer_quality.json); [cut-plane hit rejection](../../../VisRTX/devices/rtx/device/gpu/populateHit.h#L386).
- **R13:** [Device controls](../../../VisRTX/devices/rtx/device/VisRTXDevice.cpp#L485), initialization at lines 485–491, material runtimes/resources at 823 onward.
- **R14:** [CUDA interoperability documentation](../../../VisRTX/devices/rtx/README.md#L66); [CUDA pointer classification](../../../VisRTX/devices/rtx/device/array/Array.cpp#L46).
- **R15:** [Compressed image implementation](../../../VisRTX/devices/rtx/device/sampler/CompressedImage2D.cpp#L63), format dispatch at lines 98–129 and 159 onward.
