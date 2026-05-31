# RTXPT LastError Removal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `LastError` and `m_LastError` as RTXPT status channels and report failures at the source with Diligent logging or diagnostic macros.

**Architecture:** Existing `bool` returns remain the runtime control-flow path. Recoverable scene/content/fallback failures log with `LOG_ERROR_MESSAGE`, expected feature disablement stays in `DisabledReason`, and unexpected resource or invariant failures use `VERIFY`, `VERIFY_EXPR`, `DEV_ERROR`, or `UNEXPECTED` while still returning safely in release builds.

**Tech Stack:** C++17, Diligent Engine RTXPT sample, ImGui status panel, Diligent `DebugUtilities.hpp` / `Errors.hpp`, CMake on Windows.

---

## File Structure

- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`: remove `GetLastError()` and `m_LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`: log scene parsing/loading failures where they occur.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`: remove `RTXPTMaterialStats::LastError` and unused `<string>`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`: log material texture fallback failures and verify material buffer creation.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`: remove `RTXPTLightStats::LastError` and unused `<string>`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`: verify light buffer creation.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`: remove `GetLastError()` and `m_LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`: log recoverable target/accumulation failures.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.hpp`: remove `GetLastError()` and `m_LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp`: verify setup failures and use diagnostics for impossible render states.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.hpp`: remove `RTXPTComputePassStats::LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`: verify shader, PSO, and SRB creation.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`: remove `RTXPTRayTracingPassStats::LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`: verify shader/PSO/SRB/SBT creation, report unexpected binding failures at source, and remove per-frame stored trace errors.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`: remove `RTXPTAccelerationStructureStats::LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`: log scene/content limits and use diagnostics for resource creation or invalid update state.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp`: remove `RTXPTSkinnedGeometryStats::LastError`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`: log skinned content issues and verify buffer/pipeline/mapping failures.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`: remove all UI reads of removed error state.

## Task 1: Baseline Search Snapshot

**Files:**
- Read only: `DiligentSamples/Samples/RTXPT/src`

- [ ] **Step 1: Capture current LastError sites**

Run:

```powershell
rg -n "m_Stats\.LastError|m_LastError|GetLastError\(|\.LastError" DiligentSamples/Samples/RTXPT/src
```

Expected: matches in `RTXPTScene`, `RTXPTRenderTargets`, `RTXPTBlitPass`, stats headers, implementation files, and `RTXPTSample.cpp`.

- [ ] **Step 2: Capture current diagnostic macro coverage**

Run:

```powershell
rg -n "LOG_ERROR_MESSAGE|VERIFY\(|VERIFY_EXPR|DEV_ERROR|UNEXPECTED" DiligentSamples/Samples/RTXPT/src
```

Expected: no matches before the implementation change.

## Task 2: Remove LastError From Public Data Shapes

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.hpp`

- [ ] **Step 1: Remove scene error accessor and member**

In `RTXPTScene.hpp`, delete this accessor:

```cpp
const std::string& GetLastError() const { return m_LastError; }
```

Delete this member:

```cpp
std::string                   m_LastError;
```

- [ ] **Step 2: Remove stats error fields**

In `RTXPTMaterials.hpp`, make the stats struct:

```cpp
struct RTXPTMaterialStats
{
    Uint32 MaterialCount = 0;
    Uint32 TextureCount  = 0;
};
```

In `RTXPTLights.hpp`, make the stats struct:

```cpp
struct RTXPTLightStats
{
    Uint32 LightCount = 0;
};
```

In `RTXPTAccelerationStructures.hpp`, make the stats tail:

```cpp
Uint64      BLASScratchSize = 0;
Uint64      TLASScratchSize = 0;
std::string DisabledReason;
```

In `RTXPTRayTracingPass.hpp`, make the stats tail:

```cpp
bool        AnyHitEnabled        = false;
Uint32      MaterialTextureCount = 0;
Uint32      TraceCount           = 0;
std::string DisabledReason;
```

In `RTXPTComputePass.hpp`, make the stats tail:

```cpp
bool        LastDispatchExecuted = false;
Uint32      DispatchCount        = 0;
std::string DisabledReason;
```

In `RTXPTSkinnedGeometry.hpp`, make the stats tail:

```cpp
Uint32      JointMatrixCount     = 0;
Uint32      DispatchCount        = 0;
std::string DisabledReason;
```

- [ ] **Step 3: Remove helper class error accessors and members**

In `RTXPTRenderTargets.hpp`, delete:

```cpp
const std::string& GetLastError() const { return m_LastError; }
std::string             m_LastError;
```

In `RTXPTBlitPass.hpp`, delete:

```cpp
const std::string& GetLastError() const { return m_LastError; }
std::string                           m_LastError;
```

- [ ] **Step 4: Remove newly unused string includes**

Remove `#include <string>` from `RTXPTMaterials.hpp`, `RTXPTLights.hpp`, `RTXPTRenderTargets.hpp`, and `RTXPTBlitPass.hpp` only if the file no longer refers to `std::string`.

- [ ] **Step 5: Check header-only removal**

Run:

```powershell
rg -n "LastError|GetLastError|m_LastError" DiligentSamples/Samples/RTXPT/src/*.hpp
```

Expected: no matches.

## Task 3: Convert Scene Loading Errors To Source Logging

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`

- [ ] **Step 1: Add diagnostic include**

Add this include with the existing Diligent interface includes:

```cpp
#include "DebugUtilities.hpp"
```

- [ ] **Step 2: Remove reset of deleted member**

In `RTXPTScene::ResetLoadedData`, delete:

```cpp
m_LastError.clear();
```

- [ ] **Step 3: Change scene JSON helper signature**

Change the anonymous namespace helper signature to:

```cpp
bool ReadSceneModelPath(const std::string& ScenePath, std::string& ModelRelativePath)
```

For each helper failure, log and return `false`. The final helper body should use this pattern:

```cpp
std::ifstream SceneFile{ScenePath};
if (!SceneFile)
{
    LOG_ERROR_MESSAGE("Unable to open scene file: ", ScenePath);
    return false;
}

nlohmann::json SceneJson = nlohmann::json::parse(SceneFile, nullptr, false);
if (SceneJson.is_discarded() || !SceneJson.is_object())
{
    LOG_ERROR_MESSAGE("Invalid scene JSON: ", ScenePath);
    return false;
}
```

Apply the same `LOG_ERROR_MESSAGE(..., ScenePath)` pattern for the non-empty `models` array, string `models[0]`, and non-empty model path checks.

- [ ] **Step 4: Log `LoadScene` validation failures**

Replace each former assignment to `m_LastError` with immediate logging:

```cpp
if (SceneName.empty())
{
    LOG_ERROR_MESSAGE("Empty RTXPT scene file name");
    return false;
}

if (!FileSystem::FileExists(ScenePath.c_str()))
{
    LOG_ERROR_MESSAGE("Missing scene file: ", ScenePath);
    return false;
}

if (!ReadSceneModelPath(ScenePath, ModelRelativePath))
    return false;

if (!FileSystem::FileExists(m_ModelPath.c_str()))
{
    LOG_ERROR_MESSAGE("Missing glTF file: ", m_ModelPath);
    return false;
}
```

In the `catch` block, use:

```cpp
LOG_ERROR_MESSAGE("Failed to load RTXPT glTF model '", m_ModelPath, "': ", e.what());
```

- [ ] **Step 5: Verify scene source has no error state**

Run:

```powershell
rg -n "m_LastError|GetLastError|ReadSceneModelPath\(ScenePath, ModelRelativePath,|LastError" DiligentSamples/Samples/RTXPT/src/RTXPTScene.*
```

Expected: no matches.

## Task 4: Convert Material, Light, Render Target, And Blit Reporting

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp`

- [ ] **Step 1: Add diagnostic include to each file**

Add this include in each listed `.cpp` file:

```cpp
#include "DebugUtilities.hpp"
```

- [ ] **Step 2: Log material texture fallback failures**

In `RTXPTMaterials::Upload`, replace texture failure assignments with:

```cpp
LOG_ERROR_MESSAGE("RTXPT material texture is missing; texture sampling disabled");
```

and:

```cpp
LOG_ERROR_MESSAGE("RTXPT material texture view creation failed; texture sampling disabled");
```

Keep the existing `m_TextureBindings.clear();`, `m_TextureViews.clear();`, and `break;` behavior.

- [ ] **Step 3: Verify material and light buffer creation**

After `CreateBuffer` calls for material and light buffers, use:

```cpp
VERIFY(m_MaterialBuffer, "Failed to create RTXPT material buffer");
if (!m_MaterialBuffer)
    return false;
```

and:

```cpp
VERIFY(m_LightBuffer, "Failed to create RTXPT light buffer");
if (!m_LightBuffer)
    return false;
```

- [ ] **Step 4: Convert render target failure messages**

