# RTXPT Scene JSON Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the RTXPT sample's `models[0]` scene loading path with a full `.scene.json` adapter that loads the recursive graph, multi-model instances, transforms, material extensions, light metadata, settings metadata, and per-instance skinned animation.

**Architecture:** Add an RTXPT-owned scene data layer instead of trying to merge multiple glTF files into one `GLTF::Model`. `RTXPTScene` owns parsing and CPU scene data, while `RTXPTSample` keeps lifecycle/UI/resource orchestration and the scene-dependent managers consume a read-only scene view. GPU-facing material/light/sub-instance layouts remain stable; narrowly scoped vertex/index bridge work is allowed only if required for multi-model rendering correctness.

**Tech Stack:** C++17, DiligentSamples RTXPT, DiligentTools `GLTFLoader`, Diligent `FileSystem`, `nlohmann::json`, Dear ImGui, Diligent ray tracing AS APIs, HLSL DXC shaders.

---

## Context You Need Before Starting

This plan implements `docs/superpowers/specs/2026-05-31-rtxpt-scene-json-adapter-design.md`.

Current RTXPT facts:

- `RTXPTScene::LoadScene(...)` reads only `models[0]`.
- `RTXPTScene` exposes one `GLTF::Model`, one `GLTF::ModelTransforms`, one scene index, and direct vertex/index buffer getters.
- `RTXPTSample::RebuildSceneDependentResources()` calls `RTXPTMaterials::Upload(*pModel)`, `RTXPTLights::Upload(Scene, Transforms)`, `RTXPTSkinnedGeometry::Initialize(Model, SceneIndex, ...)`, and `RTXPTAccelerationStructures::BuildScene(Model, SceneIndex, Transforms, ...)`.
- `RTXPTMaterials`, `RTXPTLights`, `RTXPTAccelerationStructures`, and `RTXPTSkinnedGeometry` all assume a single `GLTF::Model`.
- `DiligentSamples/Samples/RTXPT/assets/kitchen.scene.json` has two graph instances that reference the same model id: `MechDrone` and `MechDroneInMicrowave`.
- `DiligentSamples/Samples/RTXPT/assets/living-room.scene.json` contains a trailing comma and must load through the relaxed parser.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt` lists source and header files explicitly. Any new `.cpp` or `.hpp` must be added there.
- `DiligentSamples` is a git submodule. Source commits belong inside `DiligentSamples`; this plan document lives in the umbrella repo.

## File Structure

Create these focused helper files:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.hpp`
  - Declares relaxed JSON loading and typed JSON access helpers.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.cpp`
  - Implements string-literal-aware trailing comma removal and relaxed `nlohmann::json` parsing.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
  - Declares scene graph ids, model assets, model instances, material extension metadata, light metadata, settings metadata, diagnostics, and the read-only scene view.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`
  - Implements transform helpers, material file lookup helpers, scene statistics helpers, and small metadata utilities that would make `RTXPTScene.cpp` too large.

Modify these existing files:

- `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Add the four new helper files to `SOURCE` and `INCLUDE`.

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
  - Replace single-model getters with scene-view getters while keeping temporary compatibility accessors only until all call sites are migrated.

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
  - Replace the `models[0]` loader with the full scene adapter transaction.

- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp/.cpp`
  - Upload global material and texture tables from the scene view.

- `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp/.cpp`
  - Upload lights from graph metadata and keep environment lights as CPU metadata.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp/.cpp`
  - Rename/upgrade the class to `RTXPTSkinnedSceneGeometry` and make it scene-level.

- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp/.cpp`
  - Build BLAS/TLAS across all model instances and use global material ids.

- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp/.cpp`
  - Adjust only if the final static vertex/index bridge requires additional scene-level buffers or bindings.

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp/.cpp`
  - Use scene-view rebuild APIs, update UI diagnostics, and call scene-level skinned updates.

## Cross-Cutting Contracts

- `ModelAssetId`, `GraphNodeId`, `ModelInstanceId`, `MaterialGlobalId`, `TextureGlobalId`, `LightId`, and `CameraId` are `Uint32` ids. Use `InvalidId = ~Uint32{0}` for absent references.
- Scene loading is transactional. Temporary data is committed into `RTXPTScene` only after structural loading succeeds.
- Missing material extension files do not fail the scene.
- Invalid structural scene data fails the scene and resets scene-dependent GPU resources.
- Per-instance animation is required. Instances may share glTF assets, but they must not share pose-dependent transforms, skinned output ranges, or dynamic BLAS records.
- GPU `MaterialPTData`, `PolymorphicLightInfo`, and `SubInstanceData` byte layouts remain unchanged.
- Commit messages inside `DiligentSamples` use the repo convention and include `Co-Authored-By: GPT 5.5`.

---

### Task 1: Add Scene JSON And Scene Graph Scaffolding

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.cpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Check source worktree status**

Run:

```powershell
git -C DiligentSamples status --short
```

Expected: note any existing user changes. Continue only if there are no conflicting edits under `Samples/RTXPT/src` or `Samples/RTXPT/CMakeLists.txt`.

- [ ] **Step 2: Create `RTXPTSceneJson.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.hpp` with this interface:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include <cstddef>
#include <string>

#include "json.hpp"

namespace Diligent
{

struct RTXPTJsonLoadResult
{
    nlohmann::json Json;
    std::string    Error;
    bool           StrictParseFailed  = false;
    bool           RelaxedParseUsed   = false;
    bool           CommentsIgnored    = false;
};

bool LoadRTXPTRelaxedJsonFile(const std::string& FilePath, RTXPTJsonLoadResult& Result);
bool ReadRTXPTFloatArray(const nlohmann::json& Object, const char* Key, float* Values, size_t Count);
float ReadRTXPTOptionalFloat(const nlohmann::json& Object, const char* Key, float DefaultValue);
std::string ReadRTXPTOptionalString(const nlohmann::json& Object, const char* Key, const char* DefaultValue = "");

} // namespace Diligent
```

- [ ] **Step 3: Create `RTXPTSceneJson.cpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTSceneJson.cpp` with a string-literal-aware relaxed parser:

```cpp
#include "RTXPTSceneJson.hpp"

#include <cctype>
#include <fstream>
#include <sstream>

#include "DebugUtilities.hpp"

