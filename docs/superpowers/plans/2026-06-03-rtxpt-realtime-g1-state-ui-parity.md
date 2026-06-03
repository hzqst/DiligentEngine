# RTXPT Realtime G1 State and UI Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the RTXPT-fork realtime settings model and UI controls to `DiligentSamples/Samples/RTXPT` while keeping Reference mode as the default executing path.

**Architecture:** Introduce a Diligent-local realtime settings header that mirrors the RTXPT-fork `SampleUIData` controls needed by the realtime `PathTrace` and standalone `Denoise` flow, without including NRD headers or executing unavailable DLSS-RR paths. `RTXPTSample` owns the settings, UI rendering, reset request bits, and a visible realtime-path-disabled status until G2-G10 add constants, resources, tracing, denoising, and AA/SR execution.

**Tech Stack:** C++17, ImGui, DiligentSamples RTXPT sample lifecycle, Diligent post-process and super-resolution wrappers, CMake sample source registration, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`, goal `G1 - Realtime State and UI Parity`.
- Current Diligent UI state lives in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` as `RTXPTReferenceUIState`.
- Current `RTXPTSample::UpdateUI()` shows a disabled `Mode` combo with the tooltip `Realtime mode is out of scope...`.
- Current render path is always:

```text
RTXPTRayTracingPass::Trace
  -> RTXPTPostProcessPipeline::RunAccumulation
  -> RTXPTPostProcessPipeline::RunPreToneMapping
  -> RTXPTPostProcessPipeline::RunToneMapping
  -> RTXPTBlitPass
```

- Current `RTXPTRenderTargets` already has Phase 6 depth, screen motion vectors, temporal feedback, and `RTXPTSuperResolutionPass`, but Reference mode keeps super-resolution disabled.
- Current Diligent `PathTracer/Config.h` does not define stable-plane constants. G1 uses local UI constants matching RTXPT-fork: `cStablePlaneCount = 3`, `cStablePlaneMaxVertexIndex = 15`. G4 will later move shader-side constants into the algorithm layer.
- Original defaults come from:
  - `D:/RTXPT-fork/Rtxpt/SampleUI.h:122-126`, `165-175`, `212-217`, `278-299`
  - `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:781-929`, `1335-1483`
  - `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp:15-64`

## Scope Boundaries

- G1 does not extend `SampleConstants` or `PathTracerConstants`; that is G2.
- G1 does not allocate stable-plane or denoiser render targets; that is G3.
- G1 does not create ray-tracing variants, stable-plane shaders, NRD integration, denoiser prepare/merge passes, or realtime AA/SR execution; those are G4-G10.
- G1 must not silently run the Reference raygen when `RealtimeMode` is selected. It should show a visible disabled path until the realtime renderer exists.
- `RealtimeAA == 3` / DLSS-RR must remain a disabled UI/status path with the exact deferred marker `TODO(RTXPT-Realtime-DLSS-RR)` where a source marker is needed.
- NRD headers are not included in G1. The local NRD UI settings must compile without the NRD SDK and will be converted to real NRD settings in G8.
- Reference mode remains the startup default and must keep the current accumulation, tone mapping, bloom, presentation, and camera behavior.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
  - Owns realtime UI enums, stable-plane UI constants, NRD UI settings mirrors, reset flag helpers, default values, `ActualUseStandaloneDenoiser()`, `ActualSamplesPerPixel()`, and `SanitizeRealtimeSettings()`.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Registers `src/RTXPTRealtimeSettings.hpp` in the `INCLUDE` list.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Includes `RTXPTRealtimeSettings.hpp`, stores `m_RealtimeUI`, reset pending/current flags, and declares realtime reset helpers.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Replaces disabled mode UI with live Reference/Realtime state selection.
  - Adds realtime setup, firefly, AA/SR, standalone denoiser, stable-plane, NRD, DLSS-RR disabled-marker, and status/debug UI.
  - Records reset request bits with narrow invalidation semantics.
  - Adds a visible realtime-path-disabled render gate.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  - Maps RTXPT-fork `SampleUI` realtime controls to the new Diligent owners.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/SampleUI.h`
- Verify: `D:/RTXPT-fork/Rtxpt/SampleUI.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp`

- [ ] **Step 1: Confirm dirty files before editing**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files are preserved. At planning time, the spec file is untracked:

```text
?? docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md
```

- [ ] **Step 2: Confirm current Diligent G1 anchors**

Run:

```powershell
rg -n "RealtimeMode|RealtimeAA|StandaloneDenoiser|StablePlanes|RequestAccumulationReset|Mode\"|Super resolution" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp
```

Expected: current source has no live realtime state, the `Mode` combo is disabled, and `RTXPTSuperResolutionPass` settings already exist.

- [ ] **Step 3: Confirm RTXPT-fork source defaults**

Run:

```powershell
rg -n "ActualUseStandaloneDenoiser|RealtimeMode|RealtimeSamplesPerPixel|RealtimeAA|StablePlanesActiveCount|DenoiserRadianceClampK|NRDMethod|RealtimeFireflyFilterThreshold" D:/RTXPT-fork/Rtxpt/SampleUI.h D:/RTXPT-fork/Rtxpt/SampleUI.cpp D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp
```

Expected: source anchors resolve to `SampleUI.h`, `SampleUI.cpp`, and `NrdConfig.cpp`.

- [ ] **Step 4: Confirm this phase does not touch frame constants**

Run:

```powershell
rg -n "sampleBaseIndex|invSubSampleCount|denoisingEnabled|_activeStablePlaneCount|genericTSPlaneStride" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
```

Expected: no matches in Diligent source before G2. Do not add these fields in G1.

- [ ] **Step 5: No commit for preflight**

No source changes are made in Task 0. Do not create a commit for this task.

### Task 1: Add Realtime Settings Contract

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.h:122-126`, `165-175`, `212-217`, `278-299`
- Read: `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp:15-64`

