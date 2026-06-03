# RTXPT Post-Processing Phase P4 Bloom and LDR Post-Process Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port RTXPT-fork's HDR bloom and LDR post-tone edge/test effects so they run around the P2/P3 `OutputColor -> AccumulatedRadiance -> ProcessedOutputColor -> LdrColor` display chain without changing the base chain when disabled.

**Architecture:** Add a Diligent-native bloom graphics pass that mirrors Donut `BloomPass` behavior against `ProcessedOutputColor`, then add a small post-process compute pass for the RTXPT-fork `TestRaygenPP_HDR` and LDR edge-detection hooks. `RTXPTPostProcessPipeline` becomes the orchestration owner for `RunPreToneMapping` and `RunPostToneMapping`, while `RTXPTSample` owns UI state and keeps the render order identical to RTXPT-fork: accumulation, HDR post-process, tone mapping, LDR post-process, final blit.

**Tech Stack:** C++17, HLSL/DXC, Diligent Engine graphics/compute PSO and SRB APIs, Diligent texture SRV/UAV/RTV views, ImGui, CMake sample registration, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt` and `D:/RTXPT-fork/External/Donut`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P4.
- P0 mapping already names P4 owners in `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`.
- P1 resources exist in `RTXPTRenderTargets`: `ProcessedOutputColor` has SRV/UAV/RTV views, and `LdrColor` plus `LdrColorScratch` have SRV/UAV/RTV views.
- P2/P3 implementation exists in `RTXPTPostProcessPipeline`, but it currently only runs `RunAccumulation` and `RunToneMapping`.
- `RTXPTSample::Render()` currently runs `Trace -> RunAccumulation -> RunToneMapping -> optional debug compute -> final blit`.
- `RTXPTReferenceUIState` currently has `EnableToneMapping` and `ToneMapping`, but no bloom, HDR test, or LDR edge-detection settings.
- `CMakeLists.txt` currently registers `RTXPTAccumulationPass` and `RTXPTToneMappingPass`; no P4 pass files are registered.

## RTXPT-Fork Anchors

Read these before editing and keep the behavior contracts intact:

- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1292-1299` - render-pass creation order: accumulation, tone mapping, bloom, shared post-process.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1827-1862` - `PostProcessPreToneMapping`: bloom first, optional `TestRaygenPP_HDR` after bloom.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1864-1888` - `PostProcessPostToneMapping`: copy `LdrColor` to `LdrColorScratch`, run edge detection into `LdrColor`.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2194-2203` - frame-tail order: HDR post-process, tone mapping, LDR post-process.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:301-307` - P4 UI defaults: `PostProcessTestPassHDR=false`, `PostProcessEdgeDetection=false`, threshold `0.1`, bloom enabled with radius `8.0` and intensity `0.004`.
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1534-1542` - HDR post-process UI labels and ranges.
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1614-1619` - LDR post-process UI labels and threshold range.
- `D:/RTXPT-fork/Rtxpt/Shaders/TestRaygenPP.hlsl:17-68` - exact HDR test circle and LDR Sobel edge-detection formulas.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/ColorHelpers.hlsli:134-154` - `LinearToSRGB` helper used by LDR edge output.
- `D:/RTXPT-fork/External/Donut/include/donut/render/BloomPass.h:42-90` - bloom pass state shape and public render contract.
- `D:/RTXPT-fork/External/Donut/src/render/BloomPass.cpp:48-158` - bloom setup: constant buffers, two downscale textures, two blur textures, blur PSO.
- `D:/RTXPT-fork/External/Donut/src/render/BloomPass.cpp:161-278` - bloom render sequence: two downscales, horizontal/vertical blur, blended apply.
- `D:/RTXPT-fork/External/Donut/include/donut/shaders/bloom_cb.h:23-34` - `BloomConstants` layout.
- `D:/RTXPT-fork/External/Donut/shaders/passes/bloom_ps.hlsl:23-61` - Gaussian blur shader.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.{h,cpp,hlsl}` - compute post-process framework for later denoiser/merge work. P4 should not port the denoiser branches, but the Diligent `RTXPTPostProcessPass` naming should keep this mapping recognizable.

## Scope Boundaries

- P4 must not alter raygen output, accumulation math, tone-map operators, exposure, or final blit semantics from P2/P3.
- Bloom runs only when `EnableBloom && BloomIntensity > 0.0f && BloomRadius > 0.0f`; disabled bloom returns success without touching `ProcessedOutputColor`.
- HDR test pass runs after bloom and writes `ProcessedOutputColor`; disabled HDR test returns success without touching `ProcessedOutputColor`.
- LDR edge detection runs after tone mapping and before final blit; disabled LDR edge detection returns success without touching `LdrColor`.
- LDR edge detection must copy `LdrColor` into `LdrColorScratch` before sampling, matching RTXPT-fork ping-pong behavior.
- Bloom and LDR toggles are independent from `EnableToneMapping`; tone mapping disabled is still a P3 pass-through from `ProcessedOutputColor` to `LdrColor`.
- Do not import Donut/NVRHI APIs. Preserve behavior, names, formulas, constants, UI defaults, and scheduling order in Diligent-native code.
- Do not port NRD, DLSS, DLSS-RR, stable-plane merge, shader debug buffers, zoom, or TAA in this phase.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.hpp` - Diligent bloom settings, stats, render attribs, and pass interface.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.cpp` - Diligent bloom graphics PSOs, intermediate HDR textures, constants upload, and render sequence.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp` - P4 HDR test and LDR edge-detection settings/stats/interface.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp` - Diligent compute PSOs for `RTXPT_POST_PROCESS_HDR_TEST` and `RTXPT_POST_PROCESS_EDGE_DETECTION`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomShared.h` - bloom constants layout matching Donut `BloomConstants`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomCopy.psh` - linear-sampled fullscreen copy/downscale shader for bloom downscale and apply.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomBlur.psh` - Gaussian blur shader matching Donut `bloom_ps.hlsl`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh` - HDR test and LDR edge-detection compute shader.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}` - own P4 passes, validate resources, run pre-tone and post-tone stages.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}` - add P4 UI state, call P4 pipeline stages, expose diagnostics.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register P4 C++ and shader files.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - change P4 mapping rows from planned to implemented and name exact P4 files.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `D:/RTXPT-fork/Rtxpt`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. If any file is dirty, inspect it before editing and preserve user changes.

- [ ] **Step 2: Confirm P2/P3 baseline is present**

Run:

```powershell
rg -n "RTXPTAccumulationPass|RTXPTToneMappingPass|RunAccumulation|RunToneMapping|ProcessedOutputColor|LdrColorScratch" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: `RTXPTPostProcessPipeline` owns accumulation and tone mapping, `RTXPTRenderTargets` exposes `ProcessedOutputColor` and `LdrColorScratch`, and CMake registers the P2/P3 files.

- [ ] **Step 3: Confirm upstream P4 anchors exist**

Run:

```powershell
Test-Path D:\RTXPT-fork\External\Donut\src\render\BloomPass.cpp
Test-Path D:\RTXPT-fork\External\Donut\shaders\passes\bloom_ps.hlsl
Test-Path D:\RTXPT-fork\External\Donut\include\donut\shaders\bloom_cb.h
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\TestRaygenPP.hlsl
Test-Path D:\RTXPT-fork\Rtxpt\Sample.cpp
Test-Path D:\RTXPT-fork\Rtxpt\SampleUI.h
```

Expected: every command prints `True`.

- [ ] **Step 4: Confirm the intended insertion order**

Run:

```powershell
rg -n "RunAccumulation|RunToneMapping|GetLdrColorSRV|m_BlitPass.Render" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: `RunAccumulation` appears before `RunToneMapping`, and final display reads `LdrColor` unless the debug compute pass is active.

