# RTXPT Phase R4 EnvMapBaker IBL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the procedural-gradient environment path with a Diligent-native `EnvMapBaker` pipeline that loads scene/user HDR environment maps or procedural sky, publishes baked cubemap/importance/BRDF outputs, and samples the baked environment with MIS in the reference path tracer.

**Architecture:** Add a new `RTXPTEnvMapBaker` resource owner beside the existing `RTXPTLightsBaker`. The env baker owns source selection, DiligentFX cubemap/BRDF precompute, a Diligent compute importance-map baker, stable fallback textures/views, shader constants, and UI/debug status; `RTXPTSample` updates the env baker before `RTXPTLightsBaker`, then `RTXPTRayTracingPass` binds the env outputs to raygen/miss. Runtime HLSL switches `EnvMap::Eval` and environment NEE from the current procedural cosine sampler to RTXPT-fork-style cubemap + equal-area-octahedral MIP-descent importance sampling.

**Tech Stack:** C++17 in `DiligentSamples/Samples/RTXPT/src`, HLSL 6.5 ray tracing and compute shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, Diligent textures/SRVs/UAVs/samplers/static RT bindings, DiligentFX `PBR_Renderer` for cubemap and BRDF LUT precompute, Dear ImGui. `DiligentSamples` is a git submodule; implementation commits in this plan are made inside `DiligentSamples/`.

---

## Current Baseline

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMap.hlsli` returns a hard-coded procedural gradient from `EnvMap::Eval`.
- `PathTracerMiss.rmiss` writes `Payload.emission = EnvMap::Eval(WorldRayDirection())`.
- `PathTracerSample.rgen` multiplies miss emission by `g_Const.ptConsts.environmentIntensity` and uses `ComputeBSDFEnvMISWeight` with a cosine-hemisphere pdf from the previous surface normal.
- `PathTracer.hlsli::SampleEnvironmentNEE` samples a cosine hemisphere around the BSDF normal, evaluates the procedural gradient, and uses cosine pdf for MIS.
- `RTXPTSample` already exposes an "Environment Map" UI section, but `EnvironmentMapEnabled` is disabled and only `m_EnvIntensity` is live.
- `DiligentSamples/Samples/RTXPT/assets/EnvironmentMaps` already contains `.hdr`, `.exr`, and prefiltered/precompressed `.dds` environment assets.
- Scene files already contain `EnvironmentLight` metadata with `path`, `radianceScale`, `rotation`, and `==PROCEDURAL_SKY==` sentinel values.
- `RTXPTLightsBaker` already has `LightsBakerEnvMapParamsCPU EnvMapParams` and `LightingTypes.hlsli` already has `EnvMapImportanceMapMIPCount`, `EnvMapImportanceMapResolution`, and `LightsBakerEnvMapParams`, but the sample does not fill them yet.

## RTXPT-Fork Anchors

- `D:/RTXPT-fork/Rtxpt/Lighting/Distant/EnvMapBaker.h:49-127` - public `EnvMapBaker` lifecycle, outputs, BRDF LUT, cubemap processing options.
- `D:/RTXPT-fork/Rtxpt/Lighting/Distant/EnvMapBaker.cpp:364-640` - `PreUpdate`, `Update`, cubemap rebake/compression/importance update order.
- `D:/RTXPT-fork/Rtxpt/Lighting/Distant/EnvMapImportanceSamplingBaker.h:39-87` - importance-map baker lifecycle and shader params.
- `D:/RTXPT-fork/Rtxpt/Lighting/Distant/EnvMapImportanceSamplingBaker.hlsl:16-82` - equal-area-octahedral importance base-map build.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/EnvMap.hlsli:21-264` - `EnvMap`, `EnvMapSampler`, uniform sampling, MIP-descent sampling, pdf evaluation.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1348-1412` - env baker update before lights baker update and `EnvMapParams` propagation.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1939-1952` - scene/UI env map transform, color multiplier, and enabled state.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2331-2345` - runtime bindings for env cubemap, importance map, and samplers.
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:575-631` and `:727-728` - environment source and baker debug UI.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp` - env-map source/settings/stats structs, baker class, shader-constant accessors, SRV/sampler getters.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp` - scene-source parsing, environment asset enumeration, fallback/procedural source creation, DiligentFX cubemap/BRDF precompute wrapper, importance baker orchestration, UI helpers.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.cpp` - reusable compute-pass wrapper for env importance base/reduce passes.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl` - build mip 0 radiance/importance maps and reduce mips for MIP descent.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}` - add env baker member, source list, scene default apply, update order, frame constants, UI, status/debug readouts.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.{hpp,cpp}` - consume env constants/importance metadata in baker settings and control-buffer upload.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` - bind env cubemap/importance/radiance textures and samplers to raygen/miss.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - add env-map constants mirrored by C++ `SampleConstants`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli` - declare env resources and helpers for raygen.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss` - evaluate baked env map for misses.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` - replace cosine env NEE with env-map MIP-descent sampling and pdf evaluation.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - use env MIS pdfs and remove the old procedural-only TODO marker.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMap.hlsli` - port the runtime env-map data model and sampler helpers in Diligent style.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register new C++/shader files and link `DiligentFX`.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record R4 mappings and Diligent-native divergences.

## Cross-Cutting Contracts

- **Update order:** `RTXPTEnvMapBaker::Update` runs before `RTXPTLightsBaker::UpdateBegin`; `RTXPTLightsBakerSettings.EnvMapParams` and env importance dimensions always describe the same baked env outputs that raygen/miss sample that frame.
- **Stable RT bindings:** the RT PSO binds stable fallback views even when env maps are disabled or a load fails. Changing source/resolution/compression recreates env resources and then recreates the RT pass.
- **Constants layout:** C++ `SampleConstants` and HLSL `SampleConstants` grow together. Every size change updates both `static_assert`s.
- **Intensity ownership:** environment tint/intensity lives in the new env-map constants' `ColorEnabled.rgb`; raygen and env NEE do not multiply the old procedural `environmentIntensity` a second time after Task 7.
- **Fallback behavior:** when env map is disabled or no valid source exists, the baker provides a baked procedural-gradient source so the sample never binds null env resources.
- **Sampling correctness:** env light sampling changes variance only. BSDF-sampled misses use the same env pdf that the env NEE sampler would assign to that direction.
- **Backends:** D3D12 and Vulkan remain first-class. BC6H output compression is guarded by format/capability checks and has an uncompressed RGBA16F fallback.
- **Copyright:** do not copy NVIDIA file headers, comments, or large verbatim code blocks from RTXPT-fork. Port algorithm structure and names where useful, but write Diligent-owned source files with the local Apache/Diligent header.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: current R3/R4 baseline

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Do not overwrite dirty files without reading them first.

- [ ] **Step 2: Confirm R3 baker baseline exists**

Run:

```powershell
rg -n "RTXPTLightsBaker|LightingControlData|t_LightingControl|t_LightProxyCounters|t_LocalSamplingBuffer" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `RTXPTLightsBaker.{hpp,cpp}`, `RTXPTSample.{hpp,cpp}`, `RTXPTRayTracingPass.{hpp,cpp}`, `PathTracerBridge.hlsli`, and `Lighting/LightSampler.hlsli`.

- [ ] **Step 3: Confirm R4 open-work markers**

Run:

```powershell
rg -n "Phase R4|Phase 5\.4|EnvMapBaker|HDR environment-map|procedural sky" DiligentSamples/Samples/RTXPT docs/superpowers/specs/2026-05-30-rtxpt-reference-pathtracer-completion-design.md
```

Expected: matches in the spec, `RTXPTSample`, `PathTracerMiss.rmiss`, `PathTracerSample.rgen`, `EnvMap.hlsli`, and `RTXPT_FORK_MAPPING.md`.

- [ ] **Step 4: Confirm environment-map assets and scene metadata**

Run:

```powershell
Get-ChildItem DiligentSamples\Samples\RTXPT\assets\EnvironmentMaps -File | Select-Object -First 20 Name
rg -n "EnvironmentLight|radianceScale|path|==PROCEDURAL_SKY==" DiligentSamples/Samples/RTXPT/assets -g "*.scene.json"
```

Expected: the first command lists `.hdr`, `.exr`, or `.dds` files. The second command shows environment-light metadata in sample scene files.

- [ ] **Step 5: Commit nothing**

Expected: no commit in Task 0. This task only establishes the starting point.

---