- [ ] **Step 1: Create the realtime settings header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp` with:

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

#include <algorithm>

#include "BasicTypes.h"

namespace Diligent
{

constexpr Uint32 kRTXPTStablePlaneCount          = 3;
constexpr Uint32 kRTXPTStablePlaneMaxVertexIndex = 15;
constexpr Uint32 kRTXPTRealtimeSamplesPerPixelMax = 64;

enum class RTXPTRealtimeAAMode : Uint32
{
    Disabled        = 0,
    TAA             = 1,
    SuperResolution = 2,
    DLSSRR          = 3
};

enum class RTXPTNrdMethod : Uint32
{
    REBLUR = 0,
    RELAX  = 1
};

enum class RTXPTNrdHitDistanceReconstructionMode : Uint32
{
    Off     = 0,
    Area3x3 = 1,
    Area5x5 = 2
};

enum RTXPTRealtimeResetFlags : Uint32
{
    RTXPT_REALTIME_RESET_NONE                    = 0u,
    RTXPT_REALTIME_RESET_ACCUMULATION            = 1u << 0u,
    RTXPT_REALTIME_RESET_REALTIME_CACHES         = 1u << 1u,
    RTXPT_REALTIME_RESET_NRD_HISTORY             = 1u << 2u,
    RTXPT_REALTIME_RESET_TAA_SR_HISTORY          = 1u << 3u,
    RTXPT_REALTIME_RESET_RENDER_TARGET_RECREATE  = 1u << 4u,
};

inline RTXPTRealtimeResetFlags operator|(RTXPTRealtimeResetFlags LHS, RTXPTRealtimeResetFlags RHS)
{
    return static_cast<RTXPTRealtimeResetFlags>(static_cast<Uint32>(LHS) | static_cast<Uint32>(RHS));
}

inline RTXPTRealtimeResetFlags operator&(RTXPTRealtimeResetFlags LHS, RTXPTRealtimeResetFlags RHS)
{
    return static_cast<RTXPTRealtimeResetFlags>(static_cast<Uint32>(LHS) & static_cast<Uint32>(RHS));
}

inline RTXPTRealtimeResetFlags& operator|=(RTXPTRealtimeResetFlags& LHS, RTXPTRealtimeResetFlags RHS)
{
    LHS = LHS | RHS;
    return LHS;
}

inline bool HasRealtimeResetFlag(RTXPTRealtimeResetFlags Flags, RTXPTRealtimeResetFlags Test)
{
    return (static_cast<Uint32>(Flags & Test) != 0u);
}

struct RTXPTNrdReblurUiSettings
{
    bool EnableAntiFirefly = true;
    RTXPTNrdHitDistanceReconstructionMode HitDistanceReconstructionMode = RTXPTNrdHitDistanceReconstructionMode::Area5x5;
    Uint32 MaxAccumulatedFrameNum     = 50;
    Uint32 MaxFastAccumulatedFrameNum = 0;
    Uint32 HistoryFixFrameNum         = 0;
    float  DiffusePrepassBlurRadius   = 15.0f;
    float  SpecularPrepassBlurRadius  = 40.0f;
};

struct RTXPTNrdRelaxUiSettings
{
    bool EnableAntiFirefly = true;
    RTXPTNrdHitDistanceReconstructionMode HitDistanceReconstructionMode = RTXPTNrdHitDistanceReconstructionMode::Off;
    float  DiffusePrepassBlurRadius              = 0.0f;
    float  SpecularPrepassBlurRadius             = 0.0f;
    Uint32 DiffuseMaxAccumulatedFrameNum         = 25;
    Uint32 SpecularMaxAccumulatedFrameNum        = 40;
    Uint32 DiffuseMaxFastAccumulatedFrameNum     = 5;
    Uint32 SpecularMaxFastAccumulatedFrameNum    = 6;
    Uint32 HistoryFixFrameNum                    = 0;
    Uint32 AtrousIterationNum                    = 5;
    float  LobeAngleFraction                     = 0.7f;
    float  SpecularLobeAngleSlack                = 0.2f;
    float  DepthThreshold                        = 0.004f;
    float  AntilagAccelerationAmount             = 0.55f;
    float  AntilagSpatialSigmaScale              = 2.5f;
    float  AntilagTemporalSigmaScale             = 0.3f;
    float  AntilagResetAmount                    = 0.5f;
};

struct RTXPTRealtimeSettings
{
    bool                RealtimeMode            = false;
    Int32               RealtimeSamplesPerPixel = 1;
    RTXPTRealtimeAAMode RealtimeAA              = RTXPTRealtimeAAMode::Disabled;
    bool                StandaloneDenoiser      = false;

    bool  RealtimeFireflyFilterEnabled   = true;
    float RealtimeFireflyFilterThreshold = 0.10f;
    float TexLODBias                     = -1.0f;

    Int32 StablePlanesActiveCount       = static_cast<Int32>(kRTXPTStablePlaneCount);
    Int32 StablePlanesMaxVertexDepth    = 9;
    bool  AllowPrimarySurfaceReplacement = true;
    float StablePlanesSplitStopThreshold = 0.95f;
    bool  StablePlanesSuppressPrimaryIndirectSpecular = true;
    float StablePlanesSuppressPrimaryIndirectSpecularK = 0.6f;
    float StablePlanesAntiAliasingFallthrough          = 0.6f;

    float DenoiserRadianceClampK = 8.0f;

    RTXPTNrdMethod          NRDMethod = RTXPTNrdMethod::REBLUR;
    float                   NRDDisocclusionThreshold = 0.03f;
    bool                    NRDUseAlternateDisocclusionThresholdMix = true;
    float                   NRDDisocclusionThresholdAlternate = 0.2f;
    RTXPTNrdReblurUiSettings ReblurSettings;
    RTXPTNrdRelaxUiSettings  RelaxSettings;

    float DLSSRRBrightnessClampK = 4096.0f;
    float DLSSRRMicroJitter      = 0.1f;

    bool ActualUseStandaloneDenoiser() const
    {
        return RealtimeMode &&
            static_cast<Uint32>(RealtimeAA) < static_cast<Uint32>(RTXPTRealtimeAAMode::DLSSRR) &&
            StandaloneDenoiser;
    }

    Uint32 ActualSamplesPerPixel() const
    {
        return RealtimeMode ? static_cast<Uint32>(std::max(RealtimeSamplesPerPixel, 1)) : 1u;
    }
};

inline void SanitizeRealtimeSettings(RTXPTRealtimeSettings& Settings)
{
    Settings.RealtimeSamplesPerPixel =
        std::clamp(Settings.RealtimeSamplesPerPixel, Int32{1}, static_cast<Int32>(kRTXPTRealtimeSamplesPerPixelMax));

    const Uint32 AAMode = std::clamp(static_cast<Uint32>(Settings.RealtimeAA),
                                     static_cast<Uint32>(RTXPTRealtimeAAMode::Disabled),
                                     static_cast<Uint32>(RTXPTRealtimeAAMode::DLSSRR));
    Settings.RealtimeAA = static_cast<RTXPTRealtimeAAMode>(AAMode);

    Settings.RealtimeFireflyFilterThreshold = std::clamp(Settings.RealtimeFireflyFilterThreshold, 0.00001f, 1000.0f);
    Settings.StablePlanesActiveCount =
        std::clamp(Settings.StablePlanesActiveCount, Int32{1}, static_cast<Int32>(kRTXPTStablePlaneCount));
    Settings.StablePlanesMaxVertexDepth =
        std::clamp(Settings.StablePlanesMaxVertexDepth, Int32{2}, static_cast<Int32>(kRTXPTStablePlaneMaxVertexIndex));
    Settings.StablePlanesSplitStopThreshold = std::clamp(Settings.StablePlanesSplitStopThreshold, 0.0f, 2.0f);
    Settings.StablePlanesSuppressPrimaryIndirectSpecularK =
        std::clamp(Settings.StablePlanesSuppressPrimaryIndirectSpecularK, 0.0f, 1.0f);
    Settings.StablePlanesAntiAliasingFallthrough =
        std::clamp(Settings.StablePlanesAntiAliasingFallthrough, 0.0f, 1.0f);

    Settings.DenoiserRadianceClampK = std::max(Settings.DenoiserRadianceClampK, 0.0f);
    Settings.NRDDisocclusionThreshold = std::max(Settings.NRDDisocclusionThreshold, 0.0f);
    Settings.NRDDisocclusionThresholdAlternate = std::max(Settings.NRDDisocclusionThresholdAlternate, 0.0f);
    Settings.DLSSRRBrightnessClampK = std::max(Settings.DLSSRRBrightnessClampK, 0.0f);
    Settings.DLSSRRMicroJitter = std::clamp(Settings.DLSSRRMicroJitter, 0.0f, 1.0f);

    Settings.ReblurSettings.MaxAccumulatedFrameNum = std::min(Settings.ReblurSettings.MaxAccumulatedFrameNum, 500u);
    Settings.ReblurSettings.MaxFastAccumulatedFrameNum = std::min(Settings.ReblurSettings.MaxFastAccumulatedFrameNum, 500u);
    Settings.ReblurSettings.HistoryFixFrameNum = std::min(Settings.ReblurSettings.HistoryFixFrameNum, 500u);
    Settings.ReblurSettings.DiffusePrepassBlurRadius = std::clamp(Settings.ReblurSettings.DiffusePrepassBlurRadius, 0.0f, 100.0f);
    Settings.ReblurSettings.SpecularPrepassBlurRadius = std::clamp(Settings.ReblurSettings.SpecularPrepassBlurRadius, 0.0f, 100.0f);

    Settings.RelaxSettings.DiffusePrepassBlurRadius = std::clamp(Settings.RelaxSettings.DiffusePrepassBlurRadius, 0.0f, 100.0f);
    Settings.RelaxSettings.SpecularPrepassBlurRadius = std::clamp(Settings.RelaxSettings.SpecularPrepassBlurRadius, 0.0f, 100.0f);
    Settings.RelaxSettings.AtrousIterationNum = std::clamp(Settings.RelaxSettings.AtrousIterationNum, 2u, 8u);
    Settings.RelaxSettings.LobeAngleFraction = std::clamp(Settings.RelaxSettings.LobeAngleFraction, 0.0f, 1.0f);
    Settings.RelaxSettings.SpecularLobeAngleSlack = std::clamp(Settings.RelaxSettings.SpecularLobeAngleSlack, 0.0f, 1.0f);
    Settings.RelaxSettings.DepthThreshold = std::clamp(Settings.RelaxSettings.DepthThreshold, 0.0f, 0.1f);
    Settings.RelaxSettings.AntilagAccelerationAmount = std::clamp(Settings.RelaxSettings.AntilagAccelerationAmount, 0.0f, 1.0f);
    Settings.RelaxSettings.AntilagSpatialSigmaScale = std::clamp(Settings.RelaxSettings.AntilagSpatialSigmaScale, 0.0f, 5.0f);
    Settings.RelaxSettings.AntilagTemporalSigmaScale = std::clamp(Settings.RelaxSettings.AntilagTemporalSigmaScale, 0.0f, 5.0f);
    Settings.RelaxSettings.AntilagResetAmount = std::clamp(Settings.RelaxSettings.AntilagResetAmount, 0.0f, 1.0f);
}

} // namespace Diligent
```

