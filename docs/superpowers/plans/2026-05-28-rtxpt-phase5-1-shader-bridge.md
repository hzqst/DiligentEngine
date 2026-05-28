# RTXPT Phase 5.1 Shared Shader Infrastructure And Scene/Material Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land Phase 5 shader layers 2 and 3 from the RTXPT port design — shared HLSL structs/constants/binding declarations plus the scene/material bridge — so the existing minimal ray tracing pass reads per-instance/per-geometry material attributes from the C++ scene data and shows real material base color on hits.

**Architecture:** Add three shared HLSL headers (`RTXPTShaderShared.hlsli`, `RTXPTSceneBridge.hlsli`, `RTXPTMaterialBridge.hlsli`) that mirror the C++ Phase 3 buffer layouts and centralize how shaders fetch sub-instance, material, and light data. Extend `RTXPTAccelerationStructures` to build a per-(instance, geometry) `SubInstanceData` buffer that maps `InstanceID() + GeometryIndex()` to a material id, where `InstanceID()` is the TLAS `CustomId` sub-instance base written by C++. Extend `RTXPTRayTracingPass::Initialize` to bind the material, sub-instance, and light structured buffers as static shader resources, and replace `RTXPTMinimal.rchit` with a bridge-driven flat-shaded variant that falls back to barycentric debug coloring when shader-side bridge table checks fail.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs, DiligentTools `GLTFLoader` (`GLTF::Material::ShaderAttribs`), HLSL 6.5 ray tracing shaders compiled by DXC, Diligent shader resource binding APIs, Dear ImGui.

---

## Scope Note: Phase 5 Sub-Plan Series

Phase 5 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md` enumerates nine shader dependency layers. This plan covers **only** layers 2 and 3 (shared declarations and scene/material bridge) so the increment stays runnable and reviewable. The remaining Phase 5 layers should each get their own plan in a later session:

- Phase 5.2: Reference path tracer core (Layer 4).
- Phase 5.3: Material specialization, alpha test, any-hit (Layer 5).
- Phase 5.4: Stable planes and realtime mode (Layer 6).
- Phase 5.5: RTXDI shader bridge and passes (Layer 7).
- Phase 5.6: NRD, denoising guides, post-process (Layer 8).
- Phase 5.7: NVAPI, SER, OMM, DLSS-related shader variants (Layer 9).

## Baseline

Current Phase 4 state in `DiligentSamples/Samples/RTXPT`:

- `RTXPTSample::Render()` runs the minimal RT/compute/blit chain and shows status in the ImGui panel.
- `RTXPTAccelerationStructures::BuildStaticScene` builds BLAS per mesh node and a single TLAS with `HIT_GROUP_BINDING_MODE_PER_GEOMETRY` and `HitGroupStride = 1`. Per-instance `CustomId` is the linear instance index. `InstanceContributionToHitGroupIndex` is computed automatically by Diligent.
- `RTXPTMaterials::Upload` produces a `StructuredBuffer<GLTF::Material::ShaderAttribs>` (`m_MaterialBuffer`).
- `RTXPTLights::Upload` produces a `StructuredBuffer<RTXPTLightData>` (`m_LightBuffer`).
- `RTXPTRayTracingPass::Initialize` binds `g_FrameConstants` (static), `g_TLAS` (static), `g_OutputColor` (dynamic) only.
- `RTXPTMinimal.rgen` traces a primary ray; `RTXPTMinimal.rchit` writes a barycentric/depth debug color; `RTXPTMinimal.rmiss` writes a sky gradient. There are no shared HLSL headers besides `RTXPTCommon.fxh`.

This plan assumes the Phase 4 submodule changes are already committed and the top-level repository starts clean.

---

## Scope

This plan implements Phase 5.1 runnable milestone:

- Add shared HLSL infrastructure: a single source of truth for frame constants, ray payload, material struct, sub-instance struct, and light struct.
- Add `RTXPTSubInstanceData` buffer construction (per-(instance, geometry) material id) inside `RTXPTAccelerationStructures`.
- Extend `RTXPTRayTracingPass::Initialize` to bind the material buffer, sub-instance buffer, and light buffer as static structured-buffer SRVs.
- Replace `RTXPTMinimal.rchit` with a bridge-driven hit shader that reads `g_SubInstanceData[InstanceID() + GeometryIndex()].MaterialID`, looks up `g_Materials[MaterialID].BaseColorFactor`, and writes a stable color.
- Preserve a barycentric-fallback path inside the new `rchit` when shader-side bridge table checks fail; C++ treats missing required bridge SRV bindings as initialization failure instead of tracing with unbound resources.
- Keep `RTXPTMinimal.rgen` and `RTXPTMinimal.rmiss` essentially the same; only switch their includes to use the shared bridge.
- Add ImGui status lines for sub-instance count and bridge bindings.
- Add structured `TODO(RTXPT-Port Phase 5)` markers for the bridge work that is intentionally deferred (textures, normals/UVs, alpha test, transmissive materials, light sampling).

This plan intentionally does not:

- Sample any material textures or vertex attributes beyond positions.
- Implement multi-bounce path tracing or accumulation.
- Add alpha test, any-hit, or material-specialized hit groups.
- Touch stable planes, RTXDI, NRD, DLSS, NVAPI, SER, or OMM.
- Run automated builds or runtime execution; build/runtime steps are listed for explicit user request only.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Register the three new HLSL bridge headers.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`
  - HLSL definitions of `RTXPTFrameConstants`, `RTXPTPrimaryPayload`, `RTXPTSubInstanceData`, `RTXPTMaterialAttribs`, `RTXPTLightData`. Single source of truth for layouts shared with C++.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
  - Sub-instance access helpers (`Bridge_GetSubInstanceIndex`, `Bridge_GetSubInstanceData`, `Bridge_HasSubInstanceTable`).
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`
  - Material attribute access helpers (`Bridge_GetMaterial`, `Bridge_GetMaterialBaseColor`, `Bridge_HasMaterialTable`).
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh`
  - Forward `RTXPTShaderShared.hlsli` as the single declaration source. Keep the file for backward include compatibility.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
  - Include `RTXPTSceneBridge.hlsli` to bring in the shared structs/payload.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`
  - Include `RTXPTSceneBridge.hlsli`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`
  - Switch from barycentric-only debug coloring to bridge-driven base color with barycentric fallback.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
  - Add `RTXPTSubInstanceData` struct, sub-instance buffer member, accessors, and stat counter.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
  - Populate the sub-instance buffer during `BuildStaticScene` and expose it.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
  - Always upload at least one default material entry so the bridge SRV is never null.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
  - Always upload at least one default light entry so the bridge SRV is never null.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
  - Add bridge buffer arguments to `Initialize`. Track whether bridge bindings are available.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
  - Bind `g_Materials`, `g_SubInstanceData`, `g_Lights` as static structured buffers.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Pass bridge buffers to `RTXPTRayTracingPass::Initialize` and surface status in UI.

