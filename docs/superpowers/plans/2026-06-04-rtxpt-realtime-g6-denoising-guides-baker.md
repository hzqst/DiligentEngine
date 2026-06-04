# RTXPT Realtime G6 Denoising Guides Baker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port RTXPT-fork's `DenoisingGuidesBaker` into the Diligent RTXPT sample as Diligent-native compute passes that run after realtime stable-plane `PathTrace` and before later no-denoiser/NRD final merge work.

**Architecture:** Add a focused `RTXPTDenoisingGuidesBaker` owner with three compute PSOs: `DenoiseSpecHitT`, `ComputeAvgLayerRadiance`, and `DebugViz`. The pass reads and writes the realtime resources already owned by `RTXPTRenderTargets`, uses the same `SampleConstants` frame constant buffer as the realtime path tracer, and uses a local 64-byte denoising-guide constant buffer mirroring RTXPT-fork's `DenoisingGuidesBakerConstants`. `RTXPTSample::PathTrace()` remains the orchestration point and calls the baker immediately after the FILL stable-plane loop.

**Tech Stack:** C++17, HLSL 6.x/DXC, Diligent `IRenderDevice`/`IDeviceContext`/compute PSO/SRB APIs, `RTXPTRenderTargets` realtime resources, ImGui status/debug controls, RTXPT-fork reference source under `D:/RTXPT-fork/Rtxpt`, PowerShell + `rg` verification.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`, goal `G6 - Denoising Guides Baker`.
- G1-G5 state is present in this checkout:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp` owns realtime/NRD settings and reset flags.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` and `assets/shaders/PathTracer/PathTracerShared.h` expose frame constants and `SampleMiniConstants`.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}` allocate `SpecularHitT`, `ScratchFloat1`, stable-plane resources, and `DenoiserAvgLayerRadianceHalfRes`.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` creates REF/BUILD/FILL RT variants and binds `u_SpecularHitT` plus stable-plane resources for realtime variants.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp::PathTrace()` runs BUILD, `LightsBaker.UpdateEnd`, and FILL, then reports a disabled denoising-guides hook.
- Current G6 gaps:
  - No Diligent `RTXPTDenoisingGuidesBaker` class exists.
  - No Diligent shader equivalent of `D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.hlsl` exists.
  - `RTXPTSample::PathTrace()` does not smooth `SpecularHitT`, does not compute average layer radiance, and does not expose guide debug views.
- Original RTXPT-fork invariants:
  - `DenoisingGuidesBakerConstants` is 64 bytes and matches `SampleMiniConstants` size in RTXPT-fork.
  - `DenoiseSpecHitT()` dispatches full render size twice: ping writes `ScratchFloat1`, pong writes back `SpecularHitT`.
  - `ComputeAvgLayerRadiance()` dispatches at `DenoiserAvgLayerRadianceHalfRes` dimensions.
  - `RenderDebugViz()` dispatches full render size.
  - In `Sample.cpp::PathTrace()`, the order is FILL sub-sample loop, `DenoiseSpecHitT`, `ComputeAvgLayerRadiance`, optional denoiser-guide debug visualization, optional stable-plane debug visualization.

## Scope Boundaries

- This plan implements G6 only: denoising-guide bake passes and their realtime `PathTrace` call point.
- This plan does not implement NRD, denoiser prepare/final merge, `NoDenoiserFinalMerge`, `StablePlanesDebugViz`, TAA/SR handoff, or DLSS-RR.
- `ComputeAvgLayerRadiance` must be implemented as a live Diligent shader, not copied as the upstream `#if 0` disabled body. The implementation should keep the upstream formula and use the existing Diligent stable-plane helpers.
- Debug visualization writes to `ProcessedOutputColor` only when a realtime guide debug view is selected. This is a debug presentation route, not the realtime final image route.
- Reference mode must remain unchanged: no denoising-guide pass initializes or dispatches for the reference-only frame path.
- No NVIDIA license headers are copied into Diligent files. C++ files use the existing Diligent Apache header; shaders stay header-less like the existing RTXPT shader tree.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
  - Adds `RTXPTDenoisingGuideDebugView` and a realtime setting for guide debug output.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp`
  - Declares stats, dispatch contract, and pass owner.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.cpp`
  - Creates the three compute PSOs, binds frame/local constants and realtime resources, dispatches guide passes, and records stats.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl`
  - Diligent-native shader port of `DenoiseSpecHitT`, `ComputeAvgLayerRadiance`, and `DebugViz`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Adds `RTXPTDenoisingGuidesBaker` member and helper declarations.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Initializes/resets the baker, calls it from `PathTrace`, adds optional debug presentation, and exposes UI/status.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Registers the new C++ and shader files.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  - Records the G6 source-to-port mapping and known divergence for the debug output target.

---

### Task 0: Baseline Preflight and Source Contract

**Files:**
- Verify: top-level repository
- Verify: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}`
- Verify: `D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.{h,cpp,hlsl}`
- Verify: `D:/RTXPT-fork/Rtxpt/Sample.cpp`

- [ ] **Step 1: Confirm workspace state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Preserve any existing dirty files; do not revert user changes.

- [ ] **Step 2: Confirm G6 source anchors**

Run:

```powershell
rg -n "DenoisingGuidesBaker|DenoiseSpecHitT|ComputeAvgLayerRadiance|RenderDebugViz|SpecularHitT|ScratchFloat1|DenoiserAvgLayerRadianceHalfRes" D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.h D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.cpp D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.hlsl D:/RTXPT-fork/Rtxpt/Sample.cpp DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected:

```text
RTXPT-fork files contain DenoiseSpecHitT, ComputeAvgLayerRadiance, and RenderDebugViz.
Diligent RTXPTRenderTargets contains SpecularHitT, ScratchFloat1, and DenoiserAvgLayerRadianceHalfRes accessors.
Diligent RTXPTSample::PathTrace contains a disabled denoising-guides hook after the FILL loop.
```

- [ ] **Step 3: Confirm no existing G6 owner**

Run:

```powershell
rg -n "RTXPTDenoisingGuidesBaker|DenoisingGuidesBaker.hlsl|DenoisingGuideDebugView" DiligentSamples/Samples/RTXPT
```

Expected before implementation: no matches.

- [ ] **Step 4: No commit for preflight**

No source changes are made in Task 0.

### Task 1: Add Realtime Guide Debug Setting

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add the guide debug enum**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`, add this enum after `RTXPTNrdMethod`:

```cpp
enum class RTXPTDenoisingGuideDebugView : Uint32
{
    Disabled         = 0,
    Depth            = 1,
    MotionVectors    = 2,
    SpecularHitT     = 3,
    AvgLayerRadiance = 4,
    PrimaryLayer     = 5
};
```

- [ ] **Step 2: Add the setting field**

In `RTXPTRealtimeSettings`, add this field after `DenoiserRadianceClampK`:

```cpp
    RTXPTDenoisingGuideDebugView DenoisingGuideDebugView = RTXPTDenoisingGuideDebugView::Disabled;
```

- [ ] **Step 3: Sanitize the setting**

In `SanitizeRealtimeSettings`, add this block after the `RealtimeAA` clamp:

```cpp
    const Uint32 GuideDebugView = std::clamp(static_cast<Uint32>(Settings.DenoisingGuideDebugView),
                                             static_cast<Uint32>(RTXPTDenoisingGuideDebugView::Disabled),
                                             static_cast<Uint32>(RTXPTDenoisingGuideDebugView::PrimaryLayer));
    Settings.DenoisingGuideDebugView = static_cast<RTXPTDenoisingGuideDebugView>(GuideDebugView);
```

- [ ] **Step 4: Add a display-name helper**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, add this helper after `GetNrdMethodName`:

```cpp
const char* GetDenoisingGuideDebugViewName(RTXPTDenoisingGuideDebugView View)
{
    switch (View)
    {
        case RTXPTDenoisingGuideDebugView::Disabled: return "Disabled";
        case RTXPTDenoisingGuideDebugView::Depth: return "Depth";
        case RTXPTDenoisingGuideDebugView::MotionVectors: return "Motion Vectors";
        case RTXPTDenoisingGuideDebugView::SpecularHitT: return "Specular Hit T";
        case RTXPTDenoisingGuideDebugView::AvgLayerRadiance: return "Average Layer Radiance";
        case RTXPTDenoisingGuideDebugView::PrimaryLayer: return "Primary Layer";
        default: return "Unknown";
    }
}
```

- [ ] **Step 5: Add the realtime UI combo**

In `RTXPTSample::UpdateUI()`, inside the realtime `"Post processing:"` block immediately after the standalone denoiser checkbox, add:

```cpp
                const char* GuideDebugItems[] = {
                    "Disabled",
                    "Depth",
                    "Motion Vectors",
                    "Specular Hit T",
                    "Average Layer Radiance",
                    "Primary Layer"};
                int GuideDebugView = static_cast<int>(m_RealtimeUI.DenoisingGuideDebugView);
                if (ImGui::Combo("Denoising guide debug", &GuideDebugView, GuideDebugItems, _countof(GuideDebugItems)))
                {
                    m_RealtimeUI.DenoisingGuideDebugView =
                        static_cast<RTXPTDenoisingGuideDebugView>(std::clamp(GuideDebugView, 0, static_cast<int>(_countof(GuideDebugItems) - 1)));
                    RequestRealtimeReset(RTXPT_REALTIME_RESET_TAA_SR_HISTORY, "Denoising guide debug view changed");
                }
                if (ImGui::IsItemHovered())
                    ImGui::SetTooltip("Displays guide debug output from the G6 denoising-guides pass when realtime mode is active.");
```

- [ ] **Step 6: Verify setting symbols**

Run:

```powershell
rg -n "RTXPTDenoisingGuideDebugView|DenoisingGuideDebugView|GetDenoisingGuideDebugViewName|Denoising guide debug" DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: all added symbols are found.

- [ ] **Step 7: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): add denoising guide debug setting" -m "Co-Authored-By: GPT 5.5"
```

### Task 2: Add the Denoising Guides Shader

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl`
- Read: `D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.hlsl`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`

- [ ] **Step 1: Create the shader file**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl` with this content:

```hlsl
#ifndef __DENOISING_GUIDES_BAKER__
#define __DENOISING_GUIDES_BAKER__

#include "PathTracerShared.h"

struct DenoisingGuidesBakerConstants
{
    uint2 RenderResolution;
    uint2 DisplayResolution;

    int  DebugView;
    uint Ping;
    uint _padding1;
    uint _padding2;

    uint4 _padding3;
    uint4 _padding4;
};

#define DGB_2D_THREADGROUP_SIZE 8

#if !defined(__cplusplus)

#include "PathTracerHelpers.hlsli"
#include "StablePlanes.hlsli"

ConstantBuffer<SampleConstants>                g_Const;
ConstantBuffer<DenoisingGuidesBakerConstants>  g_DenoisingGuidesBakerConstants;

Texture2D<float>                               t_Depth;
Texture2D<float2>                              t_MotionVectors;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>  u_SpecularHitT;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>  u_ScratchFloat1;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_StableRadiance;
VK_IMAGE_FORMAT("r32ui")   RWTexture2DArray<uint> u_StablePlanesHeader;
RWStructuredBuffer<StablePlane>                u_StablePlanesBuffer;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_DenoiserAvgLayerRadianceHalfRes;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_DebugOutput;

static const int RTXPT_DENOISING_GUIDE_DEBUG_DISABLED          = 0;
static const int RTXPT_DENOISING_GUIDE_DEBUG_DEPTH             = 1;
static const int RTXPT_DENOISING_GUIDE_DEBUG_MOTION_VECTORS    = 2;
static const int RTXPT_DENOISING_GUIDE_DEBUG_SPECULAR_HIT_T    = 3;
static const int RTXPT_DENOISING_GUIDE_DEBUG_AVG_LAYER_RADIANCE = 4;
static const int RTXPT_DENOISING_GUIDE_DEBUG_PRIMARY_LAYER     = 5;

float3 DenoisingGuidesHeatMap(float GreyValue)
{
    const float T = saturate(GreyValue);
    return saturate(float3(1.5 - abs(4.0 * T - 3.0),
                           1.5 - abs(4.0 * T - 2.0),
                           1.5 - abs(4.0 * T - 1.0)));
}

float SpecHitTNeighbourhood(RWTexture2D<float> TexSrc, int2 PixelPos)
{
    const float CenterD = t_Depth[PixelPos];
    float PrevHitT = max(0.0, TexSrc[PixelPos]);

    const float MinSpecHitT = 5e-2f;
    if (PrevHitT < MinSpecHitT)
        PrevHitT = 0.0;

    const int NeighbourRadius = 2;
    uint Width = 0;
    uint Height = 0;
    TexSrc.GetDimensions(Width, Height);

    float ValueAverage = PrevHitT;
    float SumWeight = PrevHitT > 0.0;
    for (int X = -NeighbourRadius; X <= NeighbourRadius; ++X)
    {
        for (int Y = -NeighbourRadius; Y <= NeighbourRadius; ++Y)
        {
            if (X == 0 && Y == 0)
                continue;

            const int2 NeighbourPixelPos = PixelPos + int2(X, Y);
            if (NeighbourPixelPos.x < 0 || NeighbourPixelPos.y < 0 ||
                NeighbourPixelPos.x >= Width || NeighbourPixelPos.y >= Height)
                continue;

            const float V = min(TexSrc[NeighbourPixelPos], HLF_MAX);
            const float D = max(0.0, t_Depth[NeighbourPixelPos]);
            const float DepthThreshold = 0.025;
            float Weight = V > 0.0;
            Weight *= abs(D - CenterD) <= (D + CenterD + 1e-5f) * DepthThreshold;

            if (Weight > 0.0)
            {
                ValueAverage += V * Weight;
                SumWeight += Weight;
            }
        }
    }

    if (SumWeight == 0.0)
        return PrevHitT;

    ValueAverage /= SumWeight;
    return PrevHitT <= 0.0 ? ValueAverage : min(PrevHitT * 1.5 + 0.5, ValueAverage);
}

