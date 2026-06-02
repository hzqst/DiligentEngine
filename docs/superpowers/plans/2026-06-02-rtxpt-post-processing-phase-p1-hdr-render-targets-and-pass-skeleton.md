# RTXPT Post-Processing Phase P1 HDR Render Targets and Pass Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the Phase 6 render-target graph and post-process pipeline skeleton so later P2-P5 ports can map 1:1 to RTXPT-fork's post-processing flow.

**Architecture:** P1 changes the resource contract first, not the image algorithm. `RTXPTRenderTargets` owns RTXPT-fork-named targets with Diligent-native texture creation, `RTXPTPostProcessPipeline` becomes the future pass orchestration owner, and `RTXPTSample` still presents through the existing `RTXPTBlitPass` source selection until accumulation and tone mapping land in P2/P3.

**Tech Stack:** C++17, HLSL ray tracing shader image-format declarations, Diligent Engine texture/view APIs, CMake sample source registration, PowerShell + `rg` verification, RTXPT-fork reference sources under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P1.
- P0 mapping is already present in `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` under `## Phase 6 Post-Processing Pipeline Mapping`.
- Current Diligent `RTXPTRenderTargets` owns `OutputColor`, optional `ComputeColor`, and optional `AccumColor` only.
- Current `RTXPTRenderTargets::Resize` receives one `TEXTURE_FORMAT Format`, and `RTXPTSample::EnsureRenderTargets` / `WindowResize` pass `TEX_FORMAT_RGBA8_UNORM`.
- Current `PathTracerSample.rgen` declares `u_Output` three times with `VK_IMAGE_FORMAT("rgba8")`.
- Current raygen still writes accumulation into `u_AccumulationBuffer`, applies `ToneMapACES(accumulated * g_Const.ptConsts.exposureScale)`, and writes display-ready color to `u_Output`. P1 must not move those behaviors.
- Current `RTXPTSample::Render` still runs `Trace -> optional RTXPTComputePass -> RTXPTBlitPass`.

## RTXPT-Fork Anchors

Use these anchors as the P1 source of truth:

- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h` - target names and ownership comments.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp:154-247` - core formats: `AccumulatedRadiance` is `RGBA32_FLOAT`, `OutputColor` is `RGBA16_FLOAT`, `ProcessedOutputColor` is HDR radiance format, `LdrColor` and `LdrColorScratch` are SDR ping-pong targets.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1279-1330` - `CreateRenderPasses` pass construction order.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2186-2213` - frame-tail order: AA/accumulation, HDR post-process, tone mapping, LDR post-process, overlays, final blit.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2330-2348` - resource binding names: `t_LdrColorScratch`, `u_OutputColor`, `u_ProcessedOutputColor`, `u_PostTonemapOutputColor`.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/AccumulationPass.*` - P2 owner, not implemented in P1.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMappingPasses.*` - P3 owner, not implemented in P1.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.*` - P4+ owner, not implemented in P1.

## P1 Scope Boundaries

- P1 must create and expose `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, and `LdrColorScratch`.
- P1 must change `PathTracerSample.rgen` `u_Output` image-format annotations from `rgba8` to `rgba16f` so the shader declaration matches the new HDR `OutputColor` texture.
- P1 must keep `u_AccumulationBuffer`, `ToneMapACES`, and `exposureScale` in raygen. Their removal belongs to P2/P3.
- P1 must keep `RTXPTComputePass` diagnostic-only.
- P1 must keep `RTXPTBlitPass` as the final swapchain copy and may continue blitting `OutputColor` or `ComputeColor` until P3/P5 switch the normal source to `LdrColor`.
- P1 must not create `RTXPTAccumulationPass`, `RTXPTToneMappingPass`, or `RTXPTPostProcessPass` implementation files. Those focused files land with P2, P3, and P4.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp` - resource contract, getters, format fields, validity helpers.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp` - separate texture creation paths and format checks.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp` - P1 orchestration skeleton and status.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp` - skeleton initialization and resource validation.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - include/member for the pipeline skeleton.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - render-target resize calls, pipeline initialization, diagnostics, and renamed accessors.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - `u_Output` image format only.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register the new pipeline source/header.
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - existing P0 mapping remains the ownership contract.

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

Expected: branch lines are present. If either worktree is dirty, inspect the listed files before editing and preserve user changes.

- [ ] **Step 2: Confirm P0 mapping exists**

Run:

```powershell
rg -n "Phase 6 Post-Processing Pipeline Mapping|OutputColor.*raw HDR|AccumulatedRadiance|ProcessedOutputColor|LdrColorScratch|RTXPTPostProcessPipeline" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: matches in the Phase 6 mapping section. If no matches appear, stop and complete P0 first.