namespace Diligent
{
namespace
{

std::string ReadWholeTextFile(const std::string& FilePath, std::string& Error)
{
    std::ifstream File{FilePath};
    if (!File)
    {
        Error = "Unable to open JSON file: " + FilePath;
        return {};
    }

    std::ostringstream Strm;
    Strm << File.rdbuf();
    return Strm.str();
}

std::string RemoveTrailingJsonCommas(const std::string& Source)
{
    std::string Output;
    Output.reserve(Source.size());

    bool InString = false;
    bool Escaped  = false;

    for (size_t i = 0; i < Source.size(); ++i)
    {
        const char Ch = Source[i];
        if (InString)
        {
            Output.push_back(Ch);
            if (Escaped)
                Escaped = false;
            else if (Ch == '\\')
                Escaped = true;
            else if (Ch == '"')
                InString = false;
            continue;
        }

        if (Ch == '"')
        {
            InString = true;
            Output.push_back(Ch);
            continue;
        }

        if (Ch == ',')
        {
            size_t Next = i + 1;
            while (Next < Source.size() && std::isspace(static_cast<unsigned char>(Source[Next])) != 0)
                ++Next;
            if (Next < Source.size() && (Source[Next] == ']' || Source[Next] == '}'))
                continue;
        }

        Output.push_back(Ch);
    }

    return Output;
}

} // namespace

bool LoadRTXPTRelaxedJsonFile(const std::string& FilePath, RTXPTJsonLoadResult& Result)
{
    Result = {};

    std::string Error;
    const std::string Text = ReadWholeTextFile(FilePath, Error);
    if (!Error.empty())
    {
        Result.Error = Error;
        LOG_ERROR_MESSAGE(Result.Error);
        return false;
    }

    Result.Json = nlohmann::json::parse(Text, nullptr, false, true);
    Result.CommentsIgnored = true;
    if (!Result.Json.is_discarded())
        return true;

    Result.StrictParseFailed = true;
    const std::string RelaxedText = RemoveTrailingJsonCommas(Text);
    Result.Json                 = nlohmann::json::parse(RelaxedText, nullptr, false, true);
    Result.RelaxedParseUsed     = true;
    Result.CommentsIgnored      = true;
    if (!Result.Json.is_discarded())
        return true;

    Result.Error = "Invalid JSON file after relaxed parsing: " + FilePath;
    LOG_ERROR_MESSAGE(Result.Error);
    return false;
}

bool ReadRTXPTFloatArray(const nlohmann::json& Object, const char* Key, float* Values, size_t Count)
{
    const auto It = Object.find(Key);
    if (It == Object.end() || !It->is_array() || It->size() < Count)
        return false;

    for (size_t Idx = 0; Idx < Count; ++Idx)
    {
        if (!(*It)[Idx].is_number())
            return false;
        Values[Idx] = (*It)[Idx].get<float>();
    }
    return true;
}

float ReadRTXPTOptionalFloat(const nlohmann::json& Object, const char* Key, float DefaultValue)
{
    const auto It = Object.find(Key);
    return It != Object.end() && It->is_number() ? It->get<float>() : DefaultValue;
}

std::string ReadRTXPTOptionalString(const nlohmann::json& Object, const char* Key, const char* DefaultValue)
{
    const auto It = Object.find(Key);
    return It != Object.end() && It->is_string() ? It->get<std::string>() : std::string{DefaultValue};
}

} // namespace Diligent
```

- [ ] **Step 4: Create `RTXPTSceneGraph.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp` with the scene ids and data containers:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "GLTFLoader.hpp"
#include "json.hpp"

namespace Diligent
{

using RTXPTSceneId = Uint32;
constexpr RTXPTSceneId InvalidRTXPTSceneId = ~Uint32{0};

struct RTXPTModelAsset
{
    std::string                  RelativePath;
    std::string                  ResolvedPath;
    std::string                  ModelName;
    std::unique_ptr<GLTF::Model> Model;
    Uint32                       SceneIndex = 0;
    GLTF::ModelTransforms        StaticTransforms;
    std::vector<Uint32>          MaterialRemap;
    std::vector<Uint32>          TextureRemap;
};

struct RTXPTAnimationState
{
    bool  Enabled        = true;
    Int32 AnimationIndex = -1;
    float Time           = 0.0f;
    float PlaySpeed      = 1.0f;
    float TimeOffset     = 0.0f;
};

struct RTXPTGraphNode
{
    std::string                 Name;
    std::string                 Type;
    RTXPTSceneId                ParentId     = InvalidRTXPTSceneId;
    RTXPTSceneId                ModelAssetId = InvalidRTXPTSceneId;
    std::vector<RTXPTSceneId>   Children;
    float4x4                    LocalTransform  = float4x4::Identity();
    float4x4                    GlobalTransform = float4x4::Identity();
    nlohmann::json              RawMetadata;
};

struct RTXPTModelInstance
{
    RTXPTSceneId         GraphNodeId  = InvalidRTXPTSceneId;
    RTXPTSceneId         ModelAssetId = InvalidRTXPTSceneId;
    std::string          Name;
    float4x4             GlobalTransform = float4x4::Identity();
    RTXPTAnimationState  Animation;
    GLTF::ModelTransforms Transforms;
};

struct RTXPTMaterialExtension
{
    std::string   FilePath;
    std::string   ModelName;
    std::string   MaterialName;
    bool          Loaded = false;
    nlohmann::json RawJson;
};

struct RTXPTSceneLightMetadata
{
    std::string   Name;
    std::string   Type;
    float4x4      GlobalTransform = float4x4::Identity();
    nlohmann::json RawJson;
};

struct RTXPTSceneSettings
{
    bool HasSampleSettings = false;
    bool HasGameSettings   = false;
    nlohmann::json SampleSettingsJson;
    nlohmann::json GameSettingsJson;
};

struct RTXPTSceneAdapterStats
{
    Uint32 ModelAssetCount             = 0;
    Uint32 GraphNodeCount              = 0;
    Uint32 ModelInstanceCount          = 0;
    Uint32 MaterialCount               = 0;
    Uint32 MaterialExtensionCount      = 0;
    Uint32 MaterialFallbackCount       = 0;
    Uint32 DirectionalLightCount       = 0;
    Uint32 PointLightCount             = 0;
    Uint32 SpotLightCount              = 0;
    Uint32 EnvironmentLightCount       = 0;
    Uint32 UnknownTypedNodeCount       = 0;
    Uint32 SkinnedInstanceCount        = 0;
    Uint32 AdapterWarningCount         = 0;
};

struct RTXPTSceneGraphData
{
    std::vector<RTXPTModelAsset>         ModelAssets;
    std::vector<RTXPTGraphNode>          GraphNodes;
    std::vector<RTXPTModelInstance>      ModelInstances;
    std::vector<RTXPTMaterialExtension>  MaterialExtensions;
    std::vector<RTXPTSceneLightMetadata> Lights;
    RTXPTSceneSettings                   Settings;
    RTXPTSceneAdapterStats               Stats;
    std::vector<std::string>             Warnings;

    void Clear();
};

std::string GetRTXPTModelNameFromPath(const std::string& ModelPath);
float4x4 MakeRTXPTNodeTransform(const nlohmann::json& Node);
std::vector<std::string> GetRTXPTMaterialCandidates(const std::string& AssetsRoot,
                                                    const std::string& SceneName,
                                                    const std::string& ModelName,
                                                    const std::string& MaterialName);

} // namespace Diligent
```

- [ ] **Step 5: Create `RTXPTSceneGraph.cpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp` with the utility implementations:

```cpp
#include "RTXPTSceneGraph.hpp"

#include <cmath>

#include "FileSystem.hpp"
#include "RTXPTSceneJson.hpp"

namespace Diligent
{

void RTXPTSceneGraphData::Clear()
{
    ModelAssets.clear();
    GraphNodes.clear();
    ModelInstances.clear();
    MaterialExtensions.clear();
    Lights.clear();
    Settings = {};
    Stats    = {};
    Warnings.clear();
}

std::string GetRTXPTModelNameFromPath(const std::string& ModelPath)
{
    const size_t Slash = ModelPath.find_last_of("/\\");
    const size_t Start = Slash == std::string::npos ? 0 : Slash + 1;
    const size_t Dot   = ModelPath.find_last_of('.');
    return Dot != std::string::npos && Dot > Start ? ModelPath.substr(Start, Dot - Start) : ModelPath.substr(Start);
}

float4x4 MakeRTXPTNodeTransform(const nlohmann::json& Node)
{
    float Translation[3] = {0.0f, 0.0f, 0.0f};
    float Rotation[4]    = {0.0f, 0.0f, 0.0f, 1.0f};
    float Scale[3]       = {1.0f, 1.0f, 1.0f};

    ReadRTXPTFloatArray(Node, "translation", Translation, 3);
    if (!ReadRTXPTFloatArray(Node, "rotation", Rotation, 4))
    {
        float Euler[3] = {0.0f, 0.0f, 0.0f};
        if (ReadRTXPTFloatArray(Node, "euler", Euler, 3))
        {
            QuaternionF Qx{std::sin(Euler[0] * 0.5f), 0.0f, 0.0f, std::cos(Euler[0] * 0.5f)};
            QuaternionF Qy{0.0f, std::sin(Euler[1] * 0.5f), 0.0f, std::cos(Euler[1] * 0.5f)};
            QuaternionF Qz{0.0f, 0.0f, std::sin(Euler[2] * 0.5f), std::cos(Euler[2] * 0.5f)};
            const QuaternionF Q = Qx * Qy * Qz;
            Rotation[0] = Q.q.x;
            Rotation[1] = Q.q.y;
            Rotation[2] = Q.q.z;
            Rotation[3] = Q.q.w;
        }
    }

    const auto ScaleIt = Node.find("scaling");
    if (ScaleIt != Node.end())
    {
        if (ScaleIt->is_number())
        {
            Scale[0] = Scale[1] = Scale[2] = ScaleIt->get<float>();
        }
        else
        {
            ReadRTXPTFloatArray(Node, "scaling", Scale, 3);
        }
    }

    const QuaternionF Q{Rotation[0], Rotation[1], Rotation[2], Rotation[3]};
    return float4x4::Scale(Scale[0], Scale[1], Scale[2]) *
           Q.ToMatrix() *
           float4x4::Translation(Translation[0], Translation[1], Translation[2]);
}

std::vector<std::string> GetRTXPTMaterialCandidates(const std::string& AssetsRoot,
                                                    const std::string& SceneName,
                                                    const std::string& ModelName,
                                                    const std::string& MaterialName)
{
    std::string SceneStem = SceneName;
    const std::string Suffix = ".json";
    if (SceneStem.size() > Suffix.size() &&
        SceneStem.compare(SceneStem.size() - Suffix.size(), Suffix.size(), Suffix) == 0)
    {
        SceneStem.resize(SceneStem.size() - Suffix.size());
    }

    const std::string MaterialsRoot = AssetsRoot + "/Materials";
    return {
        MaterialsRoot + "/" + SceneStem + "/" + ModelName + "." + MaterialName + ".material.json",
        MaterialsRoot + "/" + SceneStem + "/" + MaterialName + ".material.json",
        MaterialsRoot + "/" + ModelName + "." + MaterialName + ".material.json",
        MaterialsRoot + "/" + MaterialName + ".material.json",
    };
}

} // namespace Diligent
```