- [ ] **Step 2: Register the header in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new header after `src/RTXPTFrameConstants.hpp` in the `INCLUDE` list:

```cmake
    src/RTXPTFrameConstants.hpp
    src/RTXPTRealtimeSettings.hpp
    src/RTXPTEnvMapBaker.hpp
```

- [ ] **Step 3: Compile-check the new header in isolation through the sample target**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: CMake sees `RTXPTRealtimeSettings.hpp`; no NRD include or symbol is required by G1.

- [ ] **Step 4: Commit Task 1**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add realtime settings contract" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the implementation workflow requires frequent local commits. If the user did not ask for commits, leave the changes staged or unstaged according to the active workflow.

### Task 2: Wire Realtime State and Reset Requests Into RTXPTSample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.h:122-126`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp:1402-1460`, `1573-1582`, `2186`

- [ ] **Step 1: Include the settings header and declare reset helpers**

In `RTXPTSample.hpp`, add:

```cpp
#include "RTXPTRealtimeSettings.hpp"
```

after:

```cpp
#include "RTXPTFrameConstants.hpp"
```

Add these private methods after `RequestAccumulationReset()`:

```cpp
    void RequestRealtimeReset(RTXPTRealtimeResetFlags Flags, const char* Reason);
    void RequestRealtimeCachesReset(const char* Reason);
    void BeginRealtimeFrameResetScope();
```

Add these fields after `RTXPTReferenceUIState m_ReferenceUI;`:

```cpp
    RTXPTRealtimeSettings  m_RealtimeUI;
    RTXPTRealtimeResetFlags m_RealtimeResetPending = RTXPT_REALTIME_RESET_REALTIME_CACHES |
        RTXPT_REALTIME_RESET_NRD_HISTORY |
        RTXPT_REALTIME_RESET_TAA_SR_HISTORY;
    RTXPTRealtimeResetFlags m_CurrentFrameRealtimeReset = RTXPT_REALTIME_RESET_NONE;
```

- [ ] **Step 2: Add local UI/status helpers in `RTXPTSample.cpp`**

Inside the anonymous namespace in `RTXPTSample.cpp`, after the `static_assert`s for tone mapping UI, add:

```cpp
constexpr bool        kRTXPTStandaloneNrdAvailable = false;
constexpr bool        kRTXPTRealtimeTaaAvailable   = false;
constexpr bool        kRTXPTRealtimeSrAvailable    = false;
constexpr bool        kRTXPTDlssRrAvailable        = false;
constexpr const char* kRTXPTRealtimeDisabledReason = "Realtime PathTrace/Denoise execution starts in G2-G10.";
constexpr const char* kRTXPTNrdDisabledReason      = "Standalone denoiser disabled: NRD integration starts in G8.";

const char* GetRealtimeAAModeName(RTXPTRealtimeAAMode Mode)
{
    switch (Mode)
    {
        case RTXPTRealtimeAAMode::Disabled: return "Disabled";
        case RTXPTRealtimeAAMode::TAA: return "TAA";
        case RTXPTRealtimeAAMode::SuperResolution: return "Super Resolution";
        case RTXPTRealtimeAAMode::DLSSRR: return "DLSS-RR";
        default: return "Unknown";
    }
}

const char* GetNrdMethodName(RTXPTNrdMethod Method)
{
    switch (Method)
    {
        case RTXPTNrdMethod::REBLUR: return "REBLUR";
        case RTXPTNrdMethod::RELAX: return "RELAX";
        default: return "Unknown";
    }
}

void DrawDisabledTooltip(const char* Text)
{
    if (ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenDisabled))
        ImGui::SetTooltip("%s", Text);
}
```

