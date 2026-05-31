# RTXPT Phase R3 LightsBaker Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current direct, non-`LightsBaker` light importance sampling path with a Diligent-native `LightsBaker` lifecycle that owns light control data, global proxies, NEE-AT feedback/local sampling resources, debug UI, and the R3 photometric punctual-light conversion.

**Architecture:** Keep `RTXPTLights` as the scene light inventory and buffer uploader, but move all sampling distribution ownership out of it into a new `RTXPTLightsBaker` class that mirrors RTXPT-fork's `LightsBaker` observable contract. The baker publishes `LightingControlData`, proxy counters, proxy indices, feedback textures, and local sampling buffers; raygen consumes these through an upstream-style `LightSampler` wrapper instead of reading the current `RTXPTLightProxy` CDF directly. After the baker migration preserves current Uniform/Power+ behavior, enable NEE-AT feedback/local sampling and then land G6 photometric/shaped punctual-light units on top of the baker-owned proxy weights.

**Tech Stack:** C++17 in `DiligentSamples/Samples/RTXPT/src`, HLSL 6.5 ray tracing and compute shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, Diligent structured buffers/textures/SRVs/UAVs/static raygen bindings, Dear ImGui. `DiligentSamples` is a git submodule; commits in this plan are made inside `DiligentSamples/`.

---

## Supersedes

This plan supersedes the earlier split R3 drafts:

- `docs/superpowers/plans/2026-05-31-rtxpt-phase-r3-g5-light-importance-sampling.md`
- `docs/superpowers/plans/2026-05-31-rtxpt-phase-r3-g6-photometric-shaped-punctual-lights.md`

Those drafts describe the already-landed direct implementation path and explicitly exclude full `LightsBaker`. Do not implement them as-is. Use this plan as the Phase R3 source of truth.

## Current Baseline

- `RTXPTLights` currently owns `t_Lights`, `t_EmissiveTriangles`, and `t_LightProxies`.
- `RTXPTLights::UploadLightProxyBuffer` builds a compact CDF in `RTXPTLightProxy { prefixWeight, weight, index, kind }`.
- `PathTracer/Lighting/LightSampler.hlsli` samples that CDF directly for Uniform/Power+ and feeds `PathTracer::SampleDirectLightNEE`.
- `RTXPTRayTracingPass` binds `t_LightProxies` directly as a raygen static SRV.
- UI exposes Uniform/Power+ plus candidate/full sample counts, while NEE-AT and approximate MIS remain disabled.

The migration target is different: `RTXPTLightsBaker` owns the sampling distributions and runtime feedback resources; shaders consume a `LightSampler` object backed by `LightingControlData`, proxy counters, proxy index buffers, feedback textures, and local sampling buffers.

## RTXPT-Fork Anchors

- `D:/RTXPT-fork/Rtxpt/Lighting/LightsBaker.h:49-111` - public lifecycle and output resources.
- `D:/RTXPT-fork/Rtxpt/Lighting/LightsBaker.cpp:964-1454` - `UpdateBegin`, `UpdateEnd`, `InfoGUI`, `DebugGUI`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightingConfig.h` - proxy/local-sampling constants.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightingTypes.hlsli` - `LightsBakerConstants`, `LightingControlData`, `LightFeedbackReservoir`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightSampler.hlsli` - global/local sampling, feedback insertion, light-vs-BSDF MIS.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1320-1408` - baker creation and update ordering.
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:709-772` - light preprocessing and NEE-AT UI.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp` - Diligent-native baker class, settings, stats, and resource accessors.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp` - CPU proxy build, control buffer upload, feedback/local resource allocation, UI helpers.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBakerPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBakerPass.cpp` - reusable compute-pass wrapper for the baker's multi-entry HLSL passes.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingConfig.h`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightsBaker.hlsl`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.{hpp,cpp}` - remove sampling-distribution ownership and expose baker inputs.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}` - add baker member, lifecycle calls, settings upload, UI, and debug readouts.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` - bind baker resources instead of `RTXPTLights` proxy resources.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli` - expose baker resources.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli` - replace direct CDF sampler with `LightSampler` backed by baker resources.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` and `PathTracerSample.rgen` - instantiate and use the baker-backed sampler.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli` - G6 photometric/shaped punctual-light sampling.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register new C++ and shader files.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record the full baker migration and remaining divergences.

## Cross-Cutting Contracts

- **`RTXPTLights` stops owning sampling distributions.** It still owns scene light upload and emissive-triangle buffer allocation. The baker owns control/proxy/feedback/local resources.
- **Light index space:** analytic lights occupy `[0, analyticLightCount)`. The emissive bucket is `analyticLightCount` when emissive triangles are enabled. R4 can append env-map quads after that.
- **Proxy data:** the baker uses RTXPT-style `ProxyCounters` and `ProxyIndices`, not the current prefix-CDF `RTXPTLightProxy` shader buffer. `ProxyCounters[lightIndex] / SamplingProxyCount` is the selection pdf.
- **Control data:** raygen derives light counts and sampling mode from `LightingControlData`, while `PathTracerConstants` keeps UI frame constants and fallback fields for compatibility until all call sites are migrated.
- **NEE-AT:** `NEEType == 2` means split candidate samples between local and global samplers, write feedback from NEE, and use local/global pdfs for MIS. Uniform/Power+ remain valid fallbacks.
- **Resource lifetime:** baker buffers/textures are stable objects bound by the RT PSO. Per-frame baker updates write contents, not new bindings. Scene topology or resolution changes recreate resources and then recreate the RT pass.
- **Unbiasedness:** Uniform, Power+, and NEE-AT change variance only. Any MIS path must use the same light-selection pdfs as the sampler that produced the light.
- **Backends:** all resources are standard Diligent buffers/textures. D3D12 and Vulkan remain first-class.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: current R3 implementation

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Do not overwrite dirty files without reading them first.

- [ ] **Step 2: Confirm the current direct proxy path exists**

Run:

```powershell
rg -n "RTXPTLightProxy|UploadLightProxyBuffer|GetLightProxyBuffer|t_LightProxies|SamplePowerProxyIndex|GenerateDirectLightCandidate" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `RTXPTLights.{hpp,cpp}`, `RTXPTRayTracingPass.{hpp,cpp}`, `PathTracerBridge.hlsli`, and `Lighting/LightSampler.hlsli`.

- [ ] **Step 3: Confirm old split plans are superseded but untouched**

Run:

```powershell
Test-Path docs/superpowers/plans/2026-05-31-rtxpt-phase-r3-g5-light-importance-sampling.md
Test-Path docs/superpowers/plans/2026-05-31-rtxpt-phase-r3-g6-photometric-shaped-punctual-lights.md
```

Expected: each command prints either `True` or `False`. Do not delete either file in this plan.

---

### Task 1: Add Shared Lighting Types And Config

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingConfig.h`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create `LightingConfig.h`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingConfig.h`:

```hlsl
#ifndef __LIGHTING_CONFIG_H__
#define __LIGHTING_CONFIG_H__

#include "../Config.h"

#define RTXPT_LIGHTING_MAX_LIGHTS                      (512 * 1024)
#define RTXPT_LIGHTING_SAMPLING_PROXY_RATIO            12
#define RTXPT_LIGHTING_MAX_SAMPLING_PROXIES            (RTXPT_LIGHTING_MAX_LIGHTS * RTXPT_LIGHTING_SAMPLING_PROXY_RATIO)
#define RTXPT_LIGHTING_MAX_SAMPLING_PROXIES_PER_LIGHT  (256 * 1024)
#define RTXPT_LIGHTING_MIN_WEIGHT_THRESHOLD            1e-8

#define RTXPT_LIGHTING_SAMPLING_BUFFER_TILE_SIZE       8
#define RTXPT_LIGHTING_SAMPLING_BUFFER_WINDOW_SIZE     8
#define RTXPT_LIGHTING_LOCAL_PROXY_COUNT               128
#define RTXPT_LIGHTING_LOCAL_PROXY_BINARY_SEARCH_STEPS 8
#define RTXPT_LIGHTING_TOP_UP_SAMPLES                  (RTXPT_LIGHTING_LOCAL_PROXY_COUNT - RTXPT_LIGHTING_SAMPLING_BUFFER_WINDOW_SIZE * RTXPT_LIGHTING_SAMPLING_BUFFER_WINDOW_SIZE)
#define RTXPT_LIGHTING_MAX_SAMPLE_COUNT                63
#define RTXPT_LIGHTING_SCREEN_SPACE_COHERENT_FEEDBACK_BIAS 1.0

#define RTXPT_NEEAT_EARLY_FEEDBACK_TILE_SIZE           2
#define RTXPT_INVALID_LIGHT_INDEX                      0xFFFFFFFFu

#endif // __LIGHTING_CONFIG_H__
```

- [ ] **Step 2: Create `LightingTypes.hlsli` with CPU/HLSL-compatible layouts**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli`:

```hlsl
#ifndef __LIGHTING_TYPES_HLSLI__
#define __LIGHTING_TYPES_HLSLI__

#include "LightingConfig.h"

#if defined(__cplusplus)
#define ROW_MAJOR
#else
#define ROW_MAJOR row_major
#endif

struct LightsBakerEnvMapParams
{
    ROW_MAJOR float3x4 Transform;
    ROW_MAJOR float3x4 InvTransform;
    float3             ColorMultiplier;
    float              Enabled;
};

#define NEEAT_LIGHTS_BAKER_CONSTANTS_SIZE 464

struct LightsBakerConstants
{
    float  DistantVsLocalRelativeImportance;
    uint   EnvMapImportanceMapMIPCount;
    uint   EnvMapImportanceMapResolution;
    uint   TriangleLightTaskCount;

    uint2  FeedbackResolution;
    uint2  BlendedFeedbackResolution;
    uint2  MouseCursorPos;
    float2 PrevOverCurrentViewportSize;

    int    DebugDrawType;
    uint   DebugDrawTileLights;
    uint   UpdateCounter;
    uint   DebugDrawFrustum;

    float  ImportanceBoostIntensityDelta;
    float  ImportanceBoostFrustumMul;
    float  ImportanceBoostFrustumFadeDistance;
    float  _padding3;

    float3 SceneCameraPos;
    float  SceneAverageContentsDistance;

    float  DepthDisocclusionThreshold;
    uint   EnableMotionReprojection;
    float  ReservoirHistoryDropoff;
    uint   _padding0;

    uint   CurrentWeightsBufferOffset;
    uint   HistoricWeightsBufferOffset;
    uint   _padding1;
    uint   _padding2;

    float4 FrustumPlanes[6];
    float4 FrustumCorners[8];

    LightsBakerEnvMapParams EnvMapParams;
};

struct LightingControlData
{
    uint TotalLightCount;
    uint EnvmapQuadNodeCount;
    uint AnalyticLightCount;
    uint TriangleLightCount;

    uint SamplingProxyCount;
    uint HistoricTotalLightCount;
    uint LastFrameTemporalFeedbackAvailable;
    uint LastFrameLocalSamplesAvailable;

    uint ProxyBuildTaskCount;
    uint WeightsSumUINT;
    uint ImportanceSamplingType;
    uint _padding0;

    uint  TemporalFeedbackRequired;
    uint  TotalMaxFeedbackCount;
    float GlobalFeedbackUseWeight;
    float LocalToGlobalSampleRatio;

    uint  TileBufferHeight;
    float ScreenSpaceVsWorldSpaceThreshold;
    uint2 LocalSamplingResolution;

    uint2 LocalSamplingTileJitter;
    uint2 LocalSamplingTileJitterPrev;

    uint ValidFeedbackCount;
    uint _padding1;
    uint _padding2;
    uint _padding3;

    LightsBakerConstants BakerConstants;

#if !defined(__cplusplus)
    float WeightsSum()
    {
        return asfloat(WeightsSumUINT);
    }
#endif
};

uint ComputeCandidateSampleLocalCount(float localToGlobalRatio, uint totalCandidateSamples)
{
    return (uint)((float)(totalCandidateSamples - 1u) * localToGlobalRatio + 0.75);
}

uint ComputeCandidateSampleGlobalCount(float localToGlobalRatio, uint totalCandidateSamples)
{
    return totalCandidateSamples - ComputeCandidateSampleLocalCount(localToGlobalRatio, totalCandidateSamples);
}

#if !defined(__cplusplus)
uint PackMiniListLightAndCount(uint globalLightIndex, uint counter)
{
    return ((globalLightIndex & 0x007FFFFFu) << 9u) | ((counter - 1u) & 0x1FFu);
}

void UnpackMiniListLightAndCount(uint value, out uint globalLightIndex, out uint counter)
{
    globalLightIndex = value >> 9u;
    counter          = (value & 0x1FFu) + 1u;
}

uint UnpackMiniListLight(uint value)
{
    return value >> 9u;
}

uint UnpackMiniListCount(uint value)
{
    return (value & 0x1FFu) + 1u;
}

uint LLSB_ComputeBaseAddress(uint2 tilePos, uint2 localSamplingResolution)
{
    return (tilePos.x + tilePos.y * localSamplingResolution.x) * RTXPT_LIGHTING_LOCAL_PROXY_COUNT;
}
#endif

#endif // __LIGHTING_TYPES_HLSLI__
```

- [ ] **Step 3: Register the new shader headers**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add:

```cmake
    assets/shaders/PathTracer/Lighting/LightingConfig.h
    assets/shaders/PathTracer/Lighting/LightingTypes.hlsli
```

near the other `PathTracer/Lighting/*` entries.

- [ ] **Step 4: Source-check layout names**

Run:

```powershell
rg -n "LightingControlData|LightsBakerConstants|RTXPT_LIGHTING_LOCAL_PROXY_COUNT|ComputeCandidateSampleLocalCount" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting
```

Expected: all patterns are found.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingConfig.h Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add LightsBaker shared lighting types" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Introduce `RTXPTLightsBaker` Resource Owner

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create the baker header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp`:

```cpp
#pragma once

#include <string>
#include <vector>

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "GLTFLoader.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "Texture.h"
#include "TextureView.h"

#include "RTXPTLights.hpp"
#include "RTXPTSceneGraph.hpp"

namespace Diligent
{

struct LightsBakerEnvMapParamsCPU
{
    float3x4 Transform       = {};
    float3x4 InvTransform    = {};
    float3   ColorMultiplier = float3{1, 1, 1};
    float    Enabled         = 0.0f;
};

struct RTXPTLightsBakerSettings
{
    Uint32  ImportanceSamplingType        = 1; // 0=Uniform, 1=Power+, 2=NEE-AT.
    float3  CameraPosition                = float3{0, 0, 0};
    float3  CameraDirection               = float3{0, 0, -1};
    float   AverageContentsDistance       = 10.0f;
    Uint32  MouseCursorX                  = 0;
    Uint32  MouseCursorY                  = 0;
    float4x4 ViewProjMatrix               = float4x4::Identity();
    float   GlobalTemporalFeedbackWeight  = 0.75f;
    float   LocalToGlobalSampleRatio      = 0.65f;
    bool    UseApproximateMIS             = false;
    bool    ResetFeedback                 = false;
    float2  ViewportSize                  = float2{0, 0};
    float2  PrevViewportSize              = float2{0, 0};
    LightsBakerEnvMapParamsCPU EnvMapParams = {};
    float   DistantVsLocalImportanceScale = 1.0f;
    Int64   FrameIndex                    = -1;
};

struct RTXPTLightsBakerStats
{
    bool        Ready                       = false;
    bool        FeedbackReady               = false;
    Uint32      TotalLightCount             = 0;
    Uint32      AnalyticLightCount          = 0;
    Uint32      TriangleLightCount          = 0;
    Uint32      SamplingProxyCount          = 0;
    Uint32      LocalSamplingTileCountX     = 0;
    Uint32      LocalSamplingTileCountY     = 0;
    Uint32      UpdateCounter               = 0;
    float       ProxyTotalWeight            = 0.0f;
    std::string LastError;
};

class RTXPTLightsBaker
{
public:
    void Reset();
    void SceneReloaded();

    bool CreateResources(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, Uint32 Width, Uint32 Height, bool ComputeSupported);
    bool UpdateBegin(IRenderDevice* pDevice, const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings);
    bool UpdateEnd(IDeviceContext* pContext);

    bool InfoGUI(float Indent);
    bool DebugGUI(float Indent);

    const RTXPTLightsBakerStats& GetStats() const { return m_Stats; }
    IBuffer* GetControlBuffer() const { return m_ControlBuffer; }
    IBuffer* GetLightProxyCounters() const { return m_LightProxyCounters; }
    IBuffer* GetLightSamplingProxies() const { return m_LightSamplingProxies; }
    IBuffer* GetLocalSamplingBuffer() const { return m_LocalSamplingBuffer; }
    ITextureView* GetFeedbackTotalWeightUAV() const { return m_FeedbackTotalWeightUAV; }
    ITextureView* GetFeedbackCandidatesUAV() const { return m_FeedbackCandidatesUAV; }
    ITextureView* GetFeedbackTotalWeightSRV() const { return m_FeedbackTotalWeightSRV; }
    ITextureView* GetFeedbackCandidatesSRV() const { return m_FeedbackCandidatesSRV; }

private:
    struct ProxyBuildItem
    {
        Uint32 LightIndex = 0;
        Uint32 Count      = 0;
        float  Weight     = 0.0f;
    };

    bool BuildGlobalProxies(const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings);
    bool UploadControlBuffer(IRenderDevice* pDevice, const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings);
    bool UploadProxyBuffers(IRenderDevice* pDevice);
    bool CreateFeedbackTextures(IRenderDevice* pDevice, Uint32 Width, Uint32 Height);
    bool CreateLocalSamplingBuffer(IRenderDevice* pDevice, Uint32 Width, Uint32 Height);