[numthreads(DGB_2D_THREADGROUP_SIZE, DGB_2D_THREADGROUP_SIZE, 1)]
void DenoiseSpecHitT(uint2 DispatchThreadID : SV_DispatchThreadID)
{
    const int2 PixelPos = DispatchThreadID.xy;
    if (any(DispatchThreadID.xy >= g_DenoisingGuidesBakerConstants.RenderResolution))
        return;

    if (g_DenoisingGuidesBakerConstants.Ping != 0u)
        u_ScratchFloat1[PixelPos] = SpecHitTNeighbourhood(u_SpecularHitT, PixelPos);
    else
        u_SpecularHitT[PixelPos] = SpecHitTNeighbourhood(u_ScratchFloat1, PixelPos);
}

[numthreads(DGB_2D_THREADGROUP_SIZE, DGB_2D_THREADGROUP_SIZE, 1)]
void ComputeAvgLayerRadiance(uint2 DispatchThreadID : SV_DispatchThreadID)
{
    const uint2 HalfResPos = DispatchThreadID.xy;
    const uint2 HalfRes = (g_DenoisingGuidesBakerConstants.RenderResolution + 1u) / 2u;
    if (any(HalfResPos >= HalfRes))
        return;

    const uint2 RenderMax = max(g_DenoisingGuidesBakerConstants.RenderResolution, uint2(1u, 1u)) - 1u;
    const uint2 BasePixel = min(HalfResPos * 2u, RenderMax);
    const float2 ScreenSpaceMotion = t_MotionVectors[min(BasePixel + (HalfResPos & 1u), RenderMax)];
    const int2 HistoricPixel = int2(float2(BasePixel) + ScreenSpaceMotion + 0.5.xx);
    const uint2 HistoricHalfResPos = min(uint2(max(HistoricPixel, int2(0, 0))) / 2u, HalfRes - 1u);

    const float ExponentialFalloffK = saturate(0.05);
    float4 OutVal = clamp(u_DenoiserAvgLayerRadianceHalfRes[HistoricHalfResPos], 0.0, HLF_MAX) *
        (1.0 - ExponentialFalloffK);

    StablePlanesContext StablePlanes = StablePlanesContext::make(u_StablePlanesHeader,
                                                                  u_StablePlanesBuffer,
                                                                  u_StableRadiance,
                                                                  g_Const.ptConsts);

    for (uint StablePlaneIndex = 0;
         StablePlaneIndex < min(g_Const.ptConsts.GetActiveStablePlaneCount(), cStablePlaneCount);
         ++StablePlaneIndex)
    {
        float Avg = 0.0;
        float Count = 1e-7;
        for (uint X = 0; X < 2; ++X)
        {
            for (uint Y = 0; Y < 2; ++Y)
            {
                const uint2 PixelPos = min(BasePixel + uint2(X, Y), RenderMax);
                const uint BranchID = StablePlanes.GetBranchID(PixelPos, StablePlaneIndex);
                if (BranchID == cStablePlaneInvalidBranchID)
                    continue;

                StablePlane SP = StablePlanes.LoadStablePlane(PixelPos, StablePlaneIndex);
                float Radiance = max(1e-5, Average(SP.GetNoisyRadiance()));
                Radiance = min(Radiance, g_Const.ptConsts.preExposedGrayLuminance * 2.0);
                Avg += log(Radiance + 1.0);
                Count += 1.0;
            }
        }

        Avg = exp(Avg / Count) - 1.0;
        OutVal[StablePlaneIndex] += Avg * ExponentialFalloffK;
    }

    OutVal.w = OutVal.x + OutVal.y + OutVal.z;
    u_DenoiserAvgLayerRadianceHalfRes[HalfResPos] = OutVal;
}

uint SelectPrimaryLayer(float4 AvgLayerRadiance)
{
    uint PrimaryLayer = 0u;
    float BestValue = AvgLayerRadiance.x;
    if (AvgLayerRadiance.y > BestValue)
    {
        PrimaryLayer = 1u;
        BestValue = AvgLayerRadiance.y;
    }
    if (AvgLayerRadiance.z > BestValue)
        PrimaryLayer = 2u;
    return PrimaryLayer;
}

[numthreads(DGB_2D_THREADGROUP_SIZE, DGB_2D_THREADGROUP_SIZE, 1)]
void DebugViz(uint2 DispatchThreadID : SV_DispatchThreadID)
{
    const uint2 PixelPos = DispatchThreadID.xy;
    if (any(PixelPos >= g_DenoisingGuidesBakerConstants.RenderResolution))
        return;

    float4 Color = float4(0.0, 0.0, 0.0, 1.0);
    const int DebugView = g_DenoisingGuidesBakerConstants.DebugView;
    const uint2 HalfRes = (g_DenoisingGuidesBakerConstants.RenderResolution + 1u) / 2u;
    const uint2 HalfResPos = min(PixelPos / 2u, HalfRes - 1u);
    const float4 AvgLayerRadiance = u_DenoiserAvgLayerRadianceHalfRes[HalfResPos];

    if (DebugView == RTXPT_DENOISING_GUIDE_DEBUG_DEPTH)
    {
        Color = float4(saturate(t_Depth[PixelPos] * 100.0).xxx, 1.0);
    }
    else if (DebugView == RTXPT_DENOISING_GUIDE_DEBUG_MOTION_VECTORS)
    {
        const float2 Motion = t_MotionVectors[PixelPos];
        Color = float4(0.5.xx + Motion * 0.2, 0.0, 1.0);
    }
    else if (DebugView == RTXPT_DENOISING_GUIDE_DEBUG_SPECULAR_HIT_T)
    {
        Color = float4(DenoisingGuidesHeatMap(u_SpecularHitT[PixelPos] / 50.0), 1.0);
    }
    else if (DebugView == RTXPT_DENOISING_GUIDE_DEBUG_AVG_LAYER_RADIANCE)
    {
        Color = float4(sqrt(ReinhardMax(max(AvgLayerRadiance.xyz, 0.0.xxx))), 1.0);
    }
    else if (DebugView == RTXPT_DENOISING_GUIDE_DEBUG_PRIMARY_LAYER)
    {
        const uint PrimaryLayer = SelectPrimaryLayer(AvgLayerRadiance);
        Color = float4(PrimaryLayer == 0u, PrimaryLayer == 1u, PrimaryLayer == 2u, 1.0);
    }

    u_DebugOutput[PixelPos] = Color;
}