- [ ] **Step 3: Confirm upstream reference files exist**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\SampleCommon\RenderTargets.h
Test-Path D:\RTXPT-fork\Rtxpt\SampleCommon\RenderTargets.cpp
Test-Path D:\RTXPT-fork\Rtxpt\Sample.cpp
Test-Path D:\RTXPT-fork\Rtxpt\ProcessingPasses\AccumulationPass.hlsl
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ToneMappingPasses.h
Test-Path D:\RTXPT-fork\Rtxpt\ProcessingPasses\PostProcess.hlsl
```

Expected: every command prints `True`.

### Task 1: Freeze the P1 Resource Contract

**Files:**
- Read: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.*`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp`
- Read: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.*`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Reconfirm RTXPT-fork formats and names**

Run:

```powershell
rg -n "AccumulatedRadiance|OutputColor|ProcessedOutputColor|LdrColor|LdrColorScratch|RGBA32_FLOAT|RGBA16_FLOAT|SRGBA8_UNORM" D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp
```

Expected: output shows the five target names and the upstream formats listed in this plan.

- [ ] **Step 2: Reconfirm current Diligent debt points**

Run:

```powershell
rg -n "TEX_FORMAT_RGBA8_UNORM|m_AccumColor|GetAccumColor|GetDisplaySRV|VK_IMAGE_FORMAT\\(\"rgba8\"\\).*u_Output|ToneMapACES|u_AccumulationBuffer" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.* DiligentSamples/Samples/RTXPT/src/RTXPTSample.* DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: matches identify the single-format render target API, the current accumulation texture name, the existing blit source helper, all `rgba8` `u_Output` declarations, and the P2/P3 raygen debt.

- [ ] **Step 3: Confirm P1 pass skeleton names are still unused**

Run:

```powershell
rg -n "RTXPTPostProcessPipeline|RTXPTAccumulationPass|RTXPTToneMappingPass|RTXPTPostProcessPass" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: no matches for `RTXPTPostProcessPipeline`. `RTXPTAccumulationPass`, `RTXPTToneMappingPass`, and `RTXPTPostProcessPass` should also be absent until later phases.

### Task 2: Expand RTXPTRenderTargets

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Replace the single-format API with explicit Phase 6 formats**

In `RTXPTRenderTargets.hpp`, add a small format struct before `class RTXPTRenderTargets`:

```cpp
struct RTXPTRenderTargetFormats
{
    TEXTURE_FORMAT OutputColor          = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT AccumulatedRadiance  = TEX_FORMAT_RGBA32_FLOAT;
    TEXTURE_FORMAT ProcessedOutputColor = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT LdrColor             = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT ComputeColor         = TEX_FORMAT_RGBA8_UNORM;
};
```

Expected: `OutputColor` matches RTXPT-fork `RGBA16_FLOAT`; `AccumulatedRadiance` stays `RGBA32_FLOAT`; `LdrColor` uses the current Diligent-safe SDR format and keeps the name aligned for a later sRGB/typeless view decision.

- [ ] **Step 2: Update the public `Resize` signature**

Change:

```cpp
bool Resize(IRenderDevice* pDevice,
            Uint32         Width,
            Uint32         Height,
            TEXTURE_FORMAT Format,
            bool           CreateComputeOutput,
            bool           CreateAccumulation);
```

to:

```cpp
bool Resize(IRenderDevice*                  pDevice,
            Uint32                          Width,
            Uint32                          Height,
            const RTXPTRenderTargetFormats& Formats,
            bool                            CreateComputeOutput,
            bool                            CreateAccumulatedRadiance);
```

Expected: callers must now pass the complete post-processing format contract instead of a single display format.

- [ ] **Step 3: Add explicit resource accessors**

In `RTXPTRenderTargets.hpp`, keep the existing output/compute accessors and replace `AccumColor` names with RTXPT-fork names:

```cpp
bool IsValid() const { return HasPostProcessTargets(); }
bool HasPostProcessTargets() const;
bool IsAccumulationActive() const { return m_AccumulatedRadiance != nullptr; }

ITextureView* GetOutputColorUAV() const;
ITextureView* GetOutputColorSRV() const;
ITextureView* GetAccumulatedRadianceUAV() const;
ITextureView* GetAccumulatedRadianceSRV() const;
ITextureView* GetProcessedOutputColorUAV() const;
ITextureView* GetProcessedOutputColorSRV() const;
ITextureView* GetLdrColorUAV() const;
ITextureView* GetLdrColorSRV() const;
ITextureView* GetLdrColorScratchUAV() const;
ITextureView* GetLdrColorScratchSRV() const;
ITextureView* GetComputeColorUAV() const;
ITextureView* GetComputeColorSRV() const;
ITextureView* GetDisplaySRV(bool UseComputeOutput) const;

TEXTURE_FORMAT GetOutputColorFormat() const { return m_Formats.OutputColor; }
TEXTURE_FORMAT GetAccumulatedRadianceFormat() const { return m_Formats.AccumulatedRadiance; }
TEXTURE_FORMAT GetProcessedOutputColorFormat() const { return m_Formats.ProcessedOutputColor; }
TEXTURE_FORMAT GetLdrColorFormat() const { return m_Formats.LdrColor; }
```

Expected: P2/P3 can bind `AccumulatedRadiance`, `ProcessedOutputColor`, and `LdrColor` without renaming the target owner again.

- [ ] **Step 4: Replace private members with RTXPT-fork target names**

In `RTXPTRenderTargets.hpp`, replace the old private texture fields with:

```cpp
bool CreateTarget(IRenderDevice* pDevice,
                  const char*    Name,
                  TEXTURE_FORMAT Format,
                  BIND_FLAGS     BindFlags,
                  RefCntAutoPtr<ITexture>& Target);
bool SupportsBindFlags(IRenderDevice* pDevice, TEXTURE_FORMAT Format, BIND_FLAGS BindFlags) const;

RefCntAutoPtr<ITexture> m_OutputColor;
RefCntAutoPtr<ITexture> m_AccumulatedRadiance;
RefCntAutoPtr<ITexture> m_ProcessedOutputColor;
RefCntAutoPtr<ITexture> m_LdrColor;
RefCntAutoPtr<ITexture> m_LdrColorScratch;
RefCntAutoPtr<ITexture> m_ComputeColor;
bool                    m_AccumulatedRadianceUnavailable = false;
Uint32                  m_Width                          = 0;
Uint32                  m_Height                         = 0;
RTXPTRenderTargetFormats m_Formats;
```

Expected: no private field named `m_AccumColor` remains.

- [ ] **Step 5: Implement explicit texture creation and validation**

In `RTXPTRenderTargets.cpp`, update `Reset`, `CreateTarget`, and `Resize` to create these targets:

```cpp
const BIND_FLAGS HdrUavFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
const BIND_FLAGS HdrRtFlags  = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;
const BIND_FLAGS LdrRtFlags  = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;

CreateTarget(pDevice, "RTXPT OutputColor", m_Formats.OutputColor, HdrUavFlags, m_OutputColor);
CreateTarget(pDevice, "RTXPT AccumulatedRadiance", m_Formats.AccumulatedRadiance, HdrUavFlags, m_AccumulatedRadiance);
CreateTarget(pDevice, "RTXPT ProcessedOutputColor", m_Formats.ProcessedOutputColor, HdrRtFlags, m_ProcessedOutputColor);
CreateTarget(pDevice, "RTXPT LdrColor", m_Formats.LdrColor, LdrRtFlags, m_LdrColor);
CreateTarget(pDevice, "RTXPT LdrColorScratch", m_Formats.LdrColor, LdrRtFlags, m_LdrColorScratch);
```

Expected: `OutputColor` uses `TEX_FORMAT_RGBA16_FLOAT` and all post-process targets are allocated on a successful resize. `ComputeColor` remains optional and diagnostic-only.

- [ ] **Step 6: Treat unsupported HDR UAV formats as a visible failure**

Before creating `OutputColor` and `AccumulatedRadiance`, check required bind flags:

```cpp
if (!SupportsBindFlags(pDevice, m_Formats.OutputColor, HdrUavFlags))
{
    LOG_ERROR_MESSAGE("RGBA16F UAV OutputColor is not supported; RTXPT post-processing resource graph is unavailable");
    return false;
}

if (CreateAccumulatedRadiance && !SupportsBindFlags(pDevice, m_Formats.AccumulatedRadiance, HdrUavFlags))
{
    LOG_ERROR_MESSAGE("RGBA32F UAV AccumulatedRadiance is not supported; reference accumulation is unavailable");
    m_AccumulatedRadianceUnavailable = true;
    return true;
}
```

Expected: P1 does not silently bind a texture whose format disagrees with `PathTracerSample.rgen`. A future fallback format requires a matching shader image-format strategy.

- [ ] **Step 7: Preserve the temporary P1 display source**

Keep `GetDisplaySRV(bool UseComputeOutput)` as the temporary blit-source helper:

```cpp
ITextureView* RTXPTRenderTargets::GetDisplaySRV(bool UseComputeOutput) const
{
    if (UseComputeOutput && m_ComputeColor)
        return GetComputeColorSRV();

    return GetOutputColorSRV();
}
```

Expected: P1 still presents through the existing blit path. `LdrColor` is allocated and exposed but not yet the normal display source.

- [ ] **Step 8: Verify the render-target names and old field removal**

Run:

```powershell
rg -n "AccumulatedRadiance|ProcessedOutputColor|LdrColor|LdrColorScratch|TEX_FORMAT_RGBA16_FLOAT|TEX_FORMAT_RGBA32_FLOAT" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.*
rg -n "m_AccumColor|GetAccumColor|TEXTURE_FORMAT Format" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.*
```

Expected: first command shows the new contract; second command prints no matches.

### Task 3: Update PathTracerSample.rgen Output Format Only

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Change all raygen `u_Output` declarations to `rgba16f`**

Replace every raygen declaration:

```hlsl
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4>   u_Output;
```

with:

```hlsl
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_Output;
```

Expected: all three `u_Output` declarations in `PathTracerSample.rgen` match the new HDR `OutputColor` texture. Keep `u_AccumulationBuffer` as `rgba32f`.

- [ ] **Step 2: Leave raygen accumulation and ACES in place**

Run:

```powershell
rg -n "VK_IMAGE_FORMAT\\(\"rgba16f\"\\).*u_Output|VK_IMAGE_FORMAT\\(\"rgba32f\"\\).*u_AccumulationBuffer|ToneMapACES|exposureScale|u_AccumulationBuffer\\[pixel\\]" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
rg -n "VK_IMAGE_FORMAT\\(\"rgba8\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: first command shows `rgba16f` output, `rgba32f` accumulation, and the P2/P3 debt still present. Second command prints no matches in `PathTracerSample.rgen`.

### Task 4: Create RTXPTPostProcessPipeline Skeleton

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`

