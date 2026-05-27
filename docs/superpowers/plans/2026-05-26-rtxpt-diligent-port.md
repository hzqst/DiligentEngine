# RTXPT Diligent Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a runnable Diligent sample shell for RTXPT and deliver the first minimal Phase 2 skeleton while preserving D3D12/Vulkan support and structured TODO tracking.

**Architecture:** Build a new `DiligentSamples/Samples/RTXPT` sample that plugs into the existing `add_sample_app` pattern, uses `SampleBase` lifecycle hooks, and keeps backend capability detection and feature gating isolated in small helper classes. The first plan only establishes the sample shell, capability/debug UI, copied assets directory, and the initial fallback path; it does not attempt to port the full RTX pipeline yet.

**Tech Stack:** C++, DiligentCore, DiligentTools/AssetLoader, DiligentSamples/SampleBase, HLSL, DXC/ShaderMake, D3D12, Vulkan, ImGui.

---

### Task 1: Add the RTXPT sample target and directory skeleton

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Create: `DiligentSamples/Samples/RTXPT/readme.md`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Create: `DiligentSamples/Samples/RTXPT/assets/README.md`
- Modify: `DiligentSamples/Samples/CMakeLists.txt`

- [ ] **Step 1: Add the failing integration point**

```cmake
# DiligentSamples/Samples/CMakeLists.txt
if(TARGET Diligent-AssetLoader AND TARGET DiligentFX)
    add_subdirectory(Atmosphere)
endif()

if(TARGET Diligent-AssetLoader)
    add_subdirectory(RTXPT)
endif()
```

- [ ] **Step 2: Add the minimal sample target**

```cmake
# DiligentSamples/Samples/RTXPT/CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

project(RTXPT CXX)

add_sample_app(RTXPT
    IDE_FOLDER
        DiligentSamples/Samples
    SOURCES
        src/RTXPTSample.cpp
    INCLUDES
        src/RTXPTSample.hpp
    ASSETS
        assets/README.md
    DXC_REQUIRED
        YES
)

target_link_libraries(RTXPT
PRIVATE
    Diligent-AssetLoader
)
```

- [ ] **Step 3: Add the sample readme files required by `add_sample_app`**

```markdown
<!-- DiligentSamples/Samples/RTXPT/readme.md -->
# RTXPT

RTXPT is a staged DiligentEngine port of NVIDIA RTXPT.

The first runnable version is a capability and fallback-view sample. Ray tracing, scene resources, shaders and post-processing are restored incrementally.
```

```markdown
<!-- DiligentSamples/Samples/RTXPT/assets/README.md -->
# RTXPT Assets

Runtime assets copied from `D:/RTXPT-fork/Assets` live here. The sample must run without depending on `D:/RTXPT-fork`.
```

- [ ] **Step 4: Add the minimal sample class**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
#pragma once

#include "SampleBase.hpp"

namespace Diligent
{

class RTXPTSample final : public SampleBase
{
public:
    virtual void Initialize(const SampleInitInfo& InitInfo) override final;
    virtual void Render() override final;
    virtual void Update(double CurrTime, double ElapsedTime, bool DoUpdateUI) override final;
    virtual void WindowResize(Uint32 Width, Uint32 Height) override final;
    virtual const Char* GetSampleName() const override final { return "RTXPT"; }

protected:
    virtual void UpdateUI() override final;
};

} // namespace Diligent
```

- [ ] **Step 5: Implement the sample entry point and a clear fallback render**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
#include "RTXPTSample.hpp"
#include "GraphicsAccessories.hpp"
#include "imgui.h"

namespace Diligent
{

SampleBase* CreateSample()
{
    return new RTXPTSample();
}

void RTXPTSample::Initialize(const SampleInitInfo& InitInfo)
{
    SampleBase::Initialize(InitInfo);
}

void RTXPTSample::Render()
{
    const auto ClearColor = float4{0.05f, 0.05f, 0.07f, 1.0f};
    auto*      pRTV       = m_pSwapChain->GetCurrentBackBufferRTV();
    m_pImmediateContext->ClearRenderTarget(pRTV, ClearColor.Data(), RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
}

void RTXPTSample::Update(double CurrTime, double ElapsedTime, bool DoUpdateUI)
{
    SampleBase::Update(CurrTime, ElapsedTime, DoUpdateUI);
}

void RTXPTSample::WindowResize(Uint32 Width, Uint32 Height)
{
}

void RTXPTSample::UpdateUI()
{
    ImGui::Begin("RTXPT Status");
    ImGui::Text("RTXPT sample shell");
    ImGui::Text("TODO(RTXPT-Port Phase 1): add full capability dashboard and backend-specific status.");
    ImGui::End();
}

} // namespace Diligent
```

- [ ] **Step 6: Verify the target appears in the sample tree**

