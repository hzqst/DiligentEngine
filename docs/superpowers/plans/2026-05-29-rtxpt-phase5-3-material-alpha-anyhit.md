# RTXPT Phase 5.3 Material Textures, Alpha Test & Any-Hit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land the core of Phase 5 shader layer 5 from the RTXPT port design — material specialization, alpha test and any-hit — by giving the reference path tracer per-material GPU data with texture indices, a bindless material-texture table, textured base-color + emissive shading, and an alpha-test any-hit shader gated on alpha-masked geometry.

**Architecture:** Replace the raw 96-byte `GLTF::Material::ShaderAttribs` material buffer with a purpose-built 64-byte `RTXPTMaterialData` record that carries base color / emissive factors, alpha cutoff, per-material flags, and bindless texture indices + slices. `RTXPTMaterials` also collects one `Texture2DArray` shader-resource view per loaded GLTF texture into a bindless table (the Diligent GLTF loader creates each texture as `RESOURCE_DIM_TEX_2D_ARRAY`). `RTXPTAccelerationStructures` marks alpha-masked geometry non-opaque (`RAYTRACING_GEOMETRY_FLAG_NONE`) so the runtime invokes an any-hit shader only there. The closest-hit shader samples the base-color texture (`BaseColorFactor.rgb * tex.rgb`) and emissive texture into the payload; a new any-hit shader samples base-color alpha and calls `IgnoreHit()` below the cutoff. The textured path is gated on the `BindlessResources` capability (with a clean fallback to the Phase 5.2 factor-only path) using a compile-time shader macro (`RTXPT_ENABLE_MATERIAL_TEXTURES` / `RTXPT_MATERIAL_TEXTURE_COUNT`), following the proven Tutorial21_RayTracing texture-array binding pattern (bounded `Texture2DArray[]` array + immutable sampler + `SetArray`, indexed with `NonUniformResourceIndex`).

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs (`RayTracingPipelineStateCreateInfoX::AddTriangleHitShader` with an any-hit shader, `PipelineResourceLayoutDescX::AddImmutableSampler`, `IShaderResourceVariable::SetArray`), `ShaderMacroHelper`, DiligentTools `GLTFLoader` (`GLTF::Material::GetTextureId` / `GetTextureAttrib`, `GLTF::Material::ALPHA_MODE_MASK`, `GLTF::DefaultBaseColorTextureAttribId` / `DefaultEmissiveTextureAttribId`, `GLTF::Model::GetTexture` / `GetTextureCount`), HLSL 6.5 ray tracing shaders compiled by DXC (`[shader("anyhit")]`, `Texture2DArray g_MaterialTextures[N]`, `SampleLevel`), Dear ImGui.

---

## Scope Note: Phase 5 Sub-Plan Series

Phase 5 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md` enumerates nine shader dependency layers. Completed plans:

- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-1-shader-bridge.md` — layers 2–3 (shared declarations and scene/material bridge).
- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-2-reference-path-tracer.md` — layer 4 (reference path tracer core).

This plan covers the **core of layer 5** (material specialization, alpha test, any-hit). It deliberately delivers the runnable, highest-value slice of layer 5 — textured base-color + emissive shading and alpha-test any-hit — and **defers** the following layer-5 refinements to a follow-up plan, each preserved as a structured `TODO(RTXPT-Port Phase 5.3)` marker in code:

- Full metallic-roughness GGX BSDF and normal mapping (the bounce stays cosine-weighted Lambertian over the textured base color).
- `ALPHA_MODE_BLEND` stochastic transparency (only `ALPHA_MODE_MASK` is alpha-tested).
- Honoring `TextureShaderAttribs` UV selectors / wrap modes / atlas UV transform (this plan assumes `TEXCOORD_0`, a single immutable wrap sampler, and the non-atlas identity transform — correct for the default bistro load).
- Per-material shader permutations / full hit-group-table generation (a single über closest-hit + any-hit pair is used, matching RTXPT's IntroSample).

Remaining Phase 5 sub-phases each get their own plan in a later session:

- Phase 5.4: Stable planes and realtime mode (layer 6).
- Phase 5.5: RTXDI shader bridge and passes (layer 7).
- Phase 5.6: NRD, denoising guides, post-process (layer 8).
- Phase 5.7: NVAPI, SER, OMM, DLSS-related shader variants (layer 9).

## Baseline

Current state of `DiligentSamples/Samples/RTXPT` (submodule `HEAD = f0b1b2db fix(rtxpt): restore full bistro scene visibility`, top-level `HEAD = 252640c`):

- `RTXPTScene::LoadDefaultScene` loads `bistro-programmer-art.scene.json` / `Models/Bistro/bistro.gltf` via the Diligent GLTF loader **without** a `ResourceManager`, so each texture is an individual `RESOURCE_DIM_TEX_2D_ARRAY` (`DiligentTools/AssetLoader/src/GLTFLoader.cpp:975`). Vertex buffer 0 (POSITION + NORMAL + TEXCOORD_0) and the index buffer are `BIND_SHADER_RESOURCE`-readable; `GetVertexBuffer0()` / `GetIndexBuffer()` / `GetVertexStride0()` exist.
- `RTXPTMaterials::Upload(IRenderDevice*, const GLTF::Model&)` uploads a `StructuredBuffer` of raw `GLTF::Material::ShaderAttribs` (96 bytes/entry, always ≥ 1 entry). It exposes `GetStats()` (`MaterialCount`, `LastError`) and `GetMaterialBuffer()`. No texture data.
- `RTXPTAccelerationStructures::BuildStaticScene` builds per-mesh-node BLAS + a single TLAS (`HIT_GROUP_BINDING_MODE_PER_GEOMETRY`, `HitGroupStride = 1`, per-instance `CustomId = SubInstanceBase`). Every geometry is built with `BuildData.Flags = RAYTRACING_GEOMETRY_FLAG_OPAQUE`. It uploads a `RTXPTSubInstanceData` structured buffer (32 bytes/entry: `MaterialID`, `Flags`, `FirstIndex`, `IndexCount`, `FirstVertex`, `VertexCount`, 2× padding). `kRTXPTSubInstanceFlag_Indexed = 0x1`.
- `RTXPTRayTracingPass::Initialize(...)` compiles `RTXPTReference.{rgen,rmiss,rchit}` (HLSL 6.5, DXC), creates an RT PSO with one general raygen (`"Main"`), one miss (`"PrimaryMiss"`), one triangle hit group (`"PrimaryHit"`, closest-hit only), binds `g_FrameConstants`/`g_TLAS` (raygen, STATIC), `g_Materials`/`g_SubInstanceData`/`g_VertexBuffer`/`g_IndexBuffer` (closest hit, STATIC), `g_Lights` (miss, STATIC), `g_OutputColor`/`g_AccumColor` (raygen, DYNAMIC). `MaxPayloadSize = sizeof(float)*16`, `MaxAttributeSize = sizeof(float)*2`, `MaxRecursionDepth = 1`. `Trace(...)` binds the two output UAVs on the SRB and dispatches `TraceRays`.
- `RTXPTReference.rgen` runs an N-bounce cosine-weighted Lambertian loop with `TraceRay(..., RAY_FLAG_FORCE_OPAQUE, ...)`, accumulates into the RGBA32F `g_AccumColor`, tone-maps into `g_OutputColor`. `RTXPTReference.rchit` fills `RTXPTPathTracerPayload` with world pos/normal + `Material.BaseColorFactor.rgb`. `RTXPTReference.rmiss` writes a procedural sky into `Payload.Emission`.
- `RTXPTSample` owns `RTXPTFeatureCaps` (incl. `BindlessResources`), a `FirstPersonCamera`, `RTXPTFrameConstants` (176 bytes, with `RTXPTPathTracerSettings`), accumulation reset plumbing, and an ImGui status panel. `CreatePhase4Passes()` wires the RT pass.
- Shader bridges: `RTXPTShaderShared.hlsli` mirrors `RTXPTSubInstanceData`, `RTXPTFrameConstants`, `RTXPTPathTracerSettings`, `RTXPTPathTracerPayload`, `RTXPTMaterialAttribs` (96-byte ShaderAttribs mirror), `RTXPTLightData`, `RTXPTVertex`. `RTXPTSceneBridge.hlsli` declares `g_FrameConstants`/`g_SubInstanceData`/`g_Lights`/`g_VertexBuffer`/`g_IndexBuffer` and the `Bridge::` hit helpers (guarded by `RTXPT_ENABLE_HIT_BRIDGE`). `RTXPTMaterialBridge.hlsli` declares `g_Materials` (`StructuredBuffer<RTXPTMaterialAttribs>`) and `Bridge::GetMaterial` / `GetMaterialBaseColor`.

This plan assumes the top-level repository starts clean and the `DiligentSamples` submodule is at the Phase 5.2 state above.

---

## Scope

This plan implements the Phase 5.3 runnable milestone:

- Add a purpose-built `RTXPTMaterialData` GPU struct (64 bytes) in `RTXPTMaterials.hpp` with base color / emissive factors, alpha cutoff, metallic/roughness factors, per-material `Flags`, and bindless base-color / emissive texture indices + slices. Add the `kRTXPTMaterialFlag_*` constants and a shared `RTXPTMaterialIsAlphaTested()` helper.
- Rewrite `RTXPTMaterials::Upload` to build `RTXPTMaterialData` from each GLTF material (reading `Material.Attribs` factors + `Material.GetTextureId`/`GetTextureAttrib`) and to collect one shader-resource view per `GLTF::Model` texture into a bindless table. Expose `GetTextureCount()` / `GetTextureBindings()` and `GetStats().TextureCount`.
- Mirror `RTXPTMaterialData` + flag constants in `RTXPTShaderShared.hlsli` (replacing the now-unused `RTXPTMaterialAttribs`).
- In `RTXPTAccelerationStructures::BuildStaticScene`, set per-geometry `RAYTRACING_GEOMETRY_FLAG_NONE` for alpha-tested materials (else `..._OPAQUE`) and count them in stats.
- Extend the shader bridges: add `Bridge::InterpolateTexCoord` (scene bridge); add the bindless `Texture2DArray g_MaterialTextures[]` + `g_MaterialSampler` and `Bridge::GetBaseColor` / `GetEmission` / `AlphaTestPasses` helpers (material bridge), all macro-gated with factor-only `#else` fallbacks.
- Update `RTXPTReference.rchit` to sample base-color + emissive textures; add `RTXPTReference.rahit` (alpha-test any-hit); change `RTXPTReference.rgen` to trace with `RAY_FLAG_NONE`.
- Extend `RTXPTRayTracingPass::Initialize` to accept the bindless texture table + an `EnableMaterialTextures` flag, conditionally compile the textured shaders with macros, add the any-hit shader to the hit group, declare the texture array + immutable sampler, and bind the array via `SetArray`. Add stats (`MaterialTexturesBound`, `AnyHitEnabled`, `MaterialTextureCount`).
- Wire `RTXPTSample::CreatePhase4Passes` to pass the texture table and gate on `BindlessResources`, with a one-shot retry to the factor-only path if the textured PSO fails to build. Add ImGui diagnostics and register `RTXPTReference.rahit` in CMake.
- Narrow / re-target the structured `TODO(RTXPT-Port Phase 5.3)` markers: remove the alpha-test and base-color-texture-binding markers this plan resolves; keep markers for the deferred GGX/normal-map shading, `ALPHA_MODE_BLEND`, UV-selector/wrap handling, and material permutations.

