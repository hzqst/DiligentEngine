# HLSL GetDimensions Cross-Device Caveat

## Trigger Signal

- RTXPT material/scene data works on one GPU/driver but fails on another after hardware switch.
- Symptoms seen in RTXPT port: reference mode fell back to normal/barycentric-like color; realtime mode lost material textures or rendered black.
- Raygen output probe confirmed display/output path was alive, so the issue was inside hit/material bridge logic.

## Root Cause / Constraint

- Do not rely on shader-side `StructuredBuffer.GetDimensions()` as a portable availability/binding guard for ray tracing scene/material tables.
- In the RTXPT port, helpers such as `hasSubInstanceTable()`, `hasMaterialTable()`, and `getMaterialCount()` used `GetDimensions()` to decide whether scene/material buffers were usable. This was fragile across Intel Arc and NVIDIA RTX 5070 Ti behavior/driver/compiler paths.
- The original RTXPT upstream assumes these buffers are correctly bound and directly indexes them. Missing/invalid binding should be treated as a Diligent-side resource binding bug, not silently hidden in shader fallback logic.

## Correct Practice

- For required ray tracing bridge buffers, bind valid buffers on the C++ side and index directly in shader code:
  - `t_SubInstanceData[getSubInstanceIndex()]`
  - `t_PTMaterialData[materialID]`
- Remove shader fallback branches that turn missing scene/material tables into barycentric, normal, or factor-only debug output.
- Validate binding and index correctness at the C++/resource setup layer. If needed, add explicit debug probes or CPU-side assertions instead of shader-side `GetDimensions()` guards.
- Do not overgeneralize: `GetDimensions()` can still be acceptable for ordinary dimension queries such as image size or non-required output texture metadata, but avoid using it as a cross-device resource-availability/count test for required RT scene/material buffers. For compute passes over required scene/material tables, pass host-known counts through constants instead.

## Validation Used

- `EnableMaterialTextures=false` did not change the failure, so texture sampling itself was not the first suspect.
- Forcing material texture sampling to constant color did not change reference output.
- Raygen probe writing magenta to `u_Output[pixel]` produced full magenta, proving output/display was connected.
- After removing `hasSubInstanceTable()` / `hasMaterialTable()` / `getMaterialCount()` guards and directly indexing buffers, textures recovered on NVIDIA RTX 5070 Ti.

## Scope

- Applies to `DiligentSamples/Samples/RTXPT` PathTracer shader bridge code and similar HLSL ray tracing ports.
- Prefer upstream RTXPT semantics for required scene/material tables: resource binding correctness is a host-side invariant.