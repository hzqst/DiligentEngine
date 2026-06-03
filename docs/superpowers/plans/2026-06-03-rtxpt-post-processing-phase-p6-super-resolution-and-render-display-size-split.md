# RTXPT Post-Processing Phase P6 Super Resolution and Render/Display Size Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Diligent-native render-size/display-size split for RTXPT and integrate `ISuperResolution` as the optional temporal upscaling stage before HDR post-processing and tone mapping.

**Architecture:** Keep the P1-P5 `OutputColor -> accumulation -> ProcessedOutputColor -> LdrColor -> blit` chain as the direct fallback, where render size equals display size. When a temporal super-resolution variant is enabled, ray tracing and accumulation run at render size, accumulation writes a render-size HDR input texture, `RTXPTSuperResolutionPass` writes display-size HDR `ProcessedOutputColor`, and the existing HDR bloom/tone-mapping/presentation stages continue at display size. Motion-vector and depth resources are introduced as explicit render-size contracts; reference mode writes primary-hit depth and zero screen motion vectors, while future realtime/stable-plane work can replace the conservative motion data without changing the `ISuperResolution` binding contract.

**Tech Stack:** C++17, HLSL/DXC, Diligent Engine ray tracing, Diligent `ISuperResolutionFactory`/`ISuperResolution`, Diligent texture SRV/UAV/RTV views, ImGui, CMake sample target registration, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`, Diligent usage reference in `DiligentSamples/Tutorials/Tutorial27_PostProcessing`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P6.
- The current local RTXPT code has P1-P5 resource names and `RTXPTRenderTargets::GetPresentationSRV()`.
- `RTXPTRenderTargets` still stores a single `m_Width/m_Height`, so every texture is currently swapchain-sized.
- `RTXPTPostProcessPipeline::RunAccumulation()` writes directly to `ProcessedOutputColor`; P6 needs a render-size intermediate only when super resolution is active.
- `RTXPTSample::UpdateFrameConstants()` currently runs before `Render()->EnsureRenderTargets()`, so P6 must compute the intended render/display dimensions before writing frame constants.
- `PathTracerCameraData::Jitter` exists in C++ and HLSL but current raygen ignores it and instead uses a stateless random sub-pixel sample.
- `RTXPTRayTracingPass::Trace()` currently binds only `u_Output`; P6 must add render-size `u_Depth` and `u_ScreenMotionVectors` UAVs.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt` does not link `Diligent-SuperResolution-static`; Tutorial27 already does.
- `DiligentCore/Graphics/SuperResolution/interface/SuperResolution.h` requires temporal upscalers to receive color, depth, motion vectors, an output UAV, jitter, camera near/far/fov, time delta, and reset-history state.

## Source Anchors

Read these before editing:

- `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md:195` - P6 goal, touches, and technical direction.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp` and `.cpp` - current texture ownership and resize logic.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp` and `.cpp` - accumulation, bloom, and tone-mapping schedule.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` and `.cpp` - UI state, frame constants, resize, render order, presentation.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp` and `.cpp` - raygen resource layout and dispatch dimensions.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - raw HDR output, camera jitter, and primary-hit payload.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli` - `ComputeRayThinlens()` and ray direction construction.
- `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` and `assets/shaders/PathTracer/PathTracerShared.h` - C++/HLSL constant layout.
- `DiligentCore/Graphics/SuperResolution/interface/SuperResolution.h` and `SuperResolutionFactory.h` - Diligent super-resolution API contract.
- `DiligentSamples/Tutorials/Tutorial27_PostProcessing/src/Tutorial27_PostProcessing.cpp:264-375`, `637-685`, `950-989`, `1132-1151` - local factory, sizing, jitter, execute, and UI reference.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h:35-105` - upstream render/display-size and temporal target contract.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp:39-84`, `181-240` - upstream depth, motion-vector, `ProcessedOutputColor`, `TemporalFeedback1/2`, and `LdrColor` allocation.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1292-1314` - upstream accumulation/TAA/post-process pass construction.

## Scope Boundaries

- P6 must not port Streamline, DLSS-RR resource tagging, standalone NRD, stable-plane final merge, or RTXPT-fork's standalone TAA pass.
- P6 may enumerate spatial `ISuperResolution` variants, but the acceptance milestone is the temporal HDR path. Spatial variants are shown as unavailable for the P6 HDR temporal path unless a later task adds an explicit LDR spatial branch.
- If no temporal super-resolution variant is available, the sample must stay on the direct P1-P5 path with render size equal to display size and a visible disabled reason.
- P6 must not make `LdrColor` or swapchain presentation depend on super resolution. The final presentation source stays `RTXPTRenderTargets::GetPresentationSRV()`.
- Reference mode writes conservative guide data: primary-hit depth and zero screen-space motion vectors. Camera, scene, animation, material, light, env-map, render-size, variant, and quality changes reset accumulation and super-resolution history.
- Motion-vector quality for moving geometry and stable planes belongs to P7/P8. P6 only creates the resource and binding contract.
- `ProcessedOutputColor` becomes display-size in P6 when super resolution is active; direct mode keeps render size equal to display size.
- Do not change tone-mapping operators, bloom math, ray-tracing material/lighting behavior, or final blit shader behavior.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp` - settings, frame descriptor, stats, variant helpers, and pass interface.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp` - factory loading, variant enumeration, source-size query, upscaler creation/recreation, jitter query, and `ExecuteSuperResolutionAttribs` binding.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp` - render/display dimensions, P6 formats, render-size guide resources, super-resolution input texture, accessors, and explicit presentation-size accessors.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp` - split-size allocation, format support checks, and direct/SR output selection.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp` - own `RTXPTSuperResolutionPass`, expose variant/stats accessors, resolve dimensions, run SR, and update pipeline stats.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp` - initialize SR pass, validate P6 resources, write accumulation to the correct HDR target, execute SR before HDR post-process, and run display-size bloom/tone mapping.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp` - extend `Trace()` to accept depth and motion-vector UAVs.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` - add raygen dynamic variables for `u_Depth` and `u_ScreenMotionVectors`, bind them in `Trace()`, and dispatch render-size rays.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - write primary-hit depth and zero motion vectors, and use provider jitter when SR is active.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli` - keep ray helper signature stable; no file-wide redesign.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` and `assets/shaders/PathTracer/PathTracerShared.h` - rename one padding field to `superResolutionActive`, keep struct size unchanged, and use `PathTracerCameraData::Jitter`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - add SR UI state, target dimensions, current jitter, elapsed time, and history-reset state.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - update render/display sizing before frame constants, use render-size camera data, wire UI, trace guides, execute SR, and reset histories.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register `RTXPTSuperResolutionPass` and link `Diligent-SuperResolution-static`.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - map P6 resources and the Diligent `ISuperResolution` owner files.
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md` - add this P6 follow-up plan link.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `D:/RTXPT-fork/Rtxpt`
- Verify: `DiligentCore/Graphics/SuperResolution`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files must be inspected and preserved before editing.

- [ ] **Step 2: Confirm P1-P5 prerequisites**

Run:

```powershell
rg -n "GetPresentationSRV|RunAccumulation|RunPreToneMapping|RunToneMapping|ProcessedOutputColor|LdrColor" DiligentSamples/Samples/RTXPT/src
rg -n "RunPostToneMapping|LdrColorScratch|PostProcessEdgeDetection" DiligentSamples/Samples/RTXPT/src docs/superpowers/plans/2026-06-02-rtxpt-post-processing-phase-p4-bloom-and-ldr-post-process.md
```

Expected: P1-P5 direct chain symbols are present. If `RunPostToneMapping` is absent from source, complete the P4/P5 acceptance work before enabling P6 runtime paths; P6 files can still be prepared, but the P6 acceptance gate cannot pass until the base HDR-to-LDR chain is stable.

- [ ] **Step 3: Confirm super-resolution interfaces and example usage**

Run:

```powershell
rg -n "ISuperResolution|ExecuteSuperResolutionAttribs|GetSourceSettings|GetJitterOffset|LoadAndCreateSuperResolutionFactory" DiligentCore/Graphics/SuperResolution/interface DiligentSamples/Tutorials/Tutorial27_PostProcessing/src
```

Expected: interface declarations and Tutorial27 usage are found.

- [ ] **Step 4: Confirm upstream P6 resource anchors**

Run:

```powershell
rg -n "RenderSize|DisplaySize|TemporalFeedback1|TemporalFeedback2|CombinedHistoryClampRelax|ScreenMotionVectors|ProcessedOutputColor" D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp
```

Expected: upstream render/display split and temporal resources are found.

- [ ] **Step 5: Commit baseline status only if policy requires a checkpoint**

No source changes are made in Task 0. Do not create a commit unless the execution environment requires a preflight checkpoint.

### Task 1: Split RTXPT Render Targets Into Render and Display Dimensions

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h:35-105`
- Read: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp:39-84`, `181-240`

- [ ] **Step 1: Add P6 formats and dimensions**

In `RTXPTRenderTargets.hpp`, replace `RTXPTRenderTargetFormats` with:

```cpp
struct RTXPTRenderTargetFormats
{
    TEXTURE_FORMAT OutputColor                 = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT AccumulatedRadiance         = TEX_FORMAT_RGBA32_FLOAT;
    TEXTURE_FORMAT SuperResolutionInputColor   = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT ProcessedOutputColor        = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT LdrColor                    = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT ComputeColor                = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT Depth                       = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT ScreenMotionVectors         = TEX_FORMAT_RG16_FLOAT;
    TEXTURE_FORMAT TemporalFeedback            = TEX_FORMAT_RGBA16_SNORM;
    TEXTURE_FORMAT CombinedHistoryClampRelax   = TEX_FORMAT_R8_UNORM;
};

struct RTXPTRenderTargetDimensions
{
    Uint32 RenderWidth             = 1;
    Uint32 RenderHeight            = 1;
    Uint32 DisplayWidth            = 1;
    Uint32 DisplayHeight           = 1;
    bool   SuperResolutionActive   = false;

    bool IsValid() const
    {
        return RenderWidth > 0 && RenderHeight > 0 && DisplayWidth > 0 && DisplayHeight > 0;
    }

