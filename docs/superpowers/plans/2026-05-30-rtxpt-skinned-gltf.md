# RTXPT Skinned glTF Current Geometry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the RTXPT current-frame skinned glTF geometry path: GPU skinning writes a per-frame skinned vertex arena, dynamic BLAS updates from that arena, and ray-tracing shaders fetch the same current geometry.

**Architecture:** Keep static geometry on the existing GLTF vertex/index buffer path. Add a focused `RTXPTSkinnedGeometry` owner that detects skinned glTF node instances, allocates one skinned vertex arena plus joint buffer, and dispatches a compute shader per skinned node slice. Extend `SubInstanceData` with a skinned flag so `PathTracerBridge.hlsli` hides whether a hit fetches static vertices or skinned arena vertices.

**Tech Stack:** C++17 in `DiligentSamples/Samples/RTXPT/src/`, HLSL compute + ray-tracing bridge shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/`, DiligentCore `BuildBLAS(Update=true)`, DiligentTools `GLTFLoader`.

---

## Context You Need Before Starting

This plan implements the design in `docs/superpowers/specs/2026-05-30-rtxpt-skinned-gltf-design.md`.

The key invariant is frame coherence:

- GPU skinning output
- BLAS build/update vertex data
- closest-hit vertex fetch
- future R2 emissive-triangle build input

must all see the same current-frame geometry.

`DiligentSamples` is a git submodule. All implementation commits in this plan should be made inside `DiligentSamples/`, not at the umbrella root. The root repository only stores this plan/spec documentation and the submodule pointer.

## File Structure

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.{hpp,cpp}`: scene classification, animation transform updates, source GLTF buffer accessors.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.{hpp,cpp}`: new focused owner for skinned node records, skinned vertex arena, joint matrix buffer, skinning constants, compute dispatch.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/SkinnedVertexBuild.csh`: GPU skinning compute shader.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`: shared `SkinVertexData` and `kSubInstanceFlagSkinned`.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`: current-geometry fetch helper that switches between static and skinned buffers.
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.{hpp,cpp}`: static/skinned BLAS creation and skinned BLAS update.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}`: bind `t_SkinnedVertexBuffer` to closest-hit/any-hit stages.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}`: lifecycle orchestration and status UI.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt`: register new C++ and HLSL files.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`: record the Diligent-native current-geometry design.

## Cross-Cutting Contracts

- `SubInstanceData` stays 32 bytes. Only a new flag bit is added.
- Static vertex buffer remains the GLTF buffer 0 SRV.
- Skinned vertex arena is a separate structured buffer with the same `GeometryVertexData` layout and has `BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RAY_TRACING`.
- `t_SkinnedVertexBuffer` is always bound for the full path tracer. When no skinned geometry exists, bind a one-element dummy structured buffer.
- Source skin input uses GLTF default buffer 1 (`JOINTS_0 + WEIGHTS_0` as two `float4`s), so scene loading must give vertex buffer 1 `BIND_SHADER_RESOURCE`.
- Skinned BLAS records are created with `RAYTRACING_BUILD_AS_ALLOW_UPDATE`; static BLAS records keep the current flags.
- For the first implementation, each skinned node slice preserves source model vertex addressing. Allocate `ModelVertexCount` vertices per skinned node slice, so existing indices and `Primitive.FirstVertex` offsets remain valid without remapping. Later compaction can shrink slices if it updates all offsets together.

## Verification Note

Do not auto-run full build or GPU sample commands unless explicitly requested. Each task lists targeted checks and final manual commands. The primary acceptance check is that skinned vertex output, BLAS update, and closest-hit fetch use the same current-frame buffer.

---

### Task 1: Scene Classification, Animation Updates, And Source Buffer Accessors

Adds skinned-scene stats and exposes the GLTF skin-input buffer. This task is mostly inert: it detects and reports skinned content, but still renders static content through the existing path.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Add scene geometry stats to `RTXPTScene.hpp`**

Add this struct after `RTXPTSceneCamera`:

```cpp
struct RTXPTSceneGeometryStats
{
    bool   HasSkinnedGeometry    = false;
    bool   HasAnimations         = false;
    Uint32 SkinnedNodeCount      = 0;
    Uint32 SkinnedPrimitiveCount = 0;
    Uint32 SkinnedVertexCount    = 0;
};
```

Add these public methods to `RTXPTScene`:

```cpp
    const RTXPTSceneGeometryStats& GetGeometryStats() const { return m_GeometryStats; }
    bool                           HasSkinnedGeometry() const { return m_GeometryStats.HasSkinnedGeometry; }
    bool                           HasAnimation() const { return m_GeometryStats.HasAnimations; }
    bool                           IsGeometryDirty() const { return m_GeometryDirty; }
    void                           ClearGeometryDirty() { m_GeometryDirty = false; }

    IBuffer* GetSkinningBuffer(IRenderDevice* pDevice = nullptr, IDeviceContext* pContext = nullptr) const;
```

Add these private members:

```cpp
    RTXPTSceneGeometryStats m_GeometryStats;
    float                   m_AnimationTime  = 0.0f;
    Int32                   m_AnimationIndex = -1;
    bool                    m_GeometryDirty  = false;
```

- [ ] **Step 2: Add skinned counting helpers to `RTXPTScene.cpp`**

Add these helpers near `CountLightNodes`:

```cpp
RTXPTSceneGeometryStats ComputeGeometryStats(const GLTF::Scene& Scene, const GLTF::Model& Model)
{
    RTXPTSceneGeometryStats Stats;
    Stats.HasAnimations = !Model.Animations.empty();

    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr || pNode->pSkin == nullptr)
            continue;

        ++Stats.SkinnedNodeCount;
        Stats.HasSkinnedGeometry = true;

        for (const GLTF::Primitive& Primitive : pNode->pMesh->Primitives)
        {
            if (Primitive.VertexCount == 0 && Primitive.IndexCount == 0)
                continue;

            ++Stats.SkinnedPrimitiveCount;
            Stats.SkinnedVertexCount += Primitive.VertexCount;
        }
    }

    return Stats;
}
```

- [ ] **Step 3: Reset and cache the new stats**

In `ResetLoadedData`, reset the new fields:

```cpp
    m_GeometryStats = {};
    m_AnimationTime = 0.0f;
    m_AnimationIndex = -1;
    m_GeometryDirty = false;
