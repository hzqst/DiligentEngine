# RTXPT Phase 2 Closure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the small Phase 2 runtime-asset gap so the RTXPT sample resolves local assets, reports missing paths in the UI, and keeps the clear fallback runnable.

**Architecture:** Keep the existing `RTXPTSample` and `RTXPTScene` split. `RTXPTSample` resolves the best local assets root from a short candidate list and passes it to `RTXPTScene`; `RTXPTScene` records the resolved model path and last load error for status UI. Large runtime assets remain ignored by Git and are not added to CMake target sources.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentTools `GLTFLoader`, Diligent platform `FileSystem`, Dear ImGui.

---

## Scope

This plan only closes the Phase 2 runtime-asset path and diagnostics gap:

- Resolve `DiligentSamples/Samples/RTXPT/assets` at runtime without depending on `D:/RTXPT-fork`.
- Confirm the expected local files exist before calling `GLTF::Model`.
- Show the resolved asset root, scene load state, model path, and missing/load error in the existing RTXPT ImGui status window.

This plan intentionally does not:

- Upload or track large assets.
- Add assets back to `DiligentSamples/Samples/RTXPT/CMakeLists.txt`.
- Parse RTXPT material/light metadata.
- Create render targets, constant buffers, samplers, BLAS, TLAS, PSO, SBT, or `TraceRays`.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
  - Add read-only accessors for resolved asset root, resolved model path, and last load error.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
  - Validate the expected scene file and Bistro glTF path before constructing `GLTF::Model`.
  - Store explicit diagnostics instead of silently clearing scene state.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Add a small local helper to find the RTXPT assets root from runtime and source-tree candidates.
  - Pass the resolved root to `RTXPTScene::LoadDefaultScene`.
  - Show missing asset details in the existing status UI.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Add `std::string m_AssetsRoot` so the UI can report the root used during initialization.

---

### Task 1: Add Scene Load Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Extend `RTXPTScene` state and accessors**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`, add `<vector>` only if it is already needed by the implementation you choose; for this narrow closure use strings only. Replace the current class body with:

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

private:
    std::unique_ptr<GLTF::Model> m_Model;
    std::string                  m_LoadedSceneName;
    std::string                  m_AssetsRoot;
    std::string                  m_ModelPath;
    std::string                  m_LastError;
};
```

- [ ] **Step 2: Add `FileSystem` include and path helpers**

In `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`, add the include below the existing project include:

```cpp
#include "FileSystem.hpp"
```

Then add this helper block inside `namespace Diligent`, before `RTXPTScene::LoadDefaultScene`:

```cpp
namespace
{

std::string JoinPath(const std::string& Root, const char* RelativePath)
{
    if (Root.empty())
        return RelativePath;

    std::string Path = Root;
    if (!FileSystem::IsSlash(Path.back()))
        Path.push_back(FileSystem::SlashSymbol);
    Path += RelativePath;
    FileSystem::CorrectSlashes(Path);
    return FileSystem::SimplifyPath(Path.c_str());
}

} // namespace
```

- [ ] **Step 3: Validate expected assets before loading glTF**

Replace `RTXPTScene::LoadDefaultScene` in `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp` with:

```cpp
bool RTXPTScene::LoadDefaultScene(IRenderDevice* pDevice, IDeviceContext* pContext, const std::string& AssetsRoot)
{
    m_Model.reset();
    m_LoadedSceneName.clear();
    m_LastError.clear();

    m_AssetsRoot = FileSystem::SimplifyPath(AssetsRoot.c_str());
    const std::string ScenePath = JoinPath(m_AssetsRoot, "bistro-programmer-art.scene.json");
    m_ModelPath                = JoinPath(m_AssetsRoot, "Models/Bistro/bistro.gltf");

    if (!FileSystem::FileExists(ScenePath.c_str()))
    {
        m_LastError = "Missing scene file: " + ScenePath;
        return false;
    }

    if (!FileSystem::FileExists(m_ModelPath.c_str()))
    {
        m_LastError = "Missing glTF file: " + m_ModelPath;
        return false;
    }

    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = m_ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;

    try
    {
        m_Model           = std::make_unique<GLTF::Model>(pDevice, pContext, ModelCI);
        m_LoadedSceneName = "bistro-programmer-art.scene.json";
    }
    catch (const std::exception& e)
    {
        m_Model.reset();
        m_LoadedSceneName.clear();
        m_LastError = e.what();
    }

    return m_Model != nullptr;
}
```

- [ ] **Step 4: Preserve the existing structured deferred-work markers**

Keep the existing structured comments below the load attempt in `RTXPTScene::LoadDefaultScene`. These comments track future material parsing, acceleration structure construction, and ray dispatch work. Do not add a new missing-asset marker because this task records missing assets in `m_LastError`.

- [ ] **Step 5: Verify diagnostics are searchable**

Run:

```powershell
rg "GetLastError|Missing scene file|Missing glTF file|GetModelPath|GetAssetsRoot" DiligentSamples/Samples/RTXPT/src
```

Expected: matches appear in `RTXPTScene.hpp` and `RTXPTScene.cpp`.

- [ ] **Step 6: Commit scene diagnostics**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp
git -C DiligentSamples commit -m "fix(samples): report RTXPT scene asset load errors" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Resolve Local RTXPT Assets Root

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add string storage to the sample header**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, add this standard include near the other includes:

```cpp
#include <string>
```

Add this private member before `m_Scene`:

```cpp
    std::string      m_AssetsRoot;
