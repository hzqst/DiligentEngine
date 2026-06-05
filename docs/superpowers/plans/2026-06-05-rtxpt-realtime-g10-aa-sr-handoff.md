# RTXPT Realtime G10 AA/SR Handoff Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route realtime RTXPT denoise/no-denoise HDR output through the selected non-DLSS-RR AA/SR handoff before the existing HDR post-process, tone mapping, LDR post-process, and blit chain.

**Architecture:** Treat `OutputColor` as the realtime merged HDR radiance texture and `ProcessedOutputColor` as the post-AA/SR HDR texture consumed by bloom/tone mapping. Add a DiligentFX-backed TAA wrapper for `RealtimeAA == 1`, reuse the existing `RTXPTSuperResolutionPass` for `RealtimeAA == 2`, keep `RealtimeAA == 0` as an explicit `OutputColor -> ProcessedOutputColor` copy/pass, and leave `RealtimeAA == 3` behind a visible `TODO(RTXPT-Realtime-DLSS-RR)` disabled path.

**Tech Stack:** C++17, Diligent Engine render targets and state transitions, DiligentFX `PostFXContext` and `TemporalAntiAliasing`, Diligent `ISuperResolution`, HLSL/DXC, ImGui, RTXPT-fork references `Sample.cpp::PostProcessAA`, `Sample.cpp::ComputeCameraJitter`, and `SampleCommon/RenderTargets.*`.

---

## Scope

Implements:

- `RealtimeAA == 0`: realtime final/no-denoiser merge leaves HDR radiance in `OutputColor`; copy/pass it to `ProcessedOutputColor`.
- `RealtimeAA == 1`: run DiligentFX temporal AA on `OutputColor`, using current depth, previous depth, screen motion vectors, current/previous view state, and reset-history flags, then copy the TAA accumulated result to `ProcessedOutputColor`.
- `RealtimeAA == 2`: resolve render/display dimensions through the existing `RTXPTSuperResolutionPass`, run realtime path tracing at render size, execute temporal super-resolution from `OutputColor` into display-size `ProcessedOutputColor`, and preserve reset-history semantics.
- `RealtimeAA == 3`: keep DLSS-RR unavailable with a narrow `TODO(RTXPT-Realtime-DLSS-RR)` marker and no executing path.
- Realtime camera jitter selection before path-trace dispatch for TAA/SR modes.
- Previous-depth storage needed by DiligentFX `PostFXContext`.
- UI availability/status updates and mapping-document rows for G10.

Does not implement:

- Streamline, DLSS, DLSS-RR, DLSS-RR prepare inputs, resource tagging, or `EvaluateDLSSRR`.
- New NRD prepare/final-merge algorithms. G7/G8/G9 already own denoise and merge.
- Reference-mode AA/SR. Reference PathTracer remains on the existing accumulation path.
- Moving-geometry motion-vector quality improvements beyond consuming the current `ScreenMotionVectors` contract.

---

## Current Baseline

- `RTXPTSample::PresentRealtimeFinalOutput()` currently constructs `DisabledSuperResolution`, so SR can never execute even if `RealtimeAA == SuperResolution`.
- `kRTXPTRealtimeTaaAvailable` and `kRTXPTRealtimeSrAvailable` are currently `false`, so the UI exposes TAA/SR as disabled placeholders.
- `UpdateFrameConstants()` currently uses `CameraJitter = float2{0.0f, 0.0f}` for all non-DLSS-RR realtime paths.
- G7/G8/G9 code binds realtime merge output through `RTXPTRenderTargets::GetAccumulationOutputUAV()`. For G10, the post-denoise contract should be `OutputColor -> AA/SR -> ProcessedOutputColor`.
- `RTXPTRenderTargets` has current `Depth` but no previous-depth resource, while DiligentFX `PostFXContext::Execute()` requires both current and previous depth SRVs.
- `RTXPTSuperResolutionPass::Execute()` always reads `RenderTargets.GetSuperResolutionColorSRV()`. Realtime SR should read merged realtime `OutputColor`.

## Source Anchors

Read before editing:

- `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md` - G10 and Phase RT5 acceptance.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2621` - `Sample::PostProcessAA` mode ordering.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2818` - upstream realtime jitter source for TAA.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - `UpdateRenderTargetDimensions`, `UpdateFrameConstants`, `PresentRealtimeFinalOutput`, UI combo, and status panel.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}` - denoiser merge wrappers, SR wrapper, and HDR chain.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.{hpp,cpp}` - Diligent `ISuperResolution` wrapper.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}` - render/display-size resources.
- `DiligentFX/PostProcess/TemporalAntiAliasing/interface/TemporalAntiAliasing.hpp` - DiligentFX TAA API.
- `DiligentFX/PostProcess/Common/interface/PostFXContext.hpp` - current/previous depth and camera inputs for temporal effects.
- `DiligentSamples/Tutorials/Tutorial27_PostProcessing/src/Tutorial27_PostProcessing.cpp:360-400`, `590-620`, `885-900` - local TAA/SR integration examples.

---

## File Structure

Create:

- `DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.hpp`
  Owns the RTXPT-local wrapper around DiligentFX `PostFXContext` and `TemporalAntiAliasing`.
- `DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.cpp`
  Converts RTXPT frame constants into DiligentFX camera attributes, prepares PostFX/TAA resources, executes TAA, copies TAA output to `ProcessedOutputColor`, and updates previous depth.

Modify:

- `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  Register the new TAA pass files. `DiligentFX` is already linked.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
  Add `PreviousDepth` format, accessors, and member.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
  Allocate previous depth for realtime resources and validate its SRV/RTV support.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp`
  Add an overload or optional source/output view parameters for realtime SR.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp`
  Use explicit realtime source/output views when provided.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
  Own `RTXPTTemporalAAPass`, expose realtime copy/TAA/SR methods, and add stage stats.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
  Initialize/reset TAA, bind realtime merge output to `OutputColor`, run AA0/TAA/SR handoff, and preserve SR wrappers.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  Add current realtime camera jitter and, if needed, helper declarations.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  Resolve realtime AA/SR dimensions, compute camera jitter, set `superResolutionActive`, route `PresentRealtimeFinalOutput()` by AA mode, update UI/status, and keep DLSS-RR disabled.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  Add G10 mapping rows for `PostProcessAA`, DiligentFX TAA, SR handoff, and DLSS-RR disabled status.

Reference but do not modify:

- `DiligentFX/PostProcess/TemporalAntiAliasing/*`
- `DiligentFX/PostProcess/Common/*`
- `DiligentCore/Graphics/SuperResolution/interface/*`

---

### Task 0: Baseline Preflight

**Files:**

- Verify: top-level repository
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: `DiligentFX/PostProcess/TemporalAntiAliasing`
- Verify: `DiligentCore/Graphics/SuperResolution/interface`

- [ ] **Step 1: Check working tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files must be inspected and preserved before editing.

- [ ] **Step 2: Confirm current G10 gaps**

Run:

```powershell
rg -n "DisabledSuperResolution|kRTXPTRealtimeTaaAvailable|kRTXPTRealtimeSrAvailable|CameraJitter|PresentRealtimeFinalOutput|GetAccumulationOutputUAV" DiligentSamples/Samples/RTXPT/src
```

Expected:

```text
RTXPTSample.cpp contains DisabledSuperResolution in PresentRealtimeFinalOutput.
RTXPTSample.cpp contains kRTXPTRealtimeTaaAvailable = false and kRTXPTRealtimeSrAvailable = false.
RTXPTSample.cpp contains CameraJitter = float2{0.0f, 0.0f}.
RTXPTPostProcessPipeline.cpp binds realtime merge output through GetAccumulationOutputUAV().
```

- [ ] **Step 3: Confirm DiligentFX TAA and SR APIs**

Run:

```powershell
rg -n "class TemporalAntiAliasing|struct RenderAttributes|GetJitterOffset|PrepareResources|GetAccumulatedFrameSRV" DiligentFX/PostProcess/TemporalAntiAliasing/interface/TemporalAntiAliasing.hpp
rg -n "class PostFXContext|pCurrDepthBufferSRV|pPrevDepthBufferSRV|pMotionVectorsSRV|CopyTextureColor|CopyTextureDepth" DiligentFX/PostProcess/Common/interface/PostFXContext.hpp
rg -n "ExecuteSuperResolutionAttribs|GetJitterOffset|CreateSuperResolution" DiligentCore/Graphics/SuperResolution/interface
```

Expected: all listed API names are found.

- [ ] **Step 4: Confirm upstream ordering**

Run:

```powershell
rg -n "void Sample::PostProcessAA|RealtimeAA == 0|RealtimeAA == 1|RealtimeAA == 2|RealtimeAA == 3|EvaluateDLSSRR|ComputeCameraJitter" D:/RTXPT-fork/Rtxpt/Sample.cpp
```

Expected: upstream `PostProcessAA` mode branches and DLSS-RR execution anchor are found. G10 must only port modes 0, 1, and 2.

- [ ] **Step 5: Commit baseline status only if policy requires a checkpoint**

No source changes are made in Task 0. Do not create a commit unless the execution environment requires a preflight checkpoint.

---

### Task 1: Register the RTXPT Temporal AA Wrapper

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.cpp`

- [ ] **Step 1: Register the new files**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the source after `src/RTXPTSuperResolutionPass.cpp`:

```cmake
    src/RTXPTTemporalAAPass.cpp
```

Add the header after `src/RTXPTSuperResolutionPass.hpp`:

```cmake
    src/RTXPTTemporalAAPass.hpp
