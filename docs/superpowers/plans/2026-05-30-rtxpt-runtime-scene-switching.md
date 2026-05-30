# RTXPT Runtime Scene Switching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add runtime scene switching to the Diligent RTXPT sample by enumerating local `*.scene.json` files at startup, showing them in the ImGui Scene combo, and fully reloading scene-dependent resources when the user selects another scene.

**Architecture:** Keep `RTXPTSample` as the UI, selection, and GPU-resource orchestration owner. Keep `RTXPTScene` as the current loaded scene owner, but generalize it from a hard-coded default scene loader to a selected scene-file loader that reads `models[0]` and scene cameras. Runtime switching is a full scene-dependent resource rebuild, not an incremental swap.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentTools `GLTFLoader`, Diligent platform `FileSystem`, Dear ImGui, `nlohmann::json`, `std::filesystem`.

---

## Context You Need Before Starting

This plan implements `docs/superpowers/specs/2026-05-30-rtxpt-runtime-scene-switching-design.md`.

The current Diligent RTXPT sample has these relevant facts:

- `RTXPTScene::LoadDefaultScene(...)` hard-codes `bistro-programmer-art.scene.json` and `Models/Bistro/bistro.gltf`.
- `RTXPTScene::LoadSceneCameras(...)` already parses RTXPT scene JSON camera nodes and animated cameras.
- `RTXPTSample::Initialize(...)` loads one scene, uploads materials/lights, builds static acceleration structures, initializes passes, and applies scene camera 0.
- `RTXPTSample::UpdateUI(...)` already has a `Scene` section with a scene camera combo and diagnostics.
- `RTXPTSample::CreatePhase4Passes()` rebuilds the ray tracing pass from the current material, light, AS, scene vertex/index, and texture bindings.

`DiligentSamples` is a git submodule. Source edits in this plan are committed inside `DiligentSamples/`, not in the umbrella repo. The umbrella repo should only receive a submodule pointer update if explicitly requested after implementation.

## File Structure

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
  - Add `LoadScene(...)`.
  - Keep `LoadDefaultScene(...)` as a compatibility wrapper.

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
  - Parse the selected `.scene.json`.
  - Read `models[0]` as the first-stage model path.
  - Clear stale model path diagnostics on reset.
  - Reuse existing GLTF model creation and camera parsing.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Add available-scene list and current scene name.
  - Add scene enumeration, selection, reset, and rebuild helpers.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Add direct-child `*.scene.json` enumeration.
  - Relax asset-root detection so it does not require Bistro specifically.
  - Replace initialization's one-off load block with the shared `SetCurrentScene(...)` path.
  - Add the Scene combo and empty-list diagnostics.

No shader, CMake, or asset file changes are expected.

## Cross-Cutting Contracts

- Scene file names stored in `m_AvailableScenes` and `m_CurrentSceneName` are short file names, for example `kitchen.scene.json`.
- `RTXPTScene::GetLoadedSceneName()` remains empty after failed loads; `RTXPTSample::m_CurrentSceneName` records the user's selected scene.
- Scene cameras are optional. A scene with zero cameras still loads and resets the sample to the default free camera.
- GPU resource rebuilds are all orchestrated by `RTXPTSample`.
- Failed scene loads do not preserve old scene-dependent GPU resources.
- The first implementation uses only `models[0]`. Full scene graph composition stays outside this plan.

---

### Task 1: Generalize `RTXPTScene` To Load A Selected Scene File

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Add the public selected-scene loader declaration**

In `RTXPTScene.hpp`, replace the current first public method:

```cpp
    bool LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot);
```

with:

```cpp
    bool LoadScene(IRenderDevice*        pDevice,
                   IDeviceContext*      pContext,
                   const std::string&   AssetsRoot,
                   const std::string&   SceneName);
    bool LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot);
```

- [ ] **Step 2: Add a scene model-path parser helper**

In `RTXPTScene.cpp`, add this helper inside the anonymous namespace after `JoinPath(...)`:

```cpp
bool ReadSceneModelPath(const std::string& ScenePath, std::string& ModelRelativePath, std::string& Error)
{
    std::ifstream SceneFile{ScenePath};
    if (!SceneFile)
    {
        Error = "Unable to open scene file: " + ScenePath;
        return false;
    }

    nlohmann::json SceneJson = nlohmann::json::parse(SceneFile, nullptr, false);
    if (SceneJson.is_discarded() || !SceneJson.is_object())
    {
        Error = "Invalid scene JSON: " + ScenePath;
        return false;
    }

    const auto ModelsIt = SceneJson.find("models");
    if (ModelsIt == SceneJson.end() || !ModelsIt->is_array() || ModelsIt->empty())
    {
        Error = "Scene JSON does not contain a non-empty models array: " + ScenePath;
        return false;
    }

    const nlohmann::json& FirstModel = (*ModelsIt)[0];
    if (!FirstModel.is_string())
    {
        Error = "Scene JSON models[0] is not a string: " + ScenePath;
        return false;
    }

    ModelRelativePath = FirstModel.get<std::string>();
    if (ModelRelativePath.empty())
    {
        Error = "Scene JSON models[0] is empty: " + ScenePath;
        return false;
    }

    return true;
}
```

- [ ] **Step 3: Clear stale model diagnostics on reset**

In `RTXPTScene::ResetLoadedData()`, add `m_ModelPath.clear();` after `m_LoadedSceneName.clear();`:

```cpp
    m_Cameras.clear();
    m_LoadedSceneName.clear();
    m_ModelPath.clear();
    m_LastError.clear();
```

- [ ] **Step 4: Add `RTXPTScene::LoadScene(...)`**

In `RTXPTScene.cpp`, replace the current `LoadDefaultScene(...)` body with a new `LoadScene(...)` implementation followed by the wrapper below:

```cpp
bool RTXPTScene::LoadScene(IRenderDevice*      pDevice,
                           IDeviceContext*    pContext,
                           const std::string& AssetsRoot,
                           const std::string& SceneName)
{
    ResetLoadedData();

    m_AssetsRoot = FileSystem::SimplifyPath(AssetsRoot.c_str());
    if (SceneName.empty())
    {
        m_LastError = "Empty RTXPT scene file name";
        return false;
    }

    const std::string ScenePath = JoinPath(m_AssetsRoot, SceneName.c_str());
    if (!FileSystem::FileExists(ScenePath.c_str()))
    {
        m_LastError = "Missing scene file: " + ScenePath;
        return false;
    }

    std::string ModelRelativePath;
    if (!ReadSceneModelPath(ScenePath, ModelRelativePath, m_LastError))
        return false;

    m_ModelPath = JoinPath(m_AssetsRoot, ModelRelativePath.c_str());
    if (!FileSystem::FileExists(m_ModelPath.c_str()))
    {
        m_LastError = "Missing glTF file: " + m_ModelPath;
        return false;
    }

    LoadSceneCameras(ScenePath);

    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = m_ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;
    ModelCI.IndexType            = m_IndexType;
    ModelCI.IndBufferBindFlags   = BIND_INDEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;
    for (BIND_FLAGS& BindFlags : ModelCI.VertBufferBindFlags)
        BindFlags = BIND_VERTEX_BUFFER | BIND_RAY_TRACING;
    // Buffer 0 is the path-tracer vertex stream (POSITION + NORMAL + TEXCOORD_0); chit reads it as a StructuredBuffer<GeometryVertexData>.
    ModelCI.VertBufferBindFlags[0] = BIND_VERTEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;

    try
    {
        m_Model           = std::make_unique<GLTF::Model>(pDevice, pContext, ModelCI);
        m_LoadedSceneName = SceneName;
        CacheSceneData();
    }
    catch (const std::exception& e)
    {
        m_Model.reset();
        m_LoadedSceneName.clear();
        m_LastError = e.what();
    }

    return m_Model != nullptr;
}

bool RTXPTScene::LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot)
{
    return LoadScene(pDevice, pContext, AssetsRoot, "bistro-programmer-art.scene.json");
}
```

- [ ] **Step 5: Verify the scene loader diff**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
rg -n "LoadScene|LoadDefaultScene|ReadSceneModelPath|m_ModelPath.clear" DiligentSamples/Samples/RTXPT/src/RTXPTScene.*
```

Expected:

- `git diff --check` exits with code 0.
- `rg` shows the new `LoadScene(...)`, wrapper `LoadDefaultScene(...)`, parser helper, and reset cleanup.

- [ ] **Step 6: Commit Task 1 in the submodule**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
git -C DiligentSamples commit -m "feat(rtxpt): load selected scene files" -m "Co-Authored-By: GPT 5.5"
```