    RefCntAutoPtr<IBuffer>      m_ControlBuffer;
    RefCntAutoPtr<IBuffer>      m_LightProxyCounters;
    RefCntAutoPtr<IBuffer>      m_LightSamplingProxies;
    RefCntAutoPtr<IBuffer>      m_LocalSamplingBuffer;
    RefCntAutoPtr<ITexture>     m_FeedbackTotalWeight;
    RefCntAutoPtr<ITexture>     m_FeedbackCandidates;
    RefCntAutoPtr<ITextureView> m_FeedbackTotalWeightSRV;
    RefCntAutoPtr<ITextureView> m_FeedbackTotalWeightUAV;
    RefCntAutoPtr<ITextureView> m_FeedbackCandidatesSRV;
    RefCntAutoPtr<ITextureView> m_FeedbackCandidatesUAV;

    std::vector<Uint32> m_ProxyCounters;
    std::vector<Uint32> m_ProxyIndices;
    Uint32              m_AllocatedWidth  = 0;
    Uint32              m_AllocatedHeight = 0;
    RTXPTLightsBakerStats m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create the initial baker implementation**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp` with these includes and the reset/resource methods:

```cpp
#include "RTXPTLightsBaker.hpp"

#include "DebugUtilities.hpp"

#include "imgui.h"

#include <algorithm>
#include <cmath>
#include <numeric>

namespace Diligent
{

namespace
{

constexpr Uint32 kProxyRatio = 12u;
constexpr Uint32 kMinProxyCount = 1u;
constexpr Uint32 kLocalProxyCount = 128u;
constexpr Uint32 kTileSize = 8u;

float MaxRGB(const float4& V)
{
    return std::max(V.x, std::max(V.y, V.z));
}

float EstimateAnalyticWeight(const PolymorphicLightInfo& Light)
{
    return std::max(1.0e-6f, MaxRGB(Light.colorIntensity) * std::max(Light.colorIntensity.w, 0.0f));
}

Uint32 DivRoundUp(Uint32 Value, Uint32 Divisor)
{
    return (Value + Divisor - 1u) / Divisor;
}

} // namespace

void RTXPTLightsBaker::Reset()
{
    m_ControlBuffer.Release();
    m_LightProxyCounters.Release();
    m_LightSamplingProxies.Release();
    m_LocalSamplingBuffer.Release();
    m_FeedbackTotalWeight.Release();
    m_FeedbackCandidates.Release();
    m_FeedbackTotalWeightSRV.Release();
    m_FeedbackTotalWeightUAV.Release();
    m_FeedbackCandidatesSRV.Release();
    m_FeedbackCandidatesUAV.Release();
    m_ProxyCounters.clear();
    m_ProxyIndices.clear();
    m_AllocatedWidth  = 0;
    m_AllocatedHeight = 0;
    m_Stats = {};
}

void RTXPTLightsBaker::SceneReloaded()
{
    m_ProxyCounters.clear();
    m_ProxyIndices.clear();
    m_Stats.TotalLightCount    = 0;
    m_Stats.SamplingProxyCount = 0;
    m_Stats.ProxyTotalWeight   = 0.0f;
}

bool RTXPTLightsBaker::CreateResources(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, Uint32 Width, Uint32 Height, bool ComputeSupported)
{
    if (pDevice == nullptr)
    {
        m_Stats.LastError = "RTXPT LightsBaker requires a render device";
        return false;
    }
    if (pEngineFactory == nullptr)
    {
        m_Stats.LastError = "RTXPT LightsBaker requires an engine factory";
        return false;
    }
    if (!ComputeSupported)
    {
        m_Stats.LastError = "RTXPT LightsBaker requires compute shader support";
        return false;
    }

    const bool FeedbackOk = CreateFeedbackTextures(pDevice, Width, Height);
    const bool LocalOk    = CreateLocalSamplingBuffer(pDevice, Width, Height);
    m_Stats.Ready         = FeedbackOk && LocalOk;
    return m_Stats.Ready;
}
```

- [ ] **Step 3: Implement feedback/local resource creation**

In the same `.cpp`, add:

```cpp
bool RTXPTLightsBaker::CreateFeedbackTextures(IRenderDevice* pDevice, Uint32 Width, Uint32 Height)
{
    const Uint32 SafeWidth  = std::max(Width, 1u);
    const Uint32 SafeHeight = std::max(Height, 1u);

    TextureDesc WeightDesc;
    WeightDesc.Name      = "RTXPT LightsBaker feedback total weight";
    WeightDesc.Type      = RESOURCE_DIM_TEX_2D;
    WeightDesc.Width     = SafeWidth;
    WeightDesc.Height    = SafeHeight;
    WeightDesc.Format    = TEX_FORMAT_R32_FLOAT;
    WeightDesc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    pDevice->CreateTexture(WeightDesc, nullptr, &m_FeedbackTotalWeight);

    TextureDesc CandidateDesc = WeightDesc;
    CandidateDesc.Name   = "RTXPT LightsBaker feedback candidates";
    CandidateDesc.Format = TEX_FORMAT_R32_UINT;
    pDevice->CreateTexture(CandidateDesc, nullptr, &m_FeedbackCandidates);

    if (!m_FeedbackTotalWeight || !m_FeedbackCandidates)
    {
        m_Stats.LastError = "Failed to create RTXPT LightsBaker feedback textures";
        return false;
    }

    m_FeedbackTotalWeightSRV = m_FeedbackTotalWeight->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_FeedbackTotalWeightUAV = m_FeedbackTotalWeight->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS);
    m_FeedbackCandidatesSRV  = m_FeedbackCandidates->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE);
    m_FeedbackCandidatesUAV  = m_FeedbackCandidates->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS);
    m_Stats.FeedbackReady    = m_FeedbackTotalWeightSRV && m_FeedbackTotalWeightUAV && m_FeedbackCandidatesSRV && m_FeedbackCandidatesUAV;
    return m_Stats.FeedbackReady;
}

bool RTXPTLightsBaker::CreateLocalSamplingBuffer(IRenderDevice* pDevice, Uint32 Width, Uint32 Height)
{
    const Uint32 TileCountX = DivRoundUp(std::max(Width, 1u), kTileSize);
    const Uint32 TileCountY = DivRoundUp(std::max(Height, 1u), kTileSize);
    const Uint64 ElementCount = Uint64{TileCountX} * Uint64{TileCountY} * kLocalProxyCount;

    BufferDesc Desc;
    Desc.Name              = "RTXPT LightsBaker local sampling buffer";
    Desc.Usage             = USAGE_DEFAULT;
    Desc.BindFlags         = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(Uint32);
    Desc.Size              = ElementCount * sizeof(Uint32);
    pDevice->CreateBuffer(Desc, nullptr, &m_LocalSamplingBuffer);

    if (!m_LocalSamplingBuffer)
    {
        m_Stats.LastError = "Failed to create RTXPT LightsBaker local sampling buffer";
        return false;
    }

    m_Stats.LocalSamplingTileCountX = TileCountX;
    m_Stats.LocalSamplingTileCountY = TileCountY;
    return true;
}
```

- [ ] **Step 4: Register the C++ files**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add:

```cmake
    src/RTXPTLightsBaker.cpp
```

to `SOURCE`, and:

```cmake
    src/RTXPTLightsBaker.hpp
```

to `INCLUDE`.

- [ ] **Step 5: Source-check**

Run:

```powershell
rg -n "class RTXPTLightsBaker|CreateFeedbackTextures|CreateLocalSamplingBuffer|RTXPTLightsBaker.cpp|RTXPTLightsBaker.hpp" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: all new symbols and CMake entries are present.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTLightsBaker.hpp Samples/RTXPT/src/RTXPTLightsBaker.cpp Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add LightsBaker resource owner" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Move Global Proxy Building From `RTXPTLights` To `RTXPTLightsBaker`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.{hpp,cpp}`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`

- [ ] **Step 1: Expose baker inputs from `RTXPTLights`**

In `RTXPTLights.hpp`, remove:

```cpp
IBuffer* GetLightProxyBuffer() const { return m_LightProxyBuffer; }
Uint32   GetLightProxyCount() const { return m_Stats.LightProxyCount; }
bool UploadLightProxyBuffer(IRenderDevice* pDevice);
RefCntAutoPtr<IBuffer> m_LightProxyBuffer;
```

Add:

```cpp
const std::vector<PolymorphicLightInfo>& GetAnalyticLights() const { return m_AnalyticLights; }
float GetEmissiveProxyWeight() const { return m_EmissiveProxyWeight; }
```

Remove `LightProxyCount` and `LightProxyTotalWeight` from `RTXPTLightStats`; those move to baker stats.

- [ ] **Step 2: Remove proxy buffer ownership from `RTXPTLights.cpp`**

In `RTXPTLights::Reset`, remove:

```cpp
m_LightProxyBuffer.Release();
```

Delete the whole `RTXPTLights::UploadLightProxyBuffer` function. In `UploadEmissiveTriangles`, replace:

```cpp
return UploadEmissiveTriangleBuffer(pDevice, m_Stats.EmissiveTriangleCount) && UploadLightProxyBuffer(pDevice);
```

with:

```cpp
return UploadEmissiveTriangleBuffer(pDevice, m_Stats.EmissiveTriangleCount);
```

Remove the lines that reset `m_LightProxyBuffer`, `LightProxyCount`, and `LightProxyTotalWeight`.

- [ ] **Step 3: Build RTXPT-style proxy counters and indices in the baker**