---

### Task 0: Phase 4 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [x] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected:

```text
## RTXPT...origin/RTXPT
```

If other unrelated files appear, leave them unstaged and continue only if they do not overlap `docs/superpowers/plans` or `DiligentSamples/Samples/RTXPT`.

- [x] **Step 2: Confirm DiligentSamples Phase 4 state**

Run:

```powershell
git -C DiligentSamples status --short --branch
rg -n "TODO\(RTXPT-Port Phase 4\)" DiligentSamples/Samples/RTXPT
```

Expected:

```text
DiligentSamples branch is clean.
Multiple Phase 4 TODOs are still present in `RTXPTRayTracingPass.cpp`, `RTXPTComputePass.cpp`, and `RTXPTSample.cpp`.
```

- [x] **Step 3: Confirm Phase 5 bridge files do not exist yet**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
```

Expected:

```text
False
False
False
```

If any returns `True`, inspect the file before overwriting and preserve unrelated user work.

---

### Task 1: Add Shared HLSL Bridge Headers

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh`

- [x] **Step 1: Create the shared declarations header**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`:

```hlsl
#ifndef RTXPT_SHADER_SHARED_HLSLI
#define RTXPT_SHADER_SHARED_HLSLI

// Mirrors RTXPTFrameConstants in RTXPTSample.hpp.
struct RTXPTFrameConstants
{
    float4x4 ViewProj;
    float4x4 ViewProjInv;
    float4   CameraPosition_Time;
    float4   ViewportSize_FrameIdx;
};

// Primary ray payload shared by raygen/miss/chit.
struct RTXPTPrimaryPayload
{
    float4 ColorDepth;
};

// Mirrors RTXPTSubInstanceData in RTXPTAccelerationStructures.hpp.
// One entry per (BLAS instance, geometry) pair. C++ stores the per-instance
// sub-instance base in TLAS CustomId, exposed to closest-hit shaders as InstanceID().
// index = InstanceID() + GeometryIndex().
struct RTXPTSubInstanceData
{
    uint MaterialID;
    uint Flags;        // Reserved for Phase 5.3 alpha mode/any-hit specialization.
    uint Padding0;
    uint Padding1;
};

// Mirrors Diligent::GLTF::Material::ShaderAttribs from DiligentTools/AssetLoader/interface/GLTFLoader.hpp.
// Keep field order/sizes synchronized; total size is 96 bytes (16-byte aligned).
struct RTXPTMaterialAttribs
{
    float4 BaseColorFactor;       // offset 0

    float3 EmissiveFactor;        // offset 16
    float  NormalScale;

    float3 SpecularFactor;        // offset 32
    float  ClearcoatNormalScale;

    int    Workflow;              // offset 48
    int    AlphaMode;
    float  AlphaCutoff;
    float  MetallicFactor;

    float  RoughnessFactor;       // offset 64
    float  OcclusionFactor;
    float  ClearcoatFactor;
    float  ClearcoatRoughnessFactor;

    float4 CustomData;            // offset 80
};

// Mirrors Diligent::RTXPTLightData in RTXPTLights.hpp.
struct RTXPTLightData
{
    float4 ColorIntensity;
    float4 PositionRange;
    float4 DirectionType;
    float4 SpotAngles;
};

#endif // RTXPT_SHADER_SHARED_HLSLI
```

- [x] **Step 2: Create the scene bridge header**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`:

```hlsl
#ifndef RTXPT_SCENE_BRIDGE_HLSLI
#define RTXPT_SCENE_BRIDGE_HLSLI

#include "RTXPTShaderShared.hlsli"

// Global shader resources used by the scene bridge. C++ binds these as static SRVs.
ConstantBuffer<RTXPTFrameConstants>      g_FrameConstants;
StructuredBuffer<RTXPTSubInstanceData>   g_SubInstanceData;
StructuredBuffer<RTXPTLightData>         g_Lights;

namespace Bridge
{
#ifdef RTXPT_ENABLE_HIT_BRIDGE
    // Linear index for the SubInstanceData entry that describes the currently hit (instance, geometry).
    // C++ stores the per-instance sub-instance base in InstanceID(), and GeometryIndex() is used to
    // select the geometry within the BLAS.
    uint GetSubInstanceIndex()
    {
        return InstanceID() + GeometryIndex();
    }

    // Returns the SubInstanceData entry for the current hit.
    // The caller is responsible for guarding against an empty/unbound table via HasSubInstanceTable().
    RTXPTSubInstanceData GetSubInstanceData()
    {
        return g_SubInstanceData[GetSubInstanceIndex()];
    }

    // True when g_SubInstanceData has at least one entry. The C++ side guarantees a dummy entry
    // is bound when the scene has no real geometry so that this helper still returns a defined value.
    bool HasSubInstanceTable()
    {
        uint Count = 0;
        uint Stride = 0;
        g_SubInstanceData.GetDimensions(Count, Stride);
        return Count > 0;
    }
#endif

    // Total active light count. May be zero on scenes without lights.
    uint GetLightCount()
    {
        uint Count = 0;
        uint Stride = 0;
        g_Lights.GetDimensions(Count, Stride);
        return Count;
    }

    RTXPTLightData GetLight(uint Index)
    {
        return g_Lights[Index];
    }
}

// TODO(RTXPT-Port Phase 5.2): Add reference-path-tracer scene accessors: per-vertex normal/UV fetch, ray cone construction, and tangent frame reconstruction.
// TODO(RTXPT-Port Phase 5.3): Add alpha-mask/transparent flags to RTXPTSubInstanceData and propagate them into any-hit specialization.

#endif // RTXPT_SCENE_BRIDGE_HLSLI
```

- [x] **Step 3: Create the material bridge header**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli`:

```hlsl
#ifndef RTXPT_MATERIAL_BRIDGE_HLSLI
#define RTXPT_MATERIAL_BRIDGE_HLSLI

#include "RTXPTShaderShared.hlsli"

StructuredBuffer<RTXPTMaterialAttribs> g_Materials;

namespace Bridge
{
    bool HasMaterialTable()
    {
        uint Count = 0;
        uint Stride = 0;
        g_Materials.GetDimensions(Count, Stride);
        return Count > 0;
    }

    uint GetMaterialCount()
    {
        uint Count = 0;
        uint Stride = 0;
        g_Materials.GetDimensions(Count, Stride);
        return Count;
    }

    // Out-of-range indices clamp to the last material so a bad MaterialID never UB-reads.
    RTXPTMaterialAttribs GetMaterial(uint MaterialID)
    {
        const uint LastIndex = max(GetMaterialCount(), 1u) - 1u;
        const uint Index     = min(MaterialID, LastIndex);
        return g_Materials[Index];
    }

    float4 GetMaterialBaseColor(uint MaterialID)
    {
        return GetMaterial(MaterialID).BaseColorFactor;
    }
}

// TODO(RTXPT-Port Phase 5.2): Replace the flat base color helper with a per-hit BSDF sampler that reads roughness/metallic/normal scale/IOR and feeds the reference path tracer.
// TODO(RTXPT-Port Phase 5.3): Bind material textures (base color, normal, MR, emissive, occlusion) and expose helpers that respect TextureShaderAttribs UV selectors and wrap modes.

#endif // RTXPT_MATERIAL_BRIDGE_HLSLI
```

- [x] **Step 4: Replace `RTXPTCommon.fxh` body so it re-exports the shared header**

Replace the contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh` with:

```hlsl
#ifndef RTXPT_COMMON_FXH
#define RTXPT_COMMON_FXH

// Phase 5.1: RTXPTCommon.fxh is now a thin compatibility wrapper around the bridge headers.
#include "RTXPTShaderShared.hlsli"

ConstantBuffer<RTXPTFrameConstants> g_FrameConstants;

#endif
```

Rationale: `RTXPTDebugCompute.csh` keeps including `RTXPTCommon.fxh` and still reaches the same `RTXPTFrameConstants` declaration. RT shaders move to `RTXPTSceneBridge.hlsli` (which already declares `g_FrameConstants`), so they must **not** also include `RTXPTCommon.fxh` to avoid a duplicate declaration.

- [x] **Step 5: Register the new headers in CMake**

Modify `DiligentSamples/Samples/RTXPT/CMakeLists.txt` so the `SHADERS` list includes the three new files. Replace the existing `set(SHADERS ...)` block with:

```cmake
set(SHADERS
    assets/shaders/RTXPTCommon.fxh
    assets/shaders/RTXPTShaderShared.hlsli
    assets/shaders/RTXPTSceneBridge.hlsli
    assets/shaders/RTXPTMaterialBridge.hlsli
    assets/shaders/RTXPTMinimal.rgen
    assets/shaders/RTXPTMinimal.rmiss
    assets/shaders/RTXPTMinimal.rchit
    assets/shaders/RTXPTDebugCompute.csh
    assets/shaders/RTXPTBlit.vsh
    assets/shaders/RTXPTBlit.psh
)
```

- [x] **Step 6: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt Samples/RTXPT/assets/shaders
```

Expected: no output and exit code 0.

- [x] **Step 7: Commit shared HLSL infrastructure**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/assets/shaders/RTXPTCommon.fxh Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.1 shared shader bridge headers" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the four shader header files plus the CMake update.

---

### Task 2: Add SubInstanceData Buffer Inside RTXPTAccelerationStructures

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`

- [x] **Step 1: Extend the header with sub-instance data**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`:

Add this struct above `RTXPTAccelerationStructureStats`:

```cpp
struct RTXPTSubInstanceData
{
    Uint32 MaterialID = 0;
    Uint32 Flags      = 0;
    Uint32 Padding0   = 0;
    Uint32 Padding1   = 0;
};
static_assert(sizeof(RTXPTSubInstanceData) == 16, "RTXPTSubInstanceData layout must match RTXPTShaderShared.hlsli");
```

Update `RTXPTAccelerationStructureStats` so it also tracks the sub-instance table size:

```cpp
struct RTXPTAccelerationStructureStats
{
    bool        RayTracingSupported = false;
    bool        Built               = false;
    Uint32      GeometryCount       = 0;
    Uint32      InstanceCount       = 0;
    Uint32      BLASCount           = 0;
    Uint32      SubInstanceCount    = 0;
    Uint64      BLASScratchSize     = 0;
    Uint64      TLASScratchSize     = 0;
    std::string DisabledReason;
    std::string LastError;
};
```

Add a public accessor and a private member to `RTXPTAccelerationStructures`. Insert the accessor next to `GetTLAS()`:

```cpp
    IBuffer* GetSubInstanceBuffer() const { return m_SubInstanceBuffer; }