```

In `CacheSceneData`, after `m_LightCount = CountLightNodes(Scene);`, add:

```cpp
    m_GeometryStats = ComputeGeometryStats(Scene, *m_Model);
    m_AnimationIndex = m_GeometryStats.HasAnimations ? 0 : -1;
    m_GeometryDirty = m_GeometryStats.HasSkinnedGeometry;
```

- [ ] **Step 4: Make GLTF buffer 1 shader-readable**

In `LoadDefaultScene`, keep buffer 0 unchanged and add buffer 1 SRV support after the buffer 0 assignment:

```cpp
    // Buffer 1 is the default GLTF skinning stream (JOINTS_0 + WEIGHTS_0).
    // RTXPTSkinnedGeometry reads it as StructuredBuffer<SkinVertexData> when skinned nodes exist.
    ModelCI.VertBufferBindFlags[1] = BIND_VERTEX_BUFFER | BIND_SHADER_RESOURCE;
```

- [ ] **Step 5: Update animation transforms in `RTXPTScene::Update`**

Replace the current empty body:

```cpp
void RTXPTScene::Update(double CurrTime, double ElapsedTime)
{
    (void)CurrTime;
    (void)ElapsedTime;
}
```

with:

```cpp
void RTXPTScene::Update(double CurrTime, double ElapsedTime)
{
    (void)CurrTime;

    if (!m_Model || m_AnimationIndex < 0 || !m_GeometryStats.HasSkinnedGeometry)
    {
        m_GeometryDirty = false;
        return;
    }

    const GLTF::Animation& Animation = m_Model->Animations[static_cast<Uint32>(m_AnimationIndex)];
    const float Duration = std::max(Animation.End - Animation.Start, 1e-5f);
    m_AnimationTime += static_cast<float>(ElapsedTime);
    const float WrappedTime = Animation.Start + std::fmod(m_AnimationTime, Duration);

    m_Model->ComputeTransforms(m_SceneIndex, m_Transforms, float4x4::Identity(), m_AnimationIndex, WrappedTime);
    m_GeometryDirty = true;
}
```

- [ ] **Step 6: Add `GetSkinningBuffer` implementation**

Add after `GetVertexBuffer0`:

```cpp
IBuffer* RTXPTScene::GetSkinningBuffer(IRenderDevice* pDevice, IDeviceContext* pContext) const
{
    return m_Model && m_Model->GetVertexBufferCount() > 1 ? m_Model->GetVertexBuffer(1, pDevice, pContext) : nullptr;
}
```

- [ ] **Step 7: Verify the inert scene change**

Run only if the user asks for verification:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected result: C++ compilation succeeds; default static Bistro path still builds static AS. If a skinned model is loaded later, `RTXPTScene::GetGeometryStats()` reports nonzero skinned counts.

- [ ] **Step 8: Commit Task 1 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
git commit -m "feat(rtxpt): classify skinned gltf scene geometry" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Shared Flags And Current-Geometry Bridge

Adds the skinned flag and shader bridge switch. This remains safe before the skinned arena exists by binding a dummy buffer in a later task.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`

- [ ] **Step 1: Add the C++ sub-instance flag**

In `RTXPTAccelerationStructures.hpp`, extend the flag block:

```cpp
// Flag bits for SubInstanceData::Flags.
constexpr Uint32 kSubInstanceFlag_Indexed = 0x1u;
constexpr Uint32 kSubInstanceFlag_Skinned = 0x2u;
```

- [ ] **Step 2: Add matching HLSL flag and skin input type**

In `PathTracerShared.h`, extend the constants near the top:

```hlsl
// Mirrors Diligent::kSubInstanceFlag_Indexed in RTXPTAccelerationStructures.hpp.
static const uint kSubInstanceFlagIndexed = 0x1u;
static const uint kSubInstanceFlagSkinned = 0x2u;
```

Add after `GeometryVertexData`:

```hlsl
// Mirrors the default GLTF skin stream in vertex buffer 1: JOINTS_0 followed by WEIGHTS_0.
struct SkinVertexData
{
    float4 joints;
    float4 weights;
};
```

- [ ] **Step 3: Bind the skinned vertex arena in the HLSL bridge**

In `PathTracerBridge.hlsli`, add the second geometry buffer beside `t_VertexBuffer`:

```hlsl
StructuredBuffer<GeometryVertexData>   t_VertexBuffer;
StructuredBuffer<GeometryVertexData>   t_SkinnedVertexBuffer;
Buffer<uint>                           t_IndexBuffer;
```

- [ ] **Step 4: Add a current-geometry fetch helper**

In the `ENABLE_HIT_BRIDGE` section, add this helper before `getTriangleVertices`:

```hlsl
    GeometryVertexData getGeometryVertex(SubInstanceData subInstance, uint vertexIndex)
    {
        if ((subInstance.Flags & kSubInstanceFlagSkinned) != 0u)
            return t_SkinnedVertexBuffer[subInstance.VertexOffset + vertexIndex];

        return t_VertexBuffer[subInstance.VertexOffset + vertexIndex];
    }
```

Then replace the three direct `t_VertexBuffer[...]` lines in `getTriangleVertices`:

```hlsl
        v0                  = t_VertexBuffer[subInstance.VertexOffset + indices.x];
        v1                  = t_VertexBuffer[subInstance.VertexOffset + indices.y];
        v2                  = t_VertexBuffer[subInstance.VertexOffset + indices.z];
```

with:

```hlsl
        v0                  = getGeometryVertex(subInstance, indices.x);
        v1                  = getGeometryVertex(subInstance, indices.y);
        v2                  = getGeometryVertex(subInstance, indices.z);
```

