# RTXPT Phase 5.2 Reference Path Tracer Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land Phase 5 shader layer 4 from the RTXPT port design — the reference path tracer core — by replacing the Phase 5.1 single-bounce flat-shaded closest hit with a multi-bounce Lambertian path tracer that reads per-vertex normals + material base color, accumulates radiance across frames into an RGBA32F buffer, and resets cleanly on resize/scene reload.

**Architecture:** Make the GLTF model's primary vertex buffer (Position + Normal + TexCoord0 in buffer 0) and index buffer SRV-accessible by adding `BIND_SHADER_RESOURCE` at load time. Extend `RTXPTSubInstanceData` from 16 to 32 bytes to carry per-(instance, geometry) vertex/index offsets needed for triangle vertex fetch. Add `RTXPTPathTracerSettings` (max bounces, accumulation frame, reset flag) packed into a new field of `RTXPTFrameConstants`. Add an RGBA32F accumulation render target in `RTXPTRenderTargets` that the raygen samples-averages into; the existing blit path reads from it instead of the rgba8 OutputColor when accumulation is active. Add two new HLSL headers: `RTXPTRandom.hlsli` (xorshift/Hash32 PRNG + cosine-weighted hemisphere) and a vertex-fetch extension to `RTXPTSceneBridge.hlsli`. Replace `RTXPTMinimal.{rgen,rchit,rmiss}` with `RTXPTReference.{rgen,rchit,rmiss}`: the raygen runs an N-bounce loop with per-bounce PRNG and updates accumulation; the chit fills a richer payload (world position, world normal, base color, hit distance, geometry-hit flag); the miss writes a procedural sky (carried forward from Phase 5.1) into the payload's emission slot.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs, DiligentTools `GLTFLoader` (`GLTF::Material::ShaderAttribs`, `GLTF::Model::GetVertexBuffer`, `GLTF::Model::GetIndexBuffer`), HLSL 6.5 ray tracing shaders compiled by DXC, Diligent buffer view (`BUFFER_MODE_STRUCTURED` for vertex buffer, `BUFFER_MODE_FORMATTED` with `VT_UINT32` for index buffer), Diligent shader resource binding APIs, Dear ImGui.

---

## Scope Note: Phase 5 Sub-Plan Series

Phase 5 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md` enumerates nine shader dependency layers. The previously completed plan `docs/superpowers/plans/2026-05-28-rtxpt-phase5-1-shader-bridge.md` covered layers 2 and 3 (shared declarations and scene/material bridge). This plan covers **only** layer 4 (reference path tracer core). The remaining Phase 5 sub-phases each need their own plan in a later session:

- Phase 5.3: Material specialization, alpha test, any-hit (layer 5).
- Phase 5.4: Stable planes and realtime mode (layer 6).
- Phase 5.5: RTXDI shader bridge and passes (layer 7).
- Phase 5.6: NRD, denoising guides, post-process (layer 8).
- Phase 5.7: NVAPI, SER, OMM, DLSS-related shader variants (layer 9).

## Baseline

Current Phase 5.1 state in `DiligentSamples/Samples/RTXPT`:

- `RTXPTScene::LoadDefaultScene` loads `bistro-programmer-art.scene.json` via the Diligent GLTF loader, with `VertBufferBindFlags[i] = BIND_VERTEX_BUFFER | BIND_RAY_TRACING` and `IndBufferBindFlags = BIND_INDEX_BUFFER | BIND_RAY_TRACING`. The index type defaults to `VT_UINT32`.
- `RTXPTAccelerationStructures::BuildStaticScene` builds per-mesh-node BLAS and a single TLAS with `HIT_GROUP_BINDING_MODE_PER_GEOMETRY` and `HitGroupStride = 1`. Per-instance `CustomId` holds the sub-instance base for that instance, exposed in chit as `InstanceID()`. It uploads a `RTXPTSubInstanceData` structured buffer (16 bytes per entry: `MaterialID`, `Flags`, two padding uints), one entry per (instance, geometry).
- `RTXPTRayTracingPass` binds `g_FrameConstants`/`g_TLAS`/`g_OutputColor` to raygen, `g_Materials`/`g_SubInstanceData` to closest hit, and `g_Lights` to miss. Output is `RWTexture2D<float4>` (`TEX_FORMAT_RGBA8_UNORM`).
- `RTXPTMinimal.rgen` traces a single primary ray. `RTXPTMinimal.rchit` reads `SubInstance.MaterialID` and writes `Material.BaseColorFactor.rgb * (0.4 + 0.6 * NdotV)` with a fixed world-space up vector; falls back to barycentric debug if bridge tables are empty. `RTXPTMinimal.rmiss` writes a sky gradient tinted by the first directional light.
- `RTXPTRenderTargets` owns `OutputColor` (RGBA8) and `ComputeColor` (RGBA8). `RTXPTBlitPass` reads either as SRV and renders to the swapchain.
- `RTXPTSample` has a fixed camera at `{0, 1.5, -6}` and a `m_FrameIndex` counter incremented in `UpdateFrameConstants`.

This plan assumes the Phase 5.1 submodule changes are already committed and the top-level repository starts clean.

---

## Scope

This plan implements the Phase 5.2 runnable milestone:

- Make the GLTF default vertex buffer (Position/Normal/TexCoord0 in buffer 0) and the index buffer shader-resource bindable from C++.
- Extend `RTXPTSubInstanceData` to 32 bytes with `FirstVertex`, `FirstIndex`, `IndexCount`, `VertexCount`, plus an `IsIndexed` flag in `Flags`. Pack the same struct on the HLSL side.
- Extend `RTXPTFrameConstants` with a new `PathTracerSettings` field (`MaxBounces`, `AccumulationFrame`, `ResetAccumulation`, `MinBounces`). C++ owns those values and resets accumulation on resize and scene reload.
- Add an RGBA32F accumulation render target (`AccumColor`) inside `RTXPTRenderTargets`, plus capability-checked creation; if RGBA32F UAV is unsupported the sample falls back to the Phase 5.1 RGBA8 path with a logged reason.
- Add `RTXPTRandom.hlsli` with `Hash32`, `Hash32Combine`, `ToFloat0To1`, `SampleCosineHemisphere`, and `BuildOrthonormalBasis`.
- Extend `RTXPTSceneBridge.hlsli` with `RTXPTVertex`, `g_VertexBuffer` (StructuredBuffer), `g_IndexBuffer` (typed `Buffer<uint>`), `Bridge::GetTriangleIndices`, `Bridge::GetTriangleVertices`, `Bridge::InterpolateNormal`, and `Bridge::ComputeGeometricNormal`.
- Add `RTXPTPathTracerPayload` (richer than the 16-byte `RTXPTPrimaryPayload`): world position, world normal, base color, hit distance, hit/miss flag, emission accumulator for the miss shader.
- Add `RTXPTReference.rgen` (N-bounce loop + accumulation), `RTXPTReference.rchit` (fills the new payload), `RTXPTReference.rmiss` (writes sky into the payload's emission slot).
- Update `RTXPTRayTracingPass` to bind the vertex/index buffers and accumulation UAV, create the index buffer SRV view with `VT_UINT32` format, switch shader files to the `RTXPTReference.*` set, and bump `MaxPayloadSize` to fit the new payload.
- Update `RTXPTSample` to pass the new buffers, reset accumulation on resize/scene reload, expose `MaxBounces`/`ResetAccumulation` controls in ImGui, and route the blit input to the accumulation buffer when active.
- Delete the now-unused `RTXPTMinimal.{rgen,rchit,rmiss}` files (the bridge headers stay, since the reference shaders include them).
- Update the structured `TODO(RTXPT-Port Phase 5)` markers: remove the Phase 5.2 ones that this plan resolves, leave/append the Phase 5.3+ markers.

This plan intentionally does not:

- Sample any material textures, tangent maps, or attributes beyond Position/Normal/TexCoord0.
- Implement RTXPT's `PathState`, `StablePlanes`, `RayCone`, `InteriorList`, nested dielectrics, or `LightsBaker`.
- Implement next-event estimation (NEE) toward analytic or environment lights — the only contribution to radiance is bounce throughput times sky/light radiance from the miss shader. NEE belongs to Phase 5.5.
- Implement alpha test, any-hit, or material-specialized hit groups (Phase 5.3).
- Implement Russian roulette termination (deferred — the plan uses a fixed bounce cap).
- Run automated builds or runtime execution; build/runtime steps are listed for explicit user request only.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Register the new shader files; drop `RTXPTMinimal.*` entries.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
  - Add `BIND_SHADER_RESOURCE` to vertex buffer 0 and to the index buffer; expose buffer 0 stride.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
  - Add `GetVertexBuffer0()`/`GetIndexBuffer()`/`GetVertexStride0()` accessors that delegate to the loaded `GLTF::Model`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
  - Grow `RTXPTSubInstanceData` to 32 bytes; add `Flags` bit constant for `kRTXPTSubInstanceFlag_Indexed`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
  - Populate the new sub-instance fields from `GLTF::Primitive` during the BLAS build.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
  - Add the accumulation target, plus `IsAccumulationActive()` / `GetAccumColorUAV()` / `GetAccumColorSRV()` accessors.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
  - Create the RGBA32F accumulation texture; check format support; surface an error if unsupported.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Add `RTXPTPathTracerSettings`; extend `RTXPTFrameConstants` with `PathTracerSettings`; add `m_MaxBounces`, `m_AccumulationFrame`, `m_ResetAccumulation` fields.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Wire new buffers into the RT pass init; reset accumulation on resize/scene reload; expose UI controls; route blit input.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
  - Extend `Initialize` signature with the vertex buffer, index buffer, accumulation UAV, accumulation-active flag, and capability inputs.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
  - Create the typed index buffer SRV view; bind the new resources; switch shader file names; raise `MaxPayloadSize`/`MaxAttributeSize`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`
  - Mirror the new `RTXPTSubInstanceData` layout; add `RTXPTPathTracerSettings` and `RTXPTPathTracerPayload`; add `RTXPTVertex`; bump `RTXPTFrameConstants` with the new field.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`
  - Add `g_VertexBuffer`/`g_IndexBuffer` declarations and triangle-vertex/normal helpers.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli`
  - PRNG and cosine-weighted hemisphere sampling helpers.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`
  - N-bounce loop with accumulation.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`
  - Closest hit that fills `RTXPTPathTracerPayload`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss`
  - Sky miss that fills the payload's emission slot.
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`