### Task 1: Add P4 UI and Settings Structures

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.h:301-307`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1534-1542`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1614-1619`

- [ ] **Step 1: Add P4 fields to `RTXPTReferenceUIState`**

Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` inside `struct RTXPTReferenceUIState` immediately after tone-mapping fields:

```cpp
    bool  PostProcessTestPassHDR          = false; // Phase 6/P4: RTXPT-fork HDR test hook.
    bool  PostProcessEdgeDetection        = false; // Phase 6/P4: RTXPT-fork LDR edge detection.
    float PostProcessEdgeDetectionThreshold = 0.1f;
    bool  EnableBloom                     = true;
    float BloomRadius                     = 8.0f;
    float BloomIntensity                  = 0.004f;
```

Expected: defaults match `D:/RTXPT-fork/Rtxpt/SampleUI.h:301-307`.

- [ ] **Step 2: Add ImGui controls under Post processing**

Modify `RTXPTSample::UpdateUI()` in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` under the existing `Post processing:` section. Insert the controls before `Enable tone mapping` so the order mirrors RTXPT-fork's early HDR controls, then tone mapping, then late LDR controls:

```cpp
            ImGui::Checkbox("PostProcessTestPass", &m_ReferenceUI.PostProcessTestPassHDR);

            if (ImGui::CollapsingHeader("Bloom"))
            {
                ImGui::Checkbox("Enable Bloom", &m_ReferenceUI.EnableBloom);
                ImGui::SliderFloat("Bloom Width (Pixels)", &m_ReferenceUI.BloomRadius, 0.0f, 64.0f);
                ImGui::SliderFloat("Bloom Intensity", &m_ReferenceUI.BloomIntensity, 0.0f, 0.1f);
            }

            ImGui::Checkbox("Enable tone mapping", &m_ReferenceUI.EnableToneMapping);
```

Then insert the LDR controls immediately after `SanitizeToneMappingParameters(ToneMapping);` and before leaving the `Post processing:` block:

```cpp
            if (ImGui::CollapsingHeader("Late (LDR) post-process"))
            {
                ImGui::Checkbox("EdgeDetection", &m_ReferenceUI.PostProcessEdgeDetection);
                ImGui::SliderFloat("EdgeDetectionThreshold", &m_ReferenceUI.PostProcessEdgeDetectionThreshold, 0.0f, 1.0f);
            }
```

Expected: labels and ranges match RTXPT-fork.

- [ ] **Step 3: Clamp P4 scalar settings before render**

Modify `RTXPTSample::Render()` before the first P4 call:

```cpp
    const float BloomRadius    = std::clamp(m_ReferenceUI.BloomRadius, 0.0f, 64.0f);
    const float BloomIntensity = std::clamp(m_ReferenceUI.BloomIntensity, 0.0f, 0.1f);
    const float EdgeThreshold  = std::clamp(m_ReferenceUI.PostProcessEdgeDetectionThreshold, 0.0f, 1.0f);
```

Expected: UI and render path use the same bounds as RTXPT-fork controls.

- [ ] **Step 4: Run compile-time grep**

Run:

```powershell
rg -n "PostProcessTestPassHDR|PostProcessEdgeDetection|PostProcessEdgeDetectionThreshold|EnableBloom|BloomRadius|BloomIntensity" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: fields and UI controls are present.

- [ ] **Step 5: Commit P4 UI settings**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add P4 post-process UI controls" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only UI state and control changes.

### Task 2: Add Bloom Shader Files

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomShared.h`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomCopy.psh`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomBlur.psh`
- Read: `D:/RTXPT-fork/External/Donut/include/donut/shaders/bloom_cb.h`
- Read: `D:/RTXPT-fork/External/Donut/shaders/passes/bloom_ps.hlsl`

- [ ] **Step 1: Create `RTXPTBloomShared.h`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomShared.h`:

```hlsl
#ifndef __RTXPT_BLOOM_SHARED_H__
#define __RTXPT_BLOOM_SHARED_H__

struct RTXPTBloomConstants
{
    float2 PixStep;
    float  ArgumentScale;
    float  NormalizationScale;

    float3 Padding;
    float  NumSamples;
};

#endif // __RTXPT_BLOOM_SHARED_H__
```

Expected: field order and packing match Donut `BloomConstants`.

- [ ] **Step 2: Create `RTXPTBloomCopy.psh`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomCopy.psh`:

```hlsl
SamplerState      s_LinearSampler;
Texture2D<float4> t_Source;

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

struct PSOutput
{
    float4 Color : SV_TARGET;
};

void main(in PSInput Input,
          out PSOutput Output)
{
    Output.Color = t_Source.SampleLevel(s_LinearSampler, Input.UV, 0.0);
}
```

Expected: this replaces Donut `CommonRenderPasses::BlitTexture` for the two linear-filtered downscales and the final blended apply.

- [ ] **Step 3: Create `RTXPTBloomBlur.psh`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomBlur.psh`:

```hlsl
#pragma pack_matrix(row_major)

#include "RTXPTBloomShared.h"

cbuffer g_BloomConstants
{
    RTXPTBloomConstants g_Bloom;
};

SamplerState      s_LinearSampler;
Texture2D<float4> t_Source;

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

struct PSOutput
{
    float4 Color : SV_TARGET;
};

float Square(float X)
{
    return X * X;
}

void main(in PSInput Input,
          out PSOutput Output)
{
    float3 Result = t_Source.Load(int3(Input.Pos.xy, 0)).rgb;

    for (float X = 1.0; X < g_Bloom.NumSamples; X += 2.0)
    {
        const float W1  = exp(Square(X) * g_Bloom.ArgumentScale);
        const float W2  = exp(Square(X + 1.0) * g_Bloom.ArgumentScale);
        const float W12 = W1 + W2;
        const float P   = W2 / W12;
        const float2 Offset = g_Bloom.PixStep * (X + P);

        Result += t_Source.SampleLevel(s_LinearSampler, Input.UV + Offset, 0.0).rgb * W12;
        Result += t_Source.SampleLevel(s_LinearSampler, Input.UV - Offset, 0.0).rgb * W12;
    }

    Result *= g_Bloom.NormalizationScale;
    Output.Color = float4(Result, 1.0);
}
```

Expected: formula matches Donut `bloom_ps.hlsl` with Diligent naming.

- [ ] **Step 4: Run shader text parity grep**

Run:

```powershell
rg -n "RTXPTBloomConstants|PixStep|ArgumentScale|NormalizationScale|NumSamples|SampleLevel|Output.Color" DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloom*.*
```

Expected: shared constants and blur formula are present.

- [ ] **Step 5: Commit bloom shaders**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomShared.h Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomCopy.psh Samples/RTXPT/assets/shaders/PostProcessing/RTXPTBloomBlur.psh
git -C DiligentSamples commit -m "feat(rtxpt): add bloom shader passes" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only bloom shader files.

### Task 3: Implement `RTXPTBloomPass`

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.cpp`
- Read: `D:/RTXPT-fork/External/Donut/src/render/BloomPass.cpp:48-278`
- Read: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp`