    bool operator==(const RTXPTRenderTargetDimensions& RHS) const
    {
        return RenderWidth == RHS.RenderWidth &&
            RenderHeight == RHS.RenderHeight &&
            DisplayWidth == RHS.DisplayWidth &&
            DisplayHeight == RHS.DisplayHeight &&
            SuperResolutionActive == RHS.SuperResolutionActive;
    }
};
```

Expected: P6 resource formats and both size domains are explicit in the render-target owner.

- [ ] **Step 2: Change the resize signature and add accessors**

In `RTXPTRenderTargets.hpp`, replace the current `Resize()` declaration and size accessors with:

```cpp
    bool Resize(IRenderDevice*                     pDevice,
                const RTXPTRenderTargetDimensions& Dimensions,
                const RTXPTRenderTargetFormats&    Formats,
                bool                               CreateComputeOutput,
                bool                               CreateAccumulatedRadiance);

    ITextureView* GetSuperResolutionInputColorUAV() const;
    ITextureView* GetSuperResolutionInputColorSRV() const;
    ITextureView* GetDepthUAV() const;
    ITextureView* GetDepthSRV() const;
    ITextureView* GetScreenMotionVectorsUAV() const;
    ITextureView* GetScreenMotionVectorsSRV() const;
    ITextureView* GetTemporalFeedback1UAV() const;
    ITextureView* GetTemporalFeedback1SRV() const;
    ITextureView* GetTemporalFeedback2UAV() const;
    ITextureView* GetTemporalFeedback2SRV() const;
    ITextureView* GetCombinedHistoryClampRelaxUAV() const;
    ITextureView* GetCombinedHistoryClampRelaxSRV() const;

    ITextureView* GetAccumulationOutputUAV() const;
    ITextureView* GetSuperResolutionColorSRV() const;
    ITextureView* GetSuperResolutionOutputUAV() const;

    Uint32 GetRenderWidth() const { return m_Dimensions.RenderWidth; }
    Uint32 GetRenderHeight() const { return m_Dimensions.RenderHeight; }
    Uint32 GetDisplayWidth() const { return m_Dimensions.DisplayWidth; }
    Uint32 GetDisplayHeight() const { return m_Dimensions.DisplayHeight; }
    Uint32 GetWidth() const { return GetRenderWidth(); }
    Uint32 GetHeight() const { return GetRenderHeight(); }
    bool   IsSuperResolutionActive() const { return m_Dimensions.SuperResolutionActive; }
    const RTXPTRenderTargetDimensions& GetDimensions() const { return m_Dimensions; }
    TEXTURE_FORMAT GetSuperResolutionInputColorFormat() const { return m_Formats.SuperResolutionInputColor; }
    TEXTURE_FORMAT GetDepthFormat() const { return m_Formats.Depth; }
    TEXTURE_FORMAT GetScreenMotionVectorsFormat() const { return m_Formats.ScreenMotionVectors; }
```

Expected: call sites can choose render-size or display-size explicitly. `GetWidth()` and `GetHeight()` remain transitional render-size aliases.

- [ ] **Step 3: Add P6 texture members**

In `RTXPTRenderTargets.hpp`, replace the existing private texture members and size fields with:

```cpp
    RefCntAutoPtr<ITexture>  m_OutputColor;
    RefCntAutoPtr<ITexture>  m_AccumulatedRadiance;
    RefCntAutoPtr<ITexture>  m_SuperResolutionInputColor;
    RefCntAutoPtr<ITexture>  m_ProcessedOutputColor;
    RefCntAutoPtr<ITexture>  m_LdrColor;
    RefCntAutoPtr<ITexture>  m_ComputeColor;
    RefCntAutoPtr<ITexture>  m_Depth;
    RefCntAutoPtr<ITexture>  m_ScreenMotionVectors;
    RefCntAutoPtr<ITexture>  m_TemporalFeedback1;
    RefCntAutoPtr<ITexture>  m_TemporalFeedback2;
    RefCntAutoPtr<ITexture>  m_CombinedHistoryClampRelax;
    bool                     m_AccumulatedRadianceUnavailable = false;
    RTXPTRenderTargetDimensions m_Dimensions                  = {};
    RTXPTRenderTargetFormats m_Formats                        = {};
```

Update `CreateTarget()` in the header to accept dimensions:

```cpp
    bool CreateTarget(IRenderDevice*           pDevice,
                      const char*              Name,
                      Uint32                   Width,
                      Uint32                   Height,
                      TEXTURE_FORMAT           TargetFormat,
                      BIND_FLAGS               BindFlags,
                      RefCntAutoPtr<ITexture>& Target);
```

Expected: render-size and display-size textures can be created by the same helper.

- [ ] **Step 4: Update format matching**

In `RTXPTRenderTargets.cpp`, update `FormatsMatch()` to include every P6 format:

```cpp
bool FormatsMatch(const RTXPTRenderTargetFormats& Lhs, const RTXPTRenderTargetFormats& Rhs)
{
    return Lhs.OutputColor == Rhs.OutputColor &&
        Lhs.AccumulatedRadiance == Rhs.AccumulatedRadiance &&
        Lhs.SuperResolutionInputColor == Rhs.SuperResolutionInputColor &&
        Lhs.ProcessedOutputColor == Rhs.ProcessedOutputColor &&
        Lhs.LdrColor == Rhs.LdrColor &&
        Lhs.ComputeColor == Rhs.ComputeColor &&
        Lhs.Depth == Rhs.Depth &&
        Lhs.ScreenMotionVectors == Rhs.ScreenMotionVectors &&
        Lhs.TemporalFeedback == Rhs.TemporalFeedback &&
        Lhs.CombinedHistoryClampRelax == Rhs.CombinedHistoryClampRelax;
}
```

Expected: resize cache invalidates when any P6 format changes.

- [ ] **Step 5: Reset every P6 texture**

In `RTXPTRenderTargets::Reset()`, release all P6 textures and reset dimensions:

```cpp
    m_OutputColor.Release();
    m_AccumulatedRadiance.Release();
    m_SuperResolutionInputColor.Release();
    m_ProcessedOutputColor.Release();
    m_LdrColor.Release();
    m_ComputeColor.Release();
    m_Depth.Release();
    m_ScreenMotionVectors.Release();
    m_TemporalFeedback1.Release();
    m_TemporalFeedback2.Release();
    m_CombinedHistoryClampRelax.Release();
    m_AccumulatedRadianceUnavailable = false;
    m_Dimensions                     = {};
    m_Formats                        = {};