---

### Task 0: Phase 5.1 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [ ] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected:

```text
## RTXPT...origin/RTXPT
```

If unrelated files appear, leave them unstaged and continue only if they do not overlap `docs/superpowers/plans` or `DiligentSamples/Samples/RTXPT`.

- [ ] **Step 2: Confirm DiligentSamples Phase 5.1 state**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentSamples log --oneline -n 6
```

Expected: the `DiligentSamples` branch is clean and the most recent commits include the six Phase 5.1 commits ending with `feat(rtxpt): wire phase 5.1 bridge buffers into sample`.

- [ ] **Step 3: Confirm the Phase 5.1 bridge files exist and the reference files do not**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMaterialBridge.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli
```

Expected:

```text
True
True
True
True
False
False
```

If `RTXPTReference.rgen` or `RTXPTRandom.hlsli` already exists, inspect each before overwriting and preserve unrelated user work.

- [ ] **Step 4: Confirm remaining Phase 5.2 markers**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5\.2\)" DiligentSamples/Samples/RTXPT
```

Expected: matches in `RTXPTSceneBridge.hlsli`, `RTXPTMaterialBridge.hlsli`, `RTXPTMinimal.rgen`, `RTXPTMinimal.rchit`, `RTXPTRayTracingPass.cpp`, and `RTXPTSample.cpp`. This plan removes or narrows each of these markers as the corresponding code lands.

---

### Task 1: Make Vertex And Index Buffers Shader-Readable

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

Context: the Diligent GLTF loader (`DiligentTools/AssetLoader/src/GLTFBuilder.cpp:458-473`) automatically promotes a vertex buffer to `BUFFER_MODE_STRUCTURED` when `BIND_SHADER_RESOURCE` is present in its bind flags, using `ElementByteStride = VertexStride`. The index buffer path (`GLTFBuilder.cpp:366-372`) promotes to `BUFFER_MODE_FORMATTED` with `ElementByteStride = IndexSize`. By adding `BIND_SHADER_RESOURCE` to buffer 0 and the index buffer at load time we get the shader-resource layout for free; no extra copies are needed.

- [ ] **Step 1: Add scene accessors**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`. Replace the public accessor block so it also exposes the vertex/index buffers and the buffer-0 stride that path-tracer shaders need.

Locate this block:

```cpp
    const GLTF::Model*           GetModel() const { return m_Model.get(); }
    const GLTF::ModelTransforms& GetTransforms() const { return m_Transforms; }
    Uint32                       GetSceneIndex() const { return m_SceneIndex; }
    VALUE_TYPE                   GetIndexType() const { return m_IndexType; }
    Uint32                       GetMeshNodeCount() const { return m_MeshNodeCount; }
    Uint32                       GetPrimitiveCount() const { return m_PrimitiveCount; }
    Uint32                       GetMaterialCount() const { return m_MaterialCount; }
    Uint32                       GetLightCount() const { return m_LightCount; }
```

Replace it with:

```cpp
    const GLTF::Model*           GetModel() const { return m_Model.get(); }
    const GLTF::ModelTransforms& GetTransforms() const { return m_Transforms; }
    Uint32                       GetSceneIndex() const { return m_SceneIndex; }
    VALUE_TYPE                   GetIndexType() const { return m_IndexType; }
    Uint32                       GetMeshNodeCount() const { return m_MeshNodeCount; }
    Uint32                       GetPrimitiveCount() const { return m_PrimitiveCount; }
    Uint32                       GetMaterialCount() const { return m_MaterialCount; }
    Uint32                       GetLightCount() const { return m_LightCount; }

    // Buffer 0 packs POSITION + NORMAL + TEXCOORD_0 (the Diligent GLTF default layout).
    // VertexStride0 is the per-vertex stride for buffer 0 and must equal sizeof(RTXPTVertex) on the shader side.
    IBuffer* GetVertexBuffer0(IRenderDevice* pDevice = nullptr, IDeviceContext* pContext = nullptr) const;
    IBuffer* GetIndexBuffer(IRenderDevice* pDevice = nullptr, IDeviceContext* pContext = nullptr) const;
    Uint32   GetVertexStride0() const { return m_VertexStride0; }
```

Add the `m_VertexStride0` member next to `m_LightCount`:

```cpp
    Uint32                       m_VertexStride0  = 0;
```

- [ ] **Step 2: Add `BIND_SHADER_RESOURCE` to vertex buffer 0 and index buffer**

Modify `RTXPTScene::LoadDefaultScene` in `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`. Locate this block:

```cpp
    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = m_ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;
    ModelCI.IndexType            = m_IndexType;
    ModelCI.IndBufferBindFlags   = BIND_INDEX_BUFFER | BIND_RAY_TRACING;
    for (BIND_FLAGS& BindFlags : ModelCI.VertBufferBindFlags)
        BindFlags = BIND_VERTEX_BUFFER | BIND_RAY_TRACING;
```

Replace it with:

```cpp
    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = m_ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;
    ModelCI.IndexType            = m_IndexType;
    ModelCI.IndBufferBindFlags   = BIND_INDEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;
    for (BIND_FLAGS& BindFlags : ModelCI.VertBufferBindFlags)
        BindFlags = BIND_VERTEX_BUFFER | BIND_RAY_TRACING;
    // Buffer 0 is the path-tracer vertex stream (POSITION + NORMAL + TEXCOORD_0); chit reads it as a StructuredBuffer<RTXPTVertex>.
    ModelCI.VertBufferBindFlags[0] = BIND_VERTEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;
```

- [ ] **Step 3: Capture buffer-0 stride after model load**

In `RTXPTScene::CacheSceneData` in `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`, append the buffer-0 stride computation immediately before the closing brace:

```cpp
    m_VertexStride0 = 0;
    if (m_Model && m_Model->GetVertexBufferCount() > 0)
    {
        for (Uint32 i = 0; i < m_Model->GetNumVertexAttributes(); ++i)
        {
            const GLTF::VertexAttributeDesc& Desc = m_Model->GetVertexAttribute(i);
            if (Desc.BufferId == 0)
            {
                const Uint32 RelOffset = Desc.RelativeOffset == ~0u ? 0u : Desc.RelativeOffset;
                m_VertexStride0        = std::max(m_VertexStride0, RelOffset + GetValueSize(Desc.ValueType) * Desc.NumComponents);
            }
        }
    }
```

Also add `#include <algorithm>` and `#include "GraphicsAccessories.hpp"` to the top of `RTXPTScene.cpp` if they are not already present (`std::max` and `GetValueSize` are required).

Also extend `ResetLoadedData` to clear the new field. Locate:

```cpp
    m_MaterialCount  = 0;
    m_LightCount     = 0;
```

Replace it with:

```cpp
    m_MaterialCount  = 0;
    m_LightCount     = 0;
    m_VertexStride0  = 0;
```

- [ ] **Step 4: Implement the buffer accessors**

Append to the bottom of the `Diligent` namespace in `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp` (just before the final `} // namespace Diligent`):

```cpp
IBuffer* RTXPTScene::GetVertexBuffer0(IRenderDevice* pDevice, IDeviceContext* pContext) const
{
    return m_Model ? m_Model->GetVertexBuffer(0, pDevice, pContext) : nullptr;
}

IBuffer* RTXPTScene::GetIndexBuffer(IRenderDevice* pDevice, IDeviceContext* pContext) const
{
    return m_Model ? m_Model->GetIndexBuffer(pDevice, pContext) : nullptr;
}
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit shader-readable scene buffers**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose phase 5.2 vertex and index buffers to shaders" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the scene changes.

---

### Task 2: Grow SubInstanceData With Per-Geometry Layout

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`