```

Add the private member next to `m_InstanceBuffer`:

```cpp
    RefCntAutoPtr<IBuffer>          m_SubInstanceBuffer;
```

- [x] **Step 2: Reset the new buffer**

Modify `RTXPTAccelerationStructures::Reset()` in `RTXPTAccelerationStructures.cpp` to release `m_SubInstanceBuffer`. The body should read:

```cpp
void RTXPTAccelerationStructures::Reset()
{
    m_BLASRecords.clear();
    m_TLAS.Release();
    m_BLASScratch.Release();
    m_TLASScratch.Release();
    m_InstanceBuffer.Release();
    m_SubInstanceBuffer.Release();
    m_Stats = {};
}
```

- [x] **Step 3: Track per-primitive material id while building BLAS records**

Inside `BuildStaticScene` in `RTXPTAccelerationStructures.cpp`, add a local `SubInstance` accumulator next to the existing `Instances` vector. Locate this line:

```cpp
    std::vector<TLASBuildInstanceData> Instances;
    std::vector<std::string>           InstanceNames;
    Instances.reserve(Scene.LinearNodes.size());
    InstanceNames.reserve(Scene.LinearNodes.size());
```

Replace it with:

```cpp
    std::vector<TLASBuildInstanceData> Instances;
    std::vector<std::string>           InstanceNames;
    std::vector<RTXPTSubInstanceData>  SubInstances;
    Instances.reserve(Scene.LinearNodes.size());
    InstanceNames.reserve(Scene.LinearNodes.size());
    SubInstances.reserve(Scene.LinearNodes.size());
```

Now record one `RTXPTSubInstanceData` per primitive included in the BLAS build. Find this existing block inside the primitive loop:

```cpp
            GeometryNames.emplace_back((pNode->Name.empty() ? "RTXPTGeometry" : pNode->Name) + "_" + std::to_string(PrimitiveIndex));
```

Immediately after that line, append the sub-instance entry while the primitive is still in scope:

```cpp
            RTXPTSubInstanceData SubEntry;
            SubEntry.MaterialID = Primitive.MaterialId;
            SubInstances.emplace_back(SubEntry);
```

This guarantees `SubInstances.size()` matches the cumulative geometry count and stays consistent with `InstanceID() + GeometryIndex()`, where `InstanceID()` is the TLAS `CustomId` sub-instance base and `GeometryIndex()` selects the geometry within the BLAS.

- [x] **Step 4: Create the sub-instance buffer after the TLAS build**

Locate the existing block in `BuildStaticScene`:

```cpp
    m_Stats.BLASCount     = static_cast<Uint32>(m_BLASRecords.size());
    m_Stats.InstanceCount = static_cast<Uint32>(Instances.size());
    m_Stats.Built         = true;
    return true;
}
```

Replace it with:

```cpp
    if (SubInstances.empty())
    {
        // Always upload at least one dummy entry so the bridge buffer can be bound unconditionally.
        SubInstances.push_back(RTXPTSubInstanceData{});
    }

    BufferDesc SubInstanceDesc;
    SubInstanceDesc.Name              = "RTXPT sub-instance buffer";
    SubInstanceDesc.Usage             = USAGE_IMMUTABLE;
    SubInstanceDesc.BindFlags         = BIND_SHADER_RESOURCE;
    SubInstanceDesc.Mode              = BUFFER_MODE_STRUCTURED;
    SubInstanceDesc.ElementByteStride = sizeof(RTXPTSubInstanceData);
    SubInstanceDesc.Size              = sizeof(RTXPTSubInstanceData) * SubInstances.size();

    BufferData SubInstanceData{SubInstances.data(), SubInstanceDesc.Size};
    pDevice->CreateBuffer(SubInstanceDesc, &SubInstanceData, &m_SubInstanceBuffer);
    if (!m_SubInstanceBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT sub-instance buffer";
        return false;
    }

    m_Stats.BLASCount        = static_cast<Uint32>(m_BLASRecords.size());
    m_Stats.InstanceCount    = static_cast<Uint32>(Instances.size());
    m_Stats.SubInstanceCount = static_cast<Uint32>(SubInstances.size());
    m_Stats.Built            = true;
    return true;
}
```

- [x] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
```

Expected: no output and exit code 0.

- [x] **Step 6: Commit sub-instance buffer**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
git -C DiligentSamples commit -m "feat(rtxpt): build phase 5.1 sub-instance material map" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the acceleration structures changes.

---

