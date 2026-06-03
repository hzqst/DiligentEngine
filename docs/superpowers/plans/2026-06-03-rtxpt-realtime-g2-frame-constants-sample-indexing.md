# RTXPT Realtime G2 Frame Constants and Sample Indexing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the Diligent RTXPT shared frame constants so realtime path tracing can use RTXPT-fork-compatible frame dimensions, sample identity, exposure/denoiser controls, stable-plane controls, generic tiled-storage strides, and previous-frame camera/view data.

**Architecture:** Keep the constants Diligent-native, but align field names and semantics with `D:/RTXPT-fork/Rtxpt` so later G4-G9 shader ports can consume `g_Const.ptConsts` with minimal translation. Preserve the current reference path by keeping existing field names, add realtime-specific fields around them, and make realtime sample identity independent from reference accumulation state.

**Tech Stack:** C++17, HLSL shared headers, Diligent `SampleBase`/`FirstPersonCamera`, ImGui status UI, RTXPT tone-mapping pipeline, CMake sample target `RTXPT`, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`, goal `G2 - Realtime Frame Constants and Sample Indexing`.
- G1 state is already present in this checkout:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp`
  - `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp::m_RealtimeUI`
  - `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp::BeginRealtimeFrameResetScope`
- Current Diligent constants live in:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Current `PathTracerConstants` is 80 bytes and lacks realtime fields such as `imageWidth`, `imageHeight`, `sampleBaseIndex`, `frameIndex`, `invSubSampleCount`, `texLODBias`, `preExposedGrayLuminance`, `denoisingEnabled`, stable-plane controls, generic tiled-storage strides, and previous camera data.
- Current `SampleConstants` is 480 bytes and has `viewProj`, `viewProjInv`, `cameraPositionAndTime`, `viewportSizeAndFrameIndex`, `camera`, `ptConsts`, and `envMap`.
- Current reference raygen uses `g_Const.ptConsts.sampleIndex` for all random streams. G2 must keep reference behavior working while preparing realtime shaders to use `sampleBaseIndex`.
- Original RTXPT-fork anchors:
  - `D:/RTXPT-fork/Rtxpt/Sample.cpp:1445-1450`: reference vs realtime `m_sampleIndex`.
  - `D:/RTXPT-fork/Rtxpt/Sample.cpp:1464-1545`: `UpdatePathTracerConstants`.
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h:45-103`: realtime `PathTracerConstants`.
  - `D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h:28-56`: `view`, `previousView`, and `SampleConstants`.
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/Utils.hlsli:323-335`: generic tiled-storage stride math.

## Scope Boundaries

- G2 does not allocate stable-plane, generic tiled-storage, or NRD resources. That is G3.
- G2 does not create ray-tracing mode variants or sub-sample push constants. That is G4/G5.
- G2 does not execute NRD, REBLUR, RELAX, or denoiser prepare/merge passes. That is G7-G9.
- G2 may reserve `DLSSRRBrightnessClampK` and use `DLSSRRMicroJitter` only as constant fields and disabled-path values. It must not execute `RealtimeAA == 3`.
- G2 keeps Reference mode as the only executing render path in this checkout. Realtime mode may still hit the G1 disabled fallback until later goals remove that gate.
- G2 must not use `m_AccumulationFrame` as the realtime noise/sample source. Realtime sample identity is derived from frame-index modulo 8192, matching RTXPT-fork's active realtime sample semantics.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
  - Adds `PathTracerViewData`.
  - Expands `PathTracerConstants` to include RTXPT-fork realtime fields plus Diligent reference compatibility fields.
  - Expands `SampleConstants` with current and previous view data.
  - Adds static layout guards for sizes and important offsets.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
  - Mirrors the new C++ structs and field order.
  - Adds shader-side `GetActiveStablePlaneCount()` and generic tiled-storage stride helpers.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp`
  - Exposes a `ComputePreExposedGrayLuminance()` helper.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp`
  - Implements the pre-exposed gray helper using the same exposure scale and auto-exposure clamp as the tone mapper.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
  - Adds a wrapper so `RTXPTSample` can query pre-exposed gray without reaching into private tone-mapping state.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
  - Implements the wrapper.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Adds previous-frame view/camera storage and realtime sample-index status state.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Populates all new constants.
  - Stores previous camera/view data after upload.
  - Separates reference accumulation sample indices from realtime sample indices.
  - Adds status lines for G2 values.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
  - Uses `sampleBaseIndex` and `perPixelJitterAAScale` while keeping reference output behavior.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  - Maps G2 source constants and sample indexing to Diligent owners.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/Sample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h`
- Verify: `D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h`

- [ ] **Step 1: Confirm dirty files before editing**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files are preserved. At planning time, this repo has untracked realtime spec/plan docs; do not remove or revert them.

- [ ] **Step 2: Confirm current G2 fields are missing or partial**

Run:

```powershell
rg -n "sampleBaseIndex|invSubSampleCount|preExposedGrayLuminance|denoisingEnabled|_activeStablePlaneCount|genericTSLineStride|prevCamera|previousView" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected before G2 implementation: no matches for the new constant fields in `RTXPTFrameConstants.hpp` or `PathTracerShared.h`; `previousView` and `prevCamera` are absent in Diligent constants.

- [ ] **Step 3: Confirm RTXPT-fork source semantics**

Run:

```powershell
rg -n "m_sampleIndex =|sampleBaseIndex|invSubSampleCount|perPixelJitterAAScale|texLODBias|preExposedGrayLuminance|denoisingEnabled|genericTSLineStride|prevCamera|previousView" D:/RTXPT-fork/Rtxpt/Sample.cpp D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h
```

Expected:

```text
D:/RTXPT-fork/Rtxpt/Sample.cpp:1445: reference m_sampleIndex uses m_accumulationSampleIndex
D:/RTXPT-fork/Rtxpt/Sample.cpp:1449: realtime m_sampleIndex uses m_frameIndex % 8192
D:/RTXPT-fork/Rtxpt/Sample.cpp:1507: sampleBaseIndex = m_sampleIndex * ActualSamplesPerPixel()
D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h:45: struct PathTracerConstants
D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h:48: SimpleViewConstants view
D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h:49: SimpleViewConstants previousView
```

