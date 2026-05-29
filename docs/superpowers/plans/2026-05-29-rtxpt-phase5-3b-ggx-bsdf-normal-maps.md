# RTXPT Phase 5.3b Metallic-Roughness GGX BSDF & Normal Mapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete the deferred core of Phase 5 shader layer 5 from the RTXPT port design — replace the cosine-weighted Lambertian bounce with a glTF metallic-roughness Cook-Torrance GGX BSDF (importance-sampled), add tangent-space normal mapping (with the tangent frame derived from UV gradients because vertex buffer 0 carries no tangents), and add Russian-roulette path termination using the already-reserved `MinBounces`.

**Architecture:** The reference path tracer already gives the closest-hit shader per-material GPU data and a bindless `g_MaterialTextures[]` table that holds **every** loaded GLTF texture (base color, metallic-roughness, normal, occlusion, emissive). This plan therefore needs **no new C++ resource binding**: it only (1) grows `RTXPTMaterialData` from 64 to 96 bytes to carry metallic-roughness + normal texture indices/slices and `NormalScale`, (2) samples those existing textures in the material bridge, (3) derives a per-triangle world-space tangent from positions + UVs in the scene bridge to apply the normal map, (4) carries `Metallic`/`Roughness` through the payload (reusing the two existing padding floats so the payload stays 64 bytes), and (5) moves shading from a flat Lambertian throughput multiply to a new `RTXPTBSDF.hlsli` that evaluates and importance-samples a two-lobe (diffuse + GGX specular) BSDF in raygen. The GGX path runs even when bindless textures are unavailable (it falls back to factor-only metallic/roughness), so the Phase 5.2/5.3 fallback chain is preserved. Russian roulette starts after `MinBounces` and is exposed in the UI.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs, DiligentTools `GLTFLoader` (`GLTF::Material::GetTextureId` / `GetTextureAttrib`, `GLTF::DefaultMetallicRoughnessTextureAttribId` = 1, `GLTF::DefaultNormalTextureAttribId` = 2, `ShaderAttribs::NormalScale` / `MetallicFactor` / `RoughnessFactor`), HLSL 6.5 ray tracing shaders compiled by DXC (`Texture2DArray[]` + `NonUniformResourceIndex` + `SampleLevel`, Cook-Torrance GGX: Trowbridge-Reitz NDF, Smith height-correlated visibility, Schlick Fresnel, cosine + NDF importance sampling), Dear ImGui.

---

## Scope Note: Phase 5 Sub-Plan Series

Phase 5 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md` enumerates nine shader dependency layers. Completed plans:

- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-1-shader-bridge.md` — layers 2-3 (shared declarations and scene/material bridge).
- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-2-reference-path-tracer.md` — layer 4 (reference path tracer core).
- `docs/superpowers/plans/2026-05-29-rtxpt-phase5-3-material-alpha-anyhit.md` — core of layer 5 (textured base color + emissive, alpha-test any-hit). It explicitly **deferred** the metallic-roughness GGX BSDF and normal mapping to a follow-up plan, preserved as `TODO(RTXPT-Port Phase 5.3)` markers.

This plan (Phase 5.3b) resolves that deferred layer-5 work: the **GGX metallic-roughness BSDF, normal mapping, and Russian roulette**. It still **defers** the following, each kept as a structured `TODO(RTXPT-Port Phase 5.3)` marker in code:

- `ALPHA_MODE_BLEND` stochastic transparency (only `ALPHA_MODE_MASK` is alpha-tested).
- Transmission, nested dielectrics, clearcoat, sheen, and the specular-glossiness workflow (the BSDF is opaque metallic-roughness only).
- Honoring `TextureShaderAttribs` UV selectors / wrap modes / atlas UV transform (assumes `TEXCOORD_0`, a single immutable wrap sampler, and the non-atlas identity transform — correct for the default bistro load).
- Per-material shader permutations / multi-record hit-group tables (a single uber closest-hit + any-hit pair is used).

Remaining Phase 5 sub-phases each get their own plan in a later session:

- Phase 5.4: Stable planes and realtime mode (layer 6).
- Phase 5.5: RTXDI shader bridge and passes (layer 7).
- Phase 5.6: NRD, denoising guides, post-process (layer 8).
- Phase 5.7: NVAPI, SER, OMM, DLSS-related shader variants (layer 9).

## Baseline

Current state of `DiligentSamples/Samples/RTXPT` (submodule `HEAD = 56343eca fix(rtxpt): bind material textures as array SRVs`, top-level `HEAD = 00da911`):

- `RTXPTMaterials::Upload(IRenderDevice*, const GLTF::Model&)` builds a `StructuredBuffer<RTXPTMaterialData>` (currently **64 bytes/entry**: `BaseColorFactor`, `EmissiveFactor`, `AlphaCutoff`, `Flags`, `BaseColorTextureIndex`, `EmissiveTextureIndex`, `MetallicFactor`, `RoughnessFactor`, `BaseColorTextureSlice`, `EmissiveTextureSlice`, `Padding0`) and collects **one 2D-array SRV per loaded GLTF texture** into `m_TextureViews` (owning) + `m_TextureBindings` (`IDeviceObject*`). The table holds **all** model textures, including metallic-roughness and normal maps — they are already bound, just not yet referenced by any material index. `GetTextureCount()` / `GetTextureBindings()` / `GetStats().TextureCount` exist. Flags: `kRTXPTMaterialFlag_HasBaseColorTexture = 0x1`, `_AlphaTested = 0x2`, `_HasEmissiveTexture = 0x4`. Helper `RTXPTMaterialIsAlphaTested(const GLTF::Material&)`.
- `RTXPTRayTracingPass::Initialize(...)` compiles `RTXPTReference.{rgen,rmiss,rchit,rahit}` (HLSL 6.5, DXC). When `EnableMaterialTextures` is on it defines `RTXPT_ENABLE_MATERIAL_TEXTURES` + `RTXPT_MATERIAL_TEXTURE_COUNT`, adds the any-hit to the hit group, declares `Texture2DArray g_MaterialTextures[N]` (MUTABLE) + immutable `g_MaterialSampler`, and binds the table via `SetArray`. The closest-hit/any-hit STATIC binds are `g_Materials`, `g_SubInstanceData`, `g_VertexBuffer`, `g_IndexBuffer`. `MaxPayloadSize = sizeof(float)*16` (64 bytes), `MaxAttributeSize = sizeof(float)*2`, `MaxRecursionDepth = 1`.
- `RTXPTReference.rgen` runs an N-bounce loop, `TraceRay(..., RAY_FLAG_NONE, ...)`, accumulates into RGBA32F `g_AccumColor`, tone-maps into RGBA8 `g_OutputColor`. The bounce is **cosine-weighted Lambertian** (`Throughput *= Payload.BaseColor`). `RTXPTReference.rchit` fills `RTXPTPathTracerPayload` with world pos/normal + textured base color + emissive. `RTXPTReference.rmiss` writes a procedural sky into `Payload.Emission` (does **not** write the payload padding fields).
- `RTXPTReference.rgen` includes `RTXPTSceneBridge.hlsli` + `RTXPTRandom.hlsli`. `RTXPTRandom.hlsli` provides `RTXPTRandom`, `NextFloat`, `NextFloat2`, `BuildOrthonormalBasis`, `SampleCosineHemisphere`.
- `RTXPTShaderShared.hlsli` mirrors `RTXPTMaterialData` (64-byte), the `kRTXPTMaterialFlag*` constants, `RTXPTPathTracerSettings` (`MaxBounces`, `AccumulationFrame`, `ResetAccumulation`, `MinBounces`), and `RTXPTPathTracerPayload` (with two trailing padding floats `Padding0`/`Padding1`). `RTXPTSceneBridge.hlsli` declares the hit helpers (guarded by `RTXPT_ENABLE_HIT_BRIDGE`): `GetTriangleVertices`, `InterpolateNormal`, `ComputeGeometricNormal`, `ComputeWorldHitPosition`, `InterpolateTexCoord`. `RTXPTMaterialBridge.hlsli` declares `g_Materials`, the bindless `g_MaterialTextures[]` + `g_MaterialSampler` (gated by `RTXPT_ENABLE_MATERIAL_TEXTURES`), `SampleMaterialTexture`, `GetBaseColor`, `GetEmission`, `AlphaTestPasses`, each with factor-only `#else` fallbacks.
- `RTXPTSample` owns `RTXPTPathTracerSettings` with `MinBounces` currently forced to `0` in `UpdateFrameConstants`. The UI has a "Max bounces" slider (`m_MaxBounces`, range 1-16) and a "Reset accumulation" button. `RTXPTFrameConstants` is 176 bytes; `RTXPTPathTracerSettings` is 16 bytes (`static_assert`).
- `RTXPTScene` vertex buffer 0 packs **POSITION + NORMAL + TEXCOORD_0 only** (`RTXPTVertex`, 32 bytes) — there is **no tangent attribute**, so normal mapping must derive a tangent from triangle edges + UV deltas.
- GLTF loader constants (`DiligentTools/AssetLoader/interface/GLTFLoader.hpp`): `DefaultMetallicRoughnessTextureAttribId = 1`, `DefaultNormalTextureAttribId = 2`, `DefaultEmissiveTextureAttribId = 4`. `ShaderAttribs` has `NormalScale` (default 1), `MetallicFactor`, `RoughnessFactor`. glTF metallic-roughness texture packing is **roughness in `.g`, metallic in `.b`**. The loader creates base-color/emissive textures as sRGB (hardware-decoded to linear on sample) and metallic-roughness/normal as linear UNORM — so a single linear sampler is correct for all of them.

