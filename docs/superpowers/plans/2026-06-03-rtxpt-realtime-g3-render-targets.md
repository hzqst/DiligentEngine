# RTXPT Realtime G3 Render Targets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `RTXPTRenderTargets` with RTXPT-fork-compatible realtime stable-plane and NRD denoiser resources while keeping Reference mode render targets and presentation behavior intact.

**Architecture:** Keep ownership Diligent-native: `RTXPTRenderTargets` remains the single resource graph owner, with realtime resources requested explicitly through resize options. Stable-plane storage uses a CPU-mirrored `StablePlane` ABI and a Diligent structured buffer, while all guide, denoiser, and merge textures expose SRV/UAV accessors for later G4-G9 passes. Unsupported realtime formats fail closed during resize and are surfaced in status UI instead of letting stale resources look valid.

**Tech Stack:** C++17, Diligent `IRenderDevice`/`TextureDesc`/`BufferDesc`, Diligent `TEXTURE_FORMAT`, ImGui status UI, RTXPT realtime settings from G1, frame-constant stride values from G2, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`, goal `G3 - Realtime Render Targets`.
- G1/G2 state is already present in this checkout:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp` defines `kRTXPTStablePlaneCount = 3` and realtime UI state.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` defines `_activeStablePlaneCount`, `genericTSLineStride`, and `genericTSPlaneStride`.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` computes generic tiled-storage strides for constants.
- Current `RTXPTRenderTargets` owns only Phase 6 post-processing resources:
  - `OutputColor`, `AccumulatedRadiance`, `SuperResolutionInputColor`, `ProcessedOutputColor`, `LdrColor`, `ComputeColor`.
  - `Depth`, `ScreenMotionVectors`, `TemporalFeedback1`, `TemporalFeedback2`, `CombinedHistoryClampRelax`.
- Current `RTXPTRenderTargets::Resize()` is driven by two booleans: `CreateComputeOutput` and `CreateAccumulatedRadiance`. G3 adds explicit realtime resource requests without forcing them in Reference mode.
- Original RTXPT-fork anchors:
  - `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h`: resource names and ownership.
  - `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp`: resource formats, array sizes, half-res average radiance, and structured stable-plane buffer creation.
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`: `StablePlane` layout and generic tiled-storage addressing.
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`: `cStablePlaneCount = 3`.

## Scope Boundaries

- G3 allocates and exposes realtime resources. It does not dispatch realtime ray tracing, denoising guide passes, denoiser prepare/final-merge passes, NRD, or realtime AA/SR execution.
- G3 does not add DLSS-RR guide resources. The spec allows reserving them only when useful; this plan keeps G3 NRD-focused and leaves RR resources absent until the dedicated RR phase.
- G3 does not include NRD headers and does not create NRD instances. It prepares resize/reset status so G8 can destroy/recreate NRD state when render targets change.
- Reference mode must continue allocating only the existing post-process resource set unless `m_RealtimeUI.RealtimeMode` is enabled.
- Realtime mode may still be execution-disabled by G1 status text. G3's acceptance target is allocation/status readiness, not image output through stable planes.
- `DenoiserOutValidation` is optional and disabled by default. The resource exists only when an explicit resize option requests validation output.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
  - Promotes the generic tiled-storage tile size to a shared constant used by both frame constants and render-target buffer sizing.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
  - Adds `RTXPTStablePlaneData` with size/offset guards.
  - Adds realtime formats, resize options, status accessors, realtime texture/buffer members, and SRV/UAV accessors.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
  - Adds realtime format comparisons, failure status, texture-array creation, structured-buffer creation, stable-plane element count calculation, allocation, reset, validation, and accessor implementations.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Requests realtime render targets only when realtime mode is selected.
  - Marks realtime/NRD/TAA-SR caches invalid on render-target recreation.
  - Shows realtime resource status and last failure reason in the status UI.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  - Records G3 source-to-Diligent resource owner mapping.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h`
- Verify: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`

- [ ] **Step 1: Confirm dirty files before editing**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files are preserved. Do not remove or revert user changes.

- [ ] **Step 2: Confirm G3 fields are still absent from Diligent render targets**

Run:

```powershell
rg -n "StableRadiance|StablePlanesHeader|StablePlanesBuffer|Throughput|SpecularHitT|DenoiserViewspaceZ|DenoiserOutDiffRadianceHitDist|DenoiserAvgLayerRadianceHalfRes" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected before G3 implementation: no matches for these realtime resource names in `RTXPTRenderTargets`.

- [ ] **Step 3: Confirm upstream resource formats and stable-plane layout**

Run:

```powershell
rg -n "StableRadiance|StablePlanesHeader|StablePlanesBuffer|Throughput|DenoiserViewspaceZ|DenoiserNormalRoughness|DenoiserOutDiffRadianceHitDist|DenoiserAvgLayerRadianceHalfRes|struct StablePlane|cStablePlaneCount" D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h
```

Expected: upstream anchors show `cStablePlaneCount = 3`, `StablePlanesHeader` as a four-layer `R32_UINT` texture array, `DenoiserNormalRoughness` as packed normal/roughness, and `StablePlane` as an 80-byte structured element.

- [ ] **Step 4: No commit for preflight**

No source changes are made in Task 0. Do not create a commit for this task.

### Task 1: Add Shared Constants, Stable-Plane ABI, and Resize Options

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`

- [ ] **Step 1: Promote generic tiled-storage tile size**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`, add the shared tile-size constant next to the existing stable-plane constants:

```cpp
constexpr Uint32 kRTXPTStablePlaneCount           = 3;
constexpr Uint32 kRTXPTStablePlaneMaxVertexIndex  = 15;
constexpr Uint32 kRTXPTGenericTSTileSize          = 8;
```

Then remove the anonymous-namespace line below from `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`:

```cpp
constexpr Uint32      kRTXPTGenericTSTileSize      = 8u;
```

The existing `ComputeGenericTSLineStride()` and `ComputeGenericTSPlaneStride()` functions continue using `kRTXPTGenericTSTileSize`, now resolved from `RTXPTRealtimeSettings.hpp`.

- [ ] **Step 2: Add required includes to `RTXPTRenderTargets.hpp`**

Replace the include block in `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp` with:

```cpp
#include <array>
#include <string>

#include "BasicMath.hpp"
#include "Buffer.h"
#include "RenderDevice.h"
#include "RefCntAutoPtr.hpp"
#include "RTXPTRealtimeSettings.hpp"
#include "Texture.h"
#include "TextureView.h"
```

- [ ] **Step 3: Extend `RTXPTRenderTargetFormats`**

Replace the current `RTXPTRenderTargetFormats` declaration with:

```cpp
struct RTXPTRenderTargetFormats
{
    TEXTURE_FORMAT OutputColor               = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT AccumulatedRadiance       = TEX_FORMAT_RGBA32_FLOAT;
    TEXTURE_FORMAT SuperResolutionInputColor = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT ProcessedOutputColor      = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT LdrColor                  = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT ComputeColor              = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT Depth                     = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT ScreenMotionVectors       = TEX_FORMAT_RG16_FLOAT;
    TEXTURE_FORMAT TemporalFeedback          = TEX_FORMAT_RGBA16_SNORM;
    TEXTURE_FORMAT CombinedHistoryClampRelax = TEX_FORMAT_R8_UNORM;