- [ ] **Step 4: No commit for preflight**

No source changes are made in Task 0. Do not create a commit for this task.

### Task 1: Expand the C++ and HLSL Shared Constants ABI

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h:45-103`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h:28-56`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/Utils.hlsli:323-335`

- [ ] **Step 1: Replace the C++ view/constants block**

In `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`, keep `PathTracerCameraData` unchanged, then replace the current `PathTracerConstants` and `SampleConstants` declarations with:

```cpp
struct PathTracerViewData
{
    float4x4 MatWorldToView         = float4x4::Identity();
    float4x4 MatViewToClip          = float4x4::Identity();
    float4x4 MatWorldToClip         = float4x4::Identity();
    float4x4 MatWorldToClipNoOffset = float4x4::Identity();
    float4x4 MatClipToWorldNoOffset = float4x4::Identity();
    float2   ViewportOrigin         = float2{0, 0};
    float2   ViewportSize           = float2{1, 1};
    float2   ViewportSizeInv        = float2{1, 1};
    float2   PixelOffset            = float2{0, 0};
    float2   ClipToWindowScale      = float2{0.5f, -0.5f};
    float2   ClipToWindowBias       = float2{0.5f, 0.5f};
};
static_assert(sizeof(PathTracerViewData) == 368, "PathTracerViewData layout must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerViewData, ViewportOrigin) == 320, "PathTracerViewData ViewportOrigin offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerViewData, ClipToWindowBias) == 360, "PathTracerViewData ClipToWindowBias offset must match PathTracer/PathTracerShared.h");

struct PathTracerConstants
{
    Uint32 imageWidth            = 1;
    Uint32 imageHeight           = 1;
    Uint32 sampleBaseIndex       = 0;
    float  perPixelJitterAAScale = 1.0f;

    Uint32 bounceCount                       = 4;
    Uint32 diffuseBounceCount                = 2;
    float  EnvironmentMapDiffuseSampleMIPLevel = 0.0f;
    float  texLODBias                        = 0.0f;

    float  invSubSampleCount       = 1.0f;
    float  fireflyFilterThreshold  = 0.0f;
    float  preExposedGrayLuminance = 1.0f;
    Uint32 denoisingEnabled        = 0;

    Uint32 frameIndex        = 0;
    Uint32 useReSTIRDI       = 0;
    Uint32 useReSTIRGI       = 0;
    Uint32 resetAccumulation = 1;

    float  stablePlanesSplitStopThreshold                  = 0.95f;
    float  _padding3                                       = 0.0f;
    Uint32 _padding4                                       = 0;
    float  stablePlanesSuppressPrimaryIndirectSpecularK    = 0.0f;

    float  denoiserRadianceClampK             = 0.0f;
    float  DLSSRRBrightnessClampK             = 0.0f; // TODO(RTXPT-Realtime-DLSS-RR): reserved constant only.
    float  stablePlanesAntiAliasingFallthrough = 0.0f;
    Uint32 _activeStablePlaneCount             = 1;

    Uint32 maxStablePlaneVertexDepth      = 0;
    Uint32 allowPrimarySurfaceReplacement = 0;
    Uint32 genericTSLineStride            = 1;
    Uint32 genericTSPlaneStride           = 1;

    Uint32 NEEEnabled          = 1;
    Uint32 NEEType             = 1;
    Uint32 NEECandidateSamples = 5;
    Uint32 NEEFullSamples      = 1;

    Uint32 sampleIndex           = 0; // Diligent reference compatibility; realtime uses sampleBaseIndex.
    Uint32 minBounceCount        = 0;
    Uint32 environmentNEEEnabled = 1;
    float  environmentIntensity  = 1.0f;

    float  lightIntensityScale   = 1.0f;
    Uint32 maxNEEBounceCount     = 16;
    Uint32 analyticLightCount    = 0;
    Uint32 NEEMISType            = 0;

    Uint32 nestedDielectricsQuality = 1;
    Uint32 superResolutionActive    = 0;
    Uint32 _paddingR6_1             = 0;
    Uint32 _paddingR6_2             = 0;

    PathTracerCameraData camera     = {};
    PathTracerCameraData prevCamera = {};
};
static_assert(sizeof(PathTracerConstants) == 400, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, sampleBaseIndex) == 8, "PathTracerConstants sampleBaseIndex offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, invSubSampleCount) == 32, "PathTracerConstants invSubSampleCount offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, frameIndex) == 48, "PathTracerConstants frameIndex offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, genericTSLineStride) == 104, "PathTracerConstants genericTSLineStride offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, sampleIndex) == 128, "PathTracerConstants sampleIndex offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, camera) == 176, "PathTracerConstants camera offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(PathTracerConstants, prevCamera) == 288, "PathTracerConstants prevCamera offset must match PathTracer/PathTracerShared.h");