This plan assumes the top-level repository starts clean and the `DiligentSamples` submodule is at the state above.

---

## Scope

This plan implements:

- Grow `RTXPTMaterialData` (C++ `RTXPTMaterials.hpp`) from 64 to **96 bytes**: add `MetallicRoughnessTextureIndex` / `MetallicRoughnessTextureSlice`, `NormalTextureIndex` / `NormalTextureSlice` / `NormalScale`, three extra padding floats, and the `kRTXPTMaterialFlag_HasMetallicRoughnessTexture` (`0x8`) / `_HasNormalTexture` (`0x10`) flags. Populate them in `Upload` from `GLTF::DefaultMetallicRoughnessTextureAttribId` / `DefaultNormalTextureAttribId` and `Attribs.NormalScale`.
- Mirror the 96-byte `RTXPTMaterialData` + new flag constants in `RTXPTShaderShared.hlsli`, and rename the payload's two trailing padding floats to `Metallic` / `Roughness` (payload size unchanged at 64 bytes).
- Add `RTXPTBSDF.hlsli`: a glTF metallic-roughness Cook-Torrance GGX BSDF with `RTXPTMakeSurface`, `RTXPTEvalBSDF`, and `RTXPTSampleBSDF` (two-lobe diffuse + specular, single-sample MIS pdf). Register it in `CMakeLists.txt`.
- Extend the bridges (macro-gated, with factor-only `#else` fallbacks): `Bridge::GetMetallicRoughness` and `Bridge::GetTangentNormal` in the material bridge; `Bridge::ComputeWorldTangent` (UV-gradient-derived world tangent) in the scene bridge.
- Update `RTXPTReference.rchit` to sample metallic-roughness, apply the normal map, and write `Metallic`/`Roughness` into the payload.
- Update `RTXPTReference.rgen` to build the surface from the payload, importance-sample the GGX BSDF for each bounce, and apply Russian roulette after `MinBounces`.
- Wire `MinBounces` through `RTXPTSample::UpdateFrameConstants` and add a "Min bounces (RR start)" UI slider; re-target the resolved Phase 5.3 TODO markers.

This plan intentionally does **not**:

- Implement transmission, nested dielectrics, clearcoat, sheen, anisotropy, or the specular-glossiness workflow.
- Implement `ALPHA_MODE_BLEND` / stochastic transparency, or alpha test on shadow/visibility rays (there are no shadow rays yet — NEE is Phase 5.5).
- Honor `TextureShaderAttribs` UV selectors, wrap modes, or atlas UV transform beyond the non-atlas identity case.
- Add per-material shader permutations or a multi-record hit-group table.
- Change the RT-pass C++ binding model, the acceleration-structure contract, the payload size, or `MaxRecursionDepth` — none of these change.
- Run automated builds or runtime execution; build/runtime steps are listed for explicit user request only.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` — grow `RTXPTMaterialData` to 96 bytes; add MR/normal flags.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` — populate MR/normal indices, slices, and `NormalScale` in `Upload`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli` — mirror the 96-byte struct + flags; rename payload padding to `Metallic`/`Roughness`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli` — GGX metallic-roughness BSDF.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` — register `RTXPTBSDF.hlsli`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli` — add `Bridge::ComputeWorldTangent`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli` — add `Bridge::GetMetallicRoughness` / `GetTangentNormal`; drop the resolved GGX/normal TODO.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit` — sample MR, apply normal map, output metallic/roughness.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen` — GGX BSDF bounce + Russian roulette.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` — add `m_MinBounces` member.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` — wire `MinBounces`, add the UI slider, re-target the Phase 5.3 TODO.

---

### Task 0: Phase 5.3 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [ ] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected: branch line `## RTXPT...origin/RTXPT` and no staged/modified files under `DiligentSamples/Samples/RTXPT` or `docs/superpowers/plans`. Unrelated files may be left untouched.

- [ ] **Step 2: Confirm DiligentSamples Phase 5.3 state**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentSamples log --oneline -n 9
```

Expected: clean working tree; the most recent commit is `56343eca fix(rtxpt): bind material textures as array SRVs`, above the seven `feat(rtxpt): ... phase 5.3 ...` / `fix` commits.

- [ ] **Step 3: Confirm the BSDF file does not yet exist**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli
```

Expected: `False`. If it already exists, inspect it before overwriting and preserve unrelated work.

- [ ] **Step 4: Confirm the resolved-by-this-plan TODO markers are present**

Run:

```powershell
rg -n "metallic-roughness GGX BSDF|metallic-roughness/normal-map|single-lobe Lambertian|Shade with the metallic-roughness" DiligentSamples/Samples/RTXPT
```

Expected matches in: `RTXPTMaterialBridge.hlsli` (1, "Shade with the metallic-roughness GGX BSDF and normal maps"), `RTXPTReference.rchit` (1, "metallic-roughness/normal-map shading"), `RTXPTReference.rgen` (1, "single-lobe Lambertian"), `RTXPTSample.cpp` (1, "metallic-roughness GGX BSDF + normal maps"). This plan removes/re-targets these.

---

### Task 1: Grow The GPU Material Data With MR + Normal Texture Indices

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`

Context: the bindless texture table already holds the metallic-roughness and normal textures (they are part of `Model.GetTexture(i)`), so this task adds only the per-material **indices/slices/flags** that point into that table. `GLTF::DefaultMetallicRoughnessTextureAttribId == 1`, `GLTF::DefaultNormalTextureAttribId == 2`. The struct grows from 64 to 96 bytes (six 16-byte rows).

- [ ] **Step 1: Replace the `RTXPTMaterialData` struct and flag constants in `RTXPTMaterials.hpp`**

In `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`, locate this block (the struct, its `static_assert`, and the three flag constants):

```cpp
// GPU material record consumed by the reference path tracer (mirrors RTXPTMaterialData in RTXPTShaderShared.hlsli).
// One entry per GLTF material; the closest-hit / any-hit shaders index it via RTXPTSubInstanceData::MaterialID.
struct RTXPTMaterialData
{
    float4 BaseColorFactor = float4{1, 1, 1, 1};

    float3 EmissiveFactor = float3{0, 0, 0};
    float  AlphaCutoff    = 0.5f;

    Uint32 Flags                 = 0;
    Uint32 BaseColorTextureIndex = 0;
    Uint32 EmissiveTextureIndex  = 0;
    float  MetallicFactor        = 1.0f;