- [ ] **Step 1: Create the bloom pass header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.hpp`:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include "BasicMath.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "GraphicsTypes.h"

namespace Diligent
{

struct RTXPTBloomParameters
{
    bool  Enabled   = true;
    float Radius    = 8.0f;
    float Intensity = 0.004f;
};

struct RTXPTBloomRenderAttribs
{
    ITextureView*          pSourceSRV = nullptr;
    ITextureView*          pTargetRTV = nullptr;
    Uint32                 Width      = 0;
    Uint32                 Height     = 0;
    TEXTURE_FORMAT         Format     = TEX_FORMAT_UNKNOWN;
    RTXPTBloomParameters   Params;
};

struct RTXPTBloomPassStats
{
    bool   Ready              = false;
    bool   LastRenderExecuted = false;
    Uint32 RenderCount        = 0;
    Uint32 DownscaleWidth     = 0;
    Uint32 DownscaleHeight    = 0;
    Uint32 BlurWidth          = 0;
    Uint32 BlurHeight         = 0;
};

class RTXPTBloomPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory);
    bool ResizeResources(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT Format);
    bool Render(IDeviceContext* pContext, const RTXPTBloomRenderAttribs& Attribs);

    bool                       IsReady() const { return m_Stats.Ready; }
    const RTXPTBloomPassStats& GetStats() const { return m_Stats; }

private:
    bool CreateSampler(IRenderDevice* pDevice);
    bool CreateShaders(IRenderDevice* pDevice, IEngineFactory* pEngineFactory);
    bool CreatePipelines(IRenderDevice* pDevice, TEXTURE_FORMAT Format);
    bool CreateIntermediateTexture(IRenderDevice* pDevice, const char* Name, Uint32 Width, Uint32 Height, TEXTURE_FORMAT Format, RefCntAutoPtr<ITexture>& Texture);
    bool DrawFullscreen(IDeviceContext* pContext, IPipelineState* pPSO, IShaderResourceBinding* pSRB, ITextureView* pRTV, Uint32 Width, Uint32 Height);
    bool DrawCopy(IDeviceContext* pContext, ITextureView* pSourceSRV, ITextureView* pTargetRTV, Uint32 Width, Uint32 Height, bool BlendEnabled, float BlendFactor);
    bool DrawBlur(IDeviceContext* pContext, IShaderResourceBinding* pSRB, ITextureView* pSourceSRV, ITextureView* pTargetRTV, Uint32 Width, Uint32 Height);
    bool UpdateBlurConstants(IDeviceContext* pContext, IBuffer* pBuffer, const float2& PixStep, float EffectiveSigma);

private:
    RTXPTBloomPassStats       m_Stats;
    RefCntAutoPtr<IShader>    m_FullscreenVS;
    RefCntAutoPtr<IShader>    m_CopyPS;
    RefCntAutoPtr<IShader>    m_BlurPS;
    RefCntAutoPtr<IPipelineState> m_CopyPSO;
    RefCntAutoPtr<IPipelineState> m_ApplyPSO;
    RefCntAutoPtr<IPipelineState> m_BlurPSO;
    RefCntAutoPtr<IShaderResourceBinding> m_CopySRB;
    RefCntAutoPtr<IShaderResourceBinding> m_ApplySRB;
    RefCntAutoPtr<IShaderResourceBinding> m_HBlurSRB;
    RefCntAutoPtr<IShaderResourceBinding> m_VBlurSRB;
    RefCntAutoPtr<IBuffer>    m_HBlurCB;
    RefCntAutoPtr<IBuffer>    m_VBlurCB;
    RefCntAutoPtr<ISampler>   m_LinearSampler;
    RefCntAutoPtr<ITexture>   m_Downscale1;
    RefCntAutoPtr<ITexture>   m_Downscale2;
    RefCntAutoPtr<ITexture>   m_Blur1;
    RefCntAutoPtr<ITexture>   m_Blur2;
    TEXTURE_FORMAT            m_Format = TEX_FORMAT_UNKNOWN;
    Uint32                    m_Width  = 0;
    Uint32                    m_Height = 0;
};

} // namespace Diligent
```

Expected: the public interface is small, and temporary textures remain owned by the bloom pass.

- [ ] **Step 2: Implement reset and sampler creation**

In `RTXPTBloomPass.cpp`, follow existing RTXPT file headers and local helper style. Implement `Reset()` and `CreateSampler()`:

```cpp
void RTXPTBloomPass::Reset()
{
    m_CopyPSO.Release();
    m_ApplyPSO.Release();
    m_BlurPSO.Release();
    m_CopySRB.Release();
    m_ApplySRB.Release();
    m_HBlurSRB.Release();
    m_VBlurSRB.Release();
    m_HBlurCB.Release();
    m_VBlurCB.Release();
    m_FullscreenVS.Release();
    m_CopyPS.Release();
    m_BlurPS.Release();
    m_LinearSampler.Release();
    m_Downscale1.Release();
    m_Downscale2.Release();
    m_Blur1.Release();
    m_Blur2.Release();
    m_Format = TEX_FORMAT_UNKNOWN;
    m_Width  = 0;
    m_Height = 0;
    m_Stats  = {};
}

bool RTXPTBloomPass::CreateSampler(IRenderDevice* pDevice)
{
    SamplerDesc SamplerCI;
    SamplerCI.Name      = "RTXPT bloom linear clamp sampler";
    SamplerCI.MinFilter = FILTER_TYPE_LINEAR;
    SamplerCI.MagFilter = FILTER_TYPE_LINEAR;
    SamplerCI.MipFilter = FILTER_TYPE_LINEAR;
    SamplerCI.AddressU  = TEXTURE_ADDRESS_CLAMP;
    SamplerCI.AddressV  = TEXTURE_ADDRESS_CLAMP;
    SamplerCI.AddressW  = TEXTURE_ADDRESS_CLAMP;
    pDevice->CreateSampler(SamplerCI, &m_LinearSampler);
    return m_LinearSampler != nullptr;
}
```

Expected: sampler behavior matches Donut's `m_LinearClampSampler`.

- [ ] **Step 3: Implement shader initialization and lazy pipeline creation**

In `RTXPTBloomPass.cpp`, let `Initialize()` create shaders from `shaders;shaders\PostProcessing`, a linear sampler, and two dynamic uniform buffers named `RTXPT bloom horizontal constants` and `RTXPT bloom vertical constants`. Keep `m_FullscreenVS`, `m_CopyPS`, and `m_BlurPS` alive so `ResizeResources()` can call `CreatePipelines()` when the HDR format is known.

Create three PSOs in `CreatePipelines()`:

```cpp
GraphicsPipelineStateCreateInfo CopyPSOCreateInfo;
CopyPSOCreateInfo.PSODesc.Name                                  = "RTXPT bloom copy PSO";
CopyPSOCreateInfo.PSODesc.PipelineType                          = PIPELINE_TYPE_GRAPHICS;
CopyPSOCreateInfo.GraphicsPipeline.NumRenderTargets             = 1;
CopyPSOCreateInfo.GraphicsPipeline.RTVFormats[0]                = Format;
CopyPSOCreateInfo.GraphicsPipeline.PrimitiveTopology            = PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP;
CopyPSOCreateInfo.GraphicsPipeline.RasterizerDesc.CullMode      = CULL_MODE_NONE;
CopyPSOCreateInfo.GraphicsPipeline.DepthStencilDesc.DepthEnable = False;
CopyPSOCreateInfo.pVS                                           = pFullscreenVS;
CopyPSOCreateInfo.pPS                                           = pCopyPS;
CopyPSOCreateInfo.PSODesc.ResourceLayout.DefaultVariableType    = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;

PipelineResourceLayoutDescX CopyLayout;
CopyLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
CopyLayout
    .AddVariable(SHADER_TYPE_PIXEL, "s_LinearSampler", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_PIXEL, "t_Source", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
CopyPSOCreateInfo.PSODesc.ResourceLayout = CopyLayout;
```