Expected: one submodule commit containing only `RTXPTScene.hpp/.cpp`.

---

### Task 2: Add Scene Enumeration And Shared Scene-Rebuild Lifecycle

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add `std::vector` to the sample header**

In `RTXPTSample.hpp`, change the standard includes from:

```cpp
#include <string>
```

to:

```cpp
#include <string>
#include <vector>
```

- [ ] **Step 2: Add scene lifecycle helper declarations and state**

In `RTXPTSample.hpp`, add these private methods after `CreateFrameResources();`:

```cpp
    void EnumerateAvailableScenes();
    bool SetCurrentScene(const std::string& SceneName, bool ForceReload = false);
    void ResetSceneDependentResources();
    bool RebuildSceneDependentResources();
```

Add these members after `std::string m_AssetsRoot;`:

```cpp
    std::vector<std::string>    m_AvailableScenes;
    std::string                 m_CurrentSceneName;
```

- [ ] **Step 3: Add the required sample source includes**

In `RTXPTSample.cpp`, add these standard includes after the ImGui include block:

```cpp
#include <algorithm>
#include <filesystem>
```

- [ ] **Step 4: Add preferred-scene constants and direct-child scene enumeration helper**

In the anonymous namespace of `RTXPTSample.cpp`, add these constants after the current clip-plane constants:

```cpp
constexpr const char* kPreferredSceneName = "bistro-programmer-art.scene.json";
constexpr const char* kSceneFileSuffix    = ".scene.json";
```

Add these helpers after `JoinPath(...)` and before `IsRTXPTAssetsRoot(...)`:

```cpp
bool EndsWith(const std::string& Text, const char* Suffix)
{
    const std::string SuffixString{Suffix};
    return Text.size() >= SuffixString.size() &&
        Text.compare(Text.size() - SuffixString.size(), SuffixString.size(), SuffixString) == 0;
}

std::vector<std::string> EnumerateSceneFiles(const std::string& AssetsRoot)
{
    std::vector<std::string> SceneFiles;

    std::error_code Error;
    const std::filesystem::path RootPath{AssetsRoot};
    if (!std::filesystem::is_directory(RootPath, Error))
        return SceneFiles;

    std::filesystem::directory_iterator It{RootPath, Error};
    const std::filesystem::directory_iterator End;
    while (!Error && It != End)
    {
        const std::filesystem::directory_entry& Entry = *It;
        std::error_code                         StatusError;
        if (Entry.is_regular_file(StatusError))
        {
            const std::string FileName = Entry.path().filename().string();
            if (EndsWith(FileName, kSceneFileSuffix))
                SceneFiles.push_back(FileName);
        }
        It.increment(Error);
    }

    std::sort(SceneFiles.begin(), SceneFiles.end());
    return SceneFiles;
}
```

- [ ] **Step 5: Relax asset-root detection to scene-file presence**

Replace `IsRTXPTAssetsRoot(...)` with:

```cpp
bool IsRTXPTAssetsRoot(const std::string& Path)
{
    return !EnumerateSceneFiles(Path).empty();
}
```

This keeps the same root search order but no longer rejects an assets folder just because the preferred Bistro scene or model is missing.

- [ ] **Step 6: Implement sample scene-list and resource helpers**

Add these methods after `CreateFrameResources()`:

```cpp
void RTXPTSample::EnumerateAvailableScenes()
{
    m_AvailableScenes = EnumerateSceneFiles(m_AssetsRoot);
}

void RTXPTSample::ResetSceneDependentResources()
{
    m_Materials.Reset();
    m_Lights.Reset();
    m_AccelerationStructures.Reset();
    m_RayTracingPass.Reset();

    m_SelectedSceneCamera   = -1;
    m_AccumulationFrame     = 0;
    m_AccumulationActive    = false;
    m_HasLastCameraMatrices = false;
}

bool RTXPTSample::RebuildSceneDependentResources()
{
    const GLTF::Model* pModel = m_Scene.GetModel();
    if (pModel == nullptr)
    {
        ResetSceneDependentResources();
        CreatePhase4Passes();
        return false;
    }

    bool ResourcesReady = true;
    ResourcesReady &= m_Materials.Upload(m_pDevice, *pModel);
    if (m_Scene.GetSceneIndex() < pModel->Scenes.size())
        ResourcesReady &= m_Lights.Upload(m_pDevice, pModel->Scenes[m_Scene.GetSceneIndex()], m_Scene.GetTransforms());
    else
        m_Lights.Reset();

    ResourcesReady &=
        m_AccelerationStructures.BuildStaticScene(m_pDevice,
                                                  m_pImmediateContext,
                                                  *pModel,
                                                  m_Scene.GetSceneIndex(),
                                                  m_Scene.GetIndexType(),
                                                  m_Scene.GetTransforms(),
                                                  m_FeatureCaps.RayTracing);

    CreatePhase4Passes();
    return ResourcesReady;
}
```