    float RoughnessFactor       = 1.0f;
    float BaseColorTextureSlice = 0.0f;
    float EmissiveTextureSlice  = 0.0f;
    float Padding0              = 0.0f;
};
static_assert(sizeof(RTXPTMaterialData) == 64, "RTXPTMaterialData layout must match RTXPTShaderShared.hlsli");

// Flag bits for RTXPTMaterialData::Flags. Keep in sync with kRTXPTMaterialFlag* in RTXPTShaderShared.hlsli.
constexpr Uint32 kRTXPTMaterialFlag_HasBaseColorTexture = 0x1u;
constexpr Uint32 kRTXPTMaterialFlag_AlphaTested         = 0x2u;
constexpr Uint32 kRTXPTMaterialFlag_HasEmissiveTexture  = 0x4u;
```

Replace it with:

```cpp
// GPU material record consumed by the reference path tracer (mirrors RTXPTMaterialData in RTXPTShaderShared.hlsli).
// One entry per GLTF material; the closest-hit / any-hit shaders index it via RTXPTSubInstanceData::MaterialID.
// All texture indices/slices reference the shared bindless material-texture table (one entry per GLTF texture).
struct RTXPTMaterialData
{
    float4 BaseColorFactor = float4{1, 1, 1, 1};

    float3 EmissiveFactor = float3{0, 0, 0};
    float  AlphaCutoff    = 0.5f;

    Uint32 Flags                 = 0;
    Uint32 BaseColorTextureIndex = 0;
    Uint32 EmissiveTextureIndex  = 0;
    float  MetallicFactor        = 1.0f;

    float  RoughnessFactor               = 1.0f;
    float  BaseColorTextureSlice         = 0.0f;
    float  EmissiveTextureSlice          = 0.0f;
    Uint32 MetallicRoughnessTextureIndex = 0;

    float  MetallicRoughnessTextureSlice = 0.0f;
    Uint32 NormalTextureIndex            = 0;
    float  NormalTextureSlice            = 0.0f;
    float  NormalScale                   = 1.0f;

    float Padding0 = 0.0f;
    float Padding1 = 0.0f;
    float Padding2 = 0.0f;
    float Padding3 = 0.0f;
};
static_assert(sizeof(RTXPTMaterialData) == 96, "RTXPTMaterialData layout must match RTXPTShaderShared.hlsli");

// Flag bits for RTXPTMaterialData::Flags. Keep in sync with kRTXPTMaterialFlag* in RTXPTShaderShared.hlsli.
constexpr Uint32 kRTXPTMaterialFlag_HasBaseColorTexture         = 0x1u;
constexpr Uint32 kRTXPTMaterialFlag_AlphaTested                 = 0x2u;
constexpr Uint32 kRTXPTMaterialFlag_HasEmissiveTexture          = 0x4u;
constexpr Uint32 kRTXPTMaterialFlag_HasMetallicRoughnessTexture = 0x8u;
constexpr Uint32 kRTXPTMaterialFlag_HasNormalTexture            = 0x10u;
```

- [ ] **Step 2: Populate the MR + normal fields in `RTXPTMaterials.cpp`**

In `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`, locate the emissive-texture block followed by the alpha-test block inside the material loop:

```cpp
        const int EmissiveTextureId = Material.GetTextureId(GLTF::DefaultEmissiveTextureAttribId);
        if (EmissiveTextureId >= 0 && static_cast<Uint32>(EmissiveTextureId) < ValidTextureCount)
        {
            Data.Flags |= kRTXPTMaterialFlag_HasEmissiveTexture;
            Data.EmissiveTextureIndex = static_cast<Uint32>(EmissiveTextureId);
            Data.EmissiveTextureSlice = Material.GetTextureAttrib(GLTF::DefaultEmissiveTextureAttribId).TextureSlice;
        }

        // Alpha test requires the base-color texture (its .a channel). Only set the flag when both agree.
        if (RTXPTMaterialIsAlphaTested(Material) && (Data.Flags & kRTXPTMaterialFlag_HasBaseColorTexture) != 0u)
            Data.Flags |= kRTXPTMaterialFlag_AlphaTested;
```

Replace it with:

```cpp
        const int EmissiveTextureId = Material.GetTextureId(GLTF::DefaultEmissiveTextureAttribId);
        if (EmissiveTextureId >= 0 && static_cast<Uint32>(EmissiveTextureId) < ValidTextureCount)
        {
            Data.Flags |= kRTXPTMaterialFlag_HasEmissiveTexture;
            Data.EmissiveTextureIndex = static_cast<Uint32>(EmissiveTextureId);
            Data.EmissiveTextureSlice = Material.GetTextureAttrib(GLTF::DefaultEmissiveTextureAttribId).TextureSlice;
        }

        const int MetallicRoughnessTextureId = Material.GetTextureId(GLTF::DefaultMetallicRoughnessTextureAttribId);
        if (MetallicRoughnessTextureId >= 0 && static_cast<Uint32>(MetallicRoughnessTextureId) < ValidTextureCount)
        {
            Data.Flags |= kRTXPTMaterialFlag_HasMetallicRoughnessTexture;
            Data.MetallicRoughnessTextureIndex = static_cast<Uint32>(MetallicRoughnessTextureId);
            Data.MetallicRoughnessTextureSlice = Material.GetTextureAttrib(GLTF::DefaultMetallicRoughnessTextureAttribId).TextureSlice;
        }

        const int NormalTextureId = Material.GetTextureId(GLTF::DefaultNormalTextureAttribId);
        if (NormalTextureId >= 0 && static_cast<Uint32>(NormalTextureId) < ValidTextureCount)
        {
            Data.Flags |= kRTXPTMaterialFlag_HasNormalTexture;
            Data.NormalTextureIndex = static_cast<Uint32>(NormalTextureId);
            Data.NormalTextureSlice = Material.GetTextureAttrib(GLTF::DefaultNormalTextureAttribId).TextureSlice;
        }
        Data.NormalScale = Attribs.NormalScale;

        // Alpha test requires the base-color texture (its .a channel). Only set the flag when both agree.
        if (RTXPTMaterialIsAlphaTested(Material) && (Data.Flags & kRTXPTMaterialFlag_HasBaseColorTexture) != 0u)
            Data.Flags |= kRTXPTMaterialFlag_AlphaTested;
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the GPU material data growth**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.3 metallic-roughness and normal texture indices" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two material files.

---

### Task 2: Mirror The 96-Byte Material Layout And Rename Payload Padding

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`

- [ ] **Step 1: Replace the `RTXPTMaterialData` mirror and flag constants**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`, locate:

```hlsl
// Mirrors Diligent::RTXPTMaterialData in RTXPTMaterials.hpp (must keep order/size in sync; total size 64 bytes).
struct RTXPTMaterialData
{
    float4 BaseColorFactor; // offset 0

    float3 EmissiveFactor; // offset 16
    float  AlphaCutoff;    // offset 28

    uint  Flags;                 // offset 32
    uint  BaseColorTextureIndex; // offset 36
    uint  EmissiveTextureIndex;  // offset 40
    float MetallicFactor;        // offset 44

    float RoughnessFactor;       // offset 48
    float BaseColorTextureSlice; // offset 52
    float EmissiveTextureSlice;  // offset 56
    float Padding0;              // offset 60
};

// Mirrors the kRTXPTMaterialFlag_* constants in RTXPTMaterials.hpp.
static const uint kRTXPTMaterialFlagHasBaseColorTexture = 0x1u;
static const uint kRTXPTMaterialFlagAlphaTested         = 0x2u;
static const uint kRTXPTMaterialFlagHasEmissiveTexture  = 0x4u;
```

Replace it with:

```hlsl
// Mirrors Diligent::RTXPTMaterialData in RTXPTMaterials.hpp (must keep order/size in sync; total size 96 bytes).
struct RTXPTMaterialData
{
    float4 BaseColorFactor; // offset 0

    float3 EmissiveFactor; // offset 16
    float  AlphaCutoff;    // offset 28

    uint  Flags;                 // offset 32
    uint  BaseColorTextureIndex; // offset 36
    uint  EmissiveTextureIndex;  // offset 40
    float MetallicFactor;        // offset 44

    float RoughnessFactor;               // offset 48
    float BaseColorTextureSlice;         // offset 52
    float EmissiveTextureSlice;          // offset 56
    uint  MetallicRoughnessTextureIndex; // offset 60

    float MetallicRoughnessTextureSlice; // offset 64
    uint  NormalTextureIndex;            // offset 68
    float NormalTextureSlice;            // offset 72
    float NormalScale;                   // offset 76

    float Padding0; // offset 80
    float Padding1; // offset 84
    float Padding2; // offset 88
    float Padding3; // offset 92
};

// Mirrors the kRTXPTMaterialFlag_* constants in RTXPTMaterials.hpp.
static const uint kRTXPTMaterialFlagHasBaseColorTexture         = 0x1u;
static const uint kRTXPTMaterialFlagAlphaTested                 = 0x2u;
static const uint kRTXPTMaterialFlagHasEmissiveTexture          = 0x4u;
static const uint kRTXPTMaterialFlagHasMetallicRoughnessTexture = 0x8u;
static const uint kRTXPTMaterialFlagHasNormalTexture            = 0x10u;
```

- [ ] **Step 2: Rename the payload padding floats to Metallic / Roughness**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`, locate the payload doc comment and struct:

```hlsl
// Reference path tracer payload (Phase 5.2).
//   HitFlag    : 1 on closest hit, 0 on miss.
//   HitDistance: RayTCurrent() on hit; <= 0 on miss.
//   WorldPos   : world-space hit position.
//   WorldNormal: world-space shading normal (interpolated and renormalized).
//   BaseColor  : material base color RGB (sampled via the material bridge).
//   Emission   : RGB emission written by miss/emissive paths and accumulated by raygen.
struct RTXPTPathTracerPayload
{
    float3 WorldPos;
    float  HitDistance;

    float3 WorldNormal;
    uint   HitFlag;

    float3 BaseColor;
    float  Padding0;

    float3 Emission;
    float  Padding1;
};
```

Replace it with:

```hlsl
// Reference path tracer payload (Phase 5.2 / 5.3). Size is 64 bytes (16 floats); do not grow without
// updating RTXPTRayTracingPass::Initialize MaxPayloadSize.
//   HitFlag    : 1 on closest hit, 0 on miss.
//   HitDistance: RayTCurrent() on hit; <= 0 on miss.
//   WorldPos   : world-space hit position.
//   WorldNormal: world-space shading normal (interpolated, normal-mapped, renormalized).
//   BaseColor  : material base color RGB (sampled via the material bridge).
//   Metallic   : glTF metallic value at the hit (factor * texture .b).
//   Emission   : RGB emission written by miss/emissive paths and accumulated by raygen.
//   Roughness  : glTF perceptual roughness at the hit (factor * texture .g).
struct RTXPTPathTracerPayload
{
    float3 WorldPos;
    float  HitDistance;

    float3 WorldNormal;
    uint   HitFlag;

    float3 BaseColor;
    float  Metallic;

    float3 Emission;
    float  Roughness;
};
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the shared header update**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): mirror phase 5.3 96-byte material data and payload metallic/roughness" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the shared header.

---

### Task 3: Add The Metallic-Roughness GGX BSDF Helper

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

Context: a standard glTF metallic-roughness Cook-Torrance model — Trowbridge-Reitz (GGX) NDF, Smith height-correlated visibility, Schlick Fresnel — split into a Lambertian diffuse lobe and a GGX specular lobe. `RTXPTSampleBSDF` stochastically picks a lobe, samples a direction, and returns the throughput weight `f * NoL / pdf` using a single-sample MIS pdf. The file depends only on `RTXPTRandom.hlsli` (for `RTXPTRandom`, `NextFloat`, `NextFloat2`, `BuildOrthonormalBasis`, `SampleCosineHemisphere`); it operates on plain floats from the payload, not on `RTXPTMaterialData`.

- [ ] **Step 1: Create `RTXPTBSDF.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`:

```hlsl
#ifndef RTXPT_BSDF_HLSLI
#define RTXPT_BSDF_HLSLI

// glTF 2.0 metallic-roughness Cook-Torrance BSDF for the reference path tracer.
// Two lobes: Lambertian diffuse + GGX specular. Evaluation and importance sampling live here so
// raygen only needs base color / metallic / roughness / shading normal from the payload.

#include "RTXPTRandom.hlsli"

static const float RTXPT_PI            = 3.14159265358979323846;
static const float RTXPT_INV_PI        = 0.31830988618379067154;
// Clamp roughness away from a perfect mirror: a zero-roughness GGX lobe is a delta we cannot importance sample.
static const float RTXPT_MIN_ROUGHNESS = 0.045;

// Shading inputs resolved into the form the lobes consume.
struct RTXPTSurface
{
    float3 N;             // shading normal (world space, unit)
    float3 DiffuseAlbedo; // Lambertian albedo (base color scaled by (1 - metallic))
    float3 F0;            // specular reflectance at normal incidence
    float  Alpha;         // GGX alpha = roughness^2
};

RTXPTSurface RTXPTMakeSurface(float3 N, float3 BaseColor, float Metallic, float Roughness)
{
    const float R = clamp(Roughness, RTXPT_MIN_ROUGHNESS, 1.0);

    RTXPTSurface S;
    S.N             = N;
    S.Alpha         = R * R;
    S.DiffuseAlbedo = BaseColor * (1.0 - Metallic);
    S.F0            = lerp(float3(0.04, 0.04, 0.04), BaseColor, Metallic);
    return S;
}

float3 RTXPTFresnelSchlick(float3 F0, float VoH)
{
    const float F = pow(saturate(1.0 - VoH), 5.0);
    return F0 + (1.0 - F0) * F;
}

// Trowbridge-Reitz (GGX) normal distribution function.
float RTXPTDistributionGGX(float NoH, float Alpha)
{
    const float A2 = Alpha * Alpha;
    const float D  = (NoH * NoH) * (A2 - 1.0) + 1.0;
    return A2 / max(RTXPT_PI * D * D, 1e-7);
}

// Smith height-correlated visibility term = G / (4 * NoV * NoL).
float RTXPTVisibilitySmithGGX(float NoV, float NoL, float Alpha)
{
    const float A2 = Alpha * Alpha;
    const float V  = NoL * sqrt(NoV * NoV * (1.0 - A2) + A2);
    const float L  = NoV * sqrt(NoL * NoL * (1.0 - A2) + A2);
    return 0.5 / max(V + L, 1e-7);
}

float RTXPTLuminance(float3 C)
{
    return dot(C, float3(0.2126, 0.7152, 0.0722));
}

// Evaluate f(Wo,Wi) * NoL and the single-sample MIS pdf for the given (unit, away-from-surface) directions.
// SpecProb is the probability the sampler used to pick the specular lobe.
void RTXPTEvalBSDF(RTXPTSurface S, float3 Wo, float3 Wi, float SpecProb, out float3 FTimesNoL, out float Pdf)
{
    FTimesNoL = float3(0.0, 0.0, 0.0);
    Pdf       = 0.0;

    const float NoL = dot(S.N, Wi);
    const float NoV = dot(S.N, Wo);
    if (NoL <= 0.0 || NoV <= 0.0)
        return;

    const float3 H   = normalize(Wo + Wi);
    const float  NoH = saturate(dot(S.N, H));
    const float  VoH = saturate(dot(Wo, H));

    const float  D    = RTXPTDistributionGGX(NoH, S.Alpha);
    const float  Vis  = RTXPTVisibilitySmithGGX(NoV, NoL, S.Alpha);
    const float3 F    = RTXPTFresnelSchlick(S.F0, VoH);
    const float3 Spec = D * Vis * F; // Vis already carries 1 / (4 NoV NoL)

    const float3 Diff = S.DiffuseAlbedo * RTXPT_INV_PI * (1.0 - F);

    FTimesNoL = (Diff + Spec) * NoL;

    const float PdfDiffuse  = NoL * RTXPT_INV_PI;
    const float PdfSpecular = D * NoH / max(4.0 * VoH, 1e-7);
    Pdf = SpecProb * PdfSpecular + (1.0 - SpecProb) * PdfDiffuse;
}

// Importance-sample an incident direction Wi. Returns false for invalid samples.
// Weight = f(Wo,Wi) * NoL / pdf is the throughput multiplier the path tracer applies.
bool RTXPTSampleBSDF(RTXPTSurface S, float3 Wo, inout RTXPTRandom Rng,
                     out float3 Wi, out float3 Weight, out float Pdf)
{
    Wi     = float3(0.0, 0.0, 0.0);
    Weight = float3(0.0, 0.0, 0.0);
    Pdf    = 0.0;

    const float NoV = dot(S.N, Wo);
    if (NoV <= 0.0)
        return false;

    // Pick the lobe from the Fresnel-weighted specular vs diffuse luminance, clamped so neither lobe starves.
    const float3 Fapprox = RTXPTFresnelSchlick(S.F0, NoV);
    const float  SpecLum = RTXPTLuminance(Fapprox);
    const float  DiffLum = RTXPTLuminance(S.DiffuseAlbedo * (1.0 - Fapprox));
    const float  SpecProb = clamp(SpecLum / max(SpecLum + DiffLum, 1e-4), 0.1, 0.9);

    float3 Tangent;
    float3 Bitangent;
    BuildOrthonormalBasis(S.N, Tangent, Bitangent);

    const float2 Rand2 = NextFloat2(Rng);
    const float  Lobe  = NextFloat(Rng);

    if (Lobe < SpecProb)
    {
        // GGX half-vector (NDF) sampling in the local frame, then reflect Wo about H.
        const float A    = S.Alpha;
        const float Phi  = 2.0 * RTXPT_PI * Rand2.x;
        const float CosT = sqrt((1.0 - Rand2.y) / max(1.0 + (A * A - 1.0) * Rand2.y, 1e-7));
        const float SinT = sqrt(max(0.0, 1.0 - CosT * CosT));

        const float3 HLocal = float3(SinT * cos(Phi), SinT * sin(Phi), CosT);
        const float3 H      = normalize(Tangent * HLocal.x + Bitangent * HLocal.y + S.N * HLocal.z);
        Wi                  = reflect(-Wo, H);
        if (dot(S.N, Wi) <= 0.0)
            return false;
    }
    else
    {
        // Cosine-weighted diffuse hemisphere sample.
        float PdfUnused;
        Wi = SampleCosineHemisphere(Rand2, S.N, PdfUnused);
        if (dot(S.N, Wi) <= 0.0)
            return false;
    }

    float3 FTimesNoL;
    RTXPTEvalBSDF(S, Wo, Wi, SpecProb, FTimesNoL, Pdf);
    if (Pdf <= 0.0)
        return false;

    Weight = FTimesNoL / Pdf;
    return true;
}

#endif // RTXPT_BSDF_HLSLI
```

