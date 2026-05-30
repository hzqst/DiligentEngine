# RTXPT Phase R0.5 — Coding Style & Naming Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-organize and re-name the RTXPT sample's ported reference path-tracer shaders (and their CPU/GPU shared structs + resource bindings) to mirror RTXPT-fork's folder layout, namespaces, symbol names, struct/field names, resource-binding names, and macros — **without changing rendering behavior** (byte-identical output) — and record the unavoidable architectural divergences in a mapping document, so future upstream RTXPT changes re-port as a near-mechanical diff/merge.

**Architecture:** This is a **behavior-preserving refactor** (goal G0.5 of the reference-path-tracer-completion spec). It touches every ported HLSL file under `DiligentSamples/Samples/RTXPT/assets/shaders/`, the C++ that references shader file paths / entry points / resource-binding names (`RTXPTRayTracingPass.cpp`), the six CPU/GPU mirrored structs (`RTXPTSample.hpp` + the CPU populators in `RTXPTLights.cpp`, `RTXPTMaterials.*`, `RTXPTAccelerationStructures.*`, `RTXPTScene.*`), and `CMakeLists.txt`. The transformation is applied as **exact token substitutions and file relocations** — *not* as logic rewrites — so the GPU instruction stream is unchanged. The user has chosen **maximal alignment** on all three axes: nested folder mirror, full lexical parity (locals re-cased), and full struct + field alignment.

**Tech Stack:** HLSL (DXC, SM 6.5), Diligent Engine sample framework (C++17), CMake. clang-format 10.0.0 validation applies **only** to C/C++ headers/sources (`c,h,cpp,hpp,…`) — **not** to `.hlsli/.hlsl/.rgen/.rchit/.rmiss/.rahit/.fxh/.csh/.vsh/.psh` (confirmed in `DiligentCore/BuildTools/FormatValidation/clang-format-validate.py:32`). So shader renames are unconstrained by clang-format; only the C++ files we touch must stay clang-format-clean.

---

## Naming rule (read this before any task)

"Full alignment" does **not** mean copying RTXPT-fork's *implementations* or *byte layouts* — our reference path tracer is a flattened raygen-loop subset and several of RTXPT-fork's structs are realtime-track-shaped (packed `PathState`/`PathPayload`, `PathTracerConstants` with ReSTIR/stable-plane/DLSS fields) with **no field-level analog** to ours. It means:

1. **Exact RTXPT-fork name** when our symbol computes the *same quantity with a compatible signature* (e.g. our GGX NDF → `evalNdfGGX`, our cosine-hemisphere pdf helper, `Hash32`/`Hash32Combine` which already match).
2. **RTXPT-fork naming *style*** (PascalCase types/functions; `m_`/`g_`/`t_`/`u_`/`s_`/`k`/`c_` prefixes; camelCase locals & parameters; HLSL `namespace` + inline helpers; traditional include guards) + the closest RTXPT-fork file/folder when there is *no* exact analog (e.g. our bespoke `RTXPTPathTracerSettings`, our flat `RTXPTPathTracerPayload`, our hybrid `RTXPTSurface`).
3. **Never rename a symbol to an RTXPT-fork name whose semantics differ.** Example: our `RTXPTVisibilitySmithGGX` returns `G/(4·NoV·NoL)` (combined visibility), whereas RTXPT-fork's `evalMaskingSmithGGXCorrelated` returns the masking term `G` alone. Renaming ours to that name would be a lie. Use a port-style name (`evalVisibilitySmithGGXCorrelated`) and record the divergence.
4. **Byte-identical is a hard invariant.** Apply renames as whole-word, case-sensitive find/replace and relocations only. Do **not** "improve" math, reorder operations, change literals, or touch struct byte layout. Every `static_assert(sizeof(...) == N)` must still hold unchanged.

**Hard constraints (from the spec):**
- Files under `DiligentSamples/` keep Diligent's Apache/Diligent copyright header where one exists today (the C++ files); the shaders currently carry **no** header — keep them header-less. **Do not** add NVIDIA copyright headers anywhere.
- C++ sample-framework glue with no RTXPT-fork analog (sample lifecycle, pass classes, binding model) keeps Diligent conventions. Only the **GPU-facing mirrored structs** and the **resource-binding variable names** align, because those are the CPU/GPU contract boundary that upstream merges cross.

---

## Why no automated tests (verification model)

HLSL shaders in this sample have **no unit-test harness**; a path tracer is only observable by running the sample. Strict TDD ("write a failing test, watch it fail") does not apply. Because this is a pure rename/relocation refactor, the **runnable** per-task verification is *grep-based rename completeness* plus a *final byte-identical render check*:

- **"Failing test" (before):** `grep` shows the old token(s) still present → work remains.
- **"Passing test" (after):** `grep` shows **zero** occurrences of every old token in scope, and the new token appears the expected number of times. This is runnable without building and proves the rename is total.
- **Compile (user-initiated):** the project builds on D3D12 + Vulkan. Per the workspace rule and the user's global `CLAUDE.md`, **do not auto-run build/run/format commands** — every such command below is marked *(user-initiated)*.
- **Byte-identical render (user-initiated, final):** because our RNG is seeded purely from `(pixel, frameIndex)` and tone mapping is deterministic, the **first accumulated frame after a reset** is bit-deterministic. Capture it before the refactor and compare after.

Layout guards (the `static_assert`s) compile unchanged — they are not edited except to rename the type inside them (Task 8), and their `sizeof(...) == N` values stay the same.

---

## Baseline capture (do once, before Task 2) *(user-initiated)*

Establish the byte-identical reference. Ask the user to run the sample on **each** backend, let one frame accumulate after a fresh launch (ResetAccumulation path), and save the output:

- D3D12: launch RTXPT, screenshot/dump the first post-reset frame → `baseline_d3d12.png`
- Vulkan: same → `baseline_vk.png`

These are compared in Task 10. If the user prefers not to capture images, the fallback is visual side-by-side after Task 10. Record in the commit/PR which method was used.

---

## File-move map (Task 2 target layout)

Nested mirror of RTXPT-fork under `assets/shaders/PathTracer/`. Moves are **1:1** (no file splitting) — each current file goes to its closest RTXPT-fork file name in the correct subfolder. The non-path-tracer shaders (blit, debug-compute, compat wrapper) stay flat.

| Current `assets/shaders/…` | New `assets/shaders/…` | RTXPT-fork analog (`Rtxpt/Shaders/…`) |
|---|---|---|
| `RTXPTReference.rgen`        | `PathTracer/PathTracerSample.rgen`                    | `PathTracer/PathTracerSample.hlsl` (raygen) |
| `RTXPTReference.rchit`       | `PathTracer/PathTracerClosestHit.rchit`               | hit handling in `PathTracer.hlsli`/`Scene/HitInfo.hlsli` |
| `RTXPTReference.rmiss`       | `PathTracer/PathTracerMiss.rmiss`                     | miss handling in `PathTracer.hlsli` |
| `RTXPTReference.rahit`       | `PathTracer/PathTracerAnyHit.rahit`                   | any-hit / alpha test |
| `RTXPTShaderShared.hlsli`    | `PathTracer/PathTracerShared.h`                       | `PathTracer/PathTracerShared.h` |
| `RTXPTSceneBridge.hlsli`     | `PathTracer/PathTracerBridge.hlsli`                   | `PathTracer/PathTracerBridge.hlsli` |
| `RTXPTBSDF.hlsli`            | `PathTracer/Rendering/Materials/BxDF.hlsli`           | `PathTracer/Rendering/Materials/BxDF.hlsli` |
| `RTXPTMaterialBridge.hlsli`  | `PathTracer/Rendering/Materials/MaterialBridge.hlsli` | material loads in `PathTracerBridge*.hlsli` |
| `RTXPTLightSampling.hlsli`   | `PathTracer/Lighting/PolymorphicLight.hlsli`          | `PathTracer/Lighting/PolymorphicLight.hlsli` |
| `RTXPTEnvironment.hlsli`     | `PathTracer/Lighting/EnvMap.hlsli`                    | `PathTracer/Lighting/EnvMap.hlsli` |
| `RTXPTRandom.hlsli`          | `PathTracer/Utils/SampleGenerators.hlsli`             | `PathTracer/Utils/SampleGenerators.hlsli` |
| `RTXPTCommon.fxh`            | `RTXPTCommon.fxh` *(stays flat — port compat wrapper)*| none |
| `RTXPTDebugCompute.csh`      | `RTXPTDebugCompute.csh` *(stays flat)*                | none |
| `RTXPTBlit.vsh` / `RTXPTBlit.psh` | unchanged *(stay flat)*                          | none |

New files created later (Task 6, Task 9): `PathTracer/PathTracer.hlsli`, `PathTracer/PathTracerHelpers.hlsli`, `PathTracer/Config.h`.

**Include resolution:** the RT pass's shader source factory is created with root `"shaders"` (`RTXPTRayTracingPass.cpp:93`). Task 2 changes it to `"shaders;shaders\\PathTracer"` so that files inside `PathTracer/` can `#include` siblings the RTXPT-fork way (`#include "Rendering/Materials/BxDF.hlsli"`, `#include "Lighting/EnvMap.hlsli"`, `#include "PathTracerShared.h"`). The raygen entry path becomes `"PathTracer/PathTracerSample.rgen"` (resolved against the `"shaders"` root). The compute/blit factories keep root `"shaders"`; the one cross-reference from `RTXPTCommon.fxh` uses a full-from-root include `"PathTracer/PathTracerShared.h"`.

---

## Authoritative rename tables (the contract for Tasks 3–9)

These tables are the single source of truth. Task 1 transcribes them into the mapping doc; Tasks 3–9 apply them. **"= RTXPT-fork"** marks an exact upstream name; **"(style)"** marks a port-specific name in RTXPT-fork style; **"(divergent)"** marks a name deliberately *not* matching an upstream symbol whose semantics differ.

### T-A. Macros, include guards, constants

| Current | New | Notes |
|---|---|---|
| `RTXPT_ENABLE_HIT_BRIDGE` | `ENABLE_HIT_BRIDGE` (style) | also in C++? no — HLSL-only `#define`; keep in shaders |
| `RTXPT_ENABLE_MATERIAL_TEXTURES` | `ENABLE_MATERIAL_TEXTURES` (style) | **also a C++ string** in `RTXPTRayTracingPass.cpp:122` `Macros.Add(...)` |
| `RTXPT_MATERIAL_TEXTURE_COUNT` | `MATERIAL_TEXTURE_COUNT` (style) | **also a C++ string** in `RTXPTRayTracingPass.cpp:123` |
| include guards `RTXPT_*_HLSLI` | `__<NAME>_HLSLI__` (= RTXPT-fork form, e.g. `__BXDF_HLSLI__`) | RTXPT-fork uses `__NAME_HLSLI__` traditional guards (DXC) |
| `RTXPT_PI` | `K_PI` = RTXPT-fork | `Utils/Math/MathConstants.hlsli` |
| `RTXPT_INV_PI` | `K_1_PI` = RTXPT-fork | |
| `RTXPT_MIN_ROUGHNESS` | `kMinRoughness` (divergent) | RTXPT-fork's `kMinGGXAlpha=0.0064` is a different quantity (alpha, not roughness); keep value `0.045` |
| `RTXPT_VISIBILITY_RAY_TMIN` / `_TMAX` | `kVisibilityRayTMin` / `kVisibilityRayTMax` (style) | |
| `kRTXPTSubInstanceFlagIndexed` | `kSubInstanceFlagIndexed` (style) | also C++ `kRTXPTSubInstanceFlag_Indexed` → `kSubInstanceFlag_Indexed` (Task 8) |
| `kRTXPTMaterialFlag*` (5) | `kMaterialFlag*` (style) | also C++ `kRTXPTMaterialFlag_*` (Task 8) |

### T-B. Utils layer (`Utils/SampleGenerators.hlsli`)

| Current | New | Notes |
|---|---|---|
| `Hash32` | `Hash32` = RTXPT-fork | already matches; no change |
| `Hash32Combine` | `Hash32Combine` = RTXPT-fork | already matches; no change |
| `ToFloat0To1` | `UintToFloat01` (style) | RTXPT-fork uses helper in `NoiseAndSequences.hlsli`; keep our impl |
| `struct RTXPTRandom` | `struct SampleGenerator` (style) | RTXPT-fork uses `SampleGeneratorType`; we keep one concrete struct |
| `RTXPTRandom_Init` | `SampleGenerator_make` (style) | |
| `NextFloat` | `sampleNext1D` = RTXPT-fork | param `inout SampleGenerator sg` |
| `NextFloat2` | `sampleNext2D` = RTXPT-fork | |
| `BuildOrthonormalBasis` | `BranchlessONB` = RTXPT-fork | `Utils/Geometry.hlsli`; params `normal,out tangent,out bitangent` |
| `SampleCosineHemisphere` | `sampleCosineHemisphere` (style) | RTXPT-fork's `sample_cosine_hemisphere_polar` returns local-frame z-up; ours bakes the basis rotation — keep ours, record divergence |