- [ ] **Step 6: Add helper files to CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new source and include files:

```cmake
set(SOURCE
    src/RTXPTSample.cpp
    src/RTXPTScene.cpp
    src/RTXPTSceneJson.cpp
    src/RTXPTSceneGraph.cpp
    src/RTXPTMaterials.cpp
    src/RTXPTLights.cpp
    src/RTXPTAccelerationStructures.cpp
    src/RTXPTSkinnedGeometry.cpp
    src/RTXPTRenderTargets.cpp
    src/RTXPTRayTracingPass.cpp
    src/RTXPTComputePass.cpp
    src/RTXPTBlitPass.cpp
)

set(INCLUDE
    src/RTXPTSample.hpp
    src/RTXPTScene.hpp
    src/RTXPTSceneJson.hpp
    src/RTXPTSceneGraph.hpp
    src/RTXPTMaterials.hpp
    src/RTXPTLights.hpp
    src/RTXPTAccelerationStructures.hpp
    src/RTXPTSkinnedGeometry.hpp
    src/RTXPTRenderTargets.hpp
    src/RTXPTRayTracingPass.hpp
    src/RTXPTComputePass.hpp
    src/RTXPTBlitPass.hpp
    RTXPT_FORK_MAPPING.md
)
```

- [ ] **Step 7: Build-check the scaffold**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: either the RTXPT target builds, or the command reports the local build tree is missing. If the build tree is missing, record that exact limitation in the task handoff and run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTSceneJson.hpp Samples/RTXPT/src/RTXPTSceneJson.cpp Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp
```

Expected: no whitespace errors.

- [ ] **Step 8: Commit scaffold**

Run from the umbrella root:

```powershell
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTSceneJson.hpp Samples/RTXPT/src/RTXPTSceneJson.cpp Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add scene graph adapter scaffolding" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Replace `models[0]` Loading With Full Scene Graph Loading

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`

- [ ] **Step 1: Add scene graph ownership to `RTXPTScene`**

In `RTXPTScene.hpp`, include the graph header and add read-only scene-view accessors:

```cpp
#include "RTXPTSceneGraph.hpp"
```

Add these public methods near the existing diagnostics getters:

```cpp
    const RTXPTSceneGraphData& GetSceneGraphData() const { return m_SceneGraph; }
    const RTXPTSceneAdapterStats& GetAdapterStats() const { return m_SceneGraph.Stats; }
    Uint32 GetModelAssetCount() const { return static_cast<Uint32>(m_SceneGraph.ModelAssets.size()); }
    Uint32 GetModelInstanceCount() const { return static_cast<Uint32>(m_SceneGraph.ModelInstances.size()); }
```

Add this private member:

```cpp
    RTXPTSceneGraphData          m_SceneGraph;
```

- [ ] **Step 2: Replace single-model reset with graph reset**

In `RTXPTScene::ResetLoadedData()`, reset the new graph data and keep old fields clear during migration:

```cpp
void RTXPTScene::ResetLoadedData()
{
    m_Model.reset();
    m_Transforms = {};
    m_Cameras.clear();
    m_LoadedSceneName.clear();
    m_AssetsRoot.clear();
    m_ModelPath.clear();
    m_IndexType      = VT_UINT32;
    m_SceneIndex     = 0;
    m_MeshNodeCount  = 0;
    m_PrimitiveCount = 0;
    m_MaterialCount  = 0;
    m_LightCount     = 0;
    m_VertexStride0  = 0;
    m_GeometryStats  = {};
    m_AnimationTime  = 0.0f;
    m_AnimationIndex = -1;
    m_GeometryDirty  = false;
    m_SceneGraph.Clear();
}
```

- [ ] **Step 3: Add model creation helper**

In `RTXPTScene.cpp`, add a helper that creates one `GLTF::Model` with the same bind flags as the current loader:

```cpp
std::unique_ptr<GLTF::Model> LoadRTXPTModelAsset(IRenderDevice*     pDevice,
                                                 IDeviceContext*    pContext,
                                                 const std::string& ModelPath,
                                                 VALUE_TYPE         IndexType)
{
    GLTF::ModelCreateInfo ModelCI;
    ModelCI.FileName             = ModelPath.c_str();
    ModelCI.ComputeBoundingBoxes = true;
    ModelCI.IndexType            = IndexType;
    ModelCI.IndBufferBindFlags   = BIND_INDEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;
    for (BIND_FLAGS& BindFlags : ModelCI.VertBufferBindFlags)
        BindFlags = BIND_VERTEX_BUFFER | BIND_RAY_TRACING;
    ModelCI.VertBufferBindFlags[0] = BIND_VERTEX_BUFFER | BIND_RAY_TRACING | BIND_SHADER_RESOURCE;
    ModelCI.VertBufferBindFlags[1] = BIND_VERTEX_BUFFER | BIND_SHADER_RESOURCE;

    return std::make_unique<GLTF::Model>(pDevice, pContext, ModelCI);
}
```

- [ ] **Step 4: Add recursive graph parser**

In `RTXPTScene.cpp`, add a member or anonymous-namespace helper with this signature:

```cpp
bool AppendRTXPTGraphNode(RTXPTSceneGraphData& Data,
                          const nlohmann::json& NodeJson,
                          RTXPTSceneId ParentId,
                          const float4x4& ParentTransform)
```

The function must:

```cpp
RTXPTGraphNode Node;
Node.Name            = ReadRTXPTOptionalString(NodeJson, "name", "Node");
Node.Type            = ReadRTXPTOptionalString(NodeJson, "type", "");
Node.ParentId        = ParentId;
Node.LocalTransform  = MakeRTXPTNodeTransform(NodeJson);
Node.GlobalTransform = Node.LocalTransform * ParentTransform;
Node.RawMetadata     = NodeJson;

const auto ModelIt = NodeJson.find("model");
if (ModelIt != NodeJson.end() && ModelIt->is_number_unsigned())
    Node.ModelAssetId = ModelIt->get<Uint32>();
```

After pushing the node, create a model instance when `ModelAssetId` is valid and parse children:

```cpp
const RTXPTSceneId NodeId = static_cast<RTXPTSceneId>(Data.GraphNodes.size());
Data.GraphNodes.emplace_back(std::move(Node));
if (ParentId != InvalidRTXPTSceneId)
    Data.GraphNodes[ParentId].Children.push_back(NodeId);

if (Data.GraphNodes[NodeId].ModelAssetId != InvalidRTXPTSceneId)
{
    RTXPTModelInstance Instance;
    Instance.GraphNodeId     = NodeId;
    Instance.ModelAssetId    = Data.GraphNodes[NodeId].ModelAssetId;
    Instance.Name            = Data.GraphNodes[NodeId].Name;
    Instance.GlobalTransform = Data.GraphNodes[NodeId].GlobalTransform;
    Data.ModelInstances.emplace_back(std::move(Instance));
}