```

The private section should become:

```cpp
private:
    RTXPTFeatureCaps m_FeatureCaps;
    std::string      m_AssetsRoot;
    RTXPTScene       m_Scene;
};
```

- [ ] **Step 2: Include `FileSystem` in the sample source**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, add this include after `GraphicsAccessories.hpp`:

```cpp
#include "FileSystem.hpp"
```

- [ ] **Step 3: Add local asset-root resolver**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, inside the anonymous namespace after `MakeFeatureCaps`, add:

```cpp
std::string JoinPath(const std::string& Root, const char* RelativePath)
{
    if (Root.empty())
        return RelativePath;

    std::string Path = Root;
    if (!FileSystem::IsSlash(Path.back()))
        Path.push_back(FileSystem::SlashSymbol);
    Path += RelativePath;
    FileSystem::CorrectSlashes(Path);
    return FileSystem::SimplifyPath(Path.c_str());
}

bool IsRTXPTAssetsRoot(const std::string& Path)
{
    const std::string ScenePath = JoinPath(Path, "bistro-programmer-art.scene.json");
    const std::string ModelPath = JoinPath(Path, "Models/Bistro/bistro.gltf");
    return FileSystem::FileExists(ScenePath.c_str()) && FileSystem::FileExists(ModelPath.c_str());
}

std::string ResolveRTXPTAssetsRoot()
{
    const char* Candidates[] =
    {
        "assets",
        "Samples/RTXPT/assets",
        "../Samples/RTXPT/assets",
        "../../Samples/RTXPT/assets",
        "DiligentSamples/Samples/RTXPT/assets",
        "../DiligentSamples/Samples/RTXPT/assets",
        "../../DiligentSamples/Samples/RTXPT/assets",
    };

    for (const char* Candidate : Candidates)
    {
        const std::string Path = FileSystem::SimplifyPath(Candidate);
        if (IsRTXPTAssetsRoot(Path))
            return Path;
    }

    return FileSystem::SimplifyPath("DiligentSamples/Samples/RTXPT/assets");
}
```

- [ ] **Step 4: Use the resolved root during initialization**

Replace the scene loading block in `RTXPTSample::Initialize`:

```cpp
    const bool SceneLoaded = m_Scene.LoadDefaultScene(m_pDevice, m_pImmediateContext, ".");
    if (!SceneLoaded)
    {
    }
```

with:

```cpp
    m_AssetsRoot = ResolveRTXPTAssetsRoot();
    m_Scene.LoadDefaultScene(m_pDevice, m_pImmediateContext, m_AssetsRoot);
```

- [ ] **Step 5: Verify assets root is no longer hardcoded to current directory**

Run:

```powershell
rg 'LoadDefaultScene\(.*"\."' DiligentSamples/Samples/RTXPT/src
```

Expected: no matches.

- [ ] **Step 6: Verify the local assets are discoverable from the repository root**

Run from `d:\DiligentEngine-hzqst`:

```powershell
Test-Path 'DiligentSamples\Samples\RTXPT\assets\bistro-programmer-art.scene.json'
Test-Path 'DiligentSamples\Samples\RTXPT\assets\Models\Bistro\bistro.gltf'
```

Expected output:

```text
True
True
```

- [ ] **Step 7: Commit asset root resolution**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "fix(samples): resolve RTXPT local assets root" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Show Asset Diagnostics In UI

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Replace minimal scene status UI with detailed diagnostics**

In `RTXPTSample::UpdateUI`, replace:

```cpp
    ImGui::Text("Scene: %s", m_Scene.HasValidContent() ? "loaded" : "missing");
    ImGui::Text("Scene file: %s", m_Scene.GetLoadedSceneName().c_str());
