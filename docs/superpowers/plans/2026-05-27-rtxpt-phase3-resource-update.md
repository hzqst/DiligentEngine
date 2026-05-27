# RTXPT Phase 3 Resource Update Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first RTXPT resource-update layer: frame constants, static scene material/light buffers, and static mesh BLAS/TLAS creation while preserving the current clear fallback.

**Architecture:** Keep `RTXPTSample` as the lifecycle and UI owner, keep `RTXPTScene` as the AssetLoader-backed CPU scene owner, and add focused resource managers for materials, lights, and acceleration structures. Phase 3 builds static BLAS/TLAS from the loaded glTF scene with full TLAS rebuild semantics; dynamic/skinned updates, compaction, OMM, SBT hit-group specialization, and TraceRays remain explicitly deferred to later phases.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentTools `GLTFLoader`, DiligentCore BLAS/TLAS APIs, Diligent buffer utilities, Dear ImGui.

---

## Phase 2 Completion Decision

Phase 2 is functionally complete against the runnable milestone in `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`:

- `Samples/RTXPT` launches and no longer depends on `D:/RTXPT-fork` at runtime.
- The default scene metadata and Bistro glTF path resolve from `DiligentSamples/Samples/RTXPT/assets`.
- The sample keeps the clear/debug fallback.
- Missing asset paths are reported in UI.
- `DiligentTools/AssetLoader` has the `MSFT_texture_dds` source support needed for the Bistro glTF closure.

Strict repository closure still has one preflight item: the top-level repository records modified `DiligentSamples` and `DiligentTools` submodule pointers. Phase 3 execution must commit those pointers before changing Phase 3 code.

---

## Scope

This plan implements the first Phase 3 runnable increment:

- Load glTF vertex and index buffers with `BIND_RAY_TRACING` so they are valid BLAS inputs.
- Expose loaded model, scene index, node transforms, and scene counts from `RTXPTScene`.
- Create and update a minimal frame-constants uniform buffer every frame.
- Upload minimal material and light structured buffers from AssetLoader data.
- Build static BLAS records and a TLAS when ray tracing is available.
- Show resource-update status in the existing RTXPT ImGui panel.
- Keep the current clear fallback for rendering.

This plan intentionally does not:

- Add a ray tracing PSO, SBT, shaders, or `IDeviceContext::TraceRays`.
- Add dynamic/skinned BLAS updates.
- Add AS compaction.
- Add OMM, RTXDI, NRD, DLSS, Streamline, or NVAPI.
- Replace the current clear fallback.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Add Phase 3 resource manager source/header files.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
  - Expose `GLTF::Model`, transforms, scene index, and resource counts.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
  - Load glTF GPU buffers with `BIND_RAY_TRACING`.
  - Compute default scene transforms and counts after loading.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Add frame constants, resource managers, and per-frame update helpers.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Create/update frame constants.
  - Upload static material/light buffers.
  - Build static acceleration structures when supported.
  - Expand UI status.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
  - Own material structured buffer and upload stats.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
  - Convert AssetLoader material attributes to a GPU structured buffer.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
  - Own light structured buffer and upload stats.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
  - Convert AssetLoader lights plus node transforms to GPU data.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
  - Own BLAS/TLAS handles, scratch buffers, instance buffer, and build stats.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
  - Build one static BLAS per mesh node and one TLAS instance per mesh node.

---

### Task 0: Repository Closure Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `DiligentTools`

- [ ] **Step 1: Confirm submodules are internally clean**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentTools status --short --branch
```

Expected:

```text
## RTXPT...upstream/RTXPT [ahead 5]
## RTXPT...upstream/RTXPT
```

No `M`, `A`, `D`, or `??` entries should appear inside either submodule.

- [ ] **Step 2: Confirm top-level pointer/doc state**

Run:

```powershell
git status --short --branch
```

Expected current preflight shape before committing Phase 3 implementation:

```text
## RTXPT...origin/RTXPT [ahead 2]
 M DiligentSamples
 M DiligentTools
?? docs/superpowers/plans/2026-05-27-rtxpt-phase2-closure.md
?? docs/superpowers/plans/2026-05-27-rtxpt-phase3-resource-update.md
```

If additional unrelated files appear, leave them unstaged.

- [ ] **Step 3: Commit the Phase 2/plan top-level closure**

Run:

```bash
git add DiligentSamples DiligentTools docs/superpowers/plans/2026-05-27-rtxpt-phase2-closure.md docs/superpowers/plans/2026-05-27-rtxpt-phase3-resource-update.md
git commit -m "docs(rtxpt): close phase 2 and plan phase 3" -m "Co-Authored-By: GPT 5.5"
```

Expected: a new top-level commit records the Phase 2 submodule pointers and both plan documents. If a plan document has already been committed, Git may omit it from this commit; that is acceptable as long as `git status --short` no longer reports the Phase 2 submodule pointer changes.

---

### Task 1: Make RTXPT Scene Data Ray-Tracing Ready

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Extend `RTXPTScene` accessors and state**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`, replace the class definition with:

```cpp
class RTXPTScene
{
public:
    bool LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot);
    void Update(double CurrTime, double ElapsedTime);
    bool HasValidContent() const;

    const std::string& GetLoadedSceneName() const { return m_LoadedSceneName; }
    const std::string& GetAssetsRoot() const { return m_AssetsRoot; }
    const std::string& GetModelPath() const { return m_ModelPath; }
    const std::string& GetLastError() const { return m_LastError; }

    const GLTF::Model*           GetModel() const { return m_Model.get(); }
    const GLTF::ModelTransforms& GetTransforms() const { return m_Transforms; }
    Uint32                       GetSceneIndex() const { return m_SceneIndex; }
    Uint32                       GetMeshNodeCount() const { return m_MeshNodeCount; }
    Uint32                       GetPrimitiveCount() const { return m_PrimitiveCount; }
    Uint32                       GetMaterialCount() const { return m_MaterialCount; }
    Uint32                       GetLightCount() const { return m_LightCount; }

private:
    void ResetLoadedData();
    void CacheSceneData();

    std::unique_ptr<GLTF::Model> m_Model;
    GLTF::ModelTransforms        m_Transforms;
    std::string                  m_LoadedSceneName;
    std::string                  m_AssetsRoot;
    std::string                  m_ModelPath;
    std::string                  m_LastError;
    Uint32                       m_SceneIndex      = 0;
    Uint32                       m_MeshNodeCount   = 0;
    Uint32                       m_PrimitiveCount  = 0;
    Uint32                       m_MaterialCount   = 0;
    Uint32                       m_LightCount      = 0;
};
```

- [ ] **Step 2: Add scene-count helpers**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`, add this helper block inside the anonymous namespace, after `JoinPath`:

```cpp
Uint32 CountMeshNodes(const GLTF::Scene& Scene)
{
    Uint32 Count = 0;
    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode != nullptr && pNode->pMesh != nullptr)
            ++Count;
    }
    return Count;
}

Uint32 CountPrimitives(const GLTF::Scene& Scene)
{
    Uint32 Count = 0;
    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr)
            continue;

        for (const GLTF::Primitive& Primitive : pNode->pMesh->Primitives)
        {
            if (Primitive.VertexCount != 0 || Primitive.IndexCount != 0)
                ++Count;
        }
    }
    return Count;
}

Uint32 CountLightNodes(const GLTF::Scene& Scene)
{
    Uint32 Count = 0;
    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode != nullptr && pNode->pLight != nullptr)
            ++Count;
    }
    return Count;
}
```

- [ ] **Step 3: Add reset/cache methods**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`, add these methods before `LoadDefaultScene`:

```cpp
void RTXPTScene::ResetLoadedData()
{
    m_Model.reset();
    m_Transforms = {};
    m_LoadedSceneName.clear();
    m_LastError.clear();
    m_SceneIndex     = 0;
    m_MeshNodeCount  = 0;
    m_PrimitiveCount = 0;
    m_MaterialCount  = 0;
    m_LightCount     = 0;
}

void RTXPTScene::CacheSceneData()
{
    if (!m_Model || m_Model->Scenes.empty())
        return;

    m_SceneIndex = static_cast<Uint32>(m_Model->DefaultSceneId >= 0 ? m_Model->DefaultSceneId : 0);
    if (m_SceneIndex >= m_Model->Scenes.size())
        m_SceneIndex = 0;

    m_Model->ComputeTransforms(m_SceneIndex, m_Transforms);

    const GLTF::Scene& Scene = m_Model->Scenes[m_SceneIndex];
    m_MeshNodeCount         = CountMeshNodes(Scene);
    m_PrimitiveCount        = CountPrimitives(Scene);
    m_MaterialCount         = static_cast<Uint32>(m_Model->Materials.size());
    m_LightCount            = CountLightNodes(Scene);
}
```

- [ ] **Step 4: Load glTF buffers with ray tracing bind flags**

In `RTXPTScene::LoadDefaultScene`, replace the initial reset lines:

```cpp
    m_Model.reset();
    m_LoadedSceneName.clear();
    m_LastError.clear();
```

with:

```cpp
    ResetLoadedData();
```

Then extend the `GLTF::ModelCreateInfo` setup:

```cpp
    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = m_ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;
    ModelCI.IndBufferBindFlags   = BIND_INDEX_BUFFER | BIND_RAY_TRACING;
    for (BIND_FLAGS& BindFlags : ModelCI.VertBufferBindFlags)
        BindFlags = BIND_VERTEX_BUFFER | BIND_RAY_TRACING;
```

After `m_LoadedSceneName = "bistro-programmer-art.scene.json";`, add:

```cpp
        CacheSceneData();
```

- [ ] **Step 5: Narrow the completed Phase 3 scene TODO**

In `RTXPTScene::LoadDefaultScene`, remove this line after acceleration structures are wired in later tasks:

```cpp
    // TODO(RTXPT-Port Phase 3): build BLAS/TLAS from scene geometry.
```

Keep the Phase 2 material-extension and Phase 4 TraceRays structured markers until those phases are completed.

- [ ] **Step 6: Verify scene accessors and RT bind flags**

Run:

```powershell
rg "GetModel|GetTransforms|GetMeshNodeCount|GetPrimitiveCount|BIND_RAY_TRACING|CacheSceneData" DiligentSamples/Samples/RTXPT/src/RTXPTScene.*
```

Expected: matches appear in `RTXPTScene.hpp` and `RTXPTScene.cpp`.

- [ ] **Step 7: Commit scene data preparation**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
git -C DiligentSamples commit -m "feat(samples): expose RTXPT scene update data" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Add Frame Constants Resource Update

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add frame constants and private helpers**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, add these includes:

```cpp
#include "Buffer.h"
#include "RefCntAutoPtr.hpp"
```

Add this struct before `class RTXPTSample`:

```cpp
struct RTXPTFrameConstants
{
    float4x4 ViewProj              = float4x4::Identity();
    float4x4 ViewProjInv           = float4x4::Identity();
    float4   CameraPosition_Time   = float4{0, 0, 0, 0};
    float4   ViewportSize_FrameIdx = float4{0, 0, 0, 0};
};
```

Add these private methods and members to `RTXPTSample`:

```cpp
private:
    void CreateFrameResources();
    void UpdateFrameConstants(double CurrTime);

    RTXPTFeatureCaps        m_FeatureCaps;
    std::string             m_AssetsRoot;
    RTXPTScene              m_Scene;
    RefCntAutoPtr<IBuffer>  m_FrameConstantsCB;
    RTXPTFrameConstants     m_LastFrameConstants;
    Uint32                  m_FrameIndex = 0;
```

- [ ] **Step 2: Include buffer helpers in the sample source**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, add:

```cpp
#include "GraphicsUtilities.h"
#include "MapHelper.hpp"
```

- [ ] **Step 3: Create the uniform buffer**

Add this method before `RTXPTSample::Initialize`:

```cpp
void RTXPTSample::CreateFrameResources()
{
    CreateUniformBuffer(m_pDevice, sizeof(RTXPTFrameConstants), "RTXPT frame constants", &m_FrameConstantsCB);
}
```

In `RTXPTSample::Initialize`, after `m_FeatureCaps = MakeFeatureCaps(m_pDevice);`, add:

```cpp
    CreateFrameResources();
```

- [ ] **Step 4: Update frame constants every frame**

Add this method before `RTXPTSample::Render`:

```cpp
void RTXPTSample::UpdateFrameConstants(double CurrTime)
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    const float          Width  = static_cast<float>(SCDesc.Width);
    const float          Height = static_cast<float>(SCDesc.Height);

    const float3  CameraPosition = float3{0.0f, 1.5f, -6.0f};
    const float4x4 CameraView    = float4x4::Translation(-CameraPosition.x, -CameraPosition.y, -CameraPosition.z);
    const float4x4 CameraProj    = GetAdjustedProjectionMatrix(PI_F / 4.0f, 0.1f, 10000.0f);
    const float4x4 ViewProj      = CameraView * CameraProj;

    m_LastFrameConstants.ViewProj              = ViewProj;
    m_LastFrameConstants.ViewProjInv           = ViewProj.Inverse();
    m_LastFrameConstants.CameraPosition_Time   = float4{CameraPosition.x, CameraPosition.y, CameraPosition.z, static_cast<float>(CurrTime)};
    m_LastFrameConstants.ViewportSize_FrameIdx = float4{Width, Height, Width > 0.0f ? 1.0f / Width : 0.0f, static_cast<float>(m_FrameIndex)};

    if (m_FrameConstantsCB)
    {
        MapHelper<RTXPTFrameConstants> Constants{m_pImmediateContext, m_FrameConstantsCB, MAP_WRITE, MAP_FLAG_DISCARD};
        *Constants = m_LastFrameConstants;
    }

    ++m_FrameIndex;
}
```

In `RTXPTSample::Update`, after `m_Scene.Update(CurrTime, ElapsedTime);`, add:

```cpp
    UpdateFrameConstants(CurrTime);
```

- [ ] **Step 5: Show frame constants status in UI**

In `RTXPTSample::UpdateUI`, after the asset diagnostics block, add:

```cpp
    ImGui::Separator();
    ImGui::Text("Frame constants: %s", m_FrameConstantsCB ? "created" : "missing");
    ImGui::Text("Frame index: %u", m_FrameIndex);
    ImGui::Text("Viewport: %.0f x %.0f", m_LastFrameConstants.ViewportSize_FrameIdx.x, m_LastFrameConstants.ViewportSize_FrameIdx.y);
```

- [ ] **Step 6: Verify frame update code**

Run:

```powershell
rg "RTXPTFrameConstants|CreateFrameResources|UpdateFrameConstants|Frame constants|Frame index" DiligentSamples/Samples/RTXPT/src
```

Expected: matches appear in `RTXPTSample.hpp` and `RTXPTSample.cpp`.

- [ ] **Step 7: Commit frame constants**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(samples): add RTXPT frame constants update" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Add Material And Light GPU Buffers

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Create `RTXPTMaterials.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`:

```cpp
#pragma once

#include <string>

#include "Buffer.h"
#include "GLTFLoader.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"

namespace Diligent
{

struct RTXPTMaterialStats
{
    Uint32      MaterialCount = 0;
    std::string LastError;
};

class RTXPTMaterials
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Model& Model);

    const RTXPTMaterialStats& GetStats() const { return m_Stats; }
    IBuffer*                  GetMaterialBuffer() const { return m_MaterialBuffer; }

private:
    RefCntAutoPtr<IBuffer> m_MaterialBuffer;
    RTXPTMaterialStats     m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create `RTXPTMaterials.cpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`:

```cpp
#include "RTXPTMaterials.hpp"

#include <vector>

namespace Diligent
{

void RTXPTMaterials::Reset()
{
    m_MaterialBuffer.Release();
    m_Stats = {};
}

bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const GLTF::Model& Model)
{
    Reset();

    m_Stats.MaterialCount = static_cast<Uint32>(Model.Materials.size());
    if (Model.Materials.empty())
        return true;

    std::vector<GLTF::Material::ShaderAttribs> Materials;
    Materials.reserve(Model.Materials.size());
    for (const GLTF::Material& Material : Model.Materials)
        Materials.emplace_back(Material.Attribs);

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

} // namespace Diligent
```

- [ ] **Step 3: Create `RTXPTLights.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`:

```cpp
#pragma once