#endif

#endif // __DENOISING_GUIDES_BAKER__
```

- [ ] **Step 2: Verify shader anchors**

Run:

```powershell
rg -n "DenoisingGuidesBakerConstants|DenoiseSpecHitT|ComputeAvgLayerRadiance|DebugViz|u_SpecularHitT|u_ScratchFloat1|u_DenoiserAvgLayerRadianceHalfRes|u_DebugOutput" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl
```

Expected: every shader entry and resource binding is found.

- [ ] **Step 3: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl
git commit -m "feat(rtxpt): add denoising guides baker shader" -m "Co-Authored-By: GPT 5.5"
```

### Task 3: Add the Diligent Compute Pass Owner

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.cpp`

- [ ] **Step 1: Create the header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp` with this content:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 *  you may not use this file except in compliance with the License.
 *  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 *
 *  In no event and under no legal theory, whether in tort (including negligence),
 *  contract, or otherwise, unless required by applicable law (such as deliberate
 *  and grossly negligent acts) or agreed to in writing, shall any Contributor be
 *  liable for any damages, including any direct, indirect, special, incidental,
 *  or consequential damages of any character arising as a result of this License or
 *  out of the use or inability to use the software (including but not limited to damages
 *  for loss of goodwill, work stoppage, computer failure or malfunction, or any and
 *  all other commercial damages or losses), even if such Contributor has been advised
 *  of the possibility of such damages.
 */

#pragma once

#include <array>

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "RTXPTRealtimeSettings.hpp"
#include "RTXPTRenderTargets.hpp"
#include "ShaderResourceBinding.h"

namespace Diligent
{

struct RTXPTDenoisingGuidesBakerStats
{
    bool   Ready                              = false;
    bool   LastBakeExecuted                   = false;
    bool   LastDenoiseSpecHitTExecuted        = false;
    bool   LastAvgLayerRadianceExecuted       = false;
    bool   LastDebugVizExecuted               = false;
    Uint32 DenoiseSpecHitTDispatchCount       = 0;
    Uint32 AvgLayerRadianceDispatchCount      = 0;
    Uint32 DebugVizDispatchCount              = 0;
};

class RTXPTDenoisingGuidesBaker
{
public:
    enum class PassId : Uint32
    {
        DenoiseSpecHitT = 0,
        ComputeAvgLayerRadiance,
        DebugViz,
        Count
    };

    void Reset();

    bool Initialize(IRenderDevice*  pDevice,
                    IEngineFactory* pEngineFactory,
                    IBuffer*        pFrameConstants,
                    bool            ComputeSupported);

    bool Bake(IDeviceContext*                 pContext,
              const RTXPTRenderTargets&      RenderTargets,
              RTXPTDenoisingGuideDebugView   DebugView);

    bool IsReady() const { return m_Stats.Ready; }
    const RTXPTDenoisingGuidesBakerStats& GetStats() const { return m_Stats; }

private:
    struct PassState
    {
        RefCntAutoPtr<IPipelineState>         PSO;
        RefCntAutoPtr<IShaderResourceBinding> SRB;
    };

    bool CreatePass(IRenderDevice*                         pDevice,
                    IShaderSourceInputStreamFactory*       pShaderSourceFactory,
                    PassId                                 Pass);
    bool DispatchPass(IDeviceContext*               pContext,
                      const RTXPTRenderTargets&     RenderTargets,
                      PassId                        Pass,
                      RTXPTDenoisingGuideDebugView  DebugView,
                      Uint32                        Ping);