### T-C. Materials layer (`Rendering/Materials/BxDF.hlsli`)

| Current | New | Notes |
|---|---|---|
| `struct RTXPTSurface` | `struct StandardBSDFData` (style) | carries shading normal `N` too (RTXPT-fork splits `ShadingData`/`StandardBSDFData`) — record divergence |
| field `.N` | `.N` = RTXPT-fork | |
| field `.DiffuseAlbedo` | `.diffuse` = RTXPT-fork (`StandardBSDFData._diffuse`/`Diffuse()`) | plain public field, no fp16 packing |
| field `.F0` | `.specular` = RTXPT-fork | |
| field `.Alpha` | `.alpha` (style) | GGX alpha (=roughness²); RTXPT-fork keeps `roughness` + computes alpha — keep ours as `alpha` |
| `RTXPTMakeSurface` | `MakeStandardBSDFData` (style) | |
| `RTXPTFresnelSchlick(F0, VoH)` | `evalFresnelSchlick(f0, f90, cosTheta)` = RTXPT-fork | adopt 3-arg signature; pass `f90 = float3(1,1,1)` at call sites → byte-identical |
| `RTXPTDistributionGGX(NoH, Alpha)` | `evalNdfGGX(alpha, cosTheta)` = RTXPT-fork | param order swaps to `(alpha, cosTheta)` |
| `RTXPTVisibilitySmithGGX(NoV, NoL, Alpha)` | `evalVisibilitySmithGGXCorrelated(alpha, cosThetaO, cosThetaI)` (divergent) | returns `G/(4·NoV·NoL)`, NOT the bare masking term — do not name it `evalMaskingSmithGGXCorrelated` |
| `RTXPTLuminance` | `luminance` = RTXPT-fork (`Utils/ColorHelpers.hlsli`) | |
| `RTXPTSpecularProbability` | `getSpecularProbability` (style) | |
| `RTXPTEvalBSDF(S, Wo, Wi, SpecProb, out FTimesNoL, out Pdf)` | `EvalBSDF(bsdfData, wo, wi, specProb, out f, out pdf)` (style) | RTXPT-fork splits `eval`/`evalPdf`; we keep the combined helper, params re-cased |
| `RTXPTSampleBSDF(S, Wo, …, out Wi, out Weight, out Pdf)` | `SampleBSDF(bsdfData, wo, inout sg, out wi, out weight, out pdf)` (style) | |

### T-D. Lighting layer (`Lighting/PolymorphicLight.hlsli`, `Lighting/EnvMap.hlsli`)

| Current | New | Notes |
|---|---|---|
| `struct RTXPTLightSample` | `struct LightSample` = RTXPT-fork | |
| fields `.Wi/.Distance/.Radiance/.Valid` | `.dir/.distance/.radiance/.valid` (style) | RTXPT-fork `LightSample` uses lowercase fields |
| `RTXPTInvalidLightSample` | `LightSample_make_empty` (style) | RTXPT-fork uses `::empty()` patterns |
| `RTXPTNormalizeDirection` | `tryNormalize` (style) | |
| `RTXPTEvalAnalyticLight(Light, SurfacePos)` | `EvalAnalyticLight(light, surfacePos)` (style) | |
| `RTXPTEvalSky(RayDir)` (in EnvMap) | `EnvMap::Eval(worldDir)` = RTXPT-fork | wrap procedural sky in `namespace EnvMap` with method `Eval` (and keep a free fallback if call sites need it) |

### T-E. PathTracer core (`PathTracer/PathTracer.hlsli`, `PathTracerHelpers.hlsli`, raygen)

Helpers currently inline in `RTXPTReference.rgen` move into `namespace PathTracer` in `PathTracer/PathTracer.hlsli`; the MIS heuristic moves to `PathTracerHelpers.hlsli`.

| Current (rgen) | New | Notes |
|---|---|---|
| `RTXPTPowerHeuristic(PdfA, PdfB)` | `PowerHeuristic(nf, fPdf, ng, gPdf)` = RTXPT-fork (`PathTracerHelpers.hlsli`) | adopt 4-arg signature; call sites pass `(1, pdfA, 1, pdfB)` → byte-identical |
| `RTXPTMakeDefaultPayload(HitFlag)` | `PathTracer::MakeEmptyPayload(hitFlag)` (style) | |
| `RTXPTTraceVisibility(Origin, Dir, TMax)` | `PathTracer::TraceVisibilityRay(origin, dir, tMax)` (style) | |
| `RTXPTSampleAnalyticNEE(...)` | `PathTracer::SampleAnalyticNEE(...)` (style) | |
| `RTXPTSampleEnvNEE(...)` | `PathTracer::SampleEnvironmentNEE(...)` (style) | |
| `RTXPTBSDFSampledEnvMISWeight(...)` | `PathTracer::ComputeBSDFEnvMISWeight(...)` (style) | |
| `ToneMapACES` | `ToneMapACES` (unchanged) | port-specific; tone-map pass is separate Phase 6 |
| raygen locals (full lexical parity) | camelCase | `WorldPos4→worldPos4`, `Origin→origin`, `RayOrigin→rayOrigin`, `RayDir→rayDir`, `Throughput→throughput`, `PathRadiance→pathRadiance`, `PrevBsdfPdf→prevBsdfPdf`, `PrevNormal→prevNormal`, `PrevDidEnvNEE→prevDidEnvNEE`, `MaxBounces→maxBounces`, `MaxNEEBounces→maxNEEBounces`, `EnableNEE→enableNEE`, `EnableEnvNEE→enableEnvNEE`, `Bounce→bounce`, `Pixel→pixel`, `Dimensions→dimensions`, `FrameSeed→frameSeed`, `Rng→sg`, `Jitter→jitter`, `UV→uv`, `NDC→ndc`, `NextDir→nextDir`, `Weight→weight`, `Pdf→pdf`, `Wo→wo`, `Surface→bsdfData`, `Bias→bias`, `VisibilityOrigin→visibilityOrigin`, `UseNEE→useNEE`, `Survive→survive`, `Accumulated→accumulated`, `Reset→reset`, `Frame→frame`, `InvN→invN`, `Previous→previous`, `MISWeight→misWeight`, `EnvRadiance→envRadiance` |

### T-F. Bridge namespace (`PathTracerBridge.hlsli`, `MaterialBridge.hlsli`)

Keep `namespace Bridge`. Align method names to RTXPT-fork camelCase where it has analogs.

| Current `Bridge::` | New `Bridge::` | Notes |
|---|---|---|
| `GetLightCount` | `getLightCount` (style) | |
| `GetLight` | `getLight` (style) | |
| `GetSubInstanceIndex` | `getSubInstanceIndex` (style) | |
| `GetSubInstanceData` | `getSubInstanceData` (style) | |
| `HasSubInstanceTable` | `hasSubInstanceTable` (style) | |
| `GetTriangleIndices` | `getTriangleIndices` (style) | |
| `GetTriangleVertices` | `getTriangleVertices` (style) | |
| `InterpolateNormal` / `InterpolateTexCoord` | `interpolateNormal` / `interpolateTexCoord` (style) | |
| `ComputeGeometricNormal` | `computeGeometricNormal` (style) | |
| `ComputeWorldHitPosition` | `computeWorldHitPosition` (style) | |
| `ComputeWorldTangent` | `computeWorldTangent` (style) | |
| `HasMaterialTable` / `GetMaterialCount` / `GetMaterial` | `hasMaterialTable` / `getMaterialCount` / `getMaterial` (style) | |
| `SampleMaterialTexture` | `sampleMaterialTexture` (style) | |
| `GetBaseColor` / `GetEmission` / `GetMetallicRoughness` / `GetTangentNormal` | `getBaseColor` / `getEmission` / `getMetallicRoughness` / `getTangentNormal` (style) | |
| `AlphaTestPasses` | `alphaTestPasses` (style) | |

### T-G. Shared CPU/GPU structs — type names (Task 8; HLSL **and** C++)

| Current type | New type | RTXPT-fork |
|---|---|---|
| `RTXPTSubInstanceData` | `SubInstanceData` = RTXPT-fork | `Shaders/SubInstanceData.h` |
| `RTXPTPathTracerSettings` | `PathTracerConstants` = RTXPT-fork (name) | `PathTracerShared.h` (fields differ — divergent) |
| `RTXPTFrameConstants` | `SampleConstants` (style) | RTXPT-fork `SampleConstantBuffer.h` |
| `RTXPTPathTracerPayload` | `PathPayload` = RTXPT-fork (name) | `PathPayload.hlsli` (ours is unpacked — divergent) |
| `RTXPTPrimaryPayload` | `PrimaryPayload` (style) | compatibility-only |
| `RTXPTMaterialData` | `MaterialPTData` = RTXPT-fork (name) | `Materials/MaterialPT.h` (fields differ — divergent) |
| `RTXPTLightData` | `PolymorphicLightInfo` = RTXPT-fork (name) | `Lighting/PolymorphicLight.h` (ours is unpacked — divergent) |
| `RTXPTVertex` | `GeometryVertexData` (style) | RTXPT-fork uses separate attribute buffers — divergent; bare `Vertex` is too generic for `namespace Diligent` |

**Collision note:** the C++ types live in `namespace Diligent`. `SubInstanceData`, `PathTracerConstants`, `PathPayload`, `MaterialPTData`, `PolymorphicLightInfo`, `SampleConstants`, `GeometryVertexData`, `PrimaryPayload` were grepped against `DiligentSamples/Samples/RTXPT/src` and the engine headers this sample includes — none collide. If a collision surfaces at build time, prefix with `RTXPT` again for that one type and note it in the mapping doc.

### T-H. Shared struct field names (Task 8; HLSL **and** C++, byte layout unchanged)

`SubInstanceData` (was `RTXPTSubInstanceData`):

| Current field | New field | Notes |
|---|---|---|
| `MaterialID` | `MaterialID` = RTXPT-fork (concept) | keep |
| `Flags` | `Flags` | keep |
| `FirstIndex` | `IndexOffset` = RTXPT-fork | RTXPT-fork `SubInstanceData.IndexOffset` |
| `IndexCount` | `IndexCount` (style) | keep |
| `FirstVertex` | `VertexOffset` (style) | |
| `VertexCount` | `VertexCount` (style) | keep |
| `Padding0` / `Padding1` | `_padding0` / `_padding1` (style) | RTXPT-fork `_padding*` |

`PathTracerConstants` (was `RTXPTPathTracerSettings`):

| Current field | New field | Notes |
|---|---|---|
| `MaxBounces` | `bounceCount` = RTXPT-fork | |
| `AccumulationFrame` | `sampleIndex` (style; cf. RTXPT-fork `sampleBaseIndex`) | |
| `ResetAccumulation` | `resetAccumulation` (style) | |
| `MinBounces` | `minBounceCount` (style) | RR start |
| `EnableNEE` | `NEEEnabled` = RTXPT-fork | |
| `EnableEnvNEE` | `environmentNEEEnabled` (style) | |
| `EnvIntensity` | `environmentIntensity` (style) | |
| `LightIntensityScale` | `lightIntensityScale` (style) | |
| `MaxNEEBounces` | `maxNEEBounceCount` (style) | |
| `AnalyticLightCount` | `analyticLightCount` (style) | |
| `Padding1` / `Padding2` | `_padding0` / `_padding1` (style) | |

`SampleConstants` (was `RTXPTFrameConstants`):

| Current field | New field | Notes |
|---|---|---|
| `ViewProj` | `viewProj` (style) | |
| `ViewProjInv` | `viewProjInv` (style) | |
| `CameraPosition_Time` | `cameraPositionAndTime` (style) | |
| `ViewportSize_FrameIdx` | `viewportSizeAndFrameIndex` (style) | |
| `PathTracer` | `ptConsts` (style) | the embedded `PathTracerConstants` |

`PathPayload` (was `RTXPTPathTracerPayload`):