```

`target_link_libraries(RTXPT PRIVATE DiligentFX)` already exists. Keep it unchanged.

- [ ] **Step 2: Create `RTXPTTemporalAAPass.hpp`**

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
 */

#pragma once

#include <memory>
#include <string>

#include "BasicMath.hpp"
#include "DeviceContext.h"
#include "RenderDevice.h"
#include "RTXPTFrameConstants.hpp"
#include "RTXPTRenderTargets.hpp"

#include "PostFXContext.hpp"
#include "TemporalAntiAliasing.hpp"

namespace Diligent
{

struct RTXPTTemporalAASettings
{
    float TemporalStabilityFactor = 0.9375f;
    bool  SkipRejection           = false;
};

struct RTXPTTemporalAAFrameAttribs
{
    IRenderDevice*              pDevice           = nullptr;
    IDeviceContext*             pDeviceContext    = nullptr;
    const RTXPTRenderTargets*   pRenderTargets    = nullptr;
    const SampleConstants*      pFrameConstants   = nullptr;
    RTXPTTemporalAASettings     Settings          = {};
    Uint32                      FrameIndex        = 0;
    bool                        ResetHistory      = false;
    bool                        PreviousViewValid = false;
};

struct RTXPTTemporalAAStats
{
    bool        Ready                  = false;
    bool        LastExecute            = false;
    bool        LastCopyToProcessed    = false;
    bool        LastPreviousDepthCopy  = false;
    Uint32      ExecuteCount           = 0;
    std::string DisabledReason;
};

class RTXPTTemporalAAPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice);

    bool CopyOutputToProcessed(IRenderDevice*            pDevice,
                               IDeviceContext*           pContext,
                               const RTXPTRenderTargets& RenderTargets);

    bool Execute(const RTXPTTemporalAAFrameAttribs& Attribs);

    static float2 ComputeJitter(Uint32 FrameIndex, Uint32 Width, Uint32 Height);

    bool                            IsReady() const { return m_Stats.Ready; }
    const RTXPTTemporalAAStats&     GetStats() const { return m_Stats; }

private:
    bool PreparePostFX(const RTXPTTemporalAAFrameAttribs& Attribs);
    bool CopyCurrentDepthToPrevious(IRenderDevice*            pDevice,
                                    IDeviceContext*           pContext,
                                    const RTXPTRenderTargets& RenderTargets);

private:
    std::unique_ptr<PostFXContext>         m_PostFXContext;
    std::unique_ptr<TemporalAntiAliasing>  m_TemporalAA;
    RTXPTTemporalAAStats                   m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 3: Create `RTXPTTemporalAAPass.cpp`**

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
 */

#include "RTXPTTemporalAAPass.hpp"

#include <algorithm>

#include "DebugUtilities.hpp"

namespace Diligent
{

namespace HLSL
{
#include "Shaders/Common/public/ShaderDefinitions.fxh"
#include "Shaders/Common/public/BasicStructures.fxh"
} // namespace HLSL

namespace
{

float Halton(Uint32 Base, Uint32 Index)
{
    float Result = 0.0f;
    float Factor = 1.0f;
    while (Index > 0)
    {
        Factor /= static_cast<float>(Base);
        Result += Factor * static_cast<float>(Index % Base);
        Index /= Base;
    }
    return Result;
}

HLSL::CameraAttribs MakeCameraAttribs(const SampleConstants& Constants,
                                      const PathTracerViewData& View,
                                      const PathTracerCameraData& Camera,
                                      Uint32 FrameIndex)
{
    const float4x4 JitteredProj = TemporalAntiAliasing::GetJitteredProjMatrix(View.MatViewToClip, View.PixelOffset);
    const float4x4 ViewProj     = View.MatWorldToView * JitteredProj;

    HLSL::CameraAttribs Attribs = {};
    Attribs.f4Position          = float4{Camera.PosW, 1.0f};
    Attribs.f4ViewportSize      = float4{View.ViewportSize.x, View.ViewportSize.y, View.ViewportSizeInv.x, View.ViewportSizeInv.y};
    Attribs.SetClipPlanes(Camera.NearZ, Camera.FarZ);
    Attribs.fHandness           = 1.0f;
    Attribs.uiFrameIndex        = FrameIndex;
    Attribs.fFocusDistance      = Camera.FocalDistance;
    Attribs.fFStop              = Camera.ApertureRadius > 0.0f ? 1.0f / Camera.ApertureRadius : 5.6f;
    Attribs.fFocalLength        = 50.0f;
    Attribs.fSensorWidth        = 36.0f;
    Attribs.fSensorHeight       = 24.0f;
    Attribs.fExposure           = 0.0f;
    Attribs.f2Jitter            = View.PixelOffset;
    Attribs.mView               = View.MatWorldToView;
    Attribs.mProj               = JitteredProj;
    Attribs.mViewProj           = ViewProj;
    Attribs.mViewInv            = View.MatWorldToView.Inverse();
    Attribs.mProjInv            = JitteredProj.Inverse();
    Attribs.mViewProjInv        = ViewProj.Inverse();
    Attribs.f4ExtraData[0]      = Constants.cameraPositionAndTime;
    return Attribs;
}

bool ValidateTemporalInputs(const RTXPTTemporalAAFrameAttribs& Attribs, std::string& Reason)
{
    if (Attribs.pDevice == nullptr)
        Reason = "render device is null";
    else if (Attribs.pDeviceContext == nullptr)
        Reason = "device context is null";
    else if (Attribs.pRenderTargets == nullptr)
        Reason = "render targets are null";
    else if (Attribs.pFrameConstants == nullptr)
        Reason = "frame constants are null";
    else if (Attribs.pRenderTargets->GetOutputColorSRV() == nullptr)
        Reason = "OutputColor SRV is null";
    else if (Attribs.pRenderTargets->GetProcessedOutputColorRTV() == nullptr)
        Reason = "ProcessedOutputColor RTV is null";
    else if (Attribs.pRenderTargets->GetDepthSRV() == nullptr)
        Reason = "Depth SRV is null";
    else if (Attribs.pRenderTargets->GetPreviousDepthSRV() == nullptr)
        Reason = "PreviousDepth SRV is null";
    else if (Attribs.pRenderTargets->GetPreviousDepthRTV() == nullptr)
        Reason = "PreviousDepth RTV is null";
    else if (Attribs.pRenderTargets->GetScreenMotionVectorsSRV() == nullptr)
        Reason = "ScreenMotionVectors SRV is null";
    else
        return true;
    return false;
}

} // namespace

void RTXPTTemporalAAPass::Reset()
{
    m_TemporalAA.reset();
    m_PostFXContext.reset();
    m_Stats = {};
}

bool RTXPTTemporalAAPass::Initialize(IRenderDevice* pDevice)
{
    Reset();
    if (pDevice == nullptr)
    {
        m_Stats.DisabledReason = "render device is null";
        return false;
    }

    PostFXContext::CreateInfo PostFXCI;
    PostFXCI.EnableAsyncCreation = false;
    PostFXCI.PackMatrixRowMajor  = false;
    m_PostFXContext = std::make_unique<PostFXContext>(pDevice, PostFXCI);

    TemporalAntiAliasing::CreateInfo TAACI;
    TAACI.EnableAsyncCreation = false;
    m_TemporalAA = std::make_unique<TemporalAntiAliasing>(pDevice, TAACI);

    m_Stats.Ready = m_PostFXContext != nullptr && m_TemporalAA != nullptr;
    if (!m_Stats.Ready)
        m_Stats.DisabledReason = "failed to create DiligentFX TAA objects";
    return m_Stats.Ready;
}

float2 RTXPTTemporalAAPass::ComputeJitter(Uint32 FrameIndex, Uint32 Width, Uint32 Height)
{
    const float SafeWidth  = static_cast<float>(std::max(Width, Uint32{1}));
    const float SafeHeight = static_cast<float>(std::max(Height, Uint32{1}));
    constexpr Uint32 SampleCount = 16u;
    const Uint32 Sample = (FrameIndex % SampleCount) + 1u;
    return float2{
        (Halton(2u, Sample) - 0.5f) / (0.5f * SafeWidth),
        (Halton(3u, Sample) - 0.5f) / (0.5f * SafeHeight)};
}

bool RTXPTTemporalAAPass::PreparePostFX(const RTXPTTemporalAAFrameAttribs& Attribs)
{
    PostFXContext::FrameDesc FrameDesc;
    FrameDesc.Index        = Attribs.FrameIndex;
    FrameDesc.Width        = Attribs.pRenderTargets->GetRenderWidth();
    FrameDesc.Height       = Attribs.pRenderTargets->GetRenderHeight();
    FrameDesc.OutputWidth  = Attribs.pRenderTargets->GetDisplayWidth();
    FrameDesc.OutputHeight = Attribs.pRenderTargets->GetDisplayHeight();

    m_PostFXContext->PrepareResources(Attribs.pDevice, FrameDesc, PostFXContext::FEATURE_FLAG_NONE);
    m_TemporalAA->PrepareResources(Attribs.pDevice,
                                   Attribs.pDeviceContext,
                                   m_PostFXContext.get(),
                                   TemporalAntiAliasing::FEATURE_FLAG_BICUBIC_FILTER);
    return true;
}

bool RTXPTTemporalAAPass::CopyOutputToProcessed(IRenderDevice*            pDevice,
                                                IDeviceContext*           pContext,
                                                const RTXPTRenderTargets& RenderTargets)
{
    m_Stats.LastCopyToProcessed = false;
    if (!m_PostFXContext)
    {
        m_Stats.DisabledReason = "PostFXContext is not initialized";
        return false;
    }
    if (pDevice == nullptr || pContext == nullptr)
    {
        m_Stats.DisabledReason = "copy requires a device and context";
        return false;
    }
    if (RenderTargets.GetOutputColorSRV() == nullptr || RenderTargets.GetProcessedOutputColorRTV() == nullptr)
    {
        m_Stats.DisabledReason = "copy requires OutputColor SRV and ProcessedOutputColor RTV";
        return false;
    }

    PostFXContext::TextureOperationAttribs CopyAttribs;
    CopyAttribs.pDevice        = pDevice;
    CopyAttribs.pDeviceContext = pContext;
    m_PostFXContext->CopyTextureColor(CopyAttribs,
                                      RenderTargets.GetOutputColorSRV(),
                                      RenderTargets.GetProcessedOutputColorRTV());
    m_Stats.DisabledReason.clear();
    m_Stats.LastCopyToProcessed = true;
    return true;
}

bool RTXPTTemporalAAPass::CopyCurrentDepthToPrevious(IRenderDevice*            pDevice,
                                                     IDeviceContext*           pContext,
                                                     const RTXPTRenderTargets& RenderTargets)
{
    m_Stats.LastPreviousDepthCopy = false;
    if (RenderTargets.GetDepthSRV() == nullptr || RenderTargets.GetPreviousDepthRTV() == nullptr)
    {
        m_Stats.DisabledReason = "previous-depth update requires Depth SRV and PreviousDepth RTV";
        return false;
    }

    PostFXContext::TextureOperationAttribs CopyAttribs;
    CopyAttribs.pDevice        = pDevice;
    CopyAttribs.pDeviceContext = pContext;
    m_PostFXContext->CopyTextureDepth(CopyAttribs,
                                      RenderTargets.GetDepthSRV(),
                                      RenderTargets.GetPreviousDepthRTV());
    m_Stats.LastPreviousDepthCopy = true;
    return true;
}

bool RTXPTTemporalAAPass::Execute(const RTXPTTemporalAAFrameAttribs& Attribs)
{
    m_Stats.LastExecute         = false;
    m_Stats.LastCopyToProcessed = false;

    if (!m_Stats.Ready || !m_PostFXContext || !m_TemporalAA)
    {
        m_Stats.DisabledReason = "Temporal AA pass is not initialized";
        return false;
    }

    std::string Reason;
    if (!ValidateTemporalInputs(Attribs, Reason))
    {
        m_Stats.DisabledReason = Reason;
        return false;
    }

    PreparePostFX(Attribs);

    const SampleConstants& Constants = *Attribs.pFrameConstants;
    const bool UsePrevious = Attribs.PreviousViewValid && !Attribs.ResetHistory;
    const HLSL::CameraAttribs CurrentCamera =
        MakeCameraAttribs(Constants, Constants.view, Constants.ptConsts.camera, Attribs.FrameIndex);
    const HLSL::CameraAttribs PreviousCamera =
        UsePrevious ?
        MakeCameraAttribs(Constants, Constants.previousView, Constants.ptConsts.prevCamera, Attribs.FrameIndex - 1u) :
        CurrentCamera;

    PostFXContext::RenderAttributes PostFXAttribs;
    PostFXAttribs.pDevice             = Attribs.pDevice;
    PostFXAttribs.pDeviceContext      = Attribs.pDeviceContext;
    PostFXAttribs.pCurrDepthBufferSRV = Attribs.pRenderTargets->GetDepthSRV();
    PostFXAttribs.pPrevDepthBufferSRV = UsePrevious ?
        Attribs.pRenderTargets->GetPreviousDepthSRV() :
        Attribs.pRenderTargets->GetDepthSRV();
    PostFXAttribs.pMotionVectorsSRV   = Attribs.pRenderTargets->GetScreenMotionVectorsSRV();
    PostFXAttribs.pCurrCamera         = &CurrentCamera;
    PostFXAttribs.pPrevCamera         = &PreviousCamera;
    m_PostFXContext->Execute(PostFXAttribs);

    HLSL::TemporalAntiAliasingAttribs TAAAttribs = {};
    TAAAttribs.TemporalStabilityFactor = std::clamp(Attribs.Settings.TemporalStabilityFactor, 0.0f, 1.0f);
    TAAAttribs.ResetAccumulation       = Attribs.ResetHistory ? TRUE : FALSE;
    TAAAttribs.SkipRejection           = Attribs.Settings.SkipRejection ? TRUE : FALSE;

    TemporalAntiAliasing::RenderAttributes TAARenderAttribs;
    TAARenderAttribs.pDevice         = Attribs.pDevice;
    TAARenderAttribs.pDeviceContext  = Attribs.pDeviceContext;
    TAARenderAttribs.pPostFXContext  = m_PostFXContext.get();
    TAARenderAttribs.pColorBufferSRV = Attribs.pRenderTargets->GetOutputColorSRV();
    TAARenderAttribs.pTAAAttribs     = &TAAAttribs;
    m_TemporalAA->Execute(TAARenderAttribs);

    ITextureView* pTaaOutputSRV = m_TemporalAA->GetAccumulatedFrameSRV(false);
    if (pTaaOutputSRV == nullptr)
    {
        m_Stats.DisabledReason = "Temporal AA accumulated output SRV is null";
        return false;
    }

    PostFXContext::TextureOperationAttribs CopyAttribs;
    CopyAttribs.pDevice        = Attribs.pDevice;
    CopyAttribs.pDeviceContext = Attribs.pDeviceContext;
    m_PostFXContext->CopyTextureColor(CopyAttribs,
                                      pTaaOutputSRV,
                                      Attribs.pRenderTargets->GetProcessedOutputColorRTV());
    m_Stats.LastCopyToProcessed = true;

    if (!CopyCurrentDepthToPrevious(Attribs.pDevice, Attribs.pDeviceContext, *Attribs.pRenderTargets))
        return false;

    m_Stats.DisabledReason.clear();
    m_Stats.LastExecute = true;
    ++m_Stats.ExecuteCount;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 4: Configure and build the new empty integration point**

Run:

```powershell
cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