- [ ] **Step 1: Create the public skeleton header**

Create `RTXPTPostProcessPipeline.hpp` with the normal Diligent copyright header and this class shape:

```cpp
#pragma once

#include <string>

#include "DeviceContext.h"
#include "EngineFactory.h"
#include "RenderDevice.h"
#include "RTXPTRenderTargets.hpp"
#include "SwapChain.h"

namespace Diligent
{

struct RTXPTPostProcessPipelineStats
{
    bool        Ready                  = false;
    bool        ResourcesValid         = false;
    bool        AccumulationStageReady = false;
    bool        HdrStageReady          = false;
    bool        ToneMappingStageReady  = false;
    bool        LdrStageReady          = false;
    std::string DisabledReason;
};

class RTXPTPostProcessPipeline
{
public:
    void Reset();

    bool Initialize(IRenderDevice*  pDevice,
                    IEngineFactory* pEngineFactory,
                    ISwapChain*     pSwapChain,
                    bool            ComputeSupported);

    bool ValidateRenderTargets(const RTXPTRenderTargets& RenderTargets);

    bool                                IsReady() const { return m_Stats.Ready; }
    const RTXPTPostProcessPipelineStats& GetStats() const { return m_Stats; }

private:
    RTXPTPostProcessPipelineStats m_Stats;
};

} // namespace Diligent
```

Expected: the skeleton names future stages but owns no P2/P3/P4 pass objects yet.

- [ ] **Step 2: Implement reset, initialize, and resource validation**

Create `RTXPTPostProcessPipeline.cpp` with:

```cpp
#include "RTXPTPostProcessPipeline.hpp"

#include "DebugUtilities.hpp"

namespace Diligent
{

void RTXPTPostProcessPipeline::Reset()
{
    m_Stats = {};
}

bool RTXPTPostProcessPipeline::Initialize(IRenderDevice*  pDevice,
                                          IEngineFactory* pEngineFactory,
                                          ISwapChain*     pSwapChain,
                                          bool            ComputeSupported)
{
    Reset();

    if (pDevice == nullptr || pEngineFactory == nullptr || pSwapChain == nullptr)
    {
        m_Stats.DisabledReason = "post-process pipeline missing device, engine factory, or swap chain";
        return false;
    }

    m_Stats.Ready = true;
    if (!ComputeSupported)
        m_Stats.DisabledReason = "compute shaders are unavailable; post-process compute stages remain disabled";

    return true;
}

bool RTXPTPostProcessPipeline::ValidateRenderTargets(const RTXPTRenderTargets& RenderTargets)
{
    m_Stats.ResourcesValid =
        RenderTargets.GetOutputColorSRV() != nullptr &&
        RenderTargets.GetOutputColorUAV() != nullptr &&
        RenderTargets.GetAccumulatedRadianceSRV() != nullptr &&
        RenderTargets.GetAccumulatedRadianceUAV() != nullptr &&
        RenderTargets.GetProcessedOutputColorSRV() != nullptr &&
        RenderTargets.GetProcessedOutputColorUAV() != nullptr &&
        RenderTargets.GetLdrColorSRV() != nullptr &&
        RenderTargets.GetLdrColorUAV() != nullptr &&
        RenderTargets.GetLdrColorScratchSRV() != nullptr &&
        RenderTargets.GetLdrColorScratchUAV() != nullptr;

    if (!m_Stats.ResourcesValid)
        m_Stats.DisabledReason = "post-process render targets are incomplete";

    return m_Stats.ResourcesValid;
}

} // namespace Diligent
```

Expected: P1 has a concrete post-process owner and a resource graph validator. The stage booleans stay false until P2/P3/P4 implement them.

- [ ] **Step 3: Verify no focused pass files were created**