| Current field | New field | Notes |
|---|---|---|
| `WorldPos` | `worldPos` (style) | |
| `HitDistance` | `hitDistance` (style) | |
| `WorldNormal` | `worldNormal` (style) | |
| `HitFlag` | `hitFlag` (style) | |
| `BaseColor` | `baseColor` (style) | |
| `Metallic` | `metallic` (style) | |
| `Emission` | `emission` (style) | |
| `Roughness` | `roughness` (style) | |

`MaterialPTData` (was `RTXPTMaterialData`) — keep field semantics, camelCase the names:

| Current field | New field |
|---|---|
| `BaseColorFactor` | `baseColorFactor` |
| `EmissiveFactor` | `emissiveFactor` |
| `AlphaCutoff` | `alphaCutoff` |
| `Flags` | `flags` |
| `BaseColorTextureIndex` | `baseColorTextureIndex` |
| `EmissiveTextureIndex` | `emissiveTextureIndex` |
| `MetallicFactor` | `metallicFactor` |
| `RoughnessFactor` | `roughnessFactor` |
| `BaseColorTextureSlice` | `baseColorTextureSlice` |
| `EmissiveTextureSlice` | `emissiveTextureSlice` |
| `MetallicRoughnessTextureIndex` | `metallicRoughnessTextureIndex` |
| `MetallicRoughnessTextureSlice` | `metallicRoughnessTextureSlice` |
| `NormalTextureIndex` | `normalTextureIndex` |
| `NormalTextureSlice` | `normalTextureSlice` |
| `NormalScale` | `normalScale` |
| `Padding0..3` | `_padding0.._padding3` |

`PolymorphicLightInfo` (was `RTXPTLightData`):

| Current field | New field |
|---|---|
| `ColorIntensity` | `colorIntensity` |
| `PositionRange` | `positionRange` |
| `DirectionType` | `directionType` |
| `SpotAngles` | `spotAngles` |

`GeometryVertexData` (was `RTXPTVertex`):

| Current field | New field |
|---|---|
| `Position` | `position` |
| `Normal` | `normal` |
| `TexCoord0` | `texCoord0` |

### T-I. Resource-binding variable names (Task 9; HLSL globals + C++ binding strings)

RTXPT-fork uses Donut/NVRHI prefixes: `t_` (SRV), `u_` (UAV), `s_` (sampler), `g_` (cbuffer). Each HLSL global rename must be mirrored in the C++ `AddVariable` / `GetStaticVariableByName` / `GetVariableByName` / `AddImmutableSampler` / `SetStatic` / `BindRayGenShader`/etc. **string literals**.

| Current HLSL global | New | C++ touch points |
|---|---|---|
| `g_FrameConstants` (cbuffer) | `g_Const` (= RTXPT-fork `g_Const`) | `RTXPTRayTracingPass.cpp` (AddVariable/SetStatic), `RTXPTCommon.fxh`, all shaders |
| `g_TLAS` | `t_SceneBVH` (style; cf. RTXPT-fork `SceneBVH`) | `RTXPTRayTracingPass.cpp` |
| `g_OutputColor` (rgen UAV) | `u_Output` (style) | `RTXPTRayTracingPass.cpp` |
| `g_AccumColor` | `u_AccumulationBuffer` (style) | `RTXPTRayTracingPass.cpp` |
| `g_Lights` | `t_Lights` (style) | `RTXPTRayTracingPass.cpp` |
| `g_SubInstanceData` | `t_SubInstanceData` (style) | `RTXPTRayTracingPass.cpp` |
| `g_VertexBuffer` | `t_VertexBuffer` (style) | `RTXPTRayTracingPass.cpp` |
| `g_IndexBuffer` | `t_IndexBuffer` (style) | `RTXPTRayTracingPass.cpp` |
| `g_Materials` | `t_PTMaterialData` (style; cf. RTXPT-fork PTMaterial) | `RTXPTRayTracingPass.cpp` |
| `g_MaterialTextures` | `t_BindlessTextures` (style) | `RTXPTRayTracingPass.cpp` |
| `g_MaterialSampler` | `s_MaterialSampler` (style) | `RTXPTRayTracingPass.cpp` |
| `g_InputColor` (debug-compute) | `t_InputColor` (style) | `RTXPTDebugCompute.csh` only (no C++ string — bound by name? verify) |
| `g_OutputColor` (debug-compute) | `u_Output` (style) | verify binding strings in `RTXPTComputePass.cpp` |

> **Caution:** the debug-compute (`RTXPTDebugCompute.csh`) and blit shaders are *not* part of the path-tracer algorithm layer. Their resource names are renamed **only** if doing so does not exceed the path-tracer scope; if `RTXPTComputePass.cpp`/`RTXPTBlitPass.cpp` bind by name, rename in lockstep, otherwise leave them. Confirm by grepping those passes before editing. (These are explicitly low-priority; skipping them is an acceptable documented divergence.)

---