struct SampleConstants
{
    float4x4             viewProj                  = float4x4::Identity();
    float4x4             viewProjInv               = float4x4::Identity();
    float4               cameraPositionAndTime     = float4{0, 0, 0, 0};
    float4               viewportSizeAndFrameIndex = float4{0, 0, 0, 0};
    PathTracerViewData   view                      = {};
    PathTracerViewData   previousView              = {};
    PathTracerCameraData camera                    = {};
    PathTracerConstants  ptConsts                  = {};
    RTXPTEnvMapConstants envMap                    = {};
};
static_assert(sizeof(SampleConstants) == 1536, "SampleConstants layout must match PathTracer/PathTracerShared.h");
static_assert(offsetof(SampleConstants, view) == 160, "SampleConstants view offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(SampleConstants, previousView) == 528, "SampleConstants previousView offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(SampleConstants, camera) == 896, "SampleConstants camera offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(SampleConstants, ptConsts) == 1008, "SampleConstants ptConsts offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(SampleConstants, envMap) == 1408, "SampleConstants envMap offset must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Replace the HLSL mirror**

In `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`, keep `PathTracerCameraData` unchanged, then replace the current `PathTracerConstants` and `SampleConstants` declarations with:

```hlsl
struct PathTracerViewData
{
    float4x4 MatWorldToView;
    float4x4 MatViewToClip;
    float4x4 MatWorldToClip;
    float4x4 MatWorldToClipNoOffset;
    float4x4 MatClipToWorldNoOffset;
    float2   ViewportOrigin;
    float2   ViewportSize;
    float2   ViewportSizeInv;
    float2   PixelOffset;
    float2   ClipToWindowScale;
    float2   ClipToWindowBias;
};

// Mirrors Diligent::PathTracerConstants in RTXPTFrameConstants.hpp.
struct PathTracerConstants
{
    uint  imageWidth;
    uint  imageHeight;
    uint  sampleBaseIndex;
    float perPixelJitterAAScale;

    uint  bounceCount;
    uint  diffuseBounceCount;
    float EnvironmentMapDiffuseSampleMIPLevel;
    float texLODBias;

    float invSubSampleCount;
    float fireflyFilterThreshold;
    float preExposedGrayLuminance;
    uint  denoisingEnabled;

    uint frameIndex;
    uint useReSTIRDI;
    uint useReSTIRGI;
    uint resetAccumulation;

    float stablePlanesSplitStopThreshold;
    float _padding3;
    uint  _padding4;
    float stablePlanesSuppressPrimaryIndirectSpecularK;

    float denoiserRadianceClampK;
    float DLSSRRBrightnessClampK; // TODO(RTXPT-Realtime-DLSS-RR): reserved constant only.
    float stablePlanesAntiAliasingFallthrough;
    uint  _activeStablePlaneCount;

    uint maxStablePlaneVertexDepth;
    uint allowPrimarySurfaceReplacement;
    uint genericTSLineStride;
    uint genericTSPlaneStride;

    uint NEEEnabled;
    uint NEEType;
    uint NEECandidateSamples;
    uint NEEFullSamples;

    uint  sampleIndex;
    uint  minBounceCount;
    uint  environmentNEEEnabled;
    float environmentIntensity;

    float lightIntensityScale;
    uint  maxNEEBounceCount;
    uint  analyticLightCount;
    uint  NEEMISType;

    uint nestedDielectricsQuality;
    uint superResolutionActive;
    uint _paddingR6_1;
    uint _paddingR6_2;

    PathTracerCameraData camera;
    PathTracerCameraData prevCamera;

    uint GetActiveStablePlaneCount()
    {
    #if defined(RTXPT_ACTIVE_STABLE_PLANE_COUNT)
        return RTXPT_ACTIVE_STABLE_PLANE_COUNT;
    #else
        return _activeStablePlaneCount;
    #endif
    }
};

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

// Mirrors Diligent::SampleConstants in RTXPTFrameConstants.hpp.
struct SampleConstants
{
    float4x4             viewProj;
    float4x4             viewProjInv;
    float4               cameraPositionAndTime;
    float4               viewportSizeAndFrameIndex;
    PathTracerViewData   view;
    PathTracerViewData   previousView;
    PathTracerCameraData camera;
    PathTracerConstants  ptConsts;
    RTXPTEnvMapConstants envMap;
};
```

- [ ] **Step 3: Add HLSL generic tiled-storage stride helpers**

In `PathTracerShared.h`, immediately after `SampleConstants`, add:

```hlsl
static const uint RTXPT_GENERIC_TS_TILE_SIZE = 8u;

uint RTXPTGenericTSComputeLineStride(uint imageWidth)
{
    const uint tileCountX = (imageWidth + RTXPT_GENERIC_TS_TILE_SIZE - 1u) / RTXPT_GENERIC_TS_TILE_SIZE;
    return tileCountX * RTXPT_GENERIC_TS_TILE_SIZE;
}

uint RTXPTGenericTSComputePlaneStride(uint imageWidth, uint imageHeight)
{
    const uint tileCountY = (imageHeight + RTXPT_GENERIC_TS_TILE_SIZE - 1u) / RTXPT_GENERIC_TS_TILE_SIZE;
    return RTXPTGenericTSComputeLineStride(imageWidth) * tileCountY * RTXPT_GENERIC_TS_TILE_SIZE;
}
```

- [ ] **Step 4: Build-check layout edits**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. The C++ `static_assert`s confirm:

```text
sizeof(PathTracerViewData) == 368
sizeof(PathTracerConstants) == 400
sizeof(SampleConstants) == 1536
```

- [ ] **Step 5: Commit Task 1**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
git commit -m "feat(rtxpt): expand realtime frame constants" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits. If the user did not ask for commits, leave the changes unstaged after recording verification output.

### Task 2: Expose Pre-Exposed Gray Luminance

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`

- [ ] **Step 1: Add public tone-mapping helper declarations**

In `RTXPTToneMappingPass.hpp`, add this public method after `Render(...)`:

```cpp
    float ComputePreExposedGrayLuminance(const RTXPTToneMappingParameters& Params, bool Enabled) const;
```

In `RTXPTPostProcessPipeline.hpp`, add this public method after `RunToneMapping(...)`:

```cpp
    float ComputePreExposedGrayLuminance(const RTXPTToneMappingParameters& Params, bool Enabled) const;
```

- [ ] **Step 2: Add the tone-mapping helper implementation**

In `RTXPTToneMappingPass.cpp`, add this helper in the anonymous namespace after `CalculateWhiteBalanceTransformRGBRec709(...)`:

```cpp
float ComputeRec709Luminance(const float3& Color)
{
    return dot(Color, float3{0.2126f, 0.7152f, 0.0722f});
}
```

Add this method before `RTXPTToneMappingPass::Render(...)`:

```cpp
float RTXPTToneMappingPass::ComputePreExposedGrayLuminance(const RTXPTToneMappingParameters& Params, bool Enabled) const
{
    if (!Enabled)
        return 1.0f;

    const float3x3 WhiteBalanceTransform = Params.WhiteBalance ?
        CalculateWhiteBalanceTransformRGBRec709(Params.WhitePoint) :
        float3x3::Identity();

    const float ExposureScale = std::pow(2.0f, Params.ExposureCompensation);
    float       Exposure      = ExposureScale;

    if (Params.AutoExposure)
    {
        const float AvgLuminance =
            (std::isfinite(m_Stats.LastAvgLuminance) && m_Stats.LastAvgLuminance > 0.0f) ?
            m_Stats.LastAvgLuminance :
            kDefaultAvgLuminance;
        const float AutoExposureMin = std::pow(2.0f, Params.ExposureValueMin);
        const float AutoExposureMax = std::pow(2.0f, Params.ExposureValueMax);
        Exposure *= std::clamp(0.042f / std::max(AvgLuminance, 1.0e-6f), AutoExposureMin, AutoExposureMax);
    }
    else
    {
        const float Shutter = std::max(Params.Shutter, 0.001f);
        const float FNumber = std::max(Params.FNumber, 0.1f);
        Exposure *= (Params.FilmSpeed / 100.0f) / (Shutter * FNumber * FNumber);
    }

    const float3 Gray = mul(float3{0.5f, 0.5f, 0.5f} * Exposure, WhiteBalanceTransform);
    return std::max(ComputeRec709Luminance(Gray), 1.0e-6f);
}
```

- [ ] **Step 3: Add the post-process wrapper**

In `RTXPTPostProcessPipeline.cpp`, add this method after `RunToneMapping(...)`:

```cpp
float RTXPTPostProcessPipeline::ComputePreExposedGrayLuminance(const RTXPTToneMappingParameters& Params, bool Enabled) const
{
    return m_ToneMappingPass.ComputePreExposedGrayLuminance(Params, Enabled);
}
```

- [ ] **Step 4: Build-check the helper**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If `mul(float3, float3x3)` is not available in this Diligent math version, replace the `Gray` line with:

```cpp
    const float3 Gray{
        (0.5f * Exposure) * WhiteBalanceTransform._11 + (0.5f * Exposure) * WhiteBalanceTransform._21 + (0.5f * Exposure) * WhiteBalanceTransform._31,
        (0.5f * Exposure) * WhiteBalanceTransform._12 + (0.5f * Exposure) * WhiteBalanceTransform._22 + (0.5f * Exposure) * WhiteBalanceTransform._32,
        (0.5f * Exposure) * WhiteBalanceTransform._13 + (0.5f * Exposure) * WhiteBalanceTransform._23 + (0.5f * Exposure) * WhiteBalanceTransform._33};
```

Then rerun the same build command and record which form compiled.

- [ ] **Step 5: Commit Task 2**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
git commit -m "feat(rtxpt): expose pre-exposed gray luminance" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 3: Populate Realtime Constants and Previous-Frame Data

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp:1445-1450`, `1501-1540`

- [ ] **Step 1: Add previous-frame and realtime sample state**

In `RTXPTSample.hpp`, add these methods after `BeginRealtimeFrameResetScope()`:

```cpp
    void InvalidatePreviousFrameConstants();
```

Add these fields after `m_LastCameraProj`:

```cpp
    PathTracerCameraData m_PreviousFrameCamera       = {};
    PathTracerViewData   m_PreviousFrameView         = {};
    bool                 m_HasPreviousFrameConstants = false;
    Uint32               m_RealtimeSampleIndex       = 0;
    Uint32               m_LastSampleBaseIndex       = 0;
```

- [ ] **Step 2: Add local helper functions**

In the anonymous namespace in `RTXPTSample.cpp`, add after `kRTXPTRealtimeDisabledReason`:

```cpp
constexpr Uint32 kRTXPTRealtimeNoisePeriod = 8192u;
constexpr Uint32 kRTXPTGenericTSTileSize   = 8u;
```

Add these helpers after `MakePathTracerCameraData(...)`:

```cpp
Uint32 ComputeGenericTSLineStride(Uint32 ImageWidth)
{
    const Uint32 SafeWidth  = std::max(ImageWidth, Uint32{1});
    const Uint32 TileCountX = (SafeWidth + kRTXPTGenericTSTileSize - 1u) / kRTXPTGenericTSTileSize;
    return TileCountX * kRTXPTGenericTSTileSize;
}

Uint32 ComputeGenericTSPlaneStride(Uint32 ImageWidth, Uint32 ImageHeight)
{
    const Uint32 SafeHeight = std::max(ImageHeight, Uint32{1});
    const Uint32 TileCountY = (SafeHeight + kRTXPTGenericTSTileSize - 1u) / kRTXPTGenericTSTileSize;
    return ComputeGenericTSLineStride(ImageWidth) * TileCountY * kRTXPTGenericTSTileSize;
}

float ComputeSuperResolutionTexLODBias(Uint32 RenderWidth, Uint32 RenderHeight, Uint32 DisplayWidth, Uint32 DisplayHeight)
{
    const float RenderPixels  = static_cast<float>(std::max(RenderWidth, Uint32{1}) * std::max(RenderHeight, Uint32{1}));
    const float DisplayPixels = static_cast<float>(std::max(DisplayWidth, Uint32{1}) * std::max(DisplayHeight, Uint32{1}));
    return -std::log2(std::sqrt(DisplayPixels / RenderPixels));
}

PathTracerViewData MakePathTracerViewData(const float4x4& View,
                                          const float4x4& Proj,
                                          Uint32          RenderWidth,
                                          Uint32          RenderHeight,
                                          const float2&   PixelOffset)
{
    const Uint32 SafeWidth  = std::max(RenderWidth, Uint32{1});
    const Uint32 SafeHeight = std::max(RenderHeight, Uint32{1});
    const float  Width      = static_cast<float>(SafeWidth);
    const float  Height     = static_cast<float>(SafeHeight);

    const float4x4 ViewProj = View * Proj;

    PathTracerViewData Data;
    Data.MatWorldToView         = View;
    Data.MatViewToClip          = Proj;
    Data.MatWorldToClip         = ViewProj;
    Data.MatWorldToClipNoOffset = ViewProj;
    Data.MatClipToWorldNoOffset = ViewProj.Inverse();
    Data.ViewportOrigin         = float2{0.0f, 0.0f};
    Data.ViewportSize           = float2{Width, Height};
    Data.ViewportSizeInv        = float2{1.0f / Width, 1.0f / Height};
    Data.PixelOffset            = PixelOffset;
    Data.ClipToWindowScale      = float2{Width * 0.5f, -Height * 0.5f};
    Data.ClipToWindowBias       = float2{Width * 0.5f, Height * 0.5f};
    return Data;
}
```

- [ ] **Step 3: Invalidate previous-frame constants on hard resets**

In `EnsureRenderTargets()`, inside the branch that calls `RequestAccumulationReset("Render targets (re)created");`, add:

```cpp
        InvalidatePreviousFrameConstants();
```

In `ResetSceneDependentResources()`, after the existing line that sets `m_HasLastCameraMatrices = false`, add:

```cpp
    InvalidatePreviousFrameConstants();
```

In `SetCurrentScene(...)`, after the existing line that sets `m_HasLastCameraMatrices = false`, add:

```cpp
    InvalidatePreviousFrameConstants();
```

In `ApplySceneCamera(...)`, after the existing line that sets `m_HasLastCameraMatrices = false`, add:

```cpp
    InvalidatePreviousFrameConstants();
```

In `WindowResize(...)`, after the existing line that sets `m_HasLastCameraMatrices = false`, add:

```cpp
    InvalidatePreviousFrameConstants();
```

In `RequestRealtimeReset(...)`, after `m_RealtimeResetPending |= Flags;`, add:

```cpp
    if (HasRealtimeResetFlag(Flags, RTXPT_REALTIME_RESET_RENDER_TARGET_RECREATE) ||
        HasRealtimeResetFlag(Flags, RTXPT_REALTIME_RESET_TAA_SR_HISTORY))
    {
        InvalidatePreviousFrameConstants();
    }
```

Add the method near `BeginRealtimeFrameResetScope()`:

```cpp
void RTXPTSample::InvalidatePreviousFrameConstants()
{
    m_HasPreviousFrameConstants = false;
}
```

- [ ] **Step 4: Replace the constants population block**

In `RTXPTSample::UpdateFrameConstants(double CurrTime)`, replace the block from:

```cpp
    const float3   CameraPosition = m_Camera.GetPos();
```

through:

```cpp
    m_LastFrameConstants.envMap = m_EnvMapBaker.GetConstants();
```

with:

```cpp
    const bool   RealtimeMode = m_RealtimeUI.RealtimeMode;
    const Uint32 ActualSPP    = std::max(m_RealtimeUI.ActualSamplesPerPixel(), 1u);

    const float2 CameraJitter = float2{0.0f, 0.0f};

    const float3   CameraPosition = m_Camera.GetPos();
    const float4x4 CameraView     = m_Camera.GetViewMatrix();
    const float4x4 CameraProj     = m_Camera.GetProjMatrix();
    const float4x4 ViewProj       = CameraView * CameraProj;

    const PathTracerCameraData CurrentCamera = MakePathTracerCameraData(m_Camera,
                                                                        RenderWidth,
                                                                        RenderHeight,
                                                                        DisplayWidth,
                                                                        DisplayHeight,
                                                                        m_ReferenceUI.CameraFocalDistance,
                                                                        m_ReferenceUI.CameraAperture,
                                                                        CameraJitter);
    const PathTracerViewData CurrentView = MakePathTracerViewData(CameraView,
                                                                  CameraProj,
                                                                  RenderWidth,
                                                                  RenderHeight,
                                                                  CameraJitter);
    const PathTracerCameraData PreviousCamera = m_HasPreviousFrameConstants ? m_PreviousFrameCamera : CurrentCamera;
    const PathTracerViewData   PreviousView   = m_HasPreviousFrameConstants ? m_PreviousFrameView : CurrentView;

    m_LastFrameConstants.viewProj                  = ViewProj;
    m_LastFrameConstants.viewProjInv               = ViewProj.Inverse();
    m_LastFrameConstants.cameraPositionAndTime     = float4{CameraPosition.x, CameraPosition.y, CameraPosition.z, static_cast<float>(CurrTime)};
    m_LastFrameConstants.viewportSizeAndFrameIndex = float4{Width, Height, Width > 0.0f ? 1.0f / Width : 0.0f, static_cast<float>(m_FrameIndex)};
    m_LastFrameConstants.view                      = CurrentView;
    m_LastFrameConstants.previousView              = PreviousView;
    m_LastFrameConstants.camera                    = CurrentCamera;

    if (m_AccumulationActive)
    {
        if (m_ResetAccumulationPending)
            m_AccumulationFrame = 1;
        else
            ++m_AccumulationFrame;
    }
    else
    {
        m_AccumulationFrame = 0;
    }

    const Uint32 ReferenceSampleIndex = m_AccumulationFrame;
    const Uint32 PathTraceSampleIndex = RealtimeMode ? (m_FrameIndex % kRTXPTRealtimeNoisePeriod) : ReferenceSampleIndex;
    const Uint32 SampleBaseIndex      = PathTraceSampleIndex * ActualSPP;
    m_RealtimeSampleIndex             = RealtimeMode ? PathTraceSampleIndex : 0u;
    m_LastSampleBaseIndex             = SampleBaseIndex;

    PathTracerConstants& PtConsts = m_LastFrameConstants.ptConsts;
    PtConsts.imageWidth            = RenderWidth;
    PtConsts.imageHeight           = RenderHeight;
    PtConsts.sampleBaseIndex       = SampleBaseIndex;
    PtConsts.perPixelJitterAAScale =
        RealtimeMode ?
        (m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR ? m_RealtimeUI.DLSSRRMicroJitter : 0.0f) :
        1.0f;

    PtConsts.bounceCount        = m_MaxBounces;
    PtConsts.diffuseBounceCount = static_cast<Uint32>(std::clamp(m_ReferenceUI.DiffuseBounceCount, 0, 16));
    PtConsts.EnvironmentMapDiffuseSampleMIPLevel = 0.0f;
    PtConsts.texLODBias = RealtimeMode ?
        (m_RealtimeUI.TexLODBias + ComputeSuperResolutionTexLODBias(RenderWidth, RenderHeight, DisplayWidth, DisplayHeight)) :
        0.0f;

    PtConsts.invSubSampleCount       = 1.0f / static_cast<float>(ActualSPP);
    PtConsts.preExposedGrayLuminance =
        m_PostProcessPipeline.ComputePreExposedGrayLuminance(m_ReferenceUI.ToneMapping, m_ReferenceUI.EnableToneMapping);
    const float DisabledFireflyThreshold = 0.0f;
    if (RealtimeMode)
    {
        PtConsts.fireflyFilterThreshold = m_RealtimeUI.RealtimeFireflyFilterEnabled ?
            m_RealtimeUI.RealtimeFireflyFilterThreshold * std::sqrt(PtConsts.preExposedGrayLuminance) * 1000.0f :
            DisabledFireflyThreshold;
    }
    else
    {
        PtConsts.fireflyFilterThreshold = m_ReferenceUI.ReferenceFireflyFilterEnabled ?
            m_ReferenceUI.ReferenceFireflyFilterThreshold :
            DisabledFireflyThreshold;
    }
    PtConsts.denoisingEnabled = (m_RealtimeUI.ActualUseStandaloneDenoiser() ||
                                 m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR) ?
        1u :
        0u;

    PtConsts.frameIndex        = m_FrameIndex;
    PtConsts.useReSTIRDI       = 0u;
    PtConsts.useReSTIRGI       = 0u;
    PtConsts.resetAccumulation = m_ResetAccumulationPending ? 1u : 0u;

    PtConsts.stablePlanesSplitStopThreshold = m_RealtimeUI.StablePlanesSplitStopThreshold;
    PtConsts._padding3                      = 0.0f;
    PtConsts._padding4                      = 0u;
    PtConsts.stablePlanesSuppressPrimaryIndirectSpecularK =
        m_RealtimeUI.StablePlanesSuppressPrimaryIndirectSpecular ?
        m_RealtimeUI.StablePlanesSuppressPrimaryIndirectSpecularK :
        0.0f;

    PtConsts.denoiserRadianceClampK = m_RealtimeUI.DenoiserRadianceClampK;
    PtConsts.DLSSRRBrightnessClampK = m_RealtimeUI.DLSSRRBrightnessClampK > 0.0f ?
        m_RealtimeUI.DLSSRRBrightnessClampK * PtConsts.preExposedGrayLuminance :
        0.0f;
    PtConsts.stablePlanesAntiAliasingFallthrough = m_RealtimeUI.StablePlanesAntiAliasingFallthrough;
    PtConsts._activeStablePlaneCount =
        static_cast<Uint32>(std::clamp(m_RealtimeUI.StablePlanesActiveCount, Int32{1}, static_cast<Int32>(kRTXPTStablePlaneCount)));

    PtConsts.maxStablePlaneVertexDepth =
        std::min(static_cast<Uint32>(std::clamp(m_RealtimeUI.StablePlanesMaxVertexDepth,
                                                Int32{2},
                                                static_cast<Int32>(kRTXPTStablePlaneMaxVertexIndex))),
                 m_MaxBounces);
    PtConsts.allowPrimarySurfaceReplacement = m_RealtimeUI.AllowPrimarySurfaceReplacement ? 1u : 0u;
    PtConsts.genericTSLineStride            = ComputeGenericTSLineStride(RenderWidth);
    PtConsts.genericTSPlaneStride           = ComputeGenericTSPlaneStride(RenderWidth, RenderHeight);

    PtConsts.NEEEnabled          = m_EnableNEE ? 1u : 0u;
    PtConsts.NEEType             = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEType, 0, 2));
    PtConsts.NEECandidateSamples = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEECandidateSamples, 1, 32));
    PtConsts.NEEFullSamples      = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEFullSamples, 0, 32));

    PtConsts.sampleIndex    = PathTraceSampleIndex;
    PtConsts.minBounceCount = m_MinBounces;
    const Uint32 EmissiveTriangleCount =
        (m_EnableNEE && m_EnableEmissiveNEE && m_EmissiveTrianglePass.IsReady() && !m_EmissiveTrianglesDirty) ?
        m_Lights.GetEmissiveTriangleCount() :
        0u;
    PtConsts.environmentNEEEnabled = PackEnvironmentNEEAndEmissiveTriangleCount(m_EnableEnvNEE, EmissiveTriangleCount);
    PtConsts.environmentIntensity  = m_EnvIntensity;

    PtConsts.lightIntensityScale      = m_LightIntensityScale;
    PtConsts.maxNEEBounceCount        = m_MaxNEEBounces;
    PtConsts.analyticLightCount       = m_Lights.GetStats().LightCount;
    PtConsts.NEEMISType               = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEMISType, 0, 2));
    PtConsts.nestedDielectricsQuality = static_cast<Uint32>(std::clamp(m_ReferenceUI.NestedDielectricsQuality, 0, 2));
    PtConsts.superResolutionActive    = 0u;
    PtConsts._paddingR6_1             = 0u;
    PtConsts._paddingR6_2             = 0u;
    PtConsts.camera                   = CurrentCamera;
    PtConsts.prevCamera               = PreviousCamera;

    m_LastFrameConstants.envMap = m_EnvMapBaker.GetConstants();
```

- [ ] **Step 5: Store previous-frame data after upload**

In `UpdateFrameConstants`, after the constant buffer upload block and before `m_ResetAccumulationPending = false;`, add:

```cpp
    m_PreviousFrameCamera       = CurrentCamera;
    m_PreviousFrameView         = CurrentView;
    m_HasPreviousFrameConstants = true;
```

- [ ] **Step 6: Build-check constants population**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. Reference mode preserves the pre-G2 `m_AccumulationFrame` sample seed behavior; only realtime mode derives sample identity from `m_FrameIndex % 8192`.

- [ ] **Step 7: Commit Task 3**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): populate realtime frame constants" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 4: Update Reference Raygen to Use the Realtime-Compatible Sample Fields

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerBridgeDonut.hlsli:512-551`

- [ ] **Step 1: Replace the sample index and jitter block**

In `PathTracerSample.rgen`, replace:

```hlsl
    const uint sampleIndex = g_Const.ptConsts.sampleIndex;

    // Jitter inside the pixel - free anti-aliasing across accumulated samples (vertex 0 / Base effect).
    SampleGenerator sgCamera = SampleGenerator_makeStateless(pixel, 0u, sampleIndex, kSampleEffect_Base);
    const float2    randomJitter    = sampleNext2D(sgCamera) - 0.5.xx;
    const float2    srJitter        = g_Const.camera.Jitter;
    const float2    subPixelOffset  = g_Const.ptConsts.superResolutionActive != 0u ? srJitter : randomJitter;
    const float2    cameraDoFSample = sampleNext2D(sgCamera);
    CameraRay       cameraRay       = ComputeRayThinlens(g_Const.camera, pixel, subPixelOffset, cameraDoFSample);