    TEXTURE_FORMAT StableRadiance                   = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT StablePlanesHeader               = TEX_FORMAT_R32_UINT;
    TEXTURE_FORMAT Throughput                       = TEX_FORMAT_R32_UINT;
    TEXTURE_FORMAT SpecularHitT                     = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT ScratchFloat1                    = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT DenoiserViewspaceZ               = TEX_FORMAT_R32_FLOAT;
    TEXTURE_FORMAT DenoiserMotionVectors            = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT DenoiserNormalRoughness          = TEX_FORMAT_RGB10A2_UNORM;
    TEXTURE_FORMAT DenoiserDiffRadianceHitDist      = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT DenoiserSpecRadianceHitDist      = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT DenoiserDisocclusionThresholdMix = TEX_FORMAT_R8_UNORM;
    TEXTURE_FORMAT DenoiserOutDiffRadianceHitDist   = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT DenoiserOutSpecRadianceHitDist   = TEX_FORMAT_RGBA16_FLOAT;
    TEXTURE_FORMAT DenoiserOutValidation            = TEX_FORMAT_RGBA8_UNORM;
    TEXTURE_FORMAT DenoiserAvgLayerRadianceHalfRes  = TEX_FORMAT_RGBA16_FLOAT;
};
```

- [ ] **Step 4: Add the stable-plane CPU mirror and resize options**

Insert these declarations after `RTXPTRenderTargetDimensions`:

```cpp
struct RTXPTStablePlaneData
{
    float3 RayOrigin       = float3{0, 0, 0};
    float  LastRayTCurrent = 0.0f;
    float3 RayDir          = float3{0, 0, 1};
    float  SceneLength     = 0.0f;

    uint3  PackedThpAndMVs         = uint3{0, 0, 0};
    Uint32 VertexIndexAndRoughness = 0;
    uint3  DenoiserPackedBSDFEstimate = uint3{0, 0, 0};
    Uint32 PackedNormal               = 0;

    Uint32 PackedNoisyRadianceAndSpecAvg[2] = {};
    Uint32 FlagsAndVertexIndex              = 0;
    Uint32 PackedCounters                   = 0;
};
static_assert(sizeof(RTXPTStablePlaneData) == 80, "RTXPTStablePlaneData layout must match PathTracer/StablePlanes.hlsli StablePlane");
static_assert(offsetof(RTXPTStablePlaneData, RayDir) == 16, "RTXPTStablePlaneData RayDir offset must match StablePlane");
static_assert(offsetof(RTXPTStablePlaneData, PackedThpAndMVs) == 32, "RTXPTStablePlaneData PackedThpAndMVs offset must match StablePlane");
static_assert(offsetof(RTXPTStablePlaneData, DenoiserPackedBSDFEstimate) == 48, "RTXPTStablePlaneData DenoiserPackedBSDFEstimate offset must match StablePlane");
static_assert(offsetof(RTXPTStablePlaneData, PackedNoisyRadianceAndSpecAvg) == 64, "RTXPTStablePlaneData PackedNoisyRadianceAndSpecAvg offset must match StablePlane");

struct RTXPTRenderTargetCreateInfo
{
    RTXPTRenderTargetDimensions Dimensions;
    RTXPTRenderTargetFormats    Formats;
    bool                        CreateComputeOutput      = false;
    bool                        CreateAccumulatedRadiance = false;
    bool                        CreateRealtimeResources  = false;
    bool                        CreateDenoiserValidation = false;
};
```

- [ ] **Step 5: Add public realtime accessors and resize entry points**

In the public section of `RTXPTRenderTargets`, replace the two existing `Resize()` declarations with this set:

```cpp
    bool Resize(IRenderDevice* pDevice, const RTXPTRenderTargetCreateInfo& CreateInfo);
    bool Resize(IRenderDevice*                     pDevice,
                const RTXPTRenderTargetDimensions& Dimensions,
                const RTXPTRenderTargetFormats&    Formats,
                bool                               CreateComputeOutput,
                bool                               CreateAccumulatedRadiance);
    bool Resize(IRenderDevice*                  pDevice,
                Uint32                          Width,
                Uint32                          Height,
                const RTXPTRenderTargetFormats& Formats,
                bool                            CreateComputeOutput,
                bool                            CreateAccumulatedRadiance);
```

Then add these realtime status/accessors after `GetCombinedHistoryClampRelaxSRV()`:

```cpp
    bool        HasRealtimeRenderTargets() const;
    bool        AreRealtimeRenderTargetsRequested() const { return m_RealtimeResourcesRequested; }
    const char* GetLastFailureReason() const { return m_LastFailureReason.c_str(); }
    Uint64      GetStablePlanesElementCount() const { return m_StablePlanesElementCount; }

    ITextureView* GetStableRadianceUAV() const;
    ITextureView* GetStableRadianceSRV() const;
    ITextureView* GetStablePlanesHeaderUAV() const;
    ITextureView* GetStablePlanesHeaderSRV() const;
    IBufferView*  GetStablePlanesBufferUAV() const;
    IBufferView*  GetStablePlanesBufferSRV() const;
    IBuffer*      GetStablePlanesBuffer() const;
    ITextureView* GetThroughputUAV() const;
    ITextureView* GetThroughputSRV() const;
    ITextureView* GetSpecularHitTUAV() const;
    ITextureView* GetSpecularHitTSRV() const;
    ITextureView* GetScratchFloat1UAV() const;
    ITextureView* GetScratchFloat1SRV() const;
    ITextureView* GetDenoiserViewspaceZUAV() const;
    ITextureView* GetDenoiserViewspaceZSRV() const;
    ITextureView* GetDenoiserMotionVectorsUAV() const;
    ITextureView* GetDenoiserMotionVectorsSRV() const;
    ITextureView* GetDenoiserNormalRoughnessUAV() const;
    ITextureView* GetDenoiserNormalRoughnessSRV() const;
    ITextureView* GetDenoiserDiffRadianceHitDistUAV() const;
    ITextureView* GetDenoiserDiffRadianceHitDistSRV() const;
    ITextureView* GetDenoiserSpecRadianceHitDistUAV() const;
    ITextureView* GetDenoiserSpecRadianceHitDistSRV() const;
    ITextureView* GetDenoiserDisocclusionThresholdMixUAV() const;
    ITextureView* GetDenoiserDisocclusionThresholdMixSRV() const;
    ITextureView* GetDenoiserOutDiffRadianceHitDistUAV(Uint32 PlaneIndex) const;
    ITextureView* GetDenoiserOutDiffRadianceHitDistSRV(Uint32 PlaneIndex) const;
    ITextureView* GetDenoiserOutSpecRadianceHitDistUAV(Uint32 PlaneIndex) const;
    ITextureView* GetDenoiserOutSpecRadianceHitDistSRV(Uint32 PlaneIndex) const;
    ITextureView* GetDenoiserOutValidationUAV() const;
    ITextureView* GetDenoiserOutValidationSRV() const;
    ITextureView* GetDenoiserAvgLayerRadianceHalfResUAV() const;
    ITextureView* GetDenoiserAvgLayerRadianceHalfResSRV() const;
```

- [ ] **Step 6: Add private helper declarations and members**

In the private section of `RTXPTRenderTargets`, replace the current helper declarations with:

```cpp
    bool CreateTarget(IRenderDevice*           pDevice,
                      const char*              Name,
                      Uint32                   Width,
                      Uint32                   Height,
                      TEXTURE_FORMAT           TargetFormat,
                      BIND_FLAGS               BindFlags,
                      RefCntAutoPtr<ITexture>& Target,
                      RESOURCE_DIMENSION       Type      = RESOURCE_DIM_TEX_2D,
                      Uint32                   ArraySize = 1);
    bool CreateStablePlanesBuffer(IRenderDevice*          pDevice,
                                  Uint64                  ElementCount,
                                  RefCntAutoPtr<IBuffer>& Target);
    bool SupportsBindFlags(IRenderDevice* pDevice, TEXTURE_FORMAT TargetFormat, BIND_FLAGS BindFlags) const;
    bool FailResize(const char* Reason);