The Phase 5.1 sub-instance record stores `MaterialID` + `Flags` + padding (16 bytes). The reference path tracer needs to fetch the three triangle indices for the current hit, then the three vertex records. To do that it needs per-(instance, geometry) base offsets into the global vertex/index streams. We extend `RTXPTSubInstanceData` to 32 bytes and pack the additional fields:

- `FirstIndex` — absolute index into `g_IndexBuffer` for the geometry's first triangle (counted in `uint`s, not bytes).
- `IndexCount` — number of triangle indices the geometry uses; `0` denotes a non-indexed primitive.
- `FirstVertex` — absolute index into `g_VertexBuffer` for the geometry's first vertex.
- `VertexCount` — number of vertices the geometry uses.

The `Flags` field receives a new bit `kRTXPTSubInstanceFlag_Indexed` so the shader doesn't have to special-case `IndexCount == 0`.

- [ ] **Step 1: Update the C++ struct and add the flag constant**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`. Replace the existing `RTXPTSubInstanceData` definition (the 16-byte struct and its `static_assert`) with:

```cpp
struct RTXPTSubInstanceData
{
    Uint32 MaterialID  = 0;
    Uint32 Flags       = 0;
    Uint32 FirstIndex  = 0;
    Uint32 IndexCount  = 0;
    Uint32 FirstVertex = 0;
    Uint32 VertexCount = 0;
    Uint32 Padding0    = 0;
    Uint32 Padding1    = 0;
};
static_assert(sizeof(RTXPTSubInstanceData) == 32, "RTXPTSubInstanceData layout must match RTXPTShaderShared.hlsli");

// Flag bits for RTXPTSubInstanceData::Flags.
constexpr Uint32 kRTXPTSubInstanceFlag_Indexed = 0x1u;
```

- [ ] **Step 2: Populate the new fields during BLAS build**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`. Locate the existing `SubInstances.emplace_back` block inside the primitive loop:

```cpp
            RTXPTSubInstanceData SubEntry;
            SubEntry.MaterialID = Primitive.MaterialId;
            SubInstances.emplace_back(SubEntry);
```

Replace it with:

```cpp
            RTXPTSubInstanceData SubEntry;
            SubEntry.MaterialID  = Primitive.MaterialId;
            SubEntry.FirstVertex = BaseVertex + Primitive.FirstVertex;
            SubEntry.VertexCount = Primitive.VertexCount;
            if (Primitive.HasIndices())
            {
                SubEntry.Flags |= kRTXPTSubInstanceFlag_Indexed;
                SubEntry.FirstIndex = FirstIndex + Primitive.FirstIndex;
                SubEntry.IndexCount = Primitive.IndexCount;
            }
            SubInstances.emplace_back(SubEntry);
```

`BaseVertex` and `FirstIndex` are already computed near the top of `BuildStaticScene` (`Model.GetBaseVertex()` and `pIndexBuffer != nullptr ? Model.GetFirstIndexLocation() : 0`), so no additional bookkeeping is required.

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit extended sub-instance data**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
git -C DiligentSamples commit -m "feat(rtxpt): pack phase 5.2 vertex and index offsets into sub-instance data" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the acceleration structures changes.

---

### Task 3: Mirror The New Layouts In Shader-Shared Headers And Add Vertex/Index Bridge

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli`

- [ ] **Step 1: Update the shared HLSL header**

Replace the contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli` with:

```hlsl
#ifndef RTXPT_SHADER_SHARED_HLSLI
#define RTXPT_SHADER_SHARED_HLSLI

// Mirrors Diligent::kRTXPTSubInstanceFlag_Indexed in RTXPTAccelerationStructures.hpp.
static const uint kRTXPTSubInstanceFlagIndexed = 0x1u;

// Mirrors Diligent::RTXPTPathTracerSettings (the new sub-struct embedded in RTXPTFrameConstants).
struct RTXPTPathTracerSettings
{
    uint MaxBounces;        // Maximum number of secondary bounces; 0 means primary-ray only.
    uint AccumulationFrame; // 0-based index of the sample being added this frame.
    uint ResetAccumulation; // Non-zero means raygen should overwrite the accumulation buffer instead of blending.
    uint MinBounces;        // Reserved for Phase 5.3 Russian roulette; ignored by Phase 5.2.
};

// Mirrors Diligent::RTXPTFrameConstants in RTXPTSample.hpp (must keep order and layout in sync).
struct RTXPTFrameConstants
{
    float4x4                ViewProj;
    float4x4                ViewProjInv;
    float4                  CameraPosition_Time;
    float4                  ViewportSize_FrameIdx;
    RTXPTPathTracerSettings PathTracer;
};

// Primary ray payload (Phase 5.1 compatibility — kept for the bridge sanity helpers).
struct RTXPTPrimaryPayload
{
    float4 ColorDepth;
};

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

// Mirrors Diligent::RTXPTSubInstanceData in RTXPTAccelerationStructures.hpp.
// One entry per (BLAS instance, geometry) pair. The C++ side stores the per-instance
// sub-instance base in TLAS CustomId, exposed in closest-hit shaders as InstanceID().
// index = InstanceID() + GeometryIndex().
struct RTXPTSubInstanceData
{
    uint MaterialID;
    uint Flags;
    uint FirstIndex;
    uint IndexCount;
    uint FirstVertex;
    uint VertexCount;
    uint Padding0;
    uint Padding1;
};

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

// Mirrors Diligent::RTXPTLightData in RTXPTLights.hpp.
struct RTXPTLightData
{
    float4 ColorIntensity;
    float4 PositionRange;
    float4 DirectionType;
    float4 SpotAngles;
};

// Per-vertex layout for vertex buffer 0 of the default Diligent GLTF model
// (POSITION + NORMAL + TEXCOORD_0). Total size = 32 bytes; must equal the
// vertex stride captured by RTXPTScene::GetVertexStride0().
struct RTXPTVertex
{
    float3 Position;
    float3 Normal;
    float2 TexCoord0;
};

#endif // RTXPT_SHADER_SHARED_HLSLI
```

- [ ] **Step 2: Extend the scene bridge with vertex/index access**

Replace the contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli` with:

```hlsl
#ifndef RTXPT_SCENE_BRIDGE_HLSLI
#define RTXPT_SCENE_BRIDGE_HLSLI

#include "RTXPTShaderShared.hlsli"

// Global shader resources used by the scene bridge. C++ binds these as static SRVs.
ConstantBuffer<RTXPTFrameConstants>    g_FrameConstants;
StructuredBuffer<RTXPTSubInstanceData> g_SubInstanceData;
StructuredBuffer<RTXPTLightData>       g_Lights;
StructuredBuffer<RTXPTVertex>          g_VertexBuffer;
Buffer<uint>                           g_IndexBuffer;

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
        uint Count  = 0;
        uint Stride = 0;
        g_SubInstanceData.GetDimensions(Count, Stride);
        return Count > 0;
    }

    // Fetch the 3 vertex indices for triangle `LocalPrimitiveIndex` within the geometry
    // described by `SubInstance`. Falls back to a fan (i,i,i+1,i+2 sequence) for non-indexed
    // primitives so the math stays valid; non-indexed geometries are flagged via Flags.
    uint3 GetTriangleIndices(RTXPTSubInstanceData SubInstance, uint LocalPrimitiveIndex)
    {
        const uint Base = LocalPrimitiveIndex * 3u;
        if ((SubInstance.Flags & kRTXPTSubInstanceFlagIndexed) != 0u)
        {
            return uint3(
                g_IndexBuffer[SubInstance.FirstIndex + Base + 0u],
                g_IndexBuffer[SubInstance.FirstIndex + Base + 1u],
                g_IndexBuffer[SubInstance.FirstIndex + Base + 2u]);
        }
        return uint3(Base + 0u, Base + 1u, Base + 2u);
    }

    // Fetch the 3 vertex records for the current closest-hit triangle.
    void GetTriangleVertices(RTXPTSubInstanceData SubInstance,
                             uint                 LocalPrimitiveIndex,
                             out RTXPTVertex      V0,
                             out RTXPTVertex      V1,
                             out RTXPTVertex      V2)
    {
        const uint3 Indices = GetTriangleIndices(SubInstance, LocalPrimitiveIndex);
        V0                  = g_VertexBuffer[SubInstance.FirstVertex + Indices.x];
        V1                  = g_VertexBuffer[SubInstance.FirstVertex + Indices.y];
        V2                  = g_VertexBuffer[SubInstance.FirstVertex + Indices.z];
    }

    // Barycentric-interpolated object-space normal -> world-space, renormalized.
    float3 InterpolateNormal(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary       = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        const float3 ObjNormal  = V0.Normal * Bary.x + V1.Normal * Bary.y + V2.Normal * Bary.z;
        const float3 WorldNormal = mul((float3x3) ObjectToWorld3x4(), ObjNormal);
        const float  Len         = length(WorldNormal);
        return Len > 1e-6 ? WorldNormal / Len : float3(0.0, 1.0, 0.0);
    }

    // Geometric (face) normal in world space; used as a fallback when interpolated normals
    // collapse (e.g. degenerate triangles or missing data).
    float3 ComputeGeometricNormal(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2)
    {
        const float3 ObjFaceNormal   = cross(V1.Position - V0.Position, V2.Position - V0.Position);
        const float3 WorldFaceNormal = mul((float3x3) ObjectToWorld3x4(), ObjFaceNormal);
        const float  Len             = length(WorldFaceNormal);
        return Len > 1e-6 ? WorldFaceNormal / Len : float3(0.0, 1.0, 0.0);
    }

    // World-space hit position using ObjectToWorld3x4().
    float3 ComputeWorldHitPosition(RTXPTVertex V0, RTXPTVertex V1, RTXPTVertex V2, float2 Barycentrics)
    {
        const float3 Bary    = float3(1.0 - Barycentrics.x - Barycentrics.y, Barycentrics.x, Barycentrics.y);
        const float3 ObjPos  = V0.Position * Bary.x + V1.Position * Bary.y + V2.Position * Bary.z;
        return mul(ObjectToWorld3x4(), float4(ObjPos, 1.0));
    }