```

with:

```hlsl
    const uint sampleIndex = g_Const.ptConsts.sampleBaseIndex;

    // RTXPT-fork-compatible jitter: reference uses a full per-pixel random offset, realtime uses
    // zero jitter unless DLSS-RR reserves a small micro-jitter value.
    SampleGenerator sgCamera = SampleGenerator_makeStateless(pixel, 0u, sampleIndex, kSampleEffect_Base);
    const float2    randomJitter    = sampleNext2D(sgCamera) - 0.5.xx;
    const float2    subPixelOffset  = g_Const.ptConsts.camera.Jitter + randomJitter * g_Const.ptConsts.perPixelJitterAAScale;
    const float2    cameraDoFSample = sampleNext2D(sgCamera);
    CameraRay       cameraRay       = ComputeRayThinlens(g_Const.ptConsts.camera, pixel, subPixelOffset, cameraDoFSample);
```

- [ ] **Step 2: Replace remaining top-level camera reads in raygen**

In the same file, replace:

```hlsl
    float        primaryDepth     = g_Const.camera.FarZ;
```

with:

```hlsl
    float        primaryDepth     = g_Const.ptConsts.camera.FarZ;
```

Replace:

```hlsl
            primaryDepth = payload.hitFlag != 0u ? payload.hitDistance : g_Const.camera.FarZ;