This plan intentionally does not:

- Implement a metallic-roughness GGX BSDF, normal mapping, transmission, or nested dielectrics (the bounce stays cosine-weighted Lambertian over the textured base color).
- Implement `ALPHA_MODE_BLEND` / stochastic transparency, or alpha test on shadow/visibility rays (there are no shadow rays yet — NEE is Phase 5.5).
- Honor `TextureShaderAttribs` UV selectors, wrap modes, or atlas UV transform beyond the non-atlas identity case (assumes `TEXCOORD_0` + a single immutable wrap sampler + slice).
- Implement per-material shader permutations or a multi-record hit-group table (one über closest-hit + any-hit pair is used).
- Run automated builds or runtime execution; build/runtime steps are listed for explicit user request only.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
  - Add `RTXPTMaterialData`, `kRTXPTMaterialFlag_*`, `RTXPTMaterialIsAlphaTested`, bindless-table accessors; extend stats.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
  - Implement the alpha-test helper; rewrite `Upload` to build `RTXPTMaterialData` + collect texture SRVs.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
  - Add `AlphaTestedGeometryCount` to stats.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
  - Set per-geometry opaque/non-opaque flag from material alpha mode.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`
  - Replace `RTXPTMaterialAttribs` with `RTXPTMaterialData` + flag constants.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
  - Add `Bridge::InterpolateTexCoord`; drop resolved Phase 5.3 TODOs.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`
  - Add bindless texture array + sampler + `GetBaseColor`/`GetEmission`/`AlphaTestPasses` (macro-gated).
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`
  - Sample base-color + emissive textures.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rahit`
  - Alpha-test any-hit.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`
  - Trace with `RAY_FLAG_NONE`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
  - Extend `Initialize` signature; add texture/any-hit stats.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
  - Conditional textured PSO, any-hit hit group, texture-array + sampler binding.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Pass texture table + gate, retry fallback, UI diagnostics, TODO updates.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Register `RTXPTReference.rahit`.

---

### Task 0: Phase 5.2 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [ ] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected: branch line `## RTXPT...origin/RTXPT` and no staged/modified files under `DiligentSamples/Samples/RTXPT` or `docs/superpowers/plans`. Unrelated files may be left untouched.

- [ ] **Step 2: Confirm DiligentSamples Phase 5.2 state**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentSamples log --oneline -n 9
```

Expected: clean working tree; the most recent commit is `f0b1b2db fix(rtxpt): restore full bistro scene visibility`, above the seven `feat(rtxpt): ... phase 5.2 ...` commits.

- [ ] **Step 3: Confirm Phase 5.2 files exist and the any-hit file does not**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rahit
```

Expected:

```text
True
True
False
```

If `RTXPTReference.rahit` already exists, inspect it before overwriting and preserve unrelated work.

- [ ] **Step 4: Confirm current Phase 5.3 markers**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5\.3\)" DiligentSamples/Samples/RTXPT
```

Expected matches in: `RTXPTSceneBridge.hlsli` (2), `RTXPTMaterialBridge.hlsli` (2), `RTXPTReference.rchit` (2), `RTXPTSample.cpp` (1). This plan removes/re-targets these as the corresponding code lands.

---

### Task 1: Build GPU Material Data With Texture Indices

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`

Context: `GLTF::Material::ShaderAttribs` carries the factors (`BaseColorFactor`, `EmissiveFactor`, `AlphaMode` as `int`, `AlphaCutoff`, `MetallicFactor`, `RoughnessFactor`) but **not** texture indices. Texture references come from `Material::GetTextureId(GLTF::DefaultBaseColorTextureAttribId)` (→ index into `Model.Textures`, or `-1`) and `Material::GetTextureAttrib(...)` (→ `TextureShaderAttribs` with `.TextureSlice`, which is `0` for the non-atlas bistro load). `GLTF::Material::ALPHA_MODE_MASK == 1`. We build a compact `RTXPTMaterialData` and a bindless texture table here.

- [ ] **Step 1: Rewrite the `RTXPTMaterials.hpp` body**

Replace everything in `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` between the `#pragma once` line and the end of file (i.e. the include block, the `RTXPTMaterialStats` struct, and the `RTXPTMaterials` class) with:

```cpp
#pragma once

#include <string>
#include <vector>

#include "Buffer.h"
#include "DeviceObject.h"
#include "GLTFLoader.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "Texture.h"
#include "TextureView.h"

namespace Diligent
{

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

// A material is alpha tested only when it uses ALPHA_MODE_MASK and actually has a base-color texture to
// sample the alpha from. The acceleration-structure geometry flags and the GPU material flags must agree,
// so both sides call this single helper.
bool RTXPTMaterialIsAlphaTested(const GLTF::Material& Material);

struct RTXPTMaterialStats
{
    Uint32      MaterialCount = 0;
    Uint32      TextureCount  = 0;
    std::string LastError;
};

class RTXPTMaterials
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Model& Model);

    const RTXPTMaterialStats& GetStats() const { return m_Stats; }
    IBuffer*                  GetMaterialBuffer() const { return m_MaterialBuffer; }

    // Bindless material-texture table. Indices match GLTF::Model texture indices and are referenced by
    // RTXPTMaterialData::BaseColorTextureIndex / EmissiveTextureIndex. The views are owned by the GLTF model,
    // which must outlive the ray tracing SRB.
    Uint32                GetTextureCount() const { return static_cast<Uint32>(m_TextureBindings.size()); }
    IDeviceObject* const* GetTextureBindings() const { return m_TextureBindings.empty() ? nullptr : m_TextureBindings.data(); }

private:
    RefCntAutoPtr<IBuffer>      m_MaterialBuffer;
    std::vector<IDeviceObject*> m_TextureBindings;
    RTXPTMaterialStats          m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Rewrite the `RTXPTMaterials.cpp` body**

Replace everything in `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` between the copyright block's closing `*/` and the end of file with:

```cpp
#include "RTXPTMaterials.hpp"

#include <algorithm>
#include <vector>

namespace Diligent
{

bool RTXPTMaterialIsAlphaTested(const GLTF::Material& Material)
{
    return Material.Attribs.AlphaMode == GLTF::Material::ALPHA_MODE_MASK &&
           Material.GetTextureId(GLTF::DefaultBaseColorTextureAttribId) >= 0;
}

void RTXPTMaterials::Reset()
{
    m_MaterialBuffer.Release();
    m_TextureBindings.clear();
    m_Stats = {};
}

bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const GLTF::Model& Model)
{
    Reset();

    m_Stats.MaterialCount = static_cast<Uint32>(Model.Materials.size());

    // Collect one shader-resource view per loaded GLTF texture. The loader always provides a (stub) texture,
    // so a null view should not happen; if it does, drop the whole table and fall back to factor-only shading.
    const Uint32 ModelTextureCount = static_cast<Uint32>(Model.GetTextureCount());
    m_TextureBindings.reserve(ModelTextureCount);
    for (Uint32 i = 0; i < ModelTextureCount; ++i)
    {
        ITexture*     pTexture = Model.GetTexture(i);
        ITextureView* pSRV     = pTexture != nullptr ? pTexture->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
        if (pSRV == nullptr)
        {
            m_TextureBindings.clear();
            m_Stats.LastError = "RTXPT material texture has no shader-resource view; texture sampling disabled";
            break;
        }
        m_TextureBindings.push_back(pSRV);
    }
    m_Stats.TextureCount = static_cast<Uint32>(m_TextureBindings.size());

    const Uint32 ValidTextureCount = m_Stats.TextureCount;

    std::vector<RTXPTMaterialData> MaterialData;
    MaterialData.reserve(std::max<size_t>(Model.Materials.size(), 1));
    for (const GLTF::Material& Material : Model.Materials)
    {
        const GLTF::Material::ShaderAttribs& Attribs = Material.Attribs;

        RTXPTMaterialData Data;
        Data.BaseColorFactor = Attribs.BaseColorFactor;
        Data.EmissiveFactor  = Attribs.EmissiveFactor;
        Data.AlphaCutoff     = Attribs.AlphaCutoff;
        Data.MetallicFactor  = Attribs.MetallicFactor;
        Data.RoughnessFactor = Attribs.RoughnessFactor;

        const int BaseColorTextureId = Material.GetTextureId(GLTF::DefaultBaseColorTextureAttribId);
        if (BaseColorTextureId >= 0 && static_cast<Uint32>(BaseColorTextureId) < ValidTextureCount)
        {
            Data.Flags |= kRTXPTMaterialFlag_HasBaseColorTexture;
            Data.BaseColorTextureIndex = static_cast<Uint32>(BaseColorTextureId);
            Data.BaseColorTextureSlice = Material.GetTextureAttrib(GLTF::DefaultBaseColorTextureAttribId).TextureSlice;
        }

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

        MaterialData.emplace_back(Data);
    }

    if (MaterialData.empty())
    {
        // Always upload at least one default material so the shader-side bridge SRV is never null.
        MaterialData.emplace_back();
    }

    BufferDesc Desc;
    Desc.Name              = "RTXPT material buffer";
    Desc.Usage             = USAGE_IMMUTABLE;
    Desc.BindFlags         = BIND_SHADER_RESOURCE;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(RTXPTMaterialData);
    Desc.Size              = sizeof(RTXPTMaterialData) * MaterialData.size();

    BufferData Data{MaterialData.data(), Desc.Size};
    pDevice->CreateBuffer(Desc, &Data, &m_MaterialBuffer);

    if (!m_MaterialBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT material buffer";
        return false;
    }

    return true;
}

} // namespace Diligent
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit GPU material data**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp
git -C DiligentSamples commit -m "feat(rtxpt): build phase 5.3 gpu material data with texture indices" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two material files.

---

### Task 2: Mirror The Material Data Layout In The Shared Header

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`

- [ ] **Step 1: Replace `RTXPTMaterialAttribs` with `RTXPTMaterialData`**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`, locate this block:

```hlsl
// Mirrors Diligent::GLTF::Material::ShaderAttribs from DiligentTools/AssetLoader/interface/GLTFLoader.hpp.
// Keep field order/sizes synchronized; total size is 96 bytes (16-byte aligned).
struct RTXPTMaterialAttribs
{
    float4 BaseColorFactor; // offset 0

    float3 EmissiveFactor; // offset 16
    float  NormalScale;

    float3 SpecularFactor; // offset 32
    float  ClearcoatNormalScale;

    int   Workflow; // offset 48
    int   AlphaMode;
    float AlphaCutoff;
    float MetallicFactor;

    float RoughnessFactor; // offset 64
    float OcclusionFactor;
    float ClearcoatFactor;
    float ClearcoatRoughnessFactor;

    float4 CustomData; // offset 80
};
```

Replace it with:

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

- [ ] **Step 2: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 3: Commit the shared header update**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): mirror phase 5.3 material data layout in shared header" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the shared header.

---

### Task 3: Set Alpha-Test Geometry Flags From Material Data

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`

Context: `RAYTRACING_GEOMETRY_FLAG_OPAQUE` (`0x01`) tells the runtime to skip any-hit; `RAYTRACING_GEOMETRY_FLAG_NONE` (`0x00`) lets any-hit run. Marking only alpha-masked geometry non-opaque means the alpha-test any-hit shader is invoked exactly where it is needed and opaque geometry pays no cost. `Primitive.MaterialId` selects `Model.Materials[...]`.

- [ ] **Step 1: Add an alpha-tested geometry counter to the stats**

In `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`, locate:

```cpp
    Uint32      BLASCount           = 0;
    Uint32      SubInstanceCount    = 0;
    Uint64      BLASScratchSize     = 0;
```

Replace it with:

```cpp
    Uint32      BLASCount               = 0;
    Uint32      SubInstanceCount        = 0;
    Uint32      AlphaTestedGeometryCount = 0;
    Uint64      BLASScratchSize         = 0;
```

- [ ] **Step 2: Include the materials helper**

In `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`, locate:

```cpp
#include "GraphicsAccessories.hpp"
```

Replace it with:

```cpp
#include "GraphicsAccessories.hpp"
#include "RTXPTMaterials.hpp"
```

- [ ] **Step 3: Compute per-geometry opaque flag from material alpha mode**

In `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`, locate:

```cpp
            BuildData.PrimitiveCount       = TriangleDesc.MaxPrimitiveCount;
            BuildData.Flags                = RAYTRACING_GEOMETRY_FLAG_OPAQUE;