#include <string>

#include "Buffer.h"
#include "GLTFLoader.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"

namespace Diligent
{

struct RTXPTLightData
{
    float4 ColorIntensity = float4{1, 1, 1, 0};
    float4 PositionRange  = float4{0, 0, 0, 0};
    float4 DirectionType  = float4{0, -1, 0, 0};
    float4 SpotAngles     = float4{0, 0, 0, 0};
};

struct RTXPTLightStats
{
    Uint32      LightCount = 0;
    std::string LastError;
};

class RTXPTLights
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Scene& Scene, const GLTF::ModelTransforms& Transforms);

    const RTXPTLightStats& GetStats() const { return m_Stats; }
    IBuffer*               GetLightBuffer() const { return m_LightBuffer; }

private:
    RefCntAutoPtr<IBuffer> m_LightBuffer;
    RTXPTLightStats        m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 4: Create `RTXPTLights.cpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`:

```cpp
#include "RTXPTLights.hpp"

#include <vector>

namespace Diligent
{

namespace
{

float LightTypeToShaderValue(GLTF::Light::TYPE Type)
{
    switch (Type)
    {
        case GLTF::Light::TYPE::DIRECTIONAL: return 0.0f;
        case GLTF::Light::TYPE::POINT:       return 1.0f;
        case GLTF::Light::TYPE::SPOT:        return 2.0f;
        default:                             return -1.0f;
    }
}

RTXPTLightData MakeLightData(const GLTF::Light& Light, const float4x4& NodeTransform)
{
    RTXPTLightData Data;
    Data.ColorIntensity = float4{Light.Color.x, Light.Color.y, Light.Color.z, Light.Intensity};
    Data.PositionRange  = float4{NodeTransform._41, NodeTransform._42, NodeTransform._43, Light.Range};
    Data.DirectionType  = float4{-NodeTransform._31, -NodeTransform._32, -NodeTransform._33, LightTypeToShaderValue(Light.Type)};
    Data.SpotAngles     = float4{Light.InnerConeAngle, Light.OuterConeAngle, 0.0f, 0.0f};
    return Data;
}

} // namespace

void RTXPTLights::Reset()
{
    m_LightBuffer.Release();
    m_Stats = {};
}

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
        return true;

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

} // namespace Diligent
```

- [ ] **Step 5: Add material/light files to CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, extend `SOURCE`:

```cmake
set(SOURCE
    src/RTXPTSample.cpp
    src/RTXPTScene.cpp
    src/RTXPTMaterials.cpp
    src/RTXPTLights.cpp
)
```

Extend `INCLUDE`:

```cmake
set(INCLUDE
    src/RTXPTSample.hpp
    src/RTXPTScene.hpp
    src/RTXPTMaterials.hpp
    src/RTXPTLights.hpp
)
```

- [ ] **Step 6: Wire material/light managers into `RTXPTSample`**

In `RTXPTSample.hpp`, include:

```cpp
#include "RTXPTLights.hpp"
#include "RTXPTMaterials.hpp"
```

Add private members after `m_Scene`:

```cpp
    RTXPTMaterials         m_Materials;
    RTXPTLights            m_Lights;
```

In `RTXPTSample::Initialize`, after `m_Scene.LoadDefaultScene(...)`, add:

```cpp
    if (const GLTF::Model* pModel = m_Scene.GetModel())
    {
        m_Materials.Upload(m_pDevice, *pModel);
        if (m_Scene.GetSceneIndex() < pModel->Scenes.size())
            m_Lights.Upload(m_pDevice, pModel->Scenes[m_Scene.GetSceneIndex()], m_Scene.GetTransforms());
    }
```

- [ ] **Step 7: Show material/light buffer status in UI**

In `RTXPTSample::UpdateUI`, after the scene diagnostics block, add:

```cpp
    ImGui::Text("Mesh nodes: %u", m_Scene.GetMeshNodeCount());
    ImGui::Text("Primitives: %u", m_Scene.GetPrimitiveCount());
    ImGui::Text("Materials: %u", m_Materials.GetStats().MaterialCount);
    ImGui::Text("Lights: %u", m_Lights.GetStats().LightCount);
    if (!m_Materials.GetStats().LastError.empty())
        ImGui::TextWrapped("Material buffer error: %s", m_Materials.GetStats().LastError.c_str());
    if (!m_Lights.GetStats().LastError.empty())
        ImGui::TextWrapped("Light buffer error: %s", m_Lights.GetStats().LastError.c_str());
```

- [ ] **Step 8: Verify material/light files and CMake wiring**

Run:

```powershell
rg "RTXPTMaterials|RTXPTLights|material buffer|light buffer" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: matches appear in the new material/light files, `RTXPTSample`, and `CMakeLists.txt`.

- [ ] **Step 9: Commit material/light buffers**

```bash
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp
git -C DiligentSamples commit -m "feat(samples): upload RTXPT scene resource buffers" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Add Static Acceleration Structure Manager

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create `RTXPTAccelerationStructures.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`:

```cpp
#pragma once

#include <string>
#include <vector>

#include "BottomLevelAS.h"
#include "Buffer.h"
#include "DeviceContext.h"
#include "GLTFLoader.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "TopLevelAS.h"

namespace Diligent
{

struct RTXPTAccelerationStructureStats
{
    bool        RayTracingSupported = false;
    bool        Built               = false;
    Uint32      GeometryCount       = 0;
    Uint32      InstanceCount       = 0;
    Uint32      BLASCount           = 0;
    Uint64      BLASScratchSize     = 0;
    Uint64      TLASScratchSize     = 0;
    std::string DisabledReason;
    std::string LastError;
};

class RTXPTAccelerationStructures
{
public:
    void Reset();

    bool BuildStaticScene(IRenderDevice*                 pDevice,
                          IDeviceContext*                pContext,
                          const GLTF::Model&             Model,
                          Uint32                         SceneIndex,
                          const GLTF::ModelTransforms&   Transforms,
                          bool                           RayTracingSupported);

    bool IsBuilt() const { return m_Stats.Built && m_TLAS; }

    ITopLevelAS* GetTLAS() const { return m_TLAS; }

    const RTXPTAccelerationStructureStats& GetStats() const { return m_Stats; }

private:
    struct BLASRecord
    {
        std::string                    Name;
        RefCntAutoPtr<IBottomLevelAS>  BLAS;
        Uint32                         GeometryCount = 0;
    };

    std::vector<BLASRecord>       m_BLASRecords;
    RefCntAutoPtr<ITopLevelAS>    m_TLAS;
    RefCntAutoPtr<IBuffer>        m_BLASScratch;
    RefCntAutoPtr<IBuffer>        m_TLASScratch;
    RefCntAutoPtr<IBuffer>        m_InstanceBuffer;
    RTXPTAccelerationStructureStats m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create acceleration-structure helper code**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp` with the includes and helper types:

```cpp
#include "RTXPTAccelerationStructures.hpp"

#include <algorithm>
#include <cstring>

namespace Diligent
{

namespace
{

struct PositionLayout
{
    bool       Valid          = false;
    Uint8      BufferId       = 0;
    VALUE_TYPE ValueType      = VT_UNDEFINED;
    Uint8      ComponentCount = 0;
    Uint32     RelativeOffset = 0;
};

PositionLayout FindPositionLayout(const GLTF::Model& Model)
{
    for (Uint32 Attr = 0; Attr < Model.GetNumVertexAttributes(); ++Attr)
    {
        const GLTF::VertexAttributeDesc& Desc = Model.GetVertexAttribute(Attr);
        if (Desc.Name != nullptr && std::strcmp(Desc.Name, GLTF::PositionAttributeName) == 0)
        {
            PositionLayout Layout;
            Layout.Valid          = true;
            Layout.BufferId       = Desc.BufferId;
            Layout.ValueType      = Desc.ValueType;
            Layout.ComponentCount = Desc.NumComponents;
            Layout.RelativeOffset = Desc.RelativeOffset == ~0U ? 0 : Desc.RelativeOffset;
            return Layout;
        }
    }
    return {};
}

VALUE_TYPE GetIndexType(const GLTF::Model& Model)
{
    if (Model.IndexData.IndexSize == sizeof(Uint16))
        return VT_UINT16;
    if (Model.IndexData.IndexSize == sizeof(Uint32))
        return VT_UINT32;
    return VT_UNDEFINED;
}

bool HasBindFlag(const IBuffer* pBuffer, BIND_FLAGS Flag)
{
    return pBuffer != nullptr && (pBuffer->GetDesc().BindFlags & Flag) != 0;
}

InstanceMatrix ToInstanceMatrix(const float4x4& Transform)
{
    InstanceMatrix Matrix;
    Matrix.SetRotation(&Transform._11, 4);
    Matrix.SetTranslation(Transform._41, Transform._42, Transform._43);
    return Matrix;
}

} // namespace
```

- [ ] **Step 3: Add reset and validation flow**

Append this implementation to `RTXPTAccelerationStructures.cpp`:

```cpp
void RTXPTAccelerationStructures::Reset()
{
    m_BLASRecords.clear();
    m_TLAS.Release();
    m_BLASScratch.Release();
    m_TLASScratch.Release();
    m_InstanceBuffer.Release();
    m_Stats = {};
}

bool RTXPTAccelerationStructures::BuildStaticScene(IRenderDevice*               pDevice,
                                                   IDeviceContext*              pContext,
                                                   const GLTF::Model&           Model,
                                                   Uint32                       SceneIndex,
                                                   const GLTF::ModelTransforms& Transforms,
                                                   bool                         RayTracingSupported)
{
    Reset();
    m_Stats.RayTracingSupported = RayTracingSupported;

    if (!RayTracingSupported)
    {
        m_Stats.DisabledReason = "Ray tracing is not supported by this device";
        return false;
    }

    if (SceneIndex >= Model.Scenes.size())
    {
        m_Stats.LastError = "Invalid RTXPT scene index for acceleration structure build";
        return false;
    }

    const PositionLayout Position = FindPositionLayout(Model);
    if (!Position.Valid || Position.ValueType != VT_FLOAT32 || Position.ComponentCount != 3)
    {
        m_Stats.LastError = "RTXPT BLAS build requires float3 POSITION vertex data";
        return false;
    }

    IBuffer* pVertexBuffer = Model.GetVertexBuffer(Position.BufferId, pDevice, pContext);
    if (!HasBindFlag(pVertexBuffer, BIND_RAY_TRACING))
    {
        m_Stats.LastError = "RTXPT POSITION vertex buffer is missing BIND_RAY_TRACING";
        return false;
    }

    IBuffer* pIndexBuffer = Model.GetIndexBuffer(pDevice, pContext);
    if (pIndexBuffer != nullptr && !HasBindFlag(pIndexBuffer, BIND_RAY_TRACING))
    {
        m_Stats.LastError = "RTXPT index buffer is missing BIND_RAY_TRACING";
        return false;
    }
```