const auto ChildrenIt = NodeJson.find("children");
if (ChildrenIt != NodeJson.end() && ChildrenIt->is_array())
{
    for (const auto& Child : *ChildrenIt)
        AppendRTXPTGraphNode(Data, Child, NodeId, Data.GraphNodes[NodeId].GlobalTransform);
}
```

Do not add final typed metadata in this task. Task 3 owns light, settings, camera metadata, and unknown-type diagnostics after the basic recursive graph is loading.

- [ ] **Step 5: Implement full `LoadScene` transaction**

Replace the model path part of `RTXPTScene::LoadScene(...)` with a temporary graph transaction:

```cpp
RTXPTJsonLoadResult SceneJsonResult;
if (!LoadRTXPTRelaxedJsonFile(ScenePath, SceneJsonResult) || !SceneJsonResult.Json.is_object())
{
    LOG_ERROR_MESSAGE("Invalid scene JSON: ", ScenePath);
    return false;
}

const auto ModelsIt = SceneJsonResult.Json.find("models");
const auto GraphIt  = SceneJsonResult.Json.find("graph");
if (ModelsIt == SceneJsonResult.Json.end() || !ModelsIt->is_array() ||
    GraphIt == SceneJsonResult.Json.end() || !GraphIt->is_array())
{
    LOG_ERROR_MESSAGE("Scene JSON requires models and graph arrays: ", ScenePath);
    return false;
}

RTXPTSceneGraphData NewData;
for (const auto& ModelJson : *ModelsIt)
{
    if (!ModelJson.is_string())
    {
        LOG_ERROR_MESSAGE("Scene JSON model entry is not a string: ", ScenePath);
        return false;
    }

    RTXPTModelAsset Asset;
    Asset.RelativePath = ModelJson.get<std::string>();
    Asset.ResolvedPath = JoinPath(m_AssetsRoot, Asset.RelativePath.c_str());
    Asset.ModelName    = GetRTXPTModelNameFromPath(Asset.RelativePath);
    Asset.Model        = LoadRTXPTModelAsset(pDevice, pContext, Asset.ResolvedPath, m_IndexType);
    if (!Asset.Model)
    {
        LOG_ERROR_MESSAGE("Failed to load RTXPT glTF model: ", Asset.ResolvedPath);
        return false;
    }
    Asset.SceneIndex = static_cast<Uint32>(Asset.Model->DefaultSceneId >= 0 ? Asset.Model->DefaultSceneId : 0);
    Asset.Model->ComputeTransforms(Asset.SceneIndex, Asset.StaticTransforms);
    NewData.ModelAssets.emplace_back(std::move(Asset));
}

for (const auto& NodeJson : *GraphIt)
    AppendRTXPTGraphNode(NewData, NodeJson, InvalidRTXPTSceneId, float4x4::Identity());
```

Before committing, validate every `ModelInstance.ModelAssetId`:

```cpp
for (const RTXPTModelInstance& Instance : NewData.ModelInstances)
{
    if (Instance.ModelAssetId >= NewData.ModelAssets.size())
    {
        LOG_ERROR_MESSAGE("Scene graph model index is out of range in scene: ", ScenePath);
        return false;
    }
}

NewData.Stats.ModelAssetCount    = static_cast<Uint32>(NewData.ModelAssets.size());
NewData.Stats.GraphNodeCount     = static_cast<Uint32>(NewData.GraphNodes.size());
NewData.Stats.ModelInstanceCount = static_cast<Uint32>(NewData.ModelInstances.size());
m_SceneGraph                     = std::move(NewData);
m_LoadedSceneName                = SceneName;
```

- [ ] **Step 6: Keep camera loading working**

Update `AppendSceneCameras(...)` so it accepts both camera type strings:

```cpp
const bool IsPerspectiveCamera =
    TypeIt != Node.end() && TypeIt->is_string() &&
    (TypeIt->get<std::string>() == "PerspectiveCamera" ||
     TypeIt->get<std::string>() == "PerspectiveCameraEx");
```

Use `LoadRTXPTRelaxedJsonFile(...)` inside `LoadSceneCameras(...)` so `living-room.scene.json` succeeds.

- [ ] **Step 7: Verify parsing smoke through runtime scene load**

Build:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Manual smoke after launching RTXPT:

- select `living-room.scene.json`; expected: scene loads, no strict JSON parse failure blocks it
- select `kitchen.scene.json`; expected diagnostics show `Model assets >= 6` and `Model instances >= 8`
- select `bistro-programmer-art.scene.json`; expected diagnostics show more than one model asset

- [ ] **Step 8: Commit scene graph loading**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTSceneJson.cpp
git -C DiligentSamples commit -m "feat(rtxpt): load full scene json graph" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Parse Material Extensions, Lights, Settings, And Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`

- [ ] **Step 1: Add material extension fields needed by GPU upload**

Extend `RTXPTMaterialExtension` in `RTXPTSceneGraph.hpp`:

```cpp
    float4 BaseColorFactor = float4{1, 1, 1, 1};
    float3 EmissiveFactor  = float3{0, 0, 0};
    float  EmissiveIntensity = 1.0f;
    float  MetallicFactor  = 1.0f;
    float  RoughnessFactor = 1.0f;
    float  AlphaCutoff     = 0.5f;
    bool   EnableAlphaTesting = false;
    bool   EnableBaseTexture = true;
    bool   EnableEmissiveTexture = true;
    bool   EnableNormalTexture = true;
    bool   EnableOcclusionRoughnessMetallicTexture = true;
    bool   EnableTransmission = false;
    float  TransmissionFactor = 0.0f;
    float  IoR = 1.5f;
    bool   ThinSurface = false;
    bool   SkipRender = false;
```

- [ ] **Step 2: Add extension parsing helper**

In `RTXPTSceneGraph.cpp`, add:

```cpp
RTXPTMaterialExtension ParseRTXPTMaterialExtension(const std::string& FilePath,
                                                   const std::string& ModelName,
                                                   const std::string& MaterialName,
                                                   const nlohmann::json& Json)
{
    RTXPTMaterialExtension Ext;
    Ext.FilePath      = FilePath;
    Ext.ModelName     = ModelName;
    Ext.MaterialName  = MaterialName;
    Ext.Loaded        = true;
    Ext.RawJson       = Json;

    float BaseColor[3] = {1, 1, 1};
    if (ReadRTXPTFloatArray(Json, "BaseOrDiffuseColor", BaseColor, 3))
        Ext.BaseColorFactor = float4{BaseColor[0], BaseColor[1], BaseColor[2], ReadRTXPTOptionalFloat(Json, "Opacity", 1.0f)};

    float Emissive[3] = {0, 0, 0};
    if (ReadRTXPTFloatArray(Json, "EmissiveColor", Emissive, 3))
    {
        const float EmissiveIntensity = ReadRTXPTOptionalFloat(Json, "EmissiveIntensity", 1.0f);
        Ext.EmissiveFactor            = float3{Emissive[0] * EmissiveIntensity,
                                                Emissive[1] * EmissiveIntensity,
                                                Emissive[2] * EmissiveIntensity};
    }

    Ext.MetallicFactor  = ReadRTXPTOptionalFloat(Json, "Metalness", Ext.MetallicFactor);
    Ext.RoughnessFactor = ReadRTXPTOptionalFloat(Json, "Roughness", Ext.RoughnessFactor);
    Ext.AlphaCutoff     = ReadRTXPTOptionalFloat(Json, "AlphaCutoff", Ext.AlphaCutoff);
    Ext.TransmissionFactor = ReadRTXPTOptionalFloat(Json, "TransmissionFactor", Ext.TransmissionFactor);
    Ext.IoR = ReadRTXPTOptionalFloat(Json, "IoR", Ext.IoR);

    Ext.EnableAlphaTesting = Json.value("EnableAlphaTesting", Ext.EnableAlphaTesting);
    Ext.EnableBaseTexture = Json.value("EnableBaseTexture", Ext.EnableBaseTexture);
    Ext.EnableEmissiveTexture = Json.value("EnableEmissiveTexture", Ext.EnableEmissiveTexture);
    Ext.EnableNormalTexture = Json.value("EnableNormalTexture", Ext.EnableNormalTexture);
    Ext.EnableOcclusionRoughnessMetallicTexture =
        Json.value("EnableOcclusionRoughnessMetallicTexture", Ext.EnableOcclusionRoughnessMetallicTexture);
    Ext.EnableTransmission = Json.value("EnableTransmission", Ext.EnableTransmission);
    Ext.ThinSurface = Json.value("ThinSurface", Ext.ThinSurface);
    Ext.SkipRender = Json.value("SkipRender", Ext.SkipRender);
    return Ext;
}
```