For `m_ApplyPSO`, copy `CopyPSOCreateInfo`, set name to `RTXPT bloom apply PSO`, and set:

```cpp
auto& BlendRT = ApplyPSOCreateInfo.GraphicsPipeline.BlendDesc.RenderTargets[0];
BlendRT.BlendEnable    = True;
BlendRT.SrcBlend       = BLEND_FACTOR_BLEND_FACTOR;
BlendRT.DestBlend      = BLEND_FACTOR_INV_BLEND_FACTOR;
BlendRT.SrcBlendAlpha  = BLEND_FACTOR_ZERO;
BlendRT.DestBlendAlpha = BLEND_FACTOR_ONE;
```

For `m_BlurPSO`, use `RTXPTBloomBlur.psh` and resource layout variables:

```cpp
BlurLayout
    .AddVariable(SHADER_TYPE_PIXEL, "g_BloomConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_PIXEL, "s_LinearSampler", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_PIXEL, "t_Source", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

Bind static resources after SRB creation:

```cpp
m_CopyPSO->GetStaticVariableByName(SHADER_TYPE_PIXEL, "s_LinearSampler")->Set(m_LinearSampler);
m_ApplyPSO->GetStaticVariableByName(SHADER_TYPE_PIXEL, "s_LinearSampler")->Set(m_LinearSampler);
m_BlurPSO->GetStaticVariableByName(SHADER_TYPE_PIXEL, "s_LinearSampler")->Set(m_LinearSampler);
m_HBlurSRB->GetVariableByName(SHADER_TYPE_PIXEL, "t_Source");
m_VBlurSRB->GetVariableByName(SHADER_TYPE_PIXEL, "t_Source");
```

Bind `m_HBlurCB` and `m_VBlurCB` to separate blur SRBs through `g_BloomConstants`, matching Donut's separate horizontal and vertical constant buffers. `ResizeResources()` must recreate the three PSOs and SRBs when `Format != m_Format`.

Expected: no Donut classes are referenced, the PSO render-target format is known before PSO creation, the two blur draws cannot overwrite each other's constants, and the blend factors match `BloomPass.cpp:266-271`.

- [ ] **Step 4: Implement intermediate texture allocation**

Implement `ResizeResources()` with Donut's two downscale levels:

```cpp
const Uint32 Downscale1Width  = std::max(1u, (Width + 1u) / 2u);
const Uint32 Downscale1Height = std::max(1u, (Height + 1u) / 2u);
const Uint32 Downscale2Width  = std::max(1u, (Downscale1Width + 1u) / 2u);
const Uint32 Downscale2Height = std::max(1u, (Downscale1Height + 1u) / 2u);
```

Each temporary texture must use:

```cpp
Desc.Type      = RESOURCE_DIM_TEX_2D;
Desc.Width     = Width;
Desc.Height    = Height;
Desc.Format    = Format;
Desc.BindFlags = BIND_SHADER_RESOURCE | BIND_RENDER_TARGET;
Desc.Usage     = USAGE_DEFAULT;
```

Expected: `Downscale1` is half-res, `Downscale2`, `Blur1`, and `Blur2` are quarter-res.

- [ ] **Step 5: Implement render sequence**

Implement `Render()` with these exact guard and order rules:

```cpp
m_Stats.LastRenderExecuted = false;

const bool Enabled =
    Attribs.Params.Enabled &&
    Attribs.Params.Intensity > 0.0f &&
    Attribs.Params.Radius > 0.0f;
if (!Enabled)
    return true;

const float EffectiveSigma = std::clamp(Attribs.Params.Radius * 0.25f, 1.0f, 100.0f);

DrawCopy(pContext, Attribs.pSourceSRV, m_Downscale1->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET),
         m_Stats.DownscaleWidth, m_Stats.DownscaleHeight, false, 0.0f);
DrawCopy(pContext, m_Downscale1->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE), m_Downscale2->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET),
         m_Stats.BlurWidth, m_Stats.BlurHeight, false, 0.0f);
UpdateBlurConstants(pContext, m_HBlurCB, float2{1.0f / static_cast<float>(m_Stats.BlurWidth), 0.0f}, EffectiveSigma);
UpdateBlurConstants(pContext, m_VBlurCB, float2{0.0f, 1.0f / static_cast<float>(m_Stats.BlurHeight)}, EffectiveSigma);
DrawBlur(pContext, m_HBlurSRB, m_Downscale2->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE), m_Blur1->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET),
         m_Stats.BlurWidth, m_Stats.BlurHeight);
DrawBlur(pContext, m_VBlurSRB, m_Blur1->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE), m_Blur2->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET),
         m_Stats.BlurWidth, m_Stats.BlurHeight);
DrawCopy(pContext, m_Blur2->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE), Attribs.pTargetRTV,
         Attribs.Width, Attribs.Height, true, Attribs.Params.Intensity);

m_Stats.LastRenderExecuted = true;
++m_Stats.RenderCount;
```

`UpdateBlurConstants()` must use Donut's formulas:

```cpp
Constants.PixStep            = PixStep;
Constants.ArgumentScale      = -1.0f / (2.0f * EffectiveSigma * EffectiveSigma);
Constants.NormalizationScale = 1.0f / (std::sqrt(2.0f * PI_F) * EffectiveSigma);
Constants.NumSamples         = std::round(EffectiveSigma * 4.0f);
```

Expected: the bloom output is blended into `ProcessedOutputColor` with `Intensity` as blend constant, not added in shader.

- [ ] **Step 6: Run targeted compile checks**

Run:

```powershell
rg -n "RTXPTBloomPass|EffectiveSigma|ArgumentScale|NormalizationScale|BLEND_FACTOR_BLEND_FACTOR|SetBlendFactors|Downscale1|Blur2" DiligentSamples/Samples/RTXPT/src/RTXPTBloomPass.*
```

Expected: bloom pass owns all calculations and uses Diligent blend constants.

- [ ] **Step 7: Commit bloom pass**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTBloomPass.hpp Samples/RTXPT/src/RTXPTBloomPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): port bloom graphics pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only bloom pass C++ files.

### Task 4: Add the P4 Post-Process Compute Pass

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/TestRaygenPP.hlsl`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/ColorHelpers.hlsli:134-154`

- [ ] **Step 1: Create the P4 compute shader**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`:

```hlsl
#pragma pack_matrix(row_major)

#ifndef RTXPT_POST_PROCESS_MODE
#define RTXPT_POST_PROCESS_MODE 0
#endif

#define RTXPT_POST_PROCESS_HDR_TEST       1
#define RTXPT_POST_PROCESS_EDGE_DETECTION 2

struct RTXPTPostProcessConstants
{
    uint  Width;
    uint  Height;
    float EdgeDetectionThreshold;
    float Padding0;
};

cbuffer g_PostProcessConstants
{
    RTXPTPostProcessConstants g_Params;
};

Texture2D<float4>   t_LdrColorScratch;
RWTexture2D<float4> u_ProcessedOutputColor;
RWTexture2D<float4> u_PostTonemapOutputColor;

float LinearToSRGB(float Lin)
{
    if (Lin <= 0.0031308f)
        return Lin * 12.92f;
    return pow(Lin, 1.0f / 2.4f) * 1.055f - 0.055f;
}

float3 LinearToSRGB(float3 Lin)
{
    return float3(LinearToSRGB(Lin.x), LinearToSRGB(Lin.y), LinearToSRGB(Lin.z));
}

float3 LoadLDR(uint2 PixelPos)
{
    return t_LdrColorScratch[PixelPos].rgb;
}

void SaveLDR(uint2 PixelPos, float3 LinearColor)
{
    u_PostTonemapOutputColor[PixelPos] = float4(LinearToSRGB(LinearColor), 1.0);
}

[numthreads(8, 8, 1)]
void main(uint3 DispatchThreadID : SV_DispatchThreadID)
{
    const uint2 PixelPos = DispatchThreadID.xy;
    if (PixelPos.x >= g_Params.Width || PixelPos.y >= g_Params.Height)
        return;

#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_HDR_TEST
    float3 ExistingColor = u_ProcessedOutputColor[PixelPos].rgb;
    if (length(float2(PixelPos.xy) - float2(800.0, 500.0)) < 100.0)
        ExistingColor.z += 10.0;
    u_ProcessedOutputColor[PixelPos] = float4(ExistingColor, 1.0);
#elif RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_EDGE_DETECTION
    const int OffX = 1;
    const int OffY = 1;

    const float3 S00 = LoadLDR(PixelPos + int2(-OffX, -OffY));
    const float3 S01 = LoadLDR(PixelPos + int2(0, -OffY));
    const float3 S02 = LoadLDR(PixelPos + int2(OffX, -OffY));
    const float3 S10 = LoadLDR(PixelPos + int2(-OffX, 0));
    const float3 S12 = LoadLDR(PixelPos + int2(OffX, 0));
    const float3 S20 = LoadLDR(PixelPos + int2(-OffX, OffY));
    const float3 S21 = LoadLDR(PixelPos + int2(0, OffY));
    const float3 S22 = LoadLDR(PixelPos + int2(OffX, OffY));

    const float3 SobelX = S00 + 2.0 * S10 + S20 - S02 - 2.0 * S12 - S22;
    const float3 SobelY = S00 + 2.0 * S01 + S02 - S20 - 2.0 * S21 - S22;
    const float3 EdgeSqr = SobelX * SobelX + SobelY * SobelY;
    const float  Threshold = g_Params.EdgeDetectionThreshold;
    const float3 EdgeColor = 1.0.xxx - (EdgeSqr > Threshold.xxx * Threshold.xxx);
    SaveLDR(PixelPos, saturate(EdgeColor));
#endif
}
```

Expected: HDR circle center/radius and LDR Sobel math match `TestRaygenPP.hlsl`.

- [ ] **Step 2: Create the post-process pass header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp`:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "DeviceContext.h"
#include "EngineFactory.h"

namespace Diligent
{

struct RTXPTPostProcessParameters
{
    bool  EnableHdrTest       = false;
    bool  EnableEdgeDetection = false;
    float EdgeThreshold       = 0.1f;
};

struct RTXPTPostProcessRenderAttribs
{
    ITexture*     pLdrColorTexture       = nullptr;
    ITexture*     pLdrColorScratchTexture = nullptr;
    ITextureView* pProcessedOutputUAV    = nullptr;
    ITextureView* pLdrColorScratchSRV    = nullptr;
    ITextureView* pLdrColorUAV           = nullptr;
    Uint32        Width                  = 0;
    Uint32        Height                 = 0;
    RTXPTPostProcessParameters Params;
};

struct RTXPTPostProcessPassStats
{
    bool   Ready                     = false;
    bool   LastHdrTestExecuted       = false;
    bool   LastEdgeDetectionExecuted = false;
    Uint32 HdrTestDispatchCount      = 0;
    Uint32 EdgeDetectionDispatchCount = 0;
};

class RTXPTPostProcessPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, bool ComputeSupported);
    bool RunHdrTest(IDeviceContext* pContext, const RTXPTPostProcessRenderAttribs& Attribs);
    bool RunEdgeDetection(IDeviceContext* pContext, const RTXPTPostProcessRenderAttribs& Attribs);

    bool                            IsReady() const { return m_Stats.Ready; }
    const RTXPTPostProcessPassStats& GetStats() const { return m_Stats; }

private:
    bool CreatePipeline(IRenderDevice* pDevice, const ShaderCreateInfo& BaseShaderCI, const char* Name, const char* MacroName, IPipelineState** ppPSO, IShaderResourceBinding** ppSRB);
    bool UpdateConstants(IDeviceContext* pContext, Uint32 Width, Uint32 Height, float EdgeThreshold);
    bool Dispatch(IDeviceContext* pContext, IPipelineState* pPSO, IShaderResourceBinding* pSRB, Uint32 Width, Uint32 Height);

private:
    RTXPTPostProcessPassStats m_Stats;
    RefCntAutoPtr<IPipelineState> m_HdrTestPSO;
    RefCntAutoPtr<IPipelineState> m_EdgeDetectionPSO;
    RefCntAutoPtr<IShaderResourceBinding> m_HdrTestSRB;
    RefCntAutoPtr<IShaderResourceBinding> m_EdgeDetectionSRB;
    RefCntAutoPtr<IBuffer> m_PostProcessCB;
};

} // namespace Diligent
```

Expected: HDR and LDR effects share constants but use separate PSOs with compile-time macros.

- [ ] **Step 3: Implement compute PSO creation**

In `RTXPTPostProcessPass.cpp`, include `ShaderMacroHelper.hpp`, create two compute shaders from `PostProcessing/RTXPTPostProcess.csh`:

```cpp
ShaderMacroHelper HdrMacros;
HdrMacros.Add("RTXPT_POST_PROCESS_MODE", "RTXPT_POST_PROCESS_HDR_TEST");
ShaderCI.Macros = HdrMacros;
CreatePipeline(pDevice, ShaderCI, "RTXPT HDR post-process test", "RTXPT_POST_PROCESS_HDR_TEST", &m_HdrTestPSO, &m_HdrTestSRB);

ShaderMacroHelper EdgeMacros;
EdgeMacros.Add("RTXPT_POST_PROCESS_MODE", "RTXPT_POST_PROCESS_EDGE_DETECTION");
ShaderCI.Macros = EdgeMacros;
CreatePipeline(pDevice, ShaderCI, "RTXPT LDR edge detection", "RTXPT_POST_PROCESS_EDGE_DETECTION", &m_EdgeDetectionPSO, &m_EdgeDetectionSRB);
```

Use this resource layout for both PSOs:

```cpp
PipelineResourceLayoutDescX ResourceLayout;
ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
ResourceLayout
    .AddVariable(SHADER_TYPE_COMPUTE, "g_PostProcessConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_LdrColorScratch", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_ProcessedOutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_PostTonemapOutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

Expected: compute support gates initialization, matching P2 accumulation pass behavior.

Add a local helper in `RTXPTPostProcessPass.cpp` because `SetSRBVariable` in `RTXPTToneMappingPass.cpp` is file-local:

```cpp
namespace
{

bool SetSRBVariable(IShaderResourceBinding* pSRB, SHADER_TYPE ShaderType, const char* Name, IDeviceObject* pObject, bool AllowMissing)
{
    if (pSRB == nullptr)
        return false;

    IShaderResourceVariable* pVariable = pSRB->GetVariableByName(ShaderType, Name);
    if (pVariable == nullptr)
        return AllowMissing;

    pVariable->Set(pObject);
    return true;
}

} // namespace
```

- [ ] **Step 4: Implement HDR test dispatch**

Implement `RunHdrTest()`:

```cpp
m_Stats.LastHdrTestExecuted = false;
if (!Attribs.Params.EnableHdrTest)
    return true;