```

with:

```hlsl
            primaryDepth = payload.hitFlag != 0u ? payload.hitDistance : g_Const.ptConsts.camera.FarZ;
```

- [ ] **Step 3: Source-scan for old camera/sample reads**

Run:

```powershell
rg -n "ptConsts\\.sampleIndex|g_Const\\.camera" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: no matches in `PathTracerSample.rgen`.

- [ ] **Step 4: Build-check shader changes**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. Reference raygen compiles with `sampleBaseIndex` and `ptConsts.camera`.

- [ ] **Step 5: Commit Task 4**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "feat(rtxpt): use realtime-compatible sample seeds" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 5: Add Status and Mapping Rows

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add G2 status lines**

In the `Status / Debug` collapsing header in `RTXPTSample::UpdateUI()`, after the existing realtime reset status lines, add:

```cpp
        ImGui::Text("Path-trace frame index: %u", m_LastFrameConstants.ptConsts.frameIndex);
        ImGui::Text("Reference sample index: %u", m_AccumulationFrame);
        ImGui::Text("Realtime sample index: %u", m_RealtimeSampleIndex);
        ImGui::Text("Sample base index: %u", m_LastSampleBaseIndex);
        ImGui::Text("Sub-sample count inverse: %.4f", m_LastFrameConstants.ptConsts.invSubSampleCount);
        ImGui::Text("Pre-exposed gray luminance: %.4f", m_LastFrameConstants.ptConsts.preExposedGrayLuminance);
        ImGui::Text("Generic TS stride: line=%u plane=%u",
                    m_LastFrameConstants.ptConsts.genericTSLineStride,
                    m_LastFrameConstants.ptConsts.genericTSPlaneStride);
```