Declare this helper in `RTXPTSceneGraph.hpp`.

- [ ] **Step 3: Load material extension candidates**

In `RTXPTScene::LoadScene(...)`, after model assets load and before final stats, loop every model material:

```cpp
for (RTXPTModelAsset& Asset : NewData.ModelAssets)
{
    Asset.MaterialRemap.resize(Asset.Model->Materials.size(), 0);
    for (Uint32 MatIdx = 0; MatIdx < Asset.Model->Materials.size(); ++MatIdx)
    {
        const GLTF::Material& Material = Asset.Model->Materials[MatIdx];
        const std::string MaterialName = Material.Name.empty() ? ("material_" + std::to_string(MatIdx)) : Material.Name;
        RTXPTMaterialExtension Ext;
        Ext.ModelName    = Asset.ModelName;
        Ext.MaterialName = MaterialName;

        for (const std::string& Candidate : GetRTXPTMaterialCandidates(m_AssetsRoot, SceneName, Asset.ModelName, MaterialName))
        {
            if (!FileSystem::FileExists(Candidate.c_str()))
                continue;

            RTXPTJsonLoadResult MaterialJson;
            if (LoadRTXPTRelaxedJsonFile(Candidate, MaterialJson) && MaterialJson.Json.is_object())
            {
                Ext = ParseRTXPTMaterialExtension(Candidate, Asset.ModelName, MaterialName, MaterialJson.Json);
                break;
            }
        }

        Asset.MaterialRemap[MatIdx] = static_cast<Uint32>(NewData.MaterialExtensions.size());
        NewData.MaterialExtensions.emplace_back(std::move(Ext));
    }
}
```

Update counts:

```cpp
NewData.Stats.MaterialCount = static_cast<Uint32>(NewData.MaterialExtensions.size());
for (const RTXPTMaterialExtension& Ext : NewData.MaterialExtensions)
{
    if (Ext.Loaded)
        ++NewData.Stats.MaterialExtensionCount;
    else
        ++NewData.Stats.MaterialFallbackCount;
}
```

- [ ] **Step 4: Preserve typed light and settings metadata**

In the graph parser, when `Type` is a light, push a `RTXPTSceneLightMetadata`:

```cpp
if (Node.Type == "DirectionalLight" || Node.Type == "PointLight" ||
    Node.Type == "SpotLight" || Node.Type == "EnvironmentLight")
{
    RTXPTSceneLightMetadata Light;
    Light.Name            = Node.Name;
    Light.Type            = Node.Type;
    Light.GlobalTransform = Node.GlobalTransform;
    Light.RawJson         = NodeJson;
    Data.Lights.emplace_back(std::move(Light));
}
else if (Node.Type == "SampleSettings")
{
    Data.Settings.HasSampleSettings = true;
    Data.Settings.SampleSettingsJson = NodeJson;
}
else if (Node.Type == "GameSettings")
{
    Data.Settings.HasGameSettings = true;
    Data.Settings.GameSettingsJson = NodeJson;
}
else if (!Node.Type.empty() && Node.Type != "PerspectiveCamera" && Node.Type != "PerspectiveCameraEx")
{
    ++Data.Stats.UnknownTypedNodeCount;
}
```

- [ ] **Step 5: Verify material and metadata diagnostics in UI**

Temporarily add these `ImGui::Text` lines in the Scene section of `RTXPTSample::UpdateUI()` if Task 6 has not yet added the final UI:

```cpp
const RTXPTSceneAdapterStats& AdapterStats = m_Scene.GetAdapterStats();
ImGui::Text("Model assets: %u", AdapterStats.ModelAssetCount);
ImGui::Text("Model instances: %u", AdapterStats.ModelInstanceCount);
ImGui::Text("Material extensions: %u", AdapterStats.MaterialExtensionCount);
ImGui::Text("Material fallbacks: %u", AdapterStats.MaterialFallbackCount);
ImGui::Text("Environment lights: %u", AdapterStats.EnvironmentLightCount);
```

Run RTXPT and switch to `bistro-programmer-art.scene.json`.

Expected:

- material extensions loaded count is greater than zero
- model asset count is greater than one
- environment light count is at least one

- [ ] **Step 6: Commit metadata parsing**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTScene.cpp Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): parse scene metadata extensions" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Upload Global Materials And Lights From The Scene View

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add scene-view overloads**

In `RTXPTMaterials.hpp`, forward declare `RTXPTSceneGraphData` and add:

```cpp
struct RTXPTSceneGraphData;

class RTXPTMaterials
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Model& Model);
    bool Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData);
```

In `RTXPTLights.hpp`, forward declare and add:

```cpp
struct RTXPTSceneGraphData;

class RTXPTLights
{
public:
    void Reset();
    bool Upload(IRenderDevice* pDevice, const GLTF::Scene& Scene, const GLTF::ModelTransforms& Transforms);
    bool Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData);
```

- [ ] **Step 2: Implement global material upload**

In `RTXPTMaterials.cpp`, add `#include "RTXPTSceneGraph.hpp"`, keep the old overload, and make the scene overload build one global texture table followed by one global material table:

```cpp
bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData)
{
    Reset();

    constexpr Uint32 InvalidTextureIndex = ~Uint32{0};

    std::vector<std::vector<Uint32>> TextureRemaps(SceneData.ModelAssets.size());
    for (Uint32 AssetIdx = 0; AssetIdx < SceneData.ModelAssets.size(); ++AssetIdx)
    {
        const RTXPTModelAsset& Asset = SceneData.ModelAssets[AssetIdx];
        if (!Asset.Model)
            continue;

        const Uint32 TextureCount = static_cast<Uint32>(Asset.Model->GetTextureCount());
        TextureRemaps[AssetIdx].assign(TextureCount, InvalidTextureIndex);
        for (Uint32 TexIdx = 0; TexIdx < TextureCount; ++TexIdx)
        {
            ITexture* pTexture = Asset.Model->GetTexture(TexIdx);
            if (pTexture == nullptr)
                continue;

            TextureViewDesc ViewDesc;
            ViewDesc.ViewType   = TEXTURE_VIEW_SHADER_RESOURCE;
            ViewDesc.TextureDim = RESOURCE_DIM_TEX_2D_ARRAY;

            RefCntAutoPtr<ITextureView> pSRV;
            pTexture->CreateView(ViewDesc, &pSRV);
            if (!pSRV)
                continue;

            TextureRemaps[AssetIdx][TexIdx] = static_cast<Uint32>(m_TextureBindings.size());
            m_TextureViews.emplace_back(std::move(pSRV));
            m_TextureBindings.push_back(m_TextureViews.back().RawPtr<IDeviceObject>());
        }
    }
    m_Stats.TextureCount = static_cast<Uint32>(m_TextureBindings.size());

    std::vector<MaterialPTData> MaterialData;
    for (Uint32 AssetIdx = 0; AssetIdx < SceneData.ModelAssets.size(); ++AssetIdx)
    {
        const RTXPTModelAsset& Asset = SceneData.ModelAssets[AssetIdx];
        if (!Asset.Model)
            continue;

        for (Uint32 MatIdx = 0; MatIdx < Asset.Model->Materials.size(); ++MatIdx)
        {
            const GLTF::Material& Material = Asset.Model->Materials[MatIdx];
            MaterialPTData Data;
            FillMaterialPTDataFromGLTF(Material, Data);
            RemapMaterialTextureIndices(Data, TextureRemaps[AssetIdx]);

            if (MatIdx < Asset.MaterialRemap.size() && Asset.MaterialRemap[MatIdx] < SceneData.MaterialExtensions.size())
            {
                const RTXPTMaterialExtension& Ext = SceneData.MaterialExtensions[Asset.MaterialRemap[MatIdx]];
                if (Ext.Loaded)
                {
                    Data.baseColorFactor = Ext.BaseColorFactor;
                    Data.emissiveFactor  = Ext.EmissiveFactor;
                    Data.alphaCutoff     = Ext.AlphaCutoff;
                    Data.metallicFactor  = Ext.MetallicFactor;
                    Data.roughnessFactor = Ext.RoughnessFactor;
                    if (!Ext.EnableBaseTexture)
                        Data.flags &= ~(kMaterialFlag_HasBaseColorTexture | kMaterialFlag_AlphaTested);
                    if (!Ext.EnableEmissiveTexture)
                        Data.flags &= ~kMaterialFlag_HasEmissiveTexture;
                    if (!Ext.EnableNormalTexture)
                        Data.flags &= ~kMaterialFlag_HasNormalTexture;
                    if (!Ext.EnableOcclusionRoughnessMetallicTexture)
                        Data.flags &= ~kMaterialFlag_HasMetallicRoughnessTexture;
                    if (Ext.EnableAlphaTesting)
                        Data.flags |= kMaterialFlag_AlphaTested;
                }
            }

            MaterialData.emplace_back(Data);
        }
    }

    if (MaterialData.empty())
        MaterialData.emplace_back();

    m_Stats.MaterialCount = static_cast<Uint32>(MaterialData.size());
    return CreateMaterialBuffer(pDevice, MaterialData);
}
```