```

with:

```cpp
    ImGui::Separator();
    ImGui::Text("Assets root: %s", m_AssetsRoot.c_str());
    ImGui::Text("Scene: %s", m_Scene.HasValidContent() ? "loaded" : "missing");
    ImGui::Text("Scene file: %s", m_Scene.GetLoadedSceneName().empty() ? "none" : m_Scene.GetLoadedSceneName().c_str());
    ImGui::Text("Model path: %s", m_Scene.GetModelPath().empty() ? "none" : m_Scene.GetModelPath().c_str());
    if (!m_Scene.GetLastError().empty())
        ImGui::TextWrapped("Asset load error: %s", m_Scene.GetLastError().c_str());
```

- [ ] **Step 2: Remove the completed Phase 2 UI deferred-work line**

In `RTXPTSample::UpdateUI`, remove the visible line that says the sample still needs full RTXPT scene/material/light metadata parsing. The missing-path diagnostic is now explicit and the remaining material parsing note belongs in source, not visible UI.

Keep the existing source-level material parsing deferred-work marker in `RTXPTScene.cpp`; that is a separate Phase 2 metadata task outside this closure plan.

- [ ] **Step 3: Verify UI strings are present**

Run:

```powershell
rg "Assets root|Asset load error|Model path|Scene file" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: all four UI labels are present.

- [ ] **Step 4: Commit UI diagnostics**

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "fix(samples): show RTXPT asset diagnostics" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Closure Verification And Handoff

**Files:**
- Verify only; no expected file edits.

- [ ] **Step 1: Confirm large assets are still ignored**

Run:

```powershell
git -C DiligentSamples status --short --ignored Samples/RTXPT/assets
```

Expected:

```text
!! Samples/RTXPT/assets/ArtLicenses.txt
!! Samples/RTXPT/assets/EnvironmentMaps/
!! Samples/RTXPT/assets/Fonts/
!! Samples/RTXPT/assets/Materials/
!! Samples/RTXPT/assets/Models/
!! Samples/RTXPT/assets/SampleGame/
!! Samples/RTXPT/assets/Screenshots/
!! Samples/RTXPT/assets/StandaloneTextures/
!! Samples/RTXPT/assets/bistro-programmer-art.scene.json
```

Additional ignored scene files may also appear. `.gitignore` and `README.md` must not appear as untracked files.

- [ ] **Step 2: Confirm source changes do not add assets to CMake**

Run:

```powershell
rg "Samples/RTXPT/assets|RTXPT/assets|set\\(ASSETS|ASSETS" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: no matches.

- [ ] **Step 3: Confirm Phase 2 closure code is present**

Run:

```powershell
rg "ResolveRTXPTAssetsRoot|GetLastError|Asset load error|LoadDefaultScene\\(m_pDevice, m_pImmediateContext, m_AssetsRoot\\)" DiligentSamples/Samples/RTXPT/src
```

Expected: matches in `RTXPTSample.cpp`, `RTXPTScene.hpp`, and `RTXPTScene.cpp`.

- [ ] **Step 4: Optional compile check if the user explicitly asks for build verification**

Because the workspace rule says not to run test or build commands unless explicitly requested, do not run this step by default. If the user asks for verification, run the existing build command for the configured tree, for example:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build exits with code 0. If the local build directory or target name differs, inspect the configured build tree first and report the exact command used.

- [ ] **Step 5: Optional runtime check if the user explicitly asks for launch verification**

Because launch verification depends on the local build output location, do not run this step by default. If the user asks for runtime verification, launch `RTXPT` from the built sample output directory and confirm the UI shows:

```text
Assets root: <resolved path ending in DiligentSamples/Samples/RTXPT/assets or assets>
Scene: loaded
Scene file: bistro-programmer-art.scene.json
Model path: <resolved path ending in Models/Bistro/bistro.gltf>
```

If the glTF loader reports optional missing texture warnings such as `paris_ceilingfan_01a_em.png`, record them as non-fatal warnings while confirming whether the scene object still loads.

- [ ] **Step 6: Update the top-level submodule pointer after DiligentSamples commits**

Run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples
git commit -m "fix(samples): close RTXPT phase 2 asset diagnostics" -m "Co-Authored-By: GPT 5.5"
```

---

## Self-Review Checklist

- [ ] The plan resolves the runtime assets root without depending on `D:/RTXPT-fork`.
- [ ] The plan checks both `bistro-programmer-art.scene.json` and `Models/Bistro/bistro.gltf`.
- [ ] The plan reports missing file/load errors in the existing ImGui status window.
- [ ] The plan keeps large assets ignored and does not add them to CMake.
- [ ] The plan preserves the clear fallback render path.
- [ ] The plan leaves Phase 3 acceleration-structure and ray-tracing work out of scope.