Run:

```powershell
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTAccumulationPass.hpp
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTToneMappingPass.hpp
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTPostProcessPass.hpp
```

Expected: every command prints `False`.

### Task 5: Wire P1 Resources and Pipeline into RTXPTSample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add the pipeline include and member**

In `RTXPTSample.hpp`, include the skeleton:

```cpp
#include "RTXPTPostProcessPipeline.hpp"
```

Add the member next to the current pass objects:

```cpp
RTXPTPostProcessPipeline m_PostProcessPipeline;
```

Expected: `RTXPTSample` owns lifetime and frame orchestration, while the pipeline owns future post-process pass status.

- [ ] **Step 2: Reset the pipeline with scene-dependent passes**

In `RTXPTSample::ResetSceneDependentResources`, add:

```cpp
m_PostProcessPipeline.Reset();
```

Expected: future post-process pass state cannot survive scene reloads accidentally.

- [ ] **Step 3: Initialize the skeleton when render passes are recreated**

In `RTXPTSample::CreatePhase4Passes`, after `m_BlitPass.Initialize(...)`, add:

```cpp
m_PostProcessPipeline.Initialize(m_pDevice,
                                 m_pEngineFactory,
                                 m_pSwapChain,
                                 m_FeatureCaps.ComputeShaders);
```

Expected: `CreatePhase4Passes` keeps its current name in P1, but now also creates the Phase 6 pipeline shell.

- [ ] **Step 4: Update render-target resize calls**

In both `RTXPTSample::EnsureRenderTargets` and `RTXPTSample::WindowResize`, replace the old `Resize` call that passes `TEX_FORMAT_RGBA8_UNORM` with:

```cpp
const RTXPTRenderTargetFormats Formats;
const bool Ok = m_RenderTargets.Resize(m_pDevice,
                                       SCDesc.Width,
                                       SCDesc.Height,
                                       Formats,
                                       m_FeatureCaps.ComputeShaders,
                                       m_FeatureCaps.RayTracing);
```

For `WindowResize`, use `Width` and `Height` instead of `SCDesc.Width` and `SCDesc.Height`.

Expected: `OutputColor` is no longer requested as `TEX_FORMAT_RGBA8_UNORM`.

- [ ] **Step 5: Validate the post-process resource graph after successful resize**

After a successful resize in `EnsureRenderTargets` and `WindowResize`, add:

```cpp
if (Ok)
    m_PostProcessPipeline.ValidateRenderTargets(m_RenderTargets);
```

Expected: P1 validates all core placeholders even though the frame still blits the temporary source.

- [ ] **Step 6: Update accumulation accessor calls**

Replace:

```cpp
m_RenderTargets.GetAccumColorUAV()
```

with:

```cpp
m_RenderTargets.GetAccumulatedRadianceUAV()
```

Expected: `RTXPTRayTracingPass::Trace` still receives the raygen accumulation UAV in P1, but the resource name now matches RTXPT-fork.

- [ ] **Step 7: Preserve the current render order**

Leave this frame-tail shape intact:

```cpp
const bool TraceExecuted = m_RayTracingPass.Trace(...);
const bool ComputeExecuted = m_EnableDebugComputePass && m_DebugComputePass.Dispatch(...);
ITextureView* pDisplaySRV = m_RenderTargets.GetDisplaySRV(ComputeExecuted);
m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pDisplaySRV);
```

Expected: P1 does not call a post-process render method and does not switch the normal display source to `LdrColor`.

- [ ] **Step 8: Update debug UI text**

Replace the old accumulation target status text:

```cpp
ImGui::Text("Accumulation target: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA8 fallback)");
```

with:

```cpp
ImGui::Text("AccumulatedRadiance: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA32F unavailable)");
ImGui::Text("Post-process targets: %s", m_RenderTargets.HasPostProcessTargets() ? "allocated" : "missing");
ImGui::Text("Post-process pipeline: %s", m_PostProcessPipeline.IsReady() ? "ready" : "not ready");
if (!m_PostProcessPipeline.GetStats().DisabledReason.empty())
    ImGui::TextWrapped("Post-process disabled: %s", m_PostProcessPipeline.GetStats().DisabledReason.c_str());
```

Expected: diagnostics no longer imply an `RGBA8` output fallback and P1 resource readiness is visible.

- [ ] **Step 9: Verify sample wiring**

Run:

```powershell
rg -n "RTXPTPostProcessPipeline|RTXPTRenderTargetFormats|GetAccumulatedRadianceUAV|HasPostProcessTargets|GetDisplaySRV\\(ComputeExecuted\\)" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
rg -n "GetAccumColor|TEX_FORMAT_RGBA8_UNORM,\\s*m_FeatureCaps" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
```

Expected: first command shows P1 wiring. Second command prints no matches.

### Task 6: Register the New Pipeline Files

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Add the new source file**

In `set(SOURCE ...)`, add:

```cmake
    src/RTXPTPostProcessPipeline.cpp
```

Expected: the `.cpp` file is compiled with the sample.

- [ ] **Step 2: Add the new header file**

In `set(INCLUDE ...)`, add:

```cmake
    src/RTXPTPostProcessPipeline.hpp
```

Expected: the header is tracked by the sample project.

- [ ] **Step 3: Verify CMake registration**

Run:

```powershell
rg -n "RTXPTPostProcessPipeline\\.(cpp|hpp)" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: one match for `.cpp` and one match for `.hpp`.

### Task 7: Boundary and Contract Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Verify P1 created the Phase 6 resource graph**

Run:

```powershell
rg -n "OutputColor|AccumulatedRadiance|ProcessedOutputColor|LdrColor|LdrColorScratch|ComputeColor" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.* DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: each core resource name appears in both the render-target owner and the P0 mapping.

- [ ] **Step 2: Verify P2/P3 behavior is still deferred**

Run:

```powershell
rg -n "u_AccumulationBuffer|ToneMapACES|exposureScale" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp
```

Expected: matches still exist. P1 should not remove raygen accumulation, ACES, or temporary exposure constants.

- [ ] **Step 3: Verify OutputColor is no longer RGBA8 in raygen**

Run:

```powershell
rg -n "VK_IMAGE_FORMAT\\(\"rgba16f\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
rg -n "VK_IMAGE_FORMAT\\(\"rgba8\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: first command reports three matches; second command reports no matches.

- [ ] **Step 4: Verify diagnostic compute remains RGBA8**

Run:

```powershell
rg -n "VK_IMAGE_FORMAT\\(\"rgba8\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTDebugCompute.csh
```

Expected: one match remains because `RTXPTComputePass` writes the optional display/debug target, not the HDR path-tracing `OutputColor`.

### Task 8: Build and Runtime Smoke

**Files:**
- Verify: configured Windows build tree
- Verify: RTXPT sample executable

- [ ] **Step 1: Build the RTXPT target**

Run:

```powershell
cmake --build build\x64\Debug --target RTXPT --config Debug
```

Expected: build completes without C++ compile errors or HLSL `VK_IMAGE_FORMAT` / storage image format mismatches. If the build tree does not exist, configure it using the repository's normal Debug CMake command before running this step.

- [ ] **Step 2: Locate the sample executable**

Run:

```powershell
$Exe = Get-ChildItem -Path build\x64\Debug -Recurse -Filter RTXPT.exe | Select-Object -First 1
$Exe.FullName
```

Expected: prints the full path to `RTXPT.exe`.

- [ ] **Step 3: Run D3D12 smoke**

Run:

```powershell
& $Exe.FullName --mode d3d12 --adapters_dialog 0 -w 1280 -h 720
```

Expected: the sample launches, renders through the existing blit path, and the debug UI shows `Post-process targets: allocated`. Close the window manually after one visible frame.

- [ ] **Step 4: Run Vulkan smoke**

Run:

```powershell
& $Exe.FullName --mode vk --adapters_dialog 0 -w 1280 -h 720
```

Expected: the sample launches and shows the same P1 resource status. If Vulkan is unavailable on the machine, record the exact startup error instead of marking Vulkan smoke as passed.

### Task 9: Review and Commit P1

**Files:**
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Review: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Review: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Review the submodule diff**

Run:

```powershell
git -C DiligentSamples diff -- Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/CMakeLists.txt
```

Expected: diff shows resource contract expansion, `rgba16f` `u_Output`, pipeline skeleton, sample wiring, diagnostics, and CMake registration. It must not show accumulation pass, tone-mapping pass, or post-process shader implementations.

- [ ] **Step 2: Commit DiligentSamples changes**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add post-processing p1 resource graph" -m "Co-Authored-By: GPT 5.5"
```