#endif

    // Total active light count. May be zero on scenes without lights.
    uint GetLightCount()
    {
        uint Count  = 0;
        uint Stride = 0;
        g_Lights.GetDimensions(Count, Stride);
        return Count;
    }

    RTXPTLightData GetLight(uint Index)
    {
        return g_Lights[Index];
    }
} // namespace Bridge

// TODO(RTXPT-Port Phase 5.3): Add alpha-mask/transparent flags to RTXPTSubInstanceData and propagate them into any-hit specialization.
// TODO(RTXPT-Port Phase 5.3): Bind material textures and respect TextureShaderAttribs UV selectors / wrap modes.

#endif // RTXPT_SCENE_BRIDGE_HLSLI
```

Notes:

- `ObjectToWorld3x4()` is the HLSL ray tracing intrinsic for the per-instance object-to-world matrix; it is valid in any-hit, closest-hit, and intersection shaders. We use the 3x3 upper-left for normal transforms; that matches the BLAS instances being rigid transforms (no non-uniform scale). RTXPT's IntroPathTracer applies the same simplification.
- The Phase 5.2 path tracer uses smooth (interpolated) normals by default and only falls back to `ComputeGeometricNormal` on a near-zero normal length. That keeps the reference image consistent with what the rasterized version would show.
- The previous Phase 5.1 TODOs covered by this header are removed; the residual TODOs reference Phase 5.3 work.

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit shared header + scene bridge extensions**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli Samples/RTXPT/assets/shaders/RTXPTSceneBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): extend phase 5.2 bridge with vertex and index fetch" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the two HLSL headers.

---

### Task 4: Add Path-Tracer Settings, Frame Constants Sync, And Accumulation Render Target

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add the PathTracerSettings struct and sample state**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`. Locate the existing `RTXPTFrameConstants` struct:

```cpp
struct RTXPTFrameConstants
{
    float4x4 ViewProj              = float4x4::Identity();
    float4x4 ViewProjInv           = float4x4::Identity();
    float4   CameraPosition_Time   = float4{0, 0, 0, 0};
    float4   ViewportSize_FrameIdx = float4{0, 0, 0, 0};
};
```

Replace it with:

```cpp
struct RTXPTPathTracerSettings
{
    Uint32 MaxBounces        = 4;
    Uint32 AccumulationFrame = 0;
    Uint32 ResetAccumulation = 1;
    Uint32 MinBounces        = 0;
};
static_assert(sizeof(RTXPTPathTracerSettings) == 16, "RTXPTPathTracerSettings layout must match RTXPTShaderShared.hlsli");

struct RTXPTFrameConstants
{
    float4x4                ViewProj              = float4x4::Identity();
    float4x4                ViewProjInv           = float4x4::Identity();
    float4                  CameraPosition_Time   = float4{0, 0, 0, 0};
    float4                  ViewportSize_FrameIdx = float4{0, 0, 0, 0};
    RTXPTPathTracerSettings PathTracer            = {};
};
static_assert(sizeof(RTXPTFrameConstants) == 176, "RTXPTFrameConstants layout must match RTXPTShaderShared.hlsli");
```

Then locate the existing sample state block:

```cpp
    RefCntAutoPtr<IBuffer>      m_FrameConstantsCB;
    RTXPTFrameConstants         m_LastFrameConstants;
    Uint32                      m_FrameIndex             = 0;
    bool                        m_EnableDebugComputePass = true;
```

Replace it with:

```cpp
    RefCntAutoPtr<IBuffer>      m_FrameConstantsCB;
    RTXPTFrameConstants         m_LastFrameConstants;
    Uint32                      m_FrameIndex             = 0;
    Uint32                      m_AccumulationFrame      = 0;
    Uint32                      m_MaxBounces             = 4;
    bool                        m_EnableDebugComputePass = false;
    bool                        m_ResetAccumulationPending = true;
    bool                        m_AccumulationActive      = false;
```

`m_EnableDebugComputePass` defaults to `false` now: the Phase 5.1 debug compute pass overwrites the path-tracer image with a sweep pattern and would obscure the accumulated result. Users can re-enable it from the UI.

Add the helper method declaration in the `private:` block of `RTXPTSample`:

```cpp
    void RequestAccumulationReset(const char* Reason);
```

Replace the `EnsureRenderTargets` declaration with an overload that does not allocate the accumulation target on its own (now owned by the caller):

```cpp
    bool EnsureRenderTargets();
```

(declaration unchanged but its body in `.cpp` changes).

- [ ] **Step 2: Add the accumulation render target**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`. Replace the existing `Resize` declaration:

```cpp
    bool Resize(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT Format, bool CreateComputeOutput);
```

with:

```cpp
    bool Resize(IRenderDevice* pDevice,
                Uint32         Width,
                Uint32         Height,
                TEXTURE_FORMAT Format,
                bool           CreateComputeOutput,
                bool           CreateAccumulation);
```

Add the new accessors next to the existing ones:

```cpp
    bool          IsAccumulationActive() const { return m_AccumColor != nullptr; }
    ITextureView* GetAccumColorUAV() const;
    ITextureView* GetAccumColorSRV() const;
```

Add a private member next to `m_ComputeColor`:

```cpp
    RefCntAutoPtr<ITexture> m_AccumColor;
```

- [ ] **Step 3: Implement accumulation render target creation**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`. Replace the body of `Resize` with:

```cpp
bool RTXPTRenderTargets::Resize(IRenderDevice* pDevice,
                                Uint32         Width,
                                Uint32         Height,
                                TEXTURE_FORMAT Format,
                                bool           CreateComputeOutput,
                                bool           CreateAccumulation)
{
    if (Width == 0 || Height == 0)
        return false;

    const bool HasRequestedTargets =
        m_OutputColor != nullptr &&
        (!CreateComputeOutput || m_ComputeColor != nullptr) &&
        (CreateComputeOutput || m_ComputeColor == nullptr) &&
        (!CreateAccumulation || m_AccumColor != nullptr) &&
        (CreateAccumulation || m_AccumColor == nullptr);

    if (HasRequestedTargets && m_Width == Width && m_Height == Height && m_Format == Format)
        return true;

    Reset();
    m_Width  = Width;
    m_Height = Height;
    m_Format = Format;

    if (!CreateTarget(pDevice, "RTXPT OutputColor", m_OutputColor))
        return false;

    if (CreateComputeOutput && !CreateTarget(pDevice, "RTXPT ComputeColor", m_ComputeColor))
        return false;

    if (CreateAccumulation)
    {
        const TEXTURE_FORMAT AccumFormat = TEX_FORMAT_RGBA32_FLOAT;
        const auto&          FmtInfo     = pDevice->GetTextureFormatInfoExt(AccumFormat);
        const bool           SupportsUAV = (FmtInfo.BindFlags & BIND_UNORDERED_ACCESS) != 0;
        if (!SupportsUAV)
        {
            m_LastError = "RGBA32F UAV is not supported; reference path tracer accumulation is disabled";
            // Leave m_AccumColor null so RTXPTSample falls back to non-accumulated rendering.
            return true;
        }

        TextureDesc Desc;
        Desc.Name      = "RTXPT AccumColor";
        Desc.Type      = RESOURCE_DIM_TEX_2D;
        Desc.Width     = m_Width;
        Desc.Height    = m_Height;
        Desc.Format    = AccumFormat;
        Desc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
        pDevice->CreateTexture(Desc, nullptr, &m_AccumColor);

        if (!m_AccumColor)
        {
            m_LastError = "Failed to create RTXPT AccumColor";
            return false;
        }
    }

    return true;
}
```

Replace the body of `Reset` with:

```cpp
void RTXPTRenderTargets::Reset()
{
    m_OutputColor.Release();
    m_ComputeColor.Release();
    m_AccumColor.Release();
    m_Width  = 0;
    m_Height = 0;
    m_Format = TEX_FORMAT_UNKNOWN;
    m_LastError.clear();
}
```

Append the two new accessors immediately below `GetComputeColorSRV()`:

```cpp
ITextureView* RTXPTRenderTargets::GetAccumColorUAV() const
{
    return m_AccumColor ? m_AccumColor->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetAccumColorSRV() const
{
    return m_AccumColor ? m_AccumColor->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}
```