- [ ] **Step 3: Implement reset helpers**

Add these methods near `RequestAccumulationReset()` in `RTXPTSample.cpp`:

```cpp
void RTXPTSample::RequestRealtimeReset(RTXPTRealtimeResetFlags Flags, const char* Reason)
{
    if (HasRealtimeResetFlag(Flags, RTXPT_REALTIME_RESET_ACCUMULATION))
        RequestAccumulationReset(Reason);

    m_RealtimeResetPending |= Flags;

    if (HasRealtimeResetFlag(Flags, RTXPT_REALTIME_RESET_REALTIME_CACHES))
        m_LightsBaker.RequestFeedbackReset();
}

void RTXPTSample::RequestRealtimeCachesReset(const char* Reason)
{
    RequestRealtimeReset(RTXPT_REALTIME_RESET_REALTIME_CACHES |
                             RTXPT_REALTIME_RESET_NRD_HISTORY |
                             RTXPT_REALTIME_RESET_TAA_SR_HISTORY,
                         Reason);
}

void RTXPTSample::BeginRealtimeFrameResetScope()
{
    m_CurrentFrameRealtimeReset = m_RealtimeResetPending;
    m_RealtimeResetPending      = RTXPT_REALTIME_RESET_NONE;
}
```

- [ ] **Step 4: Snapshot reset flags once per frame**

In `RTXPTSample::UpdateFrameConstants()`, add this at the start of the function body, after local render/display dimensions are computed:

```cpp
    SanitizeRealtimeSettings(m_RealtimeUI);
    BeginRealtimeFrameResetScope();
```

The surrounding block becomes:

```cpp
void RTXPTSample::UpdateFrameConstants(double CurrTime)
{
    const Uint32 RenderWidth   = m_CurrentTargetDimensions.RenderWidth;
    const Uint32 RenderHeight  = m_CurrentTargetDimensions.RenderHeight;
    const Uint32 DisplayWidth  = m_CurrentTargetDimensions.DisplayWidth;
    const Uint32 DisplayHeight = m_CurrentTargetDimensions.DisplayHeight;
    const float  Width         = static_cast<float>(RenderWidth);
    const float  Height        = static_cast<float>(RenderHeight);

    SanitizeRealtimeSettings(m_RealtimeUI);
    BeginRealtimeFrameResetScope();
```

- [ ] **Step 5: Route existing accumulation reset through realtime cache reset only when appropriate**

Keep `RequestAccumulationReset()` as the Reference-history reset owner, but append realtime-cache invalidation when Reference mode is active, matching RTXPT-fork determinism behavior:

```cpp
void RTXPTSample::RequestAccumulationReset(const char* /*Reason*/)
{
    m_AccumulationFrame        = 0;
    m_ResetAccumulationPending = true;
    m_LightsBaker.RequestFeedbackReset();

    if (!m_RealtimeUI.RealtimeMode)
    {
        m_RealtimeResetPending |= RTXPT_REALTIME_RESET_REALTIME_CACHES |
            RTXPT_REALTIME_RESET_NRD_HISTORY |
            RTXPT_REALTIME_RESET_TAA_SR_HISTORY;
    }
}
```

- [ ] **Step 6: Build-check reset wiring**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the sample target compiles; no renderer behavior has changed while `RealtimeMode == false`.

- [ ] **Step 7: Commit Task 2**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): track realtime reset requests" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 3: Replace Mode Placeholder and Add Realtime Setup Controls

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:781-856`, `873-929`

- [ ] **Step 1: Add realtime reset lambdas in `UpdateUI()`**

In `RTXPTSample::UpdateUI()`, after `ResetLightsBakerOnChange`, add:

```cpp
    auto ResetRealtimeOnChange = [this](bool Changed, const char* Reason) -> bool {
        if (Changed)
            RequestRealtimeCachesReset(Reason);
        return Changed;
    };
    auto ResetTaaSrOnChange = [this](bool Changed, const char* Reason) -> bool {
        if (Changed)
            RequestRealtimeReset(RTXPT_REALTIME_RESET_TAA_SR_HISTORY, Reason);
        return Changed;
    };
    auto ResetRenderTargetsOnChange = [this](bool Changed, const char* Reason) -> bool {
        if (Changed)
            RequestRealtimeReset(RTXPT_REALTIME_RESET_RENDER_TARGET_RECREATE |
                                     RTXPT_REALTIME_RESET_REALTIME_CACHES |
                                     RTXPT_REALTIME_RESET_NRD_HISTORY |
                                     RTXPT_REALTIME_RESET_TAA_SR_HISTORY,
                                 Reason);
        return Changed;
    };
```

- [ ] **Step 2: Replace the disabled `Mode` combo**

Replace the current mode block:

```cpp
        // Mode (Reference only; Realtime track is out of scope).
        {
            int ModeIndex = 0;
            ImGui::BeginDisabled(true);
            ImGui::Combo("Mode", &ModeIndex, "Reference\0Realtime\0\0");
            ImGui::EndDisabled();
            PlaceholderTooltip("Realtime mode is out of scope for the reference path tracer (umbrella Phase 5.5+).");
        }
```

with:

```cpp
        {
            int ModeIndex = m_RealtimeUI.RealtimeMode ? 1 : 0;
            if (ImGui::Combo("Mode", &ModeIndex, "Reference\0Realtime\0\0"))
            {
                const bool NewRealtimeMode = ModeIndex != 0;
                if (m_RealtimeUI.RealtimeMode != NewRealtimeMode)
                {
                    m_RealtimeUI.RealtimeMode = NewRealtimeMode;
                    RequestRealtimeReset(RTXPT_REALTIME_RESET_ACCUMULATION |
                                             RTXPT_REALTIME_RESET_REALTIME_CACHES |
                                             RTXPT_REALTIME_RESET_NRD_HISTORY |
                                             RTXPT_REALTIME_RESET_TAA_SR_HISTORY,
                                         "Path-tracer mode changed");
                }
            }
            if (m_RealtimeUI.RealtimeMode && ImGui::IsItemHovered())
                ImGui::SetTooltip("%s", kRTXPTRealtimeDisabledReason);
        }
```

- [ ] **Step 3: Split setup UI into realtime and reference controls**

Inside the existing `Setup:` block, replace the reset/sample-count portion:

```cpp
            if (ImGui::Button("Reset##REFMACC"))
                RequestAccumulationReset("User reset");
            ImGui::SameLine();
            ImGui::Text("Accumulated samples: %u", m_AccumulationFrame);