- [ ] **Step 2: Add G2 mapping rows**

In `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`, near the existing realtime G1 mapping rows, add:

```markdown
| `Sample.cpp::UpdatePathTracerConstants` realtime fields | `src/RTXPTSample.cpp::UpdateFrameConstants`, `src/RTXPTFrameConstants.hpp::PathTracerConstants` | Realtime G2 | Diligent now uploads render dimensions, `sampleBaseIndex`, `frameIndex`, `invSubSampleCount`, realtime jitter scale, texture LOD bias, pre-exposed gray luminance, denoiser flags, stable-plane controls, and generic tiled-storage strides through the shared constant buffer. |
| `Shaders/PathTracer/PathTracerShared.h::PathTracerConstants` | `assets/shaders/PathTracer/PathTracerShared.h`, `src/RTXPTFrameConstants.hpp` | Realtime G2 | C++ and HLSL field order is synchronized with `static_assert` layout guards on the C++ side. Diligent reference compatibility fields are retained after the RTXPT-fork realtime fields. |
| `Sample.cpp` realtime `m_sampleIndex` semantics | `src/RTXPTSample.cpp::UpdateFrameConstants` | Realtime G2 | Reference mode keeps accumulation sample indexing. Realtime mode derives the active sample from `frameIndex % 8192` and uploads `sampleBaseIndex = realtimeSampleIndex * ActualSamplesPerPixel()`. |
| `Shaders/SampleConstantBuffer.h::view/previousView` | `src/RTXPTFrameConstants.hpp::PathTracerViewData`, `assets/shaders/PathTracer/PathTracerShared.h::PathTracerViewData` | Realtime G2 | Current and previous view constants are available for future motion-vector, denoiser guide, and NRD common-settings ports. |
```

