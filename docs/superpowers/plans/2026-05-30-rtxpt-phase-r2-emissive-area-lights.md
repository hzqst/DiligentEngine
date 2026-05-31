# RTXPT Phase R2 — Emissive-Triangle Area Lights Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the DiligentEngine RTXPT reference path tracer emissive-mesh area lighting (G4) from the current GPU geometry: build emissive triangles from the live GPU vertex/index buffers, sample them with next-event estimation (area→solid-angle pdf + shadow ray), and MIS-weight emissive BSDF hits against that estimator so static emissive-mesh-lit scenes converge dramatically faster while the converged image is unchanged. Animated/skinned glTF support already exists in the sample's current-geometry + dynamic BLAS path, so the emissive work must stay aligned with that path and never fall back to bind-pose geometry.

**Architecture:** R2 implements the static glTF path first: a GPU emissive-triangle build pass runs once after static BLAS/current static geometry is available. The pass reads the current vertex/index buffers and writes a `StructuredBuffer<EmissiveTriangle>` of world-space triangles (base + two edges + radiance) for every NEE-eligible emitter (constant emission, non-degenerate); the CPU only provides topology/count metadata and never reads mesh geometry back from the GPU. For animated/skinned glTF, the existing current-geometry pipeline already produces the current-frame skinned vertex buffer and updates the skinned BLAS from that same buffer, so the emissive builder must consume that same frame data and stay in lockstep with it. The raygen loop gains an emissive-NEE estimator (uniform triangle selection — RIS/WRS is Phase R3) that mirrors RTXPT-fork `TriangleLight::CalcSample`, and the emissive BSDF-hit emission is MIS-weighted (power heuristic) against it. To partner the two estimators, the closest-hit shader precomputes the hit triangle's area-light solid-angle pdf into a new (80-byte) payload field; the raygen multiplies in the uniform selection pdf. Emissive NEE is toggleable (and disabled by uploading a zero triangle count), so unbiasedness is directly verifiable.

**Tech Stack:** HLSL (DXC, ray-tracing pipeline) under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/`; C++ (Diligent sample framework, GLTF loader, Dear ImGui) under `DiligentSamples/Samples/RTXPT/src/`. No automated shader-test harness exists; correctness guards are compile-time `static_assert`s plus documented manual GPU verification (run on explicit user request only, per the workspace rule).

---

## Context You Need Before Starting

**Phases R0, R0.5, and R1 have already landed.** R0.5 renamed/reorganized the reference shaders into a `PathTracer/`-style tree with RTXPT-fork-aligned names; R1 added the firefly filter, NEE-at-all-bounces, and stateless seeding. The spec's "Touches" list for R2 uses pre-R0.5 names; the **current** equivalents this plan edits are:

| Spec name (pre-R0.5) | Current path (edit these) |
|---|---|
| `RTXPTLights` / `RTXPTScene` | `DiligentSamples/Samples/RTXPT/src/RTXPTLights.{hpp,cpp}` (emissive extraction lands here) |
| `RTXPTSceneBridge.hlsli` (light-list access) | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli` |
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTReference.rchit` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` |
| shared HLSL/C++ structs | `.../PathTracer/PathTracerShared.h` (HLSL) and `src/RTXPTSample.hpp` (C++ mirror) |
| NEE / light sampling math | `.../PathTracer/PathTracer.hlsli`, `.../PathTracer/Lighting/PolymorphicLight.hlsli`, `.../PathTracer/PathTracerHelpers.hlsli` |
| RT-pass bindings | `src/RTXPTRayTracingPass.{hpp,cpp}` |
| settings / UI | `src/RTXPTSample.{hpp,cpp}` |

**The submodule.** `DiligentSamples` is a git submodule on branch `RTXPT`. All edits below are inside the submodule. Commit **inside** `DiligentSamples/` (its working tree), not the umbrella repo. The umbrella repo only tracks the submodule pointer; do not bump it as part of these task commits unless asked.

**RTXPT-fork reference anchors** (read-only, for cross-checking the math, under `D:/RTXPT-fork/Rtxpt/Shaders/`):
- `PathTracer/Lighting/PolymorphicLight.hlsli:399-521` — `TriangleLight` (`CalcSample`, `CalcSolidAnglePdfForMIS`, `Create`); `:25` `MAX_SOLID_ANGLE_PDF (1e10)`.
- `PathTracer/Utils/Geometry.hlsli:33-41` — `SampleTriangleUniform`; `:79-82` — `pdfAtoW`.
- `PathTracer/PathTracer.hlsli:592-634` — emissive BSDF-hit MIS (`ComputeBSDFMISForEmissiveTriangle`).
- `PathTracer/Lighting/LightSampler.hlsli:318-345` — `ComputeLightVsBSDF_MIS_ForBSDF` / `ComputeBSDFMISForEmissiveTriangle`.
- `PathTracer/PathTracerNEE.hlsli:166-275` — visibility-ray origin/shortening + light-sample processing + NEE firefly dampening.

### Design decisions baked into this plan (read before editing)

1. **Current GPU geometry is the source of truth.** Do **not** read vertex/index buffers back to the CPU. `RTXPTLights` only scans glTF topology/materials to determine the fixed emissive-triangle count and allocate/bind the output buffer; the world-space `EmissiveTriangle` records are generated on the GPU from the same current vertex/index buffers used by ray tracing. For static glTF this is the existing static path-tracer vertex stream. For skinned glTF this means the emissive builder must consume the GPU-produced post-skinning vertex buffer already used by ray tracing, not bind-pose data.

2. **Skinned meshes reuse the existing GPU skinning + dynamic BLAS path.** Static meshes keep the existing static BLAS path. Any mesh whose vertices are changed by GPU skinning must rebuild/refit its BLAS from the current skinned vertex buffer before the emissive-triangle build pass and before ray dispatch. The emissive builder, closest-hit shader, and BLAS must see the same frame's geometry; otherwise visibility and MIS pdfs diverge. The current code already provides that prerequisite, so this plan treats it as an input contract rather than a future blocker.

3. **NEE-eligible = constant emitter, decided by one shared predicate.** A triangle becomes an emissive light only if its material has a non-zero `EmissiveFactor` **and references no emissive texture** (`GetTextureId(emissive) < 0`). For such emitters the shader's `Bridge::getEmission` returns exactly `emissiveFactor` (see `MaterialBridge.hlsli:58-64`), so the NEE radiance equals the BSDF-hit emission for the same triangle — required for the unbiasedness invariant. **Both** the GPU emissive-triangle build pass and the GPU BSDF-hit MIS gate (closest-hit) must use the *identical* eligibility test, or a mismatch biases the image. To guarantee that, a single helper `RTXPTMaterialIsEmissiveAreaLight(const GLTF::Material&)` (mirroring the existing `RTXPTMaterialIsAlphaTested`) is the sole source of truth: `RTXPTMaterials` uses it to set a new GPU material flag `kMaterialFlag_EmissiveAreaLight` (bit `0x20`), and `RTXPTLights` uses it when sizing/allocating the emissive triangle buffer. The closest-hit and emissive builder read that flag (not a re-derived condition). Textured-emissive triangles stay BSDF-only (no NEE, full-weight emission) and are deferred behind a TODO. The predicate depends only on the glTF material (not on texture-load success), so the two sides cannot drift.

4. **Two-sided emitters.** Emissive triangles emit from **both** faces (`cosTheta = abs(dot(normal, -dir))`), applied identically in the NEE estimator and the BSDF-hit MIS. This preserves the pre-R2 two-sided emissive look (the current chit adds material emission regardless of facing) and stays unbiased. It is a deliberate divergence from RTXPT-fork's one-sided `TriangleLight`; record it in `RTXPT_FORK_MAPPING.md` (Task 7) with a TODO to align one-sided + double-sided baker semantics later.

5. **Uniform selection now; RIS later.** Light selection is uniform over the emissive triangle list (`selectionPdf = 1/count`); RTXPT's RIS/WRS importance sampling is Phase R3 (G5). The plan is structured so R3 swaps only the selection step.

6. **Toggle = zero count.** The "Emissive mesh NEE + MIS" UI toggle (and the master NEE toggle) gate the feature by uploading `emissiveTriangleCount = 0` to the frame constants when off — the same disable trick R1 used for the firefly threshold. With count 0, the NEE estimator returns nothing and the BSDF-hit MIS weight collapses to 1 (full-weight emission), so toggling converges to the same image.