```

with:

```cpp
            if (m_RealtimeUI.RealtimeMode)
            {
                if (ImGui::Button("Reset##RTMACC"))
                    RequestRealtimeCachesReset("User reset realtime caches");
                if (ImGui::IsItemHovered())
                    ImGui::SetTooltip("Reset realtime temporal caches, NRD history, TAA/SR history, and lighting feedback.");
                ImGui::SameLine();

                int RealtimeSPP = m_RealtimeUI.RealtimeSamplesPerPixel;
                if (ResetRealtimeOnChange(ImGui::InputInt("Samples per pixel", &RealtimeSPP), "Realtime samples-per-pixel changed"))
                    m_RealtimeUI.RealtimeSamplesPerPixel = RealtimeSPP;
                m_RealtimeUI.RealtimeSamplesPerPixel =
                    std::clamp(m_RealtimeUI.RealtimeSamplesPerPixel, Int32{1}, static_cast<Int32>(kRTXPTRealtimeSamplesPerPixelMax));
                if (ImGui::IsItemHovered())
                    ImGui::SetTooltip("Full paths per pixel per realtime frame; camera ray anti-aliasing is handled by the selected AA/SR mode.");
            }
            else
            {
                if (ImGui::Button("Reset##REFMACC"))
                    RequestAccumulationReset("User reset");
                ImGui::SameLine();
                ImGui::Text("Accumulated samples: %u", m_AccumulationFrame);
            }
```

- [ ] **Step 4: Add realtime firefly controls next to existing reference firefly controls**

Replace the current reference-only firefly block:

```cpp
            ResetOnChange(ImGui::Checkbox("FireflyFilter (reference *)", &m_ReferenceUI.ReferenceFireflyFilterEnabled),
                          "Firefly filter toggled");
            if (m_ReferenceUI.ReferenceFireflyFilterEnabled)
            {
                ImGui::Indent(Indent);
                ResetOnChange(ImGui::InputFloat("FF Threshold", &m_ReferenceUI.ReferenceFireflyFilterThreshold, 0.1f, 0.2f, "%.5f"),
                              "Firefly threshold changed");
                ImGui::Unindent(Indent);
            }
```

with:

```cpp
            if (m_RealtimeUI.RealtimeMode)
            {
                ResetRealtimeOnChange(ImGui::Checkbox("FireflyFilter (realtime)", &m_RealtimeUI.RealtimeFireflyFilterEnabled),
                                      "Realtime firefly filter toggled");
                if (m_RealtimeUI.RealtimeFireflyFilterEnabled)
                {
                    ImGui::Indent(Indent);
                    ResetRealtimeOnChange(ImGui::InputFloat("FF Threshold", &m_RealtimeUI.RealtimeFireflyFilterThreshold, 0.01f, 0.1f, "%.5f"),
                                          "Realtime firefly threshold changed");
                    m_RealtimeUI.RealtimeFireflyFilterThreshold =
                        std::clamp(m_RealtimeUI.RealtimeFireflyFilterThreshold, 0.00001f, 1000.0f);
                    ImGui::Unindent(Indent);
                }
            }
            else
            {
                ResetOnChange(ImGui::Checkbox("FireflyFilter (reference *)", &m_ReferenceUI.ReferenceFireflyFilterEnabled),
                              "Firefly filter toggled");
                if (m_ReferenceUI.ReferenceFireflyFilterEnabled)
                {
                    ImGui::Indent(Indent);
                    ResetOnChange(ImGui::InputFloat("FF Threshold", &m_ReferenceUI.ReferenceFireflyFilterThreshold, 0.1f, 0.2f, "%.5f"),
                                  "Firefly threshold changed");
                    ImGui::Unindent(Indent);
                }
            }
```

- [ ] **Step 5: Add realtime AA/SR/denoiser controls in the post-processing group**

At the start of the existing `Post processing:` block, before the `Bloom` header, add:

```cpp
            if (m_RealtimeUI.RealtimeMode)
            {
                const char* AAItems[] = {"Disabled", "TAA", "Super Resolution", "DLSS-RR"};
                int         AAMode    = static_cast<int>(m_RealtimeUI.RealtimeAA);
                if (ImGui::BeginCombo("AA/SR/Denoising", AAItems[std::clamp(AAMode, 0, 3)]))
                {
                    for (int Item = 0; Item < 4; ++Item)
                    {
                        const bool Enabled =
                            Item == static_cast<int>(RTXPTRealtimeAAMode::Disabled) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::TAA) && kRTXPTRealtimeTaaAvailable) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::SuperResolution) && kRTXPTRealtimeSrAvailable) ||
                            (Item == static_cast<int>(RTXPTRealtimeAAMode::DLSSRR) && kRTXPTDlssRrAvailable);

                        ImGui::BeginDisabled(!Enabled);
                        const bool Selected = AAMode == Item;
                        if (ImGui::Selectable(AAItems[Item], Selected))
                        {
                            AAMode                 = Item;
                            m_RealtimeUI.RealtimeAA = static_cast<RTXPTRealtimeAAMode>(AAMode);
                            ResetTaaSrOnChange(true, "Realtime AA/SR mode changed");
                        }
                        if (Selected)
                            ImGui::SetItemDefaultFocus();
                        ImGui::EndDisabled();
                    }
                    ImGui::EndCombo();
                }
                if (ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenDisabled))
                    ImGui::SetTooltip("TAA and Super Resolution execute in G10. DLSS-RR is reserved by TODO(RTXPT-Realtime-DLSS-RR).");

                const bool DenoiserDisabled =
                    m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR ||
                    !kRTXPTStandaloneNrdAvailable;
                ImGui::BeginDisabled(DenoiserDisabled);
                if (ResetRealtimeOnChange(ImGui::Checkbox("Use standalone denoiser (NRD)", &m_RealtimeUI.StandaloneDenoiser),
                                          "Standalone denoiser toggled"))
                {
                    RequestRealtimeReset(RTXPT_REALTIME_RESET_NRD_HISTORY, "Standalone denoiser toggled");
                }
                ImGui::EndDisabled();
                if (DenoiserDisabled)
                    DrawDisabledTooltip(m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR ?
                                            "Standalone NRD is disabled for DLSS-RR; TODO(RTXPT-Realtime-DLSS-RR)." :
                                            kRTXPTNrdDisabledReason);
            }
```

- [ ] **Step 6: Add realtime texture LOD bias control**

After firefly controls and before post-processing controls, add:

```cpp
            if (m_RealtimeUI.RealtimeMode)
            {
                ResetRealtimeOnChange(ImGui::InputFloat("Texture MIP bias", &m_RealtimeUI.TexLODBias),
                                      "Realtime texture MIP bias changed");
            }
```

- [ ] **Step 7: Build-check UI setup**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the RTXPT sample compiles and the UI can store realtime setup state without NRD headers.

- [ ] **Step 8: Commit Task 3**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): expose realtime setup controls" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 4: Add Stable-Plane and NRD UI Panels

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1335-1483`
- Read: `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp:15-64`

- [ ] **Step 1: Add the stable-plane UI panel**

In `RTXPTSample::UpdateUI()`, after `PT: Advanced Settings` and before `Environment Map`, insert:

```cpp
    // ------------------------------------------------ Stable Planes (denoising layers)
    if (ImGui::CollapsingHeader("Stable Planes (denoising layers)"))
    {
        ImGui::Indent(Indent);

        if (m_RealtimeUI.RealtimeMode)
        {
            int ActiveStablePlanes = m_RealtimeUI.StablePlanesActiveCount;
            if (ResetRealtimeOnChange(ImGui::InputInt("Active stable planes", &ActiveStablePlanes),
                                      "Active stable-plane count changed"))
                m_RealtimeUI.StablePlanesActiveCount = ActiveStablePlanes;
            if (ImGui::IsItemHovered())
                ImGui::SetTooltip("How many stable planes to allow; one plane is standard denoising.");

            int MaxStablePlaneVertexDepth = m_RealtimeUI.StablePlanesMaxVertexDepth;
            if (ResetRealtimeOnChange(ImGui::InputInt("Max stable plane vertex depth", &MaxStablePlaneVertexDepth),
                                      "Stable-plane max vertex depth changed"))
                m_RealtimeUI.StablePlanesMaxVertexDepth = MaxStablePlaneVertexDepth;
            if (ImGui::IsItemHovered())
                ImGui::SetTooltip("How deep the stable part of path tracing can go.");

            ResetRealtimeOnChange(ImGui::SliderFloat("Path split stop threshold", &m_RealtimeUI.StablePlanesSplitStopThreshold, 0.0f, 2.0f),
                                  "Stable-plane split threshold changed");
            ResetRealtimeOnChange(ImGui::Checkbox("Primary Surface Replacement", &m_RealtimeUI.AllowPrimarySurfaceReplacement),
                                  "Primary surface replacement toggled");
            ResetRealtimeOnChange(ImGui::Checkbox("Suppress primary plane noisy specular", &m_RealtimeUI.StablePlanesSuppressPrimaryIndirectSpecular),
                                  "Primary-plane noisy specular suppression toggled");
            ResetRealtimeOnChange(ImGui::SliderFloat("Suppress primary plane noisy specular amount",
                                                     &m_RealtimeUI.StablePlanesSuppressPrimaryIndirectSpecularK, 0.0f, 1.0f),
                                  "Primary-plane noisy specular suppression amount changed");
            ResetRealtimeOnChange(ImGui::SliderFloat("Non-primary plane anti-aliasing fallthrough",
                                                     &m_RealtimeUI.StablePlanesAntiAliasingFallthrough, 0.0f, 1.0f),
                                  "Stable-plane anti-aliasing fallthrough changed");
        }
        else
        {
            ImGui::Text("Not available in reference mode");
        }

        ImGui::Unindent(Indent);
    }
```

- [ ] **Step 2: Add the standalone NRD panel**

Immediately after the stable-plane panel, insert:

```cpp
    // ------------------------------------------------------- Standalone Denoiser (NRD)
    if (ImGui::CollapsingHeader("Standalone Denoiser (NRD)"))
    {
        ImGui::Indent(Indent);

        if (!m_RealtimeUI.RealtimeMode)
            ImGui::TextWrapped("Not available in reference mode.");
        if (m_RealtimeUI.RealtimeMode && !kRTXPTStandaloneNrdAvailable)
            ImGui::TextWrapped("%s", kRTXPTNrdDisabledReason);

        const bool DisableNrdControls =
            !m_RealtimeUI.RealtimeMode ||
            !m_RealtimeUI.ActualUseStandaloneDenoiser() ||
            !kRTXPTStandaloneNrdAvailable;

        ImGui::BeginDisabled(DisableNrdControls);

        auto SliderUint = [&](const char* Label, Uint32& Value, int MinValue, int MaxValue, const char* Reason) {
            int ValueUI = static_cast<int>(Value);
            if (ResetRealtimeOnChange(ImGui::SliderInt(Label, &ValueUI, MinValue, MaxValue), Reason))
            {
                Value = static_cast<Uint32>(std::clamp(ValueUI, MinValue, MaxValue));
                return true;
            }
            return false;
        };

        ResetRealtimeOnChange(ImGui::InputFloat("Disocclusion Threshold", &m_RealtimeUI.NRDDisocclusionThreshold),
                              "NRD disocclusion threshold changed");
        ResetRealtimeOnChange(ImGui::Checkbox("Use Alternate Disocclusion Threshold Mix",
                                              &m_RealtimeUI.NRDUseAlternateDisocclusionThresholdMix),
                              "NRD alternate disocclusion mix toggled");
        ResetRealtimeOnChange(ImGui::InputFloat("Disocclusion Threshold Alt", &m_RealtimeUI.NRDDisocclusionThresholdAlternate),
                              "NRD alternate disocclusion threshold changed");
        ResetRealtimeOnChange(ImGui::InputFloat("Radiance clamping", &m_RealtimeUI.DenoiserRadianceClampK),
                              "NRD radiance clamp changed");

        ImGui::Separator();

        int NrdMethod = static_cast<int>(m_RealtimeUI.NRDMethod);
        if (ImGui::Combo("Denoiser Mode", &NrdMethod, "REBLUR\0RELAX\0\0"))
        {
            m_RealtimeUI.NRDMethod = static_cast<RTXPTNrdMethod>(std::clamp(NrdMethod, 0, 1));
            RequestRealtimeReset(RTXPT_REALTIME_RESET_NRD_HISTORY, "NRD mode changed");
        }

        if (ImGui::CollapsingHeader("Advanced Settings"))
        {
            if (m_RealtimeUI.NRDMethod == RTXPTNrdMethod::REBLUR)
            {
                RTXPTNrdReblurUiSettings& Reblur = m_RealtimeUI.ReblurSettings;
                SliderUint("Max Accumulated Frames", Reblur.MaxAccumulatedFrameNum, 0, 500,
                           "REBLUR max accumulated frames changed");
                SliderUint("Fast Max Accumulated Frames", Reblur.MaxFastAccumulatedFrameNum, 0, 500,
                           "REBLUR fast max accumulated frames changed");
                SliderUint("History Fix Frames", Reblur.HistoryFixFrameNum, 0, 500,
                           "REBLUR history fix frames changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Diffuse Prepass Blur Radius (pixels)", &Reblur.DiffusePrepassBlurRadius, 0.0f, 100.0f),
                                      "REBLUR diffuse prepass blur radius changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Specular Prepass Blur Radius (pixels)", &Reblur.SpecularPrepassBlurRadius, 0.0f, 100.0f),
                                      "REBLUR specular prepass blur radius changed");
                int HitDistanceMode = static_cast<int>(Reblur.HitDistanceReconstructionMode);
                if (ImGui::Combo("Hit Distance Reconstruction Mode", &HitDistanceMode, "Off\0AREA_3X3\0AREA_5X5\0\0"))
                {
                    Reblur.HitDistanceReconstructionMode =
                        static_cast<RTXPTNrdHitDistanceReconstructionMode>(std::clamp(HitDistanceMode, 0, 2));
                    RequestRealtimeReset(RTXPT_REALTIME_RESET_NRD_HISTORY, "REBLUR hit-distance mode changed");
                }
                ResetRealtimeOnChange(ImGui::Checkbox("Enable Firefly Filter", &Reblur.EnableAntiFirefly),
                                      "REBLUR anti-firefly toggled");
            }
            else
            {
                RTXPTNrdRelaxUiSettings& Relax = m_RealtimeUI.RelaxSettings;
                ResetRealtimeOnChange(ImGui::SliderFloat("Diffuse Prepass Blur Radius", &Relax.DiffusePrepassBlurRadius, 0.0f, 100.0f),
                                      "RELAX diffuse prepass blur radius changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Specular Prepass Blur Radius", &Relax.SpecularPrepassBlurRadius, 0.0f, 100.0f),
                                      "RELAX specular prepass blur radius changed");
                SliderUint("Diffuse Max Accumulated Frames", Relax.DiffuseMaxAccumulatedFrameNum, 0, 500,
                           "RELAX diffuse max accumulated frames changed");
                SliderUint("Specular Max Accumulated Frames", Relax.SpecularMaxAccumulatedFrameNum, 0, 500,
                           "RELAX specular max accumulated frames changed");
                SliderUint("Diffuse Fast Max Accumulated Frames", Relax.DiffuseMaxFastAccumulatedFrameNum, 0, 10,
                           "RELAX diffuse fast max accumulated frames changed");
                SliderUint("Specular Fast Max Accumulated Frames", Relax.SpecularMaxFastAccumulatedFrameNum, 0, 10,
                           "RELAX specular fast max accumulated frames changed");
                SliderUint("History Fix Frame Num", Relax.HistoryFixFrameNum, 0, 500,
                           "RELAX history fix frames changed");
                SliderUint("Number of Atrous iterations", Relax.AtrousIterationNum, 2, 8,
                           "RELAX atrous iterations changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Lobe Angle Fraction", &Relax.LobeAngleFraction, 0.0f, 1.0f),
                                      "RELAX lobe angle fraction changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Specular Lobe Angle Slack", &Relax.SpecularLobeAngleSlack, 0.0f, 1.0f),
                                      "RELAX specular lobe angle slack changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Edge Stopping Threshold", &Relax.DepthThreshold, 0.0f, 0.1f),
                                      "RELAX depth threshold changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Antilag Acceleration Amount", &Relax.AntilagAccelerationAmount, 0.0f, 1.0f),
                                      "RELAX antilag acceleration changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Antilag Spatial Sigma Scale", &Relax.AntilagSpatialSigmaScale, 0.0f, 5.0f),
                                      "RELAX antilag spatial sigma changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Antilag Temporal Sigma Scale", &Relax.AntilagTemporalSigmaScale, 0.0f, 5.0f),
                                      "RELAX antilag temporal sigma changed");
                ResetRealtimeOnChange(ImGui::SliderFloat("Antilag Reset Amount", &Relax.AntilagResetAmount, 0.0f, 1.0f),
                                      "RELAX antilag reset amount changed");
                int HitDistanceMode = static_cast<int>(Relax.HitDistanceReconstructionMode);
                if (ImGui::Combo("Hit Distance Reconstruction Mode", &HitDistanceMode, "Off\0AREA_3X3\0AREA_5X5\0\0"))
                {
                    Relax.HitDistanceReconstructionMode =
                        static_cast<RTXPTNrdHitDistanceReconstructionMode>(std::clamp(HitDistanceMode, 0, 2));
                    RequestRealtimeReset(RTXPT_REALTIME_RESET_NRD_HISTORY, "RELAX hit-distance mode changed");
                }
                ResetRealtimeOnChange(ImGui::Checkbox("Enable Firefly Filter", &Relax.EnableAntiFirefly),
                                      "RELAX anti-firefly toggled");
            }
        }

        ImGui::EndDisabled();
        if (DisableNrdControls && ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenDisabled))
            ImGui::SetTooltip("%s", kRTXPTNrdDisabledReason);

        ImGui::Unindent(Indent);
    }
```