If include paths for `PostFXContext.hpp`, `TemporalAntiAliasing.hpp`, or `Shaders/Common/public/BasicStructures.fxh` are missing, fix only RTXPT target include/link registration and keep the existing `DiligentFX` target unchanged.

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTTemporalAAPass.cpp
git commit -m "feat(rtxpt): add realtime temporal AA pass wrapper"
```

---

### Task 2: Add Previous Depth for Temporal AA

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add the previous-depth format and accessors**

In `RTXPTRenderTargetFormats`, add `PreviousDepth` next to `Depth`:

```cpp
    TEXTURE_FORMAT Depth                     = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT PreviousDepth             = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT ScreenMotionVectors       = TEX_FORMAT_RGBA16_FLOAT;
```

In the public accessor block, add:

```cpp
    ITextureView* GetPreviousDepthSRV() const;
    ITextureView* GetPreviousDepthRTV() const;
```

In the format accessor block, add:

```cpp
    TEXTURE_FORMAT GetPreviousDepthFormat() const { return m_Formats.PreviousDepth; }
```

In private members, add the texture after `m_Depth`:

```cpp
    RefCntAutoPtr<ITexture> m_Depth;
    RefCntAutoPtr<ITexture> m_PreviousDepth;
    RefCntAutoPtr<ITexture> m_ScreenMotionVectors;
```

- [ ] **Step 2: Update format comparison and reset**

In `RTXPTRenderTargets.cpp`, update `FormatsMatch()`:

```cpp
        Lhs.Depth == Rhs.Depth &&
        Lhs.PreviousDepth == Rhs.PreviousDepth &&
        Lhs.ScreenMotionVectors == Rhs.ScreenMotionVectors &&
```

In `RTXPTRenderTargets::Reset()`, release the new texture after `m_Depth.Release()`:

```cpp
    m_Depth.Release();
    m_PreviousDepth.Release();
    m_ScreenMotionVectors.Release();
```

- [ ] **Step 3: Validate previous-depth support for realtime resources**

In `Resize()`, add a render-target bind flag for previous depth:

```cpp
    const BIND_FLAGS UavFlags     = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    const BIND_FLAGS HdrRtFlags   = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;
    const BIND_FLAGS LdrRtFlags   = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;
    const BIND_FLAGS DepthRtFlags = BIND_SHADER_RESOURCE | BIND_RENDER_TARGET;
```

After the current `Depth` support check, add:

```cpp
    if (CreateRealtimeResources && !SupportsBindFlags(pDevice, Formats.PreviousDepth, DepthRtFlags))
        return FailResize("R32F SRV/RTV PreviousDepth is not supported; RTXPT realtime TAA is unavailable");
```

- [ ] **Step 4: Include previous depth in requested target reuse checks**

In the `HasCorePostProcessTargets` predicate, add the realtime previous-depth requirement:

```cpp
        m_Depth != nullptr &&
        (!CreateRealtimeResources || m_PreviousDepth != nullptr) &&
        m_ScreenMotionVectors != nullptr &&
```

- [ ] **Step 5: Create and assign the previous-depth texture**

In the local texture declarations, add:

```cpp
    RefCntAutoPtr<ITexture> PreviousDepth;
```

After creating `Depth`, add:

```cpp
    if (CreateRealtimeResources &&
        !CreateTarget(pDevice, "RTXPT PreviousDepth", RenderWidth, RenderHeight, Formats.PreviousDepth, DepthRtFlags, PreviousDepth))
        return FailResize("Failed to create RTXPT PreviousDepth");
```

In the assignment block, add:

```cpp
    m_Depth                            = Depth;
    m_PreviousDepth                    = PreviousDepth;
    m_ScreenMotionVectors              = ScreenMotionVectors;