    std::array<PassState, static_cast<size_t>(PassId::Count)> m_Passes;
    RefCntAutoPtr<IBuffer>                                    m_FrameConstants;
    RefCntAutoPtr<IBuffer>                                    m_Constants;
    RTXPTDenoisingGuidesBakerStats                            m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Fix the long stats field names if clang-format reveals an alignment issue**

Verify the short stats fields are present:

Run:

```powershell
rg -n "LastAvgLayerRadianceExecuted|AvgLayerRadianceDispatchCount" DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp
```

Expected: two matches.

- [ ] **Step 3: Create the source file**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.cpp` with this content:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 *  you may not use this file except in compliance with the License.
 *  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 *
 *  In no event and under no legal theory, whether in tort (including negligence),
 *  contract, or otherwise, unless required by applicable law (such as deliberate
 *  and grossly negligent acts) or agreed to in writing, shall any Contributor be
 *  liable for any damages, including any direct, indirect, special, incidental,
 *  or consequential damages of any character arising as a result of this License or
 *  out of the use or inability to use the software (including but not limited to damages
 *  for loss of goodwill, work stoppage, computer failure or malfunction, or any and
 *  all other commercial damages or losses), even if such Contributor has been advised
 *  of the possibility of such damages.
 */

#include "RTXPTDenoisingGuidesBaker.hpp"

#include "DebugUtilities.hpp"
#include "GraphicsTypesX.hpp"
#include "MapHelper.hpp"

namespace Diligent
{

namespace
{

struct DenoisingGuidesBakerConstants
{
    uint2  RenderResolution  = uint2{0, 0};
    uint2  DisplayResolution = uint2{0, 0};
    Int32  DebugView         = 0;
    Uint32 Ping              = 0;
    Uint32 _Padding1         = 0;
    Uint32 _Padding2         = 0;
    uint4  _Padding3         = {};
    uint4  _Padding4         = {};
};
static_assert(sizeof(DenoisingGuidesBakerConstants) == 64, "DenoisingGuidesBakerConstants must match PathTracer/DenoisingGuidesBaker.hlsl");

constexpr Uint32 kThreadGroupSize = 8u;

const char* GetPassName(RTXPTDenoisingGuidesBaker::PassId Pass)
{
    switch (Pass)
    {
        case RTXPTDenoisingGuidesBaker::PassId::DenoiseSpecHitT: return "RTXPT DenoiseSpecHitT";
        case RTXPTDenoisingGuidesBaker::PassId::ComputeAvgLayerRadiance: return "RTXPT ComputeAvgLayerRadiance";
        case RTXPTDenoisingGuidesBaker::PassId::DebugViz: return "RTXPT DenoisingGuides DebugViz";
        default: return "RTXPT DenoisingGuides unknown pass";
    }
}

const char* GetEntryPoint(RTXPTDenoisingGuidesBaker::PassId Pass)
{
    switch (Pass)
    {
        case RTXPTDenoisingGuidesBaker::PassId::DenoiseSpecHitT: return "DenoiseSpecHitT";
        case RTXPTDenoisingGuidesBaker::PassId::ComputeAvgLayerRadiance: return "ComputeAvgLayerRadiance";
        case RTXPTDenoisingGuidesBaker::PassId::DebugViz: return "DebugViz";
        default: return "";
    }
}

void InsertUAVBarrier(IDeviceContext* pContext, ITextureView* pView)
{
    if (pContext == nullptr || pView == nullptr || pView->GetTexture() == nullptr)
        return;

    StateTransitionDesc Barrier{pView->GetTexture(),
                                RESOURCE_STATE_UNKNOWN,
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                STATE_TRANSITION_FLAG_UPDATE_STATE};
    pContext->TransitionResourceStates(1, &Barrier);
}

void InsertUAVBarrier(IDeviceContext* pContext, IBufferView* pView)
{
    if (pContext == nullptr || pView == nullptr || pView->GetBuffer() == nullptr)
        return;

    StateTransitionDesc Barrier{pView->GetBuffer(),
                                RESOURCE_STATE_UNKNOWN,
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                STATE_TRANSITION_FLAG_UPDATE_STATE};
    pContext->TransitionResourceStates(1, &Barrier);
}

} // namespace

void RTXPTDenoisingGuidesBaker::Reset()
{
    for (PassState& Pass : m_Passes)
    {
        Pass.PSO.Release();
        Pass.SRB.Release();
    }
    m_FrameConstants.Release();
    m_Constants.Release();
    m_Stats = {};
}

bool RTXPTDenoisingGuidesBaker::Initialize(IRenderDevice*  pDevice,
                                           IEngineFactory* pEngineFactory,
                                           IBuffer*        pFrameConstants,
                                           bool            ComputeSupported)
{
    Reset();

    if (!ComputeSupported)
    {
        DEV_ERROR("RTXPT denoising guides baker requires compute shader support");
        return false;
    }
    if (pDevice == nullptr || pEngineFactory == nullptr || pFrameConstants == nullptr)
    {
        DEV_ERROR("RTXPT denoising guides baker requires device, engine factory, and frame constants");
        return false;
    }

    m_FrameConstants = pFrameConstants;

    BufferDesc ConstantsDesc;
    ConstantsDesc.Name           = "RTXPT denoising guides baker constants";
    ConstantsDesc.Size           = sizeof(DenoisingGuidesBakerConstants);
    ConstantsDesc.BindFlags      = BIND_UNIFORM_BUFFER;
    ConstantsDesc.Usage          = USAGE_DYNAMIC;
    ConstantsDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    pDevice->CreateBuffer(ConstantsDesc, nullptr, &m_Constants);
    VERIFY(m_Constants, "Failed to create RTXPT denoising guides baker constants");
    if (!m_Constants)
        return false;

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PathTracer", &pShaderSourceFactory);
    if (!pShaderSourceFactory)
        return false;

    for (Uint32 Index = 0; Index < static_cast<Uint32>(PassId::Count); ++Index)
    {
        if (!CreatePass(pDevice, pShaderSourceFactory, static_cast<PassId>(Index)))
        {
            Reset();
            return false;
        }
    }

    m_Stats.Ready = true;
    return true;
}

bool RTXPTDenoisingGuidesBaker::CreatePass(IRenderDevice*                   pDevice,
                                           IShaderSourceInputStreamFactory* pShaderSourceFactory,
                                           PassId                           Pass)
{
    PassState& State = m_Passes[static_cast<size_t>(Pass)];

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.ShaderType            = SHADER_TYPE_COMPUTE;
    ShaderCI.Desc.Name                  = GetPassName(Pass);
    ShaderCI.SourceLanguage             = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler             = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags               = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.FilePath                   = "PathTracer/DenoisingGuidesBaker.hlsl";
    ShaderCI.EntryPoint                 = GetEntryPoint(Pass);
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pCS;
    pDevice->CreateShader(ShaderCI, &pCS);
    VERIFY(pCS, "Failed to create RTXPT denoising guides shader");
    if (!pCS)
        return false;

    ComputePipelineStateCreateInfo PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = GetPassName(Pass);
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_COMPUTE;
    PSOCreateInfo.pCS                  = pCS;

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_COMPUTE, "g_Const", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "g_DenoisingGuidesBakerConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_Depth", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_MotionVectors", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_SpecularHitT", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_ScratchFloat1", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_StableRadiance", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_StablePlanesHeader", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_StablePlanesBuffer", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserAvgLayerRadianceHalfRes", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_DebugOutput", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateComputePipelineState(PSOCreateInfo, &State.PSO);
    VERIFY(State.PSO, "Failed to create RTXPT denoising guides PSO");
    if (!State.PSO)
        return false;

    auto SetStatic = [&State](const char* Name, IDeviceObject* pObject) {
        IShaderResourceVariable* pVar = State.PSO->GetStaticVariableByName(SHADER_TYPE_COMPUTE, Name);
        if (pVar == nullptr)
            return true;
        if (pObject == nullptr)
            return false;
        pVar->Set(pObject);
        return true;
    };

    if (!SetStatic("g_Const", m_FrameConstants) ||
        !SetStatic("g_DenoisingGuidesBakerConstants", m_Constants))
    {
        DEV_ERROR("Failed to bind RTXPT denoising guides static resources");
        return false;
    }

    State.PSO->CreateShaderResourceBinding(&State.SRB, true);
    VERIFY(State.SRB, "Failed to create RTXPT denoising guides SRB");
    return State.SRB != nullptr;
}

bool RTXPTDenoisingGuidesBaker::Bake(IDeviceContext*               pContext,
                                     const RTXPTRenderTargets&    RenderTargets,
                                     RTXPTDenoisingGuideDebugView DebugView)
{
    m_Stats.LastBakeExecuted = false;
    m_Stats.LastDenoiseSpecHitTExecuted = false;
    m_Stats.LastAvgLayerRadianceExecuted = false;
    m_Stats.LastDebugVizExecuted = false;

    if (!IsReady() || pContext == nullptr || !RenderTargets.HasRealtimeRenderTargets())
        return false;

    if (!DispatchPass(pContext, RenderTargets, PassId::DenoiseSpecHitT, RTXPTDenoisingGuideDebugView::Disabled, 1u))
        return false;
    InsertUAVBarrier(pContext, RenderTargets.GetScratchFloat1UAV());

    if (!DispatchPass(pContext, RenderTargets, PassId::DenoiseSpecHitT, RTXPTDenoisingGuideDebugView::Disabled, 0u))
        return false;
    InsertUAVBarrier(pContext, RenderTargets.GetSpecularHitTUAV());

    if (!DispatchPass(pContext, RenderTargets, PassId::ComputeAvgLayerRadiance, RTXPTDenoisingGuideDebugView::Disabled, 0u))
        return false;
    InsertUAVBarrier(pContext, RenderTargets.GetDenoiserAvgLayerRadianceHalfResUAV());

    m_Stats.LastDenoiseSpecHitTExecuted = true;
    m_Stats.DenoiseSpecHitTDispatchCount += 2u;
    m_Stats.LastAvgLayerRadianceExecuted = true;
    ++m_Stats.AvgLayerRadianceDispatchCount;

    if (DebugView != RTXPTDenoisingGuideDebugView::Disabled)
    {
        if (!DispatchPass(pContext, RenderTargets, PassId::DebugViz, DebugView, 0u))
            return false;
        InsertUAVBarrier(pContext, RenderTargets.GetProcessedOutputColorUAV());
        m_Stats.LastDebugVizExecuted = true;
        ++m_Stats.DebugVizDispatchCount;
    }

    m_Stats.LastBakeExecuted = true;
    return true;
}

bool RTXPTDenoisingGuidesBaker::DispatchPass(IDeviceContext*               pContext,
                                             const RTXPTRenderTargets&    RenderTargets,
                                             PassId                       Pass,
                                             RTXPTDenoisingGuideDebugView DebugView,
                                             Uint32                       Ping)
{
    PassState& State = m_Passes[static_cast<size_t>(Pass)];
    if (!State.PSO || !State.SRB || pContext == nullptr)
        return false;

    const Uint32 RenderWidth  = RenderTargets.GetRenderWidth();
    const Uint32 RenderHeight = RenderTargets.GetRenderHeight();
    if (RenderWidth == 0 || RenderHeight == 0)
        return false;

    DenoisingGuidesBakerConstants Constants;
    Constants.RenderResolution  = uint2{RenderWidth, RenderHeight};
    Constants.DisplayResolution = uint2{RenderTargets.GetDisplayWidth(), RenderTargets.GetDisplayHeight()};
    Constants.DebugView         = static_cast<Int32>(DebugView);
    Constants.Ping              = Ping;

    {
        MapHelper<DenoisingGuidesBakerConstants> Mapped{pContext, m_Constants, MAP_WRITE, MAP_FLAG_DISCARD};
        VERIFY(Mapped, "Failed to map RTXPT denoising guides baker constants");
        if (!Mapped)
            return false;
        *Mapped = Constants;
    }

    auto SetVariable = [&State](const char* Name, IDeviceObject* pObject, bool Required) {
        IShaderResourceVariable* pVar = State.SRB->GetVariableByName(SHADER_TYPE_COMPUTE, Name);
        if (pVar == nullptr)
            return !Required;
        if (pObject == nullptr)
            return false;
        pVar->Set(pObject);
        return true;
    };

    const bool NeedsDenoiseSpecHitT = Pass == PassId::DenoiseSpecHitT;
    const bool NeedsAvgLayer        = Pass == PassId::ComputeAvgLayerRadiance;
    const bool NeedsDebugViz        = Pass == PassId::DebugViz;

    const bool Bound =
        SetVariable("t_Depth", RenderTargets.GetDepthSRV(), NeedsDenoiseSpecHitT || NeedsDebugViz) &&
        SetVariable("t_MotionVectors", RenderTargets.GetScreenMotionVectorsSRV(), NeedsAvgLayer || NeedsDebugViz) &&
        SetVariable("u_SpecularHitT", RenderTargets.GetSpecularHitTUAV(), NeedsDenoiseSpecHitT || NeedsDebugViz) &&
        SetVariable("u_ScratchFloat1", RenderTargets.GetScratchFloat1UAV(), NeedsDenoiseSpecHitT) &&
        SetVariable("u_StableRadiance", RenderTargets.GetStableRadianceUAV(), NeedsAvgLayer) &&
        SetVariable("u_StablePlanesHeader", RenderTargets.GetStablePlanesHeaderUAV(), NeedsAvgLayer) &&
        SetVariable("u_StablePlanesBuffer", RenderTargets.GetStablePlanesBufferUAV(), NeedsAvgLayer) &&
        SetVariable("u_DenoiserAvgLayerRadianceHalfRes", RenderTargets.GetDenoiserAvgLayerRadianceHalfResUAV(), NeedsAvgLayer || NeedsDebugViz) &&
        SetVariable("u_DebugOutput", RenderTargets.GetProcessedOutputColorUAV(), NeedsDebugViz);
    if (!Bound)
    {
        DEV_ERROR("Failed to bind RTXPT denoising guides dynamic resources");
        return false;
    }

    pContext->SetPipelineState(State.PSO);
    pContext->CommitShaderResources(State.SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

    const bool HalfResolutionDispatch = Pass == PassId::ComputeAvgLayerRadiance;
    const Uint32 DispatchWidth  = HalfResolutionDispatch ? (RenderWidth + 1u) / 2u : RenderWidth;
    const Uint32 DispatchHeight = HalfResolutionDispatch ? (RenderHeight + 1u) / 2u : RenderHeight;

    DispatchComputeAttribs DispatchAttribs;
    DispatchAttribs.ThreadGroupCountX = (DispatchWidth + kThreadGroupSize - 1u) / kThreadGroupSize;
    DispatchAttribs.ThreadGroupCountY = (DispatchHeight + kThreadGroupSize - 1u) / kThreadGroupSize;
    DispatchAttribs.ThreadGroupCountZ = 1;
    pContext->DispatchCompute(DispatchAttribs);

    return true;
}

} // namespace Diligent
```

- [ ] **Step 4: Verify the class symbols**

Run:

```powershell
rg -n "RTXPTDenoisingGuidesBaker|DenoiseSpecHitT|ComputeAvgLayerRadiance|DebugViz|DenoisingGuidesBakerConstants|DispatchPass|GetProcessedOutputColorUAV" DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.cpp
```

Expected: the class, constants, entry names, and debug output target are found.

- [ ] **Step 5: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.hpp DiligentSamples/Samples/RTXPT/src/RTXPTDenoisingGuidesBaker.cpp
git commit -m "feat(rtxpt): add denoising guides baker pass" -m "Co-Authored-By: GPT 5.5"
```

### Task 4: Register the New Files

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Add the C++ source**

In the `set(SOURCE` list, add `src/RTXPTDenoisingGuidesBaker.cpp` after `src/RTXPTRayTracingPass.cpp`:

```cmake
    src/RTXPTRayTracingPass.cpp
    src/RTXPTDenoisingGuidesBaker.cpp
    src/RTXPTEmissiveTrianglePass.cpp
```

- [ ] **Step 2: Add the C++ header**

In the `set(INCLUDE` list, add `src/RTXPTDenoisingGuidesBaker.hpp` after `src/RTXPTRayTracingPass.hpp`:

```cmake
    src/RTXPTRayTracingPass.hpp
    src/RTXPTDenoisingGuidesBaker.hpp
    src/RTXPTEmissiveTrianglePass.hpp
```

- [ ] **Step 3: Add the shader**

In the `set(SHADERS` list, add `assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl` after `assets/shaders/PathTracer/PathTracerStablePlanes.hlsli`:

```cmake
    assets/shaders/PathTracer/StablePlanes.hlsli
    assets/shaders/PathTracer/PathTracerStablePlanes.hlsli
    assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl
    assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli
```

- [ ] **Step 4: Verify CMake registration**

Run:

```powershell
rg -n "RTXPTDenoisingGuidesBaker|DenoisingGuidesBaker.hlsl" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected:

```text
DiligentSamples/Samples/RTXPT/CMakeLists.txt contains the new cpp, hpp, and hlsl paths.
```

- [ ] **Step 5: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "chore(rtxpt): register denoising guides baker files" -m "Co-Authored-By: GPT 5.5"
```

### Task 5: Wire the Baker into RTXPTSample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Include the baker header**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, add the include near the other RTXPT pass includes:

```cpp
#include "RTXPTDenoisingGuidesBaker.hpp"
```

- [ ] **Step 2: Add helper declarations**

In the private method declarations, add these after `bool PathTrace();`:

```cpp
    bool BakeDenoisingGuides();
    bool PresentRealtimeGuideDebug();
```

- [ ] **Step 3: Add the member**

In the private members, add this after `RTXPTRayTracingPass m_RayTracingPass;`:

```cpp
    RTXPTDenoisingGuidesBaker     m_DenoisingGuidesBaker;
```

- [ ] **Step 4: Reset the baker with scene-dependent passes**

In `RTXPTSample::ResetSceneDependentResources()`, add this after `m_RayTracingPass.Reset();`:

```cpp
    m_DenoisingGuidesBaker.Reset();
```

- [ ] **Step 5: Initialize the baker**

In `RTXPTSample::CreatePhase4Passes()`, add this immediately after `m_PostProcessPipeline.Initialize(...)`:

```cpp
    m_DenoisingGuidesBaker.Initialize(m_pDevice,
                                      m_pEngineFactory,
                                      m_FrameConstantsCB,
                                      m_FeatureCaps.ComputeShaders);
```

- [ ] **Step 6: Add the bake helper**

Add this method after `RTXPTSample::DispatchPathTraceLoop`:

```cpp
bool RTXPTSample::BakeDenoisingGuides()
{
    if (!m_RenderTargets.HasRealtimeRenderTargets())
    {
        RecordRealtimePathTraceStatus("Realtime render targets are not allocated for denoising guides");
        return false;
    }

    if (!m_DenoisingGuidesBaker.IsReady())
    {
        RecordRealtimePathTraceStatus("DenoisingGuidesBaker is not ready");
        return false;
    }

    const bool GuidesOk =
        m_DenoisingGuidesBaker.Bake(m_pImmediateContext,
                                    m_RenderTargets,
                                    m_RealtimeUI.DenoisingGuideDebugView);
    if (!GuidesOk)
    {
        RecordRealtimePathTraceStatus("DenoisingGuidesBaker dispatch failed");
        return false;
    }

    return true;
}
```

- [ ] **Step 7: Add the optional debug presentation helper**

Add this method after `BakeDenoisingGuides()`:

```cpp
bool RTXPTSample::PresentRealtimeGuideDebug()
{
    if (m_RealtimeUI.DenoisingGuideDebugView == RTXPTDenoisingGuideDebugView::Disabled ||
        !m_DenoisingGuidesBaker.GetStats().LastDebugVizExecuted)
    {
        return false;
    }

    const bool ToneMappingExecuted =
        m_PostProcessPipeline.RunToneMapping(m_pImmediateContext,
                                             m_RenderTargets,
                                             m_ReferenceUI.ToneMapping,
                                             m_ReferenceUI.EnableToneMapping);
    if (!ToneMappingExecuted)
    {
        ClearFallback(float4{0.9f, 0.2f, 0.6f, 1.0f});
        return true;
    }

    ITextureView* pPresentationSRV = m_RenderTargets.GetPresentationSRV();
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pPresentationSRV))
    {
        ClearFallback(float4{0.0f, 1.0f, 1.0f, 1.0f});
        return true;
    }

    return true;
}
```

- [ ] **Step 8: Call the baker from PathTrace**

In `RTXPTSample::PathTrace()`, replace the realtime hook block:

```cpp
    if (UseStablePlanes)
    {
        // RTXDI/ReSTIR final shading, denoising-guide bake, and final merge are future port hooks.
        m_LastRealtimePathTraceExecuted = true;
        RecordRealtimePathTraceStatus("Realtime PathTrace dispatched; RTXDI/ReSTIR and DenoisingGuides hooks disabled");
    }
```

with:

```cpp
    if (UseStablePlanes)
    {
        if (!BakeDenoisingGuides())
            return false;

        // RTXDI/ReSTIR final shading and final merge are future port hooks.
        m_LastRealtimePathTraceExecuted = true;
        RecordRealtimePathTraceStatus("Realtime PathTrace and denoising guides dispatched; RTXDI/ReSTIR and final merge disabled");
    }
```

- [ ] **Step 9: Use debug presentation before the fallback clear**

In `RTXPTSample::RunRealtimePathTraceOnly()`, replace:

```cpp
    // FILL_STABLE_PLANES writes stable-plane radiance storage, not final OutputColor.
    // G7/G9 will replace this fallback with NoDenoiserFinalMerge or NRD final merge.
    ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
    return true;
```

with:

```cpp
    if (PresentRealtimeGuideDebug())
        return true;

    // FILL_STABLE_PLANES writes stable-plane radiance storage, not final OutputColor.
    // G7/G9 will replace this fallback with NoDenoiserFinalMerge or NRD final merge.
    ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
    return true;
```

- [ ] **Step 10: Add status UI**

In `RTXPTSample::UpdateUI()`, inside the debug/status block after the existing realtime render-target status, add:

```cpp
            const RTXPTDenoisingGuidesBakerStats& GuideStats = m_DenoisingGuidesBaker.GetStats();
            ImGui::Text("DenoisingGuidesBaker: %s", GuideStats.Ready ? "ready" : "not ready");
            ImGui::Text("DenoiseSpecHitT: %s (%u)",
                        GuideStats.LastDenoiseSpecHitTExecuted ? "dispatched" : "not dispatched",
                        GuideStats.DenoiseSpecHitTDispatchCount);
            ImGui::Text("Avg layer radiance: %s (%u)",
                        GuideStats.LastAvgLayerRadianceExecuted ? "dispatched" : "not dispatched",
                        GuideStats.AvgLayerRadianceDispatchCount);
            ImGui::Text("Guide debug view: %s",
                        GetDenoisingGuideDebugViewName(m_RealtimeUI.DenoisingGuideDebugView));
```

- [ ] **Step 11: Verify sample wiring**

Run:

```powershell
rg -n "RTXPTDenoisingGuidesBaker|BakeDenoisingGuides|PresentRealtimeGuideDebug|DenoisingGuidesBaker dispatch failed|PathTrace and denoising guides|Guide debug view" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: all integration points are found.

- [ ] **Step 12: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): run denoising guides after realtime pathtrace" -m "Co-Authored-By: GPT 5.5"
```

### Task 6: Update Fork Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add a G6 mapping section**

Append this section after `## Realtime G4-G5 PathTrace Variants and Orchestration`:

```markdown
## Realtime G6 Denoising Guides Baker

| RTXPT-fork source | Diligent port | Notes |
|---|---|---|
| `ProcessingPasses/DenoisingGuidesBaker.h` | `src/RTXPTDenoisingGuidesBaker.hpp` | Diligent pass owner for guide compute dispatch and stats. |
| `ProcessingPasses/DenoisingGuidesBaker.cpp::DenoiseSpecHitT` | `src/RTXPTDenoisingGuidesBaker.cpp::Bake` | Preserves ping then pong dispatch ordering using `SpecularHitT` and `ScratchFloat1`. |
| `ProcessingPasses/DenoisingGuidesBaker.cpp::ComputeAvgLayerRadiance` | `src/RTXPTDenoisingGuidesBaker.cpp::Bake` | Dispatches at half render resolution and writes `DenoiserAvgLayerRadianceHalfRes`. |
| `ProcessingPasses/DenoisingGuidesBaker.cpp::RenderDebugViz` | `src/RTXPTDenoisingGuidesBaker.cpp::Bake` + `src/RTXPTSample.cpp::PresentRealtimeGuideDebug` | Diligent debug visualization writes to `ProcessedOutputColor` for optional presentation because no RTXPT-fork `ShaderDebug` texture owner is ported. |
| `ProcessingPasses/DenoisingGuidesBaker.hlsl` | `assets/shaders/PathTracer/DenoisingGuidesBaker.hlsl` | Shader algorithms use Diligent resource names and existing `StablePlanesContext`. |
| `Sample.cpp::PathTrace` guide bake call point | `src/RTXPTSample.cpp::PathTrace` | Runs after FILL stable-plane loop and before later final merge/NRD work. |

G6 intentionally does not port `ShaderDebug`, `StablePlanesDebugViz`, NRD prepare/final merge, or no-denoiser final merge. Those remain owned by later realtime denoise phases.
```

- [ ] **Step 2: Verify mapping**

Run:

```powershell
rg -n "Realtime G6 Denoising Guides Baker|DenoisingGuidesBaker|DenoiseSpecHitT|ComputeAvgLayerRadiance|RenderDebugViz|PresentRealtimeGuideDebug" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: the new section and all mapping rows are found.

- [ ] **Step 3: Commit**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map denoising guides baker port" -m "Co-Authored-By: GPT 5.5"
```

### Task 7: Build and Smoke Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: build tree under `build/x64/Debug` when present

- [ ] **Step 1: Run source-level verification**

Run:

```powershell
rg -n "DenoisingGuidesBaker" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "DenoisingGuides hooks disabled|DenoisingGuidesBaker dispatch failed|DenoisingGuideDebugView" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp
```

Expected:

```text
The old "DenoisingGuides hooks disabled" status is absent.
The new baker symbols, shader, CMake entries, and mapping entries are present.
```

- [ ] **Step 2: Build RTXPT**

Run:

```powershell
if (Test-Path 'build\x64\Debug') {
    cmake --build build\x64\Debug --config Debug --target RTXPT
} else {
    cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DCMAKE_INSTALL_PREFIX=install\x64\Debug -DDILIGENT_BUILD_FX=TRUE -DDILIGENT_BUILD_SAMPLES=TRUE -DDILIGENT_BUILD_TOOLS=TRUE -DDILIGENT_NO_WEBGPU=TRUE -DDILIGENT_NO_ARCHIVER=FALSE -DDILIGENT_BUILD_TESTS=TRUE -DDILIGENT_DEVELOPMENT=TRUE -DDILIGENT_NO_FORMAT_VALIDATION=OFF -DDILIGENT_USE_SPIRV_TOOLCHAIN=TRUE
    cmake --build build\x64\Debug --config Debug --target RTXPT
}
```

Expected: build succeeds and compiles `RTXPTDenoisingGuidesBaker.cpp` plus `PathTracer/DenoisingGuidesBaker.hlsl`.

- [ ] **Step 3: Run a manual D3D12 smoke pass**

Run the built RTXPT sample from the existing Diligent sample launcher workflow, select a scene, then use the RTXPT ImGui panel:

```text
Mode = Realtime
Denoising guide debug = Specular Hit T
Samples per pixel = 1
Standalone denoiser = disabled
```

Expected:

```text
Realtime PathTrace status reads "Realtime PathTrace and denoising guides dispatched; RTXDI/ReSTIR and final merge disabled".
DenoiseSpecHitT dispatch count increments by 2 per realtime frame.
Avg layer radiance dispatch count increments by 1 per realtime frame.
The Specular Hit T guide debug view presents through the LDR output instead of the dark fallback clear.
```

- [ ] **Step 4: Smoke reference mode**

In the same run, switch back:

```text
Mode = Reference
```

Expected:

```text
Reference accumulation, pre-tone mapping, tone mapping, and blit still render as before.
DenoisingGuidesBaker dispatch counters stop changing while reference mode is active.
```

- [ ] **Step 5: Commit verification fixes if needed**

If any build or smoke issue required code changes, commit only those fixes:

```powershell
git add DiligentSamples/Samples/RTXPT
git commit -m "fix(rtxpt): stabilize denoising guides baker integration" -m "Co-Authored-By: GPT 5.5"
```

If no code changes were needed after verification, do not create an empty commit.

## Self-Review

**Spec coverage:** G6 requires `DenoiseSpecHitT`, `ComputeAvgLayerRadiance`, `RenderDebugViz`, correct pass order, full/half dispatch dimensions, and shared frame/stable-plane bindings. Tasks 2 and 3 implement the three compute entries, Task 5 wires the order after realtime FILL and before later final merge, Task 3 dispatches full or half resolution according to pass, and Task 3 binds `g_Const` plus stable-plane resources.

**Placeholder scan:** This plan avoids unresolved markers and unspecified edge handling. Later-phase work is explicitly out of scope and named by phase boundary rather than left as an implementation placeholder.

**Type consistency:** `RTXPTDenoisingGuideDebugView` is defined in `RTXPTRealtimeSettings.hpp`, consumed by `RTXPTDenoisingGuidesBaker` and `RTXPTSample`, and converted to the HLSL `DebugView` integer. `DenoisingGuidesBakerConstants` is 64 bytes in C++ and HLSL. Resource names in C++ match shader variables: `t_Depth`, `t_MotionVectors`, `u_SpecularHitT`, `u_ScratchFloat1`, `u_StableRadiance`, `u_StablePlanesHeader`, `u_StablePlanesBuffer`, `u_DenoiserAvgLayerRadianceHalfRes`, and `u_DebugOutput`.

## Execution Handoff

Plan complete. Recommended execution mode is `superpowers:subagent-driven-development` because tasks are separable by contract/settings, shader, pass owner, sample wiring, docs, and verification. Inline execution with `superpowers:executing-plans` is also valid if one session owns the whole integration.