7. **No CPU `LightsBaker` port or per-triangle light-index table.** RTXPT-fork carries a precomputed `neeTriangleLightIndex` per triangle (from its `LightsBaker`) and re-derives the light's solid-angle pdf in `ComputeBSDFMISForEmissiveTriangle`. The port avoids that table for R2: the GPU builder writes a dense `EmissiveTriangle` buffer, selection is uniform, and the only light-specific quantity the BSDF-hit MIS needs is the hit triangle's solid-angle pdf, which the closest-hit computes directly from the current triangle geometry it already fetches (area + normal + `RayTCurrent()`) and writes into `payload.emissiveLightPdf`. The raygen multiplies in the uniform selection pdf (`1/count`). This is equivalent for uniform selection and is the natural place to reintroduce a proxy/index table when R3 adds RIS.

8. **Static scenes build once; dynamic scenes rebuild on dirty using the existing skinned path.** A pure static glTF scene builds its BLAS and emissive-triangle buffer once at load, then stays resident. Dynamic/skinned scenes already use the current-geometry update loop and dynamic BLAS rebuild/refit path, so the emissive plan must never assume bind-pose data. No per-frame rebuild happens for static content.

## File Structure (what each task touches)

- `PathTracer/PathTracerShared.h` — GPU-shared structs/constants. Task 1 adds `kMaxSolidAnglePdf`, the `EmissiveTriangle` struct, and renames the settings `_padding1` → `emissiveTriangleCount` (size stays 48). Task 4 grows `PathPayload` by one float4 (→ 80 bytes).
- `src/RTXPTAccelerationStructures.{hpp,cpp}` — the skinned/dynamic BLAS update path that keeps ray tracing in sync with the same current-frame geometry the emissive builder reads.
- `src/RTXPTLights.{hpp,cpp}` — emissive-triangle buffer ownership, fixed topology count, and stats. Tasks 1 (struct mirror), 2 (no readback).
- `src/RTXPTEmissiveTrianglePass.{hpp,cpp}` — compute pass that fills `StructuredBuffer<EmissiveTriangle>` from the current GPU vertex/index buffers. Task 2.
- `PathTracer/EmissiveTriangleBuild.hlsl` — compute shader for the GPU emissive-triangle build pass. Task 2.
- `src/RTXPTMaterials.{hpp,cpp}` — the shared `RTXPTMaterialIsEmissiveAreaLight` helper + `kMaterialFlag_EmissiveAreaLight` (bit `0x20`), set per material. Task 1.
- `src/RTXPTSample.{hpp,cpp}` — settings mirror, `m_EnableEmissiveNEE`, static-vs-dynamic emissive build scheduling, per-frame count upload, Scene stat, UI toggle. Tasks 1, 2, 3.
- `PathTracer/Lighting/PolymorphicLight.hlsli` — `pdfAtoW`, `SampleTriangleUniform` helpers. Task 5.
- `PathTracer/PathTracerBridge.hlsli` — `t_EmissiveTriangles` buffer + `Bridge::getEmissiveTriangle*` accessors. Task 5.
- `PathTracer/Utils/SampleGenerators.hlsli` — `kSampleEffect_NEEEmissive` salt. Task 5.
- `PathTracer/PathTracerClosestHit.rchit` — precomputes the emissive solid-angle pdf into the payload. Task 4.
- `PathTracer/PathTracer.hlsli` — `MakeEmptyPayload` init (Task 4), `PathTracer::SampleEmissiveNEE` (Task 6).
- `PathTracer/PathTracerSample.rgen` — emissive NEE call + emissive BSDF-hit MIS + `prevDidEmissiveNEE` (Task 6).
- `src/RTXPTRayTracingPass.{hpp,cpp}` — `MaxPayloadSize` bump (Task 4) + `t_EmissiveTriangles` binding (Task 6).
- `RTXPT_FORK_MAPPING.md` — mapping rows + divergence notes; TODO marker retargeting (Task 7).

## Cross-Cutting Contracts — keep in lockstep

- **Settings layout.** `PathTracerConstants` is mirrored in C++ (`RTXPTSample.hpp`, `static_assert(sizeof==48)`) and HLSL (`PathTracerShared.h`), embedded in `SampleConstants` (`static_assert(sizeof==208)`). Task 1 grows it by **reusing the existing `_padding1` word** (`uint`/`Uint32`, 4 bytes) → `emissiveTriangleCount`, so **both byte sizes stay 48 / 208 and both `static_assert`s are untouched**. Field order/offsets stay identical across the two files.
- **Payload size.** `PathPayload` is currently 64 bytes (16 floats), all used; `MaxPayloadSize` is set in `RTXPTRayTracingPass::Initialize`. Task 4 adds the per-hit `emissiveLightPdf` the closest-hit must return, growing the payload to **80 bytes (20 floats)** — `MaxPayloadSize`, the struct, and the comment move together (there is no C++ `static_assert` on payload size; the comment is the contract). The firefly K and other per-path state stay raygen-local and do **not** enter the payload.
- **Geometry contract.** The emissive-triangle build pass, closest-hit shader, and BLAS build/update path must all consume the same current-frame vertex/index buffers. For static meshes this is the original path-tracer vertex stream. For skinned meshes this is the GPU-produced post-skinning buffer already used by the dynamic BLAS path. Do not mix bind-pose emissive triangles with skinned visibility geometry.
- **Dynamic BLAS contract.** Skinned geometry uses the updateable/rebuilt BLAS path already present in `RTXPTAccelerationStructures` (`RAYTRACING_BUILD_AS_ALLOW_UPDATE` when refitting is used, or a full rebuild from the current skinned buffer). The emissive build pass runs only after that current geometry is ready and before ray dispatch. Static geometry stays on the existing static path.
- **Rebuild-frequency contract.** Static GLTF content gets one BLAS build and one emissive-triangle build at load, then stays immutable until the scene topology changes. Dynamic/skinned content flips a dirty bit through the existing animation/skinning path, then re-runs dynamic BLAS plus any emissive-triangle refresh that consumes that same current-frame geometry. Never rebuild from bind-pose data.
- **Light-buffer contract.** `RTXPTLights` already uploads `StructuredBuffer<PolymorphicLightInfo> t_Lights` + an `AnalyticLightCount`, with a dummy entry for binding safety. R2 adds a parallel `StructuredBuffer<EmissiveTriangle> t_EmissiveTriangles` + `EmissiveTriangleCount`, also with a dummy entry. The buffer is created with SRV/UAV usage: the compute builder writes it before tracing, and raygen reads it as `t_EmissiveTriangles`. The raygen binding remains **STATIC to `SHADER_TYPE_RAY_GEN` only** (only the raygen references it; the closest-hit computes its pdf from geometry it already fetches), preserving the per-stage STATIC binding rule.
- **Material-flag contract.** `MaterialPTData::flags` gains one bit, `kMaterialFlag_EmissiveAreaLight = 0x20` (C++) / `kMaterialFlagEmissiveAreaLight` (HLSL). It is a new bit only — `sizeof(MaterialPTData)` stays 96 and the existing `static_assert`s are untouched. The bit is the single GPU signal the closest-hit and emissive builder use to decide emissive-NEE eligibility, set from the shared `RTXPTMaterialIsEmissiveAreaLight` helper that also drives the topology count.
- **Hit group / SBT reuse.** No new ray type. Shadow/visibility rays keep reusing hit group 0 + miss 0 with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`; `MaxRecursionDepth` stays 1.
- **Backends.** D3D12 and Vulkan stay first-class; the dynamic BLAS update, compute build pass, and new binding use backend-agnostic Diligent APIs.

## Verification Note (read once)

There is no shader unit-test runner. Per the workspace rule (global `CLAUDE.md`) and the spec's verification strategy, **do not auto-run build/test/run commands** — list them so the user can run them on request. Each task's "Verify" steps are: (a) the compile-time `static_assert`s the C++ build enforces, and (b) a manual GPU check the user runs when they choose. The phase's primary acceptance test is **unbiasedness**: toggling **Emissive mesh NEE + MIS** off must converge the accumulation image to the same result as with it on (only per-sample noise differs), and the MIS estimator weights for any shared direction sum to ≈ 1 (no double-counting). The secondary observable is convergence speed: an emissive-mesh-lit view should resolve in far fewer samples with the estimator on.

Manual GPU check (run only on explicit user request), per project `CLAUDE.md`:
```
cmake --build build\x64\Debug --config Debug
build\x64\Debug\...\RTXPT.exe          # D3D12 (default)
# and the Vulkan device variant, per the sample's device-selection UI/flags
```

---

### Task 1: Shared `EmissiveTriangle` struct + `emissiveTriangleCount` setting + emissive-area-light material flag (CPU + GPU), inert

Adds the GPU/CPU mirror of the emissive triangle record, the solid-angle clamp, the shared eligibility helper + material flag, and repurposes the settings padding word to carry the triangle count. Nothing consumes them yet — keeps the struct/layout change isolated and the build green.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`