In `RTXPTLightsBaker.cpp`, add `BuildGlobalProxies`, `UploadProxyBuffers`, `UploadControlBuffer`, and `UpdateBegin`:

```cpp
bool RTXPTLightsBaker::BuildGlobalProxies(const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings)
{
    const auto& AnalyticLights = Lights.GetAnalyticLights();
    const Uint32 AnalyticCount = static_cast<Uint32>(AnalyticLights.size());
    const bool HasEmissiveBucket = Lights.GetEmissiveTriangleCount() > 0;
    const Uint32 TotalLightCount = AnalyticCount + (HasEmissiveBucket ? 1u : 0u);

    m_ProxyCounters.assign(std::max(TotalLightCount, 1u), 0u);
    m_ProxyIndices.clear();
    m_Stats.ProxyTotalWeight = 0.0f;

    if (TotalLightCount == 0)
        return true;

    std::vector<ProxyBuildItem> Items;
    Items.reserve(TotalLightCount);
    for (Uint32 LightIndex = 0; LightIndex < AnalyticCount; ++LightIndex)
    {
        const float Weight = Settings.ImportanceSamplingType == 0 ? 1.0f : EstimateAnalyticWeight(AnalyticLights[LightIndex]);
        Items.push_back(ProxyBuildItem{LightIndex, 0u, Weight});
        m_Stats.ProxyTotalWeight += Weight;
    }

    if (HasEmissiveBucket)
    {
        const float Weight = Settings.ImportanceSamplingType == 0 ? 1.0f : std::max(1.0e-6f, Lights.GetEmissiveProxyWeight());
        Items.push_back(ProxyBuildItem{AnalyticCount, 0u, Weight});
        m_Stats.ProxyTotalWeight += Weight;
    }

    const Uint32 TargetProxyCount = std::max<Uint32>(TotalLightCount, TotalLightCount * kProxyRatio);
    Uint32 AssignedProxyCount = 0;
    for (ProxyBuildItem& Item : Items)
    {
        const float Normalized = m_Stats.ProxyTotalWeight > 0.0f ? Item.Weight / m_Stats.ProxyTotalWeight : 1.0f / float(TotalLightCount);
        Item.Count = std::max(kMinProxyCount, static_cast<Uint32>(std::round(Normalized * float(TargetProxyCount))));
        AssignedProxyCount += Item.Count;
    }

    m_ProxyIndices.reserve(AssignedProxyCount);
    for (const ProxyBuildItem& Item : Items)
    {
        m_ProxyCounters[Item.LightIndex] = Item.Count;
        for (Uint32 ProxyIndex = 0; ProxyIndex < Item.Count; ++ProxyIndex)
            m_ProxyIndices.push_back(Item.LightIndex);
    }

    m_Stats.TotalLightCount    = TotalLightCount;
    m_Stats.AnalyticLightCount = AnalyticCount;
    m_Stats.TriangleLightCount = Lights.GetEmissiveTriangleCount();
    m_Stats.SamplingProxyCount = static_cast<Uint32>(m_ProxyIndices.size());
    return true;
}

bool RTXPTLightsBaker::UploadProxyBuffers(IRenderDevice* pDevice)
{
    if (m_ProxyCounters.empty())
        m_ProxyCounters.push_back(0u);
    if (m_ProxyIndices.empty())
        m_ProxyIndices.push_back(0u);

    BufferDesc CounterDesc;
    CounterDesc.Name              = "RTXPT LightsBaker proxy counters";
    CounterDesc.Usage             = USAGE_IMMUTABLE;
    CounterDesc.BindFlags         = BIND_SHADER_RESOURCE;
    CounterDesc.Mode              = BUFFER_MODE_STRUCTURED;
    CounterDesc.ElementByteStride = sizeof(Uint32);
    CounterDesc.Size              = Uint64{m_ProxyCounters.size()} * sizeof(Uint32);
    BufferData CounterData{m_ProxyCounters.data(), CounterDesc.Size};
    pDevice->CreateBuffer(CounterDesc, &CounterData, &m_LightProxyCounters);

    BufferDesc ProxyDesc = CounterDesc;
    ProxyDesc.Name = "RTXPT LightsBaker sampling proxies";
    ProxyDesc.Size = Uint64{m_ProxyIndices.size()} * sizeof(Uint32);
    BufferData ProxyData{m_ProxyIndices.data(), ProxyDesc.Size};
    pDevice->CreateBuffer(ProxyDesc, &ProxyData, &m_LightSamplingProxies);

    if (!m_LightProxyCounters || !m_LightSamplingProxies)
    {
        m_Stats.LastError = "Failed to upload RTXPT LightsBaker proxy buffers";
        return false;
    }
    return true;
}
```

- [ ] **Step 4: Upload `LightingControlData`**

In `RTXPTLightsBaker.cpp`, include the HLSL layout for C++ by adding a local C++ mirror immediately above `UploadControlBuffer`:

```cpp
struct RTXPTLightingControlDataCPU
{
    Uint32 TotalLightCount = 0;
    Uint32 EnvmapQuadNodeCount = 0;
    Uint32 AnalyticLightCount = 0;
    Uint32 TriangleLightCount = 0;
    Uint32 SamplingProxyCount = 0;
    Uint32 HistoricTotalLightCount = 0;
    Uint32 LastFrameTemporalFeedbackAvailable = 0;
    Uint32 LastFrameLocalSamplesAvailable = 0;
    Uint32 ProxyBuildTaskCount = 0;
    Uint32 WeightsSumUINT = 0;
    Uint32 ImportanceSamplingType = 1;
    Uint32 _padding0 = 0;
    Uint32 TemporalFeedbackRequired = 0;
    Uint32 TotalMaxFeedbackCount = 0;
    float  GlobalFeedbackUseWeight = 0.75f;
    float  LocalToGlobalSampleRatio = 0.65f;
    Uint32 TileBufferHeight = 0;
    float  ScreenSpaceVsWorldSpaceThreshold = 0.3f;
    Uint32 LocalSamplingResolution[2] = {};
    Uint32 LocalSamplingTileJitter[2] = {};
    Uint32 LocalSamplingTileJitterPrev[2] = {};
    Uint32 ValidFeedbackCount = 0;
    Uint32 _padding1 = 0;
    Uint32 _padding2 = 0;
    Uint32 _padding3 = 0;
    Uint32 BakerPadding[464 / 4] = {};
};
static_assert(sizeof(RTXPTLightingControlDataCPU) == 112 + 464, "LightingControlData CPU mirror must match LightingTypes.hlsli");
```

Then add:

```cpp
bool RTXPTLightsBaker::UploadControlBuffer(IRenderDevice* pDevice, const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings)
{
    RTXPTLightingControlDataCPU Control;
    Control.TotalLightCount        = m_Stats.TotalLightCount;
    Control.AnalyticLightCount     = m_Stats.AnalyticLightCount;
    Control.TriangleLightCount     = m_Stats.TriangleLightCount;
    Control.SamplingProxyCount     = m_Stats.SamplingProxyCount;
    Control.ImportanceSamplingType = Settings.ImportanceSamplingType;
    Control.TemporalFeedbackRequired = Settings.ImportanceSamplingType == 2 ? 1u : 0u;
    Control.GlobalFeedbackUseWeight  = Settings.GlobalTemporalFeedbackWeight;
    Control.LocalToGlobalSampleRatio = Settings.LocalToGlobalSampleRatio;
    Control.LocalSamplingResolution[0] = m_Stats.LocalSamplingTileCountX;
    Control.LocalSamplingResolution[1] = m_Stats.LocalSamplingTileCountY;
    Control.TileBufferHeight = m_Stats.LocalSamplingTileCountY;
    Control.ValidFeedbackCount = 0;

    BufferDesc Desc;
    Desc.Name              = "RTXPT LightsBaker control buffer";
    Desc.Usage             = USAGE_IMMUTABLE;
    Desc.BindFlags         = BIND_SHADER_RESOURCE;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(RTXPTLightingControlDataCPU);
    Desc.Size              = sizeof(RTXPTLightingControlDataCPU);
    BufferData Data{&Control, Desc.Size};
    pDevice->CreateBuffer(Desc, &Data, &m_ControlBuffer);

    if (!m_ControlBuffer)
    {
        m_Stats.LastError = "Failed to upload RTXPT LightsBaker control buffer";
        return false;
    }
    return true;
}

bool RTXPTLightsBaker::UpdateBegin(IRenderDevice* pDevice, const RTXPTLights& Lights, const RTXPTLightsBakerSettings& Settings)
{
    m_ControlBuffer.Release();
    m_LightProxyCounters.Release();
    m_LightSamplingProxies.Release();

    if (!BuildGlobalProxies(Lights, Settings))
        return false;
    if (!UploadProxyBuffers(pDevice))
        return false;
    if (!UploadControlBuffer(pDevice, Lights, Settings))
        return false;

    ++m_Stats.UpdateCounter;
    m_Stats.Ready = true;
    return true;
}

bool RTXPTLightsBaker::UpdateEnd(IDeviceContext*)
{
    return m_Stats.Ready;
}
```

- [ ] **Step 5: Source-check direct proxy ownership is gone from `RTXPTLights`**

Run:

```powershell
rg -n "UploadLightProxyBuffer|m_LightProxyBuffer|GetLightProxyBuffer|LightProxyTotalWeight|LightProxyCount" DiligentSamples/Samples/RTXPT/src/RTXPTLights.*
```

Expected: no matches.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp Samples/RTXPT/src/RTXPTLightsBaker.cpp
git -C DiligentSamples commit -m "refactor(rtxpt): move global light proxies into LightsBaker" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Integrate The Baker Into Sample Lifecycle And RT Bindings

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}`

- [ ] **Step 1: Add the baker member**

In `RTXPTSample.hpp`, replace:

```cpp
#include "RTXPTLights.hpp"
```

with:

```cpp
#include "RTXPTLights.hpp"
#include "RTXPTLightsBaker.hpp"
```

Add after `RTXPTLights m_Lights;`:

```cpp
RTXPTLightsBaker m_LightsBaker;
```

Add UI state fields:

```cpp
float NEEAT_GlobalTemporalFeedbackWeight = 0.75f;
float NEEAT_LocalToGlobalSampleRatio     = 0.65f;
float NEEAT_DistantVsLocalImportance     = 1.0f;
```

inside `RTXPTReferenceUIState`.

- [ ] **Step 2: Create and update baker resources**

In `RTXPTSample::ResetSceneDependentResources`, add:

```cpp
m_LightsBaker.SceneReloaded();
```

In `RTXPTSample::RebuildSceneDependentResources`, after:

```cpp
ResourcesReady &= m_Lights.Upload(m_pDevice, SceneData);
ResourcesReady &= m_Lights.UploadEmissiveTriangles(m_pDevice, SceneData);
```

add:

```cpp
const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
ResourcesReady &= m_LightsBaker.CreateResources(m_pDevice, m_pEngineFactory, SCDesc.Width, SCDesc.Height, m_FeatureCaps.ComputeShaders);
```

After `BuildEmissiveTriangles()` succeeds, call:

```cpp
RTXPTLightsBakerSettings BakerSettings;
BakerSettings.ImportanceSamplingType       = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEType, 0, 2));
BakerSettings.GlobalTemporalFeedbackWeight = m_ReferenceUI.NEEAT_GlobalTemporalFeedbackWeight;
BakerSettings.LocalToGlobalSampleRatio     = m_ReferenceUI.NEEAT_LocalToGlobalSampleRatio;
BakerSettings.DistantVsLocalImportanceScale = m_ReferenceUI.NEEAT_DistantVsLocalImportance;
BakerSettings.ResetFeedback                = true;
BakerSettings.ViewportSize                 = float2{static_cast<float>(SCDesc.Width), static_cast<float>(SCDesc.Height)};
ResourcesReady &= m_LightsBaker.UpdateBegin(m_pDevice, m_Lights, BakerSettings);
```

- [ ] **Step 3: Use baker resources in RT pass initialization**

In `RTXPTRayTracingPass.hpp`, replace the `pLightProxyBuffer` parameter with:

```cpp
IBuffer* pLightingControlBuffer,
IBuffer* pLightProxyCounters,
IBuffer* pLightSamplingProxies,
IBuffer* pLocalSamplingBuffer,
ITextureView* pFeedbackTotalWeightUAV,
ITextureView* pFeedbackCandidatesUAV,
```

Keep `pLightBuffer` and `pEmissiveTriangleBuffer`.

In both `m_RayTracingPass.Initialize` calls in `RTXPTSample::CreatePhase4Passes`, replace:

```cpp
m_Lights.GetLightProxyBuffer(),
```

with:

```cpp
m_LightsBaker.GetControlBuffer(),
m_LightsBaker.GetLightProxyCounters(),
m_LightsBaker.GetLightSamplingProxies(),
m_LightsBaker.GetLocalSamplingBuffer(),
m_LightsBaker.GetFeedbackTotalWeightUAV(),
m_LightsBaker.GetFeedbackCandidatesUAV(),
```

- [ ] **Step 4: Bind baker resources in the RT pass**

In `RTXPTRayTracingPass.cpp`, add raygen static variables:

```cpp
.AddVariable(SHADER_TYPE_RAY_GEN, "t_LightingControl", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(SHADER_TYPE_RAY_GEN, "t_LightProxyCounters", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(SHADER_TYPE_RAY_GEN, "t_LightSamplingProxies", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(SHADER_TYPE_RAY_GEN, "t_LocalSamplingBuffer", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(SHADER_TYPE_RAY_GEN, "u_FeedbackTotalWeight", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
.AddVariable(SHADER_TYPE_RAY_GEN, "u_FeedbackCandidates", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

Replace `LightProxyBridgeBound` in `RTXPTRayTracingPassStats` with:

```cpp
bool LightsBakerBridgeBound = false;
```

Bind all six baker resources and set `LightsBakerBridgeBound` to the conjunction of their `SetStatic` results.

- [ ] **Step 5: Source-check old RT binding is gone**

Run:

```powershell
rg -n "t_LightProxies|GetLightProxyBuffer|LightProxyBridgeBound" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: no matches after Task 5 completes. During this task, C++ direct references should be gone; shader references are removed in Task 5.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): bind LightsBaker resources to path tracer" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Replace Direct HLSL CDF Sampling With Baker-Backed `LightSampler`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Replace bridge globals**

In `PathTracerBridge.hlsli`, remove:

```hlsl
StructuredBuffer<RTXPTLightProxy> t_LightProxies;
```

Add:

```hlsl
#include "Lighting/LightingTypes.hlsli"

StructuredBuffer<LightingControlData> t_LightingControl;
Buffer<uint>                          t_LightProxyCounters;
Buffer<uint>                          t_LightSamplingProxies;
Buffer<uint>                          t_LocalSamplingBuffer;
RWTexture2D<float>                    u_FeedbackTotalWeight;
RWTexture2D<uint>                     u_FeedbackCandidates;
```

Remove `getLightProxyCount`, `getLightProxy`, and `getLightProxyTotalWeight`. Add:

```hlsl
LightingControlData getLightingControl()
{
    return t_LightingControl[0];
}

uint getTotalLightCount()
{
    return t_LightingControl[0].TotalLightCount;
}

uint getEmissiveBucketLightIndex()
{
    return t_LightingControl[0].AnalyticLightCount;
}
```

- [ ] **Step 2: Replace `LightSampler.hlsli` sampling helpers**

In `LightSampler.hlsli`, replace `SamplePowerProxyIndex`, `GetProxySelectionPdf`, `SampleProxyIndex`, and `GetEmissiveTriangleSelectionPdf` with:

```hlsl
struct LightSamplerContext
{
    LightingControlData ctrl;
    uint2 pixelPos;
    uint localSamplingTileBase;
    bool isScreenSpaceCoherent;
};

LightSamplerContext MakeLightSamplerContext(uint2 pixelPos, bool isScreenSpaceCoherent)
{
    LightSamplerContext ctx;
    ctx.ctrl                  = t_LightingControl[0];
    ctx.pixelPos              = pixelPos;
    ctx.isScreenSpaceCoherent = isScreenSpaceCoherent;
    uint2 tilePos             = (pixelPos + ctx.ctrl.LocalSamplingTileJitter) / RTXPT_LIGHTING_SAMPLING_BUFFER_TILE_SIZE.xx;
    ctx.localSamplingTileBase = LLSB_ComputeBaseAddress(tilePos, ctx.ctrl.LocalSamplingResolution);
    return ctx;
}

uint SampleGlobalLightIndex(LightSamplerContext ctx, float rnd, out float pdf)
{
    if (ctx.ctrl.SamplingProxyCount == 0u)
    {
        pdf = 0.0;
        return RTXPT_INVALID_LIGHT_INDEX;
    }

    const uint proxyIndex = min(uint(rnd * float(ctx.ctrl.SamplingProxyCount)), ctx.ctrl.SamplingProxyCount - 1u);
    const uint lightIndex = t_LightSamplingProxies[proxyIndex];
    const uint proxyCount = t_LightProxyCounters[lightIndex];
    pdf = float(proxyCount) / float(ctx.ctrl.SamplingProxyCount);
    return lightIndex;
}

uint ReadLocalLightPacked(LightSamplerContext ctx, uint localIndex)
{
    return t_LocalSamplingBuffer[ctx.localSamplingTileBase + localIndex];
}

uint SampleLocalLightIndex(LightSamplerContext ctx, float rnd, out float pdf)
{
    const uint localIndex = min(uint(rnd * float(RTXPT_LIGHTING_LOCAL_PROXY_COUNT)), RTXPT_LIGHTING_LOCAL_PROXY_COUNT - 1u);
    uint packed = ReadLocalLightPacked(ctx, localIndex);
    uint lightIndex;
    uint proxyCount;
    UnpackMiniListLightAndCount(packed, lightIndex, proxyCount);
    pdf = float(proxyCount) / float(RTXPT_LIGHTING_LOCAL_PROXY_COUNT);
    return lightIndex;
}

float SampleGlobalPDF(uint lightIndex)
{
    const LightingControlData ctrl = t_LightingControl[0];
    if (ctrl.SamplingProxyCount == 0u || lightIndex >= ctrl.TotalLightCount)
        return 0.0;
    return float(t_LightProxyCounters[lightIndex]) / float(ctrl.SamplingProxyCount);
}

float GetEmissiveTriangleSelectionPdf()
{
    const LightingControlData ctrl = t_LightingControl[0];
    const uint triCount = Bridge::getEmissiveTriangleCount();
    if (triCount == 0u || ctrl.SamplingProxyCount == 0u)
        return 0.0;

    const uint bucketIndex = ctrl.AnalyticLightCount;
    return SampleGlobalPDF(bucketIndex) / float(triCount);
}
```

- [ ] **Step 3: Update direct-light candidate generation**

Change the signature to:

```hlsl
DirectLightSample GenerateDirectLightCandidate(StandardBSDFData bsdfData, float3 hitPos, float3 wo,
                                               inout SampleGenerator sg, LightSamplerContext ctx)
```

Inside it, choose local/global:

```hlsl
const uint localCandidateCount  = ComputeCandidateSampleLocalCount(ctx.ctrl.LocalToGlobalSampleRatio, g_Const.ptConsts.NEECandidateSamples);
const uint globalCandidateCount = g_Const.ptConsts.NEECandidateSamples - localCandidateCount;
const bool useLocal = ctx.ctrl.ImportanceSamplingType == 2u && localCandidateCount > 0u && sampleNext1D(sg) < ctx.ctrl.LocalToGlobalSampleRatio;

float selectionPdf = 0.0;
uint lightIndex = useLocal ? SampleLocalLightIndex(ctx, sampleNext1D(sg), selectionPdf) :
                             SampleGlobalLightIndex(ctx, sampleNext1D(sg), selectionPdf);
if (lightIndex == RTXPT_INVALID_LIGHT_INDEX || selectionPdf <= 0.0)
    return sample;
```

Use:

```hlsl
if (lightIndex < ctx.ctrl.AnalyticLightCount)
{
    const LightSample light = EvalAnalyticLight(Bridge::getLight(lightIndex), hitPos);
    ...
    sample.kind  = kLightProxyKindAnalytic;
    sample.index = lightIndex;
}
else if (lightIndex == ctx.ctrl.AnalyticLightCount && Bridge::getEmissiveTriangleCount() > 0u)
{
    ...
    const float trianglePdf = selectionPdf * (1.0 / float(triCount)) * solidAnglePdf;
    ...
    sample.kind  = kLightProxyKindEmissiveBucket;
    sample.index = triIndex;
}
```

- [ ] **Step 4: Update `PathTracer::SampleDirectLightNEE`**

Change the function signature:

```hlsl
float3 SampleDirectLightNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                            float3 wo, uint2 pixel, inout SampleGenerator sg, float fireflyFilterK,
                            out bool sampledEmissive)
```

At the top:

```hlsl
LightSamplerContext lightSampler = MakeLightSamplerContext(pixel, true);
if (lightSampler.ctrl.SamplingProxyCount == 0u)
    return float3(0.0, 0.0, 0.0);
```

Replace candidate generation with:

```hlsl
DirectLightSample candidate = GenerateDirectLightCandidate(bsdfData, hitPos, wo, sg, lightSampler);
```

- [ ] **Step 5: Source-check old `RTXPTLightProxy` shader path is gone**

Run:

```powershell
rg -n "RTXPTLightProxy|t_LightProxies|SamplePowerProxyIndex|GetProxySelectionPdf|GetLightProxyTotalWeight" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: no matches.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git -C DiligentSamples commit -m "refactor(rtxpt): sample lights through LightsBaker control data" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Add Feedback Reservoir Writes From NEE

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Add `LightFeedbackReservoir`**

Append to the HLSL-only section of `LightingTypes.hlsli`:

```hlsl
#define LFR_SCREEN_SPACE_COHERENT_FLAG 0x80000000u
#define LFR_MAX_WEIGHT 1e12

struct LightFeedbackReservoir
{
    uint2              pixelPos;
    RWTexture2D<float> totalWeight;
    RWTexture2D<uint>  candidates;

    static LightFeedbackReservoir make(uint2 pixelPos, RWTexture2D<float> totalWeight, RWTexture2D<uint> candidates)
    {
        LightFeedbackReservoir ret;
        ret.pixelPos    = pixelPos;
        ret.totalWeight = totalWeight;
        ret.candidates  = candidates;
        return ret;
    }

    void Clear()
    {
        totalWeight[pixelPos] = 0.0;
        candidates[pixelPos]  = RTXPT_INVALID_LIGHT_INDEX;
    }

    void Add(float randomValue, uint candidateIndex, float candidateWeight, bool coherent)
    {
        candidateWeight = min(candidateWeight, LFR_MAX_WEIGHT);
        float sum = min(totalWeight[pixelPos] + candidateWeight, LFR_MAX_WEIGHT);
        totalWeight[pixelPos] = sum;

        if (sum > 0.0 && randomValue < candidateWeight / sum)
            candidates[pixelPos] = candidateIndex | (coherent ? LFR_SCREEN_SPACE_COHERENT_FLAG : 0u);
    }
};
```

- [ ] **Step 2: Add feedback insertion**

In `LightSampler.hlsli`, add:

```hlsl
void InsertFeedbackFromNEE(LightSamplerContext ctx, uint lightIndex, float3 contribution, float randomValue)
{
    if (ctx.ctrl.TemporalFeedbackRequired == 0u || lightIndex == RTXPT_INVALID_LIGHT_INDEX)
        return;

    const float avgContribution = max(0.0, dot(contribution, float3(0.2126, 0.7152, 0.0722)));
    if (avgContribution <= 0.0)
        return;

    float feedbackWeight = avgContribution;
    const float globalPdf = max(SampleGlobalPDF(lightIndex), 1e-6);
    feedbackWeight /= pow(globalPdf, 0.65);
    if (ctx.isScreenSpaceCoherent)
        feedbackWeight *= RTXPT_LIGHTING_SCREEN_SPACE_COHERENT_FEEDBACK_BIAS;

    LightFeedbackReservoir reservoir = LightFeedbackReservoir::make(ctx.pixelPos, u_FeedbackTotalWeight, u_FeedbackCandidates);
    reservoir.Add(randomValue, lightIndex, feedbackWeight, ctx.isScreenSpaceCoherent);
}
```

- [ ] **Step 3: Call feedback insertion after visible NEE samples**

In `PathTracer::SampleDirectLightNEE`, after `contribution` is computed and before `result += contribution`, add:

```hlsl
const uint feedbackLightIndex =
    picked.kind == kLightProxyKindAnalytic ? picked.index : lightSampler.ctrl.AnalyticLightCount;
InsertFeedbackFromNEE(lightSampler, feedbackLightIndex, contribution, sampleNext1D(sg));
```

- [ ] **Step 4: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightingTypes.hlsli Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): write NEE feedback reservoirs" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Build Local Sampling Buffers For NEE-AT

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBakerPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBakerPass.cpp`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightsBaker.hlsl`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.{hpp,cpp}`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create a multi-entry compute pass wrapper**

Create `RTXPTLightsBakerPass.hpp` with:

```cpp
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

class RTXPTLightsBakerPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, const char* Name, const char* EntryPoint);
    bool Bind(IBuffer* pControl, IBuffer* pProxyCounters, IBuffer* pProxyIndices, IBuffer* pLocalSamplingBuffer,
              ITextureView* pFeedbackTotalWeightSRV, ITextureView* pFeedbackCandidatesSRV);
    bool Dispatch(IDeviceContext* pContext, Uint32 ThreadGroupsX, Uint32 ThreadGroupsY);

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
};

} // namespace Diligent
```

Implement it like `RTXPTEmissiveTrianglePass`, but compile `PathTracer/Lighting/LightsBaker.hlsl` and use the provided `EntryPoint`.

- [ ] **Step 2: Create `LightsBaker.hlsl` with clear and local-fill passes**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightsBaker.hlsl`:

```hlsl
#include "LightingTypes.hlsli"

StructuredBuffer<LightingControlData> t_LightingControl;
Buffer<uint>                          t_LightProxyCounters;
Buffer<uint>                          t_LightSamplingProxies;
Texture2D<float>                      t_FeedbackTotalWeight;
Texture2D<uint>                       t_FeedbackCandidates;
RWBuffer<uint>                        u_LocalSamplingBuffer;

[numthreads(8, 8, 1)]
void ClearLocalSamplingCS(uint3 tid : SV_DispatchThreadID)
{
    const LightingControlData ctrl = t_LightingControl[0];
    if (tid.x >= ctrl.LocalSamplingResolution.x || tid.y >= ctrl.LocalSamplingResolution.y)
        return;

    const uint base = LLSB_ComputeBaseAddress(tid.xy, ctrl.LocalSamplingResolution);
    [loop]
    for (uint i = 0; i < RTXPT_LIGHTING_LOCAL_PROXY_COUNT; ++i)
        u_LocalSamplingBuffer[base + i] = PackMiniListLightAndCount(0u, 1u);
}

[numthreads(8, 8, 1)]
void FillLocalSamplingCS(uint3 tid : SV_DispatchThreadID)
{
    const LightingControlData ctrl = t_LightingControl[0];
    if (tid.x >= ctrl.LocalSamplingResolution.x || tid.y >= ctrl.LocalSamplingResolution.y || ctrl.SamplingProxyCount == 0u)
        return;

    const uint2 pixelBase = tid.xy * RTXPT_LIGHTING_SAMPLING_BUFFER_TILE_SIZE;
    const uint base = LLSB_ComputeBaseAddress(tid.xy, ctrl.LocalSamplingResolution);

    uint feedbackLight = RTXPT_INVALID_LIGHT_INDEX;
    float feedbackWeight = 0.0;
    [loop]
    for (uint y = 0; y < RTXPT_LIGHTING_SAMPLING_BUFFER_TILE_SIZE; ++y)
    {
        [loop]
        for (uint x = 0; x < RTXPT_LIGHTING_SAMPLING_BUFFER_TILE_SIZE; ++x)
        {
            const uint2 pixel = pixelBase + uint2(x, y);
            const uint candidate = t_FeedbackCandidates.Load(int3(pixel, 0)) & ~LFR_SCREEN_SPACE_COHERENT_FLAG;
            const float weight = t_FeedbackTotalWeight.Load(int3(pixel, 0));
            if (candidate != RTXPT_INVALID_LIGHT_INDEX && weight > feedbackWeight)
            {
                feedbackLight = candidate;
                feedbackWeight = weight;
            }
        }
    }

    [loop]
    for (uint i = 0; i < RTXPT_LIGHTING_LOCAL_PROXY_COUNT; ++i)
    {
        uint lightIndex = t_LightSamplingProxies[min(i % ctrl.SamplingProxyCount, ctrl.SamplingProxyCount - 1u)];
        uint count = max(1u, t_LightProxyCounters[lightIndex]);
        if (feedbackLight != RTXPT_INVALID_LIGHT_INDEX && i < RTXPT_LIGHTING_SAMPLING_BUFFER_WINDOW_SIZE)
        {
            lightIndex = feedbackLight;
            count = RTXPT_LIGHTING_SAMPLING_BUFFER_WINDOW_SIZE;
        }
        u_LocalSamplingBuffer[base + i] = PackMiniListLightAndCount(lightIndex, count);
    }
}
```

- [ ] **Step 3: Dispatch baker local passes in `UpdateEnd`**

In `RTXPTLightsBaker.hpp`, add two pass members:

```cpp
RTXPTLightsBakerPass m_ClearLocalSamplingPass;
RTXPTLightsBakerPass m_FillLocalSamplingPass;
```

In `CreateResources`, initialize both passes after resources are created using the `pEngineFactory` parameter added in Task 2:

```cpp
m_ClearLocalSamplingPass.Initialize(pDevice, pEngineFactory, "RTXPT LightsBaker clear local sampling", "ClearLocalSamplingCS");
m_FillLocalSamplingPass.Initialize(pDevice, pEngineFactory, "RTXPT LightsBaker fill local sampling", "FillLocalSamplingCS");
```

In `UpdateEnd`, dispatch:

```cpp
const Uint32 GroupsX = (m_Stats.LocalSamplingTileCountX + 7u) / 8u;
const Uint32 GroupsY = (m_Stats.LocalSamplingTileCountY + 7u) / 8u;
m_ClearLocalSamplingPass.Bind(m_ControlBuffer, m_LightProxyCounters, m_LightSamplingProxies, m_LocalSamplingBuffer,
                              m_FeedbackTotalWeightSRV, m_FeedbackCandidatesSRV);
m_ClearLocalSamplingPass.Dispatch(pContext, GroupsX, GroupsY);
m_FillLocalSamplingPass.Bind(m_ControlBuffer, m_LightProxyCounters, m_LightSamplingProxies, m_LocalSamplingBuffer,
                             m_FeedbackTotalWeightSRV, m_FeedbackCandidatesSRV);
m_FillLocalSamplingPass.Dispatch(pContext, GroupsX, GroupsY);
```

- [ ] **Step 4: Schedule `UpdateEnd` before tracing**

In `RTXPTSample::Render`, before `m_RayTracingPass.Trace(...)`, call:

```cpp
m_LightsBaker.UpdateEnd(m_pImmediateContext);
```

This keeps the upstream ordering: baker update completes before NEE samples read the local sampler.

- [ ] **Step 5: Register new files**

Add to `SOURCE`:

```cmake
    src/RTXPTLightsBakerPass.cpp
```

Add to `INCLUDE`:

```cmake
    src/RTXPTLightsBakerPass.hpp
```

Add to `SHADERS`:

```cmake
    assets/shaders/PathTracer/Lighting/LightsBaker.hlsl
```

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTLightsBakerPass.hpp Samples/RTXPT/src/RTXPTLightsBakerPass.cpp Samples/RTXPT/src/RTXPTLightsBaker.hpp Samples/RTXPT/src/RTXPTLightsBaker.cpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightsBaker.hlsl Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): build LightsBaker local sampling buffers" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 8: Enable NEE-AT UI And Settings

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`

- [ ] **Step 1: Allow `NEEType == 2` in frame constants**

In `UpdateFrameConstants`, replace:

```cpp
m_LastFrameConstants.ptConsts.NEEType = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEType, 0, 1));
```

with:

```cpp
m_LastFrameConstants.ptConsts.NEEType = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEType, 0, 2));
```

Replace `NEEMISType` clamp to `0` with:

```cpp
m_LastFrameConstants.ptConsts.NEEMISType = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEMISType, 0, 2));
```

- [ ] **Step 2: Make the combo include NEE-AT**

In `UpdateUI`, replace:

```cpp
const char* SamplingTechniqueItems = "Uniform\0Power+\0\0";
```

with:

```cpp
const char* SamplingTechniqueItems = "Uniform\0Power+\0NEE-AT\0\0";
```

Clamp `m_ReferenceUI.NEEType` to `0, 2`.

- [ ] **Step 3: Add NEE-AT controls under "Light pre-processing and sampling"**

In the `Light pre-processing and sampling` section, add:

```cpp
ImGui::TextColored(CategoryColor, "NEE-AT settings:");
if (m_ReferenceUI.NEEType != 2)
{
    ImGui::TextWrapped("NOTE: NEE-AT inactive (enable in Path Tracer -> Next Event Estimation settings).");
}
else
{
    ResetOnChange(ImGui::SliderFloat("Global feedback weight", &m_ReferenceUI.NEEAT_GlobalTemporalFeedbackWeight, 0.0f, 0.95f),
                  "NEE-AT global feedback changed");
    ResetOnChange(ImGui::SliderFloat("Local to global sampler ratio", &m_ReferenceUI.NEEAT_LocalToGlobalSampleRatio, 0.0f, 0.95f),
                  "NEE-AT local/global ratio changed");
    ResetOnChange(ImGui::SliderFloat("Distant vs Local initial importance", &m_ReferenceUI.NEEAT_DistantVsLocalImportance, 0.01f, 100.0f, "%.2f"),
                  "NEE-AT distant/local importance changed");
}
```

- [ ] **Step 4: Add baker stats readouts**

Near the existing light stats, add:

```cpp
const RTXPTLightsBakerStats& BakerStats = m_LightsBaker.GetStats();
ImGui::Text("LightsBaker lights: %u", BakerStats.TotalLightCount);
ImGui::Text("LightsBaker proxies: %u", BakerStats.SamplingProxyCount);
ImGui::Text("LightsBaker feedback: %s", BakerStats.FeedbackReady ? "ready" : "missing");
ImGui::Text("LightsBaker update: %u", BakerStats.UpdateCounter);
```

- [ ] **Step 5: Source-check UI is live**

Run:

```powershell
rg -n "NEE-AT|Global feedback weight|Local to global sampler ratio|LightsBaker proxies|std::clamp\\(m_ReferenceUI\\.NEEType, 0, 2\\)" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: all patterns are present.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose LightsBaker NEE-AT controls" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 9: Land G6 Photometric And Shaped Punctual Lights On Baker Weights

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`

- [ ] **Step 1: Replace light layout with unit-aware fields**

In `RTXPTLights.hpp` and `PathTracerShared.h`, replace `PolymorphicLightInfo` with:

```cpp
struct PolymorphicLightInfo
{
    float4 colorType      = float4{0, 0, 0, -1};
    float4 positionRadius = float4{0, 0, 0, 0};
    float4 directionRange = float4{0, -1, 0, 0};
    float4 shaping        = float4{-1, 0, 0, 0};
};
static_assert(sizeof(PolymorphicLightInfo) == 64, "PolymorphicLightInfo layout must match PathTracer/PathTracerShared.h");
```

Use the HLSL equivalent in `PathTracerShared.h` and add:

```hlsl
static const uint kPolymorphicLightTypeSphere      = 0u;
static const uint kPolymorphicLightTypeDirectional = 2u;
static const uint kPolymorphicLightTypePoint       = 4u;
```

- [ ] **Step 2: Convert CPU lights to RTXPT units**

In `RTXPTLights.cpp`, replace `MakeLightData` with helpers that create finite sphere lights for point/spot lights and finite angular-size directional lights:

```cpp
constexpr float kPolymorphicLightTypeSphere      = 0.0f;
constexpr float kPolymorphicLightTypeDirectional = 2.0f;
constexpr float kDefaultPunctualLightRadius      = 0.01f;
constexpr float kDefaultDirectionalAngularSize   = 0.53f;