```

Replace it with:

```cpp
            BuildData.PrimitiveCount       = TriangleDesc.MaxPrimitiveCount;
            // Alpha-masked geometry must be non-opaque so the runtime invokes the alpha-test any-hit shader.
            // Everything else stays opaque to skip any-hit entirely.
            const bool GeometryAlphaTested =
                Primitive.MaterialId < Model.Materials.size() &&
                RTXPTMaterialIsAlphaTested(Model.Materials[Primitive.MaterialId]);
            BuildData.Flags = GeometryAlphaTested ? RAYTRACING_GEOMETRY_FLAG_NONE : RAYTRACING_GEOMETRY_FLAG_OPAQUE;
            if (GeometryAlphaTested)
                ++m_Stats.AlphaTestedGeometryCount;
```

- [ ] **Step 4: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit alpha-test geometry flags**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
git -C DiligentSamples commit -m "feat(rtxpt): set phase 5.3 alpha-test geometry flags from material data" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the acceleration structures files.

---

### Task 4: Add Material-Texture And Alpha-Test Bridge Helpers

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`

- [ ] **Step 1: Add `Bridge::InterpolateTexCoord` to the scene bridge**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`, locate this block (the world-position helper that closes the `RTXPT_ENABLE_HIT_BRIDGE` section):

```hlsl
    // World-space hit position using ObjectToWorld3x4().
    float3 ComputeWorldHitPosition(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary   = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        const float3 ObjPos = V0.Position * Bary.x + V1.Position * Bary.y + V2.Position * Bary.z;
        return mul(ObjectToWorld3x4(), float4(ObjPos, 1.0));
    }
#endif
```

Replace it with:

```hlsl
    // World-space hit position using ObjectToWorld3x4().
    float3 ComputeWorldHitPosition(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary   = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        const float3 ObjPos = V0.Position * Bary.x + V1.Position * Bary.y + V2.Position * Bary.z;
        return mul(ObjectToWorld3x4(), float4(ObjPos, 1.0));
    }

    // Barycentric-interpolated TEXCOORD_0 for the current closest-hit / any-hit triangle.
    float2 InterpolateTexCoord(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        return V0.TexCoord0 * Bary.x + V1.TexCoord0 * Bary.y + V2.TexCoord0 * Bary.z;
    }
#endif
```

Then locate and delete these two now-resolved TODO lines near the end of the file:

```hlsl
// TODO(RTXPT-Port Phase 5.3): Add alpha-mask/transparent flags to RTXPTSubInstanceData and propagate them into any-hit specialization.
// TODO(RTXPT-Port Phase 5.3): Bind material textures and respect TextureShaderAttribs UV selectors / wrap modes.
```

(Alpha test is now driven by material flags + geometry flags + the any-hit shader; texture binding lands in this task.)

- [ ] **Step 2: Rewrite the material bridge with the bindless texture helpers**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli` with:

```hlsl
#ifndef RTXPT_MATERIAL_BRIDGE_HLSLI
#define RTXPT_MATERIAL_BRIDGE_HLSLI

#include "RTXPTShaderShared.hlsli"

StructuredBuffer<RTXPTMaterialData> g_Materials;

#ifdef RTXPT_ENABLE_MATERIAL_TEXTURES
// One Texture2DArray per loaded GLTF texture (the Diligent loader creates RESOURCE_DIM_TEX_2D_ARRAY textures).
// RTXPT_MATERIAL_TEXTURE_COUNT is supplied at compile time and equals RTXPTMaterials::GetTextureCount().
Texture2DArray g_MaterialTextures[RTXPT_MATERIAL_TEXTURE_COUNT];
SamplerState   g_MaterialSampler;
#endif

namespace Bridge
{
    bool HasMaterialTable()
    {
        uint Count  = 0;
        uint Stride = 0;
        g_Materials.GetDimensions(Count, Stride);
        return Count > 0;
    }

    uint GetMaterialCount()
    {
        uint Count  = 0;
        uint Stride = 0;
        g_Materials.GetDimensions(Count, Stride);
        return Count;
    }

    // Out-of-range indices clamp to the last material so a bad MaterialID never UB-reads.
    RTXPTMaterialData GetMaterial(uint MaterialID)
    {
        const uint LastIndex = max(GetMaterialCount(), 1u) - 1u;
        const uint Index     = min(MaterialID, LastIndex);
        return g_Materials[Index];
    }

#ifdef RTXPT_ENABLE_MATERIAL_TEXTURES
    // Ray tracing shaders cannot derive LOD, so we sample the most detailed level. Slice is 0 for the
    // non-atlas bistro load; it is carried so an atlas path can be added later without touching call sites.
    float4 SampleMaterialTexture(uint TextureIndex, float Slice, float2 UV)
    {
        return g_MaterialTextures[NonUniformResourceIndex(TextureIndex)].SampleLevel(g_MaterialSampler, float3(UV, Slice), 0.0);
    }

    float4 GetBaseColor(RTXPTMaterialData Material, float2 UV)
    {
        float4 Color = Material.BaseColorFactor;
        if ((Material.Flags & kRTXPTMaterialFlagHasBaseColorTexture) != 0u)
            Color *= SampleMaterialTexture(Material.BaseColorTextureIndex, Material.BaseColorTextureSlice, UV);
        return Color;
    }

    float3 GetEmission(RTXPTMaterialData Material, float2 UV)
    {
        float3 Emission = Material.EmissiveFactor;
        if ((Material.Flags & kRTXPTMaterialFlagHasEmissiveTexture) != 0u)
            Emission *= SampleMaterialTexture(Material.EmissiveTextureIndex, Material.EmissiveTextureSlice, UV).rgb;
        return Emission;
    }

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

#endif // RTXPT_MATERIAL_BRIDGE_HLSLI
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
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.3 material texture and alpha-test bridge helpers" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two bridge headers.

---

### Task 5: Sample Textures In Closest Hit, Add The Alpha-Test Any-Hit, Loosen Ray Flags

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rahit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`

- [ ] **Step 1: Rewrite the closest-hit shader to sample textures**

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
        // Flip the shading normal to face the camera (single-sided diffuse lighting; transmission is deferred).
        if (dot(WorldNormal, WorldRayDirection()) > 0.0)
            WorldNormal = -WorldNormal;

        BaseColor        = Bridge::GetBaseColor(Material, TexCoord).rgb;
        Payload.Emission = Bridge::GetEmission(Material, TexCoord);
    }

    Payload.WorldPos    = WorldPos;
    Payload.WorldNormal = normalize(WorldNormal);
    Payload.BaseColor   = BaseColor;
}

// TODO(RTXPT-Port Phase 5.3): Add metallic-roughness/normal-map shading; current path is textured Lambertian.
// TODO(RTXPT-Port Phase 5.5): Add NEE shadow rays toward analytic and environment lights.
```

- [ ] **Step 2: Create the alpha-test any-hit shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rahit`:

```hlsl
#define RTXPT_ENABLE_HIT_BRIDGE 1
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTMaterialBridge.hlsli"

// Alpha-test any-hit. Only invoked for geometry whose BLAS geometry flag is RAYTRACING_GEOMETRY_FLAG_NONE
// (set for ALPHA_MODE_MASK materials in RTXPTAccelerationStructures::BuildStaticScene). Opaque geometry never
// reaches this shader. Compiled into the hit group only when RTXPT_ENABLE_MATERIAL_TEXTURES is defined.
[shader("anyhit")]
void main(inout RTXPTPathTracerPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
    if (!Bridge::HasSubInstanceTable() || !Bridge::HasMaterialTable())
        return; // No material data - accept the hit.

    const RTXPTSubInstanceData SubInstance = Bridge::GetSubInstanceData();
    const RTXPTMaterialData    Material    = Bridge::GetMaterial(SubInstance.MaterialID);

    if ((Material.Flags & kRTXPTMaterialFlagAlphaTested) == 0u)
        return; // Not alpha tested - accept.

    RTXPTVertex V0;
    RTXPTVertex V1;
    RTXPTVertex V2;
    Bridge::GetTriangleVertices(SubInstance, PrimitiveIndex(), V0, V1, V2);
    const float2 TexCoord = Bridge::InterpolateTexCoord(V0, V1, V2, Attributes.barycentrics);

    if (!Bridge::AlphaTestPasses(Material, TexCoord))
        IgnoreHit();
}

// TODO(RTXPT-Port Phase 5.3): Honor ALPHA_MODE_BLEND (stochastic transparency) in addition to ALPHA_MODE_MASK.
```