- [ ] **Step 2: Register `RTXPTBSDF.hlsli` in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, locate:

```cmake
    assets/shaders/RTXPTMaterialBridge.hlsli
    assets/shaders/RTXPTRandom.hlsli
```

Replace it with:

```cmake
    assets/shaders/RTXPTMaterialBridge.hlsli
    assets/shaders/RTXPTRandom.hlsli
    assets/shaders/RTXPTBSDF.hlsli
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli
```

Expected: no output and exit code 0 for both (the BSDF file is new/untracked — the second command produces no output regardless; it is included for symmetry).

- [ ] **Step 4: Commit the BSDF helper**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.3 metallic-roughness ggx bsdf helper" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the new BSDF file and the CMake registration.

---

### Task 4: Add MR + Normal Sampling And UV-Derived Tangents To The Bridges

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`

Context: vertex buffer 0 has no tangent, so `ComputeWorldTangent` derives one from triangle edges + UV deltas (the standard `T = (E1*dU2.y - E2*dU1.y) / det` formula), transforms it to world space with the same object-to-world matrix used for positions/normals, and orthonormalizes it against the shading normal. Degenerate UVs fall back to an arbitrary perpendicular so the frame is always valid.

- [ ] **Step 1: Add `Bridge::ComputeWorldTangent` to the scene bridge**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`, locate the `InterpolateTexCoord` helper that closes the `RTXPT_ENABLE_HIT_BRIDGE` section:

```hlsl
    // Barycentric-interpolated TEXCOORD_0 for the current closest-hit / any-hit triangle.
    float2 InterpolateTexCoord(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        return V0.TexCoord0 * Bary.x + V1.TexCoord0 * Bary.y + V2.TexCoord0 * Bary.z;
    }
#endif
```

Replace it with:

```hlsl
    // Barycentric-interpolated TEXCOORD_0 for the current closest-hit / any-hit triangle.
    float2 InterpolateTexCoord(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        return V0.TexCoord0 * Bary.x + V1.TexCoord0 * Bary.y + V2.TexCoord0 * Bary.z;
    }

    // World-space tangent derived from triangle edges + UV deltas (vertex buffer 0 carries no tangent attribute).
    // Returned tangent is orthonormalized against WorldNormal; .w is the bitangent handedness (always +1 here).
    // Degenerate UVs fall back to an arbitrary perpendicular so the TBN frame is always valid.
    float4 ComputeWorldTangent(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float3 WorldNormal)
    {
        const float3 E1  = V1.Position - V0.Position;
        const float3 E2  = V2.Position - V0.Position;
        const float2 dU1 = V1.TexCoord0 - V0.TexCoord0;
        const float2 dU2 = V2.TexCoord0 - V0.TexCoord0;
        const float  Det = dU1.x * dU2.y - dU2.x * dU1.y;

        // Fallback perpendicular for degenerate UVs (det ~ 0): cross with the least-aligned axis.
        const float3 Axis     = abs(WorldNormal.x) > 0.9 ? float3(0.0, 1.0, 0.0) : float3(1.0, 0.0, 0.0);
        float3       Fallback = normalize(cross(Axis, WorldNormal));

        if (abs(Det) < 1e-12)
            return float4(Fallback, 1.0);

        const float3 ObjTangent   = (E1 * dU2.y - E2 * dU1.y) * (1.0 / Det);
        float3       WorldTangent = mul((float3x3)ObjectToWorld3x4(), ObjTangent);

        // Gram-Schmidt against the (already world-space) shading normal.
        WorldTangent = WorldTangent - WorldNormal * dot(WorldNormal, WorldTangent);
        const float Len = length(WorldTangent);
        return Len > 1e-8 ? float4(WorldTangent / Len, 1.0) : float4(Fallback, 1.0);
    }
#endif
```

- [ ] **Step 2: Add `Bridge::GetMetallicRoughness` and `Bridge::GetTangentNormal` to the material bridge**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`, locate the textured `AlphaTestPasses` helper that closes the `#ifdef RTXPT_ENABLE_MATERIAL_TEXTURES` block, the factor-only `#else` block, and the trailing GGX TODO:

```hlsl
    // True when the hit passes the alpha test (or is not alpha tested).
    bool AlphaTestPasses(RTXPTMaterialData Material, float2 UV)
    {
        if ((Material.Flags & kRTXPTMaterialFlagAlphaTested) == 0u)
            return true;
        return GetBaseColor(Material, UV).a >= Material.AlphaCutoff;
    }
#else
    // Factor-only fallback (bindless material textures unavailable): no texture sampling, never alpha tested.
    float4 GetBaseColor(RTXPTMaterialData Material, float2 UV) { return Material.BaseColorFactor; }
    float3 GetEmission(RTXPTMaterialData Material, float2 UV) { return Material.EmissiveFactor; }
    bool   AlphaTestPasses(RTXPTMaterialData Material, float2 UV) { return true; }
#endif
} // namespace Bridge

// TODO(RTXPT-Port Phase 5.3): Shade with the metallic-roughness GGX BSDF and normal maps instead of textured Lambertian.
// TODO(RTXPT-Port Phase 5.3): Honor TextureShaderAttribs UV selectors / wrap modes / atlas transform (currently assumes TEXCOORD_0 + wrap + slice).
```

Replace it with:

```hlsl
    // True when the hit passes the alpha test (or is not alpha tested).
    bool AlphaTestPasses(RTXPTMaterialData Material, float2 UV)
    {
        if ((Material.Flags & kRTXPTMaterialFlagAlphaTested) == 0u)
            return true;
        return GetBaseColor(Material, UV).a >= Material.AlphaCutoff;
    }

    // glTF metallic-roughness packing: roughness in .g, metallic in .b, each scaled by the material factor.
    float2 GetMetallicRoughness(RTXPTMaterialData Material, float2 UV)
    {
        float Metallic  = Material.MetallicFactor;
        float Roughness = Material.RoughnessFactor;
        if ((Material.Flags & kRTXPTMaterialFlagHasMetallicRoughnessTexture) != 0u)
        {
            const float4 MR = SampleMaterialTexture(Material.MetallicRoughnessTextureIndex, Material.MetallicRoughnessTextureSlice, UV);
            Roughness *= MR.g;
            Metallic  *= MR.b;
        }
        return float2(Metallic, Roughness);
    }

    // Tangent-space normal unpacked to [-1, 1] with NormalScale applied to xy. Returns (0,0,1) when there is no
    // normal map, which the caller treats as "no perturbation".
    float3 GetTangentNormal(RTXPTMaterialData Material, float2 UV)
    {
        if ((Material.Flags & kRTXPTMaterialFlagHasNormalTexture) == 0u)
            return float3(0.0, 0.0, 1.0);

        float3 N = SampleMaterialTexture(Material.NormalTextureIndex, Material.NormalTextureSlice, UV).xyz * 2.0 - 1.0;
        N.xy *= Material.NormalScale;
        return normalize(N);
    }
#else
    // Factor-only fallback (bindless material textures unavailable): no texture sampling, never alpha tested.
    float4 GetBaseColor(RTXPTMaterialData Material, float2 UV) { return Material.BaseColorFactor; }
    float3 GetEmission(RTXPTMaterialData Material, float2 UV) { return Material.EmissiveFactor; }
    bool   AlphaTestPasses(RTXPTMaterialData Material, float2 UV) { return true; }
    float2 GetMetallicRoughness(RTXPTMaterialData Material, float2 UV) { return float2(Material.MetallicFactor, Material.RoughnessFactor); }
    float3 GetTangentNormal(RTXPTMaterialData Material, float2 UV) { return float3(0.0, 0.0, 1.0); }
#endif
} // namespace Bridge

// TODO(RTXPT-Port Phase 5.3): Honor TextureShaderAttribs UV selectors / wrap modes / atlas transform (currently assumes TEXCOORD_0 + wrap + slice).
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the bridge helpers**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.3 metallic-roughness, normal-map, and tangent bridge helpers" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two bridge headers.

---

### Task 5: Sample Metallic-Roughness And Apply The Normal Map In Closest Hit

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`

- [ ] **Step 1: Rewrite the closest-hit shader**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit` with:

```hlsl
#define RTXPT_ENABLE_HIT_BRIDGE 1
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTMaterialBridge.hlsli"

[shader("closesthit")]
void main(inout RTXPTPathTracerPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
    Payload.HitFlag     = 1u;
    Payload.HitDistance = RayTCurrent();
    Payload.Emission    = float3(0.0, 0.0, 0.0);

    // Default to a barycentric debug color so we still see something if the bridge tables are unbound.
    float3 BaseColor   = float3(Attributes.barycentrics.x,
                                Attributes.barycentrics.y,
                                1.0 - Attributes.barycentrics.x - Attributes.barycentrics.y);
    float3 WorldNormal = -WorldRayDirection();
    float3 WorldPos    = WorldRayOrigin() + WorldRayDirection() * RayTCurrent();
    float  Metallic    = 0.0;
    float  Roughness   = 1.0;

    if (Bridge::HasSubInstanceTable() && Bridge::HasMaterialTable())
    {
        const RTXPTSubInstanceData SubInstance = Bridge::GetSubInstanceData();
        const RTXPTMaterialData    Material    = Bridge::GetMaterial(SubInstance.MaterialID);

        RTXPTVertex V0;
        RTXPTVertex V1;
        RTXPTVertex V2;
        Bridge::GetTriangleVertices(SubInstance, PrimitiveIndex(), V0, V1, V2);

        const float2 TexCoord = Bridge::InterpolateTexCoord(V0, V1, V2, Attributes.barycentrics);

        WorldPos    = Bridge::ComputeWorldHitPosition(V0, V1, V2, Attributes.barycentrics);
        WorldNormal = Bridge::InterpolateNormal(V0, V1, V2, Attributes.barycentrics);
        // Renormalize against the geometric normal if the interpolated normal is nearly zero
        // (degenerate vertex data) - keeps the shader robust on bad assets.
        if (dot(WorldNormal, WorldNormal) < 1e-6)
            WorldNormal = Bridge::ComputeGeometricNormal(V0, V1, V2);
        // Flip the shading normal to face the camera (single-sided shading; transmission is deferred).
        if (dot(WorldNormal, WorldRayDirection()) > 0.0)
            WorldNormal = -WorldNormal;

        // Perturb the shading normal with the tangent-space normal map (tangent derived from UV gradients).
        const float3 TangentNormal = Bridge::GetTangentNormal(Material, TexCoord);
        if (abs(TangentNormal.x) + abs(TangentNormal.y) > 1e-5)
        {
            const float4 WorldTangent = Bridge::ComputeWorldTangent(V0, V1, V2, WorldNormal);
            const float3 T            = WorldTangent.xyz;
            const float3 B            = cross(WorldNormal, T) * WorldTangent.w;
            WorldNormal               = normalize(T * TangentNormal.x + B * TangentNormal.y + WorldNormal * TangentNormal.z);
        }

        const float2 MetalRough = Bridge::GetMetallicRoughness(Material, TexCoord);
        Metallic                = MetalRough.x;
        Roughness               = MetalRough.y;

        BaseColor        = Bridge::GetBaseColor(Material, TexCoord).rgb;
        Payload.Emission = Bridge::GetEmission(Material, TexCoord);
    }

    Payload.WorldPos    = WorldPos;
    Payload.WorldNormal = normalize(WorldNormal);
    Payload.BaseColor   = BaseColor;
    Payload.Metallic    = Metallic;
    Payload.Roughness   = Roughness;
}

// TODO(RTXPT-Port Phase 5.5): Add NEE shadow rays toward analytic and environment lights.
```

- [ ] **Step 2: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTReference.rchit
```

Expected: no output and exit code 0.

- [ ] **Step 3: Commit the closest-hit changes**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTReference.rchit
git -C DiligentSamples commit -m "feat(rtxpt): sample phase 5.3 metallic-roughness and apply normal maps in closest hit" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the closest-hit shader.

---

### Task 6: Drive The Bounce With The GGX BSDF And Add Russian Roulette

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`

- [ ] **Step 1: Include the BSDF helper**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`, locate the include block at the top:

```hlsl
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTRandom.hlsli"
```

Replace it with:

```hlsl
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTRandom.hlsli"
#include "RTXPTBSDF.hlsli"
```

- [ ] **Step 2: Initialize the renamed payload fields**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`, locate the payload initialization:

```hlsl
        Payload.BaseColor   = float3(0.0, 0.0, 0.0);
        Payload.Emission    = float3(0.0, 0.0, 0.0);
        Payload.Padding0    = 0.0;
        Payload.Padding1    = 0.0;
```

Replace it with:

```hlsl
        Payload.BaseColor   = float3(0.0, 0.0, 0.0);
        Payload.Emission    = float3(0.0, 0.0, 0.0);
        Payload.Metallic    = 0.0;
        Payload.Roughness   = 1.0;
```

- [ ] **Step 3: Replace the Lambertian bounce with a BSDF sample + Russian roulette**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`, locate the bounce body:

```hlsl
        // Lambertian bounce: cosine-weighted hemisphere sample. The throughput update for a
        // cosine-weighted sample of a Lambertian BRDF is just baseColor (the cos and pdf cancel),
        // which matches the standard reference path tracer derivation.
        float        Pdf     = 0.0;
        const float2 Rand    = NextFloat2(Rng);
        const float3 NextDir = SampleCosineHemisphere(Rand, Payload.WorldNormal, Pdf);
        if (Pdf <= 0.0)
            break;

        Throughput *= Payload.BaseColor;

        // Offset the next ray slightly along the normal to avoid self-intersection.
        const float Bias = max(1e-4, 1e-3 * Payload.HitDistance);
        RayOrigin        = Payload.WorldPos + Payload.WorldNormal * Bias;
        RayDir           = NextDir;
```

Replace it with:

```hlsl
        // Build the glTF metallic-roughness surface the closest-hit filled in, then importance-sample
        // the GGX BSDF for the next direction. Weight = f * NoL / pdf is the throughput multiplier.
        const float3 Wo      = -RayDir;
        RTXPTSurface  Surface = RTXPTMakeSurface(Payload.WorldNormal, Payload.BaseColor, Payload.Metallic, Payload.Roughness);

        float3 NextDir;
        float3 Weight;
        float  Pdf;
        if (!RTXPTSampleBSDF(Surface, Wo, Rng, NextDir, Weight, Pdf))
            break;

        Throughput *= Weight;

        // Russian roulette once we are past MinBounces - unbiased early termination of dim paths.
        if (Bounce >= g_FrameConstants.PathTracer.MinBounces)
        {
            const float Survive = clamp(max(Throughput.x, max(Throughput.y, Throughput.z)), 0.05, 1.0);
            if (NextFloat(Rng) > Survive)
                break;
            Throughput /= Survive;
        }

        // Offset the next ray slightly along the shading normal to avoid self-intersection.
        const float Bias = max(1e-4, 1e-3 * Payload.HitDistance);
        RayOrigin        = Payload.WorldPos + Surface.N * Bias;
        RayDir           = NextDir;
```

- [ ] **Step 4: Re-target the resolved BSDF TODO**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`, locate:

```hlsl
// TODO(RTXPT-Port Phase 5.3): Replace single-lobe Lambertian sampling with a proper BSDF (GGX + diffuse + transmission).
// TODO(RTXPT-Port Phase 5.5): Add explicit light sampling and MIS once the lighting baker is restored.
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

Replace it with:

```hlsl
// TODO(RTXPT-Port Phase 5.3): Add transmission / nested dielectrics to the BSDF (currently opaque diffuse + GGX specular).
// TODO(RTXPT-Port Phase 5.5): Add explicit light sampling and MIS once the lighting baker is restored.
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTReference.rgen
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit the raygen changes**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTReference.rgen
git -C DiligentSamples commit -m "feat(rtxpt): drive phase 5.3 bounce with ggx bsdf and russian roulette" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the raygen shader.

---

### Task 7: Wire MinBounces, Add The UI Slider, Re-Target The Sample TODO

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

Context: `RTXPTPathTracerSettings::MinBounces` is currently forced to `0` in `UpdateFrameConstants`. This task adds an `m_MinBounces` member (default 3), feeds it into the frame constants, and exposes a "Min bounces (RR start)" slider that resets accumulation on change.

- [ ] **Step 1: Add the `m_MinBounces` member**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, locate:

```cpp
    Uint32                      m_MaxBounces                = 4;
```

Replace it with:

```cpp
    Uint32                      m_MaxBounces                = 4;
    Uint32                      m_MinBounces                = 3;
```

- [ ] **Step 2: Feed `m_MinBounces` into the frame constants**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate (in `UpdateFrameConstants`):

```cpp
    m_LastFrameConstants.PathTracer.MaxBounces        = m_MaxBounces;
    m_LastFrameConstants.PathTracer.AccumulationFrame = m_AccumulationFrame;
    m_LastFrameConstants.PathTracer.ResetAccumulation = m_ResetAccumulationPending ? 1u : 0u;
    m_LastFrameConstants.PathTracer.MinBounces        = 0;
```

Replace it with:

```cpp
    m_LastFrameConstants.PathTracer.MaxBounces        = m_MaxBounces;
    m_LastFrameConstants.PathTracer.AccumulationFrame = m_AccumulationFrame;
    m_LastFrameConstants.PathTracer.ResetAccumulation = m_ResetAccumulationPending ? 1u : 0u;
    m_LastFrameConstants.PathTracer.MinBounces        = m_MinBounces;
```

- [ ] **Step 3: Add the "Min bounces (RR start)" UI slider**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate (in `UpdateUI`):

```cpp
    int MaxBouncesUI = static_cast<int>(m_MaxBounces);
    if (ImGui::SliderInt("Max bounces", &MaxBouncesUI, 1, 16))
    {
        m_MaxBounces = static_cast<Uint32>(MaxBouncesUI);
        RequestAccumulationReset("Max bounces changed");
    }
    if (ImGui::Button("Reset accumulation"))
        RequestAccumulationReset("User reset");
```

Replace it with:

```cpp
    int MaxBouncesUI = static_cast<int>(m_MaxBounces);
    if (ImGui::SliderInt("Max bounces", &MaxBouncesUI, 1, 16))
    {
        m_MaxBounces = static_cast<Uint32>(MaxBouncesUI);
        RequestAccumulationReset("Max bounces changed");
    }
    int MinBouncesUI = static_cast<int>(m_MinBounces);
    if (ImGui::SliderInt("Min bounces (RR start)", &MinBouncesUI, 0, 16))
    {
        m_MinBounces = static_cast<Uint32>(MinBouncesUI);
        RequestAccumulationReset("Min bounces changed");
    }
    if (ImGui::Button("Reset accumulation"))
        RequestAccumulationReset("User reset");
```

- [ ] **Step 4: Re-target the resolved Phase 5.3 sample TODO**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate (in `UpdateUI`):

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.3): shade with the metallic-roughness GGX BSDF + normal maps (current path is textured Lambertian + alpha test).");
    ImGui::Text("TODO(RTXPT-Port Phase 5.5): add explicit light sampling and MIS once the lighting baker is restored.");
```

Replace it with:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.3): add transmission / nested dielectrics and ALPHA_MODE_BLEND (current BSDF is opaque metallic-roughness GGX + alpha-mask).");
    ImGui::Text("TODO(RTXPT-Port Phase 5.5): add explicit light sampling and MIS once the lighting baker is restored.");
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit the sample wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 5.3 min-bounce russian roulette into the sample" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the sample header and source.

---