### Task 3: Guarantee Non-Null Material And Light Bridge Buffers

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`

Diligent rejects static shader resources that are declared in a PSO's resource layout but never bound. The RT pass in Task 4 declares `g_Materials` (closest hit) and `g_Lights` (miss) as static structured buffers. If a loaded scene happens to have zero materials or zero lights, the current `RTXPTMaterials::Upload` / `RTXPTLights::Upload` exit early without creating a buffer, leaving those SRVs null. This task guarantees both classes always upload at least one default entry.

- [x] **Step 1: Always upload a default material entry**

Replace the body of `RTXPTMaterials::Upload` in `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` with:

```cpp
bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const GLTF::Model& Model)
{
    Reset();

    m_Stats.MaterialCount = static_cast<Uint32>(Model.Materials.size());

    std::vector<GLTF::Material::ShaderAttribs> Materials;
    Materials.reserve(std::max<size_t>(Model.Materials.size(), 1));
    for (const GLTF::Material& Material : Model.Materials)
        Materials.emplace_back(Material.Attribs);

    if (Materials.empty())
    {
        // Always upload at least one default material so the shader-side bridge SRV is never null.
        Materials.emplace_back();
    }

    BufferDesc Desc;
    Desc.Name              = "RTXPT material buffer";
    Desc.Usage             = USAGE_IMMUTABLE;
    Desc.BindFlags         = BIND_SHADER_RESOURCE;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(GLTF::Material::ShaderAttribs);
    Desc.Size              = sizeof(GLTF::Material::ShaderAttribs) * Materials.size();

    BufferData Data{Materials.data(), Desc.Size};
    pDevice->CreateBuffer(Desc, &Data, &m_MaterialBuffer);

    if (!m_MaterialBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT material buffer";
        return false;
    }

    return true;
}
```

- [x] **Step 2: Always upload a default light entry**

Replace the body of `RTXPTLights::Upload` in `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp` with:

```cpp
bool RTXPTLights::Upload(IRenderDevice* pDevice, const GLTF::Scene& Scene, const GLTF::ModelTransforms& Transforms)
{
    Reset();

    std::vector<RTXPTLightData> Lights;
    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pLight == nullptr)
            continue;

        if (pNode->Index < 0 || static_cast<size_t>(pNode->Index) >= Transforms.NodeGlobalMatrices.size())
            continue;

        Lights.emplace_back(MakeLightData(*pNode->pLight, Transforms.NodeGlobalMatrices[pNode->Index]));
    }

    m_Stats.LightCount = static_cast<Uint32>(Lights.size());

    if (Lights.empty())
    {
        // Always upload at least one default (disabled) light so the shader-side bridge SRV is never null.
        RTXPTLightData Default;
        Default.ColorIntensity = float4{0, 0, 0, 0};
        Default.DirectionType  = float4{0, -1, 0, -1.0f}; // Type < 0 → "unused"
        Lights.emplace_back(Default);
    }

    BufferDesc Desc;
    Desc.Name              = "RTXPT light buffer";
    Desc.Usage             = USAGE_IMMUTABLE;
    Desc.BindFlags         = BIND_SHADER_RESOURCE;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(RTXPTLightData);
    Desc.Size              = sizeof(RTXPTLightData) * Lights.size();

    BufferData Data{Lights.data(), Desc.Size};
    pDevice->CreateBuffer(Desc, &Data, &m_LightBuffer);

    if (!m_LightBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT light buffer";
        return false;
    }

    return true;
}
```

The default light uses a sentinel type of `-1.0` so the miss shader's `Type < 0.5` directional branch skips it. `m_Stats.LightCount` continues to reflect the real number of scene lights (zero in this case), independent of the buffer's element count.

- [x] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTLights.cpp
```

Expected: no output and exit code 0.

- [x] **Step 4: Commit material and light fallback entries**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTLights.cpp
git -C DiligentSamples commit -m "feat(rtxpt): always upload default material and light entries" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two `.cpp` files.

---

### Task 4: Bind Material/SubInstance/Light Buffers In RTXPTRayTracingPass

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [x] **Step 1: Extend `Initialize` signature and stats**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`. Update the `RTXPTRayTracingPassStats` struct to record bridge bindings:

```cpp
struct RTXPTRayTracingPassStats
{
    bool        Ready                = false;
    bool        LastTraceExecuted    = false;
    bool        MaterialBridgeBound  = false;
    bool        SubInstanceBound     = false;
    bool        LightBridgeBound     = false;
    Uint32      TraceCount           = 0;
    std::string DisabledReason;
    std::string LastError;
};
```

Replace the `Initialize` declaration with:

```cpp
    bool Initialize(IRenderDevice*  pDevice,
                    IDeviceContext* pContext,
                    IEngineFactory* pEngineFactory,
                    IBuffer*        pFrameConstants,
                    IBuffer*        pMaterialBuffer,
                    IBuffer*        pSubInstanceBuffer,
                    IBuffer*        pLightBuffer,
                    ITopLevelAS*    pTLAS,
                    bool            RayTracingSupported,
                    bool            StandaloneRTShadersSupported);