- [ ] **Step 5: Verify shared layout remains stable**

Run only if requested:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected result: `SubInstanceData` still compiles with `sizeof(SubInstanceData) == 32`; shader compilation may require Task 5 binding changes before the full sample builds if the compiler does not eliminate the new buffer in all variants.

- [ ] **Step 6: Commit Task 2 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli
git commit -m "feat(rtxpt): add skinned current-geometry bridge flag" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: GPU Skinned Vertex Arena And Compute Pass

Adds `RTXPTSkinnedGeometry`, its GPU resources, and the compute shader that writes current-frame skinned vertices.

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/SkinnedVertexBuild.csh`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create `RTXPTSkinnedGeometry.hpp`**

Use this header skeleton:

```cpp
#pragma once

#include <string>
#include <vector>

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "GLTFLoader.hpp"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderResourceBinding.h"

namespace Diligent
{

struct RTXPTGeometryVertex
{
    float3 position = float3{0, 0, 0};
    float3 normal   = float3{0, 1, 0};
    float2 texCoord0 = float2{0, 0};
};
static_assert(sizeof(RTXPTGeometryVertex) == 32, "RTXPTGeometryVertex layout must match GeometryVertexData");

struct RTXPTSkinnedNodeGeometry
{
    const GLTF::Node* pNode = nullptr;
    Uint32            SourceVertexBase = 0;
    Uint32            VertexBase = 0;
    Uint32            VertexCount = 0;
    Uint32            JointBase = 0;
    Uint32            JointCount = 0;
};

struct RTXPTSkinnedGeometryStats
{
    bool        Ready = false;
    bool        LastDispatchExecuted = false;
    Uint32      SkinnedNodeCount = 0;
    Uint32      SkinnedVertexCount = 0;
    Uint32      JointMatrixCount = 0;
    Uint32      DispatchCount = 0;
    std::string DisabledReason;
    std::string LastError;
};

class RTXPTSkinnedGeometry
{
public:
    void Reset();

    bool Initialize(IRenderDevice*               pDevice,
                    IEngineFactory*              pEngineFactory,
                    const GLTF::Model&           Model,
                    Uint32                       SceneIndex,
                    IBuffer*                     pSourceVertexBuffer,
                    IBuffer*                     pSourceSkinBuffer,
                    bool                         ComputeSupported);

    bool Update(IDeviceContext*              pContext,
                const GLTF::ModelTransforms& Transforms);

    bool HasSkinnedGeometry() const { return !m_Nodes.empty(); }
    bool IsReady() const { return m_Stats.Ready && m_SkinnedVertexBuffer; }

    IBuffer* GetSkinnedVertexBuffer() const { return m_SkinnedVertexBuffer; }
    const std::vector<RTXPTSkinnedNodeGeometry>& GetNodes() const { return m_Nodes; }
    const RTXPTSkinnedGeometryStats& GetStats() const { return m_Stats; }

private:
    struct SkinningConstants
    {
        Uint32 SourceVertexBase = 0;
        Uint32 DestVertexBase = 0;
        Uint32 JointBase = 0;
        Uint32 VertexCount = 0;
    };
    static_assert(sizeof(SkinningConstants) == 16, "SkinningConstants must stay 16-byte aligned");

    bool CreateBuffers(IRenderDevice* pDevice, IBuffer* pSourceVertexBuffer, IBuffer* pSourceSkinBuffer);
    bool CreatePipeline(IRenderDevice* pDevice, IEngineFactory* pEngineFactory);
    void BuildNodeTable(const GLTF::Model& Model, Uint32 SceneIndex);
    void UploadJointMatrices(IDeviceContext* pContext, const GLTF::ModelTransforms& Transforms);