Run: `rg -n "add_subdirectory\\(RTXPT\\)|add_sample_app\\(RTXPT" DiligentSamples/Samples`
Expected: both the new subdirectory and the sample target are listed.

- [ ] **Step 7: Commit the skeleton**

```bash
git add DiligentSamples/Samples/RTXPT DiligentSamples/Samples/CMakeLists.txt
git commit -m "feat(samples): add RTXPT sample shell"
```

---

### Task 2: Wire backend capability detection and the initial debug UI

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Write the capability data structure**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
struct RTXPTFeatureCaps
{
    bool RayTracing                  = false;
    bool StandaloneRayTracingShaders = false;
    bool RayQuery                    = false;
    bool BindlessResources           = false;
    bool ComputeShaders              = false;
    bool DXILCompiler                = false;
    bool SPIRVCompiler               = false;
};
```

Add this private member to `RTXPTSample`:

```cpp
RTXPTFeatureCaps m_FeatureCaps;
```

- [ ] **Step 2: Fill caps from the device and adapter info**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
static RTXPTFeatureCaps MakeFeatureCaps(const IRenderDevice* pDevice)
{
    RTXPTFeatureCaps Caps{};
    const auto& DevInfo  = pDevice->GetDeviceInfo();
    const auto& RTProps  = pDevice->GetAdapterInfo().RayTracing;
    Caps.RayTracing      = DevInfo.Features.RayTracing == DEVICE_FEATURE_STATE_ENABLED;
    Caps.BindlessResources = DevInfo.Features.BindlessResources == DEVICE_FEATURE_STATE_ENABLED;
    Caps.ComputeShaders  = DevInfo.Features.ComputeShaders == DEVICE_FEATURE_STATE_ENABLED;
    Caps.StandaloneRayTracingShaders = Caps.RayTracing &&
        (RTProps.CapFlags & RAY_TRACING_CAP_FLAG_STANDALONE_SHADERS) != 0;
    Caps.RayQuery = Caps.RayTracing &&
        (RTProps.CapFlags & RAY_TRACING_CAP_FLAG_INLINE_RAY_TRACING) != 0;
    // TODO(RTXPT-Port Phase 5): replace optimistic compiler flags with explicit DXC/ShaderMake availability checks.
    Caps.DXILCompiler = true;
    Caps.SPIRVCompiler = true;
    return Caps;
}
```

Call this from `Initialize` after `SampleBase::Initialize(InitInfo)`:

```cpp
m_FeatureCaps = MakeFeatureCaps(m_pDevice);
```

- [ ] **Step 3: Show the capability dashboard in the sample UI**

```cpp
void RTXPTSample::UpdateUI()
{
    ImGui::Begin("RTXPT Status");
    ImGui::Text("Backend: %s", GetRenderDeviceTypeString(m_pDevice->GetDeviceInfo().Type));
    ImGui::Text("RayTracing: %s", m_FeatureCaps.RayTracing ? "yes" : "no");
    ImGui::Text("Standalone RT shaders: %s", m_FeatureCaps.StandaloneRayTracingShaders ? "yes" : "no");
    ImGui::Text("RayQuery: %s", m_FeatureCaps.RayQuery ? "yes" : "no");
    ImGui::Text("Bindless: %s", m_FeatureCaps.BindlessResources ? "yes" : "no");
    ImGui::Text("Compute: %s", m_FeatureCaps.ComputeShaders ? "yes" : "no");
    ImGui::Text("TODO(RTXPT-Port Phase 1): add backend-specific warnings and fallback explanations.");
    ImGui::End();
}
```

- [ ] **Step 4: Verify the sample still launches when RayTracing is unavailable**

Run: launch the sample on a non-RT-capable backend or with RT disabled by configuration.
Expected: the sample opens, renders the fallback clear path, and displays disabled capability text instead of failing during initialization.

- [ ] **Step 5: Commit the capability work**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(samples): add RTXPT capability dashboard"
```

---

### Task 3: Add the first AssetLoader-backed scene bridge and copied asset closure

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Add: copied runtime assets under `DiligentSamples/Samples/RTXPT/assets`

- [ ] **Step 1: Define the scene bridge contract**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp
#pragma once

#include <memory>
#include <string>

#include "GLTFLoader.hpp"

namespace Diligent
{

class RTXPTScene
{
public:
    bool LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot);
    void Update(double CurrTime, double ElapsedTime);
    bool HasValidContent() const;

    const std::string& GetLoadedSceneName() const { return m_LoadedSceneName; }

private:
    std::unique_ptr<GLTF::Model> m_Model;
    std::string                  m_LoadedSceneName;
};

} // namespace Diligent
```

- [ ] **Step 2: Implement AssetLoader-based loading for the first default scene**