- [ ] **Step 3: Trace with `RAY_FLAG_NONE` so the any-hit can run**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`, locate:

```hlsl
        TraceRay(g_TLAS,
                 RAY_FLAG_FORCE_OPAQUE,
                 0xFF,
                 0,
                 1,
                 0,
                 Ray,
                 Payload);
```

Replace it with:

```hlsl
        // RAY_FLAG_NONE lets the alpha-test any-hit shader run for non-opaque (alpha-masked) geometry.
        // Opaque geometry still skips any-hit via its BLAS RAYTRACING_GEOMETRY_FLAG_OPAQUE flag.
        TraceRay(g_TLAS,
                 RAY_FLAG_NONE,
                 0xFF,
                 0,
                 1,
                 0,
                 Ray,
                 Payload);
```

- [ ] **Step 4: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/assets/shaders/RTXPTReference.rahit Samples/RTXPT/assets/shaders/RTXPTReference.rgen
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit the shader changes**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/assets/shaders/RTXPTReference.rahit Samples/RTXPT/assets/shaders/RTXPTReference.rgen
git -C DiligentSamples commit -m "feat(rtxpt): sample textures and add phase 5.3 alpha-test any-hit shader" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the three shader files (one new).

---

### Task 6: Bind Material Textures And The Any-Hit In The Ray Tracing Pass

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

Context: this follows the proven Tutorial21_RayTracing pattern — a bounded `Texture2DArray g_MaterialTextures[N]` (MUTABLE) bound with `IShaderResourceVariable::SetArray`, an immutable `g_MaterialSampler`, and `NonUniformResourceIndex` in the shader. The textured path requires no special device feature beyond what `BindlessResources` (descriptor indexing) provides; the sample (Task 7) gates the `EnableMaterialTextures` argument on that capability and retries without textures on failure.

- [ ] **Step 1: Extend the pass stats and `Initialize` signature**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`, locate:

```cpp
struct RTXPTRayTracingPassStats
{
    bool        Ready                = false;
    bool        LastTraceExecuted    = false;
    bool        MaterialBridgeBound  = false;
    bool        SubInstanceBound     = false;
    bool        LightBridgeBound     = false;
    bool        VertexBufferBound    = false;
    bool        IndexBufferBound     = false;
    bool        AccumulationBound    = false;
    Uint32      TraceCount           = 0;
    std::string DisabledReason;
    std::string LastError;
};
```

Replace it with:

```cpp
struct RTXPTRayTracingPassStats
{
    bool        Ready                 = false;
    bool        LastTraceExecuted     = false;
    bool        MaterialBridgeBound   = false;
    bool        SubInstanceBound      = false;
    bool        LightBridgeBound      = false;
    bool        VertexBufferBound     = false;
    bool        IndexBufferBound      = false;
    bool        AccumulationBound     = false;
    bool        MaterialTexturesBound = false;
    bool        AnyHitEnabled         = false;
    Uint32      MaterialTextureCount  = 0;
    Uint32      TraceCount            = 0;
    std::string DisabledReason;
    std::string LastError;
};
```

Then locate the `Initialize` declaration:

```cpp
    bool Initialize(IRenderDevice*  pDevice,
                    IDeviceContext* pContext,
                    IEngineFactory* pEngineFactory,
                    IBuffer*        pFrameConstants,
                    IBuffer*        pMaterialBuffer,
                    IBuffer*        pSubInstanceBuffer,
                    IBuffer*        pLightBuffer,
                    IBuffer*        pVertexBuffer,
                    IBuffer*        pIndexBuffer,
                    VALUE_TYPE      IndexValueType,
                    ITopLevelAS*    pTLAS,
                    bool            RayTracingSupported,
                    bool            StandaloneRTShadersSupported);
```

Replace it with:

```cpp
    bool Initialize(IRenderDevice*        pDevice,
                    IDeviceContext*       pContext,
                    IEngineFactory*       pEngineFactory,
                    IBuffer*              pFrameConstants,
                    IBuffer*              pMaterialBuffer,
                    IBuffer*              pSubInstanceBuffer,
                    IBuffer*              pLightBuffer,
                    IBuffer*              pVertexBuffer,
                    IBuffer*              pIndexBuffer,
                    VALUE_TYPE            IndexValueType,
                    ITopLevelAS*          pTLAS,
                    IDeviceObject* const* pMaterialTextures,
                    Uint32                MaterialTextureCount,
                    bool                  EnableMaterialTextures,
                    bool                  RayTracingSupported,
                    bool                  StandaloneRTShadersSupported);
```

Also add the `DeviceObject.h` include. Locate:

```cpp
#include "DeviceContext.h"
#include "EngineFactory.h"
```

Replace it with:

```cpp
#include "DeviceContext.h"
#include "DeviceObject.h"
#include "EngineFactory.h"
```

- [ ] **Step 2: Add the `ShaderMacroHelper` include**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`, locate:

```cpp
#include "RTXPTRayTracingPass.hpp"

#include "GraphicsTypesX.hpp"
```

Replace it with:

```cpp
#include "RTXPTRayTracingPass.hpp"

#include "GraphicsTypesX.hpp"
#include "ShaderMacroHelper.hpp"
```

- [ ] **Step 3: Replace the `Initialize` body**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`, replace the entire `RTXPTRayTracingPass::Initialize(...)` function (from its signature line through its closing brace) with:

```cpp
bool RTXPTRayTracingPass::Initialize(IRenderDevice*        pDevice,
                                     IDeviceContext*       pContext,
                                     IEngineFactory*       pEngineFactory,
                                     IBuffer*              pFrameConstants,
                                     IBuffer*              pMaterialBuffer,
                                     IBuffer*              pSubInstanceBuffer,
                                     IBuffer*              pLightBuffer,
                                     IBuffer*              pVertexBuffer,
                                     IBuffer*              pIndexBuffer,
                                     VALUE_TYPE            IndexValueType,
                                     ITopLevelAS*          pTLAS,
                                     IDeviceObject* const* pMaterialTextures,
                                     Uint32                MaterialTextureCount,
                                     bool                  EnableMaterialTextures,
                                     bool                  RayTracingSupported,
                                     bool                  StandaloneRTShadersSupported)
{
    Reset();

    if (!RayTracingSupported)
    {
        m_Stats.DisabledReason = "Ray tracing is not supported by this device";
        return false;
    }

    if (!StandaloneRTShadersSupported)
    {
        m_Stats.DisabledReason = "Standalone ray tracing shaders are not supported by this device";
        return false;
    }

    if (pFrameConstants == nullptr || pTLAS == nullptr)
    {
        m_Stats.DisabledReason = "Frame constants or TLAS are unavailable";
        return false;
    }

    if (pVertexBuffer == nullptr || pIndexBuffer == nullptr)
    {
        m_Stats.DisabledReason = "Vertex or index buffer is unavailable for the reference path tracer";
        return false;
    }

    const bool UseTextures = EnableMaterialTextures && pMaterialTextures != nullptr && MaterialTextureCount > 0;

    m_TLAS = pTLAS;

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders", &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.UseCombinedTextureSamplers = false;
    ShaderCI.SourceLanguage                  = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler                  = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags                    = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.HLSLVersion                     = {6, 5};
    ShaderCI.pShaderSourceStreamFactory      = pShaderSourceFactory;

    // Raygen + miss do not include the material bridge, so they compile without material-texture macros.
    RefCntAutoPtr<IShader> pRayGen;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_GEN;
    ShaderCI.Desc.Name       = "RTXPT reference raygen";
    ShaderCI.FilePath        = "RTXPTReference.rgen";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pRayGen);

    RefCntAutoPtr<IShader> pMiss;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_MISS;
    ShaderCI.Desc.Name       = "RTXPT reference miss";
    ShaderCI.FilePath        = "RTXPTReference.rmiss";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pMiss);

    // The closest-hit and any-hit shaders sample the bindless material-texture table when it is available.
    ShaderMacroHelper Macros;
    if (UseTextures)
    {
        Macros.Add("RTXPT_ENABLE_MATERIAL_TEXTURES", 1);
        Macros.Add("RTXPT_MATERIAL_TEXTURE_COUNT", static_cast<int>(MaterialTextureCount));
    }
    ShaderCI.Macros = Macros;

    RefCntAutoPtr<IShader> pClosestHit;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_CLOSEST_HIT;
    ShaderCI.Desc.Name       = "RTXPT reference closest hit";
    ShaderCI.FilePath        = "RTXPTReference.rchit";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pClosestHit);

    RefCntAutoPtr<IShader> pAnyHit;
    if (UseTextures)
    {
        ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_ANY_HIT;
        ShaderCI.Desc.Name       = "RTXPT reference any hit";
        ShaderCI.FilePath        = "RTXPTReference.rahit";
        ShaderCI.EntryPoint      = "main";
        pDevice->CreateShader(ShaderCI, &pAnyHit);
    }

    if (!pRayGen || !pMiss || !pClosestHit || (UseTextures && !pAnyHit))
    {
        m_Stats.LastError = "Failed to create RTXPT reference ray tracing shaders";
        return false;
    }

    RayTracingPipelineStateCreateInfoX PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT reference RT PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_RAY_TRACING;
    PSOCreateInfo.AddGeneralShader("Main", pRayGen);
    PSOCreateInfo.AddGeneralShader("PrimaryMiss", pMiss);
    if (UseTextures)
        PSOCreateInfo.AddTriangleHitShader("PrimaryHit", pClosestHit, pAnyHit);
    else
        PSOCreateInfo.AddTriangleHitShader("PrimaryHit", pClosestHit);
    PSOCreateInfo.RayTracingPipeline.MaxRecursionDepth = 1; // Raygen drives bounces in a loop; chit/miss/anyhit do not recurse.
    PSOCreateInfo.RayTracingPipeline.ShaderRecordSize  = 0;
    PSOCreateInfo.MaxAttributeSize                     = static_cast<Uint32>(sizeof(float) * 2);
    // RTXPTPathTracerPayload = 4 * float4 = 64 bytes.
    PSOCreateInfo.MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 16);

    // Hit-bridge resources are referenced by the closest-hit shader and (when textured) the any-hit shader.
    const SHADER_TYPE HitStages = UseTextures
        ? (SHADER_TYPE_RAY_CLOSEST_HIT | SHADER_TYPE_RAY_ANY_HIT)
        : SHADER_TYPE_RAY_CLOSEST_HIT;

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_FrameConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_TLAS", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(HitStages, "g_Materials", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(HitStages, "g_SubInstanceData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(HitStages, "g_VertexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(HitStages, "g_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_MISS, "g_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_AccumColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);

    if (UseTextures)
    {
        ResourceLayout.AddVariable(HitStages, "g_MaterialTextures", SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE);

        const SamplerDesc MaterialSamplerDesc{
            FILTER_TYPE_LINEAR, FILTER_TYPE_LINEAR, FILTER_TYPE_LINEAR,
            TEXTURE_ADDRESS_WRAP, TEXTURE_ADDRESS_WRAP, TEXTURE_ADDRESS_WRAP};
        ResourceLayout.AddImmutableSampler(HitStages, "g_MaterialSampler", MaterialSamplerDesc);
    }
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateRayTracingPipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_Stats.LastError = "Failed to create RTXPT reference RT PSO";
        return false;
    }

    auto SetStatic = [&](SHADER_TYPE Stage, const char* Name, IDeviceObject* pObject) {
        if (pObject == nullptr)
            return false;

        IShaderResourceVariable* pVar = m_PSO->GetStaticVariableByName(Stage, Name);
        if (pVar == nullptr)
            return false;

        pVar->Set(pObject);
        return true;
    };

    const bool FrameConstantsBound = SetStatic(SHADER_TYPE_RAY_GEN, "g_FrameConstants", pFrameConstants);
    const bool TLASBound           = SetStatic(SHADER_TYPE_RAY_GEN, "g_TLAS", m_TLAS);

    if (!FrameConstantsBound || !TLASBound)
    {
        m_Stats.LastError = "Failed to bind required RTXPT frame constants or TLAS";
        return false;
    }

    IDeviceObject* pMaterialsView   = pMaterialBuffer != nullptr ? pMaterialBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
    IDeviceObject* pSubInstanceView = pSubInstanceBuffer != nullptr ? pSubInstanceBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
    IDeviceObject* pLightsView      = pLightBuffer != nullptr ? pLightBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
    IDeviceObject* pVertexView      = pVertexBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE);

    // The GLTF loader creates the index buffer in BUFFER_MODE_FORMATTED but does not pre-create a typed view;
    // create one here so HLSL can declare it as Buffer<uint>.
    if (IndexValueType != VT_UINT16 && IndexValueType != VT_UINT32)
    {
        m_Stats.LastError = "Reference path tracer requires VT_UINT16 or VT_UINT32 indices";
        return false;
    }
    BufferViewDesc IndexViewDesc;
    IndexViewDesc.Name                 = "RTXPT reference index buffer SRV";
    IndexViewDesc.ViewType             = BUFFER_VIEW_SHADER_RESOURCE;
    IndexViewDesc.Format.ValueType     = IndexValueType;
    IndexViewDesc.Format.NumComponents = 1;
    IndexViewDesc.Format.IsNormalized  = false;
    pIndexBuffer->CreateView(IndexViewDesc, &m_IndexBufferView);
    if (!m_IndexBufferView)
    {
        m_Stats.LastError = "Failed to create RTXPT index buffer view";
        return false;
    }

    m_Stats.MaterialBridgeBound = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_Materials", pMaterialsView);
    m_Stats.SubInstanceBound    = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_SubInstanceData", pSubInstanceView);
    m_Stats.LightBridgeBound    = SetStatic(SHADER_TYPE_RAY_MISS, "g_Lights", pLightsView);
    m_Stats.VertexBufferBound   = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_VertexBuffer", pVertexView);
    m_Stats.IndexBufferBound    = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_IndexBuffer", m_IndexBufferView);

    if (!m_Stats.MaterialBridgeBound || !m_Stats.SubInstanceBound || !m_Stats.LightBridgeBound ||
        !m_Stats.VertexBufferBound || !m_Stats.IndexBufferBound)
    {
        m_Stats.LastError = "Failed to bind required RTXPT bridge buffers";
        return false;
    }

    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_Stats.LastError = "Failed to create RTXPT reference RT SRB";
        return false;
    }

    if (UseTextures)
    {
        IShaderResourceVariable* pTexVar = m_SRB->GetVariableByName(SHADER_TYPE_RAY_CLOSEST_HIT, "g_MaterialTextures");
        if (pTexVar == nullptr)
        {
            m_Stats.LastError = "Failed to find RTXPT material texture array binding";
            return false;
        }
        pTexVar->SetArray(pMaterialTextures, 0, MaterialTextureCount);
        m_Stats.MaterialTexturesBound = true;
        m_Stats.MaterialTextureCount  = MaterialTextureCount;
    }
    m_Stats.AnyHitEnabled = UseTextures;

    ShaderBindingTableDesc SBTDesc;
    SBTDesc.Name = "RTXPT reference SBT";
    SBTDesc.pPSO = m_PSO;
    pDevice->CreateSBT(SBTDesc, &m_SBT);
    if (!m_SBT)
    {
        m_Stats.LastError = "Failed to create RTXPT reference SBT";
        return false;
    }

    m_SBT->BindRayGenShader("Main");
    m_SBT->BindMissShader("PrimaryMiss", 0);
    m_SBT->BindHitGroupForTLAS(m_TLAS, 0, "PrimaryHit");
    pContext->UpdateSBT(m_SBT);

    m_Stats.Ready = true;
    return true;
}
```