```

Then add the realtime members after `m_CombinedHistoryClampRelax`:

```cpp
    RefCntAutoPtr<ITexture> m_StableRadiance;
    RefCntAutoPtr<ITexture> m_StablePlanesHeader;
    RefCntAutoPtr<IBuffer>  m_StablePlanesBuffer;
    RefCntAutoPtr<ITexture> m_Throughput;
    RefCntAutoPtr<ITexture> m_SpecularHitT;
    RefCntAutoPtr<ITexture> m_ScratchFloat1;
    RefCntAutoPtr<ITexture> m_DenoiserViewspaceZ;
    RefCntAutoPtr<ITexture> m_DenoiserMotionVectors;
    RefCntAutoPtr<ITexture> m_DenoiserNormalRoughness;
    RefCntAutoPtr<ITexture> m_DenoiserDiffRadianceHitDist;
    RefCntAutoPtr<ITexture> m_DenoiserSpecRadianceHitDist;
    RefCntAutoPtr<ITexture> m_DenoiserDisocclusionThresholdMix;
    std::array<RefCntAutoPtr<ITexture>, kRTXPTStablePlaneCount> m_DenoiserOutDiffRadianceHitDist;
    std::array<RefCntAutoPtr<ITexture>, kRTXPTStablePlaneCount> m_DenoiserOutSpecRadianceHitDist;
    RefCntAutoPtr<ITexture> m_DenoiserOutValidation;
    RefCntAutoPtr<ITexture> m_DenoiserAvgLayerRadianceHalfRes;
```

Finally add these state fields before `m_Dimensions`:

```cpp
    bool        m_RealtimeResourcesRequested  = false;
    bool        m_DenoiserValidationRequested = false;
    Uint64      m_StablePlanesElementCount    = 0;
    std::string m_LastFailureReason;
```

- [ ] **Step 7: Build to catch header/API mistakes**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: compilation reaches `RTXPTRenderTargets.cpp` and reports missing implementations for the new declarations, or succeeds if later tasks have already been completed in the same batch.

- [ ] **Step 8: Commit Task 1**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): add realtime render target ABI" -m "Co-Authored-By: GPT 5.5"
```

### Task 2: Implement Resize Helpers, Failure State, and Format Matching

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add standard-library includes used by realtime helpers**

At the top of `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`, after the existing project includes, add:

```cpp
#include <algorithm>
```

- [ ] **Step 2: Replace `FormatsMatch()` with the expanded comparison**

Replace the existing `FormatsMatch()` body with:

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
        Lhs.CombinedHistoryClampRelax == Rhs.CombinedHistoryClampRelax &&
        Lhs.StableRadiance == Rhs.StableRadiance &&
        Lhs.StablePlanesHeader == Rhs.StablePlanesHeader &&
        Lhs.Throughput == Rhs.Throughput &&
        Lhs.SpecularHitT == Rhs.SpecularHitT &&
        Lhs.ScratchFloat1 == Rhs.ScratchFloat1 &&
        Lhs.DenoiserViewspaceZ == Rhs.DenoiserViewspaceZ &&
        Lhs.DenoiserMotionVectors == Rhs.DenoiserMotionVectors &&
        Lhs.DenoiserNormalRoughness == Rhs.DenoiserNormalRoughness &&
        Lhs.DenoiserDiffRadianceHitDist == Rhs.DenoiserDiffRadianceHitDist &&
        Lhs.DenoiserSpecRadianceHitDist == Rhs.DenoiserSpecRadianceHitDist &&
        Lhs.DenoiserDisocclusionThresholdMix == Rhs.DenoiserDisocclusionThresholdMix &&
        Lhs.DenoiserOutDiffRadianceHitDist == Rhs.DenoiserOutDiffRadianceHitDist &&
        Lhs.DenoiserOutSpecRadianceHitDist == Rhs.DenoiserOutSpecRadianceHitDist &&
        Lhs.DenoiserOutValidation == Rhs.DenoiserOutValidation &&
        Lhs.DenoiserAvgLayerRadianceHalfRes == Rhs.DenoiserAvgLayerRadianceHalfRes;
}
```

- [ ] **Step 3: Add generic tiled-storage element-count helpers**

Insert these helpers below `FormatsMatch()`:

```cpp
Uint64 ComputeGenericTSLineStride(Uint32 ImageWidth)
{
    const Uint64 SafeWidth  = std::max(ImageWidth, Uint32{1});
    const Uint64 TileCountX = (SafeWidth + kRTXPTGenericTSTileSize - 1u) / kRTXPTGenericTSTileSize;
    return TileCountX * kRTXPTGenericTSTileSize;
}

Uint64 ComputeGenericTSPlaneStride(Uint32 ImageWidth, Uint32 ImageHeight)
{
    const Uint64 SafeHeight = std::max(ImageHeight, Uint32{1});
    const Uint64 TileCountY = (SafeHeight + kRTXPTGenericTSTileSize - 1u) / kRTXPTGenericTSTileSize;
    return ComputeGenericTSLineStride(ImageWidth) * TileCountY * kRTXPTGenericTSTileSize;
}

Uint64 ComputeStablePlanesElementCount(Uint32 ImageWidth, Uint32 ImageHeight)
{
    return ComputeGenericTSPlaneStride(ImageWidth, ImageHeight) * kRTXPTStablePlaneCount;
}
```

- [ ] **Step 4: Extend `Reset()`**

In `RTXPTRenderTargets::Reset()`, add releases for all realtime resources after `m_CombinedHistoryClampRelax.Release();`:

```cpp
    m_StableRadiance.Release();
    m_StablePlanesHeader.Release();
    m_StablePlanesBuffer.Release();
    m_Throughput.Release();
    m_SpecularHitT.Release();
    m_ScratchFloat1.Release();
    m_DenoiserViewspaceZ.Release();
    m_DenoiserMotionVectors.Release();
    m_DenoiserNormalRoughness.Release();
    m_DenoiserDiffRadianceHitDist.Release();
    m_DenoiserSpecRadianceHitDist.Release();
    m_DenoiserDisocclusionThresholdMix.Release();
    for (auto& Texture : m_DenoiserOutDiffRadianceHitDist)
        Texture.Release();
    for (auto& Texture : m_DenoiserOutSpecRadianceHitDist)
        Texture.Release();
    m_DenoiserOutValidation.Release();
    m_DenoiserAvgLayerRadianceHalfRes.Release();
```

Then add state reset lines before `m_Dimensions = ...`:

```cpp
    m_RealtimeResourcesRequested  = false;
    m_DenoiserValidationRequested = false;
    m_StablePlanesElementCount    = 0;
    m_LastFailureReason.clear();
```

- [ ] **Step 5: Expand `CreateTarget()` for texture arrays**

Replace the `CreateTarget()` signature and initial descriptor setup with:

```cpp
bool RTXPTRenderTargets::CreateTarget(IRenderDevice*           pDevice,
                                      const char*              Name,
                                      Uint32                   Width,
                                      Uint32                   Height,
                                      TEXTURE_FORMAT           TargetFormat,
                                      BIND_FLAGS               BindFlags,
                                      RefCntAutoPtr<ITexture>& Target,
                                      RESOURCE_DIMENSION       Type,
                                      Uint32                   ArraySize)
{
    TextureDesc Desc;
    Desc.Name      = Name;
    Desc.Type      = Type;
    Desc.Width     = Width;
    Desc.Height    = Height;
    Desc.ArraySize = ArraySize;
    Desc.Format    = TargetFormat;
    Desc.BindFlags = BindFlags;
```

Keep the existing `Target.Release()`, `CreateTexture()`, and default-view validation logic after that descriptor setup.

- [ ] **Step 6: Add stable-plane buffer creation**

Insert this method after `CreateTarget()`:

```cpp
bool RTXPTRenderTargets::CreateStablePlanesBuffer(IRenderDevice*          pDevice,
                                                  Uint64                  ElementCount,
                                                  RefCntAutoPtr<IBuffer>& Target)
{
    BufferDesc Desc;
    Desc.Name              = "RTXPT StablePlanesBuffer";
    Desc.Usage             = USAGE_DEFAULT;
    Desc.BindFlags         = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(RTXPTStablePlaneData);
    Desc.Size              = std::max<Uint64>(ElementCount, Uint64{1}) * sizeof(RTXPTStablePlaneData);

    Target.Release();
    pDevice->CreateBuffer(Desc, nullptr, &Target);
    if (!Target)
    {
        LOG_ERROR_MESSAGE("Failed to create RTXPT StablePlanesBuffer");
        return false;
    }

    if (Target->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) == nullptr ||
        Target->GetDefaultView(BUFFER_VIEW_UNORDERED_ACCESS) == nullptr)
    {
        LOG_ERROR_MESSAGE("RTXPT StablePlanesBuffer is missing one or more required default views");
        Target.Release();
        return false;
    }

    return true;
}
```

- [ ] **Step 7: Add fail-closed resize status**

Insert this method after `SupportsBindFlags()`:

```cpp
bool RTXPTRenderTargets::FailResize(const char* Reason)
{
    std::string Failure = Reason != nullptr ? Reason : "RTXPT render target resize failed";
    Reset();
    m_LastFailureReason = Failure;
    LOG_ERROR_MESSAGE(m_LastFailureReason.c_str());
    return false;
}
```

- [ ] **Step 8: Commit Task 2**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
git commit -m "feat(rtxpt): add realtime target allocation helpers" -m "Co-Authored-By: GPT 5.5"
```