PolymorphicLightInfo MakeSphereLightData(const float3& Color, float Intensity, const float4x4& Transform,
                                         float Range, float Radius, float InnerCone, float OuterCone)
{
    const float ClampedRadius = std::max(Radius, 1.0e-4f);
    const float3 Radiance = Color * std::max(Intensity, 0.0f) / (PI_F * ClampedRadius * ClampedRadius);

    PolymorphicLightInfo Data;
    Data.colorType      = float4{Radiance.x, Radiance.y, Radiance.z, kPolymorphicLightTypeSphere};
    Data.positionRadius = float4{Transform._41, Transform._42, Transform._43, ClampedRadius};
    Data.directionRange = float4{-Transform._31, -Transform._32, -Transform._33, Range};
    Data.shaping        = float4{std::cos(OuterCone), std::max(0.0f, std::cos(InnerCone) - std::cos(OuterCone)), 0.0f, 0.0f};
    return Data;
}

PolymorphicLightInfo MakeDirectionalLightData(const float3& Color, float Intensity, const float4x4& Transform, float AngularSizeRadians)
{
    const float HalfAngle = std::max(AngularSizeRadians * 0.5f, 0.00001f);
    const float SolidAngle = std::max(2.0f * PI_F * (1.0f - std::cos(HalfAngle)), 1.0e-8f);
    const float3 Radiance = Color * std::max(Intensity, 0.0f) / SolidAngle;

    PolymorphicLightInfo Data;
    Data.colorType      = float4{Radiance.x, Radiance.y, Radiance.z, kPolymorphicLightTypeDirectional};
    Data.directionRange = float4{-Transform._31, -Transform._32, -Transform._33, 0.0f};
    Data.shaping        = float4{-1.0f, 0.0f, 0.0f, SolidAngle};
    return Data;
}
```

Then call these helpers from both GLTF and scene-json upload paths.

- [ ] **Step 3: Replace HLSL analytic light sampling**

In `PolymorphicLight.hlsli`, replace `EvalAnalyticLight` with `SampleAnalyticLight(PolymorphicLightInfo light, float2 random, float3 surfacePos)`, returning `LightSample` with `solidAnglePdf`. Sphere lights sample the finite solid angle; directional lights sample inside `shaping.w`; point fallback uses inverse-square pdf `1`.

- [ ] **Step 4: Use solid-angle pdf in baker-backed RIS**

In `GenerateDirectLightCandidate`, replace:

```hlsl
const LightSample light = EvalAnalyticLight(Bridge::getLight(lightIndex), hitPos);
...
sample.radianceOverPdf = radiance / selectionPdf;
sample.proposalPdf     = selectionPdf;
```

with:

```hlsl
const LightSample light = SampleAnalyticLight(Bridge::getLight(lightIndex), sampleNext2D(sg), hitPos);
if (!light.valid || light.solidAnglePdf <= 0.0)
    return sample;