- [ ] **Step 3: Source-scan mapping coverage**

Run:

```powershell
rg -n "Realtime G2|sampleBaseIndex|previousView|PathTracerViewData|Generic TS" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: the new mapping rows and status lines are present.

- [ ] **Step 4: Build-check status UI**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds.

- [ ] **Step 5: Commit Task 5**

Run:

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map realtime frame constants" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit succeeds only if the active execution workflow is using local commits.

### Task 6: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Run G2 source coverage scan**

Run:

```powershell
rg -n "imageWidth|imageHeight|sampleBaseIndex|frameIndex|invSubSampleCount|perPixelJitterAAScale|texLODBias|preExposedGrayLuminance|fireflyFilterThreshold|denoisingEnabled|denoiserRadianceClampK|_activeStablePlaneCount|maxStablePlaneVertexDepth|allowPrimarySurfaceReplacement|stablePlanesSplitStopThreshold|stablePlanesSuppressPrimaryIndirectSpecularK|stablePlanesAntiAliasingFallthrough|genericTSLineStride|genericTSPlaneStride|prevCamera|previousView" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: every G2 field appears in both the shared layout and CPU population code, except `previousView`, which appears in the shared layout and previous-frame assignment.

- [ ] **Step 2: Verify sample indexing does not depend on reference accumulation in realtime**