(The `Trace(...)` and `Reset()` functions are unchanged.)

- [ ] **Step 4: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit the ray tracing pass wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): bind phase 5.3 material textures in the rt pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the ray tracing pass files.

---

### Task 7: Wire The Sample, UI Diagnostics, And CMake

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Pass the texture table and gate on bindless support (with a fallback retry)**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate the body of `RTXPTSample::CreatePhase4Passes`:

```cpp
void RTXPTSample::CreatePhase4Passes()
{
    m_BlitPass.Initialize(m_pDevice, m_pEngineFactory, m_pSwapChain);

    m_RayTracingPass.Initialize(m_pDevice,
                                m_pImmediateContext,
                                m_pEngineFactory,
                                m_FrameConstantsCB,
                                m_Materials.GetMaterialBuffer(),
                                m_AccelerationStructures.GetSubInstanceBuffer(),
                                m_Lights.GetLightBuffer(),
                                m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
                                m_Scene.GetIndexBuffer(m_pDevice, m_pImmediateContext),
                                m_Scene.GetIndexType(),
                                m_AccelerationStructures.GetTLAS(),
                                m_FeatureCaps.RayTracing,
                                m_FeatureCaps.StandaloneRayTracingShaders);

    m_DebugComputePass.Initialize(m_pDevice,
                                  m_pEngineFactory,
                                  "RTXPT debug compute pass",
                                  "RTXPTDebugCompute.csh",
                                  m_FrameConstantsCB,
                                  m_FeatureCaps.ComputeShaders);
}
```

Replace it with:

```cpp
void RTXPTSample::CreatePhase4Passes()
{
    m_BlitPass.Initialize(m_pDevice, m_pEngineFactory, m_pSwapChain);

    // Material-texture sampling needs descriptor indexing. Gate it on bindless support; if the textured
    // pipeline fails to build, fall back to the Phase 5.2 factor-only path so the sample still renders.
    const bool EnableMaterialTextures = m_FeatureCaps.BindlessResources && m_Materials.GetTextureCount() > 0;

    const bool RTReady =
        m_RayTracingPass.Initialize(m_pDevice,
                                    m_pImmediateContext,
                                    m_pEngineFactory,
                                    m_FrameConstantsCB,
                                    m_Materials.GetMaterialBuffer(),
                                    m_AccelerationStructures.GetSubInstanceBuffer(),
                                    m_Lights.GetLightBuffer(),
                                    m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
                                    m_Scene.GetIndexBuffer(m_pDevice, m_pImmediateContext),
                                    m_Scene.GetIndexType(),
                                    m_AccelerationStructures.GetTLAS(),
                                    m_Materials.GetTextureBindings(),
                                    m_Materials.GetTextureCount(),
                                    EnableMaterialTextures,
                                    m_FeatureCaps.RayTracing,
                                    m_FeatureCaps.StandaloneRayTracingShaders);

    if (!RTReady && EnableMaterialTextures)
    {
        m_RayTracingPass.Initialize(m_pDevice,
                                    m_pImmediateContext,
                                    m_pEngineFactory,
                                    m_FrameConstantsCB,
                                    m_Materials.GetMaterialBuffer(),
                                    m_AccelerationStructures.GetSubInstanceBuffer(),
                                    m_Lights.GetLightBuffer(),
                                    m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
                                    m_Scene.GetIndexBuffer(m_pDevice, m_pImmediateContext),
                                    m_Scene.GetIndexType(),
                                    m_AccelerationStructures.GetTLAS(),
                                    nullptr,
                                    0,
                                    false,
                                    m_FeatureCaps.RayTracing,
                                    m_FeatureCaps.StandaloneRayTracingShaders);
    }

    m_DebugComputePass.Initialize(m_pDevice,
                                  m_pEngineFactory,
                                  "RTXPT debug compute pass",
                                  "RTXPTDebugCompute.csh",
                                  m_FrameConstantsCB,
                                  m_FeatureCaps.ComputeShaders);
}
```

- [ ] **Step 2: Add the alpha-tested geometry count to the UI**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, inside `UpdateUI()`, locate:

```cpp
    ImGui::Text("Sub-instances: %u", ASStats.SubInstanceCount);
    if (!ASStats.DisabledReason.empty())
```

Replace it with:

```cpp
    ImGui::Text("Sub-instances: %u", ASStats.SubInstanceCount);
    ImGui::Text("Alpha-tested geometries: %u", ASStats.AlphaTestedGeometryCount);
    if (!ASStats.DisabledReason.empty())
```

- [ ] **Step 3: Add the material-texture diagnostics to the UI**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, inside `UpdateUI()`, locate:

```cpp
    ImGui::Text("Vertex buffer: %s", RTPassStats.VertexBufferBound ? "bound" : "fallback");
    ImGui::Text("Index buffer: %s", RTPassStats.IndexBufferBound ? "bound" : "fallback");
    ImGui::Text("Accumulation target: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA8 fallback)");
```

Replace it with:

```cpp
    ImGui::Text("Vertex buffer: %s", RTPassStats.VertexBufferBound ? "bound" : "fallback");
    ImGui::Text("Index buffer: %s", RTPassStats.IndexBufferBound ? "bound" : "fallback");
    ImGui::Text("Material textures loaded: %u", m_Materials.GetStats().TextureCount);
    ImGui::Text("Material textures bound: %s (%u)", RTPassStats.MaterialTexturesBound ? "yes" : "no", RTPassStats.MaterialTextureCount);
    ImGui::Text("Alpha-test any-hit: %s", RTPassStats.AnyHitEnabled ? "enabled" : "disabled");
    ImGui::Text("Accumulation target: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA8 fallback)");
```

- [ ] **Step 4: Re-target the Phase 5.3 BSDF TODO line**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, inside `UpdateUI()`, locate:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.3): swap flat Lambertian for the full BSDF (GGX + transmission + alpha-test).");
```

Replace it with:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.3): shade with the metallic-roughness GGX BSDF + normal maps (current path is textured Lambertian + alpha test).");
```

- [ ] **Step 5: Register the any-hit shader in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, locate:

```cmake
    assets/shaders/RTXPTReference.rgen
    assets/shaders/RTXPTReference.rmiss
    assets/shaders/RTXPTReference.rchit
    assets/shaders/RTXPTDebugCompute.csh
```

Replace it with:

```cmake
    assets/shaders/RTXPTReference.rgen
    assets/shaders/RTXPTReference.rmiss
    assets/shaders/RTXPTReference.rchit
    assets/shaders/RTXPTReference.rahit
    assets/shaders/RTXPTDebugCompute.csh
```

- [ ] **Step 6: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/CMakeLists.txt
```

Expected: no output and exit code 0.

- [ ] **Step 7: Commit the sample wiring and CMake update**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 5.3 material textures and any-hit into the sample" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the sample and CMake changes.

---

### Task 8: Phase 5.3 Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [ ] **Step 1: Confirm structured Phase 5 TODOs**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5" DiligentSamples/Samples/RTXPT
```

Expected after this plan (the resolved alpha-test / base-color-texture markers are gone; deferred refinement markers remain):

```text
RTXPTMaterialBridge.hlsli : Phase 5.3 GGX/normal maps; Phase 5.3 UV selectors/wrap/atlas
RTXPTReference.rchit      : Phase 5.3 metallic-roughness/normal-map shading; Phase 5.5 NEE
RTXPTReference.rahit      : Phase 5.3 ALPHA_MODE_BLEND
RTXPTReference.rgen       : Phase 5.3 BSDF; Phase 5.5 NEE/MIS; Phase 6 tone mapping
RTXPTSample.cpp           : Phase 5.3 GGX BSDF + normal maps; Phase 5.5 NEE
```