if (!IsReady() || pContext == nullptr || Attribs.pProcessedOutputUAV == nullptr || Attribs.Width == 0 || Attribs.Height == 0)
    return false;

UpdateConstants(pContext, Attribs.Width, Attribs.Height, Attribs.Params.EdgeThreshold);
SetSRBVariable(m_HdrTestSRB, SHADER_TYPE_COMPUTE, "u_ProcessedOutputColor", Attribs.pProcessedOutputUAV, true);
Dispatch(pContext, m_HdrTestPSO, m_HdrTestSRB, Attribs.Width, Attribs.Height);
m_Stats.LastHdrTestExecuted = true;
++m_Stats.HdrTestDispatchCount;
return true;
```

Expected: HDR test runs after bloom and before tone mapping, just like RTXPT-fork.

- [ ] **Step 5: Implement LDR edge-detection dispatch**

Implement `RunEdgeDetection()`:

```cpp
m_Stats.LastEdgeDetectionExecuted = false;
if (!Attribs.Params.EnableEdgeDetection)
    return true;
if (!IsReady() || pContext == nullptr || Attribs.pLdrColorTexture == nullptr ||
    Attribs.pLdrColorScratchTexture == nullptr || Attribs.pLdrColorScratchSRV == nullptr ||
    Attribs.pLdrColorUAV == nullptr || Attribs.Width == 0 || Attribs.Height == 0)
    return false;

CopyTextureAttribs CopyAttribs{Attribs.pLdrColorTexture,
                               RESOURCE_STATE_TRANSITION_MODE_TRANSITION,
                               Attribs.pLdrColorScratchTexture,
                               RESOURCE_STATE_TRANSITION_MODE_TRANSITION};