- [ ] **Step 4: Build one BLAS per mesh node**

Append this code inside `BuildStaticScene`, directly after the validation block:

```cpp
    const GLTF::Scene& Scene       = Model.Scenes[SceneIndex];
    const VALUE_TYPE   IndexType   = GetIndexType(Model);
    const Uint32       VertexStride = Model.VertexData.Strides[Position.BufferId];
    const Uint32       BaseVertex   = Model.GetBaseVertex();
    const Uint32       FirstIndex   = pIndexBuffer != nullptr ? Model.GetFirstIndexLocation() : 0;

    if (pIndexBuffer != nullptr && IndexType == VT_UNDEFINED)
    {
        m_Stats.LastError = "RTXPT index buffer has an unsupported index size";
        return false;
    }

    std::vector<TLASBuildInstanceData> Instances;
    std::vector<std::string>           InstanceNames;
    Instances.reserve(Scene.LinearNodes.size());
    InstanceNames.reserve(Scene.LinearNodes.size());

    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr)
            continue;

        if (pNode->Index < 0 || static_cast<size_t>(pNode->Index) >= Transforms.NodeGlobalMatrices.size())
            continue;

        std::vector<std::string>          GeometryNames;
        std::vector<BLASTriangleDesc>     TriangleDescs;
        std::vector<BLASBuildTriangleData> TriangleData;

        GeometryNames.reserve(pNode->pMesh->Primitives.size());
        TriangleDescs.reserve(pNode->pMesh->Primitives.size());
        TriangleData.reserve(pNode->pMesh->Primitives.size());

        Uint32 PrimitiveIndex = 0;
        for (const GLTF::Primitive& Primitive : pNode->pMesh->Primitives)
        {
            if (Primitive.VertexCount == 0 && Primitive.IndexCount == 0)
                continue;

            GeometryNames.emplace_back((pNode->Name.empty() ? "RTXPTGeometry" : pNode->Name) + "_" + std::to_string(PrimitiveIndex));

            BLASTriangleDesc TriangleDesc;
            TriangleDesc.GeometryName         = GeometryNames.back().c_str();
            TriangleDesc.MaxVertexCount       = Primitive.VertexCount;
            TriangleDesc.VertexValueType      = Position.ValueType;
            TriangleDesc.VertexComponentCount = Position.ComponentCount;
            TriangleDesc.MaxPrimitiveCount    = Primitive.HasIndices() ? Primitive.IndexCount / 3 : Primitive.VertexCount / 3;
            TriangleDesc.IndexType            = Primitive.HasIndices() ? IndexType : VT_UNDEFINED;

            BLASBuildTriangleData BuildData;
            BuildData.GeometryName         = TriangleDesc.GeometryName;
            BuildData.pVertexBuffer        = pVertexBuffer;
            BuildData.VertexOffset         = (BaseVertex + Primitive.FirstVertex) * VertexStride + Position.RelativeOffset;
            BuildData.VertexStride         = VertexStride;
            BuildData.VertexCount          = Primitive.VertexCount;
            BuildData.VertexValueType      = Position.ValueType;
            BuildData.VertexComponentCount = Position.ComponentCount;
            BuildData.PrimitiveCount       = TriangleDesc.MaxPrimitiveCount;
            BuildData.Flags                = RAYTRACING_GEOMETRY_FLAG_OPAQUE;

            if (Primitive.HasIndices())
            {
                BuildData.pIndexBuffer = pIndexBuffer;
                BuildData.IndexOffset  = (FirstIndex + Primitive.FirstIndex) * Model.IndexData.IndexSize;
                BuildData.IndexType    = IndexType;
            }

            TriangleDescs.emplace_back(TriangleDesc);
            TriangleData.emplace_back(BuildData);
            ++PrimitiveIndex;
        }

        if (TriangleDescs.empty())
            continue;

        BLASRecord Record;
        Record.Name          = pNode->Name.empty() ? "RTXPT BLAS" : "RTXPT BLAS " + pNode->Name;
        Record.GeometryCount = static_cast<Uint32>(TriangleDescs.size());

        BottomLevelASDesc BLASDesc;
        BLASDesc.Name          = Record.Name.c_str();
        BLASDesc.Flags         = RAYTRACING_BUILD_AS_NONE;
        BLASDesc.pTriangles    = TriangleDescs.data();
        BLASDesc.TriangleCount = static_cast<Uint32>(TriangleDescs.size());
        pDevice->CreateBLAS(BLASDesc, &Record.BLAS);

        if (!Record.BLAS)
        {
            m_Stats.LastError = "Failed to create RTXPT BLAS";
            return false;
        }

        m_Stats.BLASScratchSize = std::max(m_Stats.BLASScratchSize, Record.BLAS->GetScratchBufferSizes().Build);

        if (!m_BLASScratch || m_BLASScratch->GetDesc().Size < Record.BLAS->GetScratchBufferSizes().Build)
        {
            BufferDesc ScratchDesc;
            ScratchDesc.Name      = "RTXPT BLAS scratch buffer";
            ScratchDesc.Usage     = USAGE_DEFAULT;
            ScratchDesc.BindFlags = BIND_RAY_TRACING;
            ScratchDesc.Size      = Record.BLAS->GetScratchBufferSizes().Build;
            pDevice->CreateBuffer(ScratchDesc, nullptr, &m_BLASScratch);
        }

        BuildBLASAttribs BLASAttribs;
        BLASAttribs.pBLAS                       = Record.BLAS;
        BLASAttribs.BLASTransitionMode          = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        BLASAttribs.GeometryTransitionMode      = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        BLASAttribs.pTriangleData               = TriangleData.data();
        BLASAttribs.TriangleDataCount           = static_cast<Uint32>(TriangleData.size());
        BLASAttribs.pScratchBuffer              = m_BLASScratch;
        BLASAttribs.ScratchBufferTransitionMode = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
        pContext->BuildBLAS(BLASAttribs);

        InstanceNames.emplace_back(Record.Name);

        TLASBuildInstanceData Instance;
        Instance.InstanceName = InstanceNames.back().c_str();
        Instance.pBLAS        = Record.BLAS;
        Instance.Transform    = ToInstanceMatrix(Transforms.NodeGlobalMatrices[pNode->Index]);
        Instance.CustomId     = static_cast<Uint32>(Instances.size());
        Instance.Flags        = RAYTRACING_INSTANCE_NONE;
        Instance.Mask         = 0xFF;
        Instances.emplace_back(Instance);

        m_Stats.GeometryCount += Record.GeometryCount;
        m_BLASRecords.emplace_back(std::move(Record));
    }
```