In `RTXPTRenderTargets::Reset`, delete:

```cpp
m_LastError.clear();
```

In `CreateTarget`, replace former assignments with:

```cpp
LOG_ERROR_MESSAGE("Failed to create ", Name);
return false;
```

and:

```cpp
LOG_ERROR_MESSAGE(Name, " is missing SRV or UAV");
Target.Release();
return false;
```

In accumulation fallback, replace the former assignment with:

```cpp
LOG_ERROR_MESSAGE("RGBA32F UAV is not supported; reference path tracer accumulation is disabled");
m_AccumulationUnavailable = true;
return true;
```

For `RTXPT AccumColor` creation, use:

```cpp
VERIFY(m_AccumColor, "Failed to create RTXPT AccumColor");
if (!m_AccumColor)
    return false;
```

- [ ] **Step 5: Convert blit setup and render failures**

In `RTXPTBlitPass::Reset`, delete:

```cpp
m_LastError.clear();
```

For shader, PSO, and SRB creation failures, use:

```cpp
VERIFY(pVS && pPS, "Failed to create RTXPT blit shaders");
if (!pVS || !pPS)
    return false;

VERIFY(m_PSO, "Failed to create RTXPT blit PSO");
if (!m_PSO)
    return false;

VERIFY(m_SRB, "Failed to create RTXPT blit SRB");
if (!m_SRB)
    return false;
```

In `Render`, replace stored skip errors with diagnostics and safe returns:

```cpp
if (!IsReady())
{
    DEV_ERROR("RTXPT blit pass is not ready");
    return false;
}
if (pSourceSRV == nullptr)
{
    DEV_ERROR("RTXPT blit source SRV is null");
    return false;
}
```

For missing texture binding, use:

```cpp
UNEXPECTED("RTXPT blit texture binding is unavailable");
return false;
```

- [ ] **Step 6: Verify converted files have no LastError state**

Run:

```powershell
rg -n "LastError|m_LastError|GetLastError|m_Stats\.LastError" DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.* DiligentSamples/Samples/RTXPT/src/RTXPTLights.* DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.* DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.*
```

Expected: no matches.

## Task 5: Convert Compute And Ray Tracing Pass Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Add diagnostic include to each file**

Add this include in both files:

```cpp
#include "DebugUtilities.hpp"
```

- [ ] **Step 2: Verify compute pass resource creation**

In `RTXPTComputePass::Initialize`, replace the three former `m_Stats.LastError` assignments with:

```cpp
VERIFY(pCS, "Failed to create ", m_Name, " shader");
if (!pCS)
    return false;

VERIFY(m_PSO, "Failed to create ", m_Name, " PSO");
if (!m_PSO)
    return false;

VERIFY(m_SRB, "Failed to create ", m_Name, " SRB");
if (!m_SRB)
    return false;
```

- [ ] **Step 3: Verify ray tracing shader and PSO creation**

In `RTXPTRayTracingPass::Initialize`, replace shader and PSO error assignments with:

```cpp
VERIFY(pRayGen && (ScreenPatternDiagnostic || (pMiss && pClosestHit && (!UseTextures || pAnyHit))),
       "Failed to create RTXPT reference ray tracing shaders");
if (!pRayGen || (!ScreenPatternDiagnostic && (!pMiss || !pClosestHit || (UseTextures && !pAnyHit))))
    return false;

VERIFY(m_PSO, "Failed to create RTXPT reference RT PSO");
if (!m_PSO)
    return false;
```

- [ ] **Step 4: Replace `SetStatic` stored errors**

Change the lambda body to use diagnostics:

```cpp
auto SetStatic = [&](SHADER_TYPE Stage, const char* Name, IDeviceObject* pObject, const char* ObjectName) {
    if (pObject == nullptr)
    {
        DEV_ERROR("RTXPT static resource object is null: ", ObjectName);
        return false;
    }

    IShaderResourceVariable* pVar = m_PSO->GetStaticVariableByName(Stage, Name);
    if (pVar == nullptr)
    {
        UNEXPECTED("RTXPT static shader variable is missing: ", Name);
        return false;
    }

    pVar->Set(pObject);
    return true;
};
```

Remove any fallback code that assigns `"Failed to bind required RTXPT frame constants or TLAS"` or `"Failed to bind required RTXPT bridge buffers"` to `m_Stats.LastError`. The existing `return false` remains.

- [ ] **Step 5: Replace ray tracing content/resource failures**

For unsupported index type, log and return:

```cpp
LOG_ERROR_MESSAGE("Reference path tracer requires VT_UINT16 or VT_UINT32 indices");
return false;
```

For index view, SRB, texture binding, and SBT failures, use:

```cpp
VERIFY(m_IndexBufferView, "Failed to create RTXPT index buffer view");
if (!m_IndexBufferView)
    return false;

VERIFY(m_SRB, "Failed to create RTXPT reference RT SRB");
if (!m_SRB)
    return false;

UNEXPECTED("Failed to find RTXPT material texture array binding");
return false;

VERIFY(m_SBT, "Failed to create RTXPT reference SBT");
if (!m_SBT)
    return false;
```

- [ ] **Step 6: Remove stored trace skip errors**

In `RTXPTRayTracingPass::Trace`, replace the combined guard with explicit safe returns:

```cpp
if (!IsReady())
    return false;
if (pOutputUAV == nullptr || pAccumulationUAV == nullptr || Width == 0 || Height == 0)
    return false;
```

For missing output bindings after readiness, use:

```cpp
UNEXPECTED("Failed to find RTXPT output bindings");
return false;
```

- [ ] **Step 7: Verify converted pass files**

Run:

```powershell
rg -n "LastError|m_Stats\.LastError" DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.* DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.*
```

Expected: no matches.

## Task 6: Convert Acceleration Structure And Skinned Geometry Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`

- [ ] **Step 1: Add diagnostic include to each file**

Add this include in both files:

```cpp
#include "DebugUtilities.hpp"
```

- [ ] **Step 2: Convert acceleration structure validation failures**

In `RTXPTAccelerationStructures::BuildScene`, replace scene/content validation assignments with `LOG_ERROR_MESSAGE` and keep `return false`, for example:

```cpp
if (pDevice == nullptr || pContext == nullptr)
{
    DEV_ERROR("RTXPT acceleration structure build requires a device and device context");
    return false;
}

if (SceneIndex >= Model.Scenes.size())
{
    LOG_ERROR_MESSAGE("Invalid RTXPT scene index for acceleration structure build");
    return false;
}

if (!Position.Valid || Position.ValueType != VT_FLOAT32 || Position.ComponentCount != 3)
{
    LOG_ERROR_MESSAGE("RTXPT BLAS build requires float3 POSITION vertex data");
    return false;
}
```

Use the same pattern for missing ray-tracing bind flags, unsupported index size, skinned node readiness issues, sub-instance table overflow, and empty TLAS instance list.

- [ ] **Step 3: Verify acceleration structure resource creation**

For BLAS, scratch buffers, TLAS, instance buffer, TLAS scratch, and sub-instance buffer creation, use `VERIFY` and keep release-mode safe returns:

```cpp
VERIFY(Record.BLAS, "Failed to create RTXPT BLAS");
if (!Record.BLAS)
    return false;

VERIFY(m_BLASScratch, "Failed to create RTXPT BLAS scratch buffer");
if (!m_BLASScratch)
    return false;

VERIFY(m_TLAS, "Failed to create RTXPT TLAS");
if (!m_TLAS)
    return false;
```

Apply the same form to `m_InstanceBuffer`, `m_TLASScratch`, and `m_SubInstanceBuffer`.

- [ ] **Step 4: Convert dynamic update invalid states**

In `UpdateTLAS` and `UpdateDynamicBLAS`, replace deleted error assignments with `DEV_ERROR` and safe returns:

```cpp
DEV_ERROR("RTXPT dynamic TLAS update requires built TLAS resources");
return false;
```

Use the same form for invalid instance data, missing node transforms, missing context, unbuilt acceleration structures, stale skinned dispatch count, mismatched skinned vertex buffer, and not-ready skinned geometry.

- [ ] **Step 5: Convert skinned geometry setup failures**

In `RTXPTSkinnedGeometry::CreateBuffers`, use `VERIFY` for buffer creation:

```cpp
VERIFY(m_SkinnedVertexBuffer, "Failed to create RTXPT skinned vertex arena");
if (!m_SkinnedVertexBuffer)
    return false;
```

For missing source vertex/skin buffers when skinned nodes exist, use:

```cpp
LOG_ERROR_MESSAGE("RTXPT skinned geometry requires source vertex and skin buffers");
return false;
```

Use `VERIFY` and safe returns for `m_JointMatrixBuffer`, `m_SkinningConstantsCB`, shader creation, PSO creation, resource binding, and SRB creation.

- [ ] **Step 6: Convert skinned update failures**

For invalid or incomplete skin transforms, use:

```cpp
LOG_ERROR_MESSAGE("RTXPT skinned node is missing skin transform data");
return false;
```

For map failures and null context, use:

```cpp
VERIFY(Mapped, "Failed to map RTXPT skinned joint matrix buffer");
if (!Mapped)
    return false;
```

and:

```cpp
DEV_ERROR("RTXPT skinned geometry requires a device context");
return false;
```

- [ ] **Step 7: Verify converted AS and skinning files**

Run:

```powershell
rg -n "LastError|m_Stats\.LastError" DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.* DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.*
```

Expected: no matches.

## Task 7: Remove LastError UI Reads

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Remove Scene panel error rows**

Delete these blocks:

```cpp
if (!m_Scene.GetLastError().empty())
    ImGui::TextWrapped("Asset load error: %s", m_Scene.GetLastError().c_str());
```

```cpp
if (!m_Materials.GetStats().LastError.empty())
    ImGui::TextWrapped("Material buffer error: %s", m_Materials.GetStats().LastError.c_str());
if (!m_Lights.GetStats().LastError.empty())
    ImGui::TextWrapped("Light buffer error: %s", m_Lights.GetStats().LastError.c_str());
```

- [ ] **Step 2: Remove Status / Debug error rows**

Delete these blocks:

```cpp
if (!ASStats.LastError.empty())
    ImGui::TextWrapped("AS error: %s", ASStats.LastError.c_str());
```

```cpp
if (!m_SkinnedGeometry.GetStats().LastError.empty())
    ImGui::TextWrapped("Skinning error: %s", m_SkinnedGeometry.GetStats().LastError.c_str());
```

```cpp
if (!RTPassStats.LastError.empty())
    ImGui::TextWrapped("TraceRays error: %s", RTPassStats.LastError.c_str());
```

```cpp
if (!ComputeStats.LastError.empty())
    ImGui::TextWrapped("Compute error: %s", ComputeStats.LastError.c_str());
```

```cpp
if (!m_RenderTargets.GetLastError().empty())
    ImGui::TextWrapped("Render target error: %s", m_RenderTargets.GetLastError().c_str());
if (!m_BlitPass.GetLastError().empty())
    ImGui::TextWrapped("Blit error: %s", m_BlitPass.GetLastError().c_str());
```

- [ ] **Step 3: Verify UI no longer references removed fields**

Run:

```powershell
rg -n "LastError|GetLastError" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: no matches.

## Task 8: Final Verification And Commit

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src`
- Commit all modified RTXPT source/header files.

- [ ] **Step 1: Run final LastError absence check**

Run:

```powershell
rg -n "LastError|GetLastError|m_LastError|m_Stats\.LastError" DiligentSamples/Samples/RTXPT/src
```

Expected: no matches.

- [ ] **Step 2: Run macro presence check**

Run:

```powershell
rg -n "LOG_ERROR_MESSAGE|VERIFY\(|VERIFY_EXPR|DEV_ERROR|UNEXPECTED" DiligentSamples/Samples/RTXPT/src
```

Expected: matches in the `.cpp` files converted by Tasks 3 through 6.

- [ ] **Step 3: Build RTXPT target**

Run:

```powershell
if (Test-Path build\x64\Debug\CMakeCache.txt) {
    cmake --build build\x64\Debug --config Debug --target RTXPT
} else {
    cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
    cmake --build build\x64\Debug --config Debug --target RTXPT
}
```

Expected: `RTXPT` target builds successfully. If local dependencies or generator setup prevent configuration, capture the exact CMake error and keep the `rg` verification results as completed static validation.

- [ ] **Step 4: Inspect changed files**

Run:

```powershell
git diff -- DiligentSamples/Samples/RTXPT/src
git status --short
```

Expected: only intended RTXPT source/header files are modified, plus any pre-existing unrelated untracked files.

- [ ] **Step 5: Commit implementation**

Run:

```powershell
git add -- DiligentSamples/Samples/RTXPT/src
git commit -m "refactor(rtxpt): replace LastError status with diagnostics" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds. Do not add unrelated untracked files such as other plan documents.

## Self-Review Notes

- Spec coverage: Tasks 2 and 7 remove the status/UI channel; Tasks 3 through 6 add source-site logging and diagnostics; Task 8 verifies absence and build health.
- Type consistency: removed fields are never read after Task 7, and `DisabledReason` remains unchanged.
- Scope: all active source edits are limited to `DiligentSamples/Samples/RTXPT/src`.