pContext->SetRenderTargets(0, nullptr, nullptr, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
pContext->CopyTexture(CopyAttribs);

UpdateConstants(pContext, Attribs.Width, Attribs.Height, Attribs.Params.EdgeThreshold);
SetSRBVariable(m_EdgeDetectionSRB, SHADER_TYPE_COMPUTE, "t_LdrColorScratch", Attribs.pLdrColorScratchSRV, true);
SetSRBVariable(m_EdgeDetectionSRB, SHADER_TYPE_COMPUTE, "u_PostTonemapOutputColor", Attribs.pLdrColorUAV, true);
Dispatch(pContext, m_EdgeDetectionPSO, m_EdgeDetectionSRB, Attribs.Width, Attribs.Height);
m_Stats.LastEdgeDetectionExecuted = true;
++m_Stats.EdgeDetectionDispatchCount;
return true;
```

Expected: copy/ping-pong behavior matches `Sample.cpp:1870` before edge detection dispatch.

- [ ] **Step 6: Run post-process pass checks**

Run:

```powershell
rg -n "RTXPTPostProcessPass|RTXPT_POST_PROCESS_HDR_TEST|RTXPT_POST_PROCESS_EDGE_DETECTION|CopyTextureAttribs|EdgeDetectionDispatchCount|SetSRBVariable" DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.* DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh
```

Expected: HDR test and LDR edge-detection paths are both present and independently gated.

- [ ] **Step 7: Commit P4 compute post-process pass**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTPostProcessPass.hpp Samples/RTXPT/src/RTXPTPostProcessPass.cpp Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh
git -C DiligentSamples commit -m "feat(rtxpt): port P4 HDR and LDR post-process pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only P4 compute post-process files.

### Task 5: Expose LDR Texture Handles for Copy

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add texture accessors**

Modify `RTXPTRenderTargets.hpp` near the existing LDR view accessors:

```cpp
    ITexture* GetLdrColorTexture() const;
    ITexture* GetLdrColorScratchTexture() const;
```

Implement in `RTXPTRenderTargets.cpp`:

```cpp
ITexture* RTXPTRenderTargets::GetLdrColorTexture() const
{
    return m_LdrColor;
}

ITexture* RTXPTRenderTargets::GetLdrColorScratchTexture() const
{
    return m_LdrColorScratch;
}
```

Expected: only texture handles needed by `CopyTextureAttribs` are exposed; ownership stays inside `RTXPTRenderTargets`.

- [ ] **Step 2: Run accessor grep**

Run:

```powershell
rg -n "GetLdrColorTexture|GetLdrColorScratchTexture" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.*
```

Expected: declarations and definitions are present.

- [ ] **Step 3: Commit LDR texture accessors**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose LDR post-process texture handles" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only render-target accessor changes.

### Task 6: Integrate P4 Into `RTXPTPostProcessPipeline`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp:2194-2203`

- [ ] **Step 1: Add includes, stats, members, and methods**

Modify `RTXPTPostProcessPipeline.hpp`:

```cpp
#include "RTXPTBloomPass.hpp"
#include "RTXPTPostProcessPass.hpp"
```

Extend `RTXPTPostProcessPipelineStats`:

```cpp
    bool BloomStageReady = false;
    bool PostProcessStageReady = false;
```

Add public methods:

```cpp
    bool RunPreToneMapping(IDeviceContext*                pContext,
                           const RTXPTRenderTargets&     RenderTargets,
                           const RTXPTBloomParameters&   BloomParams,
                           const RTXPTPostProcessParameters& PostProcessParams);

    bool RunPostToneMapping(IDeviceContext*                    pContext,
                            const RTXPTRenderTargets&         RenderTargets,
                            const RTXPTPostProcessParameters& PostProcessParams);
```

Add private members:

```cpp
    RTXPTBloomPass       m_BloomPass;
    RTXPTPostProcessPass m_PostProcessPass;
```

Expected: orchestration remains centralized in `RTXPTPostProcessPipeline`.

- [ ] **Step 2: Reset P4 passes**

Modify `RTXPTPostProcessPipeline::Reset()`:

```cpp
    m_BloomPass.Reset();
    m_PostProcessPass.Reset();
```

Expected: window resize and sample shutdown release P4 resources with P2/P3 resources.

- [ ] **Step 3: Initialize P4 passes**

Modify `RTXPTPostProcessPipeline::Initialize()` after tone-mapping initialization:

```cpp
m_Stats.BloomStageReady = m_BloomPass.Initialize(pDevice, pEngineFactory);
if (!m_Stats.BloomStageReady)
{
    DEV_ERROR("RTXPT bloom pass failed to initialize");
    return false;
}

m_Stats.PostProcessStageReady = m_PostProcessPass.Initialize(pDevice, pEngineFactory, ComputeSupported);
if (!m_Stats.PostProcessStageReady)
{
    DEV_ERROR("RTXPT P4 post-process pass failed to initialize");
    return false;
}
```

Expected: P4 fails early with explicit diagnostics if shader or PSO creation fails.

- [ ] **Step 4: Validate P4 resources**

Modify `RTXPTPostProcessPipeline::ValidateRenderTargets()` so `ResourcesValid` also requires:

```cpp
RenderTargets.GetProcessedOutputColorSRV() != nullptr &&
RenderTargets.GetProcessedOutputColorUAV() != nullptr &&
RenderTargets.GetProcessedOutputColorRTV() != nullptr &&
RenderTargets.GetLdrColorUAV() != nullptr &&
RenderTargets.GetLdrColorScratchSRV() != nullptr
```

Expected: validation catches missing P4 views before render.

- [ ] **Step 5: Implement `RunPreToneMapping()`**

Add to `RTXPTPostProcessPipeline.cpp`:

```cpp
bool RTXPTPostProcessPipeline::RunPreToneMapping(IDeviceContext*                    pContext,
                                                 const RTXPTRenderTargets&         RenderTargets,
                                                 const RTXPTBloomParameters&       BloomParams,
                                                 const RTXPTPostProcessParameters& PostProcessParams)
{
    if (!m_BloomPass.ResizeResources(m_Device, RenderTargets.GetWidth(), RenderTargets.GetHeight(), RenderTargets.GetProcessedOutputColorFormat()))
    {
        m_Stats.BloomStageReady = m_BloomPass.IsReady();
        DEV_ERROR("RTXPT bloom pass failed to resize resources");
        return false;
    }

    RTXPTBloomRenderAttribs BloomAttribs;
    BloomAttribs.pSourceSRV = RenderTargets.GetProcessedOutputColorSRV();
    BloomAttribs.pTargetRTV = RenderTargets.GetProcessedOutputColorRTV();
    BloomAttribs.Width      = RenderTargets.GetWidth();
    BloomAttribs.Height     = RenderTargets.GetHeight();
    BloomAttribs.Format     = RenderTargets.GetProcessedOutputColorFormat();
    BloomAttribs.Params     = BloomParams;

    if (!m_BloomPass.Render(pContext, BloomAttribs))
    {
        DEV_ERROR("RTXPT bloom pass failed to render");
        return false;
    }

    RTXPTPostProcessRenderAttribs PostAttribs;
    PostAttribs.pProcessedOutputUAV = RenderTargets.GetProcessedOutputColorUAV();
    PostAttribs.Width               = RenderTargets.GetWidth();
    PostAttribs.Height              = RenderTargets.GetHeight();
    PostAttribs.Params              = PostProcessParams;

    if (!m_PostProcessPass.RunHdrTest(pContext, PostAttribs))
    {
        DEV_ERROR("RTXPT HDR post-process test failed");
        return false;
    }

    m_Stats.BloomStageReady       = m_BloomPass.IsReady();
    m_Stats.PostProcessStageReady = m_PostProcessPass.IsReady();
    return true;
}
```

Expected: pre-tone order is bloom first, HDR test second.

- [ ] **Step 6: Implement `RunPostToneMapping()`**

Add to `RTXPTPostProcessPipeline.cpp`:

```cpp
bool RTXPTPostProcessPipeline::RunPostToneMapping(IDeviceContext*                    pContext,
                                                  const RTXPTRenderTargets&         RenderTargets,
                                                  const RTXPTPostProcessParameters& PostProcessParams)
{
    RTXPTPostProcessRenderAttribs Attribs;
    Attribs.pLdrColorTexture        = RenderTargets.GetLdrColorTexture();
    Attribs.pLdrColorScratchTexture = RenderTargets.GetLdrColorScratchTexture();
    Attribs.pLdrColorScratchSRV     = RenderTargets.GetLdrColorScratchSRV();
    Attribs.pLdrColorUAV            = RenderTargets.GetLdrColorUAV();
    Attribs.Width                   = RenderTargets.GetWidth();
    Attribs.Height                  = RenderTargets.GetHeight();
    Attribs.Params                  = PostProcessParams;

    if (!m_PostProcessPass.RunEdgeDetection(pContext, Attribs))
    {
        DEV_ERROR("RTXPT LDR edge detection failed");
        return false;
    }

    m_Stats.PostProcessStageReady = m_PostProcessPass.IsReady();
    return true;
}
```

This step uses the Task 5 `GetLdrColorTexture()` and `GetLdrColorScratchTexture()` accessors.

Expected: post-tone order matches `Sample.cpp:2203`.

- [ ] **Step 7: Run pipeline integration grep**

Run:

```powershell
rg -n "RunPreToneMapping|RunPostToneMapping|RTXPTBloomPass|RTXPTPostProcessPass|BloomStageReady|PostProcessStageReady" DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.*
```

Expected: P4 stages are owned by the pipeline and reflected in stats.

- [ ] **Step 8: Commit pipeline integration**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
git -C DiligentSamples commit -m "feat(rtxpt): schedule P4 post-process stages" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only pipeline integration.

### Task 7: Wire P4 Render Order in `RTXPTSample`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp:2194-2203`

- [ ] **Step 1: Build P4 settings before post-process calls**

In `RTXPTSample::Render()`, after successful `RunAccumulation()` and before tone mapping, build the P4 settings:

```cpp
    RTXPTBloomParameters BloomParams;
    BloomParams.Enabled   = m_ReferenceUI.EnableBloom;
    BloomParams.Radius    = BloomRadius;
    BloomParams.Intensity = BloomIntensity;

    RTXPTPostProcessParameters PostProcessParams;
    PostProcessParams.EnableHdrTest       = m_ReferenceUI.PostProcessTestPassHDR;
    PostProcessParams.EnableEdgeDetection = m_ReferenceUI.PostProcessEdgeDetection;
    PostProcessParams.EdgeThreshold       = EdgeThreshold;
```

Expected: settings are copied from UI state once per frame.

- [ ] **Step 2: Insert HDR pre-tone stage before tone mapping**

Insert immediately after the accumulation success block and before `RunToneMapping()`:

```cpp
    const bool PreTonePostProcessExecuted =
        m_PostProcessPipeline.RunPreToneMapping(m_pImmediateContext,
                                                m_RenderTargets,
                                                BloomParams,
                                                PostProcessParams);
    if (!PreTonePostProcessExecuted)
    {
        ClearFallback(float4{0.9f, 0.2f, 0.6f, 1.0f});
        return;
    }
```

Expected: failed P4 pre-tone resources produce a distinct visible fallback and do not continue into tone mapping.

- [ ] **Step 3: Insert LDR post-tone stage after tone mapping**

Insert immediately after the tone mapping success block and before optional debug compute:

```cpp
    const bool PostTonePostProcessExecuted =
        m_PostProcessPipeline.RunPostToneMapping(m_pImmediateContext,
                                                 m_RenderTargets,
                                                 PostProcessParams);
    if (!PostTonePostProcessExecuted)
    {
        ClearFallback(float4{0.6f, 0.2f, 0.9f, 1.0f});
        return;
    }
```

Expected: final blit still reads `LdrColor`, now after optional LDR edge detection.

- [ ] **Step 4: Add P4 debug UI counters**

In the diagnostics section near existing post-process stats, add:

```cpp
        const auto& PostStats = m_PostProcessPipeline.GetStats();
        ImGui::Text("Bloom stage: %s", PostStats.BloomStageReady ? "ready" : "not ready");
        ImGui::Text("P4 post-process stage: %s", PostStats.PostProcessStageReady ? "ready" : "not ready");
```

Expected: P4 stage readiness is visible next to P2/P3 diagnostics.

- [ ] **Step 5: Run render-order grep**

Run:

```powershell
rg -n "RunAccumulation|RunPreToneMapping|RunToneMapping|RunPostToneMapping|GetLdrColorSRV" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected order in output: `RunAccumulation`, `RunPreToneMapping`, `RunToneMapping`, `RunPostToneMapping`, `GetLdrColorSRV`.

### Task 8: Register Files in CMake and Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Register new C++ sources and headers**

Modify `DiligentSamples/Samples/RTXPT/CMakeLists.txt` source lists:

```cmake
    src/RTXPTBloomPass.cpp
    src/RTXPTPostProcessPass.cpp
```

and headers:

```cmake
    src/RTXPTBloomPass.hpp
    src/RTXPTPostProcessPass.hpp
```

Expected: new C++ files are part of the RTXPT sample target.

- [ ] **Step 2: Register new shaders**

Add to the shader list:

```cmake
    assets/shaders/PostProcessing/RTXPTBloomShared.h
    assets/shaders/PostProcessing/RTXPTBloomCopy.psh
    assets/shaders/PostProcessing/RTXPTBloomBlur.psh
    assets/shaders/PostProcessing/RTXPTPostProcess.csh
```

Expected: shader files are copied/packaged with other RTXPT shaders.

- [ ] **Step 3: Update mapping rows**

In `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`, replace the P4 rows for bloom and post-process with explicit targets:

```markdown
| `External/Donut/include/donut/render/BloomPass.h` | `src/RTXPTBloomPass.hpp` | P4 | Diligent bloom pass interface, settings, stats, and temporary-resource ownership. |
| `External/Donut/src/render/BloomPass.cpp` | `src/RTXPTBloomPass.cpp` | P4 | Two half-resolution downscales, quarter-resolution horizontal/vertical blur, and blend-constant apply into `ProcessedOutputColor`. |
| `External/Donut/include/donut/shaders/bloom_cb.h` | `assets/shaders/PostProcessing/RTXPTBloomShared.h` | P4 | `RTXPTBloomConstants` layout matching Donut `BloomConstants`. |
| `External/Donut/shaders/passes/bloom_ps.hlsl` | `assets/shaders/PostProcessing/RTXPTBloomBlur.psh` | P4 | Gaussian blur shader formula. |
| `Shaders/TestRaygenPP.hlsl` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` | P4 | HDR test circle and LDR Sobel edge-detection compute shader. |
| `Sample.cpp::PostProcessPreToneMapping` | `src/RTXPTPostProcessPipeline.cpp`, `src/RTXPTBloomPass.cpp`, `src/RTXPTPostProcessPass.cpp` | P4 | HDR post-process scheduling: bloom first, optional HDR test second. |
| `Sample.cpp::PostProcessPostToneMapping` | `src/RTXPTPostProcessPipeline.cpp`, `src/RTXPTPostProcessPass.cpp` | P4 | LDR post-process scheduling after tone mapping, including `LdrColor` to `LdrColorScratch` copy. |
```

Expected: mapping names every P4 source anchor and Diligent destination.

- [ ] **Step 4: Run CMake/mapping grep**

Run:

```powershell
rg -n "RTXPTBloomPass|RTXPTPostProcessPass|RTXPTBloomShared|RTXPTBloomCopy|RTXPTBloomBlur|RTXPTPostProcess.csh" DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: all new files are registered and mapped.

### Task 9: Verify Build, Shader Compile, and Runtime Behavior

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: build output

- [ ] **Step 1: Run targeted source checks**

Run:

```powershell
rg -n "ToneMapACES|exposureScale|u_AccumulationBuffer" DiligentSamples/Samples/RTXPT/assets/shaders DiligentSamples/Samples/RTXPT/src
rg -n "RunAccumulation|RunPreToneMapping|RunToneMapping|RunPostToneMapping" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
rg -n "EnableBloom|BloomRadius|BloomIntensity|PostProcessTestPassHDR|PostProcessEdgeDetection" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
```

Expected: first command has no stale raygen tone-map/accumulation output path matches; second command shows correct P4 order; third command shows P4 UI state and controls.

- [ ] **Step 2: Configure/build the sample**

Run the repository's normal Debug build:

```powershell
.\build-x64-Debug.bat
```

Expected: configure and build complete without C++ or HLSL compile errors. If the full build is already configured, this faster command is acceptable:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: `RTXPT` target builds successfully.

- [ ] **Step 3: Run smoke checks**

Launch the RTXPT sample from the Debug build output. Use the same scene and camera for each check:

1. Bloom disabled, HDR test disabled, LDR edge disabled: image matches the P3 baseline.
2. Bloom enabled with radius `8.0` and intensity `0.004`: bright HDR areas softly bleed before tone mapping.
3. Bloom enabled with intensity `0.0`: image matches bloom disabled.
4. HDR test enabled: a blue-tinted HDR circle appears near pixel center `(800, 500)` before tone mapping.
5. LDR edge detection enabled with threshold `0.1`: final image becomes the Sobel-style edge/test output.
6. LDR edge detection disabled after being enabled: final image returns to normal `LdrColor` without stale scratch output.
7. Tone mapping disabled while bloom is enabled: `ProcessedOutputColor` still receives bloom, and `LdrColor` still receives the P3 pass-through.

Expected: toggles are independent and no toggle forces accumulation reset unless an existing UI path deliberately requests one.

- [ ] **Step 4: Compare against RTXPT-fork behavior**

In `D:/RTXPT-fork`, run the same UI toggles and compare:

- `Enable Bloom=true`, `Bloom Width (Pixels)=8.0`, `Bloom Intensity=0.004`.
- `PostProcessTestPass=true`.
- `EdgeDetection=true`, `EdgeDetectionThreshold=0.1`.

Expected: bloom softness/intensity, HDR test circle, and LDR Sobel output match the RTXPT-fork controls and scheduling.

- [ ] **Step 5: Final status check**

Run:

```powershell
git status --short
git -C DiligentSamples status --short
```

Expected: only P4 files, CMake registration, mapping, and deliberate sample modifications are dirty.

---

## Acceptance Gate

P4 is complete only when all of these are true:

- Render order is `RunAccumulation -> RunPreToneMapping -> RunToneMapping -> RunPostToneMapping -> final LdrColor blit`.
- Bloom uses the Donut/RTXPT constants and formulas: two downscales, quarter-res two-pass blur, `EffectiveSigma = clamp(Radius * 0.25, 1, 100)`, `ArgumentScale = -1 / (2 * sigma^2)`, `NormalizationScale = 1 / (sqrt(2*pi) * sigma)`, and `NumSamples = round(sigma * 4)`.
- Bloom apply uses blend factors equivalent to Donut constant-color blending: source `BlendFactor`, destination `InvBlendFactor`, alpha source `Zero`, alpha destination `One`.
- HDR test uses the upstream pixel-space circle at `(800, 500)` with radius `100` and adds `10` to blue.
- LDR edge detection copies `LdrColor` to `LdrColorScratch`, then applies the upstream Sobel formula and threshold.
- Disabling bloom, HDR test, or LDR edge detection leaves the P3 base display chain unchanged.
- P4 files are registered in CMake and mapped in `RTXPT_FORK_MAPPING.md`.
- Build/shader compile succeeds for the RTXPT target.

## Self-Review Checklist

- Spec coverage: G4 is covered by bloom integration, HDR post-process scheduling, LDR post-process scheduling, independent toggles, and `LdrColorScratch` ping-pong.
- Source parity: every P4 behavior points to an RTXPT-fork source anchor and Diligent destination.
- Completeness scan: no open implementation gaps are intentionally left in this plan.
- Type consistency: `RTXPTBloomParameters`, `RTXPTPostProcessParameters`, `RunPreToneMapping`, and `RunPostToneMapping` names are used consistently across tasks.