### Task 8: Phase 5.3b Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [ ] **Step 1: Confirm the resolved markers are gone and the deferred markers remain**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5" DiligentSamples/Samples/RTXPT
```

Expected (resolved GGX/normal-map markers removed; deferred refinement markers remain — pre-existing `Phase 5` / `Phase 5.5` markers untouched by this plan also match the pattern and still appear):

```text
RTXPTMaterialBridge.hlsli : Phase 5.3 UV selectors / wrap / atlas
RTXPTReference.rmiss      : Phase 5.5 sun disk / environment map  (pre-existing, untouched)
RTXPTReference.rchit      : Phase 5.5 NEE
RTXPTReference.rahit      : Phase 5.3 ALPHA_MODE_BLEND
RTXPTReference.rgen       : Phase 5.3 transmission/nested dielectrics; Phase 5.5 NEE/MIS; Phase 6 tone mapping
RTXPTSample.cpp           : Phase 5 compiler flags (pre-existing, untouched); Phase 5.3 transmission/ALPHA_MODE_BLEND; Phase 5.5 NEE
```

Confirm no remaining match for `metallic-roughness GGX BSDF`, `metallic-roughness/normal-map`, or `single-lobe Lambertian`:

```powershell
rg -n "metallic-roughness GGX BSDF|metallic-roughness/normal-map|single-lobe Lambertian" DiligentSamples/Samples/RTXPT
```

Expected: no output.

- [ ] **Step 2: Confirm the BSDF file exists and is registered**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli
rg -n "RTXPTBSDF.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: `True` and one match in `CMakeLists.txt`.

- [ ] **Step 3: Confirm the layout static_assert and mirror agree**

Run:

```powershell
rg -n "sizeof\(RTXPTMaterialData\) == 96" DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp
rg -n "total size 96 bytes" DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
```

Expected: one match each. (The C++ `static_assert` is the build-time guard that the 96-byte layout matches.)

- [ ] **Step 4: Confirm the DiligentSamples log shows the Phase 5.3b commits**

Run:

```powershell
git -C DiligentSamples log --oneline -n 8
```

Expected (most recent first):

```text
feat(rtxpt): wire phase 5.3 min-bounce russian roulette into the sample
feat(rtxpt): drive phase 5.3 bounce with ggx bsdf and russian roulette
feat(rtxpt): sample phase 5.3 metallic-roughness and apply normal maps in closest hit
feat(rtxpt): add phase 5.3 metallic-roughness, normal-map, and tangent bridge helpers
feat(rtxpt): add phase 5.3 metallic-roughness ggx bsdf helper
feat(rtxpt): mirror phase 5.3 96-byte material data and payload metallic/roughness
feat(rtxpt): add phase 5.3 metallic-roughness and normal texture indices
fix(rtxpt): bind material textures as array SRVs
```

- [ ] **Step 5: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: exit code 0. If the build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 6: Optional D3D12 runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with D3D12 on a standalone-RT + bindless-capable machine. Expected ImGui facts:

```text
Bindless: yes
Material textures bound: yes (N)
Alpha-test any-hit: enabled
TraceRays pass: ready
Material bridge / Sub-instance bridge / Light bridge / Vertex buffer / Index buffer: bound
Accumulation target: active (RGBA32F)
TraceRays executed: yes; TraceRays count increases every frame
```

Expected visual result vs Phase 5.3: surfaces now show **view-dependent specular highlights** (GGX) that sharpen as roughness drops; metallic surfaces (e.g. metal fixtures) tint their reflections by base color while dielectrics keep a neutral specular; normal-mapped surfaces show **per-texel surface detail** (bumps/grooves) under the moving specular highlight that was absent in 5.3's flat Lambertian. The image still converges across accumulated frames. Moving the "Min bounces (RR start)" slider down increases noise but keeps the converged image unchanged (Russian roulette is unbiased); moving "Max bounces" or pressing "Reset accumulation" restarts convergence.

- [ ] **Step 7: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT + bindless-capable machine. Expected facts match the D3D12 run.

If the device reports `Bindless: no` (or the model loaded zero textures), the factor-only fallback now still runs the **GGX BSDF using the per-material metallic/roughness factors** (no texture sampling, no normal-map perturbation):

```text
Material textures bound: no (0)
Alpha-test any-hit: disabled
TraceRays pass: ready
```

with specular highlights driven by the constant factors. The sample must still render and converge.

If standalone ray tracing shaders are unavailable, expected facts:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
```

and the sample clears the swapchain via `ClearFallback`.

- [ ] **Step 8: Commit the top-level submodule pointer and plan**

After all `DiligentSamples` Phase 5.3b commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-29-rtxpt-phase5-3b-ggx-bsdf-normal-maps.md
git commit -m "feat(samples): plan and add RTXPT phase 5.3 ggx bsdf and normal maps" -m "Co-Authored-By: GPT 5.5"
```

Expected: one top-level commit that records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] **Spec coverage.** This plan resolves the deferred core of Phase 5 layer 5 (`docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`, "Material specialization, alpha test and any-hit" → the "Advanced BSDF parameters" / "Material shader permutation" later-material list): the metallic-roughness GGX BSDF, normal mapping, and Russian roulette named as deferred in the Phase 5.3 plan's scope note. The still-deferred layer-5 refinements (transmission/nested dielectrics, `ALPHA_MODE_BLEND`, UV selectors/wrap/atlas, per-material permutations) are each preserved as `TODO(RTXPT-Port Phase 5.3)` markers and named in the Scope Note. Layers 6-9 stay in their own future plans.
- [x] **Runnable increments.** Every task ends with a focused commit. The GGX BSDF runs in both the textured and factor-only paths, so when `BindlessResources` is unavailable the bounce still uses GGX with factor metallic/roughness (no texture sampling / no normal-map perturbation); when ray tracing / standalone RT is unavailable the existing `ClearFallback` path still runs. The accumulation/blit chain, the RT-pass binding model, the SBT, the payload size, and `MaxRecursionDepth` are all untouched.
- [x] **Single source of truth.** `RTXPTMaterialData` is defined once in C++ (`RTXPTMaterials.hpp`, `static_assert(sizeof==96)`) and mirrored once in HLSL (`RTXPTShaderShared.hlsli`) with annotated, matching offsets (BaseColorFactor@0, EmissiveFactor@16, AlphaCutoff@28, Flags@32, BaseColorTextureIndex@36, EmissiveTextureIndex@40, MetallicFactor@44, RoughnessFactor@48, BaseColorTextureSlice@52, EmissiveTextureSlice@56, MetallicRoughnessTextureIndex@60, MetallicRoughnessTextureSlice@64, NormalTextureIndex@68, NormalTextureSlice@72, NormalScale@76, Padding0-3@80/84/88/92). Flag bits match across C++/HLSL (`HasMetallicRoughnessTexture=0x8`, `HasNormalTexture=0x10`). The payload stays 16 floats (64 bytes) by renaming `Padding0`/`Padding1` to `Metallic`/`Roughness`, so `MaxPayloadSize = sizeof(float)*16` in the RT pass remains correct without edits.
- [x] **No new binding / proven pattern.** The metallic-roughness and normal textures are already members of the bindless `g_MaterialTextures[]` table (built from every `Model.GetTexture(i)` in `Upload`), so this plan adds only material **indices/slices/flags** and shader sampling — the RT-pass `Initialize`, SRB, and `SetArray` call are unchanged. Sampling uses the existing `Bridge::SampleMaterialTexture` (`NonUniformResourceIndex` + `SampleLevel(...,0)`), proven on D3D12 and Vulkan in Phase 5.3. The single linear `g_MaterialSampler` is correct because the GLTF loader marks base-color/emissive as sRGB (hardware-decoded) and metallic-roughness/normal as linear UNORM.
- [x] **Type/name consistency.** `Bridge::ComputeWorldTangent` (Task 4 Step 1) returns `float4` and is consumed by `RTXPTReference.rchit` (Task 5). `Bridge::GetMetallicRoughness` returns `float2(metallic, roughness)` and `Bridge::GetTangentNormal` returns `float3` — both declared with textured and `#else` factor-only variants (Task 4 Step 2) and consumed unconditionally by the closest-hit (Task 5). `RTXPTMakeSurface` / `RTXPTSampleBSDF` / `RTXPTSurface` (Task 3) are consumed by `RTXPTReference.rgen` (Task 6). The HLSL flag constants `kRTXPTMaterialFlagHasMetallicRoughnessTexture` / `kRTXPTMaterialFlagHasNormalTexture` match the C++ `kRTXPTMaterialFlag_HasMetallicRoughnessTexture` / `_HasNormalTexture` semantics. `m_MinBounces` (Task 7 Step 1) feeds `PathTracer.MinBounces` (Step 2) and the slider (Step 3) and is read by raygen's Russian-roulette guard (Task 6 Step 3).
- [x] **No tangent attribute handled.** Vertex buffer 0 is POSITION + NORMAL + TEXCOORD_0 only; `ComputeWorldTangent` derives the tangent from triangle edges + UV deltas, transforms it with `ObjectToWorld3x4()`, Gram-Schmidt-orthonormalizes against the shading normal, and falls back to an arbitrary perpendicular on degenerate UVs — so the TBN frame is always valid.
- [x] **No placeholders.** Every code step shows complete code; every command shows expected output. The only `TODO(...)` strings are the intentional, structured open-work markers required by the spec's TODO policy.
- [x] **House style honored.** Verification avoids build/runtime execution unless the user explicitly asks (per `CLAUDE.md`); each task is a single-purpose commit using the established `Co-Authored-By: GPT 5.5` trailer (matching every prior RTXPT commit and the Phase 5.3 plan); copyright dates stay `2026`; the obsolete payload padding fields are renamed (not left as dead fields).