```

- [x] **Step 2: Update Initialize body to bind the bridge SRVs**

Replace the body of `RTXPTRayTracingPass::Initialize` in `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` with:

```cpp
bool RTXPTRayTracingPass::Initialize(IRenderDevice*  pDevice,
                                     IDeviceContext* pContext,
                                     IEngineFactory* pEngineFactory,
                                     IBuffer*        pFrameConstants,
                                     IBuffer*        pMaterialBuffer,
                                     IBuffer*        pSubInstanceBuffer,
                                     IBuffer*        pLightBuffer,
                                     ITopLevelAS*    pTLAS,
                                     bool            RayTracingSupported,
                                     bool            StandaloneRTShadersSupported)
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

    RefCntAutoPtr<IShader> pRayGen;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_GEN;
    ShaderCI.Desc.Name       = "RTXPT minimal raygen";
    ShaderCI.FilePath        = "RTXPTMinimal.rgen";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pRayGen);

    RefCntAutoPtr<IShader> pMiss;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_MISS;
    ShaderCI.Desc.Name       = "RTXPT minimal miss";
    ShaderCI.FilePath        = "RTXPTMinimal.rmiss";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pMiss);

    RefCntAutoPtr<IShader> pClosestHit;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_CLOSEST_HIT;
    ShaderCI.Desc.Name       = "RTXPT minimal closest hit";
    ShaderCI.FilePath        = "RTXPTMinimal.rchit";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pClosestHit);

    if (!pRayGen || !pMiss || !pClosestHit)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal ray tracing shaders";
        return false;
    }

    RayTracingPipelineStateCreateInfoX PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT minimal RT PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_RAY_TRACING;
    PSOCreateInfo.AddGeneralShader("Main", pRayGen);
    PSOCreateInfo.AddGeneralShader("PrimaryMiss", pMiss);
    PSOCreateInfo.AddTriangleHitShader("PrimaryHit", pClosestHit);
    PSOCreateInfo.RayTracingPipeline.MaxRecursionDepth = 1;
    PSOCreateInfo.RayTracingPipeline.ShaderRecordSize  = 0;
    PSOCreateInfo.MaxAttributeSize = static_cast<Uint32>(sizeof(float) * 2);
    PSOCreateInfo.MaxPayloadSize   = static_cast<Uint32>(sizeof(float) * 4);

    // Bind each bridge resource only to the stage that actually references it. DXC strips unused
    // declarations during compilation, so declaring them in stages that never call the bridge
    // helpers would leave dangling layout entries that GetStaticVariableByName cannot resolve.
    //
    // Stage map for Phase 5.1:
    //   g_FrameConstants  -> raygen    (camera state for primary rays)
    //   g_TLAS            -> raygen (the only stage that issues TraceRay)
    //   g_Materials       -> closest hit (Bridge::GetMaterial)
    //   g_SubInstanceData -> closest hit (Bridge::GetSubInstanceData)
    //   g_Lights          -> miss      (Bridge::GetLight for the sun tint helper)
    //   g_OutputColor     -> raygen    (write target)
    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_FrameConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_TLAS", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_Materials", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_SubInstanceData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_MISS, "g_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateRayTracingPipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal RT PSO";
        return false;
    }

    auto SetStatic = [&](SHADER_TYPE Stage, const char* Name, IDeviceObject* pObject)
    {
        if (pObject == nullptr)
            return false;

        IShaderResourceVariable* pVar = m_PSO->GetStaticVariableByName(Stage, Name);
        if (pVar == nullptr)
            return false;

        pVar->Set(pObject);
        return true;
    };

    const bool FrameConstantsBound = SetStatic(SHADER_TYPE_RAY_GEN, "g_FrameConstants", pFrameConstants);
    const bool TLASBound = SetStatic(SHADER_TYPE_RAY_GEN, "g_TLAS", m_TLAS);

    if (!FrameConstantsBound || !TLASBound)
    {
        m_Stats.LastError = "Failed to bind required RTXPT frame constants or TLAS";
        return false;
    }

    IDeviceObject* pMaterialsView   = nullptr;
    IDeviceObject* pSubInstanceView = nullptr;
    IDeviceObject* pLightsView      = nullptr;

    if (pMaterialBuffer != nullptr)
        pMaterialsView = pMaterialBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE);
    if (pSubInstanceBuffer != nullptr)
        pSubInstanceView = pSubInstanceBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE);
    if (pLightBuffer != nullptr)
        pLightsView = pLightBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE);

    m_Stats.MaterialBridgeBound = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_Materials", pMaterialsView);
    m_Stats.SubInstanceBound    = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "g_SubInstanceData", pSubInstanceView);
    m_Stats.LightBridgeBound    = SetStatic(SHADER_TYPE_RAY_MISS, "g_Lights", pLightsView);

    if (!m_Stats.MaterialBridgeBound || !m_Stats.SubInstanceBound || !m_Stats.LightBridgeBound)
    {
        m_Stats.LastError = "Failed to bind required RTXPT bridge buffers";
        return false;
    }

    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal RT SRB";
        return false;
    }

    ShaderBindingTableDesc SBTDesc;
    SBTDesc.Name = "RTXPT minimal SBT";
    SBTDesc.pPSO = m_PSO;
    pDevice->CreateSBT(SBTDesc, &m_SBT);
    if (!m_SBT)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal SBT";
        return false;
    }

    m_SBT->BindRayGenShader("Main");
    m_SBT->BindMissShader("PrimaryMiss", 0);
    m_SBT->BindHitGroupForTLAS(m_TLAS, 0, "PrimaryHit");
    pContext->UpdateSBT(m_SBT);

    // TODO(RTXPT-Port Phase 4): Restore stable-plane pre-pass and fill-stable-planes dispatch; current path traces one minimal primary-ray pass.
    // TODO(RTXPT-Port Phase 5.2): Replace flat-shaded closest hit with the reference path tracer core (BxDF + multi-bounce + accumulation).
    m_Stats.Ready = true;
    return true;
}
```

- [x] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: no output and exit code 0.

- [x] **Step 4: Commit ray tracing pass bridge wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): bind phase 5.1 material/sub-instance/light bridge srvs" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the ray tracing pass changes.

---

### Task 5: Port The Minimal RT Shaders Onto The Bridge

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`

- [x] **Step 1: Switch the raygen include and ensure it uses the bridge declarations**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen` with:

```hlsl
#include "RTXPTSceneBridge.hlsli"

RaytracingAccelerationStructure g_TLAS;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4> g_OutputColor;