### Task 3: Allocate Optional Realtime Resource Set in `Resize()`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Convert the primary resize overload to `RTXPTRenderTargetCreateInfo`**

Replace the existing `Resize(IRenderDevice*, const RTXPTRenderTargetDimensions&, ...)` signature and opening lines with:

```cpp
bool RTXPTRenderTargets::Resize(IRenderDevice* pDevice, const RTXPTRenderTargetCreateInfo& CreateInfo)
{
    const RTXPTRenderTargetDimensions& Dimensions                = CreateInfo.Dimensions;
    const RTXPTRenderTargetFormats&    Formats                   = CreateInfo.Formats;
    const bool                         CreateComputeOutput       = CreateInfo.CreateComputeOutput;
    const bool                         CreateAccumulatedRadiance = CreateInfo.CreateAccumulatedRadiance;
    const bool                         CreateRealtimeResources   = CreateInfo.CreateRealtimeResources;
    const bool                         CreateDenoiserValidation  = CreateInfo.CreateDenoiserValidation;

    if (pDevice == nullptr)
        return FailResize("RTXPT render targets require a render device");
    if (!Dimensions.IsValid())
        return FailResize("RTXPT render target dimensions are invalid");
```

- [ ] **Step 2: Add realtime validity to reuse checks**

Inside the converted resize body, compute realtime validity before `HasRequestedTargets`:

```cpp
    const bool HasRealtimeTargets =
        !CreateRealtimeResources || HasRealtimeRenderTargets();
```

Then extend `HasRequestedTargets` with:

```cpp
        m_RealtimeResourcesRequested == CreateRealtimeResources &&
        m_DenoiserValidationRequested == CreateDenoiserValidation &&
        HasRealtimeTargets &&
```

The full `HasRequestedTargets` boolean must include the existing compute and accumulation checks plus the three realtime checks above.

- [ ] **Step 3: Replace unsupported-format returns with fail-closed returns**

For every existing unsupported format check, replace `return false;` with a specific `FailResize(...)` call. Use these exact reason strings:

```cpp
return FailResize("RGBA16F UAV OutputColor is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("RGBA32F UAV AccumulatedRadiance is not supported; reference accumulation is unavailable");
return FailResize("RGBA16F UAV SuperResolutionInputColor is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("R32F UAV Depth is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("RG16F UAV ScreenMotionVectors is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("HDR UAV/RTV ProcessedOutputColor is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("LDR UAV/RTV targets are not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("RGBA16_SNORM UAV TemporalFeedback is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("R8 UAV CombinedHistoryClampRelax is not supported; RTXPT post-processing resource graph is unavailable");
return FailResize("RGBA8 UAV ComputeColor is not supported; RTXPT post-processing resource graph is unavailable");
```

- [ ] **Step 4: Add realtime format checks**

After the existing compute-output format check, insert:

```cpp
    if (CreateRealtimeResources)
    {
        if (!SupportsBindFlags(pDevice, Formats.StableRadiance, UavFlags))
            return FailResize("RGBA16F UAV StableRadiance is not supported; RTXPT realtime resources are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.StablePlanesHeader, UavFlags))
            return FailResize("R32_UINT UAV StablePlanesHeader is not supported; RTXPT realtime resources are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.Throughput, UavFlags))
            return FailResize("R32_UINT UAV Throughput is not supported; RTXPT realtime resources are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.SpecularHitT, UavFlags))
            return FailResize("R32F UAV SpecularHitT is not supported; RTXPT realtime resources are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.ScratchFloat1, UavFlags))
            return FailResize("R32F UAV ScratchFloat1 is not supported; RTXPT realtime resources are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserViewspaceZ, UavFlags))
            return FailResize("R32F UAV DenoiserViewspaceZ is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserMotionVectors, UavFlags))
            return FailResize("RGBA16F UAV DenoiserMotionVectors is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserNormalRoughness, UavFlags))
            return FailResize("RGB10A2 UAV DenoiserNormalRoughness is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserDiffRadianceHitDist, UavFlags))
            return FailResize("RGBA16F UAV DenoiserDiffRadianceHitDist is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserSpecRadianceHitDist, UavFlags))
            return FailResize("RGBA16F UAV DenoiserSpecRadianceHitDist is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserDisocclusionThresholdMix, UavFlags))
            return FailResize("R8 UAV DenoiserDisocclusionThresholdMix is not supported; RTXPT realtime denoiser inputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserOutDiffRadianceHitDist, UavFlags))
            return FailResize("RGBA16F UAV DenoiserOutDiffRadianceHitDist is not supported; RTXPT realtime denoiser outputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserOutSpecRadianceHitDist, UavFlags))
            return FailResize("RGBA16F UAV DenoiserOutSpecRadianceHitDist is not supported; RTXPT realtime denoiser outputs are unavailable");
        if (!SupportsBindFlags(pDevice, Formats.DenoiserAvgLayerRadianceHalfRes, UavFlags))
            return FailResize("RGBA16F UAV DenoiserAvgLayerRadianceHalfRes is not supported; RTXPT realtime guide resources are unavailable");
        if (CreateDenoiserValidation && !SupportsBindFlags(pDevice, Formats.DenoiserOutValidation, UavFlags))
            return FailResize("RGBA8 UAV DenoiserOutValidation is not supported; RTXPT realtime NRD validation output is unavailable");
    }
```

- [ ] **Step 5: Add realtime local resources**

Add these local variables after the existing local target variables:

```cpp
    RefCntAutoPtr<ITexture> StableRadiance;
    RefCntAutoPtr<ITexture> StablePlanesHeader;
    RefCntAutoPtr<IBuffer>  StablePlanesBuffer;
    RefCntAutoPtr<ITexture> Throughput;
    RefCntAutoPtr<ITexture> SpecularHitT;
    RefCntAutoPtr<ITexture> ScratchFloat1;
    RefCntAutoPtr<ITexture> DenoiserViewspaceZ;
    RefCntAutoPtr<ITexture> DenoiserMotionVectors;
    RefCntAutoPtr<ITexture> DenoiserNormalRoughness;
    RefCntAutoPtr<ITexture> DenoiserDiffRadianceHitDist;
    RefCntAutoPtr<ITexture> DenoiserSpecRadianceHitDist;
    RefCntAutoPtr<ITexture> DenoiserDisocclusionThresholdMix;
    std::array<RefCntAutoPtr<ITexture>, kRTXPTStablePlaneCount> DenoiserOutDiffRadianceHitDist;
    std::array<RefCntAutoPtr<ITexture>, kRTXPTStablePlaneCount> DenoiserOutSpecRadianceHitDist;
    RefCntAutoPtr<ITexture> DenoiserOutValidation;
    RefCntAutoPtr<ITexture> DenoiserAvgLayerRadianceHalfRes;
```

- [ ] **Step 6: Allocate realtime resources**

After the existing optional `ComputeColor` creation block, insert:

```cpp
    const Uint32 HalfRenderWidth  = (RenderWidth + 1u) / 2u;
    const Uint32 HalfRenderHeight = (RenderHeight + 1u) / 2u;
    Uint64       StablePlanesElementCount = 0;

    if (CreateRealtimeResources)
    {
        StablePlanesElementCount = ComputeStablePlanesElementCount(RenderWidth, RenderHeight);

        if (!CreateTarget(pDevice, "RTXPT StableRadiance", RenderWidth, RenderHeight, Formats.StableRadiance, UavFlags, StableRadiance))
            return FailResize("Failed to create RTXPT StableRadiance");
        if (!CreateTarget(pDevice, "RTXPT StablePlanesHeader", RenderWidth, RenderHeight, Formats.StablePlanesHeader, UavFlags, StablePlanesHeader, RESOURCE_DIM_TEX_2D_ARRAY, 4))
            return FailResize("Failed to create RTXPT StablePlanesHeader");
        if (!CreateStablePlanesBuffer(pDevice, StablePlanesElementCount, StablePlanesBuffer))
            return FailResize("Failed to create RTXPT StablePlanesBuffer");
        if (!CreateTarget(pDevice, "RTXPT Throughput", RenderWidth, RenderHeight, Formats.Throughput, UavFlags, Throughput))
            return FailResize("Failed to create RTXPT Throughput");
        if (!CreateTarget(pDevice, "RTXPT SpecularHitT", RenderWidth, RenderHeight, Formats.SpecularHitT, UavFlags, SpecularHitT))
            return FailResize("Failed to create RTXPT SpecularHitT");
        if (!CreateTarget(pDevice, "RTXPT ScratchFloat1", RenderWidth, RenderHeight, Formats.ScratchFloat1, UavFlags, ScratchFloat1))
            return FailResize("Failed to create RTXPT ScratchFloat1");
        if (!CreateTarget(pDevice, "RTXPT DenoiserViewspaceZ", RenderWidth, RenderHeight, Formats.DenoiserViewspaceZ, UavFlags, DenoiserViewspaceZ))
            return FailResize("Failed to create RTXPT DenoiserViewspaceZ");
        if (!CreateTarget(pDevice, "RTXPT DenoiserMotionVectors", RenderWidth, RenderHeight, Formats.DenoiserMotionVectors, UavFlags, DenoiserMotionVectors))
            return FailResize("Failed to create RTXPT DenoiserMotionVectors");
        if (!CreateTarget(pDevice, "RTXPT DenoiserNormalRoughness", RenderWidth, RenderHeight, Formats.DenoiserNormalRoughness, UavFlags, DenoiserNormalRoughness))
            return FailResize("Failed to create RTXPT DenoiserNormalRoughness");
        if (!CreateTarget(pDevice, "RTXPT DenoiserDiffRadianceHitDist", RenderWidth, RenderHeight, Formats.DenoiserDiffRadianceHitDist, UavFlags, DenoiserDiffRadianceHitDist))
            return FailResize("Failed to create RTXPT DenoiserDiffRadianceHitDist");
        if (!CreateTarget(pDevice, "RTXPT DenoiserSpecRadianceHitDist", RenderWidth, RenderHeight, Formats.DenoiserSpecRadianceHitDist, UavFlags, DenoiserSpecRadianceHitDist))
            return FailResize("Failed to create RTXPT DenoiserSpecRadianceHitDist");
        if (!CreateTarget(pDevice, "RTXPT DenoiserDisocclusionThresholdMix", RenderWidth, RenderHeight, Formats.DenoiserDisocclusionThresholdMix, UavFlags, DenoiserDisocclusionThresholdMix))
            return FailResize("Failed to create RTXPT DenoiserDisocclusionThresholdMix");

        for (Uint32 PlaneIndex = 0; PlaneIndex < kRTXPTStablePlaneCount; ++PlaneIndex)
        {
            const std::string DiffName = "RTXPT DenoiserOutDiffRadianceHitDist[" + std::to_string(PlaneIndex) + "]";
            const std::string SpecName = "RTXPT DenoiserOutSpecRadianceHitDist[" + std::to_string(PlaneIndex) + "]";
            if (!CreateTarget(pDevice, DiffName.c_str(), RenderWidth, RenderHeight, Formats.DenoiserOutDiffRadianceHitDist, UavFlags, DenoiserOutDiffRadianceHitDist[PlaneIndex]))
                return FailResize("Failed to create RTXPT DenoiserOutDiffRadianceHitDist");
            if (!CreateTarget(pDevice, SpecName.c_str(), RenderWidth, RenderHeight, Formats.DenoiserOutSpecRadianceHitDist, UavFlags, DenoiserOutSpecRadianceHitDist[PlaneIndex]))
                return FailResize("Failed to create RTXPT DenoiserOutSpecRadianceHitDist");
        }

        if (CreateDenoiserValidation &&
            !CreateTarget(pDevice, "RTXPT DenoiserOutValidation", RenderWidth, RenderHeight, Formats.DenoiserOutValidation, UavFlags, DenoiserOutValidation))
            return FailResize("Failed to create RTXPT DenoiserOutValidation");

        if (!CreateTarget(pDevice, "RTXPT DenoiserAvgLayerRadianceHalfRes", HalfRenderWidth, HalfRenderHeight, Formats.DenoiserAvgLayerRadianceHalfRes, UavFlags, DenoiserAvgLayerRadianceHalfRes))
            return FailResize("Failed to create RTXPT DenoiserAvgLayerRadianceHalfRes");
    }
```

- [ ] **Step 7: Store realtime resources on success**

After existing member assignments for `m_CombinedHistoryClampRelax`, add:

```cpp
    m_StableRadiance                   = StableRadiance;
    m_StablePlanesHeader               = StablePlanesHeader;
    m_StablePlanesBuffer               = StablePlanesBuffer;
    m_Throughput                       = Throughput;
    m_SpecularHitT                     = SpecularHitT;
    m_ScratchFloat1                    = ScratchFloat1;
    m_DenoiserViewspaceZ               = DenoiserViewspaceZ;
    m_DenoiserMotionVectors            = DenoiserMotionVectors;
    m_DenoiserNormalRoughness          = DenoiserNormalRoughness;
    m_DenoiserDiffRadianceHitDist      = DenoiserDiffRadianceHitDist;
    m_DenoiserSpecRadianceHitDist      = DenoiserSpecRadianceHitDist;
    m_DenoiserDisocclusionThresholdMix = DenoiserDisocclusionThresholdMix;
    m_DenoiserOutDiffRadianceHitDist   = DenoiserOutDiffRadianceHitDist;
    m_DenoiserOutSpecRadianceHitDist   = DenoiserOutSpecRadianceHitDist;
    m_DenoiserOutValidation            = DenoiserOutValidation;
    m_DenoiserAvgLayerRadianceHalfRes  = DenoiserAvgLayerRadianceHalfRes;
```

Then add state assignments before `return true;`:

```cpp
    m_RealtimeResourcesRequested  = CreateRealtimeResources;
    m_DenoiserValidationRequested = CreateDenoiserValidation;
    m_StablePlanesElementCount    = StablePlanesElementCount;
    m_LastFailureReason.clear();
```

- [ ] **Step 8: Recreate the wrapper overloads**

Replace the existing wrapper overload bodies with:

```cpp
bool RTXPTRenderTargets::Resize(IRenderDevice*                     pDevice,
                                const RTXPTRenderTargetDimensions& Dimensions,
                                const RTXPTRenderTargetFormats&    Formats,
                                bool                               CreateComputeOutput,
                                bool                               CreateAccumulatedRadiance)
{
    RTXPTRenderTargetCreateInfo CreateInfo;
    CreateInfo.Dimensions                 = Dimensions;
    CreateInfo.Formats                    = Formats;
    CreateInfo.CreateComputeOutput        = CreateComputeOutput;
    CreateInfo.CreateAccumulatedRadiance  = CreateAccumulatedRadiance;
    return Resize(pDevice, CreateInfo);
}

bool RTXPTRenderTargets::Resize(IRenderDevice*                  pDevice,
                                Uint32                          Width,
                                Uint32                          Height,
                                const RTXPTRenderTargetFormats& Formats,
                                bool                            CreateComputeOutput,
                                bool                            CreateAccumulatedRadiance)
{
    RTXPTRenderTargetCreateInfo CreateInfo;
    CreateInfo.Dimensions.RenderWidth           = Width;
    CreateInfo.Dimensions.RenderHeight          = Height;
    CreateInfo.Dimensions.DisplayWidth          = Width;
    CreateInfo.Dimensions.DisplayHeight         = Height;
    CreateInfo.Dimensions.SuperResolutionActive = false;
    CreateInfo.Formats                          = Formats;
    CreateInfo.CreateComputeOutput              = CreateComputeOutput;
    CreateInfo.CreateAccumulatedRadiance        = CreateAccumulatedRadiance;
    return Resize(pDevice, CreateInfo);
}
```