- [ ] **Step 3: Build-check the stable-plane and NRD UI panels**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the sample compiles without NRD headers, and the compiler sees no `reinterpret_cast<int*>` UI writes.

- [ ] **Step 4: Commit Task 4**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): add realtime stable-plane and nrd ui" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 5: Add Realtime Render Gate and Status Reporting

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add a visible realtime render gate**

In `RTXPTSample::Render()`, after `EnsureRenderTargets()` succeeds and before `m_LightsBaker.UpdateEnd(...)`, add:

```cpp
    if (m_RealtimeUI.RealtimeMode)
    {
        ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
        return;
    }
```

This keeps Reference mode unchanged and prevents Realtime mode from accidentally presenting the reference raygen as realtime parity.

- [ ] **Step 2: Add realtime status/debug lines**

In the `Status / Debug` collapsing header, after:

```cpp
        ImGui::Text("Frame index: %u", m_FrameIndex);
```

add:

```cpp
        ImGui::Text("Path tracer mode: %s", m_RealtimeUI.RealtimeMode ? "Realtime" : "Reference");
        if (m_RealtimeUI.RealtimeMode)
            ImGui::TextWrapped("Realtime execution: disabled (%s)", kRTXPTRealtimeDisabledReason);
        ImGui::Text("Realtime samples per pixel: %u", m_RealtimeUI.ActualSamplesPerPixel());
        ImGui::Text("Realtime AA/SR: %s", GetRealtimeAAModeName(m_RealtimeUI.RealtimeAA));
        ImGui::Text("Standalone NRD requested: %s", m_RealtimeUI.ActualUseStandaloneDenoiser() ? "yes" : "no");
        ImGui::Text("NRD availability: %s", kRTXPTStandaloneNrdAvailable ? "available" : kRTXPTNrdDisabledReason);
        ImGui::Text("NRD method: %s", GetNrdMethodName(m_RealtimeUI.NRDMethod));
        ImGui::Text("Stable planes active: %d / %u", m_RealtimeUI.StablePlanesActiveCount, kRTXPTStablePlaneCount);
        ImGui::Text("Current realtime reset flags: 0x%08x", static_cast<Uint32>(m_CurrentFrameRealtimeReset));
        ImGui::Text("Pending realtime reset flags: 0x%08x", static_cast<Uint32>(m_RealtimeResetPending));
```

- [ ] **Step 3: Update super-resolution status line**

Replace:

```cpp
        ImGui::Text("Super resolution: disabled in reference mode");
```

with:

```cpp
        ImGui::Text("Super resolution: %s",
                    m_RealtimeUI.RealtimeMode && m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::SuperResolution ?
                        "selected, execution starts in G10" :
                        "disabled");
```

- [ ] **Step 4: Add mapping rows**

In `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`, near the existing `SampleUI.h` / `SampleUI.cpp` mapping rows, add:

```markdown
| `SampleUI.h::SampleUIData` realtime fields | `src/RTXPTRealtimeSettings.hpp`, `src/RTXPTSample.hpp` | Realtime G1 | Diligent-local realtime state mirrors `RealtimeMode`, `RealtimeSamplesPerPixel`, `RealtimeAA`, `StandaloneDenoiser`, stable-plane controls, firefly controls, NRD UI settings, and reset-request flags without taking a compile dependency on NRD. |
| `SampleUI.h::ActualUseStandaloneDenoiser` | `src/RTXPTRealtimeSettings.hpp::RTXPTRealtimeSettings::ActualUseStandaloneDenoiser` | Realtime G1 | Preserves RTXPT-fork semantics: true only when `RealtimeMode && RealtimeAA < 3 && StandaloneDenoiser`. |
| `SampleUI.cpp` realtime UI controls | `src/RTXPTSample.cpp::UpdateUI` | Realtime G1 | Mode, realtime setup, AA/SR/denoiser selection, stable-plane controls, and NRD controls are visible. Realtime execution remains visibly disabled until G2-G10. |
```

- [ ] **Step 5: Source-scan for stale broad placeholder text**

Run:

```powershell
rg -n "Realtime mode is out of scope|umbrella Phase 5.5|TODO\\(RTXPT-Realtime-DLSS-RR\\)" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected:

```text
TODO(RTXPT-Realtime-DLSS-RR)
```

may appear only in disabled DLSS-RR UI/status text. Broad stale placeholder strings such as `Realtime mode is out of scope` do not appear.

- [ ] **Step 6: Build-check render gate and status UI**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the sample compiles. Reference mode still uses the existing render path; Realtime mode returns through the visible disabled fallback.

- [ ] **Step 7: Commit Task 5**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "feat(rtxpt): report realtime path status" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 6: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Run source scans for G1 coverage**

Run:

```powershell
rg -n "RealtimeMode|RealtimeSamplesPerPixel|RealtimeAA|StandaloneDenoiser|ActualUseStandaloneDenoiser|StablePlanesActiveCount|StablePlanesMaxVertexDepth|AllowPrimarySurfaceReplacement|StablePlanesSplitStopThreshold|StablePlanesSuppressPrimaryIndirectSpecular|StablePlanesAntiAliasingFallthrough|DenoiserRadianceClampK|NRDMethod|NRDDisocclusionThreshold|NRDUseAlternateDisocclusionThresholdMix|ReblurSettings|RelaxSettings" DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: every G1 setting appears in either the settings header or `UpdateUI()`.

- [ ] **Step 2: Run source scans for G2 boundary control**

Run:

```powershell
rg -n "sampleBaseIndex|invSubSampleCount|denoisingEnabled|_activeStablePlaneCount|genericTSPlaneStride|DenoiserViewspaceZ|StablePlanesBuffer" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected: no new G2/G3 resource or constant fields were added by G1. Pre-existing spec text or mapping rows are fine if the search scope includes docs, so keep this scan scoped to source and shaders.

- [ ] **Step 3: Run diff hygiene**

Run:

```powershell
git diff --check
git diff -- DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: `git diff --check` reports no whitespace errors. The diff touches only the G1 files listed above unless the implementation uncovered a necessary local include/order adjustment.

- [ ] **Step 4: Run formatting validation for DiligentSamples**

Run:

```powershell
Push-Location DiligentSamples\BuildTools\FormatValidation
.\validate_format_win.bat
Pop-Location
```

Expected: formatting validation completes successfully. If the script is missing or local clang-format is unavailable, record the exact error and run the sample build in Step 5.

- [ ] **Step 5: Build the RTXPT sample target**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If `build\x64\Debug` is not configured, run the repository's existing configure flow before claiming build success.

- [ ] **Step 6: Manual UI smoke**

Run:

```powershell
$RTXPTExe = Get-ChildItem -Path build\x64\Debug -Recurse -Filter RTXPT.exe | Select-Object -First 1 -ExpandProperty FullName
& $RTXPTExe
```

Expected:

```text
Path Tracer -> Mode defaults to Reference.
Reference mode renders through the existing path.
Switching Mode to Realtime shows realtime setup controls.
Realtime mode does not execute the reference raygen and presents the disabled fallback/status.
DLSS-RR appears only as disabled/reserved.
Standalone NRD controls show a clear disabled reason when NRD is unavailable.
Stable-plane controls are hidden from execution in Reference mode and visible in Realtime mode.
```

- [ ] **Step 7: Final status**

Report:

```text
Verification run:
- git diff --check: <result>
- DiligentSamples format validation: <result or exact blocker>
- cmake --build build\x64\Debug --config Debug --target RTXPT: <result or exact blocker>
- Manual UI smoke: <result or not run with reason>
```

Do not claim G1 is complete unless the build and at least source-scan verification have been run or their blockers are explicitly reported.

## Self-Review Checklist

- Spec coverage:
  - `RealtimeMode`, `RealtimeSamplesPerPixel`, `RealtimeAA`, `StandaloneDenoiser`: Task 1 and Task 3.
  - `ActualUseStandaloneDenoiser()`: Task 1.
  - Stable-plane active count, max vertex depth, primary surface replacement, split threshold, specular suppression, anti-aliasing fallthrough: Task 1 and Task 4.
  - Realtime firefly filter, denoiser radiance clamp, reset realtime caches: Task 1, Task 3, Task 4.
  - NRD method, disocclusion thresholds, alternate mix, REBLUR settings, RELAX settings: Task 1 and Task 4.
  - `RealtimeAA == 3` disabled/deferred controls: Task 3 and Task 5.
  - Reference mode default and unchanged rendering: Task 2, Task 5, Task 6.
  - Realtime path selected only when explicitly enabled: Task 3 and Task 5.
  - Narrow reset request semantics: Task 2, Task 3, Task 4.
- Placeholder scan:
  - The only allowed deferred marker is the exact spec-required `TODO(RTXPT-Realtime-DLSS-RR)` marker in disabled DLSS-RR UI/status text.
  - No broad deferred-marker prose or unspecified implementation steps are used.
- Type consistency:
  - `RTXPTRealtimeAAMode`, `RTXPTNrdMethod`, `RTXPTNrdHitDistanceReconstructionMode`, `RTXPTRealtimeResetFlags`, and `RTXPTRealtimeSettings` names are defined in Task 1 before later tasks reference them.
  - `m_RealtimeUI`, `m_RealtimeResetPending`, and `m_CurrentFrameRealtimeReset` are declared in Task 2 before later UI/status tasks use them.
  - `ResetRealtimeOnChange`, `ResetTaaSrOnChange`, and `ResetRenderTargetsOnChange` are introduced in Task 3 before Task 4 uses `ResetRealtimeOnChange`.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-03-rtxpt-realtime-g1-state-ui-parity.md`. Two execution options:

1. **Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Which approach?