### Task 1: Add Shared Env-Map Constants

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`

- [ ] **Step 1: Add C++ constants layout**

In `RTXPTSample.hpp`, add this struct immediately after `PathTracerConstants`:

```cpp
struct RTXPTEnvMapConstants
{
    float4 LocalToWorld0      = float4{1, 0, 0, 0};
    float4 LocalToWorld1      = float4{0, 1, 0, 0};
    float4 LocalToWorld2      = float4{0, 0, 1, 0};
    float4 WorldToLocal0      = float4{1, 0, 0, 0};
    float4 WorldToLocal1      = float4{0, 1, 0, 0};
    float4 WorldToLocal2      = float4{0, 0, 1, 0};
    float4 ColorEnabled       = float4{1, 1, 1, 1}; // rgb = tint * intensity, w = enabled.
    float4 ImportanceMetadata = float4{1, 1, 0, 0}; // xy = inv dim, z = base mip, w = importance enabled.
};
static_assert(sizeof(RTXPTEnvMapConstants) == 128, "RTXPTEnvMapConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Add C++ constants to `SampleConstants`**

In `RTXPTSample.hpp`, replace the `SampleConstants` definition with:

```cpp
struct SampleConstants
{
    float4x4              viewProj                  = float4x4::Identity();
    float4x4              viewProjInv               = float4x4::Identity();
    float4                cameraPositionAndTime     = float4{0, 0, 0, 0};
    float4                viewportSizeAndFrameIndex = float4{0, 0, 0, 0};
    PathTracerConstants   ptConsts                  = {};
    RTXPTEnvMapConstants  envMap                    = {};
};
static_assert(sizeof(SampleConstants) == 352, "SampleConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 3: Mirror constants in HLSL**

In `PathTracerShared.h`, add this after `PathTracerConstants`:

```hlsl
struct RTXPTEnvMapConstants
{
    float4 LocalToWorld0;
    float4 LocalToWorld1;
    float4 LocalToWorld2;
    float4 WorldToLocal0;
    float4 WorldToLocal1;
    float4 WorldToLocal2;
    float4 ColorEnabled;
    float4 ImportanceMetadata;
};
```

Then replace `SampleConstants` with:

```hlsl
struct SampleConstants
{
    float4x4             viewProj;
    float4x4             viewProjInv;
    float4               cameraPositionAndTime;
    float4               viewportSizeAndFrameIndex;
    PathTracerConstants  ptConsts;
    RTXPTEnvMapConstants envMap;
};
```

Update the comment above `SampleConstants` to say total size is 352 bytes.

- [ ] **Step 4: Source-check layout names**

Run:

```powershell
rg -n "RTXPTEnvMapConstants|SampleConstants.*352|envMap" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
```

Expected: matches in both files, including the `static_assert(sizeof(SampleConstants) == 352` line.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
git -C DiligentSamples commit -m "feat(rtxpt): add environment map frame constants" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Add EnvMap Baker Resource Owner

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create `RTXPTEnvMapBaker.hpp`**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp`:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include <memory>
#include <string>
#include <vector>

#include "BasicMath.hpp"
#include "EngineFactory.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "Texture.h"
#include "TextureView.h"
#include "Sampler.h"

#include "RTXPTLightsBaker.hpp"
#include "RTXPTSceneGraph.hpp"
#include "RTXPTSample.hpp"

namespace Diligent
{

enum class RTXPTEnvMapSourceKind
{
    ProceduralSky,
    TextureFile
};

struct RTXPTEnvMapSource
{
    RTXPTEnvMapSourceKind Kind = RTXPTEnvMapSourceKind::ProceduralSky;
    std::string           DisplayName;
    std::string           RelativePath;
    std::string           ResolvedPath;
};

struct RTXPTEnvMapSettings
{
    bool        Enabled = true;
    std::string SourceRelativePath = "==PROCEDURAL_SKY==";
    float3      RadianceScale = float3{1, 1, 1};
    float       Intensity = 1.0f;
    float       RotationRadians = 0.0f;
    Uint32      TargetCubeResolution = 1024;
    Uint32      ImportanceMapResolution = 1024;
    int         CompressionQuality = 0; // 0=Off, 1=Fast, 2=Quality.
};

struct RTXPTEnvMapBakerStats
{
    bool        Ready = false;
    bool        SourceLoaded = false;
    bool        Procedural = true;
    bool        ImportanceReady = false;
    bool        BRDFLUTReady = false;
    bool        CompressedOutput = false;
    Uint32      CubeResolution = 0;
    Uint32      CubeMipLevels = 0;
    Uint32      ImportanceResolution = 0;
    Uint32      ImportanceMipLevels = 0;
    Uint64      Version = 0;
    std::string SourceName;
    std::string LastError;
};

class RTXPTEnvMapBaker
{
public:
    void Reset();
    void SceneReloaded();

    bool CreateResources(IRenderDevice* pDevice, IDeviceContext* pContext, IEngineFactory* pEngineFactory, bool ComputeSupported);
    bool Update(IRenderDevice* pDevice, IDeviceContext* pContext, IEngineFactory* pEngineFactory,
                const std::string& AssetsRoot, const RTXPTEnvMapSettings& Settings, bool ForceRebuild, bool ComputeSupported);

    bool InfoGUI(float Indent);
    bool DebugGUI(float Indent);

    static std::vector<RTXPTEnvMapSource> EnumerateEnvironmentSources(const std::string& AssetsRoot);
    static RTXPTEnvMapSettings MakeSceneDefaultSettings(const RTXPTSceneGraphData& SceneData);

    const RTXPTEnvMapBakerStats& GetStats() const { return m_Stats; }
    const RTXPTEnvMapConstants&  GetConstants() const { return m_Constants; }
    const LightsBakerEnvMapParamsCPU& GetLightsBakerParams() const { return m_LightsBakerParams; }

    ITextureView* GetEnvironmentMapSRV() const { return m_EnvironmentMapSRV; }
    ITextureView* GetDiffuseIrradianceSRV() const { return m_DiffuseIrradianceSRV; }
    ITextureView* GetImportanceMapSRV() const { return m_ImportanceMapSRV; }
    ITextureView* GetRadianceMapSRV() const { return m_RadianceMapSRV; }
    ITextureView* GetBRDFLUTSRV() const { return m_BRDFLUTSRV; }
    ISampler*     GetEnvironmentSampler() const { return m_EnvironmentSampler; }
    ISampler*     GetImportanceSampler() const { return m_ImportanceSampler; }

private:
    bool LoadSourceTexture(IRenderDevice* pDevice, const std::string& AssetsRoot, const RTXPTEnvMapSettings& Settings);
    bool CreateProceduralSourceTexture(IRenderDevice* pDevice, const RTXPTEnvMapSettings& Settings);
    bool PrecomputeCubemap(IDeviceContext* pContext, const RTXPTEnvMapSettings& Settings);
    bool CreateImportanceMaps(IRenderDevice* pDevice, IDeviceContext* pContext, IEngineFactory* pEngineFactory,
                              const RTXPTEnvMapSettings& Settings, bool ComputeSupported);
    bool CreateFallbackTextures(IRenderDevice* pDevice);
    bool CreateSamplers(IRenderDevice* pDevice);
    void UpdateConstants(const RTXPTEnvMapSettings& Settings);

    RefCntAutoPtr<ITexture>     m_SourceTexture;
    RefCntAutoPtr<ITextureView> m_SourceSRV;
    RefCntAutoPtr<ITextureView> m_EnvironmentMapSRV;
    RefCntAutoPtr<ITextureView> m_DiffuseIrradianceSRV;
    RefCntAutoPtr<ITextureView> m_ImportanceMapSRV;
    RefCntAutoPtr<ITextureView> m_RadianceMapSRV;
    RefCntAutoPtr<ITextureView> m_BRDFLUTSRV;
    RefCntAutoPtr<ISampler>     m_EnvironmentSampler;
    RefCntAutoPtr<ISampler>     m_ImportanceSampler;

    std::unique_ptr<class PBR_Renderer> m_IBLPrecompute;
    RTXPTEnvMapConstants       m_Constants;
    LightsBakerEnvMapParamsCPU m_LightsBakerParams;
    RTXPTEnvMapSettings        m_LastSettings;
    RTXPTEnvMapBakerStats      m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create reset/fallback scaffolding**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp` with these includes and initial methods:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#include "RTXPTEnvMapBaker.hpp"
#include "RTXPTSceneJson.hpp"

#include "DebugUtilities.hpp"
#include "FileSystem.hpp"
#include "GraphicsUtilities.h"
#include "TextureUtilities.h"
#include "imgui.h"

#include "PBR_Renderer.hpp"

#include <algorithm>
#include <cmath>
#include <filesystem>

namespace Diligent
{

namespace
{

constexpr const char* kProceduralSkyPath = "==PROCEDURAL_SKY==";
constexpr Uint32      kFallbackCubeSize = 4;

bool IsEnvironmentFile(const std::filesystem::path& Path)
{
    const std::string Ext = Path.extension().string();
    return Ext == ".hdr" || Ext == ".exr" || Ext == ".dds" || Ext == ".ktx" || Ext == ".ktx2";
}

float4 MakeRow(const float4x4& M, Uint32 Row)
{
    return Row == 0 ? float4{M._11, M._12, M._13, 0} :
        Row == 1 ? float4{M._21, M._22, M._23, 0} :
                   float4{M._31, M._32, M._33, 0};
}

} // namespace

void RTXPTEnvMapBaker::Reset()
{
    m_SourceTexture.Release();
    m_SourceSRV.Release();
    m_EnvironmentMapSRV.Release();
    m_DiffuseIrradianceSRV.Release();
    m_ImportanceMapSRV.Release();
    m_RadianceMapSRV.Release();
    m_BRDFLUTSRV.Release();
    m_EnvironmentSampler.Release();
    m_ImportanceSampler.Release();
    m_IBLPrecompute.reset();
    m_Constants = {};
    m_LightsBakerParams = {};
    m_LastSettings = {};
    m_Stats = {};
}

void RTXPTEnvMapBaker::SceneReloaded()
{
    m_LastSettings.SourceRelativePath.clear();
    ++m_Stats.Version;
}

bool RTXPTEnvMapBaker::CreateResources(IRenderDevice* pDevice, IDeviceContext*, IEngineFactory*, bool)
{
    return CreateSamplers(pDevice) && CreateFallbackTextures(pDevice);
}

bool RTXPTEnvMapBaker::CreateSamplers(IRenderDevice* pDevice)
{
    if (pDevice == nullptr)
        return false;

    SamplerDesc LinearWrap;
    LinearWrap.Name = "RTXPT environment map sampler";
    LinearWrap.MinFilter = FILTER_TYPE_LINEAR;
    LinearWrap.MagFilter = FILTER_TYPE_LINEAR;
    LinearWrap.MipFilter = FILTER_TYPE_LINEAR;
    LinearWrap.AddressU = TEXTURE_ADDRESS_WRAP;
    LinearWrap.AddressV = TEXTURE_ADDRESS_WRAP;
    LinearWrap.AddressW = TEXTURE_ADDRESS_WRAP;
    pDevice->CreateSampler(LinearWrap, &m_EnvironmentSampler);

    SamplerDesc PointClamp;
    PointClamp.Name = "RTXPT environment importance sampler";
    PointClamp.MinFilter = FILTER_TYPE_POINT;
    PointClamp.MagFilter = FILTER_TYPE_POINT;
    PointClamp.MipFilter = FILTER_TYPE_POINT;
    PointClamp.AddressU = TEXTURE_ADDRESS_CLAMP;
    PointClamp.AddressV = TEXTURE_ADDRESS_CLAMP;
    PointClamp.AddressW = TEXTURE_ADDRESS_CLAMP;
    pDevice->CreateSampler(PointClamp, &m_ImportanceSampler);

    return m_EnvironmentSampler && m_ImportanceSampler;
}

} // namespace Diligent
```

Subsequent tasks fill the private methods. Keep this file compiling at every commit.

- [ ] **Step 3: Register files and dependency**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, append these entries inside the existing `set(SOURCE)` and `set(INCLUDE)` lists, keeping the rest unchanged:

```cmake
set(SOURCE
    src/RTXPTLightsBakerPass.cpp
    src/RTXPTEnvMapBaker.cpp
    src/RTXPTEnvMapBakerPass.cpp
)

set(INCLUDE
    src/RTXPTLightsBakerPass.hpp
    src/RTXPTEnvMapBaker.hpp
    src/RTXPTEnvMapBakerPass.hpp
)
```

Add `DiligentFX` to `target_link_libraries`:

```cmake
target_link_libraries(RTXPT
PRIVATE
    Diligent-AssetLoader
    Diligent-JSON
    DiligentFX
)
```

- [ ] **Step 4: Source-check scaffolding**

Run:

```powershell
rg -n "class RTXPTEnvMapBaker|RTXPTEnvMapSettings|RTXPTEnvMapBakerStats|DiligentFX|RTXPTEnvMapBaker.cpp" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: matches in the new header/source and CMake file.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTEnvMapBaker.hpp Samples/RTXPT/src/RTXPTEnvMapBaker.cpp Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add environment map baker owner" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Load Scene And User Environment Sources

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Enumerate environment-map assets**

In `RTXPTEnvMapBaker.cpp`, implement `EnumerateEnvironmentSources`:

```cpp
std::vector<RTXPTEnvMapSource> RTXPTEnvMapBaker::EnumerateEnvironmentSources(const std::string& AssetsRoot)
{
    std::vector<RTXPTEnvMapSource> Sources;
    Sources.push_back(RTXPTEnvMapSource{RTXPTEnvMapSourceKind::ProceduralSky, "Procedural Sky", kProceduralSkyPath, ""});

    const std::filesystem::path EnvRoot = std::filesystem::path{AssetsRoot} / "EnvironmentMaps";
    std::error_code Error;
    if (!std::filesystem::is_directory(EnvRoot, Error))
        return Sources;

    for (const std::filesystem::directory_entry& Entry : std::filesystem::directory_iterator{EnvRoot, Error})
    {
        if (Error || !Entry.is_regular_file() || !IsEnvironmentFile(Entry.path()))
            continue;

        RTXPTEnvMapSource Source;
        Source.Kind = RTXPTEnvMapSourceKind::TextureFile;
        Source.DisplayName = Entry.path().filename().string();
        Source.RelativePath = std::string{"EnvironmentMaps/"} + Source.DisplayName;
        Source.ResolvedPath = Entry.path().string();
        FileSystem::CorrectSlashes(Source.ResolvedPath);
        Sources.push_back(std::move(Source));
    }

    std::sort(Sources.begin() + 1, Sources.end(),
              [](const RTXPTEnvMapSource& Lhs, const RTXPTEnvMapSource& Rhs) { return Lhs.DisplayName < Rhs.DisplayName; });
    return Sources;
}
```

- [ ] **Step 2: Parse scene default environment settings**

In `RTXPTEnvMapBaker.cpp`, implement `MakeSceneDefaultSettings`:

```cpp
RTXPTEnvMapSettings RTXPTEnvMapBaker::MakeSceneDefaultSettings(const RTXPTSceneGraphData& SceneData)
{
    RTXPTEnvMapSettings Settings;
    Settings.SourceRelativePath = kProceduralSkyPath;
    Settings.Enabled = true;

    for (const RTXPTSceneLightMetadata& Light : SceneData.Lights)
    {
        if (Light.Type != "EnvironmentLight")
            continue;

        Settings.SourceRelativePath = ReadRTXPTOptionalString(Light.RawJson, "path", kProceduralSkyPath);
        float Scale[3] = {1.0f, 1.0f, 1.0f};
        if (ReadRTXPTFloatArray(Light.RawJson, "radianceScale", Scale, 3))
            Settings.RadianceScale = float3{Scale[0], Scale[1], Scale[2]};

        float Rotation[1] = {0.0f};
        if (ReadRTXPTFloatArray(Light.RawJson, "rotation", Rotation, 1))
            Settings.RotationRadians = Rotation[0];

        return Settings;
    }

    return Settings;
}
```

- [ ] **Step 3: Add sample state**

In `RTXPTSample.hpp`, include the new header:

```cpp
#include "RTXPTEnvMapBaker.hpp"
```

Add private declarations:

```cpp
void EnumerateEnvironmentMaps();
void ApplySceneEnvironmentSettings();
bool UpdateEnvMapBaker(bool ForceRebuild);
```

Add members near `m_LightsBaker`:

```cpp
RTXPTEnvMapBaker              m_EnvMapBaker;
std::vector<RTXPTEnvMapSource> m_EnvMapSources;
RTXPTEnvMapSettings           m_EnvMapSettings;
int                           m_SelectedEnvMapSource = 0;
bool                          m_EnvMapBakerDirty = true;
```

- [ ] **Step 4: Wire source enumeration and scene defaults**

In `RTXPTSample.cpp`, implement:

```cpp
void RTXPTSample::EnumerateEnvironmentMaps()
{
    m_EnvMapSources = RTXPTEnvMapBaker::EnumerateEnvironmentSources(m_AssetsRoot);
    m_SelectedEnvMapSource = 0;
}

void RTXPTSample::ApplySceneEnvironmentSettings()
{
    m_EnvMapSettings = RTXPTEnvMapBaker::MakeSceneDefaultSettings(m_Scene.GetSceneGraphData());
    m_ReferenceUI.EnvironmentMapEnabled = m_EnvMapSettings.Enabled;
    m_EnvIntensity = m_EnvMapSettings.Intensity;

    for (size_t Index = 0; Index < m_EnvMapSources.size(); ++Index)
    {
        if (m_EnvMapSources[Index].RelativePath == m_EnvMapSettings.SourceRelativePath)
        {
            m_SelectedEnvMapSource = static_cast<int>(Index);
            break;
        }
    }

    m_EnvMapBakerDirty = true;
}
```

Call `EnumerateEnvironmentMaps()` from `Initialize` immediately after `m_AssetsRoot` is known and before scene loading. Call `ApplySceneEnvironmentSettings()` after a scene successfully loads and before `RebuildSceneDependentResources()`.

- [ ] **Step 5: Source-check scene source support**

Run:

```powershell
rg -n "EnumerateEnvironmentMaps|ApplySceneEnvironmentSettings|MakeSceneDefaultSettings|EnvironmentMaps|==PROCEDURAL_SKY==" DiligentSamples/Samples/RTXPT/src
```

Expected: matches in `RTXPTEnvMapBaker.cpp`, `RTXPTSample.hpp`, and `RTXPTSample.cpp`.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTEnvMapBaker.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): load scene environment map settings" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Build Baked Cubemap And BRDF LUT

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp`

- [ ] **Step 1: Implement source texture loading**

In `RTXPTEnvMapBaker.cpp`, implement `LoadSourceTexture`:

```cpp
bool RTXPTEnvMapBaker::LoadSourceTexture(IRenderDevice* pDevice, const std::string& AssetsRoot, const RTXPTEnvMapSettings& Settings)
{
    m_SourceTexture.Release();
    m_SourceSRV.Release();

    if (Settings.SourceRelativePath == kProceduralSkyPath)
        return CreateProceduralSourceTexture(pDevice, Settings);

    std::string ResolvedPath = AssetsRoot;
    if (!ResolvedPath.empty() && !FileSystem::IsSlash(ResolvedPath.back()))
        ResolvedPath.push_back(FileSystem::SlashSymbol);
    ResolvedPath += Settings.SourceRelativePath;
    FileSystem::CorrectSlashes(ResolvedPath);
    ResolvedPath = FileSystem::SimplifyPath(ResolvedPath.c_str());

    CreateTextureFromFile(ResolvedPath.c_str(), TextureLoadInfo{"RTXPT environment source"}, pDevice, &m_SourceTexture);
    if (!m_SourceTexture)
    {
        m_Stats.LastError = std::string{"Failed to load environment map: "} + ResolvedPath;
        LOG_ERROR_MESSAGE(m_Stats.LastError.c_str());
        return CreateProceduralSourceTexture(pDevice, Settings);
    }

    m_SourceSRV = m_SourceTexture->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_Stats.SourceLoaded = m_SourceSRV != nullptr;
    m_Stats.Procedural = false;
    m_Stats.SourceName = Settings.SourceRelativePath;
    return m_Stats.SourceLoaded;
}
```

- [ ] **Step 2: Implement procedural fallback source**

In `RTXPTEnvMapBaker.cpp`, implement `CreateProceduralSourceTexture` as a small lat-long HDR source texture:

```cpp
bool RTXPTEnvMapBaker::CreateProceduralSourceTexture(IRenderDevice* pDevice, const RTXPTEnvMapSettings&)
{
    constexpr Uint32 Width = 512;
    constexpr Uint32 Height = 256;
    std::vector<float4> Pixels(Width * Height);

    for (Uint32 y = 0; y < Height; ++y)
    {
        const float v = (static_cast<float>(y) + 0.5f) / static_cast<float>(Height);
        const float yDir = 1.0f - 2.0f * v;
        const float t = std::clamp(yDir * 0.5f + 0.5f, 0.0f, 1.0f);
        const float3 horizon{0.48f, 0.58f, 0.68f};
        const float3 zenith{0.05f, 0.08f, 0.14f};
        const float3 color = horizon * (1.0f - t) + zenith * t;
        for (Uint32 x = 0; x < Width; ++x)
            Pixels[y * Width + x] = float4{color.x, color.y, color.z, 1.0f};
    }

    TextureDesc Desc;
    Desc.Name = "RTXPT procedural environment source";
    Desc.Type = RESOURCE_DIM_TEX_2D;
    Desc.Width = Width;
    Desc.Height = Height;
    Desc.Format = TEX_FORMAT_RGBA32_FLOAT;
    Desc.BindFlags = BIND_SHADER_RESOURCE;

    TextureSubResData Subres{Pixels.data(), Uint64{Width} * sizeof(float4)};
    TextureData Data{&Subres, 1};
    pDevice->CreateTexture(Desc, &Data, &m_SourceTexture);
    if (!m_SourceTexture)
    {
        m_Stats.LastError = "Failed to create procedural environment source";
        return false;
    }

    m_SourceSRV = m_SourceTexture->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_Stats.SourceLoaded = m_SourceSRV != nullptr;
    m_Stats.Procedural = true;
    m_Stats.SourceName = kProceduralSkyPath;
    return m_Stats.SourceLoaded;
}
```

- [ ] **Step 3: Precompute cubemap and BRDF LUT**

In `RTXPTEnvMapBaker.cpp`, implement `PrecomputeCubemap`:

```cpp
bool RTXPTEnvMapBaker::PrecomputeCubemap(IDeviceContext* pContext, const RTXPTEnvMapSettings&)
{
    if (pContext == nullptr || m_SourceSRV == nullptr)
        return false;

    if (!m_IBLPrecompute)
        m_IBLPrecompute = std::make_unique<PBR_Renderer>(pContext->GetDevice(), nullptr, pContext, PBR_Renderer::CreateInfo{});

    m_IBLPrecompute->PrecomputeCubemaps(pContext, m_SourceSRV);

    m_EnvironmentMapSRV = m_IBLPrecompute->GetPrefilteredEnvMapSRV();
    m_DiffuseIrradianceSRV = m_IBLPrecompute->GetIrradianceCubeSRV();
    m_BRDFLUTSRV = m_IBLPrecompute->GetPreintegratedGGX_SRV();
    if (!m_EnvironmentMapSRV || !m_DiffuseIrradianceSRV || !m_BRDFLUTSRV)
    {
        m_Stats.LastError = "Failed to precompute environment cubemap, diffuse irradiance, or BRDF LUT";
        return false;
    }

    const TextureDesc& Desc = m_EnvironmentMapSRV->GetTexture()->GetDesc();
    m_Stats.CubeResolution = Desc.Width;
    m_Stats.CubeMipLevels = Desc.MipLevels;
    m_Stats.BRDFLUTReady = true;
    return true;
}
```

Use `pContext->GetDevice()` only if the local Diligent interface exposes it in this checkout. If it does not, pass `IRenderDevice*` into `PrecomputeCubemap` and use that device pointer.

- [ ] **Step 4: Implement top-level `Update` without importance maps**

In `RTXPTEnvMapBaker.cpp`, implement:

```cpp
bool RTXPTEnvMapBaker::Update(IRenderDevice* pDevice, IDeviceContext* pContext, IEngineFactory* pEngineFactory,
                              const std::string& AssetsRoot, const RTXPTEnvMapSettings& Settings, bool ForceRebuild, bool ComputeSupported)
{
    if (pDevice == nullptr || pContext == nullptr || pEngineFactory == nullptr)
        return false;

    const bool SourceChanged =
        ForceRebuild ||
        Settings.SourceRelativePath != m_LastSettings.SourceRelativePath ||
        Settings.TargetCubeResolution != m_LastSettings.TargetCubeResolution;

    if (SourceChanged)
    {
        if (!LoadSourceTexture(pDevice, AssetsRoot, Settings))
            return false;
        if (!PrecomputeCubemap(pContext, Settings))
            return false;
        ++m_Stats.Version;
    }

    UpdateConstants(Settings);
    m_LastSettings = Settings;
    m_Stats.Ready = m_EnvironmentMapSRV != nullptr &&
        m_DiffuseIrradianceSRV != nullptr &&
        m_ImportanceMapSRV != nullptr &&
        m_RadianceMapSRV != nullptr &&
        m_BRDFLUTSRV != nullptr &&
        m_EnvironmentSampler != nullptr &&
        m_ImportanceSampler != nullptr;
    (void)ComputeSupported;
    return m_Stats.Ready;
}
```

Task 6 extends this method to build the importance maps.

- [ ] **Step 5: Source-check cubemap precompute**

Run:

```powershell
rg -n "CreateTextureFromFile|PBR_Renderer|PrecomputeCubemaps|GetPrefilteredEnvMapSRV|GetPreintegratedGGX_SRV|CreateProceduralSourceTexture" DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp
```

Expected: all symbols are present.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTEnvMapBaker.hpp Samples/RTXPT/src/RTXPTEnvMapBaker.cpp
git -C DiligentSamples commit -m "feat(rtxpt): bake environment cubemap and brdf lut" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Add Env Importance Baker Compute Pass

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.cpp`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create compute-pass wrapper header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.hpp`:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderResourceBinding.h"
#include "TextureView.h"

namespace Diligent
{

class RTXPTEnvMapBakerPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, const char* Name, const char* EntryPoint);
    bool Bind(IBuffer* pConstants, ITextureView* pSourceCubeSRV, ITextureView* pSrcMipSRV,
              ITextureView* pImportanceUAV, ITextureView* pRadianceUAV, ISampler* pLinearSampler);
    bool Dispatch(IDeviceContext* pContext, Uint32 ThreadGroupsX, Uint32 ThreadGroupsY);

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
};

} // namespace Diligent
```

- [ ] **Step 2: Implement compute-pass wrapper**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBakerPass.cpp` by following `RTXPTLightsBakerPass.cpp`, but compile `PathTracer/Lighting/EnvMapImportanceBaker.hlsl` and bind these dynamic resources:

```cpp
ResourceLayout
    .AddVariable(SHADER_TYPE_COMPUTE, "g_EnvMapImportanceBakerConsts", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_EnvMapCube", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_SourceMip", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_ImportanceMap", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_RadianceMap", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "s_LinearWrap", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

`Bind` must ignore variables stripped from an entry point. Use this pattern:

```cpp
auto SetVariable = [this](const char* Name, IDeviceObject* pObject) {
    IShaderResourceVariable* pVar = m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, Name);
    if (pVar == nullptr)
        return true;
    if (pObject == nullptr)
        return false;
    pVar->Set(pObject);
    return true;
};
```

- [ ] **Step 3: Add importance-baker shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl`:

```hlsl
#ifndef __ENVMAP_IMPORTANCE_BAKER_HLSL__
#define __ENVMAP_IMPORTANCE_BAKER_HLSL__

#include "../Config.h"

#define RTXPT_ENVMAP_IMPORTANCE_THREADS 16

struct EnvMapImportanceBakerConstants
{
    uint  SourceCubeDim;
    uint  SourceCubeMipCount;
    uint  ImportanceMapDim;
    uint  ImportanceMapBaseMip;
    uint2 ImportanceMapDimInSamples;
    uint2 ImportanceMapNumSamples;
    float ImportanceMapInvSamples;
    uint  ReduceSrcMip;
    uint  ReduceDstMip;
    uint  _padding0;
};

ConstantBuffer<EnvMapImportanceBakerConstants> g_EnvMapImportanceBakerConsts;

TextureCube<float4> t_EnvMapCube;
Texture2D<float4>   t_SourceMip;
RWTexture2D<float>  u_ImportanceMap;
RWTexture2D<float4> u_RadianceMap;
SamplerState        s_LinearWrap;

float RTXPTEnvMapLuminance(float3 c)
{
    return dot(c, float3(0.2126, 0.7152, 0.0722));
}

float3 OctToDirEqualArea(float2 uv)
{
    float2 f = uv * 2.0 - 1.0;
    float3 n = float3(f.x, f.y, 1.0 - abs(f.x) - abs(f.y));
    if (n.z < 0.0)
    {
        const float2 old = n.xy;
        n.x = (1.0 - abs(old.y)) * (old.x >= 0.0 ? 1.0 : -1.0);
        n.y = (1.0 - abs(old.x)) * (old.y >= 0.0 ? 1.0 : -1.0);
    }
    return normalize(n);
}

[numthreads(RTXPT_ENVMAP_IMPORTANCE_THREADS, RTXPT_ENVMAP_IMPORTANCE_THREADS, 1)]
void BuildImportanceBaseCS(uint3 tid : SV_DispatchThreadID)
{
    const uint2 dim = g_EnvMapImportanceBakerConsts.ImportanceMapDim.xx;
    if (any(tid.xy >= dim))
        return;

    float importance = 0.0;
    float3 radiance = float3(0.0, 0.0, 0.0);

    [loop]
    for (uint y = 0; y < g_EnvMapImportanceBakerConsts.ImportanceMapNumSamples.y; ++y)
    {
        [loop]
        for (uint x = 0; x < g_EnvMapImportanceBakerConsts.ImportanceMapNumSamples.x; ++x)
        {
            const uint2 samplePos = tid.xy * g_EnvMapImportanceBakerConsts.ImportanceMapNumSamples + uint2(x, y);
            const float2 uv = (float2(samplePos) + 0.5) / float2(g_EnvMapImportanceBakerConsts.ImportanceMapDimInSamples);
            const float3 dir = OctToDirEqualArea(uv);
            const float3 sampleRadiance = t_EnvMapCube.SampleLevel(s_LinearWrap, dir, 0).rgb;
            importance += 0.5 * (RTXPTEnvMapLuminance(sampleRadiance) + (sampleRadiance.x + sampleRadiance.y + sampleRadiance.z) / 3.0);
            radiance += sampleRadiance;
        }
    }

    importance *= g_EnvMapImportanceBakerConsts.ImportanceMapInvSamples;
    radiance *= g_EnvMapImportanceBakerConsts.ImportanceMapInvSamples;
    u_ImportanceMap[tid.xy] = max(importance, 0.0);
    u_RadianceMap[tid.xy] = float4(radiance, max(importance, 0.0));
}

[numthreads(RTXPT_ENVMAP_IMPORTANCE_THREADS, RTXPT_ENVMAP_IMPORTANCE_THREADS, 1)]
void ReduceImportanceMipCS(uint3 tid : SV_DispatchThreadID)
{
    uint width;
    uint height;
    u_ImportanceMap.GetDimensions(width, height);
    const uint2 dstDim = max(uint2(width, height) >> g_EnvMapImportanceBakerConsts.ReduceDstMip, uint2(1, 1));
    if (any(tid.xy >= dstDim))
        return;

    const uint2 srcBase = tid.xy * 2u;
    float totalImportance = 0.0;
    float4 totalRadiance = float4(0.0, 0.0, 0.0, 0.0);
    [unroll]
    for (uint y = 0; y < 2u; ++y)
    {
        [unroll]
        for (uint x = 0; x < 2u; ++x)
        {
            const float4 src = t_SourceMip.Load(int3(srcBase + uint2(x, y), 0));
            totalImportance += src.w;
            totalRadiance += src;
        }
    }

    const float4 avg = totalRadiance * 0.25;
    u_RadianceMap[tid.xy] = float4(avg.rgb, totalImportance * 0.25);
    u_ImportanceMap[tid.xy] = totalImportance * 0.25;
}

#endif // __ENVMAP_IMPORTANCE_BAKER_HLSL__
```

During implementation, create per-mip SRV/UAV views for both maps before dispatching `ReduceImportanceMipCS`. If Diligent's UAV view creation requires explicit `TextureViewDesc`, use `MostDetailedMip = mip` and `NumMipLevels = 1`.

- [ ] **Step 4: Register files**

In `CMakeLists.txt`, add:

```cmake
src/RTXPTEnvMapBakerPass.cpp
src/RTXPTEnvMapBakerPass.hpp
assets/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl
```

- [ ] **Step 5: Source-check importance pass**

Run:

```powershell
rg -n "RTXPTEnvMapBakerPass|EnvMapImportanceBaker|BuildImportanceBaseCS|ReduceImportanceMipCS" DiligentSamples/Samples/RTXPT
```

Expected: matches in the new pass files, shader file, and CMake file.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTEnvMapBakerPass.hpp Samples/RTXPT/src/RTXPTEnvMapBakerPass.cpp Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add environment importance baker pass" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Generate Importance And Radiance Maps

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`

- [ ] **Step 1: Add importance resources to the baker**

In `RTXPTEnvMapBaker.hpp`, include the pass header and add members:

```cpp
#include "RTXPTEnvMapBakerPass.hpp"
```

```cpp
RefCntAutoPtr<IBuffer>      m_ImportanceConstants;
RefCntAutoPtr<ITexture>     m_ImportanceMap;
RefCntAutoPtr<ITexture>     m_RadianceMap;
RTXPTEnvMapBakerPass        m_BuildImportanceBasePass;
RTXPTEnvMapBakerPass        m_ReduceImportanceMipPass;
```

Add private helpers:

```cpp
bool CreateImportanceTextures(IRenderDevice* pDevice, Uint32 Resolution);
bool DispatchImportanceBuild(IDeviceContext* pContext, Uint32 Resolution);
bool DispatchImportanceReduce(IDeviceContext* pContext, Uint32 Resolution, Uint32 MipLevels);
```

- [ ] **Step 2: Add CPU constants mirror**

In `RTXPTEnvMapBaker.cpp`, add near the anonymous namespace:

```cpp
struct EnvMapImportanceBakerConstantsCPU
{
    Uint32 SourceCubeDim = 0;
    Uint32 SourceCubeMipCount = 0;
    Uint32 ImportanceMapDim = 0;
    Uint32 ImportanceMapBaseMip = 0;
    Uint32 ImportanceMapDimInSamples[2] = {};
    Uint32 ImportanceMapNumSamples[2] = {};
    float  ImportanceMapInvSamples = 1.0f;
    Uint32 ReduceSrcMip = 0;
    Uint32 ReduceDstMip = 0;
    Uint32 _padding0 = 0;
};
static_assert(sizeof(EnvMapImportanceBakerConstantsCPU) == 48, "EnvMapImportanceBakerConstantsCPU must match EnvMapImportanceBaker.hlsl");

Uint32 MipCountForPowerOfTwo(Uint32 Resolution)
{
    Uint32 Mips = 1;
    while ((Resolution >> Mips) != 0)
        ++Mips;
    return Mips;
}
```

- [ ] **Step 3: Create importance textures**

Implement `CreateImportanceTextures`:

```cpp
bool RTXPTEnvMapBaker::CreateImportanceTextures(IRenderDevice* pDevice, Uint32 Resolution)
{
    const Uint32 SafeResolution = std::max(1u, Resolution);
    const Uint32 MipLevels = MipCountForPowerOfTwo(SafeResolution);

    TextureDesc ImportanceDesc;
    ImportanceDesc.Name = "RTXPT environment importance map";
    ImportanceDesc.Type = RESOURCE_DIM_TEX_2D;
    ImportanceDesc.Width = SafeResolution;
    ImportanceDesc.Height = SafeResolution;
    ImportanceDesc.MipLevels = MipLevels;
    ImportanceDesc.Format = TEX_FORMAT_R32_FLOAT;
    ImportanceDesc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    pDevice->CreateTexture(ImportanceDesc, nullptr, &m_ImportanceMap);

    TextureDesc RadianceDesc = ImportanceDesc;
    RadianceDesc.Name = "RTXPT environment radiance map";
    RadianceDesc.Format = TEX_FORMAT_RGBA16_FLOAT;
    pDevice->CreateTexture(RadianceDesc, nullptr, &m_RadianceMap);

    if (!m_ImportanceMap || !m_RadianceMap)
    {
        m_Stats.LastError = "Failed to create environment importance textures";
        return false;
    }

    m_ImportanceMapSRV = m_ImportanceMap->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_RadianceMapSRV = m_RadianceMap->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_Stats.ImportanceResolution = SafeResolution;
    m_Stats.ImportanceMipLevels = MipLevels;
    return m_ImportanceMapSRV && m_RadianceMapSRV;
}
```

- [ ] **Step 4: Initialize passes and dispatch build/reduce**

In `CreateImportanceMaps`, initialize the two passes, create textures/constants, dispatch base build, then dispatch reductions from mip 0 to last mip:

```cpp
bool RTXPTEnvMapBaker::CreateImportanceMaps(IRenderDevice* pDevice, IDeviceContext* pContext, IEngineFactory* pEngineFactory,
                                            const RTXPTEnvMapSettings& Settings, bool ComputeSupported)
{
    if (!ComputeSupported || m_EnvironmentMapSRV == nullptr)
    {
        m_Stats.LastError = "Environment importance map requires compute support and a baked cubemap";
        return false;
    }

    const Uint32 Resolution = std::max(16u, Settings.ImportanceMapResolution);
    if (!CreateImportanceTextures(pDevice, Resolution))
        return false;

    BufferDesc ConstDesc;
    ConstDesc.Name = "RTXPT environment importance baker constants";
    ConstDesc.Usage = USAGE_DYNAMIC;
    ConstDesc.BindFlags = BIND_UNIFORM_BUFFER;
    ConstDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    ConstDesc.Size = sizeof(EnvMapImportanceBakerConstantsCPU);
    pDevice->CreateBuffer(ConstDesc, nullptr, &m_ImportanceConstants);
    if (!m_ImportanceConstants)
        return false;

    if (!m_BuildImportanceBasePass.Initialize(pDevice, pEngineFactory, "RTXPT env importance base", "BuildImportanceBaseCS"))
        return false;
    if (!m_ReduceImportanceMipPass.Initialize(pDevice, pEngineFactory, "RTXPT env importance reduce", "ReduceImportanceMipCS"))
        return false;

    if (!DispatchImportanceBuild(pContext, Resolution))
        return false;
    if (!DispatchImportanceReduce(pContext, Resolution, m_Stats.ImportanceMipLevels))
        return false;

    m_Stats.ImportanceReady = true;
    return true;
}
```

For `DispatchImportanceReduce`, create SRV/UAV views for each source/destination mip in the loop. Use `MapHelper<EnvMapImportanceBakerConstantsCPU>` to update `ReduceSrcMip` and `ReduceDstMip` before each dispatch.

- [ ] **Step 5: Call importance build from `Update`**

In `RTXPTEnvMapBaker::Update`, after `PrecomputeCubemap`, add:

```cpp
if (!CreateImportanceMaps(pDevice, pContext, pEngineFactory, Settings, ComputeSupported))
    return false;
```

- [ ] **Step 6: Fill constants after importance build**

In `UpdateConstants`, set:

```cpp
m_Constants.ImportanceMetadata =
    m_Stats.ImportanceReady ?
    float4{1.0f / static_cast<float>(m_Stats.ImportanceResolution),
           1.0f / static_cast<float>(m_Stats.ImportanceResolution),
           static_cast<float>(m_Stats.ImportanceMipLevels - 1u),
           1.0f} :
    float4{1.0f, 1.0f, 0.0f, 0.0f};
```

- [ ] **Step 7: Source-check importance resources**

Run:

```powershell
rg -n "m_ImportanceMap|m_RadianceMap|CreateImportanceMaps|DispatchImportanceBuild|DispatchImportanceReduce|ImportanceMetadata" DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.*
```

Expected: matches in both header and source.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTEnvMapBaker.hpp Samples/RTXPT/src/RTXPTEnvMapBaker.cpp
git -C DiligentSamples commit -m "feat(rtxpt): generate environment importance maps" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Integrate Env Baker Into Sample Lifecycle

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`

- [ ] **Step 1: Reset and create env resources**

In `RTXPTSample::ResetSceneDependentResources`, add:

```cpp
m_EnvMapBaker.SceneReloaded();
m_EnvMapBakerDirty = true;
```

In `RTXPTSample::RebuildSceneDependentResources`, before `m_LightsBaker.CreateResources`, add:

```cpp
ResourcesReady &= m_EnvMapBaker.CreateResources(m_pDevice, m_pImmediateContext, m_pEngineFactory, m_FeatureCaps.ComputeShaders);
ResourcesReady &= UpdateEnvMapBaker(true);
```

- [ ] **Step 2: Implement env baker update helper**

In `RTXPTSample.cpp`, add:

```cpp
bool RTXPTSample::UpdateEnvMapBaker(bool ForceRebuild)
{
    if (!m_pDevice || !m_pImmediateContext || !m_pEngineFactory)
        return false;

    m_EnvMapSettings.Enabled = m_ReferenceUI.EnvironmentMapEnabled;
    m_EnvMapSettings.Intensity = m_EnvIntensity;
    if (m_SelectedEnvMapSource >= 0 && m_SelectedEnvMapSource < static_cast<int>(m_EnvMapSources.size()))
        m_EnvMapSettings.SourceRelativePath = m_EnvMapSources[static_cast<size_t>(m_SelectedEnvMapSource)].RelativePath;

    const bool Updated = m_EnvMapBaker.Update(m_pDevice, m_pImmediateContext, m_pEngineFactory,
                                              m_AssetsRoot, m_EnvMapSettings, ForceRebuild || m_EnvMapBakerDirty,
                                              m_FeatureCaps.ComputeShaders);
    if (Updated)
        m_EnvMapBakerDirty = false;
    return Updated;
}
```

- [ ] **Step 3: Fill frame constants**

In `RTXPTSample::UpdateFrameConstants`, after filling `ptConsts`, add:

```cpp
m_LastFrameConstants.envMap = m_EnvMapBaker.GetConstants();
```

Stop writing `m_LastFrameConstants.ptConsts.environmentIntensity = m_EnvIntensity` after Task 10 removes old intensity usage. Until Task 10, leave it as a harmless compatibility value.

- [ ] **Step 4: Fill lights-baker env params**

In `RTXPTSample::UpdateLightsBaker`, add:

```cpp
BakerSettings.EnvMapParams = m_EnvMapBaker.GetLightsBakerParams();
BakerSettings.EnvMapImportanceMapResolution = m_EnvMapBaker.GetStats().ImportanceResolution;
BakerSettings.EnvMapImportanceMapMipCount = m_EnvMapBaker.GetStats().ImportanceMipLevels;
```

In `RTXPTLightsBakerSettings`, add:

```cpp
Uint32 EnvMapImportanceMapResolution = 0;
Uint32 EnvMapImportanceMapMipCount = 0;
```

Fill those fields from `m_EnvMapBaker.GetStats()` in `UpdateLightsBaker`.

- [ ] **Step 5: Upload env metadata into `LightingControlData`**

In `RTXPTLightsBaker.cpp::UploadControlBuffer`, replace the existing `BakerPadding`-only env handling with explicit writes matching `LightsBakerConstants` field order:

```cpp
Control.BakerPadding[1] = Settings.EnvMapImportanceMapMipCount;
Control.BakerPadding[2] = Settings.EnvMapImportanceMapResolution;
```

Then write the existing `EnvMapParams` rows into the `BakerPadding` slots that correspond to `LightsBakerConstants.EnvMapParams`. Keep the local static assert at `112 + 464`; if this becomes too fragile, replace `BakerPadding` with a named CPU mirror of `LightsBakerConstants`.

- [ ] **Step 6: Update resize and dirty handling**

In `WindowResize`, before recreating `m_LightsBaker`, call:

```cpp
UpdateEnvMapBaker(false);
```

In `Update`, before `UpdateLightsBaker`, add:

```cpp
if (m_EnvMapBakerDirty && UpdateEnvMapBaker(true))
{
    m_LightsBakerSettingsDirty = true;
    CreatePhase4Passes();
    RequestAccumulationReset("Environment map changed");
}
```

- [ ] **Step 7: Source-check lifecycle order**

Run:

```powershell
rg -n "UpdateEnvMapBaker|m_EnvMapBaker|GetLightsBakerParams|EnvMapImportanceMapResolution|EnvMapImportanceMapMipCount" DiligentSamples/Samples/RTXPT/src/RTXPTSample.* DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.*
```

Expected: env baker update appears before lights baker update in `RebuildSceneDependentResources` and `Update`.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTLightsBaker.hpp Samples/RTXPT/src/RTXPTLightsBaker.cpp
git -C DiligentSamples commit -m "feat(rtxpt): update environment baker before lighting baker" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 8: Bind Env Resources To Raygen And Miss

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`

- [ ] **Step 1: Add RT pass parameters**

In `RTXPTRayTracingPass.hpp`, add parameters after feedback resources:

```cpp
ITextureView*         pEnvironmentMapSRV,
ITextureView*         pEnvironmentImportanceMapSRV,
ITextureView*         pEnvironmentRadianceMapSRV,
ISampler*             pEnvironmentSampler,
ISampler*             pEnvironmentImportanceSampler,
```

Add stats fields:

```cpp
bool        EnvironmentBridgeBound = false;
```

- [ ] **Step 2: Declare shader resources**

In `PathTracerBridge.hlsli`, add after the feedback resources:

```hlsl
TextureCube<float4> t_EnvironmentMap;
Texture2D<float>    t_EnvironmentImportanceMap;
Texture2D<float4>   t_EnvironmentRadianceMap;
SamplerState        s_EnvironmentMapSampler;
SamplerState        s_EnvironmentImportanceSampler;
```

Add helpers:

```hlsl
namespace Bridge
{
    RTXPTEnvMapConstants getEnvMapConstants()
    {
        return g_Const.envMap;
    }
}
```

- [ ] **Step 3: Add resource-layout variables to raygen and miss**

In `RTXPTRayTracingPass.cpp`, define:

```cpp
const SHADER_TYPE EnvStages = SHADER_TYPE_RAY_GEN | SHADER_TYPE_RAY_MISS;
```

Add static variables:

```cpp
.AddVariable(EnvStages, "g_Const", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(EnvStages, "t_EnvironmentMap", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(EnvStages, "t_EnvironmentImportanceMap", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(EnvStages, "t_EnvironmentRadianceMap", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(EnvStages, "s_EnvironmentMapSampler", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(EnvStages, "s_EnvironmentImportanceSampler", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

If duplicate `g_Const` registration conflicts with the existing raygen-only registration, replace the old raygen registration with the `EnvStages` registration.

- [ ] **Step 4: Bind static resources for both stages**

Replace `SetStatic` with a helper that accepts a stage mask and binds each actual stage separately:

```cpp
auto SetStaticForStages = [&](SHADER_TYPE Stages, const char* Name, IDeviceObject* pObject, const char* ObjectName) {
    bool Ok = true;
    for (SHADER_TYPE Stage : {SHADER_TYPE_RAY_GEN, SHADER_TYPE_RAY_MISS, SHADER_TYPE_RAY_CLOSEST_HIT, SHADER_TYPE_RAY_ANY_HIT})
    {
        if ((Stages & Stage) == 0)
            continue;
        IShaderResourceVariable* pVar = m_PSO->GetStaticVariableByName(Stage, Name);
        if (pVar == nullptr)
            continue;
        if (pObject == nullptr)
        {
            DEV_ERROR("RTXPT static resource object is null: ", ObjectName);
            Ok = false;
            continue;
        }
        pVar->Set(pObject);
    }
    return Ok;
};
```

Bind env resources with `EnvStages`.

- [ ] **Step 5: Pass env resources from sample**

In both `m_RayTracingPass.Initialize` calls in `RTXPTSample::CreatePhase4Passes`, pass:

```cpp
m_EnvMapBaker.GetEnvironmentMapSRV(),
m_EnvMapBaker.GetImportanceMapSRV(),
m_EnvMapBaker.GetRadianceMapSRV(),
m_EnvMapBaker.GetEnvironmentSampler(),
m_EnvMapBaker.GetImportanceSampler(),
```

- [ ] **Step 6: Source-check env bindings**

Run:

```powershell
rg -n "t_EnvironmentMap|t_EnvironmentImportanceMap|t_EnvironmentRadianceMap|s_EnvironmentMapSampler|EnvironmentBridgeBound" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `RTXPTRayTracingPass`, `RTXPTSample`, and `PathTracerBridge.hlsli`.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): bind environment baker outputs to ray tracing" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 9: Port Runtime EnvMap Sampler

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMap.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`

- [ ] **Step 1: Replace procedural-only `EnvMap.hlsli`**

Replace `EnvMap.hlsli` with a Diligent-native runtime sampler:

```hlsl
#ifndef __ENVMAP_HLSLI__
#define __ENVMAP_HLSLI__

#include "../PathTracerShared.h"

static const float kEnvMapInvFourPi = 0.07957747154594767;

float3 RTXPTEnvFallback(float3 worldDir)
{
    const float  t       = saturate(worldDir.y * 0.5 + 0.5);
    const float3 horizon = float3(0.48, 0.58, 0.68);
    const float3 zenith  = float3(0.05, 0.08, 0.14);
    return lerp(horizon, zenith, t);
}

float3 RTXPTEnvMul3x3(float3 v, float4 r0, float4 r1, float4 r2)
{
    return float3(dot(v, r0.xyz), dot(v, r1.xyz), dot(v, r2.xyz));
}

float2 RTXPTDirToOctEqualArea(float3 n)
{
    n /= max(abs(n.x) + abs(n.y) + abs(n.z), 1.0e-8);
    float2 uv = n.xy;
    if (n.z < 0.0)
    {
        const float2 old = uv;
        uv.x = (1.0 - abs(old.y)) * (old.x >= 0.0 ? 1.0 : -1.0);
        uv.y = (1.0 - abs(old.x)) * (old.y >= 0.0 ? 1.0 : -1.0);
    }
    return uv * 0.5 + 0.5;
}

float3 RTXPTEnvOctToDirEqualArea(float2 uv)
{
    float2 f = uv * 2.0 - 1.0;
    float3 n = float3(f.x, f.y, 1.0 - abs(f.x) - abs(f.y));
    if (n.z < 0.0)
    {
        const float2 old = n.xy;
        n.x = (1.0 - abs(old.y)) * (old.x >= 0.0 ? 1.0 : -1.0);
        n.y = (1.0 - abs(old.x)) * (old.y >= 0.0 ? 1.0 : -1.0);
    }
    return normalize(n);
}

struct DistantLightSample
{
    float3 Dir;
    float  Pdf;
    float3 Le;
};

struct EnvMapSampler
{
    RTXPTEnvMapConstants Constants;

    float3 ToLocal(float3 worldDir)
    {
        return normalize(RTXPTEnvMul3x3(worldDir, Constants.WorldToLocal0, Constants.WorldToLocal1, Constants.WorldToLocal2));
    }

    float3 ToWorld(float3 localDir)
    {
        return normalize(RTXPTEnvMul3x3(localDir, Constants.LocalToWorld0, Constants.LocalToWorld1, Constants.LocalToWorld2));
    }

    float3 EvalLocal(float3 localDir, float lod)
    {
        if (Constants.ColorEnabled.w <= 0.0)
            return float3(0.0, 0.0, 0.0);
        return t_EnvironmentMap.SampleLevel(s_EnvironmentMapSampler, localDir, lod).rgb * Constants.ColorEnabled.rgb;
    }

    float3 Eval(float3 worldDir, float lod)
    {
        if (Constants.ColorEnabled.w <= 0.0)
            return float3(0.0, 0.0, 0.0);
        return EvalLocal(ToLocal(worldDir), lod);
    }

    bool HasImportance()
    {
        return Constants.ImportanceMetadata.w > 0.0 && Constants.ImportanceMetadata.z > 0.0;
    }

    DistantLightSample UniformSample(float2 rnd)
    {
        const float z = 1.0 - 2.0 * rnd.y;
        const float r = sqrt(max(0.0, 1.0 - z * z));
        const float phi = 6.28318530718 * rnd.x;
        float s;
        float c;
        sincos(phi, s, c);
        const float3 localDir = float3(r * c, z, r * s);

        DistantLightSample sample;
        sample.Dir = ToWorld(localDir);
        sample.Pdf = kEnvMapInvFourPi;
        sample.Le = EvalLocal(localDir, 0.0);
        return sample;
    }

    float UniformEvalPdf(float3)
    {
        return kEnvMapInvFourPi;
    }

    DistantLightSample MIPDescentSample(float2 rnd)
    {
        if (!HasImportance())
            return UniformSample(rnd);

        float2 p = rnd;
        uint2 pos = uint2(0, 0);
        const uint baseMip = (uint)Constants.ImportanceMetadata.z;

        [loop]
        for (int mip = int(baseMip) - 1; mip >= 0; --mip)
        {
            pos *= 2u;
            const float w0 = t_EnvironmentImportanceMap.Load(int3(pos + uint2(0, 0), mip));
            const float w1 = t_EnvironmentImportanceMap.Load(int3(pos + uint2(1, 0), mip));
            const float w2 = t_EnvironmentImportanceMap.Load(int3(pos + uint2(0, 1), mip));
            const float w3 = t_EnvironmentImportanceMap.Load(int3(pos + uint2(1, 1), mip));

            const float left = w0 + w2;
            const float right = w1 + w3;
            const float splitX = (left + right) > 0.0 ? saturate(left / (left + right)) : 0.5;
            uint2 off = uint2(0, 0);

            if (p.x < splitX)
                p.x = splitX > 0.0 ? p.x / splitX : p.x;
            else
            {
                off.x = 1u;
                p.x = splitX < 1.0 ? (p.x - splitX) / (1.0 - splitX) : p.x;
            }

            const float bottom = off.x == 0u ? w0 : w1;
            const float top = off.x == 0u ? w2 : w3;
            const float splitY = (bottom + top) > 0.0 ? saturate(bottom / (bottom + top)) : 0.5;
            if (p.y < splitY)
                p.y = splitY > 0.0 ? p.y / splitY : p.y;
            else
            {
                off.y = 1u;
                p.y = splitY < 1.0 ? (p.y - splitY) / (1.0 - splitY) : p.y;
            }

            pos += off;
        }

        const float2 uv = (float2(pos) + p) * Constants.ImportanceMetadata.xy;
        const float3 localDir = RTXPTEnvOctToDirEqualArea(uv);
        const float avgWeight = max(t_EnvironmentImportanceMap.Load(int3(0, 0, baseMip)), 1.0e-8);
        const float weight = max(t_EnvironmentImportanceMap.Load(int3(pos, 0)), 0.0);

        DistantLightSample sample;
        sample.Dir = ToWorld(localDir);
        sample.Pdf = max(weight / avgWeight, 0.0) * kEnvMapInvFourPi;
        sample.Le = EvalLocal(localDir, 0.0);
        return sample;
    }

    float MIPDescentEvalPdf(float3 worldDir)
    {
        if (!HasImportance())
            return UniformEvalPdf(worldDir);

        const uint baseMip = (uint)Constants.ImportanceMetadata.z;
        const float2 uv = RTXPTDirToOctEqualArea(ToLocal(worldDir));
        const float avgWeight = max(t_EnvironmentImportanceMap.Load(int3(0, 0, baseMip)), 1.0e-8);
        const float weight = max(t_EnvironmentImportanceMap.SampleLevel(s_EnvironmentImportanceSampler, uv, 0.0), 0.0);
        return max(weight / avgWeight, 0.0) * kEnvMapInvFourPi;
    }
};

EnvMapSampler RTXPTCreateEnvMapSampler(RTXPTEnvMapConstants Constants)
{
    EnvMapSampler sampler;
    sampler.Constants = Constants;
    return sampler;
}

#endif // __ENVMAP_HLSLI__
```

This shader intentionally uses `t_EnvironmentMap` and samplers declared by `PathTracerBridge.hlsli`; include order in users of this file must include bridge first.

- [ ] **Step 2: Use bridge resources in miss shader**

In `PathTracerMiss.rmiss`, replace:

```hlsl
#include "PathTracerShared.h"
#include "Lighting/EnvMap.hlsli"
```

with:

```hlsl
#include "PathTracerBridge.hlsli"
#include "Lighting/EnvMap.hlsli"
```

Replace:

```hlsl
Payload.emission    = EnvMap::Eval(WorldRayDirection());
```

with:

```hlsl
EnvMapSampler EnvSampler = RTXPTCreateEnvMapSampler(Bridge::getEnvMapConstants());
Payload.emission = EnvSampler.Eval(WorldRayDirection(), 0.0);
```

- [ ] **Step 3: Remove stale miss marker**

Delete the old marker that says the procedural sky still needs to be replaced. Leave no R4 marker in `PathTracerMiss.rmiss`.

- [ ] **Step 4: Source-check runtime env sampler**

Run:

```powershell
rg -n "EnvMapSampler|MIPDescentSample|MIPDescentEvalPdf|RTXPTCreateEnvMapSampler|t_EnvironmentMap" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `EnvMap.hlsli`, `PathTracerBridge.hlsli`, and `PathTracerMiss.rmiss`.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Lighting/EnvMap.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss
git -C DiligentSamples commit -m "feat(rtxpt): sample baked environment map in miss shader" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 10: Replace Environment NEE And MIS

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Replace `SampleEnvironmentNEE`**

In `PathTracer.hlsli`, replace `SampleEnvironmentNEE` with:

```hlsl
float3 SampleEnvironmentNEE(StandardBSDFData bsdfData, float3 visibilityOrigin,
                            float3 wo, inout SampleGenerator sg, float fireflyFilterK)
{
    EnvMapSampler envSampler = RTXPTCreateEnvMapSampler(Bridge::getEnvMapConstants());
    if (envSampler.Constants.ColorEnabled.w <= 0.0)
        return float3(0.0, 0.0, 0.0);

    const DistantLightSample envSample = envSampler.MIPDescentSample(sampleNext2D(sg));
    if (envSample.Pdf <= 0.0)
        return float3(0.0, 0.0, 0.0);

    const float specProb = getSpecularProbability(bsdfData, wo);
    float3      f;
    float       bsdfPdf;
    EvalBSDF(bsdfData, wo, envSample.Dir, specProb, f, bsdfPdf);
    if (bsdfPdf <= 0.0)
        return float3(0.0, 0.0, 0.0);

    if (!TraceVisibilityRay(visibilityOrigin, envSample.Dir, kVisibilityRayTMax))
        return float3(0.0, 0.0, 0.0);

    const float misWeight = PowerHeuristic(1.0, envSample.Pdf, 1.0, bsdfPdf);
    float3 contribution = f * envSample.Le * (misWeight / envSample.Pdf);

    const float ffThreshold = g_Const.ptConsts.fireflyFilterThreshold;
    if (ffThreshold != 0.0)
    {
        const float neeK = ComputeNewScatterFireflyFilterK(fireflyFilterK, envSample.Pdf, 1.0);
        contribution *= FireflyFilterShort(Average(contribution), ffThreshold, neeK);
    }

    return contribution;
}
```

- [ ] **Step 2: Replace BSDF environment MIS helper**

In `PathTracer.hlsli`, replace `ComputeBSDFEnvMISWeight` with:

```hlsl
float ComputeBSDFEnvMISWeight(bool didEnvNEE, float prevBsdfPdf, float3 rayDir)
{
    if (!didEnvNEE || prevBsdfPdf <= 0.0)
        return 1.0;

    EnvMapSampler envSampler = RTXPTCreateEnvMapSampler(Bridge::getEnvMapConstants());
    const float envPdf = envSampler.MIPDescentEvalPdf(rayDir);
    if (envPdf <= 0.0)
        return 1.0;

    return PowerHeuristic(1.0, prevBsdfPdf, 1.0, envPdf);
}
```

- [ ] **Step 3: Update raygen miss accumulation**

In `PathTracerSample.rgen`, replace:

```hlsl
const float  misWeight   = PathTracer::ComputeBSDFEnvMISWeight(prevDidEnvNEE, prevBsdfPdf, prevNormal, rayDir);
const float3 envRadiance = payload.emission * g_Const.ptConsts.environmentIntensity;
float3       environmentEmission = envRadiance * misWeight;
```

with:

```hlsl
const float  misWeight = PathTracer::ComputeBSDFEnvMISWeight(prevDidEnvNEE, prevBsdfPdf, rayDir);
float3       environmentEmission = payload.emission * misWeight;
```

Remove `prevNormal` if it no longer has another use.

- [ ] **Step 4: Stop double-writing procedural intensity**

In `RTXPTSample::UpdateFrameConstants`, remove the assignment:

```cpp
m_LastFrameConstants.ptConsts.environmentIntensity  = m_EnvIntensity;
```

Keep the `environmentIntensity` field in `PathTracerConstants` until a later cleanup phase if removing it would create avoidable churn. Set its default to `1.0f` and do not read it in R4 shaders.

- [ ] **Step 5: Source-check no cosine env pdf remains**

Run:

```powershell
rg -n "environmentIntensity|max\\(dot\\(prevNormal|sampleCosineHemisphere\\(sampleNext2D\\(sg\\), bsdfData\\.N|ComputeBSDFEnvMISWeight" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: `ComputeBSDFEnvMISWeight` remains, but old cosine env pdf and runtime `environmentIntensity` uses are gone from env sampling.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): use env map importance sampling for nee" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 11: Add UI, Debug Status, And Compression Controls

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`

- [ ] **Step 1: Enable Environment Map UI**

In `RTXPTSample::UpdateUI`, replace the disabled Environment Map section with:

```cpp
ResetOnChange(ImGui::Checkbox("Enabled", &m_ReferenceUI.EnvironmentMapEnabled), "Environment map toggled");

const char* EnvPreview = m_EnvMapSources.empty() ? "none" : m_EnvMapSources[static_cast<size_t>(std::max(0, m_SelectedEnvMapSource))].DisplayName.c_str();
if (ImGui::BeginCombo("Source", EnvPreview))
{
    for (size_t Index = 0; Index < m_EnvMapSources.size(); ++Index)
    {
        const bool Selected = static_cast<int>(Index) == m_SelectedEnvMapSource;
        if (ImGui::Selectable(m_EnvMapSources[Index].DisplayName.c_str(), Selected))
        {
            m_SelectedEnvMapSource = static_cast<int>(Index);
            m_EnvMapBakerDirty = true;
            RequestAccumulationReset("Environment source changed");
        }
        if (Selected)
            ImGui::SetItemDefaultFocus();
    }
    ImGui::EndCombo();
}

ResetOnChange(ImGui::SliderFloat("Intensity", &m_EnvIntensity, 0.0f, 20.0f), "Environment intensity changed");
ResetOnChange(ImGui::SliderFloat("Rotation", &m_EnvMapSettings.RotationRadians, -PI_F, PI_F), "Environment rotation changed");
int CubeResolution = static_cast<int>(m_EnvMapSettings.TargetCubeResolution);
if (ResetOnChange(ImGui::SliderInt("Cube resolution", &CubeResolution, 256, 4096), "Environment cube resolution changed"))
    m_EnvMapSettings.TargetCubeResolution = static_cast<Uint32>(CubeResolution);
```

Keep the `CubeResolution` variable local to `UpdateUI` so the ImGui pointer type stays correct.

- [ ] **Step 2: Mark baker dirty from env controls**

Use the existing `ResetOnChange` pattern, but set `m_EnvMapBakerDirty = true` for source/resolution/rotation/intensity changes. If only intensity/rotation changes, `UpdateEnvMapBaker(false)` should update constants without rebuilding textures.

- [ ] **Step 3: Implement baker UI helpers**

In `RTXPTEnvMapBaker.cpp`, implement:

```cpp
bool RTXPTEnvMapBaker::InfoGUI(float Indent)
{
    ImGui::Indent(Indent);
    ImGui::Text("Ready: %s", m_Stats.Ready ? "yes" : "no");
    ImGui::Text("Source: %s", m_Stats.SourceName.empty() ? "none" : m_Stats.SourceName.c_str());
    ImGui::Text("Procedural: %s", m_Stats.Procedural ? "yes" : "no");
    ImGui::Text("Cubemap: %u (%u mips)", m_Stats.CubeResolution, m_Stats.CubeMipLevels);
    ImGui::Text("Diffuse irradiance: %s", m_DiffuseIrradianceSRV ? "ready" : "missing");
    ImGui::Text("Importance: %s %u (%u mips)", m_Stats.ImportanceReady ? "ready" : "missing",
                m_Stats.ImportanceResolution, m_Stats.ImportanceMipLevels);
    ImGui::Text("BRDF LUT: %s", m_Stats.BRDFLUTReady ? "ready" : "missing");
    if (!m_Stats.LastError.empty())
        ImGui::TextWrapped("EnvMapBaker error: %s", m_Stats.LastError.c_str());
    ImGui::Unindent(Indent);
    return false;
}

bool RTXPTEnvMapBaker::DebugGUI(float Indent)
{
    ImGui::Indent(Indent);
    ImGui::Text("Version: %llu", static_cast<unsigned long long>(m_Stats.Version));
    ImGui::Text("Compressed output: %s", m_Stats.CompressedOutput ? "yes" : "no");
    ImGui::Text("Environment SRV: %s", m_EnvironmentMapSRV ? "bound" : "missing");
    ImGui::Text("Diffuse irradiance SRV: %s", m_DiffuseIrradianceSRV ? "bound" : "missing");
    ImGui::Text("Importance SRV: %s", m_ImportanceMapSRV ? "bound" : "missing");
    ImGui::Text("Radiance SRV: %s", m_RadianceMapSRV ? "bound" : "missing");
    ImGui::Unindent(Indent);
    return false;
}
```

- [ ] **Step 4: Add status/debug readouts**

In `Status / Debug`, add:

```cpp
const RTXPTEnvMapBakerStats& EnvStats = m_EnvMapBaker.GetStats();
ImGui::Text("EnvMapBaker: %s", EnvStats.Ready ? "ready" : "not ready");
ImGui::Text("Env source: %s", EnvStats.SourceName.empty() ? "none" : EnvStats.SourceName.c_str());
ImGui::Text("Env bridge: %s", RTPassStats.EnvironmentBridgeBound ? "bound" : "missing");
ImGui::Text("Env importance: %s", EnvStats.ImportanceReady ? "ready" : "missing");
```

Call `m_EnvMapBaker.InfoGUI(Indent)` inside the Environment Map section and `m_EnvMapBaker.DebugGUI(Indent)` in Status / Debug.

- [ ] **Step 5: Add guarded compression UI**

Add:

```cpp
ImGui::BeginDisabled(m_pDevice->GetDeviceInfo().Type == RENDER_DEVICE_TYPE_VULKAN);
if (ImGui::Combo("BC6H compression", &m_EnvMapSettings.CompressionQuality, "Off\0Fast\0Quality\0\0"))
{
    m_EnvMapBakerDirty = true;
    RequestAccumulationReset("Environment compression changed");
}
ImGui::EndDisabled();
if (m_pDevice->GetDeviceInfo().Type == RENDER_DEVICE_TYPE_VULKAN)
    PlaceholderTooltip("BC6H output compression remains disabled on Vulkan until the Diligent compute-copy path is validated.");
```

The baker must still load and sample BC6H `.dds` source files on Vulkan; only newly compressed output is disabled.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTEnvMapBaker.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose environment baker controls" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 12: Update Mapping And Remove R4 Markers

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Update mapping document**

In `RTXPT_FORK_MAPPING.md`, add this R4 section near the existing environment-map note:

```markdown
## Phase R4 EnvMapBaker Mapping

| RTXPT-fork | Diligent RTXPT | Notes |
|---|---|---|
| `Lighting/Distant/EnvMapBaker.*` | `src/RTXPTEnvMapBaker.*` | Diligent-native resource owner; uses DiligentFX cubemap/BRDF precompute and local compute importance-map passes |
| `Lighting/Distant/EnvMapImportanceSamplingBaker.*` | `src/RTXPTEnvMapBaker.*` + `Lighting/EnvMapImportanceBaker.hlsl` | Builds R32 importance and RGBA16F radiance mip chains for MIP descent |
| `SampleProceduralSky` | procedural source path in `RTXPTEnvMapBaker` | Current port bakes the existing procedural gradient into the same env-map path; source sentinel is `==PROCEDURAL_SKY==` |
| `PathTracer/Lighting/EnvMap.hlsli` | `PathTracer/Lighting/EnvMap.hlsli` | Runtime `EnvMapSampler`, MIP-descent sampling, and env pdf evaluation |
| global env bindings `t_EnvironmentMap`, `t_EnvironmentMapImportanceMap`, `t_EnvironmentRadianceMap` | RT static resources in `RTXPTRayTracingPass` | Bound for raygen and miss with stable fallback views; BRDF LUT remains a baker output until a later composite path consumes it |
```

- [ ] **Step 2: Remove resolved R4 markers**

In `PathTracerSample.rgen`, remove:

```hlsl
// TODO(RTXPT-Port Phase R4): Add HDR environment-map importance sampling + MIS (procedural-sky cosine env sampler today).
```

In `RTXPTSample.cpp` Status / Debug, remove:

```cpp
ImGui::TextWrapped("TODO(RTXPT-Port Phase R4): HDR environment map with importance sampling + MIS.");
```

- [ ] **Step 3: Source-check old R4 statements are gone**

Run:

```powershell
rg -n "procedural-sky cosine|HDR environment map with importance sampling|Replace the procedural sky|Phase 5\.4" DiligentSamples/Samples/RTXPT
```

Expected: no matches. If a remaining marker describes a deliberate, smaller follow-up such as physical procedural-sky parity, rewrite it to name that exact follow-up and phase instead of broad R4.

- [ ] **Step 4: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "docs(rtxpt): record environment baker migration" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 13: Final Verification

**Files:**
- Verify: all files touched by Tasks 1-12

- [ ] **Step 1: Run stale-symbol scans**

```powershell
rg -n "procedural-sky cosine|Replace the procedural sky|Phase 5\.4|HDR environment-map loading lands in Phase R4" DiligentSamples/Samples/RTXPT
```

Expected: no matches.

- [ ] **Step 2: Run new-symbol scans**

```powershell
rg -n "RTXPTEnvMapBaker|EnvMapSampler|MIPDescentSample|t_EnvironmentMap|t_EnvironmentImportanceMap|EnvironmentBridgeBound|EnvMapImportanceBaker" DiligentSamples/Samples/RTXPT
```

Expected: matches in C++, HLSL, and CMake files.

- [ ] **Step 3: Run whitespace validation**

```powershell
git -C DiligentSamples diff --check
```

Expected: no output.

- [ ] **Step 4: Run targeted format validation after C++ edits**

```powershell
cd DiligentSamples\BuildTools\FormatValidation
.\validate_format_win.bat
```

Expected: formatting validation succeeds. If the script fails because the local build tools are not configured, report the exact failure and run `git -C DiligentSamples diff --check` as the minimum fallback.

- [ ] **Step 5: Build RTXPT**

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If shader compilation fails, first check stage-specific static bindings and `SampleConstants` size/layout.

- [ ] **Step 6: Manual D3D12 smoke**

Run the RTXPT sample with the D3D12 backend from the built sample launcher or Visual Studio target.

Manual checks:

```text
1. Load bistro-programmer-art.scene.json.
2. Confirm Environment Map source defaults to EnvironmentMaps/shanghai_bund_4k_cube_bc6u.dds.
3. Toggle Environment Map off and on.
4. Expected: image changes immediately, accumulation resets, no null-resource validation errors.
5. Switch source to ==PROCEDURAL_SKY==.
6. Expected: sky is still sampled through EnvMapBaker, not the old shader-only gradient path.
7. Enable Environment NEE + MIS.
8. Expected: sky-lit regions converge faster than BSDF-only while matching the long-run brightness.
9. Rotate the environment.
10. Expected: miss radiance and env NEE rotate together; no MIS brightness jump.
```

- [ ] **Step 7: Manual Vulkan smoke**

Run the RTXPT sample with the Vulkan backend.

Manual checks:

```text
1. Load kitchen.scene.json.
2. Confirm BC6H source DDS files load as SRVs.
3. Confirm BC6H output compression UI is disabled if the compression path is not validated.
4. Toggle Environment NEE + MIS.
5. Expected: no descriptor/static-binding validation errors; image remains unbiased versus BSDF-only.
```

- [ ] **Step 8: Final status**

```powershell
git -C DiligentSamples status --short
git status --short
```

Expected: `DiligentSamples` contains only intentional implementation commits. Top-level repo may show a modified submodule pointer if implementation commits were made inside `DiligentSamples`.

## Self-Review

- [x] **Spec coverage:** Covers G7: env-map loading, procedural fallback, baked cubemap, diffuse irradiance cube, BRDF LUT, importance/radiance maps, runtime env sampling, MIS, UI/debug, baker lifecycle/update order, and D3D12/Vulkan verification.
- [x] **Baseline fit:** Starts from the current post-R3 state where `RTXPTLightsBaker` already exists and R4 remains procedural-gradient/cosine-env sampling.
- [x] **Type consistency:** `RTXPTEnvMapConstants`, `SampleConstants.envMap`, `LightingControlData` env fields, RT bindings, and HLSL `EnvMapSampler` names are consistent across tasks.
- [x] **No unresolved broad R4 markers:** The final tasks remove broad R4 markers and require any remaining note to name a smaller explicit follow-up.
- [x] **Verification:** Includes source scans, diff check, format validation, build command, and D3D12/Vulkan smoke checks.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-31-rtxpt-phase-r4-envmapbaker-ibl.md`. Two execution options:

**1. Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** - Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Choose one before implementation begins.