```

- [ ] **Step 6: Expose SRV/RTV getters**

Add these implementations near the other depth accessors:

```cpp
ITextureView* RTXPTRenderTargets::GetPreviousDepthSRV() const
{
    return m_PreviousDepth ? m_PreviousDepth->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetPreviousDepthRTV() const
{
    return m_PreviousDepth ? m_PreviousDepth->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET) : nullptr;
}
```

- [ ] **Step 7: Require previous depth in realtime target completeness**

In `HasRealtimeRenderTargets()`, add `m_PreviousDepth != nullptr` to the realtime core requirement:

```cpp
    return m_RealtimeResourcesRequested &&
        m_PreviousDepth != nullptr &&
        m_StableRadiance != nullptr &&
        ...
```

Expected: realtime target allocation now includes current depth, previous depth, motion vectors, and stable-plane/denoiser resources.

- [ ] **Step 8: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 9: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
git commit -m "feat(rtxpt): add previous depth for realtime temporal AA"
```

---

### Task 3: Restore the Realtime Merge Contract to OutputColor

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Bind G7/G8/G9 final merge to `OutputColor`**

In `RTXPTPostProcessPipeline.cpp`, replace `MakeRealtimePostProcessAttribs()` with:

```cpp
RTXPTDenoiserPostProcessAttribs MakeRealtimePostProcessAttribs(const RTXPTRenderTargets& RenderTargets)
{
    RTXPTDenoiserPostProcessAttribs Attribs;
    Attribs.pRenderTargets  = &RenderTargets;
    Attribs.pMergeOutputUAV = RenderTargets.GetOutputColorUAV();
    return Attribs;
}
```

Expected: shader variable `u_OutputColor` is bound to the actual `OutputColor` resource for realtime prepare/final-merge/no-denoiser/debug-viz passes.

- [ ] **Step 2: Update denoiser output barriers**

In `RTXPTSample.cpp`, change `InsertDenoiserPrepareOutputBarriers()` first barrier:

```cpp
    RTXPTRayTracingPass::InsertUAVBarrier(pContext, RenderTargets.GetOutputColorUAV());
```

In `InsertDenoiserFinalMergeOutputBarrier()`, change the barrier to:

```cpp
    RTXPTRayTracingPass::InsertUAVBarrier(pContext, RenderTargets.GetOutputColorUAV());
```

- [ ] **Step 3: Add a no-denoiser output barrier**

In `RunRealtimeNoDenoiserFinalMerge()`, after a successful merge, insert:

```cpp
    if (MergeOk)
        RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetOutputColorUAV());
```

Keep the existing status text.

- [ ] **Step 4: Source-scan the contract**

Run:

```powershell
rg -n "pMergeOutputUAV = RenderTargets.GetOutputColorUAV|GetAccumulationOutputUAV\\(\\)" DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected:

```text
RTXPTPostProcessPipeline.cpp contains pMergeOutputUAV = RenderTargets.GetOutputColorUAV().
RTXPTSample.cpp no longer uses GetAccumulationOutputUAV() in denoiser output barriers.
```

- [ ] **Step 5: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 6: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "fix(rtxpt): route realtime merge through OutputColor"
```

---

### Task 4: Add Pipeline-Level AA0, TAA, and Realtime SR Methods

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`

- [ ] **Step 1: Allow explicit SR source/output views**

In `RTXPTSuperResolutionPass.hpp`, change `Execute()` to:

```cpp
    bool Execute(IDeviceContext*                      pContext,
                 const RTXPTRenderTargets&            RenderTargets,
                 const RTXPTSuperResolutionFrameDesc& FrameDesc,
                 float                                CameraNear,
                 float                                CameraFar,
                 float                                CameraFovAngleVert,
                 ITextureView*                        pColorSRV  = nullptr,
                 ITextureView*                        pOutputUAV = nullptr);
```

In `RTXPTSuperResolutionPass.cpp`, replace the color/output view assignments with:

```cpp
    Attribs.pColorTextureSRV    = pColorSRV != nullptr ? pColorSRV : RenderTargets.GetSuperResolutionColorSRV();
    Attribs.pDepthTextureSRV    = RenderTargets.GetDepthSRV();
    Attribs.pMotionVectorsSRV   = RenderTargets.GetScreenMotionVectorsSRV();
    Attribs.pOutputTextureView  = pOutputUAV != nullptr ? pOutputUAV : RenderTargets.GetSuperResolutionOutputUAV();
```

Keep the existing null checks unchanged so explicit realtime views fail closed if missing.

- [ ] **Step 2: Add TAA pipeline ownership and stats**

In `RTXPTPostProcessPipeline.hpp`, include the wrapper:

```cpp
#include "RTXPTTemporalAAPass.hpp"
```

Add stats fields:

```cpp
    bool TemporalAAStageReady      = false;
    bool LastTemporalAAActive      = false;
    bool LastRealtimeCopyExecuted  = false;
```

Add public methods after `RunSuperResolution()`:

```cpp
    float2 GetRealtimeTAAJitter(Uint32 FrameIndex, Uint32 Width, Uint32 Height) const;

    bool CopyRealtimeOutputToProcessed(IDeviceContext*           pContext,
                                       const RTXPTRenderTargets& RenderTargets);

    bool RunTemporalAA(IDeviceContext*             pContext,
                       const RTXPTRenderTargets&   RenderTargets,
                       const SampleConstants&      FrameConstants,
                       Uint32                      FrameIndex,
                       bool                        ResetHistory,
                       bool                        PreviousViewValid,
                       const RTXPTTemporalAASettings& Settings);

    bool RunRealtimeSuperResolution(IDeviceContext*                      pContext,
                                    const RTXPTRenderTargets&            RenderTargets,
                                    const RTXPTSuperResolutionFrameDesc& FrameDesc,
                                    float                                CameraNear,
                                    float                                CameraFar,
                                    float                                CameraFovAngleVert);

    const RTXPTTemporalAAPass& GetTemporalAAPass() const { return m_TemporalAAPass; }
```

Add the member:

```cpp
    RTXPTTemporalAAPass          m_TemporalAAPass;
```

- [ ] **Step 3: Initialize and reset TAA in the pipeline**

In `RTXPTPostProcessPipeline::Reset()`, add:

```cpp
    m_TemporalAAPass.Reset();
```

In `Initialize()`, after SR initialization and before accumulation initialization, add:

```cpp
    m_Stats.TemporalAAStageReady = m_TemporalAAPass.Initialize(pDevice);
    if (!m_Stats.TemporalAAStageReady)
    {
        DEV_ERROR("RTXPT temporal AA pass failed to initialize");
        return false;
    }
```

- [ ] **Step 4: Implement pipeline methods**

Add these definitions to `RTXPTPostProcessPipeline.cpp` after `RunSuperResolution()`:

```cpp
float2 RTXPTPostProcessPipeline::GetRealtimeTAAJitter(Uint32 FrameIndex, Uint32 Width, Uint32 Height) const
{
    return RTXPTTemporalAAPass::ComputeJitter(FrameIndex, Width, Height);
}

bool RTXPTPostProcessPipeline::CopyRealtimeOutputToProcessed(IDeviceContext*           pContext,
                                                             const RTXPTRenderTargets& RenderTargets)
{
    const bool Executed = m_TemporalAAPass.CopyOutputToProcessed(m_Device, pContext, RenderTargets);
    m_Stats.LastRealtimeCopyExecuted = Executed;
    if (!Executed)
        DEV_ERROR("RTXPT realtime OutputColor copy failed: ", m_TemporalAAPass.GetStats().DisabledReason.c_str());
    return Executed;
}