During this step, extract existing inline material filling and buffer creation into private helpers, and add the texture-remap helper used above:

```cpp
void FillMaterialPTDataFromGLTF(const GLTF::Material& Material, MaterialPTData& Data);
void RemapMaterialTextureIndices(MaterialPTData& Data, const std::vector<Uint32>& TextureRemap);
bool CreateMaterialBuffer(IRenderDevice* pDevice, const std::vector<MaterialPTData>& MaterialData);
```

- [ ] **Step 3: Implement texture remapping helper**

Add this helper in the anonymous namespace of `RTXPTMaterials.cpp` so every material texture index becomes a global texture-table index:

```cpp
void RemapTextureIndex(Uint32 Flag, Uint32& Flags, Uint32& TextureIndex, const std::vector<Uint32>& TextureRemap)
{
    if ((Flags & Flag) == 0)
        return;

    if (TextureIndex >= TextureRemap.size() || TextureRemap[TextureIndex] == ~Uint32{0})
    {
        Flags &= ~Flag;
        TextureIndex = 0;
        return;
    }

    TextureIndex = TextureRemap[TextureIndex];
}

void RemapMaterialTextureIndices(MaterialPTData& Data, const std::vector<Uint32>& TextureRemap)
{
    RemapTextureIndex(kMaterialFlag_HasBaseColorTexture, Data.flags, Data.baseColorTextureIndex, TextureRemap);
    RemapTextureIndex(kMaterialFlag_HasEmissiveTexture, Data.flags, Data.emissiveTextureIndex, TextureRemap);
    RemapTextureIndex(kMaterialFlag_HasMetallicRoughnessTexture, Data.flags, Data.metallicRoughnessTextureIndex, TextureRemap);
    RemapTextureIndex(kMaterialFlag_HasNormalTexture, Data.flags, Data.normalTextureIndex, TextureRemap);

    if ((Data.flags & kMaterialFlag_HasBaseColorTexture) == 0)
        Data.flags &= ~kMaterialFlag_AlphaTested;
}
```

- [ ] **Step 4: Implement graph light upload**

In `RTXPTLights.cpp`, implement a scene overload:

```cpp
bool RTXPTLights::Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData)
{
    Reset();

    std::vector<PolymorphicLightInfo> Lights;
    for (const RTXPTSceneLightMetadata& LightMeta : SceneData.Lights)
    {
        if (LightMeta.Type == "EnvironmentLight")
            continue;

        PolymorphicLightInfo Light;
        Light.colorIntensity = float4{1, 1, 1, ReadRTXPTOptionalFloat(LightMeta.RawJson, "intensity", 1.0f)};
        Light.positionRange  = float4{LightMeta.GlobalTransform._41,
                                      LightMeta.GlobalTransform._42,
                                      LightMeta.GlobalTransform._43,
                                      ReadRTXPTOptionalFloat(LightMeta.RawJson, "range", 0.0f)};
        Light.directionType  = float4{-LightMeta.GlobalTransform._31,
                                      -LightMeta.GlobalTransform._32,
                                      -LightMeta.GlobalTransform._33,
                                      0.0f};
        if (LightMeta.Type == "DirectionalLight")
            Light.directionType.w = 0.0f;
        else if (LightMeta.Type == "PointLight")
            Light.directionType.w = 1.0f;
        else if (LightMeta.Type == "SpotLight")
            Light.directionType.w = 2.0f;
        else
            continue;

        Lights.emplace_back(Light);
    }

    return UploadLightBuffer(pDevice, Lights);
}
```

Extract the existing light buffer creation code into:

```cpp
bool UploadLightBuffer(IRenderDevice* pDevice, std::vector<PolymorphicLightInfo>& Lights);
```

- [ ] **Step 5: Update sample rebuild to call scene overloads**

In `RTXPTSample::RebuildSceneDependentResources()`, replace:

```cpp
ResourcesReady &= m_Materials.Upload(m_pDevice, *pModel);
```

with:

```cpp
const RTXPTSceneGraphData& SceneData = m_Scene.GetSceneGraphData();
ResourcesReady &= m_Materials.Upload(m_pDevice, SceneData);
ResourcesReady &= m_Lights.Upload(m_pDevice, SceneData);
```

Leave AS and skinning on the old path until Task 5 migrates them.

- [ ] **Step 6: Build and commit materials/lights**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: build succeeds or reports a pre-existing build-tree limitation; diff check succeeds.

Commit:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): upload scene graph materials and lights" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Add Scene-Level Skinning And Per-Instance Animation

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Rename the class in headers first**

In `RTXPTSkinnedGeometry.hpp`, rename the class and stats types while keeping the file name unchanged:

```cpp
#include "RTXPTSceneGraph.hpp"

struct RTXPTSkinnedSceneAssetBinding
{
    RTXPTSceneId                    ModelAssetId = InvalidRTXPTSceneId;
    IBuffer*                        pSourceVertexBuffer = nullptr;
    IBuffer*                        pSourceSkinBuffer   = nullptr;
    RefCntAutoPtr<IShaderResourceBinding> pSRB;
};

struct RTXPTSkinnedSceneGeometryStats
{
    bool        Ready                = false;
    bool        LastDispatchExecuted = false;
    Uint32      SkinnedInstanceCount = 0;
    Uint32      SkinningJobCount     = 0;
    Uint32      SkinnedVertexCount   = 0;
    Uint32      JointMatrixCount     = 0;
    Uint32      DispatchCount        = 0;
    std::string DisabledReason;
};

class RTXPTSkinnedSceneGeometry
{
public:
    void Reset();
```

Update `RTXPTSample.hpp`:

```cpp
    RTXPTSkinnedSceneGeometry    m_SkinnedGeometry;
```

Add these new private members to the scene geometry class:

```cpp
    std::vector<RTXPTSkinnedSceneAssetBinding> m_AssetBindings;
```

- [ ] **Step 2: Add scene-level skinning records**

In `RTXPTSkinnedGeometry.hpp`, replace `RTXPTSkinnedNodeGeometry` with scene identifiers:

```cpp
struct RTXPTSkinnedSceneNodeGeometry
{
    RTXPTSceneId ModelAssetId      = InvalidRTXPTSceneId;
    RTXPTSceneId ModelInstanceId   = InvalidRTXPTSceneId;
    const GLTF::Node* pNode        = nullptr;
    Uint32 SourceVertexBase        = 0;
    Uint32 VertexBase              = 0;
    Uint32 VertexCount             = 0;
    Uint32 JointBase               = 0;
    Uint32 JointCount              = 0;
};
```

Add a public lookup:

```cpp
    const RTXPTSkinnedSceneNodeGeometry* FindNode(RTXPTSceneId ModelAssetId,
                                                  RTXPTSceneId ModelInstanceId,
                                                  const GLTF::Node* pNode) const;
```

- [ ] **Step 3: Initialize from scene data**

Replace the old `Initialize(...)` signature with:

```cpp
bool Initialize(IRenderDevice*                pDevice,
                IEngineFactory*               pEngineFactory,
                const RTXPTSceneGraphData&    SceneData,
                bool                          ComputeSupported);
```

The new build-node-table loop must traverse every `ModelInstance`:

```cpp
for (Uint32 InstanceId = 0; InstanceId < SceneData.ModelInstances.size(); ++InstanceId)
{
    const RTXPTModelInstance& Instance = SceneData.ModelInstances[InstanceId];
    const RTXPTModelAsset& Asset = SceneData.ModelAssets[Instance.ModelAssetId];
    const GLTF::Scene& Scene = Asset.Model->Scenes[Asset.SceneIndex];
    const Uint32 SourceVertexBase = Asset.Model->GetBaseVertex();
    const IBuffer* pVertexBuffer = Asset.Model->GetVertexBufferCount() > 0 ? Asset.Model->GetVertexBuffer(0) : nullptr;
    const Uint64 SourceVertexOffset = Uint64{SourceVertexBase} * sizeof(RTXPTGeometryVertex);
    const Uint32 ModelVertexCount = pVertexBuffer != nullptr && pVertexBuffer->GetDesc().Size > SourceVertexOffset ?
        static_cast<Uint32>((pVertexBuffer->GetDesc().Size - SourceVertexOffset) / sizeof(RTXPTGeometryVertex)) :
        0;

    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr || pNode->pSkin == nullptr)
            continue;

        RTXPTSkinnedSceneNodeGeometry Record;
        Record.ModelAssetId    = Instance.ModelAssetId;
        Record.ModelInstanceId = InstanceId;
        Record.pNode           = pNode;
        Record.SourceVertexBase = SourceVertexBase;
        Record.VertexBase       = VertexBase;
        Record.VertexCount      = ModelVertexCount;
        Record.JointBase        = JointBase;
        Record.JointCount       = static_cast<Uint32>(pNode->pSkin->Joints.size());
        m_Nodes.push_back(Record);

        VertexBase += Record.VertexCount;
        JointBase += Record.JointCount;
    }
}
```

After `CreatePipeline(...)` succeeds, create one SRB per `ModelAssetId` so each dispatch can bind the correct source vertex and skin buffers while sharing the same joint matrix and skinned output buffers:

```cpp
bool CreateAssetBindings(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData);
```

- [ ] **Step 4: Add per-instance transform update in `RTXPTScene`**

In `RTXPTScene::Update(...)`, replace the single animation update with a loop:

```cpp
for (RTXPTModelInstance& Instance : m_SceneGraph.ModelInstances)
{
    RTXPTModelAsset& Asset = m_SceneGraph.ModelAssets[Instance.ModelAssetId];
    const bool HasAnimation = Asset.Model && !Asset.Model->Animations.empty();
    if (!HasAnimation)
    {
        Instance.Transforms = Asset.StaticTransforms;
        continue;
    }

    if (Instance.Animation.AnimationIndex < 0)
        Instance.Animation.AnimationIndex = 0;

    if (Instance.Animation.AnimationIndex < 0 || static_cast<Uint32>(Instance.Animation.AnimationIndex) >= Asset.Model->Animations.size())
        Instance.Animation.AnimationIndex = 0;

    const GLTF::Animation& Animation = Asset.Model->Animations[static_cast<Uint32>(Instance.Animation.AnimationIndex)];
    const float Duration = std::max(Animation.End - Animation.Start, 1e-5f);
    Instance.Animation.Time += static_cast<float>(ElapsedTime) * Instance.Animation.PlaySpeed;
    const float WrappedTime = Animation.Start + std::fmod(Instance.Animation.Time + Instance.Animation.TimeOffset, Duration);
    Asset.Model->ComputeTransforms(Asset.SceneIndex,
                                   Instance.Transforms,
                                   Instance.GlobalTransform,
                                   Instance.Animation.AnimationIndex,
                                   WrappedTime);
    m_GeometryDirty = true;
}
```

- [ ] **Step 5: Update scene-level joint matrix upload**

Change `UploadJointMatrices(...)` to accept `RTXPTSceneGraphData` and read transforms from each instance:

```cpp
const RTXPTModelInstance& Instance = SceneData.ModelInstances[Node.ModelInstanceId];
const GLTF::ModelTransforms& Transforms = Instance.Transforms;
```

Use `Node.JointBase + i` as the global destination index exactly as the old path did for one model.

- [ ] **Step 6: Rebind source buffers per job**

Because different model assets have different source vertex and skin buffers, make `Update(...)` bind source SRVs before each dispatch by looking up the asset binding for the current `ModelAssetId`:

```cpp
const RTXPTSkinnedSceneAssetBinding& Binding = m_AssetBindings[Node.ModelAssetId];
pContext->CommitShaderResources(Binding.pSRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
```

Each dispatch writes to the shared `m_SkinnedVertexBuffer` at `Node.VertexBase`. The `CreateAssetBindings(...)` helper binds `Binding.pSourceVertexBuffer`, `Binding.pSourceSkinBuffer`, `m_JointMatrixBuffer`, and `m_SkinnedVertexBuffer` once per asset.

- [ ] **Step 7: Build and commit scene-level skinning**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
```

Commit:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp Samples/RTXPT/src/RTXPTScene.hpp Samples/RTXPT/src/RTXPTScene.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): support scene-level skinned animation" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Build Acceleration Structures Across All Model Instances

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add scene-view AS signatures**

In `RTXPTAccelerationStructures.hpp`, replace the primary scene build API with:

```cpp
bool BuildScene(IRenderDevice*                    pDevice,
                IDeviceContext*                   pContext,
                const RTXPTSceneGraphData&        SceneData,
                VALUE_TYPE                        IndexType,
                const RTXPTSkinnedSceneGeometry*  pSkinnedGeometry,
                bool                              RayTracingSupported);

bool UpdateDynamicBLAS(IDeviceContext*                    pContext,
                       const RTXPTSceneGraphData&          SceneData,
                       const RTXPTSkinnedSceneGeometry&    SkinnedGeometry);
```

Keep the old single-model helper only if another call site still needs it during migration.

- [ ] **Step 2: Extend BLAS records with scene ids**

In the private `BLASRecord`, add:

```cpp
RTXPTSceneId ModelAssetId    = InvalidRTXPTSceneId;
RTXPTSceneId ModelInstanceId = InvalidRTXPTSceneId;
const GLTF::Node* pNode      = nullptr;
```

- [ ] **Step 3: Traverse every model instance**

In `RTXPTAccelerationStructures::BuildScene(...)`, replace the single-scene loop with:

```cpp
for (Uint32 InstanceId = 0; InstanceId < SceneData.ModelInstances.size(); ++InstanceId)
{
    const RTXPTModelInstance& Instance = SceneData.ModelInstances[InstanceId];
    if (Instance.ModelAssetId >= SceneData.ModelAssets.size())
        return false;

    const RTXPTModelAsset& Asset = SceneData.ModelAssets[Instance.ModelAssetId];
    if (!Asset.Model)
        return false;

    const GLTF::Model& Model = *Asset.Model;
    const GLTF::Scene& Scene = Model.Scenes[Asset.SceneIndex];
    const GLTF::ModelTransforms& Transforms = Instance.Transforms.NodeGlobalMatrices.empty() ?
        Asset.StaticTransforms :
        Instance.Transforms;

    for (const GLTF::Node* pNode : Scene.LinearNodes)
    {
        if (pNode == nullptr || pNode->pMesh == nullptr)
            continue;

        if (pNode->Index < 0 || static_cast<size_t>(pNode->Index) >= Transforms.NodeGlobalMatrices.size())
            continue;

        const float4x4 NodeGlobal = Transforms.NodeGlobalMatrices[pNode->Index];
        const float4x4 InstanceNodeGlobal = NodeGlobal;
        // Move the current per-node primitive loop into a private helper in `RTXPTAccelerationStructures.cpp`
        // that appends BLAS records, TLAS instances, and sub-instance data for one mesh node.
        // The helper should take the asset, instance, node, and per-node global transform as inputs.
    }
}
```

Move the current per-node primitive loop into a private helper in `RTXPTAccelerationStructures.cpp` that takes the asset, instance, node, and per-node global transform as inputs.

- [ ] **Step 4: Use global material ids**

When filling `SubInstanceData`, replace:

```cpp
SubEntry.MaterialID = Primitive.MaterialId;
```

with:

```cpp
SubEntry.MaterialID = Primitive.MaterialId < Asset.MaterialRemap.size() ?
    Asset.MaterialRemap[Primitive.MaterialId] :
    0;