- [ ] **Step 1: Add the solid-angle clamp constant near the top of the shared header**

In `PathTracerShared.h`, just after the existing `kSubInstanceFlagIndexed` constant (currently lines 4-5):

```hlsl
// Mirrors Diligent::kSubInstanceFlag_Indexed in RTXPTAccelerationStructures.hpp.
static const uint kSubInstanceFlagIndexed = 0x1u;
```

add:

```hlsl

// Maximum area-light solid-angle pdf (G4). Mirrors RTXPT-fork PolymorphicLight.hlsli:25 MAX_SOLID_ANGLE_PDF;
// clamps the area->solid-angle conversion for near-grazing / very close emissive-triangle samples.
static const float kMaxSolidAnglePdf = 1e10;
```

- [ ] **Step 2: Rename the settings padding word to `emissiveTriangleCount` (HLSL)**

In `PathTracerShared.h`, `PathTracerConstants` currently ends:

```hlsl
    uint  maxNEEBounceCount;      // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    uint  analyticLightCount;     // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float fireflyFilterThreshold; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter entirely.
    uint  _padding1;
};
```

Replace the `_padding1` line so it becomes:

```hlsl
    uint  maxNEEBounceCount;      // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    uint  analyticLightCount;     // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float fireflyFilterThreshold; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter entirely.
    uint  emissiveTriangleCount;  // G4: NEE-eligible emissive triangle count; 0 disables emissive-triangle NEE + MIS.
};
```

- [ ] **Step 3: Add the `EmissiveTriangle` struct to the shared header**

In `PathTracerShared.h`, after the `PolymorphicLightInfo` struct (currently ends at line 126) and before `GeometryVertexData`, add:

```hlsl
// Mirrors Diligent::EmissiveTriangle in RTXPTLights.hpp (total size 64 bytes). One world-space, NEE-eligible
// emissive triangle (constant emitter, non-degenerate). Stores base + two edges + radiance like RTXPT-fork
// TriangleLight (PolymorphicLight.hlsli:399-521); the surface normal and area are recomputed on the fly.
struct EmissiveTriangle
{
    float4 base;     // xyz: world-space vertex 0          (.w unused)
    float4 edge1;    // xyz: world-space (vertex1 - vertex0) (.w unused)
    float4 edge2;    // xyz: world-space (vertex2 - vertex0) (.w unused)
    float4 radiance; // rgb: emitted radiance               (.w unused)
};
```

- [ ] **Step 4: Rename the matching padding word in the C++ settings mirror**

In `RTXPTSample.hpp`, `PathTracerConstants` currently ends:

```cpp
    Uint32 maxNEEBounceCount      = 16;   // Default covers the full bounce budget (NEE at every vertex); a lower value is an optional perf/TDR clamp.
    Uint32 analyticLightCount     = 0;    // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float  fireflyFilterThreshold = 0.0f; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter (set from UI each frame).
    Uint32 _padding1              = 0;
};
static_assert(sizeof(PathTracerConstants) == 48, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

Replace the `_padding1` line:

```cpp
    Uint32 maxNEEBounceCount      = 16;   // Default covers the full bounce budget (NEE at every vertex); a lower value is an optional perf/TDR clamp.
    Uint32 analyticLightCount     = 0;    // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float  fireflyFilterThreshold = 0.0f; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter (set from UI each frame).
    Uint32 emissiveTriangleCount  = 0;    // G4: NEE-eligible emissive triangle count; 0 disables emissive-triangle NEE (set from UI each frame).
};
static_assert(sizeof(PathTracerConstants) == 48, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

(`float`/`Uint32` are both 4 bytes; size stays 48; both `static_assert`s are unchanged.)

- [ ] **Step 5: Add the C++ `EmissiveTriangle` mirror to `RTXPTLights.hpp`**

In `RTXPTLights.hpp`, after the `PolymorphicLightInfo` struct + its `static_assert` (currently lines 39-46), add:

```cpp
struct EmissiveTriangle
{
    float4 base     = float4{0, 0, 0, 0}; // xyz: world-space vertex 0
    float4 edge1    = float4{0, 0, 0, 0}; // xyz: world-space (vertex1 - vertex0)
    float4 edge2    = float4{0, 0, 0, 0}; // xyz: world-space (vertex2 - vertex0)
    float4 radiance = float4{0, 0, 0, 0}; // rgb: emitted radiance
};
static_assert(sizeof(EmissiveTriangle) == 64, "EmissiveTriangle layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 6: Add the emissive-area-light material flag (HLSL)**

In `PathTracerShared.h`, the material flag constants currently read:

```hlsl
static const uint kMaterialFlagHasBaseColorTexture         = 0x1u;
static const uint kMaterialFlagAlphaTested                 = 0x2u;
static const uint kMaterialFlagHasEmissiveTexture          = 0x4u;
static const uint kMaterialFlagHasMetallicRoughnessTexture = 0x8u;
static const uint kMaterialFlagHasNormalTexture            = 0x10u;
```

Append the new bit:

```hlsl
static const uint kMaterialFlagHasBaseColorTexture         = 0x1u;
static const uint kMaterialFlagAlphaTested                 = 0x2u;
static const uint kMaterialFlagHasEmissiveTexture          = 0x4u;
static const uint kMaterialFlagHasMetallicRoughnessTexture = 0x8u;
static const uint kMaterialFlagHasNormalTexture            = 0x10u;
static const uint kMaterialFlagEmissiveAreaLight           = 0x20u; // G4: NEE-eligible constant emitter (no emissive texture, nonzero emissive factor).
```

- [ ] **Step 7: Add the material flag + shared eligibility helper to `RTXPTMaterials.hpp`**

In `RTXPTMaterials.hpp`, the C++ flag constants currently read:

```cpp
constexpr Uint32 kMaterialFlag_HasBaseColorTexture         = 0x1u;
constexpr Uint32 kMaterialFlag_AlphaTested                 = 0x2u;
constexpr Uint32 kMaterialFlag_HasEmissiveTexture          = 0x4u;
constexpr Uint32 kMaterialFlag_HasMetallicRoughnessTexture = 0x8u;
constexpr Uint32 kMaterialFlag_HasNormalTexture            = 0x10u;
```

Append the new bit:

```cpp
constexpr Uint32 kMaterialFlag_HasBaseColorTexture         = 0x1u;
constexpr Uint32 kMaterialFlag_AlphaTested                 = 0x2u;
constexpr Uint32 kMaterialFlag_HasEmissiveTexture          = 0x4u;
constexpr Uint32 kMaterialFlag_HasMetallicRoughnessTexture = 0x8u;
constexpr Uint32 kMaterialFlag_HasNormalTexture            = 0x10u;
constexpr Uint32 kMaterialFlag_EmissiveAreaLight           = 0x20u; // G4: NEE-eligible constant emitter.
```

Declare the shared helper next to the existing `RTXPTMaterialIsAlphaTested` declaration:

```cpp
bool RTXPTMaterialIsAlphaTested(const GLTF::Material& Material);

// G4: a material is a NEE-eligible emissive area light when it has a non-zero emissive factor and references
// no emissive texture (constant emission, so Bridge::getEmission == emissiveFactor). This single predicate
// is the sole source of truth shared by the GPU material flag (RTXPTMaterials) and the emissive-triangle
// topology sizing / GPU builder (RTXPTLights), so the two can never disagree.
bool RTXPTMaterialIsEmissiveAreaLight(const GLTF::Material& Material);
```

- [ ] **Step 8: Define the helper + set the flag in `RTXPTMaterials.cpp`**

In `RTXPTMaterials.cpp`, after the `RTXPTMaterialIsAlphaTested` definition (currently lines 36-40), add:

```cpp
bool RTXPTMaterialIsEmissiveAreaLight(const GLTF::Material& Material)
{
    if (Material.GetTextureId(GLTF::DefaultEmissiveTextureAttribId) >= 0)
        return false;
    const float3& E = Material.Attribs.EmissiveFactor;
    return E.x > 0.0f || E.y > 0.0f || E.z > 0.0f;
}
```

In `RTXPTMaterials::Upload`, the per-material flag block currently ends (just before `MaterialData.emplace_back(Data);`):

```cpp
        // Alpha test requires the base-color texture (its .a channel). Only set the flag when both agree.
        if (RTXPTMaterialIsAlphaTested(Material) && (Data.flags & kMaterialFlag_HasBaseColorTexture) != 0u)
            Data.flags |= kMaterialFlag_AlphaTested;

        MaterialData.emplace_back(Data);
```

Insert the emissive-area-light flag set before `MaterialData.emplace_back(Data);`:

```cpp
        // Alpha test requires the base-color texture (its .a channel). Only set the flag when both agree.
        if (RTXPTMaterialIsAlphaTested(Material) && (Data.flags & kMaterialFlag_HasBaseColorTexture) != 0u)
            Data.flags |= kMaterialFlag_AlphaTested;

        // G4: mark NEE-eligible constant emitters so the closest-hit can MIS-weight their BSDF hits.
        if (RTXPTMaterialIsEmissiveAreaLight(Material))
            Data.flags |= kMaterialFlag_EmissiveAreaLight;

        MaterialData.emplace_back(Data);
```

- [ ] **Step 9: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h \
        Samples/RTXPT/src/RTXPTSample.hpp \
        Samples/RTXPT/src/RTXPTLights.hpp \
        Samples/RTXPT/src/RTXPTMaterials.hpp \
        Samples/RTXPT/src/RTXPTMaterials.cpp
git commit -m "feat(rtxpt): add EmissiveTriangle struct, settings count, + emissive-area-light material flag (R2/G4, inert)"
```

**Verify:** C++ build compiles; `static_assert(sizeof(PathTracerConstants)==48)`, `sizeof(SampleConstants)==208`, `sizeof(MaterialPTData)==96`, and the new `sizeof(EmissiveTriangle)==64` all hold. No behavior change (nothing reads the new fields/flag yet). Run the build only if the user asks.

---

### Task 2: Build emissive triangles on the GPU from current geometry

R2 implements the static/current-geometry path here. The same pass must also stay compatible with animated/skinned glTF because the sample already has a current-frame skinned vertex buffer and dynamic BLAS update path; the rule is to consume that same frame data, not bind-pose data.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Add: `DiligentSamples/Samples/RTXPT/src/RTXPTEmissiveTrianglePass.hpp`
- Add: `DiligentSamples/Samples/RTXPT/src/RTXPTEmissiveTrianglePass.cpp`
- Add: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/EmissiveTriangleBuild.hlsl`

- [ ] **Step 1: Add topology sizing in `RTXPTLights`**

In `RTXPTLights.hpp`, keep `m_EmissiveTriangleBuffer` as the owned SRV/UAV buffer, but change the class contract so the CPU only computes the fixed emissive-triangle count from scene topology and the shared material predicate. The GPU builder later fills the buffer.

```cpp
struct RTXPTLightStats
{
    Uint32      LightCount            = 0;
    Uint32      EmissiveTriangleCount = 0;
    std::string LastError;
};

class RTXPTLights
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Scene& Scene, const GLTF::ModelTransforms& Transforms);
    bool UploadEmissiveTriangles(IRenderDevice*               pDevice,
                                 const GLTF::Model&           Model,
                                 Uint32                       SceneIndex,
                                 const GLTF::ModelTransforms& Transforms,
                                 VALUE_TYPE                   IndexType);

    const RTXPTLightStats& GetStats() const { return m_Stats; }
    IBuffer*               GetLightBuffer() const { return m_LightBuffer; }
    IBuffer*               GetEmissiveTriangleBuffer() const { return m_EmissiveTriangleBuffer; }
    Uint32                 GetEmissiveTriangleCount() const { return m_Stats.EmissiveTriangleCount; }