[shader("raygeneration")]
void main()
{
    const uint2 Pixel      = DispatchRaysIndex().xy;
    const uint2 Dimensions = DispatchRaysDimensions().xy;
    const float2 UV        = (float2(Pixel) + 0.5) / float2(Dimensions);
    const float2 NDC       = UV * 2.0 - 1.0;

    const float4 WorldPos = mul(float4(NDC, 1.0, 1.0), g_FrameConstants.ViewProjInv);
    const float3 Origin   = g_FrameConstants.CameraPosition_Time.xyz;
    const float3 RayDir   = normalize(WorldPos.xyz / WorldPos.w - Origin);

    RayDesc Ray;
    Ray.Origin    = Origin;
    Ray.Direction = RayDir;
    Ray.TMin      = 0.001;
    Ray.TMax      = 10000.0;

    RTXPTPrimaryPayload Payload;
    Payload.ColorDepth = float4(0.02, 0.03, 0.04, 1.0);

    TraceRay(g_TLAS,
             RAY_FLAG_FORCE_OPAQUE,
             0xFF,
             0,
             1,
             0,
             Ray,
             Payload);

    g_OutputColor[Pixel] = float4(saturate(Payload.ColorDepth.rgb), 1.0);
}

// TODO(RTXPT-Port Phase 5.2): Replace the single-bounce primary ray with the reference path tracer entry point (multi-bounce, accumulation).
```

- [x] **Step 2: Rewrite the miss shader to exercise the light bridge**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss` with:

```hlsl
#include "RTXPTSceneBridge.hlsli"

[shader("miss")]
void main(inout RTXPTPrimaryPayload Payload)
{
    const float3 RayDir  = WorldRayDirection();
    const float  T       = saturate(RayDir.y * 0.5 + 0.5);
    const float3 Horizon = float3(0.48, 0.58, 0.68);
    const float3 Zenith  = float3(0.05, 0.08, 0.14);
    float3       Sky     = lerp(Horizon, Zenith, T);

    // Light bridge sanity exercise: tint the sky toward the first directional light, if any.
    // This both validates the binding and serves as a placeholder until Phase 5.5 lands a real
    // environment / sun sampler.
    if (Bridge::GetLightCount() > 0)
    {
        const RTXPTLightData L    = Bridge::GetLight(0);
        const float          Type = L.DirectionType.w;
        // Type encoding matches LightTypeToShaderValue in RTXPTLights.cpp: 0=Directional, 1=Point, 2=Spot.
        if (Type < 0.5)
        {
            const float SunDot = saturate(dot(RayDir, -L.DirectionType.xyz));
            const float Disk   = pow(SunDot, 32.0);
            Sky += L.ColorIntensity.rgb * L.ColorIntensity.a * Disk * 0.05;
        }
    }

    Payload.ColorDepth = float4(Sky, 1.0);
}

// TODO(RTXPT-Port Phase 5.5): Replace the placeholder sun disk with environment map / NEE-driven sun sampling once the lighting baker is restored.
```

- [x] **Step 3: Rewrite the closest hit to use the bridge**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit` with:

```hlsl
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTMaterialBridge.hlsli"

static float3 ComputeBarycentricFallback(in BuiltInTriangleIntersectionAttributes Attributes)
{
    const float3 Barycentrics = float3(1.0 - Attributes.barycentrics.x - Attributes.barycentrics.y,
                                       Attributes.barycentrics.x,
                                       Attributes.barycentrics.y);
    const float InstanceTint = frac(float(InstanceID() * 17 + PrimitiveIndex() * 3) * 0.037);
    const float Depth        = saturate(RayTCurrent() / 150.0);
    return lerp(Barycentrics, float3(Depth, 1.0 - Depth, InstanceTint), 0.35);
}

[shader("closesthit")]
void main(inout RTXPTPrimaryPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
    const float Depth = saturate(RayTCurrent() / 150.0);
    float3 Color      = ComputeBarycentricFallback(Attributes);

    if (Bridge::HasSubInstanceTable() && Bridge::HasMaterialTable())
    {
        const RTXPTSubInstanceData SubInstance = Bridge::GetSubInstanceData();
        const RTXPTMaterialAttribs Material    = Bridge::GetMaterial(SubInstance.MaterialID);
        const float NdotV                      = saturate(dot(-WorldRayDirection(), float3(0.0, 1.0, 0.0)) * 0.5 + 0.5);
        Color                                  = Material.BaseColorFactor.rgb * (0.4 + 0.6 * NdotV);
    }

    Payload.ColorDepth = float4(Color, Depth);
}

// TODO(RTXPT-Port Phase 5.2): Replace flat base color with the reference path tracer shading (BSDF + light sampling + NEE).
// TODO(RTXPT-Port Phase 5.3): Honor RTXPTMaterialAttribs.AlphaMode/AlphaCutoff via any-hit specialization instead of forcing opaque rays.
```

- [x] **Step 4: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders
```

Expected: no output and exit code 0.

- [x] **Step 5: Commit shader port**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit
git -C DiligentSamples commit -m "feat(rtxpt): port phase 5.1 minimal rt shaders onto bridge" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the three updated shader files.

---

### Task 6: Wire The New Bridge Buffers Into RTXPTSample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [x] **Step 1: Pass the bridge buffers when initializing the ray tracing pass**

Replace the body of `RTXPTSample::CreatePhase4Passes` in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` with:

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

- [x] **Step 2: Surface bridge status in the ImGui panel**

Inside `RTXPTSample::UpdateUI()`, locate this line:

```cpp
    ImGui::Text("Acceleration structures: %s", m_AccelerationStructures.IsBuilt() ? "built" : "not built");
```

After the existing `ImGui::Text("RT geometries: %u", ASStats.GeometryCount);` line, insert:

```cpp
    ImGui::Text("Sub-instances: %u", ASStats.SubInstanceCount);