```

- [ ] **Step 5: Use skinned scene lookup**

Replace the old `FindSkinnedNode(pSkinnedGeometry, pNode)` with:

```cpp
const RTXPTSkinnedSceneNodeGeometry* pSkinnedNode =
    pSkinnedGeometry != nullptr ?
        pSkinnedGeometry->FindNode(Instance.ModelAssetId, InstanceId, pNode) :
        nullptr;
```

Set skinned vertex offset using `pSkinnedNode->VertexBase + Primitive.FirstVertex`.

- [ ] **Step 6: Update sample rebuild**

In `RTXPTSample::RebuildSceneDependentResources()`, replace the old AS call with:

```cpp
ResourcesReady &= m_AccelerationStructures.BuildScene(m_pDevice,
                                                      m_pImmediateContext,
                                                      SceneData,
                                                      m_Scene.GetIndexType(),
                                                      &m_SkinnedGeometry,
                                                      m_FeatureCaps.RayTracing);
```

- [ ] **Step 7: Build and commit AS scene traversal**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp Samples/RTXPT/src/RTXPTSample.cpp
```

Manual smoke:

- switch to `kitchen.scene.json`
- expected UI `TLAS instances` is greater than the old single-model count
- expected material/light bridges still bind or clearly report fallback

Commit:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): build acceleration structures for scene graph instances" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Update Sample UI, Diagnostics, And Final Resource Wiring

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Remove obsolete single-model assumptions from rebuild**

In `RTXPTSample::RebuildSceneDependentResources()`, the first validity check should become:

```cpp
const RTXPTSceneGraphData& SceneData = m_Scene.GetSceneGraphData();
if (SceneData.ModelAssets.empty() || SceneData.ModelInstances.empty())
{
    ResetSceneDependentResources();
    CreatePhase4Passes();
    return false;
}
```

- [ ] **Step 2: Run skinning before AS build**

Before `m_AccelerationStructures.BuildScene(...)`, add:

```cpp
ResourcesReady &= m_SkinnedGeometry.Initialize(m_pDevice,
                                               m_pEngineFactory,
                                               SceneData,
                                               m_FeatureCaps.ComputeShaders);

if (m_SkinnedGeometry.HasSkinnedGeometry() && m_SkinnedGeometry.IsReady())
    ResourcesReady &= m_SkinnedGeometry.Update(m_pImmediateContext, SceneData);
```

- [ ] **Step 3: Update per-frame animation path**

In `RTXPTSample::Update(...)`, replace the old dynamic skinning block with:

```cpp
if (m_Scene.IsGeometryDirty() && m_SkinnedGeometry.HasSkinnedGeometry() && m_SkinnedGeometry.IsReady())
{
    const RTXPTSceneGraphData& SceneData = m_Scene.GetSceneGraphData();
    const bool SkinningExecuted = m_SkinnedGeometry.Update(m_pImmediateContext, SceneData);
    if (SkinningExecuted)
    {
        m_AccelerationStructures.UpdateDynamicBLAS(m_pImmediateContext, SceneData, m_SkinnedGeometry);
        m_Scene.ClearGeometryDirty();
        RequestAccumulationReset("Skinned scene geometry updated");
    }
}
```

- [ ] **Step 4: Add adapter diagnostics to Scene UI**

In `RTXPTSample::UpdateUI()`, add:

```cpp
const RTXPTSceneAdapterStats& AdapterStats = m_Scene.GetAdapterStats();
ImGui::Text("Graph nodes: %u", AdapterStats.GraphNodeCount);
ImGui::Text("Model assets: %u", AdapterStats.ModelAssetCount);
ImGui::Text("Model instances: %u", AdapterStats.ModelInstanceCount);
ImGui::Text("Material extensions: %u", AdapterStats.MaterialExtensionCount);
ImGui::Text("Material fallbacks: %u", AdapterStats.MaterialFallbackCount);
ImGui::Text("Directional lights: %u", AdapterStats.DirectionalLightCount);
ImGui::Text("Point lights: %u", AdapterStats.PointLightCount);
ImGui::Text("Spot lights: %u", AdapterStats.SpotLightCount);
ImGui::Text("Environment lights: %u", AdapterStats.EnvironmentLightCount);
ImGui::Text("Unknown typed nodes: %u", AdapterStats.UnknownTypedNodeCount);
ImGui::Text("Skinned instances: %u", AdapterStats.SkinnedInstanceCount);
```

- [ ] **Step 5: Adjust ray tracing pass only if needed**

If the final multi-model bridge still provides a single global static vertex buffer and index buffer, leave `RTXPTRayTracingPass` unchanged.

If static geometry needs scene-level static geometry arenas, add parameters with this exact naming:

```cpp
IBuffer* pStaticVertexBuffer,
IBuffer* pStaticIndexBuffer,
```

and keep shader resource names unchanged where possible:

```cpp
"t_VertexBuffer"
"t_IndexBuffer"
```

Do not add material or light struct fields in this task.

- [ ] **Step 6: Build and commit UI/resource wiring**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Commit:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire scene graph diagnostics and resources" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 8: Final Verification And Cleanup

**Files:**
- Modify only files needed to fix verification failures from previous tasks.

- [ ] **Step 1: Run final source status checks**

Run:

```powershell
git -C DiligentSamples status --short
git status --short
```

Expected:

- `DiligentSamples` only shows intentional uncommitted changes if final fixes are still pending.
- Umbrella repo may show the `DiligentSamples` submodule pointer changed and this plan file.
- Do not add unrelated untracked files, including existing unrelated plan files.

- [ ] **Step 2: Run formatting whitespace check**

Run:

```powershell
git -C DiligentSamples diff --check
```

Expected: no output.

- [ ] **Step 3: Build RTXPT**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If the build tree is unavailable, configure using the repository's documented Debug x64 flow before retrying:

```powershell
cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DCMAKE_INSTALL_PREFIX=install\x64\Debug -DDILIGENT_BUILD_FX=TRUE -DDILIGENT_BUILD_SAMPLES=TRUE -DDILIGENT_BUILD_TOOLS=TRUE -DDILIGENT_NO_WEBGPU=TRUE -DDILIGENT_NO_ARCHIVER=FALSE -DDILIGENT_BUILD_TESTS=TRUE -DDILIGENT_DEVELOPMENT=TRUE -DDILIGENT_NO_FORMAT_VALIDATION=FALSE -DDILIGENT_USE_SPIRV_TOOLCHAIN=TRUE
cmake --build build\x64\Debug --config Debug --target RTXPT
```

- [ ] **Step 4: Manual runtime smoke**

Launch RTXPT from the built Debug output or Visual Studio. Verify these scene switches:

- `bistro-programmer-art.scene.json`: expected model asset count greater than one, material extension count greater than zero, environment light count at least one.
- `kitchen.scene.json`: expected `MechDrone` and `MechDroneInMicrowave` count as separate model instances, model asset count at least six, scene loads without stale `models[0]` behavior.
- `living-room.scene.json`: expected scene loads despite trailing comma in `models`.
- `convergence-test.scene.json`: expected directional light and environment light metadata are counted.

- [ ] **Step 5: Commit final fixes if any**

If Step 2 through Step 4 required additional fixes, commit them:

```powershell
git -C DiligentSamples add Samples/RTXPT
git -C DiligentSamples commit -m "fix(rtxpt): stabilize scene graph adapter integration" -m "Co-Authored-By: GPT 5.5"
```

- [ ] **Step 6: Report final state**

Collect:

```powershell
git -C DiligentSamples log --oneline -n 8
git -C DiligentSamples status --short
git status --short
```

Report:

- which commits were created in `DiligentSamples`
- whether the umbrella repo submodule pointer changed
- exact build command and result
- exact manual smoke scenes and result
- any verification that could not run and why