private:
    RefCntAutoPtr<IBuffer> m_LightBuffer;
    RefCntAutoPtr<IBuffer> m_EmissiveTriangleBuffer;
    RTXPTLightStats        m_Stats;
};
```

The count helper walks scene topology only, using the shared predicate from `RTXPTMaterials`, and sums the eligible triangles without touching vertex data:

```cpp
for (const GLTF::Primitive& Primitive : pNode->pMesh->Primitives)
{
    if (Primitive.MaterialId >= Model.Materials.size())
        continue;
    if (!RTXPTMaterialIsEmissiveAreaLight(Model.Materials[Primitive.MaterialId]))
        continue;

    const Uint32 TriCount = Primitive.HasIndices() ? Primitive.IndexCount / 3u : Primitive.VertexCount / 3u;
    m_Stats.EmissiveTriangleCount += TriCount;
}
```

Allocate `m_EmissiveTriangleBuffer` as a structured buffer with both SRV and UAV views so the compute pass can write it and raygen can read it. Use a dummy one-element buffer when the count is zero.

- [ ] **Step 2: Add the GPU build pass**

Add `RTXPTEmissiveTrianglePass` as a small compute pass that writes `EmissiveTriangle` records into the buffer owned by `RTXPTLights`.

```cpp
bool Initialize(IRenderDevice*  pDevice,
                IEngineFactory* pEngineFactory,
                IBuffer*        pFrameConstants,
                IBuffer*        pMaterialBuffer,
                IBuffer*        pSubInstanceBuffer,
                IBuffer*        pVertexBuffer,
                IBuffer*        pIndexBuffer,
                IBuffer*        pEmissiveTriangleBuffer,
                bool            ComputeSupported);