Run:

```powershell
rg -n "m_RealtimeSampleIndex|m_LastSampleBaseIndex|m_FrameIndex % kRTXPTRealtimeNoisePeriod|m_AccumulationFrame|sampleBaseIndex" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected:

```text
m_RealtimeSampleIndex is assigned from PathTraceSampleIndex only in realtime mode.
PathTraceSampleIndex is m_FrameIndex % kRTXPTRealtimeNoisePeriod in realtime mode.
sampleBaseIndex is PathTraceSampleIndex * ActualSPP.
m_AccumulationFrame remains the reference accumulation counter.
```

- [ ] **Step 3: Verify C++/HLSL layout names stay synchronized**

Run:

```powershell
rg -n "struct PathTracerViewData|struct PathTracerConstants|struct SampleConstants|sizeof\\(PathTracerConstants\\)|sizeof\\(SampleConstants\\)|offsetof\\(PathTracerConstants, sampleBaseIndex\\)|offsetof\\(SampleConstants, previousView\\)" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
```

Expected: both files define the same three structs, and C++ has static asserts for `PathTracerConstants` and `SampleConstants` size/offsets.

- [ ] **Step 4: Verify DLSS-RR remains non-executing**

Run:

```powershell
rg -n "DLSSRR|RealtimeAA == RTXPTRealtimeAAMode::DLSSRR|TODO\\(RTXPT-Realtime-DLSS-RR\\)" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected: `DLSSRRBrightnessClampK` and `DLSSRRMicroJitter` are constants/status inputs only. There is no DLSS-RR dispatch, Streamline tagging, or `EvaluateDLSSRR` call.

- [ ] **Step 5: Run diff hygiene**

Run:

```powershell
git diff --check
git diff -- DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: `git diff --check` reports no whitespace errors. The diff is limited to the G2 files listed in this plan unless the build required a local include/order adjustment.

- [ ] **Step 6: Run formatting validation**

Run:

```powershell
Push-Location DiligentSamples\BuildTools\FormatValidation
.\validate_format_win.bat
Pop-Location
```

Expected: formatting validation completes successfully. If local clang-format is missing, record the exact error and continue to the sample build.

- [ ] **Step 7: Build the RTXPT sample target**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If `build\x64\Debug` is not configured, run the repository's existing configure flow before claiming build success.

- [ ] **Step 8: Manual smoke for reference mode**

Run:

```powershell
$RTXPTExe = Get-ChildItem -Path build\x64\Debug -Recurse -Filter RTXPT.exe | Select-Object -First 1 -ExpandProperty FullName
& $RTXPTExe
```

Expected:

```text
Path Tracer -> Mode defaults to Reference.
Reference mode renders through the existing path.
Status / Debug shows Path-trace frame index, Reference sample index, Sample base index, invSubSampleCount, pre-exposed gray luminance, and Generic TS stride.
Reference accumulation still converges, and changing camera still resets accumulation.
Realtime mode still reports the G1 disabled execution path until G4/G5 replace the gate.
```

- [ ] **Step 9: Final status report**

Report:

```text
Verification run:
- G2 source coverage scan: <result>
- realtime/reference sample-index scan: <result>
- layout sync scan: <result>
- DLSS-RR non-execution scan: <result>
- git diff --check: <result>
- DiligentSamples format validation: <result or exact blocker>
- cmake --build build\x64\Debug --config Debug --target RTXPT: <result or exact blocker>
- Manual reference smoke: <result or not run with reason>
```

Do not claim G2 is complete unless the build and at least the source/layout scans have run or their blockers are explicitly reported.

## Self-Review Checklist

- Spec coverage:
  - render dimensions `imageWidth`, `imageHeight`: Task 1 and Task 3.
  - sample identity `sampleBaseIndex`, `frameIndex`, `invSubSampleCount`: Task 1, Task 3, Task 4, Task 6.
  - realtime jitter and texture LOD `perPixelJitterAAScale`, `texLODBias`: Task 1, Task 3, Task 4.
  - exposure/denoiser values `preExposedGrayLuminance`, `fireflyFilterThreshold`, `denoisingEnabled`, `denoiserRadianceClampK`: Task 1, Task 2, Task 3.
  - stable-plane controls and storage strides: Task 1 and Task 3.
  - previous camera/view data: Task 1 and Task 3.
  - `RealtimeAA == 3` DLSS-RR fields reserved only: Task 1, Task 3, Task 6.
  - C++/HLSL layout synchronization with static asserts: Task 1 and Task 6.
  - `sampleBaseIndex = realtimeSampleIndex * ActualSamplesPerPixel()`: Task 3 and Task 6.
  - realtime resets do not use reference accumulation indices: Task 3 and Task 6.
- Placeholder scan:
  - The only allowed deferred marker is the exact spec-required `TODO(RTXPT-Realtime-DLSS-RR)` marker attached to reserved DLSS-RR constants.
  - No broad unspecified implementation steps are used.
- Type consistency:
  - `PathTracerViewData`, `PathTracerConstants`, and `SampleConstants` are defined in C++ and HLSL before later tasks use them.
  - `m_RealtimeSampleIndex` and `m_LastSampleBaseIndex` are declared before UI status uses them.
  - `ComputePreExposedGrayLuminance` is exposed through `RTXPTPostProcessPipeline` before `RTXPTSample::UpdateFrameConstants` calls it.
  - Generic tiled-storage stride helpers use the same tile size, 8, in CPU and HLSL.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-03-rtxpt-realtime-g2-frame-constants-sample-indexing.md`. Two execution options:

1. **Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Which approach?