- [ ] **Step 5: Build the TLAS**

Append this code inside `BuildStaticScene`, after the BLAS loop:

```cpp
    if (Instances.empty())
    {
        m_Stats.LastError = "No RTXPT mesh instances were available for TLAS build";
        return false;
    }

    TopLevelASDesc TLASDesc;
    TLASDesc.Name             = "RTXPT TLAS";
    TLASDesc.MaxInstanceCount = static_cast<Uint32>(Instances.size());
    TLASDesc.Flags            = RAYTRACING_BUILD_AS_NONE;
    pDevice->CreateTLAS(TLASDesc, &m_TLAS);

    if (!m_TLAS)
    {
        m_Stats.LastError = "Failed to create RTXPT TLAS";
        return false;
    }

    m_Stats.TLASScratchSize = m_TLAS->GetScratchBufferSizes().Build;

    BufferDesc InstanceBufferDesc;
    InstanceBufferDesc.Name      = "RTXPT TLAS instance buffer";
    InstanceBufferDesc.Usage     = USAGE_DEFAULT;
    InstanceBufferDesc.BindFlags = BIND_RAY_TRACING;
    InstanceBufferDesc.Size      = TLAS_INSTANCE_DATA_SIZE * Instances.size();
    pDevice->CreateBuffer(InstanceBufferDesc, nullptr, &m_InstanceBuffer);

    BufferDesc TLASScratchDesc;
    TLASScratchDesc.Name      = "RTXPT TLAS scratch buffer";
    TLASScratchDesc.Usage     = USAGE_DEFAULT;
    TLASScratchDesc.BindFlags = BIND_RAY_TRACING;
    TLASScratchDesc.Size      = m_TLAS->GetScratchBufferSizes().Build;
    pDevice->CreateBuffer(TLASScratchDesc, nullptr, &m_TLASScratch);

    BuildTLASAttribs TLASAttribs;
    TLASAttribs.pTLAS                        = m_TLAS;
    TLASAttribs.TLASTransitionMode           = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
    TLASAttribs.BLASTransitionMode           = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
    TLASAttribs.pInstances                   = Instances.data();
    TLASAttribs.InstanceCount                = static_cast<Uint32>(Instances.size());
    TLASAttribs.pInstanceBuffer              = m_InstanceBuffer;
    TLASAttribs.InstanceBufferTransitionMode = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
    TLASAttribs.HitGroupStride               = 1;
    TLASAttribs.BindingMode                  = HIT_GROUP_BINDING_MODE_PER_GEOMETRY;
    TLASAttribs.pScratchBuffer               = m_TLASScratch;
    TLASAttribs.ScratchBufferTransitionMode  = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;
    pContext->BuildTLAS(TLASAttribs);

    m_Stats.BLASCount     = static_cast<Uint32>(m_BLASRecords.size());
    m_Stats.InstanceCount = static_cast<Uint32>(Instances.size());
    m_Stats.Built         = true;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 6: Add AS files to CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, extend `SOURCE`:

```cmake
set(SOURCE
    src/RTXPTSample.cpp
    src/RTXPTScene.cpp
    src/RTXPTMaterials.cpp
    src/RTXPTLights.cpp
    src/RTXPTAccelerationStructures.cpp
)
```

Extend `INCLUDE`:

```cmake
set(INCLUDE
    src/RTXPTSample.hpp
    src/RTXPTScene.hpp
    src/RTXPTMaterials.hpp
    src/RTXPTLights.hpp
    src/RTXPTAccelerationStructures.hpp
)
```

- [ ] **Step 7: Verify AS manager references**

Run:

```powershell
rg "RTXPTAccelerationStructures|BuildStaticScene|CreateBLAS|BuildBLAS|CreateTLAS|BuildTLAS|TLAS_INSTANCE_DATA_SIZE" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: matches appear in the new AS files and `CMakeLists.txt`.

- [ ] **Step 8: Commit acceleration structure manager**

```bash
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp
git -C DiligentSamples commit -m "feat(samples): add RTXPT static acceleration structures" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Wire Acceleration Structures Into RTXPT Sample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Add AS manager to `RTXPTSample`**

In `RTXPTSample.hpp`, include:

```cpp
#include "RTXPTAccelerationStructures.hpp"
```

Add a private member after `m_Lights`:

```cpp
    RTXPTAccelerationStructures m_AccelerationStructures;