- [ ] **Step 9: Commit Task 3**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
git commit -m "feat(rtxpt): allocate realtime render targets" -m "Co-Authored-By: GPT 5.5"
```

### Task 4: Implement Realtime Validation and Accessors

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add realtime validity check**

Insert this method after `HasPostProcessTargets()`:

```cpp
bool RTXPTRenderTargets::HasRealtimeRenderTargets() const
{
    if (!m_RealtimeResourcesRequested)
        return false;

    const bool HasPerPlaneOutputs =
        std::all_of(m_DenoiserOutDiffRadianceHitDist.begin(), m_DenoiserOutDiffRadianceHitDist.end(),
                    [](const RefCntAutoPtr<ITexture>& Texture) { return Texture != nullptr; }) &&
        std::all_of(m_DenoiserOutSpecRadianceHitDist.begin(), m_DenoiserOutSpecRadianceHitDist.end(),
                    [](const RefCntAutoPtr<ITexture>& Texture) { return Texture != nullptr; });

    return m_StableRadiance != nullptr &&
        m_StablePlanesHeader != nullptr &&
        m_StablePlanesBuffer != nullptr &&
        m_Throughput != nullptr &&
        m_SpecularHitT != nullptr &&
        m_ScratchFloat1 != nullptr &&
        m_DenoiserViewspaceZ != nullptr &&
        m_DenoiserMotionVectors != nullptr &&
        m_DenoiserNormalRoughness != nullptr &&
        m_DenoiserDiffRadianceHitDist != nullptr &&
        m_DenoiserSpecRadianceHitDist != nullptr &&
        m_DenoiserDisocclusionThresholdMix != nullptr &&
        HasPerPlaneOutputs &&
        (!m_DenoiserValidationRequested || m_DenoiserOutValidation != nullptr) &&
        m_DenoiserAvgLayerRadianceHalfRes != nullptr &&
        m_StablePlanesElementCount > 0;
}
```

- [ ] **Step 2: Add small view helpers**

Insert these anonymous-namespace helpers before the accessor implementations:

```cpp
namespace
{

ITextureView* GetTextureUAV(const RefCntAutoPtr<ITexture>& Texture)
{
    return Texture ? Texture->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* GetTextureSRV(const RefCntAutoPtr<ITexture>& Texture)
{
    return Texture ? Texture->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

IBufferView* GetBufferUAV(const RefCntAutoPtr<IBuffer>& Buffer)
{
    return Buffer ? Buffer->GetDefaultView(BUFFER_VIEW_UNORDERED_ACCESS) : nullptr;
}

IBufferView* GetBufferSRV(const RefCntAutoPtr<IBuffer>& Buffer)
{
    return Buffer ? Buffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
}

} // namespace
```

Then it is fine to leave existing accessors unchanged or mechanically replace repeated ternaries with these helpers. Do not change public behavior for existing resources.

- [ ] **Step 3: Implement realtime texture and buffer accessors**

Append these methods after `GetCombinedHistoryClampRelaxSRV()`:

```cpp
ITextureView* RTXPTRenderTargets::GetStableRadianceUAV() const { return GetTextureUAV(m_StableRadiance); }
ITextureView* RTXPTRenderTargets::GetStableRadianceSRV() const { return GetTextureSRV(m_StableRadiance); }
ITextureView* RTXPTRenderTargets::GetStablePlanesHeaderUAV() const { return GetTextureUAV(m_StablePlanesHeader); }
ITextureView* RTXPTRenderTargets::GetStablePlanesHeaderSRV() const { return GetTextureSRV(m_StablePlanesHeader); }
IBufferView*  RTXPTRenderTargets::GetStablePlanesBufferUAV() const { return GetBufferUAV(m_StablePlanesBuffer); }
IBufferView*  RTXPTRenderTargets::GetStablePlanesBufferSRV() const { return GetBufferSRV(m_StablePlanesBuffer); }
IBuffer*      RTXPTRenderTargets::GetStablePlanesBuffer() const { return m_StablePlanesBuffer; }
ITextureView* RTXPTRenderTargets::GetThroughputUAV() const { return GetTextureUAV(m_Throughput); }
ITextureView* RTXPTRenderTargets::GetThroughputSRV() const { return GetTextureSRV(m_Throughput); }
ITextureView* RTXPTRenderTargets::GetSpecularHitTUAV() const { return GetTextureUAV(m_SpecularHitT); }
ITextureView* RTXPTRenderTargets::GetSpecularHitTSRV() const { return GetTextureSRV(m_SpecularHitT); }
ITextureView* RTXPTRenderTargets::GetScratchFloat1UAV() const { return GetTextureUAV(m_ScratchFloat1); }
ITextureView* RTXPTRenderTargets::GetScratchFloat1SRV() const { return GetTextureSRV(m_ScratchFloat1); }
ITextureView* RTXPTRenderTargets::GetDenoiserViewspaceZUAV() const { return GetTextureUAV(m_DenoiserViewspaceZ); }
ITextureView* RTXPTRenderTargets::GetDenoiserViewspaceZSRV() const { return GetTextureSRV(m_DenoiserViewspaceZ); }
ITextureView* RTXPTRenderTargets::GetDenoiserMotionVectorsUAV() const { return GetTextureUAV(m_DenoiserMotionVectors); }
ITextureView* RTXPTRenderTargets::GetDenoiserMotionVectorsSRV() const { return GetTextureSRV(m_DenoiserMotionVectors); }
ITextureView* RTXPTRenderTargets::GetDenoiserNormalRoughnessUAV() const { return GetTextureUAV(m_DenoiserNormalRoughness); }
ITextureView* RTXPTRenderTargets::GetDenoiserNormalRoughnessSRV() const { return GetTextureSRV(m_DenoiserNormalRoughness); }
ITextureView* RTXPTRenderTargets::GetDenoiserDiffRadianceHitDistUAV() const { return GetTextureUAV(m_DenoiserDiffRadianceHitDist); }
ITextureView* RTXPTRenderTargets::GetDenoiserDiffRadianceHitDistSRV() const { return GetTextureSRV(m_DenoiserDiffRadianceHitDist); }
ITextureView* RTXPTRenderTargets::GetDenoiserSpecRadianceHitDistUAV() const { return GetTextureUAV(m_DenoiserSpecRadianceHitDist); }
ITextureView* RTXPTRenderTargets::GetDenoiserSpecRadianceHitDistSRV() const { return GetTextureSRV(m_DenoiserSpecRadianceHitDist); }
ITextureView* RTXPTRenderTargets::GetDenoiserDisocclusionThresholdMixUAV() const { return GetTextureUAV(m_DenoiserDisocclusionThresholdMix); }
ITextureView* RTXPTRenderTargets::GetDenoiserDisocclusionThresholdMixSRV() const { return GetTextureSRV(m_DenoiserDisocclusionThresholdMix); }

ITextureView* RTXPTRenderTargets::GetDenoiserOutDiffRadianceHitDistUAV(Uint32 PlaneIndex) const
{
    return PlaneIndex < m_DenoiserOutDiffRadianceHitDist.size() ? GetTextureUAV(m_DenoiserOutDiffRadianceHitDist[PlaneIndex]) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDenoiserOutDiffRadianceHitDistSRV(Uint32 PlaneIndex) const
{
    return PlaneIndex < m_DenoiserOutDiffRadianceHitDist.size() ? GetTextureSRV(m_DenoiserOutDiffRadianceHitDist[PlaneIndex]) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDenoiserOutSpecRadianceHitDistUAV(Uint32 PlaneIndex) const
{
    return PlaneIndex < m_DenoiserOutSpecRadianceHitDist.size() ? GetTextureUAV(m_DenoiserOutSpecRadianceHitDist[PlaneIndex]) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDenoiserOutSpecRadianceHitDistSRV(Uint32 PlaneIndex) const
{
    return PlaneIndex < m_DenoiserOutSpecRadianceHitDist.size() ? GetTextureSRV(m_DenoiserOutSpecRadianceHitDist[PlaneIndex]) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDenoiserOutValidationUAV() const { return GetTextureUAV(m_DenoiserOutValidation); }
ITextureView* RTXPTRenderTargets::GetDenoiserOutValidationSRV() const { return GetTextureSRV(m_DenoiserOutValidation); }
ITextureView* RTXPTRenderTargets::GetDenoiserAvgLayerRadianceHalfResUAV() const { return GetTextureUAV(m_DenoiserAvgLayerRadianceHalfRes); }
ITextureView* RTXPTRenderTargets::GetDenoiserAvgLayerRadianceHalfResSRV() const { return GetTextureSRV(m_DenoiserAvgLayerRadianceHalfRes); }
```

- [ ] **Step 4: Build to verify accessors and default views**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: `RTXPT` compiles. If the build directory does not exist, configure it with the repo's normal Debug command before retrying the build.

- [ ] **Step 5: Commit Task 4**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
git commit -m "feat(rtxpt): expose realtime render target views" -m "Co-Authored-By: GPT 5.5"
```

### Task 5: Request Realtime Resources from `RTXPTSample`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add a helper to build render-target create info**

Insert this helper in the anonymous namespace near `MakeFeatureCaps()`:

```cpp
RTXPTRenderTargetCreateInfo MakeRenderTargetCreateInfo(const RTXPTRenderTargetDimensions& Dimensions,
                                                       const RTXPTRealtimeSettings&       RealtimeUI,
                                                       bool                               RayTracingAvailable)
{
    RTXPTRenderTargetCreateInfo CreateInfo;
    CreateInfo.Dimensions                 = Dimensions;
    CreateInfo.Formats                    = {};
    CreateInfo.CreateComputeOutput        = false;
    CreateInfo.CreateAccumulatedRadiance  = RayTracingAvailable;
    CreateInfo.CreateRealtimeResources    = RealtimeUI.RealtimeMode;
    CreateInfo.CreateDenoiserValidation   = false;
    return CreateInfo;
}
```

- [ ] **Step 2: Use the helper in `EnsureRenderTargets()`**

Replace the current local `Formats`, `CreateComputeOutput`, and `m_RenderTargets.Resize(...)` call in `RTXPTSample::EnsureRenderTargets()` with:

```cpp
    SanitizeRealtimeSettings(m_RealtimeUI);
    const RTXPTRenderTargetCreateInfo CreateInfo =
        MakeRenderTargetCreateInfo(m_CurrentTargetDimensions, m_RealtimeUI, m_FeatureCaps.RayTracing);
    const bool Ok = m_RenderTargets.Resize(m_pDevice, CreateInfo);
```

Leave `ResourcesValid` and `m_AccumulationActive` computation in place.

- [ ] **Step 3: Expand render-target recreation reset flags**

In `EnsureRenderTargets()`, replace the existing reset block with:

```cpp
    if (Ok && ResourcesValid &&
        (!WasValid ||
         !(OldDimensions == m_CurrentTargetDimensions) ||
         WasAccumulationActive != m_AccumulationActive))
    {
        RequestAccumulationReset("Render targets (re)created");
        RequestRealtimeReset(RTXPT_REALTIME_RESET_RENDER_TARGET_RECREATE |
                                 RTXPT_REALTIME_RESET_REALTIME_CACHES |
                                 RTXPT_REALTIME_RESET_NRD_HISTORY |
                                 RTXPT_REALTIME_RESET_TAA_SR_HISTORY,
                             "Render targets (re)created");
        InvalidatePreviousFrameConstants();
    }
```

- [ ] **Step 4: Use the helper in `WindowResize()`**

Replace the current local `Formats`, `CreateComputeOutput`, and `m_RenderTargets.Resize(...)` call in `RTXPTSample::WindowResize()` with:

```cpp
    SanitizeRealtimeSettings(m_RealtimeUI);
    const RTXPTRenderTargetCreateInfo CreateInfo =
        MakeRenderTargetCreateInfo(m_CurrentTargetDimensions, m_RealtimeUI, m_FeatureCaps.RayTracing);
    const bool Ok = m_RenderTargets.Resize(m_pDevice, CreateInfo);
```

Then replace the existing successful resize reset line:

```cpp
        RequestAccumulationReset("Window resized");
```

with:

```cpp
        RequestAccumulationReset("Window resized");
        RequestRealtimeReset(RTXPT_REALTIME_RESET_RENDER_TARGET_RECREATE |
                                 RTXPT_REALTIME_RESET_REALTIME_CACHES |
                                 RTXPT_REALTIME_RESET_NRD_HISTORY |
                                 RTXPT_REALTIME_RESET_TAA_SR_HISTORY,
                             "Window resized");
```

- [ ] **Step 5: Add status UI for realtime resources**

In the `Status / Debug` section of `RTXPTSample::UpdateUI()`, after:

```cpp
        ImGui::Text("Post-process targets: %s", m_RenderTargets.HasPostProcessTargets() ? "allocated" : "missing");
```

insert:

```cpp
        ImGui::Text("Realtime render targets: %s",
                    m_RenderTargets.AreRealtimeRenderTargetsRequested() ?
                        (m_RenderTargets.HasRealtimeRenderTargets() ? "allocated" : "missing") :
                        "not requested");
        if (m_RenderTargets.AreRealtimeRenderTargetsRequested())
        {
            ImGui::Text("StablePlanesBuffer elements: %llu",
                        static_cast<unsigned long long>(m_RenderTargets.GetStablePlanesElementCount()));
            ImGui::Text("StableRadiance: %s", m_RenderTargets.GetStableRadianceSRV() != nullptr ? "created" : "missing");
            ImGui::Text("StablePlanesHeader: %s", m_RenderTargets.GetStablePlanesHeaderSRV() != nullptr ? "created" : "missing");
            ImGui::Text("StablePlanesBuffer: %s", m_RenderTargets.GetStablePlanesBufferSRV() != nullptr ? "created" : "missing");
            ImGui::Text("NRD guide inputs: %s",
                        m_RenderTargets.GetDenoiserViewspaceZSRV() != nullptr &&
                                m_RenderTargets.GetDenoiserMotionVectorsSRV() != nullptr &&
                                m_RenderTargets.GetDenoiserNormalRoughnessSRV() != nullptr ?
                            "created" :
                            "missing");
        }
        if (m_RenderTargets.GetLastFailureReason()[0] != '\0')
            ImGui::TextWrapped("Render target error: %s", m_RenderTargets.GetLastFailureReason());
```

- [ ] **Step 6: Build and smoke-check Reference mode still allocates old resources only**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
rg -n "MakeRenderTargetCreateInfo|CreateRealtimeResources|Realtime render targets|Render target error" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected:

```text
MakeRenderTargetCreateInfo appears once.
CreateRealtimeResources is assigned from RealtimeUI.RealtimeMode.
Status UI reports "Realtime render targets".
```

- [ ] **Step 7: Commit Task 5**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): request realtime render targets" -m "Co-Authored-By: GPT 5.5"
```

### Task 6: Update Fork Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Append the G3 mapping section**

Append this section near the existing realtime/post-process mapping area, or at the end of the document if no realtime section exists yet:

```markdown
## Realtime G3 Render Target Map

| RTXPT-fork source | Diligent owner | Notes |
|---|---|---|
| `SampleCommon/RenderTargets.h` `RenderTargets::StableRadiance` | `src/RTXPTRenderTargets.hpp` `m_StableRadiance` | Render-size `RGBA16_FLOAT`, SRV/UAV. |
| `SampleCommon/RenderTargets.h` `RenderTargets::StablePlanesHeader` | `src/RTXPTRenderTargets.hpp` `m_StablePlanesHeader` | Render-size 4-layer `R32_UINT` texture array, SRV/UAV. |
| `SampleCommon/RenderTargets.h` `RenderTargets::StablePlanesBuffer` | `src/RTXPTRenderTargets.hpp` `m_StablePlanesBuffer` | Structured buffer of `RTXPTStablePlaneData`, element count = generic TS plane stride * `kRTXPTStablePlaneCount`. |
| `SampleCommon/RenderTargets.h` `RenderTargets::Throughput` | `src/RTXPTRenderTargets.hpp` `m_Throughput` | Render-size `R32_UINT`, SRV/UAV. |
| `SampleCommon/RenderTargets.h` `RenderTargets::SpecularHitT` | `src/RTXPTRenderTargets.hpp` `m_SpecularHitT` | Render-size `R32_FLOAT`, SRV/UAV. |
| `SampleCommon/RenderTargets.h` `RenderTargets::ScratchFloat1` | `src/RTXPTRenderTargets.hpp` `m_ScratchFloat1` | Render-size `R32_FLOAT`, SRV/UAV. |
| `SampleCommon/RenderTargets.h` denoiser input textures | `src/RTXPTRenderTargets.hpp` `m_Denoiser*` input members | Render-size NRD input resources with names preserved for G7-G9 bindings. |
| `SampleCommon/RenderTargets.h` `DenoiserOutDiffRadianceHitDist[cStablePlaneCount]` | `src/RTXPTRenderTargets.hpp` `m_DenoiserOutDiffRadianceHitDist` | Three per-plane output textures. |
| `SampleCommon/RenderTargets.h` `DenoiserOutSpecRadianceHitDist[cStablePlaneCount]` | `src/RTXPTRenderTargets.hpp` `m_DenoiserOutSpecRadianceHitDist` | Three per-plane output textures. |
| `SampleCommon/RenderTargets.h` `DenoiserOutValidation` | `src/RTXPTRenderTargets.hpp` `m_DenoiserOutValidation` | Optional validation texture, disabled by default. |
| `SampleCommon/RenderTargets.h` `DenoiserAvgLayerRadianceHalfRes` | `src/RTXPTRenderTargets.hpp` `m_DenoiserAvgLayerRadianceHalfRes` | Half render-size `RGBA16_FLOAT`, SRV/UAV. |
| `Shaders/PathTracer/StablePlanes.hlsli` `struct StablePlane` | `src/RTXPTRenderTargets.hpp` `RTXPTStablePlaneData` | CPU mirror guarded at 80 bytes with important offsets checked. |
```

- [ ] **Step 2: Verify mapping anchors resolve**

Run:

```powershell
rg -n "Realtime G3 Render Target Map|RTXPTStablePlaneData|m_StableRadiance|m_DenoiserAvgLayerRadianceHalfRes" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp
```

Expected: the mapping section and Diligent owner symbols are found.

- [ ] **Step 3: Commit Task 6**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map realtime render targets" -m "Co-Authored-By: GPT 5.5"
```

### Task 7: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Run the RTXPT build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: target `RTXPT` builds successfully. If `build\x64\Debug` is missing, configure first:

```powershell
cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
cmake --build build\x64\Debug --config Debug --target RTXPT
```

- [ ] **Step 2: Verify all G3 resources have accessors**

Run:

```powershell
rg -n "GetStableRadiance|GetStablePlanesHeader|GetStablePlanesBuffer|GetThroughput|GetSpecularHitT|GetScratchFloat1|GetDenoiserViewspaceZ|GetDenoiserMotionVectors|GetDenoiserNormalRoughness|GetDenoiserDiffRadianceHitDist|GetDenoiserSpecRadianceHitDist|GetDenoiserDisocclusionThresholdMix|GetDenoiserOutDiffRadianceHitDist|GetDenoiserOutSpecRadianceHitDist|GetDenoiserOutValidation|GetDenoiserAvgLayerRadianceHalfRes" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: each accessor name appears in both declaration and implementation, except `GetStablePlanesBuffer` whose texture-view suffix is not present by design.

- [ ] **Step 3: Verify DLSS-RR guide resources were not added**

Run:

```powershell
rg -n "RRDiffuseAlbedo|RRSpecAlbedo|RRNormalsAndRoughness|RRSpecMotionVectors|RRTransparencyLayer" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: no matches. G3 intentionally leaves RR guide resources out.

- [ ] **Step 4: Verify failure status and fail-closed resize paths**

Run:

```powershell
rg -n "FailResize|GetLastFailureReason|Render target error|not supported; RTXPT realtime|Reset\\(\\);" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: `FailResize` is implemented, unsupported realtime formats call it, and the status UI prints the last render-target error.

- [ ] **Step 5: Verify reference-mode call sites still use compatibility overloads or create-info safely**

Run:

```powershell
rg -n "Resize\\(m_pDevice|RTXPTRenderTargetCreateInfo|CreateRealtimeResources|CreateAccumulatedRadiance" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: `RTXPTSample` uses `MakeRenderTargetCreateInfo()`, `CreateRealtimeResources` is set from `RealtimeUI.RealtimeMode`, and wrapper overloads still exist for any older call sites.

- [ ] **Step 6: Optional smoke run**

Run the Debug RTXPT sample with the existing local executable path for this checkout. If no standard launch command exists, start it from Visual Studio or the generated sample executable and check the Status / Debug panel.

Expected:

```text
Reference mode: "Realtime render targets: not requested"
Realtime mode: "Realtime render targets: allocated" on supported devices, or a visible "Render target error: ..." line on unsupported formats
OutputColor, Depth, ScreenMotionVectors, and post-process targets remain allocated in Reference mode
```

- [ ] **Step 7: Final source scan**

Run:

```powershell
rg -n "StablePlanesBuffer|DenoiserAvgLayerRadianceHalfRes|Realtime render targets|Realtime G3 Render Target Map" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git status --short
```

Expected: G3 source and mapping changes are visible; no unexpected generated files are present.

- [ ] **Step 8: Commit verification-only fixes if needed**

If verification revealed a typo or missing accessor, fix it and commit:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "fix(rtxpt): complete realtime render target wiring" -m "Co-Authored-By: GPT 5.5"
```

If no fixes were needed, do not create a verification commit.

---

## Acceptance Checklist

- [ ] Reference mode still creates the existing post-process render-target set and does not request realtime resources.
- [ ] Realtime mode requests stable-plane and NRD input/output resources through `RTXPTRenderTargetCreateInfo`.
- [ ] `StablePlanesHeader` is a render-size four-layer `R32_UINT` texture array with SRV/UAV accessors.
- [ ] `StablePlanesBuffer` is a structured buffer of 80-byte `RTXPTStablePlaneData` elements, sized by generic tiled-storage plane stride times `kRTXPTStablePlaneCount`.
- [ ] All G3 textures expose SRV/UAV accessors needed by G4-G9.
- [ ] `DenoiserAvgLayerRadianceHalfRes` uses `(RenderWidth + 1) / 2` by `(RenderHeight + 1) / 2`.
- [ ] Unsupported realtime formats call `FailResize()`, release owned targets, and expose a visible failure reason.
- [ ] Resize/window-size changes request realtime cache, NRD history, and TAA/SR history reset flags.
- [ ] No DLSS-RR guide resources are added in G3.
- [ ] `RTXPT_FORK_MAPPING.md` records the G3 source-to-owner mapping.