bool Dispatch(IDeviceContext* pContext, Uint32 EmissiveTriangleCount);
```

The compute resource layout should bind the live buffers as static SRVs and the emissive output as a UAV:

```cpp
ResourceLayout
    .AddVariable(SHADER_TYPE_COMPUTE, "g_Const", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_PTMaterialData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_SubInstanceData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_VertexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_EmissiveTriangles", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

`EmissiveTriangleBuild.hlsl` reads the current-frame vertex/index data, fetches the same triangle corners the ray tracer will see, transforms them to world space, and writes the world-space base/edge/radiance record for each eligible triangle. For static glTF this is the existing vertex/index stream. For skinned glTF, the same code path must be fed by the existing GPU-produced post-skinning vertex buffer that the dynamic BLAS path already consumes; never fall back to bind-pose data. The output order must match the topology scan from `RTXPTLights` so the count and the buffer agree.

- [ ] **Step 3: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTLights.hpp \
        Samples/RTXPT/src/RTXPTLights.cpp \
        Samples/RTXPT/src/RTXPTEmissiveTrianglePass.hpp \
        Samples/RTXPT/src/RTXPTEmissiveTrianglePass.cpp \
        Samples/RTXPT/assets/shaders/PathTracer/EmissiveTriangleBuild.hlsl
git commit -m "feat(rtxpt): build emissive triangles on the GPU from current geometry (R2/G4)"
```

**Verify:** C++ build compiles. No CPU geometry readback remains. The emitted triangle order is deterministic, and the buffer stays in sync with the current-frame geometry path that the BLAS and closest-hit shader use, including the skinned/dynamic path already present in the sample. Run the build only if the user asks.

---

### Task 3: Wire the emissive builder with a static fast path + dirty rebuilds + UI

Runs the emissive builder once for static scenes, uploads `emissiveTriangleCount` each frame (0 when disabled), and exposes the toggle/stat. The count is topology-derived and stable; the GPU pass refreshes world-space triangle data on the static fast path in R2, while any skinned/dynamic refresh must reuse the already-integrated current-geometry + dynamic BLAS path.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add the emissive NEE toggle + build scheduling members**

In `RTXPTSample.hpp`, include the new pass header with the other RTXPT pass headers:

```cpp
#include "RTXPTComputePass.hpp"
#include "RTXPTEmissiveTrianglePass.hpp"
#include "RTXPTLights.hpp"
```

Add the pass member next to the existing compute/ray-tracing passes:

```cpp
    RTXPTRayTracingPass         m_RayTracingPass;
    RTXPTComputePass            m_DebugComputePass;
    RTXPTEmissiveTrianglePass   m_EmissiveTrianglePass;
    RTXPTBlitPass               m_BlitPass;
```

Then add the scheduling state next to the other NEE members (after `bool m_EnableEnvNEE = true;`, currently line 159):

```cpp
    bool                        m_EnableNEE               = true;
    bool                        m_EnableEnvNEE            = true;
    bool                        m_EnableEmissiveNEE       = true;
    bool                        m_HasDynamicGeometry = false; // scene contains skinned/animated vertices; keep emissive work aligned with current-frame geometry.
    bool                        m_EmissiveTrianglesDirty = true; // static-path initial build / later topology changes only.
```

- [ ] **Step 2: Initialize the emissive build pass at load**

In `RTXPTSample.cpp`, `Initialize`, keep the existing material/light uploads, then size the emissive-triangle buffer from topology and initialize the GPU builder with the current material, subinstance, vertex, and index buffers. There is no geometry readback and no CPU-side triangle-position walk. Detect whether the scene has dynamic vertex deformation and keep the emissive builder aligned with the existing skinned current-geometry path when that data is present.

```cpp
        m_Materials.Upload(m_pDevice, *pModel);
        if (m_Scene.GetSceneIndex() < pModel->Scenes.size())
            m_Lights.Upload(m_pDevice, pModel->Scenes[m_Scene.GetSceneIndex()], m_Scene.GetTransforms());

        m_Lights.UploadEmissiveTriangles(m_pDevice, *pModel,
                                         m_Scene.GetSceneIndex(),
                                         m_Scene.GetTransforms(),
                                         m_Scene.GetIndexType());
        m_HasDynamicGeometry = pModel->SkinTransformsCount > 0;
        m_EmissiveTrianglesDirty = !m_HasDynamicGeometry;

        m_AccelerationStructures.BuildStaticScene(m_pDevice,
                                                  m_pImmediateContext,
                                                  *pModel,
                                                  m_Scene.GetSceneIndex(),
                                                  m_Scene.GetIndexType(),
                                                  m_Scene.GetTransforms(),
                                                  m_FeatureCaps.RayTracing);

        m_EmissiveTrianglePass.Initialize(m_pDevice,
                                          m_pEngineFactory,
                                          m_FrameConstantsCB,
                                          m_Materials.GetMaterialBuffer(),
                                          m_AccelerationStructures.GetSubInstanceBuffer(),
                                          m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
                                          m_Scene.GetIndexBuffer(m_pDevice, m_pImmediateContext),
                                          m_Lights.GetEmissiveTriangleBuffer(),
                                          m_FeatureCaps.ComputeShaders);
```

- [ ] **Step 3: Build static emissive triangles once**

Immediately after the static BLAS is built and the emissive builder is initialized, build the emissive-triangle buffer once for scenes without dynamic vertex deformation. This is the static fast path: static glTF content does not dispatch the emissive builder again in the frame loop. If the scene is skinned/animated, route any emissive refresh through the existing current-geometry update path rather than bind-pose data.

```cpp
        if (!m_HasDynamicGeometry)
        {
            if (m_Lights.GetEmissiveTriangleCount() > 0u)
                m_EmissiveTrianglePass.Dispatch(m_pImmediateContext, m_Lights.GetEmissiveTriangleCount());
            m_EmissiveTrianglesDirty = false;
        }
```

- [ ] **Step 4: Rebuild dynamic emissive triangles only when geometry is dirty**

In the frame path, keep the static rebuild gate separate from the dynamic path. If `m_HasDynamicGeometry` is true, let the existing skinning + dynamic BLAS update produce the current vertex buffer first, then trigger any emissive refresh from that same current geometry. Static scenes never enter that dynamic branch because `m_HasDynamicGeometry == false`.

```cpp
    m_LastFrameConstants.ptConsts.emissiveTriangleCount =
        (m_EnableNEE && m_EnableEmissiveNEE && !m_HasDynamicGeometry) ? m_Lights.GetEmissiveTriangleCount() : 0u;

    if (m_HasDynamicGeometry)
    {
        // Skinned geometry already flows through RTXPTSkinnedGeometry + UpdateDynamicBLAS.
        // Any dynamic emissive refresh must consume that same current-frame skinned buffer.
    }
    else if (m_EmissiveTrianglesDirty &&
             m_EnableNEE && m_EnableEmissiveNEE && m_Lights.GetEmissiveTriangleCount() > 0u)
    {
        m_EmissiveTrianglePass.Dispatch(m_pImmediateContext, m_Lights.GetEmissiveTriangleCount());
        m_EmissiveTrianglesDirty = false;
    }
```

- [ ] **Step 5: Add the `Emissive mesh NEE + MIS` UI toggle**

In `RTXPTSample.cpp`, `UpdateUI`, the "Light sampling" group currently has:

```cpp
            ResetOnChange(ImGui::Checkbox("Use Next Event Estimation", &m_EnableNEE), "NEE toggled");
            ResetOnChange(ImGui::Checkbox("Environment NEE + MIS", &m_EnableEnvNEE), "Environment NEE toggled");
```

Insert the emissive toggle between them:

```cpp
            ResetOnChange(ImGui::Checkbox("Use Next Event Estimation", &m_EnableNEE), "NEE toggled");
            ResetOnChange(ImGui::Checkbox("Emissive mesh NEE + MIS", &m_EnableEmissiveNEE), "Emissive NEE toggled");
            ResetOnChange(ImGui::Checkbox("Environment NEE + MIS", &m_EnableEnvNEE), "Environment NEE toggled");
```

- [ ] **Step 6: Add an emissive-triangle stat to the Scene panel**

In `RTXPTSample.cpp`, `UpdateUI`, the Scene section currently has:

```cpp
        ImGui::Text("Lights: %u", m_Lights.GetStats().LightCount);
```

add a line after it:

```cpp
        ImGui::Text("Lights: %u", m_Lights.GetStats().LightCount);
        ImGui::Text("Emissive triangles: %u", m_Lights.GetStats().EmissiveTriangleCount);
```

- [ ] **Step 7: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): wire emissive triangle GPU build + UI toggle + per-frame count (R2/G4)"
```

**Verify:** Build compiles; sample launches. The Scene panel shows a non-zero "Emissive triangles" count for the Bistro scene; the new toggle appears. For the static Bistro scene, BLAS is built once and `RTXPTEmissiveTrianglePass::DispatchCount` stays at 1 across frames. No geometry readback occurs on the CPU path. Run only if the user asks.

---
### Task 4: Grow the payload + closest-hit precomputes the emissive solid-angle pdf

Adds the per-hit `emissiveLightPdf` the raygen needs for MIS and has the closest-hit fill it for NEE-eligible emitters. The field is written but not read yet — inert, runnable.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Grow `PathPayload` to 80 bytes**

In `PathTracerShared.h`, the `PathPayload` struct currently ends:

```hlsl
    float3 emission;
    float  roughness;
};
```

Replace those closing lines with:

```hlsl
    float3 emission;
    float  roughness;

    // G4: area-light solid-angle pdf of this emissive triangle as seen from the previous vertex, for
    // NEE/BSDF MIS in the raygen loop. 0 unless the hit is a NEE-eligible (constant-emission, non-degenerate)
    // emitter. The uniform selection pdf (1/emissiveTriangleCount) is applied raygen-side, not here.
    float  emissiveLightPdf;
    float  _pad0;
    float  _pad1;
    float  _pad2;
};
```

Also update the size note in the comment block above `PathPayload` — change "Size is 64 bytes (16 floats)" to "Size is 80 bytes (20 floats)" and the `MaxPayloadSize` reference accordingly.

- [ ] **Step 2: Initialize the new field in `MakeEmptyPayload`**

In `PathTracer.hlsli`, `PathTracer::MakeEmptyPayload`, the body currently ends:

```hlsl
        payload.metallic    = 0.0;
        payload.roughness   = 1.0;
        return payload;
```

Insert the init before `return`:

```hlsl
        payload.metallic         = 0.0;
        payload.roughness        = 1.0;
        payload.emissiveLightPdf = 0.0;
        return payload;
```

- [ ] **Step 3: Bump `MaxPayloadSize` to 80 bytes**

In `RTXPTRayTracingPass.cpp`, `Initialize`, this block currently reads:

```cpp
    PSOCreateInfo.MaxAttributeSize                     = static_cast<Uint32>(sizeof(float) * 2);
    // PathPayload = 4 * float4 = 64 bytes.
    PSOCreateInfo.MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 16);
```

Replace the payload lines:

```cpp
    PSOCreateInfo.MaxAttributeSize                     = static_cast<Uint32>(sizeof(float) * 2);
    // PathPayload = 5 * float4 = 80 bytes (worldPos/normal/baseColor/emission + emissiveLightPdf for G4 MIS).
    PSOCreateInfo.MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 20);
```

- [ ] **Step 4: Compute the emissive solid-angle pdf in the closest hit**

In `PathTracerClosestHit.rchit`, inside the `if (Bridge::hasSubInstanceTable() && Bridge::hasMaterialTable())` block, the emission is set at:

```cpp
        BaseColor        = Bridge::getBaseColor(material, texCoord).rgb;
        Payload.emission = Bridge::getEmission(material, texCoord);
    }
```

Insert the pdf computation just before the closing `}` of that block (after the `Payload.emission` line):

```cpp
        BaseColor        = Bridge::getBaseColor(material, texCoord).rgb;
        Payload.emission = Bridge::getEmission(material, texCoord);

        // G4: precompute this triangle's area-light solid-angle pdf so the raygen can MIS-weight emissive
        // BSDF hits against emissive-triangle NEE. Only NEE-eligible emitters get a non-zero pdf; the flag is
        // the same single source of truth that sized the topology and drove the GPU builder
        // (RTXPTMaterialIsEmissiveAreaLight), so the BSDF-hit MIS gate and the NEE list agree exactly.
        // Textured/non-emissive hits stay BSDF-only.
        if ((material.flags & kMaterialFlagEmissiveAreaLight) != 0u)
        {
            const float3 wp0   = mul(ObjectToWorld3x4(), float4(V0.position, 1.0));
            const float3 wp1   = mul(ObjectToWorld3x4(), float4(V1.position, 1.0));
            const float3 wp2   = mul(ObjectToWorld3x4(), float4(V2.position, 1.0));
            const float3 ng    = cross(wp1 - wp0, wp2 - wp0);
            const float  ngLen = length(ng);
            const float  area  = 0.5 * ngLen;
            if (area > 1e-9)
            {
                // Two-sided emitter (abs): emit from both faces so the converged image matches the pre-R2
                // two-sided emission. RTXPT-fork TriangleLight is one-sided; see RTXPT_FORK_MAPPING.md.
                const float cosTheta = abs(dot(ng / ngLen, -WorldRayDirection()));
                if (cosTheta > 2e-9)
                    Payload.emissiveLightPdf = min(kMaxSolidAnglePdf, (1.0 / area) * (RayTCurrent() * RayTCurrent()) / cosTheta);
            }
        }
    }
```

(`kMaxSolidAnglePdf` and `kMaterialFlagHasEmissiveTexture` come from `PathTracerShared.h`, which the chit already includes via `PathTracerBridge.hlsli`.)

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit \
        Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git commit -m "feat(rtxpt): closest-hit precomputes emissive triangle solid-angle pdf into payload (R2/G4)"
```

**Verify:** Build compiles; sample renders identically (the new payload field is written but not yet read). The RT PSO still builds with the larger 80-byte payload on both D3D12 and Vulkan. Run only if the user asks.

---

### Task 5: Shader plumbing — triangle-sampling helpers, bridge buffer + accessors, effect salt

Adds the pure helpers and the `t_EmissiveTriangles` buffer declaration + accessors. The buffer/accessors stay unreferenced for now (DXC strips them; no binding needed yet), so the build stays green and runnable.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli`

- [ ] **Step 1: Add `pdfAtoW` + `SampleTriangleUniform` to `PolymorphicLight.hlsli`**

In `PolymorphicLight.hlsli`, after `#include "../PathTracerShared.h"` (currently line 4) and before `struct LightSample`, add:

```hlsl
// Area-measure -> solid-angle-measure pdf conversion (RTXPT-fork Utils/Geometry.hlsli:79).
float pdfAtoW(float pdfA, float distance, float cosTheta)
{
    return pdfA * (distance * distance) / max(cosTheta, 2e-9);
}

// Uniformly sampled barycentric coordinates inside a triangle (RTXPT-fork Utils/Geometry.hlsli:33).
float3 SampleTriangleUniform(float2 rnd)
{
    const float sqrtx = sqrt(rnd.x);
    return float3(1.0 - sqrtx, sqrtx * (1.0 - rnd.y), sqrtx * rnd.y);
}
```

- [ ] **Step 2: Declare the emissive-triangle buffer in the bridge**

In `PathTracerBridge.hlsli`, the global resource block currently reads:

```hlsl
ConstantBuffer<SampleConstants>        g_Const;
StructuredBuffer<SubInstanceData>      t_SubInstanceData;
StructuredBuffer<PolymorphicLightInfo> t_Lights;
StructuredBuffer<GeometryVertexData>   t_VertexBuffer;
Buffer<uint>                           t_IndexBuffer;
```

Add the emissive buffer after `t_Lights`:

```hlsl
ConstantBuffer<SampleConstants>        g_Const;
StructuredBuffer<SubInstanceData>      t_SubInstanceData;
StructuredBuffer<PolymorphicLightInfo> t_Lights;
StructuredBuffer<EmissiveTriangle>     t_EmissiveTriangles;
StructuredBuffer<GeometryVertexData>   t_VertexBuffer;
Buffer<uint>                           t_IndexBuffer;
```

- [ ] **Step 3: Add the emissive accessors to the `Bridge` namespace**

In `PathTracerBridge.hlsli`, after `getLight` (currently lines 148-151) and before the closing `} // namespace Bridge`:

```hlsl
    PolymorphicLightInfo getLight(uint index)
    {
        return t_Lights[index];
    }

    // G4: NEE-eligible emissive triangle count (0 disables emissive NEE + MIS) and per-triangle fetch.
    uint getEmissiveTriangleCount()
    {
        return g_Const.ptConsts.emissiveTriangleCount;
    }

    EmissiveTriangle getEmissiveTriangle(uint index)
    {
        return t_EmissiveTriangles[index];
    }
} // namespace Bridge
```

- [ ] **Step 4: Add the emissive NEE effect salt**

In `Utils/SampleGenerators.hlsli`, the per-effect salts currently read:

```hlsl
static const uint kSampleEffect_Base                = 0u;
static const uint kSampleEffect_ScatterBSDF         = 1u;
static const uint kSampleEffect_NextEventEstimation = 2u;
static const uint kSampleEffect_NEELightSampler     = 3u;
static const uint kSampleEffect_RussianRoulette     = 6u;
```

Insert the emissive salt (RTXPT-fork leaves enum value 4 unused; the port uses it to decorrelate emissive-triangle sampling from analytic NEE at the same vertex):

```hlsl
static const uint kSampleEffect_Base                = 0u;
static const uint kSampleEffect_ScatterBSDF         = 1u;
static const uint kSampleEffect_NextEventEstimation = 2u;
static const uint kSampleEffect_NEELightSampler     = 3u;
static const uint kSampleEffect_NEEEmissive         = 4u; // emissive-triangle NEE (RTXPT lumps lights into one sampler; the port decorrelates this on its own salt)
static const uint kSampleEffect_RussianRoulette     = 6u;
```

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli
git commit -m "feat(rtxpt): add emissive triangle bridge buffer + sampling helpers (R2/G4, unreferenced)"
```

**Verify:** Build compiles; sample renders identically. `t_EmissiveTriangles` and the new accessors are unreferenced, so DXC strips them and no binding is required yet. Run only if the user asks.

---

### Task 6: Turn on emissive NEE + MIS (atomic feature flip)

Adds `PathTracer::SampleEmissiveNEE`, calls it from the raygen, MIS-weights emissive BSDF hits, and binds `t_EmissiveTriangles`. The shader reference and the C++ STATIC binding must land **together** (a referenced static resource needs a binding; an unreferenced one cannot be bound), so this is one task.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add `PathTracer::SampleEmissiveNEE`**

In `PathTracer.hlsli`, after `SampleEnvironmentNEE` (currently ends at line 130) and before `ComputeBSDFEnvMISWeight`, add:

```hlsl
    // Emissive-triangle area-light NEE with power-heuristic MIS against the BSDF (G4). Uniform triangle
    // selection (RIS/WRS is Phase R3); two-sided emitters; radiance is the unscaled emissive factor so the
    // converged result matches BSDF-only emissive gathering. Mirrors RTXPT-fork TriangleLight::CalcSample.
    float3 SampleEmissiveNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                             float3 wo, inout SampleGenerator sg, float fireflyFilterK)
    {
        const uint triCount = Bridge::getEmissiveTriangleCount();
        if (triCount == 0u)
            return float3(0.0, 0.0, 0.0);

        const uint             triIndex = min(uint(sampleNext1D(sg) * float(triCount)), triCount - 1u);
        const EmissiveTriangle tri      = Bridge::getEmissiveTriangle(triIndex);

        const float3 ng    = cross(tri.edge1.xyz, tri.edge2.xyz);
        const float  ngLen = length(ng);
        if (ngLen <= 0.0)
            return float3(0.0, 0.0, 0.0);
        const float  area   = 0.5 * ngLen;
        const float3 normal = ng / ngLen;

        const float3 bary = SampleTriangleUniform(sampleNext2D(sg));
        const float3 P    = tri.base.xyz + tri.edge1.xyz * bary.y + tri.edge2.xyz * bary.z;

        const float3 toLight = P - hitPos;
        const float  distSq  = max(1e-9, dot(toLight, toLight));
        const float  dist    = sqrt(distSq);
        const float3 wi      = toLight / dist;

        const float cosTheta = abs(dot(normal, -wi)); // two-sided emitter
        if (cosTheta <= 2e-9)
            return float3(0.0, 0.0, 0.0);

        const float solidAnglePdf = min(kMaxSolidAnglePdf, pdfAtoW(1.0 / area, dist, cosTheta));
        const float selectionPdf  = 1.0 / float(triCount);
        const float lightPdf      = selectionPdf * solidAnglePdf;
        if (lightPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, wi, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        // Shorten the shadow ray to avoid self-intersecting the light surface (RTXPT selfIntersectionShorteningK).
        if (!TraceVisibilityRay(visibilityOrigin, wi, dist * 0.9985))
            return float3(0.0, 0.0, 0.0);

        // f already includes NoL; divide by the light-sampling pdf and power-heuristic-weight against the BSDF.
        const float  misWeight    = PowerHeuristic(1.0, lightPdf, 1.0, bsdfPdf);
        float3       contribution = f * tri.radiance.rgb * (misWeight / lightPdf);

        // G1: dampen NEE fireflies using the light-sampling pdf as the spread proxy (matches analytic/env NEE).
        const float ffThreshold = g_Const.ptConsts.fireflyFilterThreshold;
        if (ffThreshold != 0.0)
        {
            const float neeK = ComputeNewScatterFireflyFilterK(fireflyFilterK, lightPdf, 1.0);
            contribution *= FireflyFilterShort(Average(contribution), ffThreshold, neeK);
        }

        return contribution;
    }
```

- [ ] **Step 2: Track `prevDidEmissiveNEE` in the raygen**

In `PathTracerSample.rgen`, the pre-loop state currently declares:

```hlsl
    float  prevBsdfPdf   = 0.0;
    float3 prevNormal    = float3(0.0, 1.0, 0.0);
    bool   prevDidEnvNEE = false;
```

Add the emissive tracker:

```hlsl
    float  prevBsdfPdf        = 0.0;
    float3 prevNormal         = float3(0.0, 1.0, 0.0);
    bool   prevDidEnvNEE      = false;
    bool   prevDidEmissiveNEE = false;
```

- [ ] **Step 3: MIS-weight emissive BSDF hits**

In `PathTracerSample.rgen`, the emissive accumulation currently reads:

```hlsl
        // Accumulate emissive surfaces hit by BSDF sampling. Area-light NEE is deferred to a later lighting pass.
        float3 surfaceEmission = payload.emission;
        if (ffThreshold != 0.0)
            surfaceEmission = FireflyFilter(surfaceEmission, ffThreshold, fireflyFilterK);
        pathRadiance += throughput * surfaceEmission;
```

Replace it with the MIS-weighted version:

```hlsl
        // Accumulate emissive surfaces hit by BSDF sampling, MIS-weighted against emissive-triangle NEE (G4).
        float3     surfaceEmission = payload.emission;
        const uint emissiveCount   = g_Const.ptConsts.emissiveTriangleCount;
        if (prevDidEmissiveNEE && prevBsdfPdf > 0.0 && payload.emissiveLightPdf > 0.0 && emissiveCount > 0u)
        {
            const float lightPdf = (1.0 / float(emissiveCount)) * payload.emissiveLightPdf;
            surfaceEmission *= PowerHeuristic(1.0, prevBsdfPdf, 1.0, lightPdf);
        }
        if (ffThreshold != 0.0)
            surfaceEmission = FireflyFilter(surfaceEmission, ffThreshold, fireflyFilterK);
        pathRadiance += throughput * surfaceEmission;
```

- [ ] **Step 4: Call emissive NEE in the NEE block**

In `PathTracerSample.rgen`, the NEE block currently reads:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            SampleGenerator sgNEELight = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEELightSampler);
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sgNEELight, fireflyFilterK);
            if (enableEnvNEE)
            {
                SampleGenerator sgEnvNEE = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NextEventEstimation);
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sgEnvNEE, fireflyFilterK);
            }
        }
```

Insert the emissive NEE call after the analytic NEE call:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            SampleGenerator sgNEELight = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEELightSampler);
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sgNEELight, fireflyFilterK);

            SampleGenerator sgEmissive = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEEEmissive);
            pathRadiance += throughput * PathTracer::SampleEmissiveNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sgEmissive, fireflyFilterK);

            if (enableEnvNEE)
            {
                SampleGenerator sgEnvNEE = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NextEventEstimation);
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sgEnvNEE, fireflyFilterK);
            }
        }