bool RTXPTPostProcessPipeline::RunTemporalAA(IDeviceContext*                  pContext,
                                             const RTXPTRenderTargets&        RenderTargets,
                                             const SampleConstants&           FrameConstants,
                                             Uint32                           FrameIndex,
                                             bool                             ResetHistory,
                                             bool                             PreviousViewValid,
                                             const RTXPTTemporalAASettings&   Settings)
{
    RTXPTTemporalAAFrameAttribs Attribs;
    Attribs.pDevice           = m_Device;
    Attribs.pDeviceContext    = pContext;
    Attribs.pRenderTargets    = &RenderTargets;
    Attribs.pFrameConstants   = &FrameConstants;
    Attribs.Settings          = Settings;
    Attribs.FrameIndex        = FrameIndex;
    Attribs.ResetHistory      = ResetHistory;
    Attribs.PreviousViewValid = PreviousViewValid;

    const bool Executed          = m_TemporalAAPass.Execute(Attribs);
    m_Stats.TemporalAAStageReady = m_TemporalAAPass.IsReady();
    m_Stats.LastTemporalAAActive = Executed;
    if (!Executed)
        DEV_ERROR("RTXPT temporal AA failed: ", m_TemporalAAPass.GetStats().DisabledReason.c_str());
    return Executed;
}

bool RTXPTPostProcessPipeline::RunRealtimeSuperResolution(IDeviceContext*                      pContext,
                                                          const RTXPTRenderTargets&            RenderTargets,
                                                          const RTXPTSuperResolutionFrameDesc& FrameDesc,
                                                          float                                CameraNear,
                                                          float                                CameraFar,
                                                          float                                CameraFovAngleVert)
{
    const bool Executed =
        m_SuperResolutionPass.Execute(pContext,
                                      RenderTargets,
                                      FrameDesc,
                                      CameraNear,
                                      CameraFar,
                                      CameraFovAngleVert,
                                      RenderTargets.GetOutputColorSRV(),
                                      RenderTargets.GetProcessedOutputColorUAV());
    const auto& SRStats = m_SuperResolutionPass.GetStats();
    m_Stats.SuperResolutionStageReady = !FrameDesc.Enabled || (Executed && SRStats.UpscalerReady);
    m_Stats.LastSuperResolutionActive = Executed && SRStats.LastExecute && FrameDesc.Enabled;
    if (!Executed && SRStats.DisabledReason.empty())
        DEV_ERROR("RTXPT realtime super-resolution pass failed");
    return Executed;
}
```

- [ ] **Step 5: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 6: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
git commit -m "feat(rtxpt): expose realtime AA and SR handoff stages"
```

---

### Task 5: Resolve Realtime Dimensions and Camera Jitter Before PathTrace

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add current jitter storage**

In `RTXPTSample.hpp`, add after `m_CurrentSuperResolutionFrame`:

```cpp
    float2                        m_CurrentRealtimeCameraJitter = float2{0.0f, 0.0f};
```

- [ ] **Step 2: Add helper functions near existing AA helpers**

In the anonymous namespace in `RTXPTSample.cpp`, add:

```cpp
bool IsRealtimeSuperResolutionSelected(const RTXPTRealtimeSettings& RealtimeUI)
{
    return RealtimeUI.RealtimeMode && RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::SuperResolution;
}

RTXPTSuperResolutionSettings MakeRealtimeSuperResolutionSettings(const RTXPTRealtimeSettings& RealtimeUI)
{
    RTXPTSuperResolutionSettings Settings;
    Settings.Enabled = IsRealtimeSuperResolutionSelected(RealtimeUI);
    return Settings;
}

RTXPTTemporalAASettings MakeRealtimeTemporalAASettings()
{
    RTXPTTemporalAASettings Settings;
    Settings.TemporalStabilityFactor = 0.9375f;
    Settings.SkipRejection           = false;
    return Settings;
}

float2 NormalizeSuperResolutionJitterForCamera(const RTXPTSuperResolutionFrameDesc& FrameDesc)
{
    const float SafeWidth  = static_cast<float>(std::max(FrameDesc.Dimensions.RenderWidth, Uint32{1}));
    const float SafeHeight = static_cast<float>(std::max(FrameDesc.Dimensions.RenderHeight, Uint32{1}));
    return float2{
        FrameDesc.Jitter.x / (0.5f * SafeWidth),
        -FrameDesc.Jitter.y / (0.5f * SafeHeight)};
}
```

Keep `TODO(RTXPT-Realtime-DLSS-RR)` out of these helpers; DLSS-RR is handled at the selection gate.

- [ ] **Step 3: Resolve SR dimensions in `UpdateRenderTargetDimensions()`**

Replace the local disabled SR settings with realtime-aware settings:

```cpp
    const RTXPTSuperResolutionSettings SuperResolution =
        MakeRealtimeSuperResolutionSettings(m_RealtimeUI);

    m_CurrentSuperResolutionFrame =
        m_PostProcessPipeline.ResolveSuperResolutionFrameDesc(SuperResolution,
                                                              SCDesc.Width,
                                                              SCDesc.Height,
                                                              Formats.ProcessedOutputColor,
                                                              false,
                                                              TimeDeltaSeconds);
    m_CurrentTargetDimensions = m_CurrentSuperResolutionFrame.Dimensions;
```

Expected:

- `RealtimeAA == SuperResolution` may produce render-size dimensions smaller than display size.
- `RealtimeAA == Disabled` and `RealtimeAA == TAA` keep render size equal to display size.
- `RealtimeAA == DLSSRR` does not activate SR dimensions.

- [ ] **Step 4: Compute the camera jitter after reset flags are captured**

In `UpdateFrameConstants()`, replace:

```cpp
    const float2 CameraJitter = float2{0.0f, 0.0f};
```

with:

```cpp
    m_CurrentSuperResolutionFrame.ResetHistory =
        HasRealtimeResetFlag(m_CurrentFrameRealtimeReset, RTXPT_REALTIME_RESET_TAA_SR_HISTORY);

    float2 CameraJitter = float2{0.0f, 0.0f};
    if (RealtimeMode && m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::TAA)
    {
        CameraJitter =
            m_PostProcessPipeline.GetRealtimeTAAJitter(m_FrameIndex, RenderWidth, RenderHeight);
    }
    else if (RealtimeMode &&
             m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::SuperResolution &&
             m_CurrentSuperResolutionFrame.Enabled)
    {
        CameraJitter = NormalizeSuperResolutionJitterForCamera(m_CurrentSuperResolutionFrame);
    }
    m_CurrentRealtimeCameraJitter = CameraJitter;
```

Expected: the same jitter is fed to `MakePathTracerCameraData()` and `MakePathTracerViewData()` before realtime path tracing writes depth and motion vectors.

- [ ] **Step 5: Upload SR activity to frame constants**

Replace:

```cpp
    PtConsts.superResolutionActive    = 0u;
```

with:

```cpp
    PtConsts.superResolutionActive =
        RealtimeMode && m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::SuperResolution &&
            m_CurrentSuperResolutionFrame.Enabled ?
        1u :
        0u;
```

- [ ] **Step 6: Preserve DLSS-RR as disabled**