- [ ] **Step 7: Implement `SetCurrentScene(...)`**

Add this method after `RebuildSceneDependentResources()`:

```cpp
bool RTXPTSample::SetCurrentScene(const std::string& SceneName, bool ForceReload)
{
    if (SceneName.empty())
        return false;

    if (!ForceReload && SceneName == m_CurrentSceneName)
        return m_Scene.HasValidContent();

    m_CurrentSceneName = SceneName;
    ResetSceneDependentResources();

    const bool SceneLoaded = m_Scene.LoadScene(m_pDevice, m_pImmediateContext, m_AssetsRoot, SceneName);
    const bool ResourcesReady = SceneLoaded && RebuildSceneDependentResources();
    if (!SceneLoaded)
        CreatePhase4Passes();

    if (SceneLoaded && m_Scene.GetCameraCount() > 0)
    {
        ApplySceneCamera(0);
    }
    else
    {
        InitializeCamera();
        m_SelectedSceneCamera = -1;
    }

    m_HasLastCameraMatrices = false;
    RequestAccumulationReset("Scene changed");
    return SceneLoaded && ResourcesReady;
}
```

- [ ] **Step 8: Route initialization through scene enumeration and selection**

In `RTXPTSample::Initialize(...)`, replace the block from:

```cpp
    m_AssetsRoot = ResolveRTXPTAssetsRoot();
    m_Scene.LoadDefaultScene(m_pDevice, m_pImmediateContext, m_AssetsRoot);
    if (const GLTF::Model* pModel = m_Scene.GetModel())
    {
        m_Materials.Upload(m_pDevice, *pModel);
        if (m_Scene.GetSceneIndex() < pModel->Scenes.size())
            m_Lights.Upload(m_pDevice, pModel->Scenes[m_Scene.GetSceneIndex()], m_Scene.GetTransforms());

        m_AccelerationStructures.BuildStaticScene(m_pDevice,
                                                  m_pImmediateContext,
                                                  *pModel,
                                                  m_Scene.GetSceneIndex(),
                                                  m_Scene.GetIndexType(),
                                                  m_Scene.GetTransforms(),
                                                  m_FeatureCaps.RayTracing);
    }
    else
    {
        m_AccelerationStructures.Reset();
    }

    if (m_Scene.GetCameraCount() > 0)
        ApplySceneCamera(0);

    CreatePhase4Passes();
```

through:

```cpp
    m_AssetsRoot = ResolveRTXPTAssetsRoot();
    EnumerateAvailableScenes();

    std::string InitialScene;
    auto        PreferredIt = std::find(m_AvailableScenes.begin(), m_AvailableScenes.end(), kPreferredSceneName);
    if (PreferredIt != m_AvailableScenes.end())
        InitialScene = *PreferredIt;
    else if (!m_AvailableScenes.empty())
        InitialScene = m_AvailableScenes.front();

    if (!InitialScene.empty())
    {
        SetCurrentScene(InitialScene, true);
    }
    else
    {
        ResetSceneDependentResources();
        CreatePhase4Passes();
    }
```

Keep the existing `EnsureRenderTargets();` after this block.

- [ ] **Step 9: Verify the lifecycle diff**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
rg -n "EnumerateAvailableScenes|SetCurrentScene|ResetSceneDependentResources|RebuildSceneDependentResources|EnumerateSceneFiles|kPreferredSceneName" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
```

Expected:

- `git diff --check` exits with code 0.
- `rg` shows all new helpers and the startup route through `SetCurrentScene(...)`.

- [ ] **Step 10: Commit Task 2 in the submodule**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): rebuild resources when switching scenes" -m "Co-Authored-By: GPT 5.5"
```