Expected: a DiligentSamples submodule commit is created.

- [ ] **Step 3: Commit the top-level submodule pointer**

Run:

```powershell
git add DiligentSamples
git commit -m "feat(rtxpt): advance post-processing phase p1" -m "Co-Authored-By: GPT 5.5"
```

Expected: a top-level commit records the submodule pointer update.

## Completion Verification

Run:

```powershell
rg -n "AccumulatedRadiance|ProcessedOutputColor|LdrColor|LdrColorScratch|RTXPTPostProcessPipeline" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
rg -n "VK_IMAGE_FORMAT\\(\"rgba16f\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
rg -n "VK_IMAGE_FORMAT\\(\"rgba8\"\\).*u_Output" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
rg -n "u_AccumulationBuffer|ToneMapACES|exposureScale" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
cmake --build build\x64\Debug --target RTXPT --config Debug
git -C DiligentSamples status --short
git status --short
```

Expected:

- The first command prints the P1 resource graph and pipeline skeleton.
- The second command prints three `rgba16f` `u_Output` matches.
- The third command prints no matches in `PathTracerSample.rgen`.
- The fourth command confirms P2/P3 raygen debt is still present.
- The build command completes successfully.
- `git -C DiligentSamples status --short` is clean after the submodule commit.
- `git status --short` is clean after the top-level commit, unless commits were intentionally skipped.

## P1 Handoff Criteria

P1 is complete when:

- `RTXPTRenderTargets` exposes RTXPT-fork-named `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, and `LdrColorScratch`.
- `OutputColor` is allocated as `TEX_FORMAT_RGBA16_FLOAT` and `PathTracerSample.rgen` declares `u_Output` as `rgba16f`.
- `AccumulatedRadiance` is allocated as `TEX_FORMAT_RGBA32_FLOAT` when supported.
- `ProcessedOutputColor` and the LDR ping-pong targets are allocated and validated, even though they are not yet used as the normal display chain.
- `RTXPTPostProcessPipeline.{hpp,cpp}` exists, is registered in CMake, and validates the P1 render-target graph.
- `RTXPTSample::Render` still presents through `RTXPTBlitPass` using `GetDisplaySRV(ComputeExecuted)`.
- `u_AccumulationBuffer`, raygen `ToneMapACES`, and `exposureScale` remain in place for P2/P3 to remove deliberately.
- No focused P2/P3/P4 pass implementation files are introduced in P1.
- D3D12 build passes, and D3D12/Vulkan smoke results are recorded honestly.

## Self-Review Notes

- Spec coverage: G1 is covered by Tasks 2-3 and 5; partial G5 is covered by Tasks 4-5 and the preserved final blit path; P1 verification is covered by Tasks 7-8.
- Placeholder scan: this plan contains concrete steps and explicit P2/P3/P4 phase boundaries, with no unresolved placeholder steps.
- Type/name consistency: the plan consistently uses `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, `LdrColorScratch`, `ComputeColor`, and `RTXPTPostProcessPipeline`.