```cpp
// DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp
#include "RTXPTScene.hpp"

namespace Diligent
{

bool RTXPTScene::LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot)
{
    const std::string ModelPath = AssetsRoot + "/Models/Bistro/bistro.gltf";

    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;

    m_Model = std::make_unique<GLTF::Model>(pDevice, pContext, ModelCI);
    m_LoadedSceneName = "bistro-programmer-art.scene.json";

    // TODO(RTXPT-Port Phase 2): parse bistro-programmer-art.scene.json and merge RTXPT camera, material and light metadata.
    return m_Model != nullptr;
}

void RTXPTScene::Update(double CurrTime, double ElapsedTime)
{
}

bool RTXPTScene::HasValidContent() const
{
    return m_Model != nullptr;
}

} // namespace Diligent
```

- [ ] **Step 3: Copy the first default scene closure**

Copy the current RTXPT default scene and model closure with PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path 'DiligentSamples\Samples\RTXPT\assets\Models' | Out-Null
New-Item -ItemType Directory -Force -Path 'DiligentSamples\Samples\RTXPT\assets\Materials' | Out-Null
Copy-Item -LiteralPath 'D:\RTXPT-fork\Assets\bistro-programmer-art.scene.json' -Destination 'DiligentSamples\Samples\RTXPT\assets\bistro-programmer-art.scene.json' -Force
Copy-Item -LiteralPath 'D:\RTXPT-fork\Assets\Models\Bistro' -Destination 'DiligentSamples\Samples\RTXPT\assets\Models\Bistro' -Recurse -Force
Get-ChildItem -LiteralPath 'D:\RTXPT-fork\Assets\Materials' -Filter 'bistro.*.material.json' | Copy-Item -Destination 'DiligentSamples\Samples\RTXPT\assets\Materials' -Force
```

This keeps the first copied closure concrete: `bistro-programmer-art.scene.json`, `Models/Bistro`, and `Materials/bistro.*.material.json`.

- [ ] **Step 4: Show scene load status in the sample UI**

```cpp
ImGui::Text("Scene: %s", m_Scene.HasValidContent() ? "loaded" : "missing");
ImGui::Text("Scene file: %s", m_Scene.GetLoadedSceneName().c_str());
ImGui::Text("TODO(RTXPT-Port Phase 2): add full RTXPT scene/material/light metadata parsing.");
```

Add this private member to `RTXPTSample`:

```cpp
RTXPTScene m_Scene;
```

Call the loader from `Initialize`:

```cpp
const bool SceneLoaded = m_Scene.LoadDefaultScene(m_pDevice, m_pImmediateContext, ".");
if (!SceneLoaded)
{
    // TODO(RTXPT-Port Phase 2): report missing asset paths through the sample UI and keep the fallback clear path active.
}
```

- [ ] **Step 5: Verify the copied assets are included in the target**

Run: `Get-ChildItem DiligentSamples/Samples/RTXPT/assets -Recurse -File | Measure-Object`
Expected: the copied scene assets and at least the selected default scene file are present.

- [ ] **Step 6: Commit the first asset-backed scene bridge**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/assets
git commit -m "feat(samples): add RTXPT scene bridge"
```

---

### Task 4: Record the first structured TODO registry in code and plan the next phase gate

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Add structured TODO markers for every intentionally incomplete behavior**

```cpp
// TODO(RTXPT-Port Phase 2): add full material parsing for RTXPT extension fields.
// TODO(RTXPT-Port Phase 3): build BLAS/TLAS from scene geometry.
// TODO(RTXPT-Port Phase 4): add TraceRays path and RT PSO/SBT.
```

- [ ] **Step 2: Make sure fallback behavior is explicit**

```cpp
if (!m_FeatureCaps.RayTracing)
{
    // Fallback stays runnable until the RT path is wired in.
    return;
}
```

- [ ] **Step 3: Verify structured TODOs are searchable**

Run: `rg -n "TODO\\(RTXPT-Port" DiligentSamples/Samples/RTXPT`
Expected: every incomplete migration point is visible and grouped by phase.

- [ ] **Step 4: Commit the TODO policy plumbing**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp
git commit -m "docs(samples): record RTXPT port todo markers"
```

---

### Task 5: Prepare the next implementation checkpoint

**Files:**
- Update: `docs/superpowers/plans/2026-05-26-rtxpt-diligent-port.md` if task boundaries change

- [ ] **Step 1: Re-scan the plan for coverage gaps**

Check that the plan explicitly covers:
- sample shell and CMake integration
- backend capability detection
- first AssetLoader-backed scene bridge
- copied assets under `DiligentSamples/Samples/RTXPT/assets`
- structured TODO registry

- [ ] **Step 2: Confirm the next scope is the smallest runnable Phase 3 slice**

The next plan should be the smallest slice that can add one new visible capability without breaking the sample:

- minimal render targets and resource init
- then basic static geometry upload
- then BLAS/TLAS creation and a simple diagnostic render

- [ ] **Step 3: Hand off for implementation**

Run the plan task-by-task with a review checkpoint after each commit.