```

Expected: resize or shutdown releases the full P6 resource graph.

- [ ] **Step 6: Update `CreateTarget()` implementation**

Replace the start of `RTXPTRenderTargets::CreateTarget()` with:

```cpp
bool RTXPTRenderTargets::CreateTarget(IRenderDevice*           pDevice,
                                      const char*              Name,
                                      Uint32                   Width,
                                      Uint32                   Height,
                                      TEXTURE_FORMAT           TargetFormat,
                                      BIND_FLAGS               BindFlags,
                                      RefCntAutoPtr<ITexture>& Target)
{
    TextureDesc Desc;
    Desc.Name      = Name;
    Desc.Type      = RESOURCE_DIM_TEX_2D;
    Desc.Width     = Width;
    Desc.Height    = Height;
    Desc.Format    = TargetFormat;
    Desc.BindFlags = BindFlags;
```

Expected: target dimensions are supplied per resource instead of using one stored size.

- [ ] **Step 7: Replace `Resize()` implementation with split-size allocation**

Replace `RTXPTRenderTargets::Resize()` with the split-size version below:

```cpp
bool RTXPTRenderTargets::Resize(IRenderDevice*                     pDevice,
                                const RTXPTRenderTargetDimensions& Dimensions,
                                const RTXPTRenderTargetFormats&    Formats,
                                bool                               CreateComputeOutput,
                                bool                               CreateAccumulatedRadiance)
{
    if (pDevice == nullptr || !Dimensions.IsValid())
        return false;

    const bool HasCorePostProcessTargets =
        m_OutputColor != nullptr &&
        m_ProcessedOutputColor != nullptr &&
        m_LdrColor != nullptr &&
        m_Depth != nullptr &&
        m_ScreenMotionVectors != nullptr &&
        (!Dimensions.SuperResolutionActive || m_SuperResolutionInputColor != nullptr);

    const bool HasRequestedTargets =
        HasCorePostProcessTargets &&
        (!CreateComputeOutput || m_ComputeColor != nullptr) &&
        (CreateComputeOutput || m_ComputeColor == nullptr) &&
        (!CreateAccumulatedRadiance || m_AccumulatedRadiance != nullptr || m_AccumulatedRadianceUnavailable) &&
        (CreateAccumulatedRadiance || m_AccumulatedRadiance == nullptr);

    if (HasRequestedTargets && m_Dimensions == Dimensions && FormatsMatch(m_Formats, Formats))
        return true;

    Reset();
    m_Dimensions = Dimensions;
    m_Formats    = Formats;

    const BIND_FLAGS HdrUavFlags       = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    const BIND_FLAGS HdrRtFlags        = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;
    const BIND_FLAGS LdrRtFlags        = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS | BIND_RENDER_TARGET;
    const BIND_FLAGS TemporalUavFlags  = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;

    if (!SupportsBindFlags(pDevice, m_Formats.OutputColor, HdrUavFlags))
    {
        LOG_ERROR_MESSAGE("HDR UAV OutputColor is not supported; RTXPT post-processing resource graph is unavailable");
        return false;
    }

    if (CreateAccumulatedRadiance && !SupportsBindFlags(pDevice, m_Formats.AccumulatedRadiance, HdrUavFlags))
    {
        LOG_ERROR_MESSAGE("RGBA32F UAV AccumulatedRadiance is not supported; reference accumulation is unavailable");
        m_AccumulatedRadianceUnavailable = true;
        return false;
    }

    if (m_Dimensions.SuperResolutionActive && !SupportsBindFlags(pDevice, m_Formats.SuperResolutionInputColor, HdrUavFlags))
    {
        LOG_ERROR_MESSAGE("HDR UAV SuperResolutionInputColor is not supported; temporal upscaling is unavailable");
        return false;
    }

    if (!SupportsBindFlags(pDevice, m_Formats.ProcessedOutputColor, HdrRtFlags))
    {
        LOG_ERROR_MESSAGE("HDR UAV/RTV ProcessedOutputColor is not supported; RTXPT post-processing resource graph is unavailable");
        return false;
    }

    if (!SupportsBindFlags(pDevice, m_Formats.LdrColor, LdrRtFlags))
    {
        LOG_ERROR_MESSAGE("LDR UAV/RTV targets are not supported; RTXPT post-processing resource graph is unavailable");
        return false;
    }

    if (!SupportsBindFlags(pDevice, m_Formats.Depth, TemporalUavFlags) ||
        !SupportsBindFlags(pDevice, m_Formats.ScreenMotionVectors, TemporalUavFlags) ||
        !SupportsBindFlags(pDevice, m_Formats.TemporalFeedback, TemporalUavFlags) ||
        !SupportsBindFlags(pDevice, m_Formats.CombinedHistoryClampRelax, TemporalUavFlags))
    {
        LOG_ERROR_MESSAGE("One or more P6 temporal guide formats are not supported");
        return false;
    }

    if (!CreateTarget(pDevice, "RTXPT OutputColor", m_Dimensions.RenderWidth, m_Dimensions.RenderHeight, m_Formats.OutputColor, HdrUavFlags, m_OutputColor))
        return false;

    if (CreateAccumulatedRadiance &&
        !m_AccumulatedRadianceUnavailable &&
        !CreateTarget(pDevice, "RTXPT AccumulatedRadiance", m_Dimensions.RenderWidth, m_Dimensions.RenderHeight, m_Formats.AccumulatedRadiance, HdrUavFlags, m_AccumulatedRadiance))
        return false;

    if (m_Dimensions.SuperResolutionActive &&
        !CreateTarget(pDevice, "RTXPT SuperResolutionInputColor", m_Dimensions.RenderWidth, m_Dimensions.RenderHeight, m_Formats.SuperResolutionInputColor, HdrUavFlags, m_SuperResolutionInputColor))
        return false;

    if (!CreateTarget(pDevice, "RTXPT ProcessedOutputColor", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.ProcessedOutputColor, HdrRtFlags, m_ProcessedOutputColor))
        return false;

    if (!CreateTarget(pDevice, "RTXPT LdrColor", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.LdrColor, LdrRtFlags, m_LdrColor))
        return false;

    if (!CreateTarget(pDevice, "RTXPT Depth", m_Dimensions.RenderWidth, m_Dimensions.RenderHeight, m_Formats.Depth, TemporalUavFlags, m_Depth))
        return false;

    if (!CreateTarget(pDevice, "RTXPT ScreenMotionVectors", m_Dimensions.RenderWidth, m_Dimensions.RenderHeight, m_Formats.ScreenMotionVectors, TemporalUavFlags, m_ScreenMotionVectors))
        return false;

    if (!CreateTarget(pDevice, "RTXPT TemporalFeedback1", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.TemporalFeedback, TemporalUavFlags, m_TemporalFeedback1))
        return false;

    if (!CreateTarget(pDevice, "RTXPT TemporalFeedback2", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.TemporalFeedback, TemporalUavFlags, m_TemporalFeedback2))
        return false;

    if (!CreateTarget(pDevice, "RTXPT CombinedHistoryClampRelax", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.CombinedHistoryClampRelax, TemporalUavFlags, m_CombinedHistoryClampRelax))
        return false;

    if (CreateComputeOutput &&
        !CreateTarget(pDevice, "RTXPT ComputeColor", m_Dimensions.DisplayWidth, m_Dimensions.DisplayHeight, m_Formats.ComputeColor, HdrUavFlags, m_ComputeColor))
        return false;

    return true;
}
```

Expected: render-size textures and display-size textures are allocated in the correct size domain.

- [ ] **Step 8: Add P6 accessors**

Add implementations in `RTXPTRenderTargets.cpp`:

```cpp
ITextureView* RTXPTRenderTargets::GetSuperResolutionInputColorUAV() const
{
    return m_SuperResolutionInputColor ? m_SuperResolutionInputColor->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetSuperResolutionInputColorSRV() const
{
    return m_SuperResolutionInputColor ? m_SuperResolutionInputColor->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDepthUAV() const
{
    return m_Depth ? m_Depth->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDepthSRV() const
{
    return m_Depth ? m_Depth->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetScreenMotionVectorsUAV() const
{
    return m_ScreenMotionVectors ? m_ScreenMotionVectors->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetScreenMotionVectorsSRV() const
{
    return m_ScreenMotionVectors ? m_ScreenMotionVectors->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetTemporalFeedback1UAV() const
{
    return m_TemporalFeedback1 ? m_TemporalFeedback1->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetTemporalFeedback1SRV() const
{
    return m_TemporalFeedback1 ? m_TemporalFeedback1->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetTemporalFeedback2UAV() const
{
    return m_TemporalFeedback2 ? m_TemporalFeedback2->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetTemporalFeedback2SRV() const
{
    return m_TemporalFeedback2 ? m_TemporalFeedback2->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetCombinedHistoryClampRelaxUAV() const
{
    return m_CombinedHistoryClampRelax ? m_CombinedHistoryClampRelax->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetCombinedHistoryClampRelaxSRV() const
{
    return m_CombinedHistoryClampRelax ? m_CombinedHistoryClampRelax->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetAccumulationOutputUAV() const
{
    return IsSuperResolutionActive() ? GetSuperResolutionInputColorUAV() : GetProcessedOutputColorUAV();
}

ITextureView* RTXPTRenderTargets::GetSuperResolutionColorSRV() const
{
    return IsSuperResolutionActive() ? GetSuperResolutionInputColorSRV() : GetProcessedOutputColorSRV();
}

ITextureView* RTXPTRenderTargets::GetSuperResolutionOutputUAV() const
{
    return GetProcessedOutputColorUAV();
}
```

Expected: accumulation and SR stages can use explicit accessors without knowing texture ownership details.

- [ ] **Step 9: Update `HasPostProcessTargets()`**

Replace `HasPostProcessTargets()` with:

```cpp
bool RTXPTRenderTargets::HasPostProcessTargets() const
{
    return m_OutputColor != nullptr &&
        m_AccumulatedRadiance != nullptr &&
        m_ProcessedOutputColor != nullptr &&
        m_LdrColor != nullptr &&
        m_Depth != nullptr &&
        m_ScreenMotionVectors != nullptr &&
        m_TemporalFeedback1 != nullptr &&
        m_TemporalFeedback2 != nullptr &&
        m_CombinedHistoryClampRelax != nullptr &&
        (!IsSuperResolutionActive() || m_SuperResolutionInputColor != nullptr);
}
```

Expected: validity includes P6 temporal contract resources.

- [ ] **Step 10: Run render-target grep**

Run:

```powershell
rg -n "RTXPTRenderTargetDimensions|SuperResolutionInputColor|GetRenderWidth|GetDisplayWidth|ScreenMotionVectors|TemporalFeedback|CombinedHistoryClampRelax|GetAccumulationOutputUAV" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.*
```

Expected: new P6 formats, dimensions, resources, and accessors are present.

- [ ] **Step 11: Commit render-target split**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp
git -C DiligentSamples commit -m "feat(rtxpt): split render and display targets" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only render-target ownership and accessor changes.

### Task 2: Add Raygen Depth, Motion-Vector, and SR Jitter Contract

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Rename one padding field for SR activation**

In `RTXPTFrameConstants.hpp`, replace the tail of `PathTracerConstants`:

```cpp
    Uint32 diffuseBounceCount       = 2; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    Uint32 nestedDielectricsQuality = 1; // Nested dielectrics quality: 0=Off, 1=Fast, 2=Quality.
    Uint32 _paddingR6_0             = 0;
    Uint32 _paddingR6_1             = 0;
```

with:

```cpp
    Uint32 diffuseBounceCount       = 2; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    Uint32 nestedDielectricsQuality = 1; // Nested dielectrics quality: 0=Off, 1=Fast, 2=Quality.
    Uint32 superResolutionActive    = 0; // P6: non-zero means camera.Jitter comes from ISuperResolution.
    Uint32 _paddingR6_1             = 0;
```

Keep the existing static asserts unchanged except for any assert that names `_paddingR6_0`.

Expected: struct size stays `80` bytes and one field carries the SR activation flag.

- [ ] **Step 2: Mirror the HLSL constant layout**

In `PathTracerShared.h`, replace:

```hlsl
    uint  diffuseBounceCount;       // R5/G9: max diffuse bounces and BSDF LD sampling window.
    uint  nestedDielectricsQuality; // R6/G10: 0=Off, 1=Fast, 2=Quality.
    uint  _paddingR6_0;
    uint  _paddingR6_1;
```

with:

```hlsl
    uint  diffuseBounceCount;       // R5/G9: max diffuse bounces and BSDF LD sampling window.
    uint  nestedDielectricsQuality; // R6/G10: 0=Off, 1=Fast, 2=Quality.
    uint  superResolutionActive;    // P6: non-zero means camera.Jitter comes from ISuperResolution.
    uint  _paddingR6_1;
```

Expected: C++ and HLSL names match without changing the byte layout.

- [ ] **Step 3: Add guide UAV declarations to every raygen branch**

In each branch of `PathTracerSample.rgen` that declares `u_Output`, add:

```hlsl
VK_IMAGE_FORMAT("r32f")  RWTexture2D<float>  u_Depth;
VK_IMAGE_FORMAT("rg16f") RWTexture2D<float2> u_ScreenMotionVectors;
```

Expected: screen-pattern, minimal-trace, and full path-tracing raygen variants expose the same dynamic resource names.

- [ ] **Step 4: Write diagnostic guide defaults**

In the `RTXPT_SCREEN_PATTERN_DIAGNOSTIC` branch, after writing `u_Output[pixel]`, add:

```hlsl
    u_Depth[pixel]                = 1.0;
    u_ScreenMotionVectors[pixel]  = float2(0.0, 0.0);
```

In the `RTXPT_MINIMAL_TRACE_RAY_DIAGNOSTIC` branch, after writing `u_Output[pixel]`, add:

```hlsl
    u_Depth[pixel]                = payload.hit != 0u ? ray.TMax : 1.0;
    u_ScreenMotionVectors[pixel]  = float2(0.0, 0.0);
```

Expected: diagnostic modes compile with the same bindings and produce valid guide textures.

- [ ] **Step 5: Use SR jitter in the full raygen path**

In the full path-tracing branch of `PathTracerSample.rgen`, replace:

```hlsl
    const float2    jitter          = sampleNext2D(sgCamera);
    const float2    subPixelOffset  = jitter - 0.5.xx;
    const float2    cameraDoFSample = sampleNext2D(sgCamera);
    CameraRay       cameraRay       = ComputeRayThinlens(g_Const.camera, pixel, subPixelOffset, cameraDoFSample);
```

with:

```hlsl
    const float2 randomJitter       = sampleNext2D(sgCamera) - 0.5.xx;
    const float2 srJitter           = g_Const.camera.Jitter;
    const float2 subPixelOffset     = g_Const.ptConsts.superResolutionActive != 0u ? srJitter : randomJitter;
    const float2 cameraDoFSample    = sampleNext2D(sgCamera);
    CameraRay    cameraRay          = ComputeRayThinlens(g_Const.camera, pixel, subPixelOffset, cameraDoFSample);
```

Expected: direct accumulation keeps existing stochastic AA; temporal SR uses provider jitter.

- [ ] **Step 6: Capture primary-hit depth in full raygen**

Before the path loop, add:

```hlsl
    float primaryDepth = g_Const.camera.FarZ;
```

Immediately after `TraceRay(...)` returns inside the path loop, add:

```hlsl
        if (bounce == 0u)
            primaryDepth = payload.hitFlag != 0u ? payload.hitDistance : g_Const.camera.FarZ;
```

At the final write near the end of the shader, replace:

```hlsl
    u_Output[pixel] = float4(pathRadiance, 1.0);
```

with:

```hlsl
    u_Output[pixel]               = float4(pathRadiance, 1.0);
    u_Depth[pixel]                = primaryDepth;
    u_ScreenMotionVectors[pixel]  = float2(0.0, 0.0);
```

Expected: reference mode provides primary-hit depth and conservative zero motion vectors.

- [ ] **Step 7: Extend the ray tracing pass trace signature**

In `RTXPTRayTracingPass.hpp`, replace `Trace()` with:

```cpp
    bool Trace(IDeviceContext* pContext,
               ITextureView*   pOutputUAV,
               ITextureView*   pDepthUAV,
               ITextureView*   pScreenMotionVectorsUAV,
               Uint32          Width,
               Uint32          Height);
```

Expected: C++ call sites must supply guide outputs.

- [ ] **Step 8: Add dynamic raygen variables**

In `RTXPTRayTracingPass.cpp`, replace the resource-layout line:

```cpp
        .AddVariable(SHADER_TYPE_RAY_GEN, "u_Output", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

with:

```cpp
        .AddVariable(SHADER_TYPE_RAY_GEN, "u_Output", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "u_Depth", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "u_ScreenMotionVectors", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

Expected: the raygen SRB can bind all P6 UAVs.

- [ ] **Step 9: Bind P6 UAVs in `Trace()`**

Replace the first part of `RTXPTRayTracingPass::Trace()` with:

```cpp
bool RTXPTRayTracingPass::Trace(IDeviceContext* pContext,
                                ITextureView*   pOutputUAV,
                                ITextureView*   pDepthUAV,
                                ITextureView*   pScreenMotionVectorsUAV,
                                Uint32          Width,
                                Uint32          Height)
{
    m_Stats.LastTraceExecuted = false;

    if (!IsReady())
        return false;
    if (pOutputUAV == nullptr || pDepthUAV == nullptr || pScreenMotionVectorsUAV == nullptr || Width == 0 || Height == 0)
        return false;

    IShaderResourceVariable* pOutputColorVar = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "u_Output");
    IShaderResourceVariable* pDepthVar       = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "u_Depth");
    IShaderResourceVariable* pMotionVar      = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "u_ScreenMotionVectors");
    if (pOutputColorVar == nullptr || pDepthVar == nullptr || pMotionVar == nullptr)
    {
        UNEXPECTED("Failed to find RTXPT raygen P6 output bindings");
        return false;
    }

    pOutputColorVar->Set(pOutputUAV);
    pDepthVar->Set(pDepthUAV);
    pMotionVar->Set(pScreenMotionVectorsUAV);
```

Keep the existing pipeline state, commit, `TraceRaysAttribs`, stats update, and return code after this block.

Expected: raygen writes raw HDR, depth, and motion vectors in the same render-size dispatch.

- [ ] **Step 10: Run guide-binding grep**

Run:

```powershell
rg -n "superResolutionActive|u_Depth|u_ScreenMotionVectors|primaryDepth|Trace\\(" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.*
```

Expected: constants, UAV declarations, shader writes, and extended trace signature are present.

- [ ] **Step 11: Commit raygen guide contract**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): output super resolution guides" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only frame-constant, raygen, and ray-tracing pass binding changes.

### Task 3: Add `RTXPTSuperResolutionPass`

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp`
- Read: `DiligentCore/Graphics/SuperResolution/interface/SuperResolution.h`
- Read: `DiligentCore/Graphics/SuperResolution/interface/SuperResolutionFactory.h`
- Read: `DiligentSamples/Tutorials/Tutorial27_PostProcessing/src/Tutorial27_PostProcessing.cpp:264-375`, `637-685`, `950-989`

- [ ] **Step 1: Create the pass header**

Create `RTXPTSuperResolutionPass.hpp`:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#pragma once

#include <string>
#include <vector>

#include "BasicMath.hpp"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "DeviceContext.h"
#include "SuperResolution.h"
#include "SuperResolutionFactory.h"
#include "RTXPTRenderTargets.hpp"

namespace Diligent
{

struct RTXPTSuperResolutionSettings
{
    bool                               Enabled          = false;
    Int32                              ActiveVariantIdx = 0;
    SUPER_RESOLUTION_OPTIMIZATION_TYPE OptimizationType = SUPER_RESOLUTION_OPTIMIZATION_TYPE_BALANCED;
    float                              Sharpness        = 1.0f;
};

struct RTXPTSuperResolutionFrameDesc
{
    RTXPTRenderTargetDimensions Dimensions;
    INTERFACE_ID                VariantId        = {};
    SUPER_RESOLUTION_TYPE       Type             = SUPER_RESOLUTION_TYPE_SPATIAL;
    TEXTURE_FORMAT              ColorFormat      = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT              DepthFormat      = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT              MotionFormat     = TEX_FORMAT_RG16_FLOAT;
    TEXTURE_FORMAT              OutputFormat     = TEX_FORMAT_RGBA16_FLOAT;
    bool                        Enabled          = false;
    bool                        Temporal         = false;
    bool                        ResetHistory     = false;
    float2                      Jitter           = float2{0.0f, 0.0f};
    float                       Sharpness        = 0.0f;
    float                       TimeDeltaSeconds = 0.0f;
};

struct RTXPTSuperResolutionStats
{
    bool        FactoryReady         = false;
    bool        UpscalerReady        = false;
    bool        LastExecute          = false;
    bool        LastFrameTemporal    = false;
    Uint32      VariantCount         = 0;
    Uint32      ExecuteCount         = 0;
    Uint32      RenderWidth          = 0;
    Uint32      RenderHeight         = 0;
    Uint32      DisplayWidth         = 0;
    Uint32      DisplayHeight        = 0;
    std::string DisabledReason;
};

class RTXPTSuperResolutionPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice);

    RTXPTSuperResolutionFrameDesc ResolveFrameDesc(const RTXPTSuperResolutionSettings& Settings,
                                                   Uint32                              DisplayWidth,
                                                   Uint32                              DisplayHeight,
                                                   TEXTURE_FORMAT                      OutputFormat,
                                                   bool                                ResetHistory,
                                                   float                               TimeDeltaSeconds);

    bool Execute(IDeviceContext*                         pContext,
                 const RTXPTRenderTargets&              RenderTargets,
                 const RTXPTSuperResolutionFrameDesc&   FrameDesc,
                 float                                  CameraNear,
                 float                                  CameraFar,
                 float                                  CameraFovAngleVert);

    const std::vector<SuperResolutionInfo>& GetVariants() const { return m_Variants; }
    const RTXPTSuperResolutionStats&        GetStats() const { return m_Stats; }
    bool                                    HasTemporalVariant() const;
    bool                                    SupportsSharpness(const SuperResolutionInfo& Info) const;

private:
    const SuperResolutionInfo* GetActiveVariant(Int32 ActiveVariantIdx) const;
    bool EnsureUpscaler(const RTXPTSuperResolutionFrameDesc& FrameDesc);
    SUPER_RESOLUTION_FLAGS GetFlags(const SuperResolutionInfo& Info, float Sharpness) const;

private:
    RefCntAutoPtr<IRenderDevice>              m_Device;
    RefCntAutoPtr<ISuperResolutionFactory>    m_Factory;
    RefCntAutoPtr<ISuperResolution>           m_Upscaler;
    std::vector<SuperResolutionInfo>          m_Variants;
    RTXPTSuperResolutionStats                 m_Stats;
};

} // namespace Diligent
```

Expected: the pass owns factory/upscaler state but render targets own textures.

- [ ] **Step 2: Create includes and reset/initialize implementation**

Create `RTXPTSuperResolutionPass.cpp` with:

```cpp
/*
 *  Copyright 2026 Diligent Graphics LLC
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 */

#include "RTXPTSuperResolutionPass.hpp"

#include <algorithm>

#include "DebugUtilities.hpp"
#include "SuperResolutionFactoryLoader.h"

namespace Diligent
{

void RTXPTSuperResolutionPass::Reset()
{
    m_Upscaler.Release();
    m_Factory.Release();
    m_Device.Release();
    m_Variants.clear();
    m_Stats = {};
}

bool RTXPTSuperResolutionPass::Initialize(IRenderDevice* pDevice)
{
    Reset();

    if (pDevice == nullptr)
    {
        m_Stats.DisabledReason = "Render device is null";
        return false;
    }

    m_Device = pDevice;
    LoadAndCreateSuperResolutionFactory(pDevice, &m_Factory);
    if (!m_Factory)
    {
        m_Stats.DisabledReason = "Diligent SuperResolution factory is unavailable";
        return true;
    }

    Uint32 NumVariants = 0;
    m_Factory->EnumerateVariants(NumVariants, nullptr);
    m_Variants.resize(NumVariants);
    if (NumVariants != 0)
        m_Factory->EnumerateVariants(NumVariants, m_Variants.data());

    m_Stats.FactoryReady = true;
    m_Stats.VariantCount = static_cast<Uint32>(m_Variants.size());
    if (m_Variants.empty())
        m_Stats.DisabledReason = "No super-resolution variants are available";
    return true;
}
```

Expected: missing factory or variants disables SR without failing the whole RTXPT sample.

- [ ] **Step 3: Add variant helpers**

Append to `RTXPTSuperResolutionPass.cpp`:

```cpp
const SuperResolutionInfo* RTXPTSuperResolutionPass::GetActiveVariant(Int32 ActiveVariantIdx) const
{
    if (m_Variants.empty())
        return nullptr;

    const Int32 ClampedIdx = std::clamp(ActiveVariantIdx, Int32{0}, static_cast<Int32>(m_Variants.size() - 1));
    return &m_Variants[static_cast<size_t>(ClampedIdx)];
}

bool RTXPTSuperResolutionPass::HasTemporalVariant() const
{
    for (const SuperResolutionInfo& Info : m_Variants)
    {
        if (Info.Type == SUPER_RESOLUTION_TYPE_TEMPORAL)
            return true;
    }
    return false;
}

bool RTXPTSuperResolutionPass::SupportsSharpness(const SuperResolutionInfo& Info) const
{
    return (Info.Type == SUPER_RESOLUTION_TYPE_SPATIAL &&
            (Info.SpatialCapFlags & SUPER_RESOLUTION_SPATIAL_CAP_FLAG_SHARPNESS) != 0) ||
        (Info.Type == SUPER_RESOLUTION_TYPE_TEMPORAL &&
         (Info.TemporalCapFlags & SUPER_RESOLUTION_TEMPORAL_CAP_FLAG_SHARPNESS) != 0);
}

SUPER_RESOLUTION_FLAGS RTXPTSuperResolutionPass::GetFlags(const SuperResolutionInfo& Info, float Sharpness) const
{
    SUPER_RESOLUTION_FLAGS Flags = Info.Type == SUPER_RESOLUTION_TYPE_TEMPORAL ?
        SUPER_RESOLUTION_FLAG_AUTO_EXPOSURE :
        SUPER_RESOLUTION_FLAG_NONE;

    if (Sharpness > 0.0f && SupportsSharpness(Info))
        Flags = Flags | SUPER_RESOLUTION_FLAG_ENABLE_SHARPENING;

    return Flags;
}
```

Expected: UI can safely clamp indices and only expose sharpness for supported variants.

- [ ] **Step 4: Resolve render/display dimensions**

Append `ResolveFrameDesc()`:

```cpp
RTXPTSuperResolutionFrameDesc RTXPTSuperResolutionPass::ResolveFrameDesc(const RTXPTSuperResolutionSettings& Settings,
                                                                         Uint32                              DisplayWidth,
                                                                         Uint32                              DisplayHeight,
                                                                         TEXTURE_FORMAT                      OutputFormat,
                                                                         bool                                ResetHistory,
                                                                         float                               TimeDeltaSeconds)
{
    RTXPTSuperResolutionFrameDesc FrameDesc;
    FrameDesc.Dimensions.DisplayWidth  = std::max(DisplayWidth, Uint32{1});
    FrameDesc.Dimensions.DisplayHeight = std::max(DisplayHeight, Uint32{1});
    FrameDesc.Dimensions.RenderWidth   = FrameDesc.Dimensions.DisplayWidth;
    FrameDesc.Dimensions.RenderHeight  = FrameDesc.Dimensions.DisplayHeight;
    FrameDesc.OutputFormat             = OutputFormat;
    FrameDesc.ResetHistory             = ResetHistory;
    FrameDesc.TimeDeltaSeconds         = TimeDeltaSeconds;
    FrameDesc.Sharpness                = std::clamp(Settings.Sharpness, 0.0f, 1.0f);

    m_Stats.LastFrameTemporal = false;
    m_Stats.RenderWidth       = FrameDesc.Dimensions.RenderWidth;
    m_Stats.RenderHeight      = FrameDesc.Dimensions.RenderHeight;
    m_Stats.DisplayWidth      = FrameDesc.Dimensions.DisplayWidth;
    m_Stats.DisplayHeight     = FrameDesc.Dimensions.DisplayHeight;
    m_Stats.DisabledReason.clear();

    if (!Settings.Enabled)
    {
        m_Stats.DisabledReason = "Super resolution is disabled";
        return FrameDesc;
    }

    if (!m_Factory || m_Variants.empty())
    {
        if (m_Stats.DisabledReason.empty())
            m_Stats.DisabledReason = "No super-resolution provider is available";
        return FrameDesc;
    }

    const SuperResolutionInfo* pInfo = GetActiveVariant(Settings.ActiveVariantIdx);
    if (pInfo == nullptr)
    {
        m_Stats.DisabledReason = "Selected super-resolution variant is invalid";
        return FrameDesc;
    }

    if (pInfo->Type != SUPER_RESOLUTION_TYPE_TEMPORAL)
    {
        m_Stats.DisabledReason = "Selected variant is spatial; P6 HDR path requires a temporal variant";
        return FrameDesc;
    }

    const SUPER_RESOLUTION_FLAGS Flags = GetFlags(*pInfo, FrameDesc.Sharpness);

    SuperResolutionSourceSettingsAttribs QueryAttribs;
    QueryAttribs.VariantId        = pInfo->VariantId;
    QueryAttribs.OutputWidth      = FrameDesc.Dimensions.DisplayWidth;
    QueryAttribs.OutputHeight     = FrameDesc.Dimensions.DisplayHeight;
    QueryAttribs.OutputFormat     = OutputFormat;
    QueryAttribs.Flags            = Flags;
    QueryAttribs.OptimizationType = Settings.OptimizationType;

    SuperResolutionSourceSettings SourceSettings;
    m_Factory->GetSourceSettings(QueryAttribs, SourceSettings);
    if (SourceSettings.OptimalInputWidth == 0 || SourceSettings.OptimalInputHeight == 0)
    {
        m_Stats.DisabledReason = "Selected temporal variant returned an invalid render size";
        return FrameDesc;
    }

    FrameDesc.Dimensions.RenderWidth             = SourceSettings.OptimalInputWidth;
    FrameDesc.Dimensions.RenderHeight            = SourceSettings.OptimalInputHeight;
    FrameDesc.Dimensions.SuperResolutionActive   = true;
    FrameDesc.Enabled                            = true;
    FrameDesc.Temporal                           = true;
    FrameDesc.VariantId                          = pInfo->VariantId;
    FrameDesc.Type                               = pInfo->Type;

    if (!EnsureUpscaler(FrameDesc))
    {
        FrameDesc.Dimensions.RenderWidth             = FrameDesc.Dimensions.DisplayWidth;
        FrameDesc.Dimensions.RenderHeight            = FrameDesc.Dimensions.DisplayHeight;
        FrameDesc.Dimensions.SuperResolutionActive   = false;
        FrameDesc.Enabled                            = false;
        FrameDesc.Temporal                           = false;
        FrameDesc.Jitter                             = float2{0.0f, 0.0f};
        return FrameDesc;
    }

    float JitterX = 0.0f;
    float JitterY = 0.0f;
    m_Upscaler->GetJitterOffset(m_Stats.ExecuteCount, JitterX, JitterY);
    FrameDesc.Jitter = float2{JitterX, JitterY};

    m_Stats.RenderWidth       = FrameDesc.Dimensions.RenderWidth;
    m_Stats.RenderHeight      = FrameDesc.Dimensions.RenderHeight;
    m_Stats.DisplayWidth      = FrameDesc.Dimensions.DisplayWidth;
    m_Stats.DisplayHeight     = FrameDesc.Dimensions.DisplayHeight;
    m_Stats.LastFrameTemporal = true;
    return FrameDesc;
}
```

Expected: direct fallback is automatic; temporal SR activates only with a valid temporal variant, source size, upscaler object, and provider jitter.

- [ ] **Step 5: Add upscaler creation**

Append `EnsureUpscaler()`:

```cpp
bool RTXPTSuperResolutionPass::EnsureUpscaler(const RTXPTSuperResolutionFrameDesc& FrameDesc)
{
    if (!FrameDesc.Enabled || !FrameDesc.Temporal)
        return true;
    if (!m_Factory)
        return false;

    bool NeedRecreate = !m_Upscaler;
    if (m_Upscaler)
    {
        const auto& Desc = m_Upscaler->GetDesc();
        NeedRecreate =
            Desc.VariantId != FrameDesc.VariantId ||
            Desc.InputWidth != FrameDesc.Dimensions.RenderWidth ||
            Desc.InputHeight != FrameDesc.Dimensions.RenderHeight ||
            Desc.OutputWidth != FrameDesc.Dimensions.DisplayWidth ||
            Desc.OutputHeight != FrameDesc.Dimensions.DisplayHeight ||
            Desc.ColorFormat != FrameDesc.ColorFormat ||
            Desc.DepthFormat != FrameDesc.DepthFormat ||
            Desc.MotionFormat != FrameDesc.MotionFormat ||
            Desc.OutputFormat != FrameDesc.OutputFormat;
    }

    if (!NeedRecreate)
        return true;

    const SuperResolutionInfo* pInfo = GetActiveVariant(0);
    for (const SuperResolutionInfo& Info : m_Variants)
    {
        if (Info.VariantId == FrameDesc.VariantId)
        {
            pInfo = &Info;
            break;
        }
    }
    if (pInfo == nullptr)
        return false;

    SuperResolutionDesc Desc;
    Desc.Name         = "RTXPT temporal super resolution";
    Desc.VariantId    = FrameDesc.VariantId;
    Desc.InputWidth   = FrameDesc.Dimensions.RenderWidth;
    Desc.InputHeight  = FrameDesc.Dimensions.RenderHeight;
    Desc.OutputWidth  = FrameDesc.Dimensions.DisplayWidth;
    Desc.OutputHeight = FrameDesc.Dimensions.DisplayHeight;
    Desc.OutputFormat = FrameDesc.OutputFormat;
    Desc.ColorFormat  = FrameDesc.ColorFormat;
    Desc.DepthFormat  = FrameDesc.DepthFormat;
    Desc.MotionFormat = FrameDesc.MotionFormat;
    Desc.Flags        = GetFlags(*pInfo, FrameDesc.Sharpness);

    m_Upscaler.Release();
    m_Factory->CreateSuperResolution(Desc, &m_Upscaler);
    m_Stats.UpscalerReady = m_Upscaler != nullptr;
    if (!m_Upscaler)
        m_Stats.DisabledReason = "Failed to create temporal super-resolution upscaler";
    return m_Upscaler != nullptr;
}
```

Expected: upscaler recreation follows variant, size, format, and output contract changes.

- [ ] **Step 6: Execute temporal super resolution**

Append `Execute()`:

```cpp
bool RTXPTSuperResolutionPass::Execute(IDeviceContext*                       pContext,
                                       const RTXPTRenderTargets&            RenderTargets,
                                       const RTXPTSuperResolutionFrameDesc& FrameDesc,
                                       float                                CameraNear,
                                       float                                CameraFar,
                                       float                                CameraFovAngleVert)
{
    m_Stats.LastExecute = false;
    if (!FrameDesc.Enabled)
        return true;

    if (!EnsureUpscaler(FrameDesc))
        return false;

    ExecuteSuperResolutionAttribs Attribs;
    Attribs.pContext              = pContext;
    Attribs.pColorTextureSRV      = RenderTargets.GetSuperResolutionColorSRV();
    Attribs.pDepthTextureSRV      = RenderTargets.GetDepthSRV();
    Attribs.pMotionVectorsSRV     = RenderTargets.GetScreenMotionVectorsSRV();
    Attribs.pOutputTextureView    = RenderTargets.GetSuperResolutionOutputUAV();
    Attribs.JitterX               = FrameDesc.Jitter.x;
    Attribs.JitterY               = FrameDesc.Jitter.y;
    Attribs.MotionVectorScaleX    = 1.0f;
    Attribs.MotionVectorScaleY    = 1.0f;
    Attribs.PreExposure           = 1.0f;
    Attribs.ExposureScale         = 1.0f;
    Attribs.Sharpness             = FrameDesc.Sharpness;
    Attribs.CameraNear            = CameraNear;
    Attribs.CameraFar             = CameraFar;
    Attribs.CameraFovAngleVert    = CameraFovAngleVert;
    Attribs.TimeDeltaInSeconds    = FrameDesc.TimeDeltaSeconds;
    Attribs.ResetHistory          = FrameDesc.ResetHistory ? True : False;
    Attribs.StateTransitionMode   = RESOURCE_STATE_TRANSITION_MODE_TRANSITION;

    if (!Attribs.pColorTextureSRV || !Attribs.pDepthTextureSRV || !Attribs.pMotionVectorsSRV || !Attribs.pOutputTextureView)
    {
        DEV_ERROR("RTXPT temporal super resolution resources are incomplete");
        return false;
    }

    m_Upscaler->Execute(Attribs);
    m_Stats.LastExecute = true;
    ++m_Stats.ExecuteCount;
    return true;
}
```

Expected: temporal execution binds HDR color, depth, motion vectors, output UAV, the same jitter stored in frame constants, camera metadata, and reset state.

- [ ] **Step 7: Run pass grep**

Run:

```powershell
rg -n "RTXPTSuperResolutionPass|ResolveFrameDesc|ExecuteSuperResolutionAttribs|LoadAndCreateSuperResolutionFactory|GetSourceSettings|GetJitterOffset" DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.*
```

Expected: factory load, variant query, source-size query, upscaler creation, jitter, and execute binding are present.

- [ ] **Step 8: Commit the SR pass**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSuperResolutionPass.hpp Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add super resolution pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only the new pass files.

### Task 4: Integrate Super Resolution Into the Post-Process Pipeline

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`

- [ ] **Step 1: Include and own the SR pass**

In `RTXPTPostProcessPipeline.hpp`, add:

```cpp
#include "RTXPTSuperResolutionPass.hpp"
```

Extend `RTXPTPostProcessPipelineStats`:

```cpp
    bool SuperResolutionStageReady = false;
    bool LastSuperResolutionActive = false;
```

Add public methods:

```cpp
    RTXPTSuperResolutionFrameDesc ResolveSuperResolutionFrameDesc(const RTXPTSuperResolutionSettings& Settings,
                                                                  Uint32                              DisplayWidth,
                                                                  Uint32                              DisplayHeight,
                                                                  TEXTURE_FORMAT                      OutputFormat,
                                                                  bool                                ResetHistory,
                                                                  float                               TimeDeltaSeconds);

    bool RunSuperResolution(IDeviceContext*                       pContext,
                            const RTXPTRenderTargets&            RenderTargets,
                            const RTXPTSuperResolutionFrameDesc& FrameDesc,
                            float                                CameraNear,
                            float                                CameraFar,
                            float                                CameraFovAngleVert);

    const RTXPTSuperResolutionPass& GetSuperResolutionPass() const { return m_SuperResolutionPass; }
```

Add the private member:

```cpp
    RTXPTSuperResolutionPass     m_SuperResolutionPass;
```

Expected: the post-process pipeline owns the SR stage between accumulation and HDR post-process.

- [ ] **Step 2: Reset and initialize SR**

In `RTXPTPostProcessPipeline::Reset()`, add:

```cpp
    m_SuperResolutionPass.Reset();
```

In `Initialize()`, after `m_Device = pDevice;`, add:

```cpp
    m_Stats.SuperResolutionStageReady = m_SuperResolutionPass.Initialize(pDevice);
    if (!m_Stats.SuperResolutionStageReady)
    {
        DEV_ERROR("RTXPT super-resolution pass failed to initialize");
        return false;
    }
```

Expected: factory absence is handled inside the pass and does not fail initialization.

- [ ] **Step 3: Validate P6 resources**

In `ValidateRenderTargets()`, include the P6 guide resources and accumulation output:

```cpp
        RenderTargets.GetAccumulationOutputUAV() != nullptr &&
        RenderTargets.GetDepthUAV() != nullptr &&
        RenderTargets.GetDepthSRV() != nullptr &&
        RenderTargets.GetScreenMotionVectorsUAV() != nullptr &&
        RenderTargets.GetScreenMotionVectorsSRV() != nullptr &&
        RenderTargets.GetTemporalFeedback1UAV() != nullptr &&
        RenderTargets.GetTemporalFeedback2UAV() != nullptr &&
        RenderTargets.GetCombinedHistoryClampRelaxUAV() != nullptr &&
        (!RenderTargets.IsSuperResolutionActive() || RenderTargets.GetSuperResolutionInputColorSRV() != nullptr);
```

Expected: incomplete SR resources are caught before render.

- [ ] **Step 4: Resolve SR frame descriptors**

Add to `RTXPTPostProcessPipeline.cpp`:

```cpp
RTXPTSuperResolutionFrameDesc RTXPTPostProcessPipeline::ResolveSuperResolutionFrameDesc(const RTXPTSuperResolutionSettings& Settings,
                                                                                        Uint32                              DisplayWidth,
                                                                                        Uint32                              DisplayHeight,
                                                                                        TEXTURE_FORMAT                      OutputFormat,
                                                                                        bool                                ResetHistory,
                                                                                        float                               TimeDeltaSeconds)
{
    return m_SuperResolutionPass.ResolveFrameDesc(Settings,
                                                  DisplayWidth,
                                                  DisplayHeight,
                                                  OutputFormat,
                                                  ResetHistory,
                                                  TimeDeltaSeconds);
}
```

Expected: `RTXPTSample` can determine render size before frame constants are uploaded.

- [ ] **Step 5: Make accumulation write the correct HDR target**

In `RunAccumulation()`, replace:

```cpp
    Dispatch.pProcessedOutputUAV     = RenderTargets.GetProcessedOutputColorUAV();
    Dispatch.InputWidth              = RenderTargets.GetWidth();
    Dispatch.InputHeight             = RenderTargets.GetHeight();
    Dispatch.OutputWidth             = RenderTargets.GetWidth();
    Dispatch.OutputHeight            = RenderTargets.GetHeight();
```

with:

```cpp
    Dispatch.pProcessedOutputUAV     = RenderTargets.GetAccumulationOutputUAV();
    Dispatch.InputWidth              = RenderTargets.GetRenderWidth();
    Dispatch.InputHeight             = RenderTargets.GetRenderHeight();
    Dispatch.OutputWidth             = RenderTargets.GetRenderWidth();
    Dispatch.OutputHeight            = RenderTargets.GetRenderHeight();
```

Expected: direct mode writes `ProcessedOutputColor`; SR mode writes `SuperResolutionInputColor`.

- [ ] **Step 6: Add `RunSuperResolution()`**

Add:

```cpp
bool RTXPTPostProcessPipeline::RunSuperResolution(IDeviceContext*                       pContext,
                                                  const RTXPTRenderTargets&            RenderTargets,
                                                  const RTXPTSuperResolutionFrameDesc& FrameDesc,
                                                  float                                CameraNear,
                                                  float                                CameraFar,
                                                  float                                CameraFovAngleVert)
{
    const bool Executed = m_SuperResolutionPass.Execute(pContext,
                                                        RenderTargets,
                                                        FrameDesc,
                                                        CameraNear,
                                                        CameraFar,
                                                        CameraFovAngleVert);
    m_Stats.SuperResolutionStageReady = true;
    m_Stats.LastSuperResolutionActive = FrameDesc.Enabled;
    if (!Executed)
        DEV_ERROR("RTXPT temporal super-resolution pass failed");
    return Executed;
}
```

Expected: direct fallback returns success without executing SR, while active SR writes display-size `ProcessedOutputColor`.

- [ ] **Step 7: Run display-size HDR stages**

In `RunPreToneMapping()`, replace width/height uses with display size:

```cpp
    if (BloomEnabled && !m_BloomPass.ResizeResources(m_Device, RenderTargets.GetDisplayWidth(), RenderTargets.GetDisplayHeight(), RenderTargets.GetProcessedOutputColorFormat()))
```

and:

```cpp
    BloomAttribs.Width      = RenderTargets.GetDisplayWidth();
    BloomAttribs.Height     = RenderTargets.GetDisplayHeight();
```

In `RunToneMapping()`, replace width/height uses with:

```cpp
    if (!m_ToneMappingPass.ResizeResources(m_Device, RenderTargets.GetDisplayWidth(), RenderTargets.GetDisplayHeight(), RenderTargets.GetProcessedOutputColorFormat()))
```

and:

```cpp
    Attribs.Width      = RenderTargets.GetDisplayWidth();
    Attribs.Height     = RenderTargets.GetDisplayHeight();
```

Expected: bloom and tone mapping operate on display-size `ProcessedOutputColor` and `LdrColor`.

- [ ] **Step 8: Run pipeline grep**

Run:

```powershell
rg -n "RTXPTSuperResolutionPass|ResolveSuperResolutionFrameDesc|RunSuperResolution|GetRenderWidth|GetDisplayWidth|GetAccumulationOutputUAV" DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.*
```

Expected: pipeline owns SR and all size-domain calls are explicit.

- [ ] **Step 9: Commit pipeline integration**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
git -C DiligentSamples commit -m "feat(rtxpt): schedule temporal super resolution" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only post-process pipeline integration.

### Task 5: Wire Sample UI, Sizing, Frame Constants, and Render Order

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add SR state to the sample header**

In `RTXPTSample.hpp`, add `RTXPTSuperResolutionSettings` to `RTXPTReferenceUIState` after bloom fields:

```cpp
    RTXPTSuperResolutionSettings SuperResolution;
```

Add private helper declarations:

```cpp
    void UpdateRenderTargetDimensions(float TimeDeltaSeconds);
    float2 GetCurrentSuperResolutionJitter() const;
```

Add private members:

```cpp
    RTXPTRenderTargetDimensions    m_CurrentTargetDimensions      = {};
    RTXPTSuperResolutionFrameDesc  m_CurrentSuperResolutionFrame  = {};
    float                          m_LastElapsedTimeSeconds       = 0.0f;
    bool                           m_ResetSuperResolutionHistory  = true;
```

Expected: sample state can carry the frame descriptor computed before constants upload.

- [ ] **Step 2: Compute render/display dimensions before frame constants**

Add to `RTXPTSample.cpp`:

```cpp
void RTXPTSample::UpdateRenderTargetDimensions(float TimeDeltaSeconds)
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    const RTXPTRenderTargetFormats Formats;

    m_CurrentSuperResolutionFrame =
        m_PostProcessPipeline.ResolveSuperResolutionFrameDesc(m_ReferenceUI.SuperResolution,
                                                              SCDesc.Width,
                                                              SCDesc.Height,
                                                              Formats.ProcessedOutputColor,
                                                              m_ResetSuperResolutionHistory || m_ResetAccumulationPending,
                                                              TimeDeltaSeconds);
    m_CurrentTargetDimensions = m_CurrentSuperResolutionFrame.Dimensions;
}

float2 RTXPTSample::GetCurrentSuperResolutionJitter() const
{
    return m_CurrentSuperResolutionFrame.Enabled ? m_CurrentSuperResolutionFrame.Jitter : float2{0.0f, 0.0f};
}
```

Then in `Update()`, before `UpdateFrameConstants(CurrTime);`, add:

```cpp
    m_LastElapsedTimeSeconds = static_cast<float>(ElapsedTime);
    UpdateRenderTargetDimensions(m_LastElapsedTimeSeconds);
```

Expected: frame constants use the same render size and SR state that `Render()` will allocate.

- [ ] **Step 3: Update camera data to use render viewport and display aspect**

Change `MakePathTracerCameraData()` signature:

```cpp
PathTracerCameraData MakePathTracerCameraData(const FirstPersonCamera& Camera,
                                              Uint32                   RenderWidth,
                                              Uint32                   RenderHeight,
                                              Uint32                   DisplayWidth,
                                              Uint32                   DisplayHeight,
                                              float                    FocalDistance,
                                              float                    ApertureRadius,
                                              const float2&            Jitter)
```

Inside it, replace width/height math with:

```cpp
    const float SafeRenderHeight  = static_cast<float>(std::max(RenderHeight, Uint32{1}));
    const float AspectRatio       = DisplayHeight > 0 ? static_cast<float>(DisplayWidth) / static_cast<float>(DisplayHeight) : 1.0f;
```

Replace `SafeHeight` uses with `SafeRenderHeight`, and set:

```cpp
    Data.ViewportWidth        = std::max(RenderWidth, Uint32{1});
    Data.ViewportHeight       = std::max(RenderHeight, Uint32{1});
    Data.Jitter               = Jitter;
```

Expected: raygen dispatches in render pixels but keeps the output display aspect.

- [ ] **Step 4: Update frame constants**

In `UpdateFrameConstants()`, replace swapchain width/height reads with:

```cpp
    const float Width  = static_cast<float>(m_CurrentTargetDimensions.RenderWidth);
    const float Height = static_cast<float>(m_CurrentTargetDimensions.RenderHeight);
```

Replace `MakePathTracerCameraData()` call with:

```cpp
    m_LastFrameConstants.camera = MakePathTracerCameraData(m_Camera,
                                                           m_CurrentTargetDimensions.RenderWidth,
                                                           m_CurrentTargetDimensions.RenderHeight,
                                                           m_CurrentTargetDimensions.DisplayWidth,
                                                           m_CurrentTargetDimensions.DisplayHeight,
                                                           m_ReferenceUI.CameraFocalDistance,
                                                           m_ReferenceUI.CameraAperture,
                                                           GetCurrentSuperResolutionJitter());
```

Set the SR activation flag:

```cpp
    m_LastFrameConstants.ptConsts.superResolutionActive = m_CurrentSuperResolutionFrame.Enabled ? 1u : 0u;
```

Expected: constants carry render viewport, display aspect, provider jitter, and SR activation state.

- [ ] **Step 5: Update render-target creation**

In `EnsureRenderTargets()`, replace old width/height locals with:

```cpp
    const RTXPTRenderTargetDimensions OldDimensions = m_RenderTargets.GetDimensions();
```

Replace the `Resize()` call with:

```cpp
    const bool Ok = m_RenderTargets.Resize(m_pDevice,
                                           m_CurrentTargetDimensions,
                                           Formats,
                                           CreateComputeOutput,
                                           m_FeatureCaps.RayTracing);
```

Replace resize-change comparisons with:

```cpp
         OldDimensions != m_CurrentTargetDimensions ||
```

Expected: render-target creation follows the frame descriptor chosen in `Update()`.

- [ ] **Step 6: Update trace and post-process render order**

In `Render()`, replace the trace call with:

```cpp
    const bool TraceExecuted =
        m_RayTracingPass.Trace(m_pImmediateContext,
                               m_RenderTargets.GetOutputColorUAV(),
                               m_RenderTargets.GetDepthUAV(),
                               m_RenderTargets.GetScreenMotionVectorsUAV(),
                               m_RenderTargets.GetRenderWidth(),
                               m_RenderTargets.GetRenderHeight());
```

After successful `RunAccumulation()` and before `RunPreToneMapping()`, add:

```cpp
    const bool SuperResolutionExecuted =
        m_PostProcessPipeline.RunSuperResolution(m_pImmediateContext,
                                                 m_RenderTargets,
                                                 m_CurrentSuperResolutionFrame,
                                                 m_CameraNearPlane,
                                                 m_CameraFarPlane,
                                                 m_CameraVerticalFov);
    if (!SuperResolutionExecuted)
    {
        ClearFallback(float4{0.2f, 0.4f, 1.0f, 1.0f});
        return;
    }
    if (m_CurrentSuperResolutionFrame.Enabled)
        m_ResetSuperResolutionHistory = false;
```

Expected render order: trace guides at render size, accumulation, optional temporal SR, HDR post-process, tone mapping, presentation.

- [ ] **Step 7: Update `WindowResize()`**

At the start of `WindowResize()` after `UpdateCameraProjection(Width, Height);`, add:

```cpp
    m_ResetSuperResolutionHistory = true;
    UpdateRenderTargetDimensions(m_LastElapsedTimeSeconds);
```

Replace the old `Resize()` call with the new dimensions version:

```cpp
    const bool Ok = m_RenderTargets.Resize(m_pDevice,
                                           m_CurrentTargetDimensions,
                                           Formats,
                                           CreateComputeOutput,
                                           m_FeatureCaps.RayTracing);
```

When recreating light-baker resources, keep display-size resources:

```cpp
                m_LightsBaker.CreateResources(m_pDevice, m_pEngineFactory, m_CurrentTargetDimensions.DisplayWidth, m_CurrentTargetDimensions.DisplayHeight, m_FeatureCaps.ComputeShaders);
```

Expected: swapchain resize resets SR history and keeps display-facing resources aligned.

- [ ] **Step 8: Reset SR history on invalidating changes**

In `RequestAccumulationReset()`, add:

```cpp
    m_ResetSuperResolutionHistory = true;
```

Expected: camera, scene, animation, material, light, env-map, and UI changes reset both accumulation and SR history.

- [ ] **Step 9: Add ImGui controls and diagnostics**

In the post-processing UI section after bloom controls, add:

```cpp
            const auto& SRPass     = m_PostProcessPipeline.GetSuperResolutionPass();
            const auto& SRVariants = SRPass.GetVariants();
            if (!SRVariants.empty())
            {
                if (ImGui::Checkbox("Enable Super Resolution", &m_ReferenceUI.SuperResolution.Enabled))
                    RequestAccumulationReset("Super resolution toggled");

                std::vector<const char*> VariantNames;
                VariantNames.reserve(SRVariants.size());
                for (const SuperResolutionInfo& Info : SRVariants)
                    VariantNames.push_back(Info.Name);

                if (ImGui::Combo("Super Resolution Mode",
                                 &m_ReferenceUI.SuperResolution.ActiveVariantIdx,
                                 VariantNames.data(),
                                 static_cast<int>(VariantNames.size())))
                {
                    RequestAccumulationReset("Super resolution mode changed");
                }

                const char* QualityNames[] = {"Max Quality", "High Quality", "Balanced", "High Performance", "Max Performance"};
                if (ImGui::Combo("Super Resolution Quality",
                                 reinterpret_cast<Int32*>(&m_ReferenceUI.SuperResolution.OptimizationType),
                                 QualityNames,
                                 IM_ARRAYSIZE(QualityNames)))
                {
                    RequestAccumulationReset("Super resolution quality changed");
                }

                const Int32 ActiveIdx = std::clamp(m_ReferenceUI.SuperResolution.ActiveVariantIdx, 0, static_cast<Int32>(SRVariants.size() - 1));
                if (SRPass.SupportsSharpness(SRVariants[static_cast<size_t>(ActiveIdx)]))
                    ImGui::SliderFloat("Super Resolution Sharpness", &m_ReferenceUI.SuperResolution.Sharpness, 0.0f, 1.0f);
            }
```

In diagnostics, add:

```cpp
        const auto& SRStats = m_PostProcessPipeline.GetSuperResolutionPass().GetStats();
        ImGui::Text("Super resolution: %s", SRStats.LastFrameTemporal ? "temporal" : "direct");
        ImGui::Text("SR input: %u x %u", SRStats.RenderWidth, SRStats.RenderHeight);
        ImGui::Text("SR output: %u x %u", SRStats.DisplayWidth, SRStats.DisplayHeight);
        if (!SRStats.DisabledReason.empty())
            ImGui::Text("SR disabled: %s", SRStats.DisabledReason.c_str());
```

Expected: users can enable a temporal provider when available and see direct fallback reasons.

- [ ] **Step 10: Run sample integration grep**

Run:

```powershell
rg -n "UpdateRenderTargetDimensions|m_CurrentTargetDimensions|m_CurrentSuperResolutionFrame|m_ResetSuperResolutionHistory|RunSuperResolution|GetRenderWidth|GetDisplayWidth|Enable Super Resolution|Super Resolution Mode" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
```

Expected: UI, sizing, render order, and history-reset state are present.

- [ ] **Step 11: Commit sample integration**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire super resolution controls" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only sample state, UI, frame-constant, resize, and render-order changes.

### Task 6: Register Build Dependencies and Documentation Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Register new source/header files**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add to `SOURCE`:

```cmake
    src/RTXPTSuperResolutionPass.cpp
```

Add to `INCLUDE`:

```cmake
    src/RTXPTSuperResolutionPass.hpp
```

Expected: the new pass compiles with the RTXPT target.

- [ ] **Step 2: Link the Diligent SuperResolution library**

In `target_link_libraries(RTXPT PRIVATE ...)`, add:

```cmake
    Diligent-SuperResolution-static
```

Expected: `LoadAndCreateSuperResolutionFactory()` and `ISuperResolution` implementations link like Tutorial27.

- [ ] **Step 3: Update mapping rows**

In `RTXPT_FORK_MAPPING.md`, add or update the P6 rows:

```markdown
| `SampleCommon/RenderTargets.h` `RenderSize` / `DisplaySize` | `src/RTXPTRenderTargets.{hpp,cpp}` `RTXPTRenderTargetDimensions` | P6 | Splits render-size raw/guide resources from display-size `ProcessedOutputColor`, `LdrColor`, and presentation. |
| `SampleCommon/RenderTargets.h` `TemporalFeedback1`, `TemporalFeedback2` | `src/RTXPTRenderTargets.{hpp,cpp}` | P6 | Display-size temporal feedback resources reserved for P6/P7 temporal contracts. |
| `SampleCommon/RenderTargets.h` `ScreenMotionVectors`, `Depth` | `src/RTXPTRenderTargets.{hpp,cpp}`, `assets/shaders/PathTracer/PathTracerSample.rgen` | P6 | Render-size depth and screen motion-vector inputs for `ISuperResolution`; reference mode writes primary depth and zero motion vectors. |
| `SampleCommon/RenderTargets.h` `CombinedHistoryClampRelax` | `src/RTXPTRenderTargets.{hpp,cpp}` | P6 | Display-size history-relax resource reserved for temporal AA/denoiser handoff. |
| RTXPT-fork DLSS/TAA scheduling hooks | `src/RTXPTSuperResolutionPass.{hpp,cpp}`, `src/RTXPTPostProcessPipeline.{hpp,cpp}` | P6 | Uses Diligent `ISuperResolution` instead of porting Donut TAA or Streamline in this phase. |
```

Expected: mapping names every P6 source anchor and Diligent destination.

- [ ] **Step 4: Add the P6 follow-up plan link to the spec**

In `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, update Phase P6 to include:

```markdown
- Follow-up plan: `docs/superpowers/plans/2026-06-03-rtxpt-post-processing-phase-p6-super-resolution-and-render-display-size-split.md`.
```

Expected: the spec points to this implementation plan.

- [ ] **Step 5: Run docs/build registration grep**

Run:

```powershell
rg -n "RTXPTSuperResolutionPass|Diligent-SuperResolution-static" DiligentSamples/Samples/RTXPT/CMakeLists.txt
rg -n "RTXPTRenderTargetDimensions|TemporalFeedback1|ScreenMotionVectors|CombinedHistoryClampRelax|ISuperResolution" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md
```

Expected: CMake, mapping, and spec all reference P6.

- [ ] **Step 6: Commit build and docs updates**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): map P6 super resolution resources" -m "Co-Authored-By: GPT 5.5"
git add docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md docs/superpowers/plans/2026-06-03-rtxpt-post-processing-phase-p6-super-resolution-and-render-display-size-split.md
git commit -m "docs(rtxpt): add P6 super resolution plan" -m "Co-Authored-By: GPT 5.5"
```

Expected: submodule commit contains CMake and mapping updates; top-level commit contains spec link and this plan.

### Task 7: Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`
- Verify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Verify: build output

- [ ] **Step 1: Run source-level P6 contract checks**

Run:

```powershell
rg -n "RTXPTRenderTargetDimensions|GetRenderWidth|GetDisplayWidth|SuperResolutionInputColor|TemporalFeedback1|TemporalFeedback2|CombinedHistoryClampRelax|ScreenMotionVectors" DiligentSamples/Samples/RTXPT/src
rg -n "u_Depth|u_ScreenMotionVectors|superResolutionActive|primaryDepth|camera.Jitter" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "RunAccumulation|RunSuperResolution|RunPreToneMapping|RunToneMapping|GetPresentationSRV|m_BlitPass.Render" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: render/display size resources, guide UAVs, SR activation, and render order are visible.

- [ ] **Step 2: Check for ambiguous size-domain use**

Run:

```powershell
rg -n "GetWidth\\(|GetHeight\\(" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
```

Expected: no matches in `RTXPTSample.cpp` or `RTXPTPostProcessPipeline.cpp`; those files must use `GetRenderWidth/Height` or `GetDisplayWidth/Height`.

- [ ] **Step 3: Check stale direct-only trace signature**

Run:

```powershell
rg -n "Trace\\([^\\n]*GetOutputColorUAV\\(\\)" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
rg -n "bool Trace\\(IDeviceContext\\* pContext,\\s*ITextureView\\*\\s+pOutputUAV,\\s*Uint32" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.*
```

Expected: first command shows the new multi-resource call only if the line wrapping includes guide UAVs nearby; second command has no matches.

- [ ] **Step 4: Build the RTXPT target**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: target builds without C++ or HLSL compile errors. If the build directory does not exist, run:

```powershell
.\build-x64-Debug.bat
```

Expected: configure/build completes and produces the RTXPT sample target.

- [ ] **Step 5: Runtime smoke with SR disabled on D3D12**

Launch the RTXPT sample on D3D12.

Expected:

1. The sample launches and presents the P5 `LdrColor` path.
2. UI shows `Super resolution: direct`.
3. SR input and output dimensions match the swapchain size.
4. Accumulation converges as before.
5. Bloom and tone mapping still run.

- [ ] **Step 6: Runtime smoke with SR disabled on Vulkan**

Launch the RTXPT sample on Vulkan.

Expected:

1. The sample launches and presents the P5 `LdrColor` path.
2. If no temporal SR provider is available, UI displays the disabled reason and stays direct.
3. No SR factory/provider absence crashes the sample.

- [ ] **Step 7: Runtime smoke with a temporal provider when available**

On a device/backend where `ISuperResolutionFactory` enumerates a temporal variant, enable Super Resolution.

Expected:

1. UI shows a render size smaller than or equal to display size.
2. `OutputColor`, `AccumulatedRadiance`, `SuperResolutionInputColor`, `Depth`, and `ScreenMotionVectors` are render-size resources.
3. `ProcessedOutputColor`, `TemporalFeedback1`, `TemporalFeedback2`, `CombinedHistoryClampRelax`, and `LdrColor` are display-size resources.
4. Render order is trace, accumulation, temporal SR, bloom, tone mapping, presentation.
5. Camera movement resets accumulation and SR history.
6. Disabling SR returns to direct display-size rendering without stale history artifacts.

- [ ] **Step 8: Compare against Tutorial27 API expectations**

Run:

```powershell
rg -n "GetSourceSettings|SuperResolutionDesc|ExecuteSuperResolutionAttribs|MotionVectorScaleX|CameraFovAngleVert|ResetHistory" DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp DiligentSamples/Tutorials/Tutorial27_PostProcessing/src/Tutorial27_PostProcessing.cpp
```

Expected: RTXPT uses the same Diligent API concepts as Tutorial27, with RTXPT-specific resource ownership.

- [ ] **Step 9: Final status check**

Run:

```powershell
git status --short
git -C DiligentSamples status --short
```

Expected: only deliberate P6 files are dirty if commits were not created during execution.

---

## Acceptance Gate

P6 is complete only when all of these are true:

- `RTXPTRenderTargets` stores explicit render and display dimensions.
- Direct fallback keeps render size equal to display size and preserves the P1-P5 presentation path.
- Temporal SR mode creates render-size `OutputColor`, `AccumulatedRadiance`, `SuperResolutionInputColor`, `Depth`, and `ScreenMotionVectors`.
- Temporal SR mode creates display-size `ProcessedOutputColor`, `TemporalFeedback1`, `TemporalFeedback2`, `CombinedHistoryClampRelax`, `LdrColor`, and presentation source.
- `RTXPTRayTracingPass::Trace()` binds and raygen writes `u_Output`, `u_Depth`, and `u_ScreenMotionVectors`.
- `PathTracerSample.rgen` uses `camera.Jitter` only when `ptConsts.superResolutionActive != 0`.
- `RTXPTSuperResolutionPass` loads the Diligent factory, enumerates variants, queries source size, creates `ISuperResolution`, and executes temporal upscaling through `ExecuteSuperResolutionAttribs`.
- `RTXPTPostProcessPipeline` render order is accumulation, optional temporal SR, HDR post-process, tone mapping.
- `RTXPTSample::UpdateFrameConstants()` uses render size for dispatch viewport and display size for camera aspect.
- SR history resets on camera, scene, animation, material, light, env-map, render-size, variant, and quality changes.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt` links `Diligent-SuperResolution-static`.
- `RTXPT_FORK_MAPPING.md` documents the P6 resource and pass mapping.
- RTXPT target builds, SR-disabled runtime smoke passes on D3D12 and Vulkan, and temporal SR smoke passes on a device/backend with a temporal provider.

## Self-Review Checklist

- Spec coverage: G6 is covered by render/display-size split, temporal resources, motion-vector/depth contracts, `ISuperResolution` factory/upscaler usage, jitter, execute binding, resize logic, direct fallback, and documentation mapping.
- Scope coverage: standalone RTXPT TAA, Streamline, DLSS-RR tagging, NRD, and stable-plane merge are not included.
- Type consistency: `RTXPTSuperResolutionSettings`, `RTXPTSuperResolutionFrameDesc`, `RTXPTRenderTargetDimensions`, and `RTXPTSuperResolutionPass` are used consistently across tasks.
- Resource consistency: render-size resources feed ray tracing, accumulation, and SR input; display-size resources feed HDR post-process, tone mapping, and final blit.
- Placeholder scan: this plan contains no open implementation blanks; provider absence is handled through explicit direct fallback and disabled reasons.