Leave this existing realtime jitter scale branch unchanged:

```cpp
    PtConsts.perPixelJitterAAScale =
        RealtimeMode ?
        (m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR ? m_RealtimeUI.DLSSRRMicroJitter : 0.0f) :
        1.0f;
```

Rationale: `RealtimeAA == 3` remains a reserved constant-only path. TAA/SR camera jitter flows through `PathTracerCameraData::Jitter` and `PathTracerViewData::PixelOffset`.

- [ ] **Step 7: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 8: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): resolve realtime AA jitter and SR dimensions"
```

---

### Task 6: Route `PresentRealtimeFinalOutput()` by Realtime AA Mode

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Replace the disabled-SR branch**

In `RTXPTSample::PresentRealtimeFinalOutput()`, replace the initial `DisabledSuperResolution` frame-desc block with a switch:

```cpp
    const bool ResetTaaSrHistory =
        HasRealtimeResetFlag(m_CurrentFrameRealtimeReset, RTXPT_REALTIME_RESET_TAA_SR_HISTORY);

    switch (m_RealtimeUI.RealtimeAA)
    {
        case RTXPTRealtimeAAMode::Disabled:
        {
            if (!m_PostProcessPipeline.CopyRealtimeOutputToProcessed(m_pImmediateContext, m_RenderTargets))
            {
                RecordRealtimePathTraceStatus("Realtime OutputColor copy to ProcessedOutputColor failed");
                return false;
            }
            break;
        }

        case RTXPTRealtimeAAMode::TAA:
        {
            if (!m_PostProcessPipeline.RunTemporalAA(m_pImmediateContext,
                                                     m_RenderTargets,
                                                     m_LastFrameConstants,
                                                     m_FrameIndex,
                                                     ResetTaaSrHistory,
                                                     m_HasPreviousFrameConstants,
                                                     MakeRealtimeTemporalAASettings()))
            {
                const auto& TAAStats = m_PostProcessPipeline.GetTemporalAAPass().GetStats();
                RecordRealtimePathTraceStatus(
                    TAAStats.DisabledReason.empty() ? "Realtime TAA failed" : TAAStats.DisabledReason.c_str());
                return false;
            }
            break;
        }

        case RTXPTRealtimeAAMode::SuperResolution:
        {
            if (!m_CurrentSuperResolutionFrame.Enabled)
            {
                const auto& SRStats = m_PostProcessPipeline.GetSuperResolutionPass().GetStats();
                RecordRealtimePathTraceStatus(
                    SRStats.DisabledReason.empty() ? "Realtime super resolution is unavailable" : SRStats.DisabledReason.c_str());
                return false;
            }

            if (!m_PostProcessPipeline.RunRealtimeSuperResolution(m_pImmediateContext,
                                                                  m_RenderTargets,
                                                                  m_CurrentSuperResolutionFrame,
                                                                  m_CameraNearPlane,
                                                                  m_CameraFarPlane,
                                                                  m_CameraVerticalFov))
            {
                RecordRealtimePathTraceStatus("Realtime super-resolution pass failed");
                return false;
            }
            break;
        }

        case RTXPTRealtimeAAMode::DLSSRR:
        {
            RecordRealtimePathTraceStatus("DLSS-RR is unavailable in this phase; TODO(RTXPT-Realtime-DLSS-RR).");
            return false;
        }

        default:
            RecordRealtimePathTraceStatus("Unknown realtime AA/SR mode");
            return false;
    }
```

Keep the existing bloom, tone mapping, presentation blit, and failure handling after this switch.

- [ ] **Step 2: Ensure `ProcessedOutputColor` is the only HDR post-process input**

Run:

```powershell
rg -n "RunPreToneMapping|RunToneMapping|GetProcessedOutputColorSRV|GetOutputColorSRV" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
```

Expected:

- `RunPreToneMapping()` and `RunToneMapping()` still read `ProcessedOutputColor`.
- `PresentRealtimeFinalOutput()` uses `OutputColor` only through the AA/SR handoff methods.

- [ ] **Step 3: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): route realtime output through AA SR handoff"
```

---

### Task 7: Enable UI Selection and Status Without DLSS-RR Execution

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Replace compile-time availability constants**

At the top of `RTXPTSample.cpp`, replace:

```cpp
constexpr bool        kRTXPTRealtimeTaaAvailable = false;
constexpr bool        kRTXPTRealtimeSrAvailable  = false;
constexpr bool        kRTXPTDlssRrAvailable      = false;
```

with:

```cpp
constexpr bool        kRTXPTDlssRrAvailable = false;
```

TAA availability comes from `m_PostProcessPipeline.GetTemporalAAPass().IsReady()`. SR availability comes from `m_PostProcessPipeline.GetSuperResolutionPass().HasTemporalVariant()`.

- [ ] **Step 2: Update the AA/SR combo enable logic**

Inside the AA/SR combo loop, add local runtime availability before the loop:

```cpp
                const bool TaaAvailable = m_PostProcessPipeline.GetTemporalAAPass().IsReady();
                const bool SrAvailable  = m_PostProcessPipeline.GetSuperResolutionPass().HasTemporalVariant();
```

Replace the `Enabled` expression with:

```cpp
                        const bool Enabled =
                            Item == static_cast<int>(RTXPTRealtimeAAMode::Disabled) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::TAA) && TaaAvailable) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::SuperResolution) && SrAvailable) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::DLSSRR) && kRTXPTDlssRrAvailable);
```

- [ ] **Step 3: Add disabled tooltips for TAA/SR/DLSS-RR**

Inside the loop, after `ImGui::EndDisabled()`, add:

```cpp
                        if (ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenDisabled) && !Enabled)
                        {
                            if (Item == static_cast<int>(RTXPTRealtimeAAMode::TAA))
                                ImGui::SetTooltip("DiligentFX TemporalAntiAliasing is not initialized.");
                            else if (Item == static_cast<int>(RTXPTRealtimeAAMode::SuperResolution))
                            {
                                const auto& SRStats = m_PostProcessPipeline.GetSuperResolutionPass().GetStats();
                                ImGui::SetTooltip("%s",
                                                  SRStats.DisabledReason.empty() ?
                                                      "No temporal Diligent super-resolution variant is available." :
                                                      SRStats.DisabledReason.c_str());
                            }
                            else if (Item == static_cast<int>(RTXPTRealtimeAAMode::DLSSRR))
                                ImGui::SetTooltip("DLSS-RR is reserved by TODO(RTXPT-Realtime-DLSS-RR).");
                        }
```

- [ ] **Step 4: Update debug/status panel**

Replace the existing "Super resolution" status line with:

```cpp
        const auto& TAAStats = m_PostProcessPipeline.GetTemporalAAPass().GetStats();
        const auto& SRStats  = m_PostProcessPipeline.GetSuperResolutionPass().GetStats();
        ImGui::Text("Realtime TAA: %s, executed=%s",
                    TAAStats.Ready ? "ready" : "not ready",
                    TAAStats.LastExecute ? "yes" : "no");
        if (!TAAStats.DisabledReason.empty())
            ImGui::TextWrapped("Realtime TAA status: %s", TAAStats.DisabledReason.c_str());
        ImGui::Text("Super resolution: variants=%u, active=%s, executed=%s",
                    SRStats.VariantCount,
                    m_CurrentSuperResolutionFrame.Enabled ? "yes" : "no",
                    SRStats.LastExecute ? "yes" : "no");
        if (!SRStats.DisabledReason.empty())
            ImGui::TextWrapped("Super resolution status: %s", SRStats.DisabledReason.c_str());
```