Expected: one submodule commit containing only `RTXPTSample.hpp/.cpp`.

---

### Task 3: Add The Runtime Scene Combo And Empty-List Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Insert the scene selection combo in the Scene section**

In `RTXPTSample::UpdateUI()`, inside the `if (ImGui::CollapsingHeader("Scene"))` block, insert this code immediately after `ImGui::Indent(Indent);` and before the existing `ImGui::Text("Scene: %s", ...)` line:

```cpp
        const char* ScenePreview = "none";
        if (!m_CurrentSceneName.empty())
            ScenePreview = m_CurrentSceneName.c_str();
        else if (m_AvailableScenes.empty())
            ScenePreview = "no scenes found";

        ImGui::BeginDisabled(m_AvailableScenes.empty());
        if (ImGui::BeginCombo("Scene", ScenePreview))
        {
            for (const std::string& SceneName : m_AvailableScenes)
            {
                const bool IsSelected = SceneName == m_CurrentSceneName;
                if (ImGui::Selectable(SceneName.c_str(), IsSelected))
                    SetCurrentScene(SceneName);
                if (IsSelected)
                    ImGui::SetItemDefaultFocus();
            }
            ImGui::EndCombo();
        }
        ImGui::EndDisabled();

        if (m_AvailableScenes.empty())
            ImGui::TextWrapped("No RTXPT scene files found under assets root: %s", m_AssetsRoot.c_str());
```

- [ ] **Step 2: Keep existing scene diagnostics below the combo**

Make sure the existing diagnostics still appear after the new combo:

```cpp
        ImGui::Text("Scene: %s", m_Scene.HasValidContent() ? "loaded" : "missing");
        ImGui::Text("Scene file: %s", m_Scene.GetLoadedSceneName().empty() ? "none" : m_Scene.GetLoadedSceneName().c_str());
        ImGui::Text("Model path: %s", m_Scene.GetModelPath().empty() ? "none" : m_Scene.GetModelPath().c_str());
        ImGui::Text("Scene cameras: %u", m_Scene.GetCameraCount());
```

- [ ] **Step 3: Verify UI wiring**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.cpp
rg -n "BeginCombo\\(\"Scene\"|No RTXPT scene files found|SetCurrentScene\\(SceneName\\)" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected:

- `git diff --check` exits with code 0.
- `rg` finds the Scene combo, empty-list message, and UI call into `SetCurrentScene(...)`.

- [ ] **Step 4: Commit Task 3 in the submodule**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose runtime scene selection" -m "Co-Authored-By: GPT 5.5"
```

Expected: one submodule commit containing the Scene UI change.

---

### Task 4: Build Verification And Manual Smoke

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Run formatting and diff checks**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples status --short
```

Expected:

- `git diff --check` exits with code 0.
- `git status --short` is clean inside `DiligentSamples` after the task commits.

- [ ] **Step 2: Build the RTXPT target**

Run from the umbrella root:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

- Build exits with code 0.
- If the build tree does not exist or is configured differently, report the exact missing path or CMake error and do not claim build success.

- [ ] **Step 3: Run the manual scene-switch smoke check**

Launch the RTXPT sample from the built Debug output and verify:

```text
1. The Scene combo lists the *.scene.json files under DiligentSamples/Samples/RTXPT/assets.
2. bistro-programmer-art.scene.json is selected initially when present.
3. Selecting kitchen.scene.json changes Scene file, Model path, Scene cameras, Mesh nodes, Primitives, Materials, and Lights diagnostics.
4. Selecting living-room.scene.json or convergence-test.scene.json also changes the diagnostics.
5. After each switch, the camera resets to scene camera 0 when the scene has cameras.
6. If a scene has no cameras, the default free camera is active and Scene camera preview is none.
7. Accumulated samples restart after each switch.
8. If a scene load fails, the old scene-dependent GPU resources are not still rendered and the UI shows the load error.
```

- [ ] **Step 4: Record final implementation state**

Run:

```powershell
git -C DiligentSamples log -3 --oneline
git status --short --branch
```

Expected:

- The three most recent submodule commits correspond to Tasks 1-3.
- Umbrella repo status shows the `DiligentSamples` submodule pointer modified and no unexpected source-file edits at the root.