```

- [ ] **Step 2: Build AS after scene resource upload**

In `RTXPTSample::Initialize`, inside the existing `if (const GLTF::Model* pModel = m_Scene.GetModel())` block, after material/light uploads, add:

```cpp
        m_AccelerationStructures.BuildStaticScene(m_pDevice,
                                                  m_pImmediateContext,
                                                  *pModel,
                                                  m_Scene.GetSceneIndex(),
                                                  m_Scene.GetTransforms(),
                                                  m_FeatureCaps.RayTracing);
```

If the scene is not loaded, add this `else` branch:

```cpp
    else
    {
        m_AccelerationStructures.Reset();
    }
```

- [ ] **Step 3: Show AS status in UI**

In `RTXPTSample::UpdateUI`, after the material/light status, add:

```cpp
    const RTXPTAccelerationStructureStats& ASStats = m_AccelerationStructures.GetStats();
    ImGui::Separator();
    ImGui::Text("Acceleration structures: %s", m_AccelerationStructures.IsBuilt() ? "built" : "not built");
    ImGui::Text("BLAS: %u", ASStats.BLASCount);
    ImGui::Text("TLAS instances: %u", ASStats.InstanceCount);
    ImGui::Text("RT geometries: %u", ASStats.GeometryCount);
    if (!ASStats.DisabledReason.empty())
        ImGui::TextWrapped("AS disabled: %s", ASStats.DisabledReason.c_str());
    if (!ASStats.LastError.empty())
        ImGui::TextWrapped("AS error: %s", ASStats.LastError.c_str());
```

- [ ] **Step 4: Remove the completed Phase 3 scene marker**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`, remove:

```cpp
    // TODO(RTXPT-Port Phase 3): build BLAS/TLAS from scene geometry.
```

Add this narrower structured marker after the Phase 2 material marker:

```cpp
    // TODO(RTXPT-Port Phase 3): add dynamic/skinned BLAS update, AS compaction, and alpha/OMM geometry flags; current path builds static opaque geometry.
```

- [ ] **Step 5: Verify Phase 3 AS wiring**

Run:

```powershell
rg "m_AccelerationStructures|Acceleration structures|AS disabled|AS error|dynamic/skinned BLAS update" DiligentSamples/Samples/RTXPT/src
```

Expected: matches appear in `RTXPTSample.hpp`, `RTXPTSample.cpp`, and `RTXPTScene.cpp`.

- [ ] **Step 6: Commit AS sample wiring**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTScene.cpp
git -C DiligentSamples commit -m "feat(samples): build RTXPT scene acceleration structures" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Phase 3 Verification And Handoff

**Files:**
- Verify only unless a command reveals an issue.

- [ ] **Step 1: Static source verification**

Run:

```powershell
rg "BIND_RAY_TRACING" DiligentSamples/Samples/RTXPT/src
rg "RTXPTFrameConstants|RTXPTMaterials|RTXPTLights|RTXPTAccelerationStructures" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
rg "TODO\\(RTXPT-Port Phase 3\\)" DiligentSamples/Samples/RTXPT/src
```

Expected:

```text
First command: matches in RTXPTScene.cpp and RTXPTAccelerationStructures.cpp
Second command: matches in RTXPTSample, new manager files, and CMakeLists.txt
Third command: only the narrowed dynamic/skinned/compaction/alpha/OMM marker remains
```

- [ ] **Step 2: Confirm submodule status**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentTools status --short --branch
git status --short --branch
```

Expected:

```text
DiligentSamples: RTXPT branch ahead of upstream by the new Phase 3 commits, with no uncommitted files
DiligentTools: clean
Top-level: modified DiligentSamples submodule pointer only, unless docs were intentionally changed
```

- [ ] **Step 3: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run the configured RTXPT target:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: command exits with code 0. If this build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 4: Optional runtime verification when the user explicitly requests it**

If the user asks for runtime verification, launch `Samples/RTXPT` once with D3D12 and once with Vulkan on an RT-capable machine. Expected UI facts:

```text
Scene: loaded
Frame constants: created
Materials: non-zero for Bistro
Acceleration structures: built
BLAS: non-zero
TLAS instances: non-zero
RT geometries: non-zero
```

If ray tracing is unavailable, expected UI facts:

```text
Scene: loaded
Frame constants: created
Acceleration structures: not built
AS disabled: Ray tracing is not supported by this device
```

- [ ] **Step 5: Commit top-level Phase 3 submodule pointer**

After `DiligentSamples` Phase 3 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples
git commit -m "feat(samples): add RTXPT phase 3 resource update" -m "Co-Authored-By: GPT 5.5"
```

Expected: top-level commit records the updated `DiligentSamples` submodule pointer.

---

## Self-Review Checklist

- [ ] The plan directly follows Phase 3 of `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`.
- [ ] The plan preserves the current runnable clear fallback.
- [ ] The plan makes AssetLoader-created glTF buffers valid for Diligent BLAS input with `BIND_RAY_TRACING`.
- [ ] The plan creates frame constants, material buffers, light buffers, BLAS, and TLAS.
- [ ] The plan explicitly handles the no-ray-tracing fallback path.
- [ ] The plan keeps TraceRays, RT PSO/SBT, shader porting, OMM, AS compaction, and dynamic/skinned updates out of this runnable increment.
- [ ] Every new source/header file is added to `DiligentSamples/Samples/RTXPT/CMakeLists.txt`.
- [ ] Verification commands avoid build/runtime execution unless the user explicitly asks for it.
- [ ] Commit commands use the required `Co-Authored-By: GPT 5.5` trailer.