## Task 1: Author the mapping document + naming rule

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` (register the doc under `INCLUDE` so it shows in the IDE — optional but keeps it discoverable)

- [ ] **Step 1: Create the mapping doc**

Create `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` containing:
1. A short intro: this file records the RTXPT-fork ↔ DiligentEngine-port correspondence so upstream re-ports are near-mechanical. Reference the source spec (`docs/superpowers/specs/2026-05-30-rtxpt-reference-pathtracer-completion-design.md`, goal G0.5).
2. The **Naming rule** section (copy the four numbered rules + hard constraints from this plan verbatim).
3. The **File-move map** table (copy from this plan).
4. The **Authoritative rename tables** T-A … T-I (copy from this plan).
5. A **Divergences** section, initially seeded with the known structural divergences (expand in Task 10):
   - Raygen-flattened N-bounce loop vs. RTXPT-fork's `PathState`/`PathPayload` packed state machine + stable planes.
   - Diligent bridge (`Bridge::` over Diligent structured buffers) vs. Donut/NVRHI bridge (`PathTracerBridgeDonut.hlsli`).
   - `StandardBSDFData` carries the shading normal `N` (RTXPT-fork splits `ShadingData`/`StandardBSDFData`); two-lobe Lambert+GGX vs. `FalcorBSDF`.
   - `evalVisibilitySmithGGXCorrelated` returns `G/(4·NoV·NoL)`, not RTXPT-fork's bare masking `G`.
   - `PathTracerConstants`/`MaterialPTData`/`PolymorphicLightInfo`/`PathPayload` reuse the upstream *names* but are unpacked, reference-only layouts — **not** field-compatible with upstream.
   - `BxDF.hlsli` also contains the Fresnel/Microfacet helpers RTXPT-fork separates into `Fresnel.hlsli`/`Microfacet.hlsli` (consolidated for the port; splitting is a future option).

- [ ] **Step 2: (optional) Register the doc in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add to the `INCLUDE` list (after the last `src/*.hpp`):
```cmake
    RTXPT_FORK_MAPPING.md
```
*(Skip if the build system rejects non-source entries; the doc does not need to be a build input.)*

- [ ] **Step 3: Commit**
```bash
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "docs(rtxpt): add R0.5 RTXPT-fork shader naming/structure mapping doc"
```

---

## Task 2: Folder tree + file relocation (no symbol changes)

Pure relocation. Symbols, fields, macros, and code bodies are **unchanged**; only file paths, `#include` directives, the CMake `SHADERS` list, the C++ `FilePath` strings, and the RT pass source-factory search path change. Result must build and render byte-identically.

**Files:**
- Move (via `git mv`): the 11 files in the File-move map.
- Modify: every moved shader's `#include` lines; `RTXPTCommon.fxh`; `CMakeLists.txt`; `RTXPTRayTracingPass.cpp` (`FilePath` strings + factory search path).

- [ ] **Step 1: Create folders and move files**

```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders
mkdir -p PathTracer/Rendering/Materials PathTracer/Lighting PathTracer/Utils
git mv RTXPTReference.rgen        PathTracer/PathTracerSample.rgen
git mv RTXPTReference.rchit       PathTracer/PathTracerClosestHit.rchit
git mv RTXPTReference.rmiss       PathTracer/PathTracerMiss.rmiss
git mv RTXPTReference.rahit       PathTracer/PathTracerAnyHit.rahit
git mv RTXPTShaderShared.hlsli    PathTracer/PathTracerShared.h
git mv RTXPTSceneBridge.hlsli     PathTracer/PathTracerBridge.hlsli
git mv RTXPTBSDF.hlsli            PathTracer/Rendering/Materials/BxDF.hlsli
git mv RTXPTMaterialBridge.hlsli  PathTracer/Rendering/Materials/MaterialBridge.hlsli
git mv RTXPTLightSampling.hlsli   PathTracer/Lighting/PolymorphicLight.hlsli
git mv RTXPTEnvironment.hlsli     PathTracer/Lighting/EnvMap.hlsli
git mv RTXPTRandom.hlsli          PathTracer/Utils/SampleGenerators.hlsli
```

- [ ] **Step 2: Update `#include` directives inside the moved shaders**

Apply these include rewrites (paths are relative to the `shaders/PathTracer` search root added in Step 5, except `RTXPTCommon.fxh` which is relative to `shaders`):

| File | Old include | New include |
|---|---|---|
| `PathTracer/PathTracerSample.rgen` | `"RTXPTSceneBridge.hlsli"` | `"PathTracerBridge.hlsli"` |
| | `"RTXPTEnvironment.hlsli"` | `"Lighting/EnvMap.hlsli"` |
| | `"RTXPTLightSampling.hlsli"` | `"Lighting/PolymorphicLight.hlsli"` |
| | `"RTXPTRandom.hlsli"` | `"Utils/SampleGenerators.hlsli"` |
| | `"RTXPTBSDF.hlsli"` | `"Rendering/Materials/BxDF.hlsli"` |
| `PathTracer/PathTracerClosestHit.rchit` | `"RTXPTSceneBridge.hlsli"` | `"PathTracerBridge.hlsli"` |
| | `"RTXPTMaterialBridge.hlsli"` | `"Rendering/Materials/MaterialBridge.hlsli"` |
| `PathTracer/PathTracerAnyHit.rahit` | `"RTXPTSceneBridge.hlsli"` | `"PathTracerBridge.hlsli"` |
| | `"RTXPTMaterialBridge.hlsli"` | `"Rendering/Materials/MaterialBridge.hlsli"` |
| `PathTracer/PathTracerMiss.rmiss` | `"RTXPTShaderShared.hlsli"` | `"PathTracerShared.h"` |
| | `"RTXPTEnvironment.hlsli"` | `"Lighting/EnvMap.hlsli"` |
| `PathTracer/PathTracerBridge.hlsli` | `"RTXPTShaderShared.hlsli"` | `"PathTracerShared.h"` |
| `PathTracer/Rendering/Materials/BxDF.hlsli` | `"RTXPTRandom.hlsli"` | `"../../Utils/SampleGenerators.hlsli"` |
| `PathTracer/Rendering/Materials/MaterialBridge.hlsli` | `"RTXPTShaderShared.hlsli"` | `"../../PathTracerShared.h"` |
| `PathTracer/Lighting/PolymorphicLight.hlsli` | `"RTXPTShaderShared.hlsli"` | `"../PathTracerShared.h"` |

> Note on relative depths: with the search root `shaders/PathTracer`, a file in `Rendering/Materials/` includes a `Utils/` sibling via `../../Utils/…`. Either form (relative-to-file or relative-to-root `"Utils/SampleGenerators.hlsli"`) works with Diligent's default factory; the table uses relative-to-file to match how DXC resolves `#include "..."`. If the executor prefers root-relative includes everywhere, that is equally valid — pick one convention and keep it.

- [ ] **Step 3: Update `RTXPTCommon.fxh` (stays flat)**

In `assets/shaders/RTXPTCommon.fxh`, change:
```hlsl
#include "RTXPTShaderShared.hlsli"
```
to:
```hlsl
#include "PathTracer/PathTracerShared.h"
```
*(resolved against the `"shaders"` root used by the compute pass).*

- [ ] **Step 4: Update `CMakeLists.txt` SHADERS list**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, replace the `set(SHADERS …)` block's path-tracer entries with the new paths (keep blit/debug entries unchanged):
```cmake
set(SHADERS
    assets/shaders/RTXPTCommon.fxh
    assets/shaders/PathTracer/PathTracerShared.h
    assets/shaders/PathTracer/PathTracerBridge.hlsli
    assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli
    assets/shaders/PathTracer/Lighting/EnvMap.hlsli
    assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli
    assets/shaders/PathTracer/Utils/SampleGenerators.hlsli
    assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
    assets/shaders/PathTracer/PathTracerSample.rgen
    assets/shaders/PathTracer/PathTracerMiss.rmiss
    assets/shaders/PathTracer/PathTracerClosestHit.rchit
    assets/shaders/PathTracer/PathTracerAnyHit.rahit
    assets/shaders/RTXPTDebugCompute.csh
    assets/shaders/RTXPTBlit.vsh
    assets/shaders/RTXPTBlit.psh
)
```

- [ ] **Step 5: Update the RT pass source-factory search path + entry FilePaths**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`:

Change the factory creation (line ~93):
```cpp
pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders", &pShaderSourceFactory);
```
to:
```cpp
pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PathTracer", &pShaderSourceFactory);
```

Change the four `ShaderCI.FilePath` strings:
| Line ~ | Old | New |
|---|---|---|
| 107 | `"RTXPTReference.rgen"` | `"PathTracer/PathTracerSample.rgen"` |
| 114 | `"RTXPTReference.rmiss"` | `"PathTracer/PathTracerMiss.rmiss"` |
| 130 | `"RTXPTReference.rchit"` | `"PathTracer/PathTracerClosestHit.rchit"` |
| 139 | `"RTXPTReference.rahit"` | `"PathTracer/PathTracerAnyHit.rahit"` |

- [ ] **Step 6: Verify no stale references remain**

Run *(no build needed)*:
```bash
cd DiligentSamples/Samples/RTXPT
grep -rn "RTXPTReference\.\(rgen\|rmiss\|rchit\|rahit\)\|RTXPTShaderShared\.hlsli\|RTXPTSceneBridge\.hlsli\|RTXPTBSDF\.hlsli\|RTXPTMaterialBridge\.hlsli\|RTXPTLightSampling\.hlsli\|RTXPTEnvironment\.hlsli\|RTXPTRandom\.hlsli" src assets CMakeLists.txt
```
Expected: **no output** (every old shader path is gone). If anything prints, fix that reference.

- [ ] **Step 7: Compile-check both backends** *(user-initiated)*

Run *(user-initiated)*: `cmake --build build/x64/Debug --config Debug`
Expected: builds; RT PSO compiles (includes resolve through the new search path).

- [ ] **Step 8: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT
git commit -m "refactor(rtxpt): relocate reference path-tracer shaders into PathTracer/ tree (no behavior change)"
```

---

## Task 3: Utils layer symbol renames (`Utils/SampleGenerators.hlsli`)

Apply table **T-B** + the constants/guard rows of **T-A** that live here. These symbols are referenced by `BxDF.hlsli`, `PathTracerSample.rgen`, and `PolymorphicLight.hlsli` — update **all** call sites in this task.

**Files:**
- Modify: `PathTracer/Utils/SampleGenerators.hlsli`
- Modify (call sites): `PathTracer/Rendering/Materials/BxDF.hlsli`, `PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Confirm the old symbols are present (the "failing test")**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTRandom\|RTXPTRandom_Init\|NextFloat\b\|NextFloat2\|BuildOrthonormalBasis\|SampleCosineHemisphere\|ToFloat0To1\|RTXPT_RANDOM_HLSLI" .
```
Expected: matches in `Utils/SampleGenerators.hlsli`, `Rendering/Materials/BxDF.hlsli`, `PathTracerSample.rgen`.

- [ ] **Step 2: Rename inside `Utils/SampleGenerators.hlsli`**

Apply whole-word, case-sensitive substitutions:
- guard `RTXPT_RANDOM_HLSLI` → `__SAMPLE_GENERATORS_HLSLI__` (both `#ifndef` and `#define` and the trailing `#endif` comment)
- `struct RTXPTRandom` → `struct SampleGenerator`; every `RTXPTRandom` type usage → `SampleGenerator`
- `RTXPTRandom_Init` → `SampleGenerator_make`
- `NextFloat` → `sampleNext1D`; `NextFloat2` → `sampleNext2D`
- `ToFloat0To1` → `UintToFloat01`
- `BuildOrthonormalBasis` → `BranchlessONB`
- `SampleCosineHemisphere` → `sampleCosineHemisphere`
- parameter/local re-casing within these functions: `PixelPos→pixelPos`, `FrameSeed→frameSeed`, `Rng→sg`, `Normal→normal`, `Tangent→tangent`, `Bitangent→bitangent`, `Rand→rand`, `Pdf→pdf`, `Theta→theta`, plus `Seed→seed`, `Value→value`, `x→x` (leave single-letter math vars).

`Hash32` and `Hash32Combine` already match RTXPT-fork — **do not change**.

- [ ] **Step 3: Update call sites in `BxDF.hlsli` and `PathTracerSample.rgen`**

In `Rendering/Materials/BxDF.hlsli`:
- include already points to `SampleGenerators.hlsli` (Task 2)
- `inout RTXPTRandom Rng` → `inout SampleGenerator sg` (in `RTXPTSampleBSDF` signature — its body is renamed in Task 4, but the type/param token changes here)
- `NextFloat2(Rng)` → `sampleNext2D(sg)`, `NextFloat(Rng)` → `sampleNext1D(sg)`
- `BuildOrthonormalBasis(...)` → `BranchlessONB(...)`
- `SampleCosineHemisphere(...)` → `sampleCosineHemisphere(...)`

In `PathTracerSample.rgen` (only the Utils symbols here; rgen local re-casing is Task 6):
- `RTXPTRandom Rng = RTXPTRandom_Init(...)` → `SampleGenerator Rng = SampleGenerator_make(...)` *(the `Rng→sg` local rename happens in Task 6; here only the type + factory + the Utils calls change)*
- `NextFloat2(Rng)` → `sampleNext2D(Rng)`, `NextFloat(Rng)` → `sampleNext1D(Rng)`
- `inout RTXPTRandom Rng` parameters on the rgen-local NEE helpers → `inout SampleGenerator Rng`

- [ ] **Step 4: Verify the rename is total (the "passing test")**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTRandom\|\bNextFloat\b\|NextFloat2\|BuildOrthonormalBasis\|SampleCosineHemisphere\|ToFloat0To1\|RTXPT_RANDOM_HLSLI" .
```
Expected: **no output**.
```bash
grep -rln "SampleGenerator\|sampleNext1D\|sampleNext2D\|BranchlessONB\|sampleCosineHemisphere" .
```
Expected: lists `Utils/SampleGenerators.hlsli`, `Rendering/Materials/BxDF.hlsli`, `PathTracerSample.rgen`.

- [ ] **Step 5: Compile-check** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`

- [ ] **Step 6: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT/assets/shaders
git commit -m "refactor(rtxpt): align Utils sample-generator/geometry symbol names with RTXPT-fork"
```

---

## Task 4: Materials layer symbol renames (`Rendering/Materials/BxDF.hlsli`)

Apply table **T-C** + the math-constant rows of **T-A** (`RTXPT_PI`, `RTXPT_INV_PI`, `RTXPT_MIN_ROUGHNESS`, guard). Update call sites in `PathTracerSample.rgen`. (The `PowerHeuristic`/`Luminance` lived in this file historically but move to `PathTracerHelpers.hlsli`/`ColorHelpers` — handled in Step 3 below.)

**Files:**
- Modify: `PathTracer/Rendering/Materials/BxDF.hlsli`
- Modify (call sites): `PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Confirm old symbols present**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTSurface\|RTXPTMakeSurface\|RTXPTFresnelSchlick\|RTXPTDistributionGGX\|RTXPTVisibilitySmithGGX\|RTXPTLuminance\|RTXPTSpecularProbability\|RTXPTEvalBSDF\|RTXPTSampleBSDF\|RTXPT_PI\|RTXPT_INV_PI\|RTXPT_MIN_ROUGHNESS\|RTXPT_BSDF_HLSLI\|RTXPTPowerHeuristic" .
```

- [ ] **Step 2: Rename types/constants/helpers inside `BxDF.hlsli`**

- guard `RTXPT_BSDF_HLSLI` → `__BXDF_HLSLI__`
- `RTXPT_PI` → `K_PI`; `RTXPT_INV_PI` → `K_1_PI`; `RTXPT_MIN_ROUGHNESS` → `kMinRoughness` (values unchanged)
- `struct RTXPTSurface` → `struct StandardBSDFData`; all `RTXPTSurface` usages → `StandardBSDFData`
  - field `DiffuseAlbedo` → `diffuse`; `F0` → `specular`; `Alpha` → `alpha`; `N` stays `N`
- `RTXPTMakeSurface` → `MakeStandardBSDFData`
- `RTXPTFresnelSchlick(float3 F0, float VoH)` → `evalFresnelSchlick(float3 f0, float3 f90, float cosTheta)`:
  ```hlsl
  float3 evalFresnelSchlick(float3 f0, float3 f90, float cosTheta)
  {
      const float f = pow(saturate(1.0 - cosTheta), 5.0);
      return f0 + (f90 - f0) * f;
  }
  ```
  (3-arg form; the previous `(1.0 - F0)` becomes `(f90 - f0)` with `f90 = float3(1,1,1)` at call sites — algebraically identical.)
- `RTXPTDistributionGGX(float NoH, float Alpha)` → `evalNdfGGX(float alpha, float cosTheta)` (swap param order; body uses `cosTheta` where `NoH` was, `alpha` where `Alpha` was)
- `RTXPTVisibilitySmithGGX(float NoV, float NoL, float Alpha)` → `evalVisibilitySmithGGXCorrelated(float alpha, float cosThetaO, float cosThetaI)` (re-cased params; body unchanged — still returns `0.5 / (V + L)`)
- `RTXPTLuminance` → `luminance`
- `RTXPTSpecularProbability(RTXPTSurface S, float3 Wo)` → `getSpecularProbability(StandardBSDFData bsdfData, float3 wo)` (re-case all locals: `S→bsdfData`, `Wo→wo`, `NoV→NdotV`, `Fapprox→fApprox`, `SpecLum→specLum`, `DiffLum→diffLum`)
- `RTXPTEvalBSDF(RTXPTSurface S, float3 Wo, float3 Wi, float SpecProb, out float3 FTimesNoL, out float Pdf)` → `EvalBSDF(StandardBSDFData bsdfData, float3 wo, float3 wi, float specProb, out float3 f, out float pdf)` (re-case all locals: `NoL→NdotL`, `NoV→NdotV`, `H→h`, `NoH→NdotH`, `VoH→VdotH`, `D→d`, `Vis→vis`, `F→fresnel`, `Spec→spec`, `Diff→diff`, `PdfDiffuse→pdfDiffuse`, `PdfSpecular→pdfSpecular`)
  - update internal calls: `RTXPTDistributionGGX(NoH, S.Alpha)` → `evalNdfGGX(bsdfData.alpha, NdotH)`; `RTXPTVisibilitySmithGGX(NoV, NoL, S.Alpha)` → `evalVisibilitySmithGGXCorrelated(bsdfData.alpha, NdotV, NdotL)`; `RTXPTFresnelSchlick(S.F0, VoH)` → `evalFresnelSchlick(bsdfData.specular, float3(1,1,1), VdotH)`
- `RTXPTSampleBSDF(RTXPTSurface S, float3 Wo, inout SampleGenerator sg, out float3 Wi, out float3 Weight, out float Pdf)` → `SampleBSDF(StandardBSDFData bsdfData, float3 wo, inout SampleGenerator sg, out float3 wi, out float3 weight, out float pdf)` (re-case locals: `NoV→NdotV`, `SpecProb→specProb`, `Tangent→tangent`, `Bitangent→bitangent`, `Rand2→rand2`, `Lobe→lobe`, `A→a`, `Phi→phi`, `CosT→cosT`, `SinT→sinT`, `HLocal→hLocal`, `H→h`, `PdfUnused→pdfUnused`, `FTimesNoL→f`)
  - update internal calls accordingly (`getSpecularProbability`, `BranchlessONB`, `sampleNext2D`, `sampleNext1D`, `sampleCosineHemisphere`, `EvalBSDF`)
- `RTXPTPowerHeuristic` lived here — **delete it from `BxDF.hlsli`** (it moves to `PathTracerHelpers.hlsli` in Task 6). For now, to keep the file self-consistent and compilable, leave `RTXPTPowerHeuristic` **only if** `BxDF.hlsli` itself calls it (it does not). Confirm: `grep -n RTXPTPowerHeuristic Rendering/Materials/BxDF.hlsli` — it is *defined* here but called only from the rgen. Move its definition to `PathTracerHelpers.hlsli` is Task 6; for Task 4, **leave the definition in place but renamed** `PowerHeuristic(float nf, float fPdf, float ng, float gPdf)` returning `(nf*fPdf)² / ((nf*fPdf)² + (ng*gPdf)²)`. Task 6 relocates it.

> Decision for executor: to avoid a transient broken state, keep `PowerHeuristic` defined in `BxDF.hlsli` at the end of Task 4 (renamed + 4-arg), and physically relocate it to `PathTracerHelpers.hlsli` in Task 6. The rgen call site is updated in Task 6.

- [ ] **Step 3: Update call sites in `PathTracerSample.rgen`**

Only the BSDF/surface tokens (rgen local re-casing is Task 6):
- `RTXPTSurface  Surface = RTXPTMakeSurface(...)` → `StandardBSDFData Surface = MakeStandardBSDFData(...)`
- `RTXPTSpecularProbability(Surface, Wo)` → `getSpecularProbability(Surface, Wo)`
- `RTXPTEvalBSDF(Surface, Wo, ..., FTimesNoL, BsdfPdf)` → `EvalBSDF(Surface, Wo, ..., FTimesNoL, BsdfPdf)`
- `RTXPTSampleBSDF(Surface, Wo, Rng, NextDir, Weight, Pdf)` → `SampleBSDF(Surface, Wo, Rng, NextDir, Weight, Pdf)`
- `Surface.N` field accesses unchanged (`N` kept); any `.DiffuseAlbedo/.F0/.Alpha` in rgen → `.diffuse/.specular/.alpha` (grep to confirm; rgen reads `Surface.N` only)
- `RTXPTPowerHeuristic(...)` call sites → `PowerHeuristic(1.0, pdfA, 1.0, pdfB)` form: in `RTXPTSampleEnvNEE` `RTXPTPowerHeuristic(EnvPdf, BsdfPdf)` → `PowerHeuristic(1.0, EnvPdf, 1.0, BsdfPdf)`; in `RTXPTBSDFSampledEnvMISWeight` `RTXPTPowerHeuristic(PrevBsdfPdf, EnvPdf)` → `PowerHeuristic(1.0, PrevBsdfPdf, 1.0, EnvPdf)`

- [ ] **Step 4: Verify total rename**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTSurface\|RTXPTMakeSurface\|RTXPTFresnelSchlick\|RTXPTDistributionGGX\|RTXPTVisibilitySmithGGX\|RTXPTLuminance\|RTXPTSpecularProbability\|RTXPTEvalBSDF\|RTXPTSampleBSDF\|RTXPT_PI\|RTXPT_INV_PI\|RTXPT_MIN_ROUGHNESS\|RTXPT_BSDF_HLSLI\|RTXPTPowerHeuristic" .
```
Expected: **no output**.

- [ ] **Step 5: Compile-check** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`

- [ ] **Step 6: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT/assets/shaders
git commit -m "refactor(rtxpt): align BSDF (BxDF) symbol names + math constants with RTXPT-fork"
```

---

## Task 5: Lighting layer symbol renames (`Lighting/PolymorphicLight.hlsli`, `Lighting/EnvMap.hlsli`)

Apply table **T-D**. Update call sites in `PathTracerSample.rgen` and `PathTracerMiss.rmiss`.

**Files:**
- Modify: `PathTracer/Lighting/PolymorphicLight.hlsli`, `PathTracer/Lighting/EnvMap.hlsli`
- Modify (call sites): `PathTracer/PathTracerSample.rgen`, `PathTracer/PathTracerMiss.rmiss`

- [ ] **Step 1: Confirm old symbols present**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTLightSample\|RTXPTInvalidLightSample\|RTXPTNormalizeDirection\|RTXPTEvalAnalyticLight\|RTXPTEvalSky\|RTXPT_LIGHT_SAMPLING_HLSLI\|RTXPT_ENVIRONMENT_HLSLI" .
```

- [ ] **Step 2: Rename inside `Lighting/PolymorphicLight.hlsli`**
- guard `RTXPT_LIGHT_SAMPLING_HLSLI` → `__POLYMORPHIC_LIGHT_HLSLI__`
- `struct RTXPTLightSample` → `struct LightSample`; fields `Wi→dir`, `Distance→distance`, `Radiance→radiance`, `Valid→valid`
- `RTXPTInvalidLightSample` → `LightSample_make_empty`
- `RTXPTNormalizeDirection` → `tryNormalize`
- `RTXPTEvalAnalyticLight(RTXPTLightData Light, float3 SurfacePos)` → `EvalAnalyticLight(PolymorphicLightInfo light, float3 surfacePos)` *(the `RTXPTLightData` → `PolymorphicLightInfo` type token is finalized in Task 8; for now this file still sees `RTXPTLightData` until Task 8 renames the struct — **keep the type token as `RTXPTLightData` here** and only re-case the parameter/local names + function name in this task; the type rename is Task 8 to keep the C++/HLSL contract change atomic).* Re-case locals: `Light→light`, `SurfacePos→surfacePos`, `Type→type`, `Radiance→radiance`, `MaxEnergy→maxEnergy`, `Sample→ls`, `ToLight→toLight`, `DistSq→distSq`, `Distance→distance`, `Range→range`, `Attenuation→attenuation`, `LightDir→lightDir`, `CosTheta→cosTheta`, `InnerCos→innerCos`, `OuterCos→outerCos`, `Cone→cone`, `V→v`, `Dir→dir`, `LenSq→lenSq`
- update `Sample.Wi/.Distance/.Radiance/.Valid` → `ls.dir/.distance/.radiance/.valid`

> **Type-token discipline:** struct *type* renames that cross the CPU/GPU boundary (`RTXPTLightData`, `RTXPTSubInstanceData`, `RTXPTMaterialData`, …) are all done atomically in **Task 8** so HLSL and C++ change together. In Tasks 4–7 you rename *functions, fields-of-shader-only-structs, namespaces, params, locals, macros* — but leave the six shared **type tokens** at their `RTXPT…` names until Task 8. (Field renames of the shared structs are also Task 8.)

- [ ] **Step 3: Rename inside `Lighting/EnvMap.hlsli`**

Wrap the procedural sky in a `namespace EnvMap` with an `Eval` method-style free function, matching RTXPT-fork's `EnvMap::Eval`:
```hlsl
#ifndef __ENVMAP_HLSLI__
#define __ENVMAP_HLSLI__

// Procedural-sky environment (RTXPT-fork's EnvMap importance-sampled HDR map lands in Phase R4).
namespace EnvMap
{
    float3 Eval(float3 worldDir)
    {
        const float  t       = saturate(worldDir.y * 0.5 + 0.5);
        const float3 horizon = float3(0.48, 0.58, 0.68);
        const float3 zenith  = float3(0.05, 0.08, 0.14);
        return lerp(horizon, zenith, t);
    }
} // namespace EnvMap

#endif // __ENVMAP_HLSLI__
```

- [ ] **Step 4: Update call sites**

In `PathTracerSample.rgen`:
- `RTXPTLightSample Light = RTXPTEvalAnalyticLight(Bridge::GetLight(LightIndex), HitPos)` → `LightSample Light = EvalAnalyticLight(Bridge::GetLight(LightIndex), HitPos)` *(the `Bridge::GetLight` rename is Task 7; here only the light-sample type + eval fn change)*
- `Light.Valid` → `Light.valid`, `Light.Wi` → `Light.dir`, `Light.Distance` → `Light.distance`, `Light.Radiance` → `Light.radiance`
- `RTXPTEvalSky(Wi)` → `EnvMap::Eval(Wi)`

In `PathTracerMiss.rmiss`:
- `RTXPTEvalSky(WorldRayDirection())` → `EnvMap::Eval(WorldRayDirection())`

- [ ] **Step 5: Verify total rename**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "RTXPTLightSample\|RTXPTInvalidLightSample\|RTXPTNormalizeDirection\|RTXPTEvalAnalyticLight\|RTXPTEvalSky\|RTXPT_LIGHT_SAMPLING_HLSLI\|RTXPT_ENVIRONMENT_HLSLI" .
```
Expected: **no output**.

- [ ] **Step 6: Compile-check** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`

- [ ] **Step 7: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT/assets/shaders
git commit -m "refactor(rtxpt): align lighting (PolymorphicLight/EnvMap) symbol names with RTXPT-fork"
```

---

## Task 6: PathTracer core — namespace extraction + helpers + raygen lexical parity

Extract the rgen-local helpers into `namespace PathTracer` in a new `PathTracer/PathTracer.hlsli`; move the MIS heuristic into a new `PathTracer/PathTracerHelpers.hlsli`; re-case **all** raygen locals (table **T-E**). This is the **highest behavior-risk task** — apply as substitution/relocation only, never altering math or ordering.

**Files:**
- Create: `PathTracer/PathTracer.hlsli`, `PathTracer/PathTracerHelpers.hlsli`
- Modify: `PathTracer/PathTracerSample.rgen`, `PathTracer/Rendering/Materials/BxDF.hlsli` (remove the relocated `PowerHeuristic`)

- [ ] **Step 1: Create `PathTracer/PathTracerHelpers.hlsli`**

Move the MIS heuristic here (it was renamed to `PowerHeuristic` 4-arg in Task 4, currently in `BxDF.hlsli`):
```hlsl
#ifndef __PATH_TRACER_HELPERS_HLSLI__
#define __PATH_TRACER_HELPERS_HLSLI__

// Power heuristic for MIS (Veach). Matches RTXPT-fork PathTracerHelpers.hlsli signature.
float PowerHeuristic(float nf, float fPdf, float ng, float gPdf)
{
    const float f = nf * fPdf;
    const float g = ng * gPdf;
    const float f2 = f * f;
    const float g2 = g * g;
    return f2 / max(f2 + g2, 1e-7);
}

#endif // __PATH_TRACER_HELPERS_HLSLI__
```
Then **delete** the `PowerHeuristic` definition from `BxDF.hlsli`.

> Byte-identical check: the original `RTXPTPowerHeuristic(A, B)` computed `A² / max(A² + B², 1e-7)`. With `nf = ng = 1`, this helper computes `(1·fPdf)² / max((1·fPdf)² + (1·gPdf)², 1e-7)` — identical when call sites pass `(1, A, 1, B)` (done in Task 4 Step 3).

- [ ] **Step 2: Create `PathTracer/PathTracer.hlsli` with `namespace PathTracer`**

Move the five rgen-local helper functions here, wrapped in `namespace PathTracer`, renamed per table **T-E**, with camelCase locals/params. Include the dependencies. The file:

```hlsl
#ifndef __PATH_TRACER_HLSLI__
#define __PATH_TRACER_HLSLI__

#include "PathTracerBridge.hlsli"
#include "PathTracerHelpers.hlsli"
#include "Lighting/EnvMap.hlsli"
#include "Lighting/PolymorphicLight.hlsli"
#include "Utils/SampleGenerators.hlsli"
#include "Rendering/Materials/BxDF.hlsli"

static const float kVisibilityRayTMin = 1e-4;
static const float kVisibilityRayTMax = 1e30;

namespace PathTracer
{
    PathPayload MakeEmptyPayload(uint hitFlag)
    {
        PathPayload payload;
        payload.worldPos    = float3(0.0, 0.0, 0.0);
        payload.hitDistance = -1.0;
        payload.worldNormal = float3(0.0, 1.0, 0.0);
        payload.hitFlag     = hitFlag;
        payload.baseColor   = float3(0.0, 0.0, 0.0);
        payload.emission    = float3(0.0, 0.0, 0.0);
        payload.metallic    = 0.0;
        payload.roughness   = 1.0;
        return payload;
    }

    bool TraceVisibilityRay(float3 origin, float3 dir, float tMax)
    {
        if (tMax <= kVisibilityRayTMin)
            return false;

        RayDesc ray;
        ray.Origin    = origin;
        ray.Direction = dir;
        ray.TMin      = kVisibilityRayTMin;
        ray.TMax      = tMax;

        PathPayload payload = MakeEmptyPayload(1u);
        TraceRay(t_SceneBVH,
                 RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER,
                 0xFF, 0, 1, 0, ray, payload);
        return payload.hitFlag == 0u;
    }

    float3 SampleAnalyticNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                             float3 wo, inout SampleGenerator sg)
    {
        const uint lightCount = Bridge::getLightCount();
        if (lightCount == 0u || g_Const.ptConsts.lightIntensityScale <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const uint lightIndex = min(uint(sampleNext1D(sg) * float(lightCount)), lightCount - 1u);

        LightSample light = EvalAnalyticLight(Bridge::getLight(lightIndex), hitPos);
        if (!light.valid)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, light.dir, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, light.dir, light.distance))
            return float3(0.0, 0.0, 0.0);

        return f * light.radiance * g_Const.ptConsts.lightIntensityScale * float(lightCount);
    }

    float3 SampleEnvironmentNEE(StandardBSDFData bsdfData, float3 visibilityOrigin,
                                float3 wo, inout SampleGenerator sg)
    {
        if (g_Const.ptConsts.environmentIntensity <= 0.0)
            return float3(0.0, 0.0, 0.0);

        float envPdf;
        const float3 wi = sampleCosineHemisphere(sampleNext2D(sg), bsdfData.N, envPdf);
        if (envPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, wi, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, wi, kVisibilityRayTMax))
            return float3(0.0, 0.0, 0.0);

        const float3 envRadiance = EnvMap::Eval(wi) * g_Const.ptConsts.environmentIntensity;
        const float  misWeight   = PowerHeuristic(1.0, envPdf, 1.0, bsdfPdf);
        return f * envRadiance * (misWeight / envPdf);
    }

    float ComputeBSDFEnvMISWeight(bool didEnvNEE, float prevBsdfPdf, float3 prevNormal, float3 rayDir)
    {
        if (!didEnvNEE || prevBsdfPdf <= 0.0)
            return 1.0;

        const float envPdf = max(dot(prevNormal, rayDir), 0.0) * K_1_PI;
        return PowerHeuristic(1.0, prevBsdfPdf, 1.0, envPdf);
    }
} // namespace PathTracer

#endif // __PATH_TRACER_HLSLI__
```

> Note the references to `t_SceneBVH`, `g_Const`, `g_Const.ptConsts.*`, and field-renamed `PathPayload`/`PolymorphicLightInfo`: those tokens are finalized in Tasks 7/8/9. **When authoring this file in Task 6, use the *current* names** (`g_TLAS`, `g_FrameConstants`, `g_FrameConstants.PathTracer.LightIntensityScale`, `RTXPTPathTracerPayload` if the struct token is still `RTXPT…`, etc.) and let Tasks 7–9 rename them globally. The version shown above is the **final** state after all tasks; the executor should write Task 6 against the names that exist at Task 6 time and trust the later global renames. (Grep checks in Tasks 7–9 will catch anything missed.)

- [ ] **Step 3: Rewrite `PathTracerSample.rgen` to call `PathTracer::` and use camelCase locals**

The raygen file becomes (final post-all-tasks state shown; in Task 6 use names current at this point and let Tasks 7–9 finish global renames):

```hlsl
#include "PathTracer.hlsli"

RaytracingAccelerationStructure                t_SceneBVH;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4>   u_Output;
VK_IMAGE_FORMAT("rgba32f") RWTexture2D<float4> u_AccumulationBuffer;

static float3 ToneMapACES(float3 x)
{
    const float a = 2.51;
    const float b = 0.03;
    const float c = 2.43;
    const float d = 0.59;
    const float e = 0.14;
    return saturate((x * (a * x + b)) / (x * (c * x + d) + e));
}

[shader("raygeneration")]
void main()
{
    const uint2 pixel      = DispatchRaysIndex().xy;
    const uint2 dimensions = DispatchRaysDimensions().xy;

    const uint      frameSeed = asuint(g_Const.viewportSizeAndFrameIndex.w);
    SampleGenerator sg        = SampleGenerator_make(pixel, frameSeed);

    const float2 jitter = sampleNext2D(sg);
    const float2 uv     = (float2(pixel) + jitter) / float2(dimensions);
    const float2 ndc    = uv * 2.0 - 1.0;

    const float4 worldPos4 = mul(float4(ndc, 1.0, 1.0), g_Const.viewProjInv);
    const float3 origin    = g_Const.cameraPositionAndTime.xyz;
    float3       rayOrigin = origin;
    float3       rayDir    = normalize(worldPos4.xyz / worldPos4.w - origin);

    float3 throughput   = float3(1.0, 1.0, 1.0);
    float3 pathRadiance = float3(0.0, 0.0, 0.0);

    float  prevBsdfPdf   = 0.0;
    float3 prevNormal    = float3(0.0, 1.0, 0.0);
    bool   prevDidEnvNEE = false;
    const uint maxBounces    = max(g_Const.ptConsts.bounceCount, 1u);
    const uint maxNEEBounces = min(g_Const.ptConsts.maxNEEBounceCount, maxBounces);
    const bool enableNEE     = g_Const.ptConsts.NEEEnabled != 0u;
    const bool enableEnvNEE  = enableNEE && g_Const.ptConsts.environmentNEEEnabled != 0u;

    [loop]
    for (uint bounce = 0u; bounce < maxBounces; ++bounce)
    {
        RayDesc ray;
        ray.Origin    = rayOrigin;
        ray.Direction = rayDir;
        ray.TMin      = 1e-3;
        ray.TMax      = 10000.0;

        PathPayload payload = PathTracer::MakeEmptyPayload(0u);

        TraceRay(t_SceneBVH, RAY_FLAG_NONE, 0xFF, 0, 1, 0, ray, payload);

        if (payload.hitFlag == 0u)
        {
            const float  misWeight   = PathTracer::ComputeBSDFEnvMISWeight(prevDidEnvNEE, prevBsdfPdf, prevNormal, rayDir);
            const float3 envRadiance = payload.emission * g_Const.ptConsts.environmentIntensity;
            pathRadiance += throughput * envRadiance * misWeight;
            break;
        }

        pathRadiance += throughput * payload.emission;

        const float3    wo       = -rayDir;
        StandardBSDFData bsdfData = MakeStandardBSDFData(payload.worldNormal, payload.baseColor, payload.metallic, payload.roughness);

        const float  bias             = max(1e-4, 1e-3 * payload.hitDistance);
        const float3 visibilityOrigin = payload.worldPos + bsdfData.N * bias;

        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sg);
            if (enableEnvNEE)
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sg);
        }

        float3 nextDir;
        float3 weight;
        float  pdf;
        if (!SampleBSDF(bsdfData, wo, sg, nextDir, weight, pdf))
            break;

        throughput *= weight;

        if (bounce >= g_Const.ptConsts.minBounceCount)
        {
            const float survive = clamp(max(throughput.x, max(throughput.y, throughput.z)), 0.05, 1.0);
            if (sampleNext1D(sg) > survive)
                break;
            throughput /= survive;
        }

        prevBsdfPdf   = pdf;
        prevNormal    = bsdfData.N;
        prevDidEnvNEE = useNEE && enableEnvNEE;

        rayOrigin = visibilityOrigin;
        rayDir    = nextDir;
    }

    float3     accumulated = pathRadiance;
    const uint reset       = g_Const.ptConsts.resetAccumulation;
    const uint frame       = max(g_Const.ptConsts.sampleIndex, 1u);
    if (reset == 0u)
    {
        const float4 previous = u_AccumulationBuffer[pixel];
        const float  invN     = 1.0 / float(frame);
        accumulated           = previous.rgb + (pathRadiance - previous.rgb) * invN;
    }
    u_AccumulationBuffer[pixel] = float4(accumulated, 1.0);

    u_Output[pixel] = float4(ToneMapACES(accumulated), 1.0);
}

// TODO(RTXPT-Port Phase R6): Add transmission / nested dielectrics to the BSDF (currently opaque diffuse + GGX specular).
// TODO(RTXPT-Port Phase R2/R3/R4): Emissive-triangle area lights, light importance sampling / RIS, HDR environment-map MIS.
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

> The trailing `TODO(RTXPT-Port Phase 5.3/5.4)` markers are re-targeted to their R-phase IDs per the spec's marker policy (5.3→R6, 5.4→R2/R3/R4). This is a comment-only change and keeps byte-identical output.

- [ ] **Step 4: Behavior-identity self-check (manual diff review)**

Diff the new rgen against the pre-Task-6 version (`git diff`). Confirm **every** change is one of: (a) a token rename, (b) a function-call relocation into `PathTracer::`, or (c) a comment edit. There must be **no** changed numeric literal, operator, control-flow, or operation order. Pay special attention to: `max(...,1u)` clamps, `1e-3`/`1e-4`/`10000.0`/`0.05` literals, the accumulation blend (`previous.rgb + (pathRadiance - previous.rgb) * invN`), and the RR `clamp(..., 0.05, 1.0)`.

- [ ] **Step 5: Verify no rgen-local PascalCase tokens remain + helpers relocated**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -nE "\b(Throughput|PathRadiance|RayOrigin|RayDir|PrevBsdfPdf|PrevNormal|PrevDidEnvNEE|MaxBounces|MaxNEEBounces|EnableNEE|EnableEnvNEE|NextDir|VisibilityOrigin|Surface)\b" PathTracerSample.rgen
grep -rn "RTXPTMakeDefaultPayload\|RTXPTTraceVisibility\|RTXPTSampleAnalyticNEE\|RTXPTSampleEnvNEE\|RTXPTBSDFSampledEnvMISWeight\|RTXPT_VISIBILITY_RAY" .
grep -n "PowerHeuristic" Rendering/Materials/BxDF.hlsli
```
Expected: first two **no output**; third (`PowerHeuristic` in BxDF) **no output** (relocated).

- [ ] **Step 6: Update CMake SHADERS for the two new files**

Add to the `SHADERS` list in `CMakeLists.txt`:
```cmake
    assets/shaders/PathTracer/PathTracer.hlsli
    assets/shaders/PathTracer/PathTracerHelpers.hlsli
```

- [ ] **Step 7: Compile-check** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`

- [ ] **Step 8: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT
git commit -m "refactor(rtxpt): extract PathTracer:: namespace + PathTracerHelpers; raygen camelCase locals"
```

---

## Task 7: Bridge namespace method renames

Apply table **T-F** to `PathTracerBridge.hlsli` and `MaterialBridge.hlsli` (the `Bridge::` methods), plus the macro `RTXPT_ENABLE_HIT_BRIDGE` → `ENABLE_HIT_BRIDGE` (table T-A). Update **all** `Bridge::` call sites across the closest-hit, any-hit, and `PathTracer.hlsli`.

**Files:**
- Modify: `PathTracer/PathTracerBridge.hlsli`, `PathTracer/Rendering/Materials/MaterialBridge.hlsli`
- Modify (call sites): `PathTracer/PathTracerClosestHit.rchit`, `PathTracer/PathTracerAnyHit.rahit`, `PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Confirm old symbols present**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rn "Bridge::Get\|Bridge::Has\|Bridge::Interpolate\|Bridge::Compute\|Bridge::Sample\|Bridge::AlphaTest\|RTXPT_ENABLE_HIT_BRIDGE\|RTXPT_SCENE_BRIDGE_HLSLI\|RTXPT_MATERIAL_BRIDGE_HLSLI" .
```

- [ ] **Step 2: Rename `Bridge::` method definitions**

In `PathTracerBridge.hlsli` (guard `RTXPT_SCENE_BRIDGE_HLSLI` → `__PATH_TRACER_BRIDGE_HLSLI__`) and `MaterialBridge.hlsli` (guard `RTXPT_MATERIAL_BRIDGE_HLSLI` → `__MATERIAL_BRIDGE_HLSLI__`), rename each `Bridge::` method per table **T-F** (PascalCase → camelCase). Re-case locals/params within each (e.g. `SubInstance→subInstance`, `LocalPrimitiveIndex→localPrimitiveIndex`, `Barycentrics→barycentrics`, `WorldNormal→worldNormal`, `Material→material`, `MaterialID→materialID`, `UV→uv`, `TextureIndex→textureIndex`, `Slice→slice`, `Index→index`, `Count→count`, `Stride→stride`, etc.). Macro `RTXPT_ENABLE_HIT_BRIDGE` → `ENABLE_HIT_BRIDGE` (the `#ifdef` and the `#define` in the chit/rahit).

> Keep the shared struct *type tokens* (`RTXPTSubInstanceData`, `RTXPTVertex`, `RTXPTMaterialData`) and the global resource names (`g_SubInstanceData`, `g_VertexBuffer`, `g_IndexBuffer`, `g_Materials`, `g_MaterialTextures`, `g_MaterialSampler`, `g_FrameConstants`) **unchanged** here — they are renamed atomically in Tasks 8/9. Also keep `kRTXPTSubInstanceFlagIndexed`/`kRTXPTMaterialFlag*` until Task 8/9.

- [ ] **Step 3: Update `Bridge::` call sites**

In `PathTracerClosestHit.rchit` and `PathTracerAnyHit.rahit`: replace each `Bridge::GetSubInstanceData()`, `Bridge::HasSubInstanceTable()`, `Bridge::GetMaterial(...)`, `Bridge::GetTriangleVertices(...)`, `Bridge::InterpolateTexCoord(...)`, `Bridge::ComputeGeometricNormal(...)`, `Bridge::InterpolateNormal(...)`, `Bridge::ComputeWorldHitPosition(...)`, `Bridge::ComputeWorldTangent(...)`, `Bridge::GetTangentNormal(...)`, `Bridge::GetMetallicRoughness(...)`, `Bridge::GetBaseColor(...)`, `Bridge::GetEmission(...)`, `Bridge::HasMaterialTable()`, `Bridge::AlphaTestPasses(...)` with their camelCase forms. Also `#define RTXPT_ENABLE_HIT_BRIDGE 1` → `#define ENABLE_HIT_BRIDGE 1`.

In `PathTracer.hlsli`: `Bridge::getLightCount()`/`Bridge::getLight(...)` were written in Task 6 already in their final camelCase if you followed the final-state listing; if you wrote them as `Bridge::GetLightCount`/`GetLight`, rename now.

- [ ] **Step 4: Verify total rename**
```bash
cd DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
grep -rnE "Bridge::(Get|Has|Interpolate|Compute|Sample|AlphaTest)" .
grep -rn "RTXPT_ENABLE_HIT_BRIDGE\|RTXPT_SCENE_BRIDGE_HLSLI\|RTXPT_MATERIAL_BRIDGE_HLSLI" .
```
Expected: **no output**.

- [ ] **Step 5: Compile-check** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`

- [ ] **Step 6: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT/assets/shaders
git commit -m "refactor(rtxpt): align Bridge:: method names + hit-bridge macro with RTXPT-fork"
```

---

## Task 8: Shared CPU/GPU struct + field alignment (HLSL **and** C++, atomic)

Apply tables **T-G** (type names) and **T-H** (field names) across the HLSL shared header **and every C++ mirror + populator + `static_assert`**, in one atomic change so the CPU/GPU contract never breaks. **Byte layout is unchanged** — only names change; the `sizeof(...) == 48 / 208 / …` asserts keep their numbers.

**Files:**
- Modify (HLSL): `PathTracer/PathTracerShared.h` (all six structs + the flag constants), `RTXPTCommon.fxh` (uses `RTXPTFrameConstants`), and every shader that names a shared type/field/flag (`PathTracerSample.rgen`, `PathTracer.hlsli`, `PathTracerBridge.hlsli`, `MaterialBridge.hlsli`, `PolymorphicLight.hlsli`, `PathTracerClosestHit.rchit`, `PathTracerAnyHit.rahit`, `PathTracerMiss.rmiss`).
- Modify (C++): `RTXPTSample.hpp` (struct defs + `static_assert`s), `RTXPTSample.cpp`, `RTXPTLights.{hpp,cpp}`, `RTXPTMaterials.{hpp,cpp}`, `RTXPTAccelerationStructures.{hpp,cpp}`, `RTXPTScene.{hpp,cpp}`.

- [ ] **Step 1: Inventory the blast radius (the "failing test")**
```bash
cd DiligentSamples/Samples/RTXPT
grep -rn "RTXPTSubInstanceData\|RTXPTPathTracerSettings\|RTXPTFrameConstants\|RTXPTPathTracerPayload\|RTXPTPrimaryPayload\|RTXPTMaterialData\|RTXPTLightData\|RTXPTVertex\|kRTXPTSubInstanceFlag\|kRTXPTMaterialFlag" src assets | wc -l
```
Record the count; it must reach **0** for the old tokens at Step 4.

- [ ] **Step 2: Rename the HLSL structs + fields + flags in `PathTracerShared.h`**

Apply tables **T-G** and **T-H** to the six structs in `PathTracer/PathTracerShared.h`, plus:
- guard `RTXPT_SHADER_SHARED_HLSLI` → `__PATH_TRACER_SHARED_H__`
- `kRTXPTSubInstanceFlagIndexed` → `kSubInstanceFlagIndexed`
- `kRTXPTMaterialFlagHasBaseColorTexture` → `kMaterialFlagHasBaseColorTexture` (and the other 4)
- update the doc-comments that name the C++ types (e.g. "Mirrors `Diligent::RTXPTSubInstanceData`" → "Mirrors `Diligent::SubInstanceData`")

Then propagate the **field-access** renames to every shader that reads them. Key sites:
- `g_FrameConstants.PathTracer.<field>` → `g_Const.ptConsts.<newField>` *(the `g_FrameConstants`→`g_Const` global rename is Task 9; here change `.PathTracer.`→`.ptConsts.` and the field names — i.e. `.PathTracer.MaxBounces` → `.ptConsts.bounceCount`, etc. Leave `g_FrameConstants` as-is until Task 9, OR do both now if you prefer — but keep the grep checks consistent.)*
- `g_FrameConstants.ViewProjInv` → `g_FrameConstants.viewProjInv`; `.CameraPosition_Time` → `.cameraPositionAndTime`; `.ViewportSize_FrameIdx` → `.viewportSizeAndFrameIndex`
- payload field reads/writes in chit/rahit/miss/rgen: `.WorldPos→.worldPos`, `.HitDistance→.hitDistance`, `.WorldNormal→.worldNormal`, `.HitFlag→.hitFlag`, `.BaseColor→.baseColor`, `.Metallic→.metallic`, `.Emission→.emission`, `.Roughness→.roughness`
- `SubInstanceData` field reads in bridge: `.FirstIndex→.IndexOffset`, `.FirstVertex→.VertexOffset` (others keep names)
- `MaterialPTData` field reads in `MaterialBridge.hlsli`: per table T-H (`.BaseColorFactor→.baseColorFactor`, `.Flags→.flags`, `.AlphaCutoff→.alphaCutoff`, etc.)
- `PolymorphicLightInfo` field reads in `PolymorphicLight.hlsli`: `.ColorIntensity→.colorIntensity`, `.PositionRange→.positionRange`, `.DirectionType→.directionType`, `.SpotAngles→.spotAngles`
- `GeometryVertexData` field reads in bridge: `.Position→.position`, `.Normal→.normal`, `.TexCoord0→.texCoord0`
- the HLSL `ConstantBuffer<RTXPTFrameConstants> g_FrameConstants;` declarations (in `PathTracerBridge.hlsli` and `RTXPTCommon.fxh`) → `ConstantBuffer<SampleConstants> g_FrameConstants;`
- `StructuredBuffer<RTXPTSubInstanceData>` → `StructuredBuffer<SubInstanceData>`; `StructuredBuffer<RTXPTLightData>` → `StructuredBuffer<PolymorphicLightInfo>`; `StructuredBuffer<RTXPTVertex>` → `StructuredBuffer<GeometryVertexData>`; `StructuredBuffer<RTXPTMaterialData>` → `StructuredBuffer<MaterialPTData>`; `RTXPTPathTracerPayload` → `PathPayload`; `RTXPTPrimaryPayload` → `PrimaryPayload`

- [ ] **Step 3: Rename the C++ structs + fields + flags in lockstep**

In `RTXPTSample.hpp`:
- `struct RTXPTPathTracerSettings` → `struct PathTracerConstants`; rename its fields per table **T-H**; update `static_assert(sizeof(RTXPTPathTracerSettings) == 48, …)` → `static_assert(sizeof(PathTracerConstants) == 48, …)` (keep `48`)
- `struct RTXPTFrameConstants` → `struct SampleConstants`; fields per T-H; member `RTXPTPathTracerSettings PathTracer;` → `PathTracerConstants ptConsts;`; `static_assert(sizeof(RTXPTFrameConstants) == 208, …)` → `static_assert(sizeof(SampleConstants) == 208, …)` (keep `208`)
- the `m_LastFrameConstants` member type `RTXPTFrameConstants` → `SampleConstants`

In `RTXPTAccelerationStructures.hpp/.cpp`: `struct RTXPTSubInstanceData` → `struct SubInstanceData`; fields `FirstIndex→IndexOffset`, `FirstVertex→VertexOffset`, `Padding0/1→_padding0/1`; `kRTXPTSubInstanceFlag_Indexed` → `kSubInstanceFlag_Indexed`. Update every populator assignment (`.FirstIndex =` → `.IndexOffset =`, etc.).

In `RTXPTMaterials.hpp/.cpp`: `struct RTXPTMaterialData` → `struct MaterialPTData`; fields per T-H; `kRTXPTMaterialFlag_*` → `kMaterialFlag_*`. Update all populator assignments.

In `RTXPTLights.hpp/.cpp`: `struct RTXPTLightData` → `struct PolymorphicLightInfo`; fields per T-H. Update populators.

In `RTXPTScene.hpp/.cpp`: `struct RTXPTVertex` → `struct GeometryVertexData`; fields per T-H. Update populators / `sizeof` uses (the vertex stride check — keep the value, rename the type).

In `RTXPTSample.cpp`: `UpdateFrameConstants` writes `m_LastFrameConstants.PathTracer.<field>` → `m_LastFrameConstants.ptConsts.<newField>` and the `SampleConstants` top-level fields (`viewProj`/`viewProjInv`/`cameraPositionAndTime`/`viewportSizeAndFrameIndex`). Update every reference.

> **Layout discipline:** do not reorder fields, change types, or add/remove padding. Only the *names* change. After editing, the `static_assert` values must be untouched and must still compile.

- [ ] **Step 4: Verify total rename**
```bash
cd DiligentSamples/Samples/RTXPT
grep -rn "RTXPTSubInstanceData\|RTXPTPathTracerSettings\|RTXPTFrameConstants\|RTXPTPathTracerPayload\|RTXPTPrimaryPayload\|RTXPTMaterialData\|RTXPTLightData\|RTXPTVertex\|kRTXPTSubInstanceFlag\|kRTXPTMaterialFlag" src assets
```
Expected: **no output**. Then confirm the new types appear in both languages:
```bash
grep -rln "SubInstanceData\|PathTracerConstants\|SampleConstants\|PathPayload\|MaterialPTData\|PolymorphicLightInfo\|GeometryVertexData" src assets | sort -u
```
Expected: both `src/*.{hpp,cpp}` and `assets/shaders/PathTracer/*` files listed.

- [ ] **Step 5: Compile-check both backends + verify `static_assert`s hold** *(user-initiated)*

Run *(user-initiated)*: `cmake --build build/x64/Debug --config Debug`
Expected: builds; **no** `static_assert` failure (sizes unchanged). If a `static_assert` fires, a field type/order/padding changed accidentally — revert that field and re-apply as a pure name change.

- [ ] **Step 6: clang-format the touched C++** *(user-initiated)*

Run *(user-initiated)*: `cmake --build build/x64/Debug --target DiligentSamples-ValidateFormatting` (or `DiligentSamples/BuildTools/FormatValidation/validate_format_win.bat`).
Expected: pass. Field renames may shift column alignment; re-run clang-format on the edited C++ files if it flags alignment.

- [ ] **Step 7: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT
git commit -m "refactor(rtxpt): align shared CPU/GPU struct + field names with RTXPT-fork (layout unchanged)"
```

---

## Task 9: Resource-binding variable + Config.h macro alignment

Apply table **T-I** (HLSL globals + C++ binding strings) and the remaining macro rows of **T-A** (`RTXPT_ENABLE_MATERIAL_TEXTURES`, `RTXPT_MATERIAL_TEXTURE_COUNT`), and create `PathTracer/Config.h` to hold the path-tracer compile macros (RTXPT-fork keeps them in `Config.h`).

**Files:**
- Create: `PathTracer/Config.h`
- Modify (HLSL globals): `PathTracer/PathTracerSample.rgen`, `PathTracer/PathTracer.hlsli`, `PathTracer/PathTracerBridge.hlsli`, `PathTracer/Rendering/Materials/MaterialBridge.hlsli`, `RTXPTCommon.fxh`
- Modify (C++ binding strings): `RTXPTRayTracingPass.cpp`
- Modify (CMake): `CMakeLists.txt` (register `Config.h`)

- [ ] **Step 1: Confirm old globals/macros present**
```bash
cd DiligentSamples/Samples/RTXPT
grep -rn "g_FrameConstants\|g_TLAS\|g_OutputColor\|g_AccumColor\|g_Lights\|g_SubInstanceData\|g_VertexBuffer\|g_IndexBuffer\|g_Materials\|g_MaterialTextures\|g_MaterialSampler\|RTXPT_ENABLE_MATERIAL_TEXTURES\|RTXPT_MATERIAL_TEXTURE_COUNT" src assets
```

- [ ] **Step 2: Rename HLSL global resource declarations + every use**

Apply table **T-I** to the HLSL declarations and every reference:
- `g_FrameConstants` → `g_Const` (declarations in `PathTracerBridge.hlsli` + `RTXPTCommon.fxh`; uses in rgen/PathTracer.hlsli/compute)
- `g_TLAS` → `t_SceneBVH` (rgen decl + `TraceRay` call sites in rgen + `PathTracer.hlsli`)
- `g_OutputColor` (rgen) → `u_Output`; `g_AccumColor` → `u_AccumulationBuffer`
- `g_Lights` → `t_Lights`; `g_SubInstanceData` → `t_SubInstanceData`; `g_VertexBuffer` → `t_VertexBuffer`; `g_IndexBuffer` → `t_IndexBuffer`; `g_Materials` → `t_PTMaterialData`; `g_MaterialTextures` → `t_BindlessTextures`; `g_MaterialSampler` → `s_MaterialSampler`

- [ ] **Step 3: Rename C++ binding strings in `RTXPTRayTracingPass.cpp`**

Every string literal passed to `AddVariable`, `AddImmutableSampler`, `GetStaticVariableByName`, `GetVariableByName`, and `SetStatic` must match the new HLSL names. Apply the same map as Step 2 to the literals (e.g. `.AddVariable(SHADER_TYPE_RAY_GEN, "g_FrameConstants", …)` → `"g_Const"`, `"g_TLAS"`→`"t_SceneBVH"`, `"g_OutputColor"`→`"u_Output"`, `"g_AccumColor"`→`"u_AccumulationBuffer"`, `"g_Lights"`→`"t_Lights"`, `"g_SubInstanceData"`→`"t_SubInstanceData"`, `"g_VertexBuffer"`→`"t_VertexBuffer"`, `"g_IndexBuffer"`→`"t_IndexBuffer"`, `"g_Materials"`→`"t_PTMaterialData"`, `"g_MaterialTextures"`→`"t_BindlessTextures"`, `"g_MaterialSampler"`→`"s_MaterialSampler"`). The `BindRayGenShader("Main")`/`AddGeneralShader("Main"/"PrimaryMiss")`/`AddTriangleHitShader("PrimaryHit")`/`BindMissShader`/`BindHitGroupForTLAS` group names are PSO record names, **not** resource bindings — leave them (or align separately; out of scope).

- [ ] **Step 4: Create `PathTracer/Config.h` and move the texture macros**

```hlsl
#ifndef __CONFIG_H__
#define __CONFIG_H__

// Path-tracer compile-time configuration (RTXPT-fork keeps these in PathTracer/Config.h).
// ENABLE_MATERIAL_TEXTURES and MATERIAL_TEXTURE_COUNT are supplied by the C++ side
// (RTXPTRayTracingPass) via ShaderMacroHelper when the bindless material-texture table exists.

#endif // __CONFIG_H__
```
Have `MaterialBridge.hlsli` `#include "../../Config.h"` (or root-relative) at the top, and change `#ifdef RTXPT_ENABLE_MATERIAL_TEXTURES` → `#ifdef ENABLE_MATERIAL_TEXTURES` and `RTXPT_MATERIAL_TEXTURE_COUNT` → `MATERIAL_TEXTURE_COUNT` in the declaration `Texture2DArray g_MaterialTextures[...]` (now `t_BindlessTextures`). In `RTXPTRayTracingPass.cpp`, change `Macros.Add("RTXPT_ENABLE_MATERIAL_TEXTURES", 1)` → `Macros.Add("ENABLE_MATERIAL_TEXTURES", 1)` and `Macros.Add("RTXPT_MATERIAL_TEXTURE_COUNT", …)` → `Macros.Add("MATERIAL_TEXTURE_COUNT", …)`.

Register in `CMakeLists.txt` `SHADERS`:
```cmake
    assets/shaders/PathTracer/Config.h
```

- [ ] **Step 5: (optional) Debug-compute/blit resource names**

Grep `RTXPTComputePass.cpp` / `RTXPTBlitPass.cpp` for `g_InputColor`/`g_OutputColor` binding strings. Rename to `t_InputColor`/`u_Output` in the `.csh`/`.psh` **and** the matching C++ strings **only if** present; otherwise leave (documented divergence — these are not path-tracer algorithm-layer shaders).

- [ ] **Step 6: Verify total rename**
```bash
cd DiligentSamples/Samples/RTXPT
grep -rn "g_FrameConstants\|g_TLAS\|g_OutputColor\|g_AccumColor\|g_Lights\|g_SubInstanceData\|g_VertexBuffer\|g_IndexBuffer\|g_Materials\b\|g_MaterialTextures\|g_MaterialSampler\|RTXPT_ENABLE_MATERIAL_TEXTURES\|RTXPT_MATERIAL_TEXTURE_COUNT" src assets/shaders/PathTracer RTXPTCommon.fxh 2>/dev/null
```
Expected: **no output** (path-tracer scope). Any remaining `g_OutputColor` in the *debug-compute* shader is acceptable if Step 5 was skipped.

- [ ] **Step 7: Compile-check both backends** *(user-initiated)* — `cmake --build build/x64/Debug --config Debug`. The PSO resource-layout binding-by-name must still resolve (HLSL globals ↔ C++ strings now match).

- [ ] **Step 8: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT
git commit -m "refactor(rtxpt): align resource-binding names (t_/u_/s_/g_) + texture macros + Config.h with RTXPT-fork"
```

---

## Task 10: Finalize mapping doc + byte-identical verification + wrap-up

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Final grep sweep — no stale `RTXPT…` algorithm-layer tokens**
```bash
cd DiligentSamples/Samples/RTXPT
# Allowed remaining RTXPT* names: pass/class names (RTXPTRayTracingPass…), file names (RTXPTSample.cpp…),
# RTXPTCommon.fxh, RTXPTDebugCompute/RTXPTBlit shaders. Algorithm-layer symbols must be gone:
grep -rn "RTXPTSurface\|RTXPTSampleBSDF\|RTXPTEvalBSDF\|RTXPTEvalSky\|RTXPTEvalAnalyticLight\|RTXPTRandom\|RTXPTLightSample\|RTXPTMakeSurface\|RTXPT_PI\|RTXPT_INV_PI\|RTXPTPowerHeuristic\|RTXPTSubInstanceData\|RTXPTMaterialData\|RTXPTLightData\|RTXPTVertex\|RTXPTPathTracerSettings\|RTXPTFrameConstants\|RTXPTPathTracerPayload" src assets/shaders/PathTracer
```
Expected: **no output**.

- [ ] **Step 2: Finalize the Divergences section of the mapping doc**

Update `RTXPT_FORK_MAPPING.md`'s Divergences section to its final form, ensuring each "(divergent)" / "(style)" entry from tables T-A…T-I is reflected, plus:
- the consolidation of Fresnel/Microfacet helpers into `BxDF.hlsli` (RTXPT-fork separates them),
- `EnvMap::Eval` returns a procedural gradient (HDR map deferred to Phase R4),
- `Bridge::` runs over Diligent structured buffers, not Donut/NVRHI,
- `t_PTMaterialData`/`PolymorphicLightInfo`/`GeometryVertexData`/`PathPayload`/`PathTracerConstants`/`SampleConstants` reuse RTXPT-fork *names* with port-specific *layouts*,
- resource names follow RTXPT-fork `t_/u_/s_/g_` prefixes but the set is the reference-mode subset,
- debug-compute/blit shaders (and `RTXPTCommon.fxh`) keep `RTXPT`-prefixed names — outside the algorithm layer.

- [ ] **Step 3: Byte-identical render verification** *(user-initiated)*

Ask the user to rebuild and run on **both** backends, capture the first post-reset frame, and compare to the Task-0 baselines:
- D3D12: new capture vs `baseline_d3d12.png` → must be identical (or visually indistinguishable if exact-pixel capture isn't available).
- Vulkan: new capture vs `baseline_vk.png` → same.

If any difference appears, it is a refactor bug (not an intended change). Bisect by `git bisect` over the per-task commits, or re-diff the offending layer's task for a non-rename edit (changed literal/order/field-layout). Fix and re-verify.

- [ ] **Step 4: Final clang-format validation of all touched C++** *(user-initiated)*

Run *(user-initiated)*: `DiligentSamples/BuildTools/FormatValidation/validate_format_win.bat`
Expected: pass for `RTXPTSample.{hpp,cpp}`, `RTXPTRayTracingPass.cpp`, `RTXPTLights.*`, `RTXPTMaterials.*`, `RTXPTAccelerationStructures.*`, `RTXPTScene.*`.

- [ ] **Step 5: Commit**
```bash
git add -A DiligentSamples/Samples/RTXPT
git commit -m "docs(rtxpt): finalize R0.5 RTXPT-fork mapping divergences; close style/naming alignment"
```

---

## Self-Review

**Spec coverage (goal G0.5 / Phase R0.5 "Touches" list):**
- "all ported HLSL files under `assets/shaders/` (renames + reorg, likely into a `PathTracer/`-style subfolder)" → Tasks 2–7, 9. ✅
- "the shared HLSL/C++ struct headers" → Task 8. ✅
- "`CMakeLists.txt` (shader registration)" → Task 2 Step 4, Task 6 Step 6, Task 9 Step 4. ✅
- "the C++ that references renamed shader entry points/structs" → Task 2 (FilePath/factory), Task 8 (structs), Task 9 (binding strings + macros). ✅ (Entry-point names `main` are unchanged — Diligent convention; PSO group names left intact, noted in Task 9.)
- "A new mapping doc (RTXPT-fork ↔ port symbol/file correspondence + divergences)" → Task 1 (create) + Task 10 (finalize). ✅
- Success criterion "adopts RTXPT-fork symbol names, namespace structure (`PathTracer::`, `Bridge::`), file/folder organization, and macro names" → namespaces (Tasks 5/6/7), folders (Task 2), symbols (3–9), macros (T-A, Tasks 4/7/9). ✅
- "byte-identically on D3D12 + Vulkan" → verification model + Task 10 Step 3. ✅
- "clang-format validation passes" → Task 8 Step 6 + Task 10 Step 4 (C++ only; shaders not validated — noted). ✅
- Hard constraints (no NVIDIA headers; keep Diligent header on C++; C++ glue keeps Diligent conventions; only GPU-facing structs + bindings align) → Naming-rule section + Task scoping. ✅
- Cross-cutting contracts (settings `static_assert(==48)`, payload `MaxPayloadSize`, light-buffer dummy-light invariant, hit-group/SBT reuse, `MaxRecursionDepth=1`) → Task 8 keeps the asserts/values; no payload-size or SBT change is made (rename-only), so those contracts are untouched. ✅

**Placeholder scan:** every rename has concrete old→new strings in tables T-A…T-I; the two highest-risk artifacts (raygen, PathTracer.hlsli, PathTracerHelpers.hlsli, EnvMap.hlsli, Config.h) are given in full; verification steps are concrete `grep`/build commands. No "TBD"/"handle edge cases"/"similar to". ✅

**Type/name consistency:** the type tokens introduced in Task 8 (`StandardBSDFData`, `SampleConstants`/`g_Const`/`ptConsts`, `PathPayload`, `SubInstanceData`, `MaterialPTData`, `PolymorphicLightInfo`, `GeometryVertexData`) are used identically wherever they appear in later code blocks (the Task 6 listings show the *final* names and a note tells the executor to defer the type/global/field tokens to Tasks 8/9 — the "type-token discipline" callout in Task 5 makes this ordering explicit and consistent). Function names (`EvalBSDF`/`SampleBSDF`/`getSpecularProbability`/`evalNdfGGX`/`evalFresnelSchlick`/`evalVisibilitySmithGGXCorrelated`/`PowerHeuristic`/`sampleNext1D`/`sampleCosineHemisphere`/`BranchlessONB`/`EnvMap::Eval`/`Bridge::get*`) match between their definition task and call-site tasks. ✅

**Ordering risk:** because we can't auto-build between tasks, each task is internally self-consistent (renames a symbol *and* all its call sites together). The one subtlety — shared *type tokens*, *globals*, and *shared-struct fields* are deferred to Tasks 8/9 so HLSL+C++ change atomically — is called out in Task 5 ("type-token discipline") and repeated in Tasks 6/7/8 notes. The Task 6 full listings intentionally show post-all-tasks names with an explicit instruction to write current-names-then-let-later-tasks-rename; grep gates in Tasks 8/9 catch any token written early or missed. ✅

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-30-rtxpt-phase-r0-5-style-naming-alignment.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Given this refactor's byte-identical invariant and large blast radius (especially Tasks 6 and 8), per-task review of the `git diff` is valuable.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints for review.

Which approach?