    std::vector<RTXPTSkinnedNodeGeometry> m_Nodes;
    std::vector<float4x4>                 m_JointMatrices;
    RefCntAutoPtr<IBuffer>                m_SourceVertexBuffer;
    RefCntAutoPtr<IBuffer>                m_SourceSkinBuffer;
    RefCntAutoPtr<IBuffer>                m_SkinnedVertexBuffer;
    RefCntAutoPtr<IBuffer>                m_JointMatrixBuffer;
    RefCntAutoPtr<IBuffer>                m_SkinningConstantsCB;
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
    RTXPTSkinnedGeometryStats             m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create the HLSL skinning shader**

Create `assets/shaders/PathTracer/SkinnedVertexBuild.csh`:

```hlsl
#include "PathTracerShared.h"

cbuffer cbSkinningConstants
{
    uint g_SourceVertexBase;
    uint g_DestVertexBase;
    uint g_JointBase;
    uint g_VertexCount;
};

StructuredBuffer<GeometryVertexData> t_SourceVertices;
StructuredBuffer<SkinVertexData>     t_SourceSkinData;
StructuredBuffer<float4x4>           t_JointMatrices;
RWStructuredBuffer<GeometryVertexData> u_SkinnedVertices;

[numthreads(128, 1, 1)]
void main(uint3 dispatchThreadId : SV_DispatchThreadID)
{
    const uint localVertex = dispatchThreadId.x;
    if (localVertex >= g_VertexCount)
        return;

    const uint sourceIndex = g_SourceVertexBase + localVertex;
    const GeometryVertexData src = t_SourceVertices[sourceIndex];
    const SkinVertexData skin = t_SourceSkinData[sourceIndex];

    float4 skinnedPosition = float4(0.0, 0.0, 0.0, 0.0);
    float3 skinnedNormal = float3(0.0, 0.0, 0.0);

    [unroll]
    for (uint i = 0; i < 4; ++i)
    {
        const float weight = skin.weights[i];
        if (weight <= 0.0)
            continue;

        const uint jointIndex = g_JointBase + uint(skin.joints[i] + 0.5);
        const float4x4 jointMatrix = t_JointMatrices[jointIndex];
        skinnedPosition += mul(jointMatrix, float4(src.position, 1.0)) * weight;
        skinnedNormal += mul((float3x3)jointMatrix, src.normal) * weight;
    }

    GeometryVertexData dst;
    dst.position = skinnedPosition.xyz;
    const float normalLen = length(skinnedNormal);
    dst.normal = normalLen > 1e-6 ? skinnedNormal / normalLen : src.normal;
    dst.texCoord0 = src.texCoord0;

    u_SkinnedVertices[g_DestVertexBase + localVertex] = dst;
}
```

- [ ] **Step 3: Implement `Reset`, node table, and buffer creation**

In `RTXPTSkinnedGeometry.cpp`, start with includes and `Reset`:

```cpp
#include "RTXPTSkinnedGeometry.hpp"

#include <algorithm>

#include "GraphicsTypesX.hpp"
#include "MapHelper.hpp"

namespace Diligent
{

void RTXPTSkinnedGeometry::Reset()
{
    m_Nodes.clear();
    m_JointMatrices.clear();
    m_SourceVertexBuffer.Release();
    m_SourceSkinBuffer.Release();
    m_SkinnedVertexBuffer.Release();
    m_JointMatrixBuffer.Release();
    m_SkinningConstantsCB.Release();
    m_PSO.Release();
    m_SRB.Release();
    m_Stats = {};
}
```

Add `BuildNodeTable`:

```cpp
void RTXPTSkinnedGeometry::BuildNodeTable(const GLTF::Model& Model, Uint32 SceneIndex)
{
    if (SceneIndex >= Model.Scenes.size())
        return;

    const GLTF::Scene& Scene = Model.Scenes[SceneIndex];
    Uint32 VertexBase = 0;
    Uint32 JointBase = 0;
    const Uint32 SourceVertexBase = Model.GetBaseVertex();

    const Uint32 ModelVertexCount = Model.GetVertexBufferCount() > 0 ?
        static_cast<Uint32>(Model.GetVertexBuffer(0)->GetDesc().Size / sizeof(RTXPTGeometryVertex)) :
        0;

    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr || pNode->pSkin == nullptr)
            continue;

        const Uint32 JointCount = static_cast<Uint32>(pNode->pSkin->Joints.size());
        RTXPTSkinnedNodeGeometry Node;
        Node.pNode = pNode;
        Node.SourceVertexBase = SourceVertexBase;
        Node.VertexBase = VertexBase;
        Node.VertexCount = ModelVertexCount;
        Node.JointBase = JointBase;
        Node.JointCount = JointCount;
        m_Nodes.push_back(Node);

        VertexBase += ModelVertexCount;
        JointBase += JointCount;
    }

    m_Stats.SkinnedNodeCount = static_cast<Uint32>(m_Nodes.size());
    m_Stats.SkinnedVertexCount = VertexBase;
    m_Stats.JointMatrixCount = JointBase;
}
```

Then add `CreateBuffers`:

```cpp
bool RTXPTSkinnedGeometry::CreateBuffers(IRenderDevice* pDevice, IBuffer* pSourceVertexBuffer, IBuffer* pSourceSkinBuffer)
{
    const Uint32 VertexCountForBuffer = std::max<Uint32>(m_Stats.SkinnedVertexCount, 1);

    BufferDesc VertexDesc;
    VertexDesc.Name = "RTXPT current skinned vertex arena";
    VertexDesc.Usage = USAGE_DEFAULT;
    VertexDesc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RAY_TRACING;
    VertexDesc.Mode = BUFFER_MODE_STRUCTURED;
    VertexDesc.ElementByteStride = sizeof(RTXPTGeometryVertex);
    VertexDesc.Size = Uint64{VertexCountForBuffer} * sizeof(RTXPTGeometryVertex);
    pDevice->CreateBuffer(VertexDesc, nullptr, &m_SkinnedVertexBuffer);
    if (!m_SkinnedVertexBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT skinned vertex arena";
        return false;
    }

    if (m_Nodes.empty())
        return true;

    if (pSourceVertexBuffer == nullptr || pSourceSkinBuffer == nullptr)
    {
        m_Stats.LastError = "RTXPT skinned geometry requires source vertex and skin buffers";
        return false;
    }

    m_SourceVertexBuffer = pSourceVertexBuffer;
    m_SourceSkinBuffer = pSourceSkinBuffer;

    BufferDesc JointDesc;
    JointDesc.Name = "RTXPT skinned joint matrices";
    JointDesc.Usage = USAGE_DYNAMIC;
    JointDesc.BindFlags = BIND_SHADER_RESOURCE;
    JointDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    JointDesc.Mode = BUFFER_MODE_STRUCTURED;
    JointDesc.ElementByteStride = sizeof(float4x4);
    JointDesc.Size = Uint64{std::max<Uint32>(m_Stats.JointMatrixCount, 1)} * sizeof(float4x4);
    pDevice->CreateBuffer(JointDesc, nullptr, &m_JointMatrixBuffer);
    if (!m_JointMatrixBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT skinned joint matrix buffer";
        return false;
    }

    BufferDesc ConstantsDesc;
    ConstantsDesc.Name = "RTXPT skinning constants";
    ConstantsDesc.Usage = USAGE_DYNAMIC;
    ConstantsDesc.BindFlags = BIND_UNIFORM_BUFFER;
    ConstantsDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    ConstantsDesc.Size = sizeof(SkinningConstants);
    pDevice->CreateBuffer(ConstantsDesc, nullptr, &m_SkinningConstantsCB);
    if (!m_SkinningConstantsCB)
    {
        m_Stats.LastError = "Failed to create RTXPT skinning constants";
        return false;
    }

    return true;
}
```

- [ ] **Step 4: Implement compute pipeline creation**

Add `CreatePipeline`:

```cpp
bool RTXPTSkinnedGeometry::CreatePipeline(IRenderDevice* pDevice, IEngineFactory* pEngineFactory)
{
    if (m_Nodes.empty())
        return true;

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PathTracer", &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_COMPUTE;
    ShaderCI.Desc.Name = "RTXPT skinned vertex build";
    ShaderCI.SourceLanguage = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.FilePath = "PathTracer/SkinnedVertexBuild.csh";
    ShaderCI.EntryPoint = "main";
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pCS;
    pDevice->CreateShader(ShaderCI, &pCS);
    if (!pCS)
    {
        m_Stats.LastError = "Failed to create RTXPT skinned vertex build shader";
        return false;
    }

    ComputePipelineStateCreateInfo PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name = "RTXPT skinned vertex build PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_COMPUTE;
    PSOCreateInfo.pCS = pCS;

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_COMPUTE, "cbSkinningConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_SourceVertices", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_SourceSkinData", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_JointMatrices", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_SkinnedVertices", SHADER_RESOURCE_VARIABLE_TYPE_STATIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateComputePipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_Stats.LastError = "Failed to create RTXPT skinned vertex build PSO";
        return false;
    }

    auto SetStatic = [&](const char* Name, IDeviceObject* pObject) {
        IShaderResourceVariable* pVar = m_PSO->GetStaticVariableByName(SHADER_TYPE_COMPUTE, Name);
        if (pVar == nullptr || pObject == nullptr)
            return false;
        pVar->Set(pObject);
        return true;
    };

    const bool Bound =
        SetStatic("cbSkinningConstants", m_SkinningConstantsCB) &&
        SetStatic("t_SourceVertices", m_SourceVertexBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE)) &&
        SetStatic("t_SourceSkinData", m_SourceSkinBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE)) &&
        SetStatic("t_JointMatrices", m_JointMatrixBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE)) &&
        SetStatic("u_SkinnedVertices", m_SkinnedVertexBuffer->GetDefaultView(BUFFER_VIEW_UNORDERED_ACCESS));
    if (!Bound)
    {
        m_Stats.LastError = "Failed to bind RTXPT skinned vertex build resources";
        return false;
    }

    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_Stats.LastError = "Failed to create RTXPT skinned vertex build SRB";
        return false;
    }

    return true;
}
```

- [ ] **Step 5: Implement initialization and per-frame update**

Add `Initialize`, `UploadJointMatrices`, and `Update`:

```cpp
bool RTXPTSkinnedGeometry::Initialize(IRenderDevice*               pDevice,
                                      IEngineFactory*              pEngineFactory,
                                      const GLTF::Model&           Model,
                                      Uint32                       SceneIndex,
                                      IBuffer*                     pSourceVertexBuffer,
                                      IBuffer*                     pSourceSkinBuffer,
                                      bool                         ComputeSupported)
{
    Reset();

    BuildNodeTable(Model, SceneIndex);
    if (!CreateBuffers(pDevice, pSourceVertexBuffer, pSourceSkinBuffer))
        return false;

    if (m_Nodes.empty())
    {
        m_Stats.Ready = true;
        return true;
    }

    if (!ComputeSupported)
    {
        m_Stats.DisabledReason = "Skinned RTXPT geometry requires compute shaders";
        return false;
    }

    if (!CreatePipeline(pDevice, pEngineFactory))
        return false;

    m_Stats.Ready = true;
    return true;
}

void RTXPTSkinnedGeometry::UploadJointMatrices(IDeviceContext* pContext, const GLTF::ModelTransforms& Transforms)
{
    m_JointMatrices.assign(std::max<Uint32>(m_Stats.JointMatrixCount, 1), float4x4::Identity());

    for (const RTXPTSkinnedNodeGeometry& Node : m_Nodes)
    {
        if (Node.pNode == nullptr || Node.pNode->SkinTransformsIndex < 0)
            continue;

        const Uint32 SkinIndex = static_cast<Uint32>(Node.pNode->SkinTransformsIndex);
        if (SkinIndex >= Transforms.Skins.size())
            continue;

        const std::vector<float4x4>& Source = Transforms.Skins[SkinIndex].JointMatrices;
        const Uint32 Count = std::min<Uint32>(Node.JointCount, static_cast<Uint32>(Source.size()));
        for (Uint32 i = 0; i < Count; ++i)
            m_JointMatrices[Node.JointBase + i] = Source[i];
    }

    MapHelper<float4x4> Mapped{pContext, m_JointMatrixBuffer, MAP_WRITE, MAP_FLAG_DISCARD};
    std::copy(m_JointMatrices.begin(), m_JointMatrices.end(), Mapped.Ptr());
}

bool RTXPTSkinnedGeometry::Update(IDeviceContext* pContext, const GLTF::ModelTransforms& Transforms)
{
    m_Stats.LastDispatchExecuted = false;
    if (!IsReady() || m_Nodes.empty())
        return false;

    UploadJointMatrices(pContext, Transforms);

    pContext->SetPipelineState(m_PSO);

    for (const RTXPTSkinnedNodeGeometry& Node : m_Nodes)
    {
        SkinningConstants Constants;
        Constants.SourceVertexBase = Node.SourceVertexBase;
        Constants.DestVertexBase = Node.VertexBase;
        Constants.JointBase = Node.JointBase;
        Constants.VertexCount = Node.VertexCount;

        {
            MapHelper<SkinningConstants> Mapped{pContext, m_SkinningConstantsCB, MAP_WRITE, MAP_FLAG_DISCARD};
            *Mapped = Constants;
        }

        pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

        DispatchComputeAttribs DispatchAttribs;
        DispatchAttribs.ThreadGroupCountX = (Node.VertexCount + 127u) / 128u;
        DispatchAttribs.ThreadGroupCountY = 1;
        DispatchAttribs.ThreadGroupCountZ = 1;
        pContext->DispatchCompute(DispatchAttribs);
    }

    m_Stats.LastDispatchExecuted = true;
    ++m_Stats.DispatchCount;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 6: Register new files in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add:

```cmake
    src/RTXPTSkinnedGeometry.cpp
```

to `SOURCE`, add:

```cmake
    src/RTXPTSkinnedGeometry.hpp
```

to `INCLUDE`, and add:

```cmake
    assets/shaders/PathTracer/SkinnedVertexBuild.csh
```

to `SHADERS`.

- [ ] **Step 7: Verify shader and C++ registration**

Run only if requested:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected result: the new files compile and the new shader is visible in the RTXPT project.

- [ ] **Step 8: Commit Task 3 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp Samples/RTXPT/assets/shaders/PathTracer/SkinnedVertexBuild.csh Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add gpu skinned vertex arena" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Static/Skinned BLAS Build And Dynamic BLAS Update

Extends AS ownership so skinned geometry builds from the skinned vertex arena and can be updated after each skinning dispatch.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`

- [ ] **Step 1: Include skinned geometry and extend the public API**

In `RTXPTAccelerationStructures.hpp`, include the new header:

```cpp
#include "RTXPTSkinnedGeometry.hpp"
```

Rename the existing build method declaration to:

```cpp
    bool BuildScene(IRenderDevice*                 pDevice,
                    IDeviceContext*                pContext,
                    const GLTF::Model&             Model,
                    Uint32                         SceneIndex,
                    VALUE_TYPE                     IndexType,
                    const GLTF::ModelTransforms&   Transforms,
                    const RTXPTSkinnedGeometry*    pSkinnedGeometry,
                    bool                           RayTracingSupported);

    bool UpdateDynamicBLAS(IDeviceContext*             pContext,
                           const RTXPTSkinnedGeometry& SkinnedGeometry);
```

- [ ] **Step 2: Store BLAS build data for updateable records**

Replace `BLASRecord` with:

```cpp
    struct BLASRecord
    {
        std::string                         Name;
        RefCntAutoPtr<IBottomLevelAS>       BLAS;
        std::vector<std::string>            GeometryNames;
        std::vector<BLASBuildTriangleData>  TriangleData;
        Uint32                              GeometryCount = 0;
        bool                                Dynamic = false;
    };
```

The stored `GeometryNames` keep the `const char* GeometryName` pointers in `TriangleData` valid across update calls.

- [ ] **Step 3: Add a lookup helper for skinned nodes**

In `RTXPTAccelerationStructures.cpp`, add:

```cpp
const RTXPTSkinnedNodeGeometry* FindSkinnedNode(const RTXPTSkinnedGeometry* pSkinnedGeometry, const GLTF::Node* pNode)
{
    if (pSkinnedGeometry == nullptr || pNode == nullptr)
        return nullptr;

    for (const RTXPTSkinnedNodeGeometry& Node : pSkinnedGeometry->GetNodes())
    {
        if (Node.pNode == pNode)
            return &Node;
    }
    return nullptr;
}
```

- [ ] **Step 4: Convert `BuildStaticScene` into `BuildScene`**

Rename the definition and update its signature to match the header. Inside the node loop, add:

```cpp
        const RTXPTSkinnedNodeGeometry* pSkinnedNode = FindSkinnedNode(pSkinnedGeometry, pNode);
        const bool IsSkinnedNode = pSkinnedNode != nullptr && pSkinnedGeometry->GetSkinnedVertexBuffer() != nullptr;
        IBuffer* pNodeVertexBuffer = IsSkinnedNode ? pSkinnedGeometry->GetSkinnedVertexBuffer() : pVertexBuffer;
        const Uint32 NodeVertexStride = IsSkinnedNode ? static_cast<Uint32>(sizeof(RTXPTGeometryVertex)) : VertexStride;
        const Uint64 NodeModelVertexOffset = IsSkinnedNode ?
            Uint64{pSkinnedNode->VertexBase} * sizeof(RTXPTGeometryVertex) :
            ModelVertexOffset;
        const Uint32 NodeModelVertexCount = IsSkinnedNode ? pSkinnedNode->VertexCount : ModelVertexCount;
```

When building `SubEntry`, set the flag and offset:

```cpp
            if (IsSkinnedNode)
            {
                SubEntry.Flags |= kSubInstanceFlag_Skinned;
                SubEntry.VertexOffset = pSkinnedNode->VertexBase + Primitive.FirstVertex;
            }
            else
            {
                SubEntry.VertexOffset = BaseVertex + Primitive.FirstVertex;
            }
```

Then replace static `PrimitiveVertexOff`, `PrimitiveVertexCnt`, and build data buffer/stride/count values with node-aware values:

```cpp
            const Uint64 PrimitiveVertexOff = IsIndexed ?
                NodeModelVertexOffset :
                (IsSkinnedNode ?
                     (Uint64{pSkinnedNode->VertexBase + Primitive.FirstVertex} * sizeof(RTXPTGeometryVertex)) :
                     (Uint64{BaseVertex + Primitive.FirstVertex} * Uint64{VertexStride} + Uint64{Position.RelativeOffset}));
            const Uint32 PrimitiveVertexCnt = IsIndexed ? NodeModelVertexCount : Primitive.VertexCount;
```

and:

```cpp
            BuildData.pVertexBuffer        = pNodeVertexBuffer;
            BuildData.VertexOffset         = PrimitiveVertexOff;
            BuildData.VertexStride         = NodeVertexStride;
            BuildData.VertexCount          = PrimitiveVertexCnt;
```

- [ ] **Step 5: Create dynamic BLAS records with update support**

When filling `BLASRecord`, add:

```cpp
        Record.Dynamic = IsSkinnedNode;
        Record.GeometryNames = std::move(GeometryNames);
        Record.TriangleData = TriangleData;
        for (size_t i = 0; i < Record.TriangleData.size(); ++i)
            Record.TriangleData[i].GeometryName = Record.GeometryNames[i].c_str();
```

Set BLAS flags by node type:

```cpp
        BLASDesc.Flags = IsSkinnedNode ? RAYTRACING_BUILD_AS_ALLOW_UPDATE : RAYTRACING_BUILD_AS_NONE;
```

Use `Record.TriangleData` for the initial build:

```cpp
        BLASAttribs.pTriangleData = Record.TriangleData.data();
        BLASAttribs.TriangleDataCount = static_cast<Uint32>(Record.TriangleData.size());
```

When sizing `m_BLASScratch`, use both build and update sizes:

```cpp
        const Uint64 ScratchSize = std::max(Record.BLAS->GetScratchBufferSizes().Build,
                                            Record.BLAS->GetScratchBufferSizes().Update);
```

- [ ] **Step 6: Implement `UpdateDynamicBLAS`**

Add after `BuildScene`:

```cpp
bool RTXPTAccelerationStructures::UpdateDynamicBLAS(IDeviceContext* pContext, const RTXPTSkinnedGeometry& SkinnedGeometry)
{
    if (!IsBuilt() || !SkinnedGeometry.IsReady())
        return false;

    bool Updated = false;
    for (BLASRecord& Record : m_BLASRecords)
    {
        if (!Record.Dynamic || !Record.BLAS)
            continue;

        BuildBLASAttribs BLASAttribs;
        BLASAttribs.pBLAS = Record.BLAS;
        BLASAttribs.BLASTransitionMode = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        BLASAttribs.GeometryTransitionMode = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        BLASAttribs.pTriangleData = Record.TriangleData.data();
        BLASAttribs.TriangleDataCount = static_cast<Uint32>(Record.TriangleData.size());
        BLASAttribs.pScratchBuffer = m_BLASScratch;
        BLASAttribs.ScratchBufferTransitionMode = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        BLASAttribs.Update = true;
        pContext->BuildBLAS(BLASAttribs);
        Updated = true;
    }

    return Updated;
}
```

- [ ] **Step 7: Verify AS update path compiles**

Run only if requested:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected result: static scenes still build AS. Skinned scenes create dynamic BLAS records with `ALLOW_UPDATE` and can call `UpdateDynamicBLAS` after skinning dispatch.

- [ ] **Step 8: Commit Task 4 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
git commit -m "feat(rtxpt): build dynamic blas from skinned geometry" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Ray-Tracing Binding And Sample Lifecycle

Wires the skinned arena into `RTXPTRayTracingPass` and dispatches skinning before dynamic BLAS update and tracing.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Extend RT pass stats and signature**

In `RTXPTRayTracingPassStats`, add:

```cpp
    bool        SkinnedVertexBufferBound = false;
```

In `RTXPTRayTracingPass::Initialize`, add one parameter after `pVertexBuffer`:

```cpp
                    IBuffer*              pSkinnedVertexBuffer,
```

Update both declaration and definition.

- [ ] **Step 2: Register and bind `t_SkinnedVertexBuffer`**

In `RTXPTRayTracingPass.cpp`, add the resource variable alongside `t_VertexBuffer`:

```cpp
            .AddVariable(HitStages, "t_VertexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
            .AddVariable(HitStages, "t_SkinnedVertexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
            .AddVariable(HitStages, "t_IndexBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

Create the SRV:

```cpp
    IDeviceObject* pSkinnedVertexView =
        FullPathTracer && pSkinnedVertexBuffer != nullptr ? pSkinnedVertexBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
```

Bind it to closest-hit:

```cpp
        m_Stats.SkinnedVertexBufferBound = SetStatic(SHADER_TYPE_RAY_CLOSEST_HIT, "t_SkinnedVertexBuffer", pSkinnedVertexView, "skinned vertex buffer");
```

and to any-hit in the textured path:

```cpp
            m_Stats.SkinnedVertexBufferBound = m_Stats.SkinnedVertexBufferBound &&
                SetStatic(SHADER_TYPE_RAY_ANY_HIT, "t_SkinnedVertexBuffer", pSkinnedVertexView, "skinned vertex buffer");
```

Include it in the bridge failure condition:

```cpp
        !m_Stats.VertexBufferBound || !m_Stats.SkinnedVertexBufferBound || !m_Stats.IndexBufferBound)
```

- [ ] **Step 3: Add skinned geometry member to `RTXPTSample.hpp`**

Include the new header:

```cpp
#include "RTXPTSkinnedGeometry.hpp"
```

Add the member after `RTXPTAccelerationStructures`:

```cpp
    RTXPTSkinnedGeometry          m_SkinnedGeometry;
```

- [ ] **Step 4: Initialize skinning before AS build**

In `RTXPTSample::Initialize`, after light/material upload and before AS build, initialize and dispatch skinning once:

```cpp
        m_SkinnedGeometry.Initialize(m_pDevice,
                                     m_pEngineFactory,
                                     *pModel,
                                     m_Scene.GetSceneIndex(),
                                     m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
                                     m_Scene.GetSkinningBuffer(m_pDevice, m_pImmediateContext),
                                     m_FeatureCaps.ComputeShaders);