```

Then locate the block that prints `TraceRays pass: ...` and replace it with:

```cpp
    ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
    ImGui::Text("Material bridge: %s", RTPassStats.MaterialBridgeBound ? "bound" : "fallback");
    ImGui::Text("Sub-instance bridge: %s", RTPassStats.SubInstanceBound ? "bound" : "fallback");
    ImGui::Text("Light bridge: %s", RTPassStats.LightBridgeBound ? "bound" : "fallback");
    ImGui::Text("TraceRays executed: %s", RTPassStats.LastTraceExecuted ? "yes" : "no");
    ImGui::Text("TraceRays count: %u", RTPassStats.TraceCount);
```

Finally, replace the line:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 4): expose stable-plane, RTXDI, light feedback, and denoising-guide pass toggles after their shaders are ported.");
```

with:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 4): expose stable-plane, RTXDI, light feedback, and denoising-guide pass toggles after their shaders are ported.");
    ImGui::Text("TODO(RTXPT-Port Phase 5.2): swap closest-hit flat shading for the reference path tracer once shader layer 4 lands.");
```

- [x] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: no output and exit code 0.

- [x] **Step 4: Commit sample wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 5.1 bridge buffers into sample" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the sample changes.

---

### Task 7: Phase 5.1 Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [x] **Step 1: Confirm structured Phase 5 TODOs**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5" DiligentSamples/Samples/RTXPT
```

Expected: TODOs appear in `RTXPTShaderShared.hlsli` is fine without one, but the following must be present:

```text
RTXPTSceneBridge.hlsli  : Phase 5.2 scene accessors, Phase 5.3 alpha flags
RTXPTMaterialBridge.hlsli: Phase 5.2 BSDF sampler, Phase 5.3 textures
RTXPTMinimal.rgen        : Phase 5.2 reference path tracer entry point
RTXPTMinimal.rmiss       : Phase 5.5 environment / sun sampling
RTXPTMinimal.rchit       : Phase 5.2 BSDF + NEE, Phase 5.3 alpha-test any-hit
RTXPTRayTracingPass.cpp  : Phase 5.2 reference path tracer body
RTXPTSample.cpp          : Phase 5.2 path tracer swap-in
```

- [x] **Step 2: Confirm file registration**

Run:

```powershell
rg -n "RTXPTShaderShared|RTXPTSceneBridge|RTXPTMaterialBridge" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: each of the three new HLSL headers is listed in `CMakeLists.txt`.

- [x] **Step 3: Confirm DiligentSamples log shows the Phase 5.1 commits**

Run:

```powershell
git -C DiligentSamples log --oneline -n 8
```

Expected (most recent first):

```text
feat(rtxpt): wire phase 5.1 bridge buffers into sample
feat(rtxpt): port phase 5.1 minimal rt shaders onto bridge
feat(rtxpt): bind phase 5.1 material/sub-instance/light bridge srvs
feat(rtxpt): always upload default material and light entries
feat(rtxpt): build phase 5.1 sub-instance material map
feat(rtxpt): add phase 5.1 shared shader bridge headers
... (prior Phase 4 commits)
```

- [ ] **Step 4: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: exit code 0. If the build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 5: Optional D3D12 runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with D3D12 on a standalone-RT-capable machine. Expected ImGui facts:

```text
Acceleration structures: built
Sub-instances: equals total primitive count of the loaded scene
TraceRays pass: ready
Material bridge: bound
Sub-instance bridge: bound
Light bridge: bound
TraceRays executed: yes
TraceRays count: increases every frame
```

Expected visual result: hits show flat per-material base color modulated by a soft NdotV term instead of pure barycentric debug coloring. Sky still appears on misses.

- [ ] **Step 6: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT-capable machine. Expected facts match the D3D12 run.

If standalone ray tracing shaders are unavailable on the Vulkan device, expected fallback facts are:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
Material/Sub-instance/Light bridge: any state is acceptable when TraceRays is disabled
The sample still launches and clears the swapchain.
```

- [x] **Step 7: Commit top-level submodule pointer and plan**

After all `DiligentSamples` Phase 5.1 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-28-rtxpt-phase5-1-shader-bridge.md
git commit -m "feat(samples): plan and add RTXPT phase 5.1 shader bridge" -m "Co-Authored-By: GPT 5.5"
```

Expected: one top-level commit that records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] The plan implements only Phase 5 layers 2 and 3 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`. Layers 4–9 are explicitly deferred to subsequent plans.
- [x] Each task ends with a runnable sample: shader-side bridge table checks keep the barycentric/depth debug fallback, while missing required C++ bridge bindings disable the RT pass instead of tracing with unbound SRVs.
- [x] Shared HLSL declarations (`RTXPTShaderShared.hlsli`) are the single source of truth for structures mirrored between C++ and HLSL (`RTXPTSubInstanceData`, `RTXPTMaterialAttribs`, `RTXPTLightData`, `RTXPTFrameConstants`, `RTXPTPrimaryPayload`).
- [x] `RTXPTAccelerationStructures` owns the sub-instance contract (`InstanceID() + GeometryIndex() → MaterialID`, with `InstanceID()` sourced from TLAS `CustomId`), matching Diligent's shader-visible custom-id semantics.
- [x] `RTXPTRayTracingPass` keeps the no-ray-tracing and no-standalone-RT fallbacks intact, validates required static bridge bindings, and exposes per-bridge bound/not-bound status in stats.
- [x] Every new shader file is registered in `DiligentSamples/Samples/RTXPT/CMakeLists.txt`.
- [x] No textures, vertex normals/UVs, or alpha-mask handling are introduced — they remain `TODO(RTXPT-Port Phase 5.2/5.3)`.
- [x] Verification steps avoid build/runtime execution unless the user explicitly requests it.
- [x] Each task ends with a focused, single-purpose commit using the project's `Co-Authored-By: GPT 5.5` trailer convention from prior phase plans.