- [ ] **Step 4: Update `RTXPTSample` callers to request accumulation**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`. Locate the existing `EnsureRenderTargets`:

```cpp
bool RTXPTSample::EnsureRenderTargets()
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    return m_RenderTargets.Resize(m_pDevice,
                                  SCDesc.Width,
                                  SCDesc.Height,
                                  TEX_FORMAT_RGBA8_UNORM,
                                  m_FeatureCaps.ComputeShaders);
}
```

Replace it with:

```cpp
bool RTXPTSample::EnsureRenderTargets()
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    const bool           Ok     = m_RenderTargets.Resize(m_pDevice,
                                              SCDesc.Width,
                                              SCDesc.Height,
                                              TEX_FORMAT_RGBA8_UNORM,
                                              m_FeatureCaps.ComputeShaders,
                                              m_FeatureCaps.RayTracing);
    m_AccumulationActive       = Ok && m_RenderTargets.IsAccumulationActive();
    if (Ok)
        RequestAccumulationReset("Render targets (re)created");
    return Ok;
}
```

Locate `WindowResize`:

```cpp
void RTXPTSample::WindowResize(Uint32 Width, Uint32 Height)
{
    if (Width == 0 || Height == 0)
        return;

    m_RenderTargets.Resize(m_pDevice,
                           Width,
                           Height,
                           TEX_FORMAT_RGBA8_UNORM,
                           m_FeatureCaps.ComputeShaders);
}
```

Replace it with:

```cpp
void RTXPTSample::WindowResize(Uint32 Width, Uint32 Height)
{
    if (Width == 0 || Height == 0)
        return;

    if (m_RenderTargets.Resize(m_pDevice,
                               Width,
                               Height,
                               TEX_FORMAT_RGBA8_UNORM,
                               m_FeatureCaps.ComputeShaders,
                               m_FeatureCaps.RayTracing))
    {
        m_AccumulationActive = m_RenderTargets.IsAccumulationActive();
        RequestAccumulationReset("Window resized");
    }
}
```

- [ ] **Step 5: Add the accumulation reset helper and update frame constants**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, append the helper inside the `Diligent` namespace (just before the closing `} // namespace Diligent`):

```cpp
void RTXPTSample::RequestAccumulationReset(const char* /*Reason*/)
{
    m_AccumulationFrame        = 0;
    m_ResetAccumulationPending = true;
}
```

Then replace the body of `UpdateFrameConstants` with:

```cpp
void RTXPTSample::UpdateFrameConstants(double CurrTime)
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    const float          Width  = static_cast<float>(SCDesc.Width);
    const float          Height = static_cast<float>(SCDesc.Height);

    const float3   CameraPosition = float3{0.0f, 1.5f, -6.0f};
    const float4x4 CameraView     = float4x4::Translation(-CameraPosition.x, -CameraPosition.y, -CameraPosition.z);
    const float4x4 CameraProj     = GetAdjustedProjectionMatrix(PI_F / 4.0f, 0.1f, 10000.0f);
    const float4x4 ViewProj       = CameraView * CameraProj;

    m_LastFrameConstants.ViewProj              = ViewProj;
    m_LastFrameConstants.ViewProjInv           = ViewProj.Inverse();
    m_LastFrameConstants.CameraPosition_Time   = float4{CameraPosition.x, CameraPosition.y, CameraPosition.z, static_cast<float>(CurrTime)};
    m_LastFrameConstants.ViewportSize_FrameIdx = float4{Width, Height, Width > 0.0f ? 1.0f / Width : 0.0f, static_cast<float>(m_FrameIndex)};

    if (m_AccumulationActive)
    {
        if (m_ResetAccumulationPending)
            m_AccumulationFrame = 1;
        else
            ++m_AccumulationFrame;
    }
    else
    {
        m_AccumulationFrame = 0;
    }

    m_LastFrameConstants.PathTracer.MaxBounces        = m_MaxBounces;
    m_LastFrameConstants.PathTracer.AccumulationFrame = m_AccumulationFrame;
    m_LastFrameConstants.PathTracer.ResetAccumulation = m_ResetAccumulationPending ? 1u : 0u;
    m_LastFrameConstants.PathTracer.MinBounces        = 0;

    if (m_FrameConstantsCB)
    {
        MapHelper<RTXPTFrameConstants> Constants{m_pImmediateContext, m_FrameConstantsCB, MAP_WRITE, MAP_FLAG_DISCARD};
        *Constants = m_LastFrameConstants;
    }

    m_ResetAccumulationPending = false;

    ++m_FrameIndex;
}
```

`AccumulationFrame` is the 1-based index of the sample being added this frame: the first frame after a reset uses `AccumulationFrame = 1` with `ResetAccumulation = 1` (raygen overwrites the buffer); the second frame uses `AccumulationFrame = 2` with `ResetAccumulation = 0` (raygen computes `Accumulated = Previous + (Sample2 - Previous) * 1/2`, i.e. a running average). The increment must happen **before** the constant-buffer upload so the GPU sees the value matching this frame's sample, not the next frame's.

- [ ] **Step 6: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 7: Commit accumulation infrastructure**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.2 accumulation target and path tracer constants" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the four files.

---

### Task 5: Add Random Helpers And Reference Path Tracer Shaders

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss`

- [ ] **Step 1: Create the PRNG header**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli`:

```hlsl
#ifndef RTXPT_RANDOM_HLSLI
#define RTXPT_RANDOM_HLSLI

// Lightweight integer hash for PRNG seeding. Same constants used by RTXPT's
// IntroPathTracer (D:/RTXPT-fork/Rtxpt/Shaders/IntroSample/IntroPathTracer.hlsl)
// — chosen for good visual distribution rather than statistical strength.
uint Hash32(uint x)
{
    x ^= x >> 16;
    x *= 0x7feb352du;
    x ^= x >> 15;
    x *= 0x846ca68bu;
    x ^= x >> 16;
    return x;
}

uint Hash32Combine(uint Seed, uint Value)
{
    return Seed ^ (Hash32(Value) + 0x9e3779b9u + (Seed << 6) + (Seed >> 2));
}

// Map 23 random bits to [0, 1).
float ToFloat0To1(uint x)
{
    return asfloat(0x3f800000u | (x & 0x7fffffu)) - 1.0;
}

struct RTXPTRandom
{
    uint State;
};

RTXPTRandom RTXPTRandom_Init(uint2 PixelPos, uint FrameSeed)
{
    RTXPTRandom Rng;
    const uint  PixelSeed = (PixelPos.x << 16) | (PixelPos.y & 0xffffu);
    Rng.State             = Hash32Combine(Hash32(FrameSeed), PixelSeed);
    return Rng;
}

float NextFloat(inout RTXPTRandom Rng)
{
    Rng.State = Hash32(Rng.State);
    return ToFloat0To1(Rng.State);
}

float2 NextFloat2(inout RTXPTRandom Rng)
{
    const float X = NextFloat(Rng);
    const float Y = NextFloat(Rng);
    return float2(X, Y);
}

// Build an orthonormal basis (Tangent, Bitangent) from a unit normal. Frisvad 2012.
void BuildOrthonormalBasis(float3 Normal, out float3 Tangent, out float3 Bitangent)
{
    if (Normal.z < -0.9999999)
    {
        Tangent   = float3(0.0, -1.0, 0.0);
        Bitangent = float3(-1.0, 0.0, 0.0);
        return;
    }
    const float A = 1.0 / (1.0 + Normal.z);
    const float B = -Normal.x * Normal.y * A;
    Tangent       = float3(1.0 - Normal.x * Normal.x * A, B, -Normal.x);
    Bitangent     = float3(B, 1.0 - Normal.y * Normal.y * A, -Normal.y);
}

// Cosine-weighted hemisphere sample around `Normal`. Returns a unit vector and
// the matching PDF in `Pdf`. PDF for Lambertian sampling is cos(theta) / PI.
float3 SampleCosineHemisphere(float2 Rand, float3 Normal, out float Pdf)
{
    const float R     = sqrt(Rand.x);
    const float Theta = 6.28318530718 * Rand.y;
    const float X     = R * cos(Theta);
    const float Y     = R * sin(Theta);
    const float Z     = sqrt(max(0.0, 1.0 - Rand.x));

    float3 Tangent;
    float3 Bitangent;
    BuildOrthonormalBasis(Normal, Tangent, Bitangent);

    Pdf = Z * 0.318309886184; // Z / PI
    return normalize(Tangent * X + Bitangent * Y + Normal * Z);
}

#endif // RTXPT_RANDOM_HLSLI
```

- [ ] **Step 2: Create the reference miss shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss`:

```hlsl
#include "RTXPTSceneBridge.hlsli"