const float lightPdf = selectionPdf * light.solidAnglePdf;
sample.radianceOverPdf = light.radiance * max(g_Const.ptConsts.lightIntensityScale, 0.0) / lightPdf;
sample.proposalPdf     = lightPdf;
```

- [ ] **Step 5: Update baker proxy weights**

In `RTXPTLightsBaker.cpp`, replace `EstimateAnalyticWeight` with:

```cpp
float EstimateAnalyticWeight(const PolymorphicLightInfo& Light)
{
    const float Luma = Light.colorType.x * 0.2126f + Light.colorType.y * 0.7152f + Light.colorType.z * 0.0722f;
    const float Radius = std::max(Light.positionRadius.w, 0.0f);
    const float AreaOrSolidAngle = Light.colorType.w == 2.0f ? std::max(Light.shaping.w, 1.0e-8f) :
        std::max(PI_F * Radius * Radius, 1.0f);
    return std::max(1.0e-6f, Luma * AreaOrSolidAngle);
}
```

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp Samples/RTXPT/src/RTXPTLightsBaker.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): convert punctual lights to RTXPT units" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 10: Documentation And Stale Marker Cleanup

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Update mapping rows**

In `RTXPT_FORK_MAPPING.md`, replace the old direct proxy rows with:

```markdown
| `LightsBaker` | `RTXPTLightsBaker` | Diligent-native implementation of the baker lifecycle; owns control/proxy/feedback/local sampling resources |
| `LightingControlData` | `LightingControlData` | Ported layout in `Lighting/LightingTypes.hlsli`; CPU mirror is local to `RTXPTLightsBaker.cpp` |
| `LightSampler::SampleGlobal` | `SampleGlobalLightIndex` | Uses `t_LightProxyCounters` + `t_LightSamplingProxies`, not the old `RTXPTLightProxy` prefix CDF |
| `LightSampler::SampleLocal` | `SampleLocalLightIndex` | Uses `t_LocalSamplingBuffer`; populated by `LightsBaker.hlsl` |
| `LightFeedbackReservoir` | `LightFeedbackReservoir` | Raygen writes NEE feedback into baker-owned UAV textures |
```

- [ ] **Step 2: Retarget stale compute-pass marker**

In `RTXPTComputePass.cpp`, replace:

```cpp
// TODO(RTXPT-Port Phase 4): Restore RTXDI DI/GI, light feedback, and denoising-guide compute chains; current helper runs only the debug color pass.
```

with:

```cpp
// TODO(RTXPT-Port Phase 5.5): Restore RTXDI DI/GI and denoising-guide compute chains; R3 LightsBaker feedback uses RTXPTLightsBakerPass.
```

- [ ] **Step 3: Retarget raygen TODO**

In `PathTracerSample.rgen`, ensure no marker says light RIS/NEE-AT is unimplemented. The remaining marker should name R4 only:

```hlsl
// TODO(RTXPT-Port Phase R4): Add HDR environment-map importance sampling + MIS.
```

- [ ] **Step 4: Source-check stale direct implementation references**

Run:

```powershell
rg -n "RTXPTLightProxy|t_LightProxies|UploadLightProxyBuffer|GetLightProxyBuffer|NEE-AT requires RTXPT-fork|RIS/WRS is Phase R3" DiligentSamples/Samples/RTXPT
```

Expected: no matches.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md Samples/RTXPT/src/RTXPTComputePass.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "docs(rtxpt): record LightsBaker migration" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 11: Final Verification

**Files:**
- Verify: all files touched by Tasks 1-10

- [ ] **Step 1: Run stale-symbol scan**

```powershell
rg -n "RTXPTLightProxy|t_LightProxies|UploadLightProxyBuffer|GetLightProxyBuffer|LightProxyBridgeBound|NEE-AT requires RTXPT-fork" DiligentSamples/Samples/RTXPT
```

Expected: no matches.

- [ ] **Step 2: Run new-symbol scan**

```powershell
rg -n "RTXPTLightsBaker|LightingControlData|t_LightingControl|t_LightProxyCounters|t_LightSamplingProxies|t_LocalSamplingBuffer|u_FeedbackTotalWeight|SampleLocalLightIndex|InsertFeedbackFromNEE" DiligentSamples/Samples/RTXPT
```

Expected: all symbols are present.

- [ ] **Step 3: Run whitespace validation**

```powershell
git -C DiligentSamples diff --check
```

Expected: no output.

- [ ] **Step 4: Build when the user explicitly requests verification**

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If shader compilation fails, fix missing static bindings or struct-layout mismatches before proceeding.

- [ ] **Step 5: Manual GPU smoke when requested**

Run D3D12 and Vulkan. For each backend:

```text
1. Load a many-light scene.
2. Compare Uniform, Power+, and NEE-AT at the same sample count.
3. Expected: brightness remains stable; variance changes.
4. Move camera to invalidate feedback.
5. Expected: NEE-AT recovers without stale-light artifacts; feedback/local buffers rebuild.
6. Toggle NEE off/on.
7. Expected: converged image is unchanged; direct lighting only changes variance/noise.
```

- [ ] **Step 6: Final status**

```powershell
git -C DiligentSamples status --short
git status --short
```

Expected: `DiligentSamples` is clean except for intentional user changes. Top-level repo may show a modified submodule pointer only if the user asked to record submodule commits.

## Self-Review

- [x] **Spec coverage:** Covers full R3: `LightsBaker` lifecycle/resources, global Power+ proxies, NEE feedback, local sampling / NEE-AT, debug UI, and G6 photometric/shaped punctual-light units.
- [x] **Migration constraint:** Explicitly starts from the current direct `RTXPTLightProxy` implementation and removes it in favor of baker-owned control/proxy/feedback/local resources.
- [x] **Type consistency:** `LightingControlData`, proxy counters/indices, feedback textures, and local sampling buffers are named consistently across C++ and HLSL tasks.
- [x] **Verification:** Includes stale-symbol scans, source checks, build command, and manual D3D12/Vulkan acceptance checks.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-31-rtxpt-phase-r3-lightsbaker-migration.md`. Two execution options:

**1. Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** - Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Choose one before implementation begins.