No `Phase 5.3` markers should remain in `RTXPTSceneBridge.hlsli` (both were removed in Task 4).

- [ ] **Step 2: Confirm file registration and the new any-hit file**

Run:

```powershell
rg -n "RTXPTReference.rahit" DiligentSamples/Samples/RTXPT/CMakeLists.txt
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rahit
```

Expected: one match in `CMakeLists.txt` and `True`.

- [ ] **Step 3: Confirm the DiligentSamples log shows the Phase 5.3 commits**

Run:

```powershell
git -C DiligentSamples log --oneline -n 8
```

Expected (most recent first):

```text
feat(rtxpt): wire phase 5.3 material textures and any-hit into the sample
feat(rtxpt): bind phase 5.3 material textures in the rt pass
feat(rtxpt): sample textures and add phase 5.3 alpha-test any-hit shader
feat(rtxpt): add phase 5.3 material texture and alpha-test bridge helpers
feat(rtxpt): set phase 5.3 alpha-test geometry flags from material data
feat(rtxpt): mirror phase 5.3 material data layout in shared header
feat(rtxpt): build phase 5.3 gpu material data with texture indices
fix(rtxpt): restore full bistro scene visibility
```

- [ ] **Step 4: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: exit code 0. If the build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 5: Optional D3D12 runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with D3D12 on a standalone-RT + bindless-capable machine. Expected ImGui facts:

```text
Bindless: yes
Acceleration structures: built
Alpha-tested geometries: > 0 (the bistro has masked foliage/fences)
TraceRays pass: ready
Material bridge / Sub-instance bridge / Light bridge / Vertex buffer / Index buffer: bound
Material textures loaded: > 0
Material textures bound: yes (N) where N == "Material textures loaded"
Alpha-test any-hit: enabled
Accumulation target: active (RGBA32F)
TraceRays executed: yes; TraceRays count increases every frame
```

Expected visual result: the bistro scene rendered with textured Lambertian base color and emissive surfaces (e.g. lamps/signs glow), converging across accumulated frames. Alpha-masked geometry (foliage, fences, decals) shows cut-out silhouettes through the alpha test rather than solid quads. Adjusting "Max bounces" or "Reset accumulation" restarts convergence.

- [ ] **Step 6: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT + bindless-capable machine. Expected facts match the D3D12 run.

If the device reports `Bindless: no` (or the model loaded zero textures), expected fallback facts are:

```text
Material textures loaded: 0  (or Bindless: no)
Material textures bound: no (0)
Alpha-test any-hit: disabled
TraceRays pass: ready
```

with the Phase 5.2 factor-only image (base color from `BaseColorFactor`, no cut-outs). The sample must still render and converge.

If standalone ray tracing shaders are unavailable, expected facts:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
```

and the sample clears the swapchain via `ClearFallback`.

- [ ] **Step 7: Commit the top-level submodule pointer and plan**

After all `DiligentSamples` Phase 5.3 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-29-rtxpt-phase5-3-material-alpha-anyhit.md
git commit -m "feat(samples): plan and add RTXPT phase 5.3 material alpha test" -m "Co-Authored-By: GPT 5.5"
```

Expected: one top-level commit that records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] **Spec coverage.** This plan implements the runnable core of Phase 5 layer 5 (`docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`): "Material specialization, alpha test and any-hit." Material specialization is realized as material-flag-driven opaque/non-opaque geometry + a single über closest-hit/any-hit pair (matching RTXPT's IntroSample); alpha test + any-hit are fully implemented. The deferred layer-5 refinements (GGX/normal maps, `ALPHA_MODE_BLEND`, UV selectors/wrap/atlas, per-material permutations) are each preserved as `TODO(RTXPT-Port Phase 5.3)` markers and named in the Scope Note. Layers 6–9 stay in their own future plans.
- [x] **Runnable increments.** Every task ends with a focused commit. When `BindlessResources` is unavailable, the textured shaders/any-hit are not built and the sample renders the Phase 5.2 factor-only image; when the textured PSO fails to build, `CreatePhase4Passes` retries the factor-only path; when ray tracing / standalone RT is unavailable, the existing `ClearFallback` path still runs. The accumulation/blit chain is untouched.
- [x] **Single source of truth.** `RTXPTMaterialData` is defined once in C++ (`RTXPTMaterials.hpp`, `static_assert(sizeof==64)`) and mirrored once in HLSL (`RTXPTShaderShared.hlsli`); offsets are annotated and match (BaseColorFactor@0, EmissiveFactor@16, AlphaCutoff@28, Flags@32, BaseColorTextureIndex@36, EmissiveTextureIndex@40, MetallicFactor@44, RoughnessFactor@48, BaseColorTextureSlice@52, EmissiveTextureSlice@56, Padding0@60). Flag bits match (`HasBaseColorTexture=0x1`, `AlphaTested=0x2`, `HasEmissiveTexture=0x4`). The alpha-test predicate lives in one helper (`RTXPTMaterialIsAlphaTested`) consumed by both `RTXPTMaterials` (material flag) and `RTXPTAccelerationStructures` (geometry flag).
- [x] **Type/name consistency.** `RTXPTMaterials::GetTextureBindings()` returns `IDeviceObject* const*` and `GetTextureCount()` returns `Uint32`; the new `RTXPTRayTracingPass::Initialize` parameters `pMaterialTextures` / `MaterialTextureCount` / `EnableMaterialTextures` appear in the same order in the header (Task 6 Step 1), the implementation (Task 6 Step 3), and both call sites (Task 7 Step 1). `Bridge::InterpolateTexCoord` (Task 4 Step 1) is consumed by `RTXPTReference.rchit` and `RTXPTReference.rahit` (Task 5). `Bridge::GetBaseColor`/`GetEmission`/`AlphaTestPasses` (Task 4 Step 2) are declared with both textured and `#else` factor-only variants and consumed unconditionally by the shaders. The HLSL flag constant `kRTXPTMaterialFlagAlphaTested` matches the C++ `kRTXPTMaterialFlag_AlphaTested` semantics.
- [x] **Binding model is proven.** The bindless `Texture2DArray g_MaterialTextures[RTXPT_MATERIAL_TEXTURE_COUNT]` + immutable `g_MaterialSampler` + `SetArray` + `NonUniformResourceIndex(idx)` + `SampleLevel(...,0)` pattern mirrors `DiligentSamples/Tutorials/Tutorial21_RayTracing` (`CubePrimaryHit.rchit`, `Tutorial21_RayTracing.cpp`), which runs on both D3D12 and Vulkan. The GLTF non-atlas texture default SRV is `Texture2DArray` (`GLTFLoader.cpp:975`), with `TextureSlice=0` and identity `AtlasUVScaleAndBias` — so `float3(uv,0)` sampling is correct.
- [x] **AS contract preserved.** `RTXPTAccelerationStructures` keeps `HIT_GROUP_BINDING_MODE_PER_GEOMETRY`, `HitGroupStride=1`, `CustomId` sub-instance bases, and the 32-byte `RTXPTSubInstanceData` from Phase 5.2 unchanged; Phase 5.3 only sets per-geometry opaque flags from material alpha mode and adds a diagnostic counter.
- [x] **No placeholders.** Every code step shows complete code; every command shows expected output. No "TODO/implement later" instructions; the only `TODO(...)` strings are the intentional, structured open-work markers required by the spec's TODO policy.
- [x] **House style honored.** Verification avoids build/runtime execution unless the user explicitly asks (per `CLAUDE.md`); each task is a single-purpose commit using the established `Co-Authored-By: GPT 5.5` trailer (matching every prior RTXPT commit); copyright dates stay `2026`; the obsolete `RTXPTMaterialAttribs` mirror is replaced (not left as dead code).