[shader("miss")]
void main(inout RTXPTPathTracerPayload Payload)
{
    const float3 RayDir  = WorldRayDirection();
    const float  T       = saturate(RayDir.y * 0.5 + 0.5);
    const float3 Horizon = float3(0.48, 0.58, 0.68);
    const float3 Zenith  = float3(0.05, 0.08, 0.14);
    float3       Sky     = lerp(Horizon, Zenith, T);

    // Optional sun-disk tint from the first directional light. Type encoding matches
    // LightTypeToShaderValue in RTXPTLights.cpp: 0 = Directional, 1 = Point, 2 = Spot.
    if (Bridge::GetLightCount() > 0)
    {
        const RTXPTLightData L    = Bridge::GetLight(0);
        const float          Type = L.DirectionType.w;
        if (Type < 0.5)
        {
            const float SunDot = saturate(dot(RayDir, -L.DirectionType.xyz));
            const float Disk   = pow(SunDot, 32.0);
            Sky += L.ColorIntensity.rgb * L.ColorIntensity.a * Disk * 0.05;
        }
    }

    Payload.WorldPos    = float3(0.0, 0.0, 0.0);
    Payload.HitDistance = -1.0;
    Payload.WorldNormal = float3(0.0, 1.0, 0.0);
    Payload.HitFlag     = 0u;
    Payload.BaseColor   = float3(0.0, 0.0, 0.0);
    Payload.Emission    = Sky;
}

// TODO(RTXPT-Port Phase 5.5): Replace the placeholder sun disk with environment map / NEE-driven sun sampling once the lighting baker is restored.
```

- [ ] **Step 3: Create the reference closest hit shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`:

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
    float3 BaseColor = float3(Attributes.barycentrics.x,
                              Attributes.barycentrics.y,
                              1.0 - Attributes.barycentrics.x - Attributes.barycentrics.y);
    float3 WorldNormal = -WorldRayDirection();
    float3 WorldPos    = WorldRayOrigin() + WorldRayDirection() * RayTCurrent();

    if (Bridge::HasSubInstanceTable() && Bridge::HasMaterialTable())
    {
        const RTXPTSubInstanceData SubInstance = Bridge::GetSubInstanceData();
        const RTXPTMaterialAttribs Material    = Bridge::GetMaterial(SubInstance.MaterialID);

        RTXPTVertex V0;
        RTXPTVertex V1;
        RTXPTVertex V2;
        Bridge::GetTriangleVertices(SubInstance, PrimitiveIndex(), V0, V1, V2);

        WorldPos    = Bridge::ComputeWorldHitPosition(V0, V1, V2, Attributes.barycentrics);
        WorldNormal = Bridge::InterpolateNormal(V0, V1, V2, Attributes.barycentrics);
        // Renormalize against the geometric normal if the interpolated normal is nearly zero
        // (degenerate vertex data) — keeps the shader robust on bad assets.
        if (dot(WorldNormal, WorldNormal) < 1e-6)
            WorldNormal = Bridge::ComputeGeometricNormal(V0, V1, V2);
        // Flip the shading normal to face the camera (single-sided diffuse lighting; transmission lands in Phase 5.3).
        if (dot(WorldNormal, WorldRayDirection()) > 0.0)
            WorldNormal = -WorldNormal;

        BaseColor = Material.BaseColorFactor.rgb;
    }

    Payload.WorldPos    = WorldPos;
    Payload.WorldNormal = normalize(WorldNormal);
    Payload.BaseColor   = BaseColor;
}

// TODO(RTXPT-Port Phase 5.3): Honor RTXPTMaterialAttribs.AlphaMode/AlphaCutoff via any-hit specialization instead of forcing opaque rays.
// TODO(RTXPT-Port Phase 5.3): Sample base color / normal / metallic-roughness textures using TextureShaderAttribs UV selectors.
// TODO(RTXPT-Port Phase 5.5): Add NEE shadow rays toward analytic and environment lights.
```

- [ ] **Step 4: Create the reference raygen shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`:

```hlsl
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTRandom.hlsli"

RaytracingAccelerationStructure          g_TLAS;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4>   g_OutputColor;
VK_IMAGE_FORMAT("rgba32f") RWTexture2D<float4> g_AccumColor;

static float3 ToneMapACES(float3 X)
{
    // Krzysztof Narkowicz fitted ACES curve. Matches Phase 5.1 visual mood when the accumulation buffer
    // is converted back to rgba8 for the blit pass.
    const float A = 2.51;
    const float B = 0.03;
    const float C = 2.43;
    const float D = 0.59;
    const float E = 0.14;
    return saturate((X * (A * X + B)) / (X * (C * X + D) + E));
}

[shader("raygeneration")]
void main()
{
    const uint2 Pixel      = DispatchRaysIndex().xy;
    const uint2 Dimensions = DispatchRaysDimensions().xy;

    const uint  FrameSeed = asuint(g_FrameConstants.ViewportSize_FrameIdx.w);
    RTXPTRandom Rng       = RTXPTRandom_Init(Pixel, FrameSeed);

    // Jitter inside the pixel — gives free anti-aliasing across accumulated samples.
    const float2 Jitter = NextFloat2(Rng);
    const float2 UV     = (float2(Pixel) + Jitter) / float2(Dimensions);
    const float2 NDC    = UV * 2.0 - 1.0;

    const float4 WorldPos4 = mul(float4(NDC, 1.0, 1.0), g_FrameConstants.ViewProjInv);
    const float3 Origin    = g_FrameConstants.CameraPosition_Time.xyz;
    float3       RayOrigin = Origin;
    float3       RayDir    = normalize(WorldPos4.xyz / WorldPos4.w - Origin);

    float3 Throughput  = float3(1.0, 1.0, 1.0);
    float3 PathRadiance = float3(0.0, 0.0, 0.0);

    const uint MaxBounces = max(g_FrameConstants.PathTracer.MaxBounces, 1u);

    [loop]
    for (uint Bounce = 0u; Bounce < MaxBounces; ++Bounce)
    {
        RayDesc Ray;
        Ray.Origin    = RayOrigin;
        Ray.Direction = RayDir;
        Ray.TMin      = 1e-3;
        Ray.TMax      = 10000.0;

        RTXPTPathTracerPayload Payload;
        Payload.WorldPos    = float3(0.0, 0.0, 0.0);
        Payload.HitDistance = -1.0;
        Payload.WorldNormal = float3(0.0, 1.0, 0.0);
        Payload.HitFlag     = 0u;
        Payload.BaseColor   = float3(0.0, 0.0, 0.0);
        Payload.Emission    = float3(0.0, 0.0, 0.0);
        Payload.Padding0    = 0.0;
        Payload.Padding1    = 0.0;

        TraceRay(g_TLAS,
                 RAY_FLAG_FORCE_OPAQUE,
                 0xFF,
                 0,
                 1,
                 0,
                 Ray,
                 Payload);

        // Accumulate any emission the payload picked up (sky on miss, future emissives on hit).
        PathRadiance += Throughput * Payload.Emission;

        if (Payload.HitFlag == 0u)
            break;

        // Lambertian bounce: cosine-weighted hemisphere sample. The throughput update for a
        // cosine-weighted sample of a Lambertian BRDF is just baseColor (the cos and pdf cancel),
        // which matches the standard reference path tracer derivation.
        float  Pdf       = 0.0;
        const float2 Rand    = NextFloat2(Rng);
        const float3 NextDir = SampleCosineHemisphere(Rand, Payload.WorldNormal, Pdf);
        if (Pdf <= 0.0)
            break;

        Throughput *= Payload.BaseColor;

        // Offset the next ray slightly along the normal to avoid self-intersection.
        const float Bias = max(1e-4, 1e-3 * Payload.HitDistance);
        RayOrigin       = Payload.WorldPos + Payload.WorldNormal * Bias;
        RayDir          = NextDir;
    }

    // Blend into the accumulation buffer. ResetAccumulation == 1 means this is the first sample after a reset.
    float3 Accumulated = PathRadiance;
    const uint Reset   = g_FrameConstants.PathTracer.ResetAccumulation;
    const uint Frame   = max(g_FrameConstants.PathTracer.AccumulationFrame, 1u);
    if (Reset == 0u)
    {
        const float4 Previous = g_AccumColor[Pixel];
        const float  InvN     = 1.0 / float(Frame);
        Accumulated           = Previous.rgb + (PathRadiance - Previous.rgb) * InvN;
    }
    g_AccumColor[Pixel]  = float4(Accumulated, 1.0);

    // OutputColor is the rgba8 image consumed by the existing blit/compute chain.
    g_OutputColor[Pixel] = float4(ToneMapACES(Accumulated), 1.0);
}

// TODO(RTXPT-Port Phase 5.3): Replace single-lobe Lambertian sampling with a proper BSDF (GGX + diffuse + transmission).
// TODO(RTXPT-Port Phase 5.5): Add explicit light sampling and MIS once the lighting baker is restored.
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli Samples/RTXPT/assets/shaders/RTXPTReference.rgen Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/assets/shaders/RTXPTReference.rmiss
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit the reference shader set**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli Samples/RTXPT/assets/shaders/RTXPTReference.rgen Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/assets/shaders/RTXPTReference.rmiss
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.2 reference path tracer shaders" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the four shader files.

---

### Task 6: Wire The Reference Path Tracer Into RTXPTRayTracingPass

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Extend the pass interface and stats**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`. Replace `RTXPTRayTracingPassStats` with:

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

Replace the `Initialize` declaration with:

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

Replace the `Trace` declaration with:

```cpp
    bool Trace(IDeviceContext* pContext,
               ITextureView*   pOutputUAV,
               ITextureView*   pAccumulationUAV,
               Uint32          Width,
               Uint32          Height);
```

Add a private member to keep the explicit index buffer view alive:

```cpp
    RefCntAutoPtr<IBufferView> m_IndexBufferView;
```

- [ ] **Step 2: Implement the new Initialize body**

Replace the body of `RTXPTRayTracingPass::Initialize` in `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` with:

```cpp
bool RTXPTRayTracingPass::Initialize(IRenderDevice*  pDevice,
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

    if (pVertexBuffer == nullptr || pIndexBuffer == nullptr)
    {
        m_Stats.DisabledReason = "Vertex or index buffer is unavailable for the reference path tracer";
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

    RefCntAutoPtr<IShader> pClosestHit;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_CLOSEST_HIT;
    ShaderCI.Desc.Name       = "RTXPT reference closest hit";
    ShaderCI.FilePath        = "RTXPTReference.rchit";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pClosestHit);

    if (!pRayGen || !pMiss || !pClosestHit)
    {
        m_Stats.LastError = "Failed to create RTXPT reference ray tracing shaders";
        return false;
    }

    RayTracingPipelineStateCreateInfoX PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT reference RT PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_RAY_TRACING;
    PSOCreateInfo.AddGeneralShader("Main", pRayGen);
    PSOCreateInfo.AddGeneralShader("PrimaryMiss", pMiss);
    PSOCreateInfo.AddTriangleHitShader("PrimaryHit", pClosestHit);
    PSOCreateInfo.RayTracingPipeline.MaxRecursionDepth = 1; // Raygen drives bounces in a loop; chit/miss do not recurse.
    PSOCreateInfo.RayTracingPipeline.ShaderRecordSize  = 0;
    PSOCreateInfo.MaxAttributeSize                     = static_cast<Uint32>(sizeof(float) * 2);
    // RTXPTPathTracerPayload = 4 * float4 = 64 bytes.
    PSOCreateInfo.MaxPayloadSize                       = static_cast<Uint32>(sizeof(float) * 16);

    // Stage map for Phase 5.2:
    //   g_FrameConstants   -> raygen    (camera + path tracer settings)
    //   g_TLAS             -> raygen
    //   g_Materials        -> closest hit
    //   g_SubInstanceData  -> closest hit
    //   g_VertexBuffer     -> closest hit
    //   g_IndexBuffer      -> closest hit
    //   g_Lights           -> miss
    //   g_OutputColor      -> raygen (rgba8 display image)
    //   g_AccumColor       -> raygen (rgba32f accumulation image)
    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_FrameConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_TLAS", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_Materials", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_SubInstanceData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_VertexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_CLOSEST_HIT, "g_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_MISS, "g_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_AccumColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
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
    IndexViewDesc.Name              = "RTXPT reference index buffer SRV";
    IndexViewDesc.ViewType          = BUFFER_VIEW_SHADER_RESOURCE;
    IndexViewDesc.Format.ValueType  = IndexValueType;
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

- [ ] **Step 3: Implement the new Trace body**

Replace the body of `RTXPTRayTracingPass::Trace` with:

```cpp
bool RTXPTRayTracingPass::Trace(IDeviceContext* pContext,
                                ITextureView*   pOutputUAV,
                                ITextureView*   pAccumulationUAV,
                                Uint32          Width,
                                Uint32          Height)
{
    m_Stats.LastTraceExecuted = false;
    m_Stats.AccumulationBound = false;

    if (!IsReady() || pOutputUAV == nullptr || pAccumulationUAV == nullptr || Width == 0 || Height == 0)
        return false;

    IShaderResourceVariable* pOutputColorVar = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "g_OutputColor");
    IShaderResourceVariable* pAccumColorVar  = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "g_AccumColor");
    if (pOutputColorVar == nullptr || pAccumColorVar == nullptr)
    {
        m_Stats.LastError = "Failed to find RTXPT output bindings";
        return false;
    }

    pOutputColorVar->Set(pOutputUAV);
    pAccumColorVar->Set(pAccumulationUAV);
    m_Stats.AccumulationBound = true;

    pContext->SetPipelineState(m_PSO);
    pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

    TraceRaysAttribs Attribs;
    Attribs.DimensionX = Width;
    Attribs.DimensionY = Height;
    Attribs.pSBT       = m_SBT;
    pContext->TraceRays(Attribs);

    m_Stats.LastTraceExecuted = true;
    ++m_Stats.TraceCount;
    return true;
}
```

- [ ] **Step 4: Reset the new state**

Update the body of `RTXPTRayTracingPass::Reset`:

```cpp
void RTXPTRayTracingPass::Reset()
{
    m_PSO.Release();
    m_SRB.Release();
    m_SBT.Release();
    m_TLAS.Release();
    m_IndexBufferView.Release();
    m_Stats = {};
}
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit RT pass wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): drive phase 5.2 reference path tracer from the rt pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the ray tracing pass changes.

---

### Task 7: Wire The Sample, Drop Phase 5.1 Minimal Shaders, Update CMake

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`
- Delete: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`

- [ ] **Step 1: Forward the new buffers and trace inputs from the sample**

Modify `RTXPTSample::CreatePhase4Passes` in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`. Replace the existing body with:

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

Replace the body of `RTXPTSample::Render` with:

```cpp
void RTXPTSample::Render()
{
    const auto ClearColor = float4{0.05f, 0.05f, 0.07f, 1.0f};

    if (!EnsureRenderTargets())
    {
        ClearFallback(ClearColor);
        return;
    }

    const bool TraceExecuted =
        m_RayTracingPass.Trace(m_pImmediateContext,
                               m_RenderTargets.GetOutputColorUAV(),
                               m_RenderTargets.GetAccumColorUAV(),
                               m_RenderTargets.GetWidth(),
                               m_RenderTargets.GetHeight());

    if (!TraceExecuted)
    {
        ClearFallback(ClearColor);
        return;
    }

    const bool ComputeExecuted =
        m_EnableDebugComputePass &&
        m_DebugComputePass.Dispatch(m_pImmediateContext,
                                    m_RenderTargets.GetOutputColorSRV(),
                                    m_RenderTargets.GetComputeColorUAV(),
                                    m_RenderTargets.GetWidth(),
                                    m_RenderTargets.GetHeight());

    ITextureView* pDisplaySRV = m_RenderTargets.GetDisplaySRV(ComputeExecuted);
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pDisplaySRV))
    {
        ClearFallback(ClearColor);
        return;
    }
}
```

The raygen writes the tone-mapped result into `OutputColor` (rgba8), so the existing blit pass continues to work without changes. The accumulation buffer is the source of truth and gets blended back into OutputColor every frame.

- [ ] **Step 2: Add the UI for path-tracer settings and accumulation diagnostics**

Inside `RTXPTSample::UpdateUI()` in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate the existing ImGui block that prints `TraceRays pass:`:

```cpp
    ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
    ImGui::Text("Material bridge: %s", RTPassStats.MaterialBridgeBound ? "bound" : "fallback");
    ImGui::Text("Sub-instance bridge: %s", RTPassStats.SubInstanceBound ? "bound" : "fallback");
    ImGui::Text("Light bridge: %s", RTPassStats.LightBridgeBound ? "bound" : "fallback");
    ImGui::Text("TraceRays executed: %s", RTPassStats.LastTraceExecuted ? "yes" : "no");
    ImGui::Text("TraceRays count: %u", RTPassStats.TraceCount);
```

Replace it with:

```cpp
    ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
    ImGui::Text("Material bridge: %s", RTPassStats.MaterialBridgeBound ? "bound" : "fallback");
    ImGui::Text("Sub-instance bridge: %s", RTPassStats.SubInstanceBound ? "bound" : "fallback");
    ImGui::Text("Light bridge: %s", RTPassStats.LightBridgeBound ? "bound" : "fallback");
    ImGui::Text("Vertex buffer: %s", RTPassStats.VertexBufferBound ? "bound" : "fallback");
    ImGui::Text("Index buffer: %s", RTPassStats.IndexBufferBound ? "bound" : "fallback");
    ImGui::Text("Accumulation target: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA8 fallback)");
    ImGui::Text("Accumulation frame: %u", m_AccumulationFrame);
    ImGui::Text("TraceRays executed: %s", RTPassStats.LastTraceExecuted ? "yes" : "no");
    ImGui::Text("TraceRays count: %u", RTPassStats.TraceCount);
    int MaxBouncesUI = static_cast<int>(m_MaxBounces);
    if (ImGui::SliderInt("Max bounces", &MaxBouncesUI, 1, 16))
    {
        m_MaxBounces = static_cast<Uint32>(MaxBouncesUI);
        RequestAccumulationReset("Max bounces changed");
    }
    if (ImGui::Button("Reset accumulation"))
        RequestAccumulationReset("User reset");