```

- [ ] **Step 5: Set `prevDidEmissiveNEE` after the scatter**

In `PathTracerSample.rgen`, the per-iteration state update currently reads:

```hlsl
        prevBsdfPdf   = pdf;
        prevNormal    = bsdfData.N;
        prevDidEnvNEE = useNEE && enableEnvNEE;
```

Add the emissive line:

```hlsl
        prevBsdfPdf        = pdf;
        prevNormal         = bsdfData.N;
        prevDidEnvNEE      = useNEE && enableEnvNEE;
        prevDidEmissiveNEE = useNEE && (g_Const.ptConsts.emissiveTriangleCount > 0u);
```

- [ ] **Step 6: Add the emissive buffer parameter to the RT-pass `Initialize` signature**

In `RTXPTRayTracingPass.hpp`, the `Initialize` declaration currently has:

```cpp
                    IBuffer*              pSubInstanceBuffer,
                    IBuffer*              pLightBuffer,
                    IBuffer*              pVertexBuffer,
```

Insert the emissive buffer after `pLightBuffer`:

```cpp
                    IBuffer*              pSubInstanceBuffer,
                    IBuffer*              pLightBuffer,
                    IBuffer*              pEmissiveTriangleBuffer,
                    IBuffer*              pVertexBuffer,
