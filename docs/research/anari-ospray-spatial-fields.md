# anari-ospray: spatial-field compatibility evidence

The public upstream sources were inspected while preparing the P1 proposals in the [VisRTX standardization review](visrtx-extension-ratification-review.md). This note records evidence, not backend conformance or Working Group approval.

## Sources inspected

- [`ospray/anari-ospray` main at `549dfd9422d468b2768da08cb9abad4d433eb4c0`](https://github.com/ospray/anari-ospray/tree/549dfd9422d468b2768da08cb9abad4d433eb4c0): [`StructuredRegularField.cpp`](https://github.com/ospray/anari-ospray/blob/549dfd9422d468b2768da08cb9abad4d433eb4c0/scene/volume/spatial_field/StructuredRegularField.cpp#L29-L52) reads `data`, `origin`, `spacing`, and `filter`, but no centering parameter.
- The same revision's [device dependencies](https://github.com/ospray/anari-ospray/blob/549dfd9422d468b2768da08cb9abad4d433eb4c0/code_gen/ospray_device.json) and [extension aggregate](https://github.com/ospray/anari-ospray/blob/549dfd9422d468b2768da08cb9abad4d433eb4c0/code_gen/ospray_extensions.json) advertise no centering extension.
- The public `amr` branch at [`714e1b4689efe4c224da20c6dd459b1cbf6f643d`](https://github.com/ospray/anari-ospray/blob/714e1b4689efe4c224da20c6dd459b1cbf6f643d/scene/volume/spatial_field/AMRField.cpp) uses block data, origins, spacing, and refinement. [anari-ospray PR #25](https://github.com/ospray/anari-ospray/pull/25) links this work to [ANARI-Docs PR #117](https://github.com/KhronosGroup/ANARI-Docs/pull/117).
- PR #117's inspected head was `f5e4d30c306ee25962f6fc93adf595da53e08801`. Its [AMR field proposal](https://github.com/KhronosGroup/ANARI-Docs/blob/f5e4d30c306ee25962f6fc93adf595da53e08801/chapters/object_types/spatial_fields.txt) and discussion concern block layout and coordinate interpretation, not a general `dataCentering` parameter.

No standalone data-centering extension was found in the inspected public sources and proposal search. This does not exclude private branches or unpublished WG discussions. If another branch or proposal becomes available, reconcile its naming and semantics before submission. Do not count AMR support as an independent implementation of dense-grid centering.

## Consequences for the drafts

- Preserve existing regular-grid node-centered defaults and the `origin`/`spacing` interface.
- Keep AMR topology and storage decisions separate from dense-field sample association.
- Do not infer ANARI support from OSPRay/OpenVKL capabilities. A half-cell origin shift alone does not establish the required outer half-cell domain and boundary reconstruction.
- Ask OSPRay maintainers to review cell-centered bounds, their cubic-filter path, and ROI over unstructured fields. Backend feasibility still needs to be demonstrated.
- A second implementation and application examples are recommended readiness evidence, not an asserted formal Khronos requirement.

No backend source files were changed or GPU tests run during this research.