- [ ] **Step 5: Source-scan DLSS-RR execution**

Run:

```powershell
rg -n "DLSSRR|EvaluateDLSSRR|Streamline|TODO\\(RTXPT-Realtime-DLSS-RR\\)" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected:

- `DLSSRR` appears in enum/status/disabled UI/constant reservation code.
- `TODO(RTXPT-Realtime-DLSS-RR)` appears only in disabled guards or reserved constants.
- `EvaluateDLSSRR` and Streamline resource tagging do not appear in the Diligent RTXPT sample.

- [ ] **Step 6: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 7: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): enable realtime TAA SR UI handoff modes"
```

---

### Task 8: Update Mapping and Run Final Verification

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add G10 mapping rows**

Append rows near the existing post-processing/realtime mapping section:

```markdown
| `Sample.cpp::PostProcessAA` `RealtimeAA == 0` | `src/RTXPTSample.cpp::PresentRealtimeFinalOutput` + `src/RTXPTPostProcessPipeline.cpp::CopyRealtimeOutputToProcessed` | Realtime G10 | Copies/passes merged realtime `OutputColor` into HDR `ProcessedOutputColor` before bloom/tone mapping. |
| `Sample.cpp::PostProcessAA` `RealtimeAA == 1` | `src/RTXPTTemporalAAPass.{hpp,cpp}` + `src/RTXPTPostProcessPipeline.cpp::RunTemporalAA` | Realtime G10 | Uses DiligentFX `TemporalAntiAliasing` with RTXPT depth, previous depth, motion vectors, current/previous camera state, and reset-history flags. |
| `Sample.cpp::PostProcessAA` `RealtimeAA == 2` | `src/RTXPTSuperResolutionPass.{hpp,cpp}` + `src/RTXPTPostProcessPipeline.cpp::RunRealtimeSuperResolution` | Realtime G10 | Uses Diligent `ISuperResolution` from merged realtime `OutputColor` to display-size `ProcessedOutputColor`. |
| `Sample.cpp::PostProcessAA` `RealtimeAA == 3` / `EvaluateDLSSRR` | Disabled UI/status guards with `TODO(RTXPT-Realtime-DLSS-RR)` | Deferred | DLSS-RR input preparation, Streamline resource tagging, and evaluation remain non-executing in this phase. |
```

- [ ] **Step 2: Link the plan from the spec if the project convention wants plan backlinks**

If the spec has an implementation-plan list, add:

```markdown
- G10 / Phase RT5 plan: `docs/superpowers/plans/2026-06-05-rtxpt-realtime-g10-aa-sr-handoff.md`
```

If the spec has no plan-link section, leave the spec unchanged.

- [ ] **Step 3: Run source scans**

Run:

```powershell
rg -n "RTXPTTemporalAAPass|RunTemporalAA|CopyRealtimeOutputToProcessed|RunRealtimeSuperResolution|PreviousDepth" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "pMergeOutputUAV = RenderTargets.GetOutputColorUAV|GetOutputColorUAV\\(\\)" DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
rg -n "EvaluateDLSSRR|Streamline|TODO\\(RTXPT-Realtime-DLSS-RR\\)" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected:

- New G10 wrapper and methods are present.
- Realtime merge binds `OutputColor`.
- DLSS-RR has no executing Streamline/evaluate path.

- [ ] **Step 4: Build RTXPT**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 5: Run format validation for touched sample C++ files**

Run from the DiligentSamples format-validation folder:

```powershell
Push-Location DiligentSamples\BuildTools\FormatValidation
.\validate_format_win.bat
Pop-Location
```

Expected: validation completes without formatting errors for touched RTXPT files.

- [ ] **Step 6: Run realtime smoke tests manually**

Run the RTXPT sample from the built sample output directory and verify these modes:

```text
Reference mode:
- Default reference path still renders.
- Accumulation and tone mapping continue to work.

Realtime mode, StandaloneDenoiser off:
- RealtimeAA Disabled renders through NoDenoiserFinalMerge -> OutputColor -> ProcessedOutputColor -> tone mapping -> blit.
- RealtimeAA TAA renders and the debug panel reports Realtime TAA executed=yes.
- RealtimeAA Super Resolution renders when a temporal SR variant is available; the debug panel reports SR active=yes and executed=yes.

Realtime mode, StandaloneDenoiser on with NRD available:
- REBLUR denoise runs before AA/SR handoff.
- RELAX denoise runs before AA/SR handoff.
- Toggling AA/SR mode resets TAA/SR history and does not force unrelated reference accumulation behavior.

RealtimeAA DLSS-RR:
- The combo entry remains disabled.
- Forced selection through debugging or config reports TODO(RTXPT-Realtime-DLSS-RR) and does not execute Streamline/DLSS-RR code.
```

- [ ] **Step 7: Commit mapping and final fixes**

```bash
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map realtime AA SR handoff"
```

If final verification required small code fixes, include those touched files in the same commit only when they directly belong to G10 verification.

---

## Acceptance Checklist

- [ ] `RealtimeAA == 0` always produces `ProcessedOutputColor` before HDR post-process and tone mapping.
- [ ] `RealtimeAA == 1` executes DiligentFX TAA from merged realtime `OutputColor`, uses current depth, previous depth, screen motion vectors, and current/previous camera state, then copies the accumulated TAA output to `ProcessedOutputColor`.
- [ ] `RealtimeAA == 2` uses `RTXPTSuperResolutionPass` / Diligent `ISuperResolution` where a temporal variant is available, with `OutputColor`, `Depth`, and `ScreenMotionVectors` as inputs and `ProcessedOutputColor` as output.
- [ ] `RealtimeAA == 3` is visibly unavailable and cannot execute DLSS-RR, Streamline tagging, or `EvaluateDLSSRR`.
- [ ] Camera jitter is selected before realtime path-trace dispatch for TAA/SR modes.
- [ ] Previous view state is invalidated on render-target recreate and TAA/SR history reset.
- [ ] Standalone NRD still runs before AA/SR handoff.
- [ ] Reference mode continues to use the existing reference accumulation/tone-mapping path and does not expose or execute realtime TAA/SR.
- [ ] `OutputColor` is realtime merged HDR radiance; `ProcessedOutputColor` is the post-AA/SR HDR input to bloom/tone mapping.
- [ ] Build and format validation results are reported with exact commands.

## Self-Review

- Spec coverage: G10 bullets map to Tasks 3, 4, 5, 6, and 7. The success criteria map to Tasks 5, 6, 7, and 8.
- Placeholder scan: The only `TODO` string in this plan is the required `TODO(RTXPT-Realtime-DLSS-RR)` marker for the explicitly deferred DLSS-RR path.
- Type consistency: `RTXPTTemporalAAPass`, `RTXPTTemporalAASettings`, `RTXPTTemporalAAFrameAttribs`, `GetPreviousDepthSRV`, `GetPreviousDepthRTV`, `CopyRealtimeOutputToProcessed`, `RunTemporalAA`, and `RunRealtimeSuperResolution` are introduced before later tasks use them.