```

Add a stat to `RTXPTRayTracingPassStats` (next to `LightBridgeBound`):

```cpp
    bool        LightBridgeBound         = false;
    bool        EmissiveLightBridgeBound = false;
```

- [ ] **Step 7: Match the signature, layout, and binding in `RTXPTRayTracingPass.cpp`**

In `RTXPTRayTracingPass.cpp`, update the `Initialize` definition signature the same way (insert `IBuffer* pEmissiveTriangleBuffer,` after `IBuffer* pLightBuffer,`).

Add the layout variable — the `FullPathTracer` resource-layout block currently reads:

```cpp
            .AddVariable(HitStages, "t_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
            .AddVariable(SHADER_TYPE_RAY_GEN, "t_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC);
```

Append the emissive buffer (raygen-only, like `t_Lights`):

```cpp
            .AddVariable(HitStages, "t_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
            .AddVariable(SHADER_TYPE_RAY_GEN, "t_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
            .AddVariable(SHADER_TYPE_RAY_GEN, "t_EmissiveTriangles", SHADER_RESOURCE_VARIABLE_TYPE_STATIC);
```

In the `!FullPathTracer` stats block, set the new flag true (so the bound-check passes for diagnostic modes):

```cpp
        m_Stats.LightBridgeBound      = true;
        m_Stats.EmissiveLightBridgeBound = true;
```

Add the SRV resolve next to the others (after `pLightsView`):

```cpp
    IDeviceObject* pLightsView      = pLightBuffer != nullptr ? pLightBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
    IDeviceObject* pEmissiveView    = pEmissiveTriangleBuffer != nullptr ? pEmissiveTriangleBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
```

In the `FullPathTracer` binding block, after the `t_Lights` `SetStatic`:

```cpp
        m_Stats.LightBridgeBound         = SetStatic(SHADER_TYPE_RAY_GEN, "t_Lights", pLightsView, "light buffer");
        m_Stats.EmissiveLightBridgeBound = SetStatic(SHADER_TYPE_RAY_GEN, "t_EmissiveTriangles", pEmissiveView, "emissive triangle buffer");
```

Extend the bound-resources guard:

```cpp
    if (!m_Stats.MaterialBridgeBound || !m_Stats.SubInstanceBound || !m_Stats.LightBridgeBound ||
        !m_Stats.EmissiveLightBridgeBound || !m_Stats.VertexBufferBound || !m_Stats.IndexBufferBound)
    {
        if (m_Stats.LastError.empty())
            m_Stats.LastError = "Failed to bind required RTXPT bridge buffers";
        return false;
    }
```

- [ ] **Step 8: Pass the emissive buffer from the sample (both Initialize calls)**

In `RTXPTSample.cpp`, `CreatePhase4Passes`, both `m_RayTracingPass.Initialize(...)` calls pass `m_Lights.GetLightBuffer(),` followed by `m_Scene.GetVertexBuffer0(...)`. In **both** calls, insert the emissive buffer between them:

```cpp
                                    m_AccelerationStructures.GetSubInstanceBuffer(),
                                    m_Lights.GetLightBuffer(),
                                    m_Lights.GetEmissiveTriangleBuffer(),
                                    m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
```

- [ ] **Step 9: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen \
        Samples/RTXPT/src/RTXPTRayTracingPass.hpp \
        Samples/RTXPT/src/RTXPTRayTracingPass.cpp \
        Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): emissive-triangle area-light NEE + BSDF MIS (R2/G4)"
```

**Verify (the phase acceptance test):** Build + run on D3D12 and Vulkan (user-initiated).
1. **Unbiasedness:** point the camera at an emissive-mesh-lit area, let it converge with **Emissive mesh NEE + MIS** ON, then OFF — the converged images must match (only noise differs). Also toggle the master **Use Next Event Estimation** to confirm the same.
2. **Convergence delta:** with the toggle ON, the emissive-lit region should resolve in far fewer samples (much less noise at low sample counts).
3. **No double counting:** brightness must not increase when enabling the estimator (MIS weights for a shared direction sum to ≈ 1).
4. Both backends launch and render. Run only if the user asks.

---

### Task 7: Documentation — mapping rows + TODO marker retargeting

Records the new symbols and the two-sided-emitter divergence, and resolves/retargets the in-code `Phase 5.4` / `Phase R2` markers this phase closes.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Add mapping rows + divergence notes**

In `RTXPT_FORK_MAPPING.md`, add rows mapping the new port symbols to RTXPT-fork (match the file's existing table/`(style)` format):

```
| `TriangleLight` (struct) | `EmissiveTriangle` (R2/G4 backing layout) | base+edge1+edge2+radiance; normal/area recomputed |
| `TriangleLight::CalcSample` | `PathTracer::SampleEmissiveNEE` | uniform triangle selection (RIS is R3) |
| `ComputeBSDFMISForEmissiveTriangle` | raygen emissive BSDF-hit MIS (`payload.emissiveLightPdf` + power heuristic) | |
| `SampleTriangleUniform` / `pdfAtoW` / `MAX_SOLID_ANGLE_PDF` | `SampleTriangleUniform` / `pdfAtoW` / `kMaxSolidAnglePdf` (style) | |
| `LightsBaker` emissive triangle list | `RTXPTLights::UploadEmissiveTriangles` + `RTXPTEmissiveTrianglePass` (GPU build from current geometry) | minimal Diligent data path, not the baker |
| (R2/G4 new) | `kSampleEffect_NEEEmissive` | fills RTXPT-fork's unused enum value 4 |
```

Add to the divergence-notes list at the bottom of the file:

```
- Emissive-triangle area lights (R2/G4) are **two-sided** (`abs(cosTheta)` in both the
  NEE estimator and the BSDF-hit MIS), unlike RTXPT-fork's one-sided `TriangleLight`.
  This preserves the port's pre-R2 two-sided emissive look and stays unbiased.
  Textured-emissive triangles are excluded from NEE (BSDF-only) for now. Selection is
  uniform (RIS/WRS is Phase R3). TODO: align one-sided + double-sided baker semantics.
```

- [ ] **Step 2: Resolve the closest-hit `Phase 5.4` marker**

In `PathTracerClosestHit.rchit`, the trailing marker currently reads:

```cpp
// TODO(RTXPT-Port Phase 5.4): Emissive surfaces are gathered by BSDF sampling only; add emissive-triangle area-light NEE + MIS once an emissive light list exists.
```

Replace it (the closest hit now feeds emissive MIS; remaining gaps are textured-emissive NEE and one-sided/double-sided semantics):

```cpp
// TODO(RTXPT-Port Phase R2): Emissive triangles feed area-light NEE + MIS (constant emitters only). Textured
// emissive triangles stay BSDF-only, and emitters are two-sided rather than RTXPT-fork's one-sided TriangleLight.
```

- [ ] **Step 3: Retarget the raygen marker**

In `PathTracerSample.rgen`, the trailing marker currently reads:

```cpp
// TODO(RTXPT-Port Phase R2/R3/R4): Add emissive-triangle area lights, light importance sampling / RIS, and HDR environment-map MIS (NEE currently uses uniform light selection + a procedural-sky cosine env sampler).
```

Replace it (R2 is now done; R3/R4 remain):

```cpp
// TODO(RTXPT-Port Phase R3/R4): Add light importance sampling / RIS (uniform analytic + emissive selection today) and HDR environment-map importance sampling + MIS (procedural-sky cosine env sampler today).
```

- [ ] **Step 4: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/RTXPT_FORK_MAPPING.md \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "docs(rtxpt): map emissive-triangle symbols + retarget R2 TODO markers (R2/G4)"
```

**Verify:** Mapping doc has the new rows + divergence note; the `Phase 5.4` emissive marker is resolved and the raygen marker now names only R3/R4. No code behavior change. Build only if the user asks.

---

## Phase Completion Checklist

- [ ] Static glTF builds emissive triangles once at load (`RTXPTLights::UploadEmissiveTriangles` + `RTXPTEmissiveTrianglePass`) and binds the buffer to the raygen as `t_EmissiveTriangles`; dynamic/skinned glTF uses the existing current-geometry + dynamic BLAS path and never falls back to bind-pose data.
- [ ] Emissive NEE samples a triangle uniformly, converts area→solid-angle pdf, traces a (shortened) shadow ray, and applies power-heuristic MIS against the BSDF.
- [ ] Emissive BSDF hits are MIS-weighted via the closest-hit-precomputed `payload.emissiveLightPdf` × uniform selection pdf.
- [ ] **Unbiasedness:** toggling **Emissive mesh NEE + MIS** (or the master NEE toggle) off converges to the same image as on; an emissive-mesh-lit scene converges dramatically faster with it on; brightness does not increase (no double counting).
- [ ] `static_assert(sizeof(PathTracerConstants)==48)`, `sizeof(SampleConstants)==208)`, `sizeof(EmissiveTriangle)==64)` hold; `MaxPayloadSize` and the `PathPayload` comment agree at 80 bytes.
- [ ] Sample launches and renders on D3D12 and Vulkan.
- [ ] Static glTF uses the one-time emissive-triangle build; skinned/dynamic glTF remains aligned with the already-integrated current-frame skinned vertex buffer + dynamic BLAS path, and no bind-pose fallback is used.
- [ ] `Phase 5.4` emissive TODO resolved; raygen TODO retargeted to R3/R4; mapping doc records the new symbols + two-sided divergence.
- [ ] All commits are inside the `DiligentSamples` submodule working tree.

## Deferred to later phases (leave the markers)

- **Light importance sampling (RIS/WRS)** over emissive + analytic candidates — Phase R3 (G5); swaps only the uniform selection step in `SampleEmissiveNEE` and the analytic selector.
- **Dynamic emissive-triangle rebuild for animated glTF** — later emissive work, but the GPU skinning + dynamic BLAS prerequisite is already present; any extension must plug into the existing current-frame skinned buffer path instead of silently using bind-pose geometry.
- **Textured-emissive NEE** (spatially-varying emission) and **one-sided / double-sided baker semantics** — tracked by the `Phase R2` chit marker.
- **Face-normal shadow-ray origin + grazing-angle fadeout** — Phase R7 (G11); R2 reuses the existing shading-normal-offset `visibilityOrigin`.