```

Then replace the existing TODO ImGui lines (the two `ImGui::Text("TODO(RTXPT-Port Phase 4): ...")` and `ImGui::Text("TODO(RTXPT-Port Phase 5.2): ...")` lines near the end of the panel) with:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 4): expose stable-plane, RTXDI, light feedback, and denoising-guide pass toggles after their shaders are ported.");
    ImGui::Text("TODO(RTXPT-Port Phase 5.3): swap flat Lambertian for the full BSDF (GGX + transmission + alpha-test).");
    ImGui::Text("TODO(RTXPT-Port Phase 5.5): add explicit light sampling and MIS once the lighting baker is restored.");
```

- [ ] **Step 3: Update CMakeLists.txt and delete the Phase 5.1 shader stubs**

Modify `DiligentSamples/Samples/RTXPT/CMakeLists.txt`. Replace the existing `SHADERS` block with:

```cmake
set(SHADERS
    assets/shaders/RTXPTCommon.fxh
    assets/shaders/RTXPTShaderShared.hlsli
    assets/shaders/RTXPTSceneBridge.hlsli
    assets/shaders/RTXPTMaterialBridge.hlsli
    assets/shaders/RTXPTRandom.hlsli
    assets/shaders/RTXPTReference.rgen
    assets/shaders/RTXPTReference.rmiss
    assets/shaders/RTXPTReference.rchit
    assets/shaders/RTXPTDebugCompute.csh
    assets/shaders/RTXPTBlit.vsh
    assets/shaders/RTXPTBlit.psh
)
```

Delete the now-unused minimal shader files:

```bash
git -C DiligentSamples rm Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit
```

- [ ] **Step 4: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/CMakeLists.txt
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit sample wiring and shader file cleanup**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 5.2 reference path tracer into the sample" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the sample changes, the CMake update, and the three deleted shader files.

---

### Task 8: Phase 5.2 Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [ ] **Step 1: Confirm structured Phase 5 TODOs**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 5" DiligentSamples/Samples/RTXPT
```

Expected (Phase 5.2 markers gone; Phase 5.3+ markers remain):

```text
RTXPTSceneBridge.hlsli     : Phase 5.3 alpha flags, Phase 5.3 textures
RTXPTMaterialBridge.hlsli  : Phase 5.3 textures (still present)
RTXPTReference.rgen        : Phase 5.3 BSDF, Phase 5.5 NEE/MIS, Phase 6 tone mapping
RTXPTReference.rmiss       : Phase 5.5 environment / sun sampling
RTXPTReference.rchit       : Phase 5.3 alpha-test/any-hit, Phase 5.3 texture sampling, Phase 5.5 NEE
RTXPTSample.cpp            : Phase 5.3 BSDF swap, Phase 5.5 NEE
```

No matches should remain for `Phase 5.2` after this plan is complete.

- [ ] **Step 2: Confirm file registration**

Run:

```powershell
rg -n "RTXPTReference|RTXPTRandom" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: each of the three reference shader files and `RTXPTRandom.hlsli` is listed in `CMakeLists.txt`.

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit
```

Expected:

```text
False
False
False
```

- [ ] **Step 3: Confirm DiligentSamples log shows the Phase 5.2 commits**

Run:

```powershell
git -C DiligentSamples log --oneline -n 12
```

Expected (most recent first, with the seven Phase 5.2 commits above the six Phase 5.1 commits):

```text
feat(rtxpt): wire phase 5.2 reference path tracer into the sample
feat(rtxpt): drive phase 5.2 reference path tracer from the rt pass
feat(rtxpt): add phase 5.2 reference path tracer shaders
feat(rtxpt): add phase 5.2 accumulation target and path tracer constants
feat(rtxpt): extend phase 5.2 bridge with vertex and index fetch
feat(rtxpt): pack phase 5.2 vertex and index offsets into sub-instance data
feat(rtxpt): expose phase 5.2 vertex and index buffers to shaders
feat(rtxpt): wire phase 5.1 bridge buffers into sample
feat(rtxpt): port phase 5.1 minimal rt shaders onto bridge
feat(rtxpt): bind phase 5.1 material/sub-instance/light bridge srvs
feat(rtxpt): always upload default material and light entries
feat(rtxpt): build phase 5.1 sub-instance material map
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
Vertex buffer: bound
Index buffer: bound
Accumulation target: active (RGBA32F)
Accumulation frame: increases every frame, resets on resize / Max bounces / Reset button
TraceRays executed: yes
TraceRays count: increases every frame
```

Expected visual result: the bistro scene rendered with smooth Lambertian shading. The image is noisy after the first frame and visibly converges over the next ~50–200 frames as `Accumulation frame` grows. Adjusting "Max bounces" or clicking "Reset accumulation" restarts the convergence.

- [ ] **Step 6: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT-capable machine. Expected facts match the D3D12 run.

If standalone ray tracing shaders are unavailable on the Vulkan device, expected fallback facts are:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
Material/Sub-instance/Light/Vertex/Index bridge: any state is acceptable when TraceRays is disabled
Accumulation target: active or inactive (either is acceptable when TraceRays is disabled)
The sample still launches and clears the swapchain.
```

If the device does not support RGBA32F UAVs, expected facts are:

```text
Accumulation target: inactive (RGBA8 fallback)
Reference path tracer in progress, but EnsureRenderTargets logged "RGBA32F UAV is not supported; reference path tracer accumulation is disabled".
TraceRays pass: not ready (because the new Initialize requires both UAVs to be available — see Task 4 Step 4 wiring).
```

- [ ] **Step 7: Commit top-level submodule pointer and plan**

After all `DiligentSamples` Phase 5.2 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-28-rtxpt-phase5-2-reference-path-tracer.md
git commit -m "feat(samples): plan and add RTXPT phase 5.2 reference path tracer" -m "Co-Authored-By: GPT 5.5"
```

Expected: one top-level commit that records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] The plan implements only Phase 5 layer 4 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`. Layers 5–9 are explicitly deferred to subsequent plans.
- [x] Each task ends with a runnable sample: when ray tracing or RGBA32F UAVs are unavailable, the sample still launches and clears the swapchain via the existing `ClearFallback` path; missing required bridge bindings disable the RT pass instead of tracing with unbound SRVs.
- [x] Shared HLSL declarations (`RTXPTShaderShared.hlsli`) remain the single source of truth for the mirrored structures (`RTXPTSubInstanceData` at 32 bytes, `RTXPTFrameConstants` at 176 bytes, `RTXPTPathTracerSettings` at 16 bytes, `RTXPTPathTracerPayload` at 64 bytes, `RTXPTVertex` at 32 bytes). All C++ structs use `static_assert` to enforce the matching size.
- [x] `RTXPTAccelerationStructures` continues to own the sub-instance contract (`InstanceID() + GeometryIndex() → SubInstanceData` with `InstanceID()` sourced from TLAS `CustomId`); Phase 5.2 only widens that record with per-geometry vertex/index offsets.
- [x] `RTXPTRayTracingPass` keeps the no-RT and no-standalone-RT fallbacks intact, validates all required static bridge bindings (now including vertex and index buffers), and explicitly creates the typed `Buffer<uint>` view that the index buffer needs.
- [x] Every new shader file (`RTXPTRandom.hlsli`, `RTXPTReference.rgen`, `RTXPTReference.rmiss`, `RTXPTReference.rchit`) is registered in `DiligentSamples/Samples/RTXPT/CMakeLists.txt`; the obsolete `RTXPTMinimal.*` files are deleted and unregistered together.
- [x] No textures, tangent maps, NEE/MIS, alpha test, transmission, Russian roulette, stable planes, RTXDI, NRD, SER, OMM, or DLSS code is introduced — each remains a `TODO(RTXPT-Port Phase 5.3/5.4/5.5/5.6/5.7)` marker in the appropriate file.
- [x] Verification steps avoid build/runtime execution unless the user explicitly requests it.
- [x] Each task ends with a focused, single-purpose commit using the project's `Co-Authored-By: GPT 5.5` trailer convention (matches every Phase 5.1 `DiligentSamples` commit and the prior Phase 5.1 plan).
- [x] Type/name consistency check: `RTXPTSubInstanceData` has identical fields and 32-byte size in both C++ (Task 2 Step 1) and HLSL (Task 3 Step 1). `RTXPTFrameConstants` adds `PathTracer` as the last field in both C++ (Task 4 Step 1) and HLSL (Task 3 Step 1). `RTXPTPathTracerPayload` (HLSL only) declares 64 bytes, matched by `MaxPayloadSize = sizeof(float) * 16` in Task 6 Step 2. `RTXPTRayTracingPass::Initialize` adds `pVertexBuffer`, `pIndexBuffer`, `IndexValueType` in the same order in the header (Task 6 Step 1) and the implementation (Task 6 Step 2) and the call site (Task 7 Step 1). `Bridge::GetTriangleVertices`/`Bridge::InterpolateNormal`/`Bridge::ComputeGeometricNormal`/`Bridge::ComputeWorldHitPosition` are all declared in Task 3 Step 2 and consumed in `RTXPTReference.rchit` in Task 5 Step 3.
- [x] CLAUDE.md guidance honored: no backward-compatibility hacks (Phase 5.1 minimal shaders are deleted rather than kept as dead code); copyright dates remain `2026` per existing pattern; no test/build commands run automatically.