        if (m_SkinnedGeometry.HasSkinnedGeometry() && m_SkinnedGeometry.IsReady())
            m_SkinnedGeometry.Update(m_pImmediateContext, m_Scene.GetTransforms());

        m_AccelerationStructures.BuildScene(m_pDevice,
                                            m_pImmediateContext,
                                            *pModel,
                                            m_Scene.GetSceneIndex(),
                                            m_Scene.GetIndexType(),
                                            m_Scene.GetTransforms(),
                                            &m_SkinnedGeometry,
                                            m_FeatureCaps.RayTracing);
```

In the `else` branch where no model exists, add:

```cpp
        m_SkinnedGeometry.Reset();
```

- [ ] **Step 5: Pass the skinned buffer into RT pass initialization**

In both `m_RayTracingPass.Initialize` calls inside `CreatePhase4Passes`, pass:

```cpp
                                    m_SkinnedGeometry.GetSkinnedVertexBuffer(),
```

immediately after:

```cpp
                                    m_Scene.GetVertexBuffer0(m_pDevice, m_pImmediateContext),
```

Task 3 already creates a one-element skinned arena when the scene has no skinned nodes, so this pointer is always valid. If the RT pass still reports it missing, the failure is in task ordering or resource binding, not in the design.

- [ ] **Step 6: Dispatch skinning before dynamic BLAS update each frame**

In `RTXPTSample::Update`, after `m_Scene.Update(CurrTime, ElapsedTime);`, add:

```cpp
    if (m_Scene.IsGeometryDirty() && m_SkinnedGeometry.HasSkinnedGeometry() && m_SkinnedGeometry.IsReady())
    {
        const bool SkinningExecuted = m_SkinnedGeometry.Update(m_pImmediateContext, m_Scene.GetTransforms());
        const bool ASUpdated = SkinningExecuted && m_AccelerationStructures.UpdateDynamicBLAS(m_pImmediateContext, m_SkinnedGeometry);
        if (ASUpdated)
            RequestAccumulationReset("Skinned geometry updated");
        m_Scene.ClearGeometryDirty();
    }
```

This order is required: skinning dispatch first, BLAS update second, frame constants / trace later.

- [ ] **Step 7: Add UI status lines**

In the Scene section of `RTXPTSample::UpdateUI`, after primitive/material/light counts, add:

```cpp
        const RTXPTSceneGeometryStats& GeometryStats = m_Scene.GetGeometryStats();
        ImGui::Text("Animations: %s", GeometryStats.HasAnimations ? "yes" : "no");
        ImGui::Text("Skinned nodes: %u", GeometryStats.SkinnedNodeCount);
        ImGui::Text("Skinned primitives: %u", GeometryStats.SkinnedPrimitiveCount);
```

In Status / Debug, after `Vertex buffer`, add:

```cpp
        ImGui::Text("Skinned vertex buffer: %s", RTPassStats.SkinnedVertexBufferBound ? "bound" : "fallback");
        ImGui::Text("Skinning pass: %s", m_SkinnedGeometry.IsReady() ? "ready" : "not ready");
        if (!m_SkinnedGeometry.GetStats().DisabledReason.empty())
            ImGui::TextWrapped("Skinning disabled: %s", m_SkinnedGeometry.GetStats().DisabledReason.c_str());
        if (!m_SkinnedGeometry.GetStats().LastError.empty())
            ImGui::TextWrapped("Skinning error: %s", m_SkinnedGeometry.GetStats().LastError.c_str());
```

- [ ] **Step 8: Verify lifecycle ordering**

Run only if requested:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Manual check after build:

```powershell
build\x64\Debug\DiligentSamples\Samples\RTXPT\Debug\RTXPT.exe
```

Expected result: static Bistro still renders; status panel shows zero skinned nodes and no skinning error. On a skinned asset, status panel shows nonzero skinned nodes, skinning pass ready, skinned vertex buffer bound.

- [ ] **Step 9: Commit Task 5 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): wire skinned geometry into ray tracing" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Validation Hardening And Documentation

Closes the remaining validation gap and records the design divergence for future R2 emissive geometry work.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add validation to `RTXPTSkinnedGeometry::Initialize`**

After `BuildNodeTable`, if skinned nodes exist but source buffers are missing, fail with a precise error:

```cpp
    if (!m_Nodes.empty() && (pSourceVertexBuffer == nullptr || pSourceSkinBuffer == nullptr))
    {
        m_Stats.LastError = "Skinned GLTF nodes require buffer 0 POSITION/NORMAL/TEXCOORD_0 and buffer 1 JOINTS_0/WEIGHTS_0";
        return false;
    }
```

- [ ] **Step 2: Record the current-geometry design in mapping docs**

Append this section to `RTXPT_FORK_MAPPING.md`:

```markdown
## Skinned glTF Current Geometry

The Diligent RTXPT port uses a Diligent-native current-geometry path for skinned glTF:

- static primitives read the original GLTF vertex buffer 0
- skinned glTF node instances write current-frame vertices into `RTXPTSkinnedGeometry`
- `SubInstanceData::Flags & kSubInstanceFlag_Skinned` selects the skinned vertex arena in `PathTracerBridge.hlsli`
- skinned BLAS records update from that same arena before ray dispatch

This intentionally differs from RTXPT-fork's scene framework and keeps the invariant needed by emissive-triangle R2 work: BLAS, closest-hit fetch, and future emissive-triangle extraction must all consume the same current-frame GPU geometry. Bind-pose fallback is not allowed for skinned emissive meshes.
```

- [ ] **Step 3: Final targeted verification**

Run these only on explicit request:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected result: build succeeds.

Static smoke:

```powershell
build\x64\Debug\DiligentSamples\Samples\RTXPT\Debug\RTXPT.exe
```

Expected result: default Bistro renders as before; `Skinned nodes: 0`; `Skinned vertex buffer: bound`; no skinning errors.

Skinned asset smoke when a skinned glTF scene is available:

```powershell
build\x64\Debug\DiligentSamples\Samples\RTXPT\Debug\RTXPT.exe
```

Expected result: status panel shows nonzero skinned nodes; skinned geometry animates; accumulation resets when animation advances; no bind-pose fallback is used.

- [ ] **Step 4: Commit Task 6 inside `DiligentSamples`**

```powershell
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): document skinned current geometry path" -m "Co-Authored-By: GPT 5.5"
```

---

## Final Acceptance Checklist

- [ ] Static Bistro path still renders.
- [ ] Scene UI reports animation/skinning stats.
- [ ] `t_SkinnedVertexBuffer` is bound even when no skinned geometry exists.
- [ ] Skinned nodes write current-frame vertices into a GPU arena.
- [ ] Skinned BLAS updates after skinning dispatch and before ray tracing.
- [ ] Closest-hit and any-hit fetch static or skinned vertices through `Bridge::getGeometryVertex`.
- [ ] No CPU readback of skinned vertex data exists.
- [ ] No bind-pose fallback is used for skinned RT geometry.
- [ ] R2 emissive-triangle work can consume the same current-geometry contract.
