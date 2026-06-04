# RTXPT G7 PostProcess Denoiser Prepare and Final Merge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port the RTXPT realtime post-process denoiser prepare, final merge, no-denoiser final merge, and stable-plane debug-viz compute variants into the Diligent RTXPT sample.

**Architecture:** Extend the existing `RTXPTPostProcessPass` into the owner for G7 compute variants, then attach it to `RTXPTPostProcessPipeline` and the realtime render route. The pass writes denoiser inputs and merge results into existing `RTXPTRenderTargets` resources, using `GetAccumulationOutputUAV()` as the realtime merge work target so the result naturally feeds SR when active and tone mapping when SR is inactive.

**Tech Stack:** C++17, Diligent Engine compute PSOs/SRBs, HLSL/DXC, CMake, RTXPT-fork reference code from `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.*`.

---

## Scope

G7 implements:

- `RELAXDenoiserPrepareInputs`
- `REBLURDenoiserPrepareInputs`
- `RELAXDenoiserFinalMerge`
- `REBLURDenoiserFinalMerge`
- `NoDenoiserFinalMerge`
- `StablePlanesDebugViz`
- A narrow `TODO(RTXPT-Realtime-DLSS-RR)` marker for the deferred DLSS-RR prepare path
- Realtime no-denoiser presentation through the normal post-process chain
- C++ entry points that G8 can call around NRD dispatch

G7 does not implement:

- NRD instance creation, NRD permanent/transient pool allocation, NRD common settings, or NRD dispatch. Those remain G8.
- DLSS-RR denoiser prepare.
- TAA/SR implementations beyond routing the merge output into the existing `GetAccumulationOutputUAV()` / `RunSuperResolution()` path.

---

## File Structure

Modify:

- `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  Register `RTXPTPostProcessPass.cpp/.hpp`, `RTXPTPostProcess.csh`, and the new NRD helper include.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
  Allocate `CombinedHistoryClampRelax` when realtime resources are requested, not only when SR resources are active.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp`
  Add G7 pass IDs, dispatch parameter structs, stats, and public run methods.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp`
  Create G7 PSOs/SRBs, bind frame constants and dynamic resources, dispatch prepare/final-merge/no-denoiser/debug-viz passes.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`
  Add G7 shader modes while preserving existing HDR test and edge detection modes.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
  Hold `RTXPTPostProcessPass`, expose realtime prepare/final-merge/no-denoiser methods, and include stats.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
  Initialize the pass and implement pipeline wrappers.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  Add realtime post-process helper declarations and status fields.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  Replace the current realtime fallback clear with G7 final merge, tone mapping, and presentation.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  Add G7 mapping notes.

Create:

- `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli`
  Local wrapper for RELAX/REBLUR pack/unpack helpers. It compiles without external NRD headers today and can forward to NRD headers when G8 adds the dependency gate.

---

### Task 1: Register G7 Build Inputs and Realtime Render Target Prerequisite

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Add post-process pass files to the RTXPT target**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add `src/RTXPTPostProcessPass.cpp` to `SOURCE`, add `src/RTXPTPostProcessPass.hpp` to `INCLUDE`, and add the post-process shader files to `SHADERS`:

```cmake
set(SOURCE
    src/RTXPTSample.cpp
    src/RTXPTScene.cpp
    src/RTXPTSceneJson.cpp
    src/RTXPTSceneGraph.cpp
    src/RTXPTMaterials.cpp
    src/RTXPTLights.cpp
    src/RTXPTLightsBaker.cpp
    src/RTXPTLightsBakerPass.cpp
    src/RTXPTEnvMapBaker.cpp
    src/RTXPTEnvMapBakerPass.cpp
    src/RTXPTAccelerationStructures.cpp
    src/RTXPTSkinnedGeometry.cpp
    src/RTXPTRenderTargets.cpp
    src/RTXPTPostProcessPipeline.cpp
    src/RTXPTPostProcessPass.cpp
    src/RTXPTRayTracingPass.cpp
    src/RTXPTDenoisingGuidesBaker.cpp
    src/RTXPTEmissiveTrianglePass.cpp
    src/RTXPTComputePass.cpp
    src/RTXPTAccumulationPass.cpp
    src/RTXPTBloomPass.cpp
    src/RTXPTSuperResolutionPass.cpp
    src/RTXPTToneMappingPass.cpp
    src/RTXPTBlitPass.cpp
)

set(INCLUDE
    src/RTXPTSample.hpp
    src/RTXPTCameraBasis.hpp
    src/RTXPTScene.hpp
    src/RTXPTSceneJson.hpp
    src/RTXPTSceneGraph.hpp
    src/RTXPTMaterials.hpp
    src/RTXPTLights.hpp
    src/RTXPTLightsBaker.hpp
    src/RTXPTLightsBakerPass.hpp
    src/RTXPTFrameConstants.hpp
    src/RTXPTRealtimeSettings.hpp
    src/RTXPTEnvMapBaker.hpp
    src/RTXPTEnvMapBakerPass.hpp
    src/RTXPTAccelerationStructures.hpp
    src/RTXPTSkinnedGeometry.hpp
    src/RTXPTRenderTargets.hpp
    src/RTXPTPostProcessPipeline.hpp
    src/RTXPTPostProcessPass.hpp
    src/RTXPTRayTracingPass.hpp
    src/RTXPTDenoisingGuidesBaker.hpp
    src/RTXPTEmissiveTrianglePass.hpp
    src/RTXPTComputePass.hpp
    src/RTXPTAccumulationPass.hpp
    src/RTXPTBloomPass.hpp
    src/RTXPTSuperResolutionPass.hpp
    src/RTXPTToneMappingPass.hpp
    src/RTXPTBlitPass.hpp
    RTXPT_FORK_MAPPING.md
)
```

Also add these shader entries under the existing post-processing shader entries:

```cmake
    assets/shaders/PostProcessing/RTXPTPostProcess.csh
    assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli
```

- [ ] **Step 2: Configure to catch missing file registration**

Run:

```powershell
cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
```

Expected: configure succeeds and the RTXPT target lists `RTXPTPostProcessPass.cpp` and `RTXPTPostProcess.csh`.

- [ ] **Step 3: Allocate `CombinedHistoryClampRelax` for realtime**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`, replace the current `RequiresSuperResolutionTargets`-only logic for `CombinedHistoryClampRelax` with this local predicate:

```cpp
const bool RequiresSuperResolutionTargets = Dimensions.SuperResolutionActive;
const bool RequiresHistoryClampRelax      = RequiresSuperResolutionTargets || CreateRealtimeResources;
```

Use `RequiresHistoryClampRelax` in all checks and creation sites:

```cpp
const bool HasP6PostProcessTargets =
    HasCorePostProcessTargets &&
    (!RequiresSuperResolutionTargets ||
     (m_TemporalFeedback1 != nullptr &&
      m_TemporalFeedback2 != nullptr)) &&
    (!RequiresHistoryClampRelax || m_CombinedHistoryClampRelax != nullptr);
```

```cpp
if (RequiresHistoryClampRelax && !SupportsBindFlags(pDevice, Formats.CombinedHistoryClampRelax, UavFlags))
    return FailResize("R8 UAV CombinedHistoryClampRelax is not supported; RTXPT post-processing resource graph is unavailable");
```

```cpp
if (RequiresHistoryClampRelax &&
    !CreateTarget(pDevice, "RTXPT CombinedHistoryClampRelax", DisplayWidth, DisplayHeight, Formats.CombinedHistoryClampRelax, UavFlags, CombinedHistoryClampRelax))
    return FailResize("Failed to create RTXPT CombinedHistoryClampRelax");
```

- [ ] **Step 4: Build after render-target change**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If it fails because G7 files are registered but not yet updated, continue to Task 2 and rebuild after Task 4.

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
git commit -m "chore(rtxpt): register G7 post-process inputs"
```

---

### Task 2: Add NRD Shader Helper Wrapper

**Files:**

- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli`

- [ ] **Step 1: Create the helper include**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli` with this content:

```hlsl
#ifndef __RTXPT_DENOISER_NRD_HLSLI__
#define __RTXPT_DENOISER_NRD_HLSLI__

#include "../PathTracer/PathTracerHelpers.hlsli"

#ifndef RTXPT_HAS_NRD_HEADERS
#define RTXPT_HAS_NRD_HEADERS 0
#endif

#if RTXPT_HAS_NRD_HEADERS
#define NRD_HEADER_ONLY
#include <NRDEncoding.hlsli>
#include <NRD.hlsli>
#endif

#ifndef RTXPT_VIEWZ_SKY_MARKER
#define RTXPT_VIEWZ_SKY_MARKER 3.402823466e+38F
#endif

float4 RTXPTDenoiserPackNormalAndRoughness(float3 Normal, float Roughness)
{
#if RTXPT_HAS_NRD_HEADERS
    return NRD_FrontEnd_PackNormalAndRoughness(Normal, Roughness, 0);
#else
    return float4(normalize(Normal) * 0.5 + 0.5, saturate(Roughness));
#endif
}

float4 RTXPTDenoiserRelaxPackRadianceHitDist(float3 Radiance, float HitDist)
{
#if RTXPT_HAS_NRD_HEADERS
    return RELAX_FrontEnd_PackRadianceAndHitDist(Radiance, HitDist, true);
#else
    return float4(Radiance, HitDist);
#endif
}

float4 RTXPTDenoiserReblurPackRadianceNormHitDist(float3 Radiance, float NormHitDist)
{
#if RTXPT_HAS_NRD_HEADERS
    return REBLUR_FrontEnd_PackRadianceAndNormHitDist(Radiance, NormHitDist, true);
#else
    return float4(Radiance, NormHitDist);
#endif
}

float RTXPTDenoiserReblurGetNormHitDist(float HitDist, float ViewZ, float Roughness)
{
#if RTXPT_HAS_NRD_HEADERS
    // G8 replaces the fallback hit params with NRD CommonSettings-derived constants.
    const float4 HitParams = float4(3.0, 0.1, 10.0, 0.0);
    return REBLUR_FrontEnd_GetNormHitDist(HitDist, ViewZ, HitParams, Roughness);
#else
    return HitDist > 0.0 ? saturate(HitDist / max(abs(ViewZ), 1.0)) : 0.0;
#endif
}

void RTXPTDenoiserPostDenoiseProcess(float3 DiffBSDFEstimate,
                                     float3 SpecBSDFEstimate,
                                     inout float4 DiffRadianceHitDistDenoised,
                                     inout float4 SpecRadianceHitDistDenoised)
{
#if RTXPT_HAS_NRD_HEADERS
#if USE_RELAX
    DiffRadianceHitDistDenoised.xyz = RELAX_BackEnd_UnpackRadiance(DiffRadianceHitDistDenoised).xyz;
    SpecRadianceHitDistDenoised.xyz = RELAX_BackEnd_UnpackRadiance(SpecRadianceHitDistDenoised).xyz;
#else
    DiffRadianceHitDistDenoised.xyz = REBLUR_BackEnd_UnpackRadianceAndNormHitDist(DiffRadianceHitDistDenoised).xyz;
    SpecRadianceHitDistDenoised.xyz = REBLUR_BackEnd_UnpackRadianceAndNormHitDist(SpecRadianceHitDistDenoised).xyz;
#endif
#endif

    DiffRadianceHitDistDenoised.xyz *= max(DiffBSDFEstimate, 0.0.xxx);
    SpecRadianceHitDistDenoised.xyz *= max(SpecBSDFEstimate, 0.0.xxx);
}

#endif // __RTXPT_DENOISER_NRD_HLSLI__
```

- [ ] **Step 2: Run shader source search**

Run:

```powershell
rg -n "RTXPT_HAS_NRD_HEADERS|RTXPTDenoiserPostDenoiseProcess|RTXPT_VIEWZ_SKY_MARKER" DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing
```

Expected: the new helper file defines all three names.

- [ ] **Step 3: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli
git commit -m "feat(rtxpt): add G7 NRD shader helper wrapper"
```

---

### Task 3: Port G7 Shader Modes into `RTXPTPostProcess.csh`

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`

- [ ] **Step 1: Add G7 mode constants and includes**

At the top of `RTXPTPostProcess.csh`, preserve existing mode values and extend them:

```hlsl
#define RTXPT_POST_PROCESS_HDR_TEST                 1
#define RTXPT_POST_PROCESS_EDGE_DETECTION           2
#define RTXPT_POST_PROCESS_STABLE_PLANES_DEBUG_VIZ  3
#define RTXPT_POST_PROCESS_RELAX_PREPARE_INPUTS     4
#define RTXPT_POST_PROCESS_REBLUR_PREPARE_INPUTS    5
#define RTXPT_POST_PROCESS_RELAX_FINAL_MERGE        6
#define RTXPT_POST_PROCESS_REBLUR_FINAL_MERGE       7
#define RTXPT_POST_PROCESS_NO_DENOISER_FINAL_MERGE  8

#define NUM_COMPUTE_THREADS_PER_DIM 8

#include "../PathTracer/PathTracerShared.h"
#include "../PathTracer/PathTracerHelpers.hlsli"
#include "../PathTracer/StablePlanes.hlsli"
#include "RTXPTDenoiserNRD.hlsli"
```

Add resource declarations after the existing P4 resources. Use these names because the C++ pass will bind by name:

```hlsl
cbuffer g_Const
{
    SampleConstants g_Frame;
};

cbuffer g_MiniConst
{
    SampleMiniConstants g_Mini;
};

Texture2D<float>        t_Depth;
Texture2D<float2>       t_MotionVectors;
Texture2D<float>        t_SpecularHitT;
Texture2D<float4>       t_DenoiserOutDiffRadianceHitDist;
Texture2D<float4>       t_DenoiserOutSpecRadianceHitDist;
Texture2D<float4>       t_DenoiserOutValidation;
Texture2D<float>        t_DenoiserViewspaceZ;
Texture2D<float>        t_DenoiserDisocclusionThresholdMix;

VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_OutputColor;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_StableRadiance;
RWTexture2DArray<uint>                              u_StablePlanesHeader;
RWStructuredBuffer<StablePlane>                     u_StablePlanesBuffer;
VK_IMAGE_FORMAT("r32f")   RWTexture2D<float>         u_DenoiserViewspaceZ;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_DenoiserMotionVectors;
VK_IMAGE_FORMAT("rgba8")   RWTexture2D<float4>       u_DenoiserNormalRoughness;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_DenoiserDiffRadianceHitDist;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_DenoiserSpecRadianceHitDist;
VK_IMAGE_FORMAT("r8")      RWTexture2D<float>        u_DenoiserDisocclusionThresholdMix;
VK_IMAGE_FORMAT("r8")      RWTexture2D<float>        u_CombinedHistoryClampRelax;
```

- [ ] **Step 2: Add small shared shader helpers**

Add these helpers before `main`:

```hlsl
float RTXPTLuminance(float3 Color)
{
    return dot(Color, float3(0.2126, 0.7152, 0.0722));
}

void RTXPTClampRadiance(inout float3 Radiance, float RangeK)
{
    const float SafeRangeK = max(RangeK, 1.0);
    const float ClampMax   = min(255.0, g_Frame.ptConsts.preExposedGrayLuminance * SafeRangeK);
    const float Lum        = RTXPTLuminance(Radiance);
    if (Lum > ClampMax)
        Radiance *= ClampMax / Lum;
}

float RTXPTComputeNeighbourDisocclusionRelaxation(StablePlanesContext StablePlanes,
                                                  int2 PixelPos,
                                                  int2 ImageSize,
                                                  uint StablePlaneIndex,
                                                  float3 RayDirC,
                                                  int2 Offset)
{
    const float kEdge     = 0.02;
    uint2       PixelPosN = clamp(PixelPos + Offset, int2(0, 0), ImageSize - 1.xx);
    uint        BranchID  = StablePlanes.GetBranchID(PixelPosN, StablePlaneIndex);
    if (BranchID == cStablePlaneInvalidBranchID)
        return kEdge;

    StablePlane SP      = StablePlanes.LoadStablePlane(PixelPosN, StablePlaneIndex);
    float3      RayDirN = SP.GetNormal();
    return 1.0 - dot(RayDirC, RayDirN);
}

float RTXPTComputeDisocclusionRelaxation(StablePlanesContext StablePlanes,
                                         uint2 PixelPos,
                                         uint StablePlaneIndex,
                                         StablePlane SP)
{
    const int2   ImageSize = int2(g_Frame.ptConsts.imageWidth, g_Frame.ptConsts.imageHeight);
    const float3 RayDirC   = SP.GetNormal();
    float        Relax     = 0.0;

    Relax += RTXPTComputeNeighbourDisocclusionRelaxation(StablePlanes, PixelPos, ImageSize, StablePlaneIndex, RayDirC, int2(-1, 0));
    Relax += RTXPTComputeNeighbourDisocclusionRelaxation(StablePlanes, PixelPos, ImageSize, StablePlaneIndex, RayDirC, int2(1, 0));
    Relax += RTXPTComputeNeighbourDisocclusionRelaxation(StablePlanes, PixelPos, ImageSize, StablePlaneIndex, RayDirC, int2(0, -1));
    Relax += RTXPTComputeNeighbourDisocclusionRelaxation(StablePlanes, PixelPos, ImageSize, StablePlaneIndex, RayDirC, int2(0, 1));
    return saturate((Relax - 0.00002) * 25.0);
}
```

- [ ] **Step 3: Add the NRD prepare branch**

Inside `main`, before the existing P4 branches or by splitting into helper functions, add the prepare branch:

```hlsl
#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_RELAX_PREPARE_INPUTS || RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_REBLUR_PREPARE_INPUTS
    const uint StablePlaneIndex       = g_Mini.params.x;
    const bool InitWithStableRadiance = g_Mini.params.y != 0;

    StablePlanesContext StablePlanes = StablePlanesContext::make(
        u_StablePlanesHeader, u_StablePlanesBuffer, u_StableRadiance, g_Frame.ptConsts);

    if (InitWithStableRadiance)
    {
        u_OutputColor[PixelPos]                 = float4(StablePlanes.LoadStableRadiance(PixelPos), 1.0);
        u_CombinedHistoryClampRelax[PixelPos]   = 0.0;
    }

    bool HasSurface = false;
    uint BranchID   = StablePlanes.GetBranchID(PixelPos, StablePlaneIndex);
    if (BranchID != cStablePlaneInvalidBranchID)
    {
        StablePlane SP         = StablePlanes.LoadStablePlane(PixelPos, StablePlaneIndex);
        const bool  HitSurface = isfinite(SP.SceneLength);
        if (HitSurface)
        {
            HasSurface = true;

            float3 DiffBSDFEstimate;
            float3 SpecBSDFEstimate;
            UnpackTwoFp32ToFp16(SP.DenoiserPackedBSDFEstimate, DiffBSDFEstimate, SpecBSDFEstimate);
            DiffBSDFEstimate = max(DiffBSDFEstimate, 1e-4.xxx);
            SpecBSDFEstimate = max(SpecBSDFEstimate, 1e-4.xxx);

            float3 Throughput;
            float3 MotionVectors;
            UnpackTwoFp32ToFp16(SP.PackedThpAndMVs, Throughput, MotionVectors);

            const float3 VirtualWorldPos = SP.RayOrigin + SP.RayDir * SP.SceneLength;
            const float4 ViewPos         = mul(float4(VirtualWorldPos, 1.0), g_Frame.view.MatWorldToView);
            const float  VirtualViewZ    = ViewPos.z;

            float FinalRoughness = max(0.2, SP.GetRoughness());
            float DisocclusionRelax = 0.0;
            if (StablePlanesVertexIndexFromBranchID(BranchID) > 1)
                DisocclusionRelax = RTXPTComputeDisocclusionRelaxation(StablePlanes, PixelPos, StablePlaneIndex, SP);

            u_DenoiserViewspaceZ[PixelPos]               = VirtualViewZ;
            u_DenoiserMotionVectors[PixelPos]            = float4(MotionVectors, 0.0);
            u_DenoiserDisocclusionThresholdMix[PixelPos] = DisocclusionRelax;
            u_CombinedHistoryClampRelax[PixelPos]        = saturate(u_CombinedHistoryClampRelax[PixelPos] + DisocclusionRelax * saturate(RTXPTLuminance(Throughput)));

            FinalRoughness = saturate(FinalRoughness + DisocclusionRelax);

            float3 DenoiserDiffRadiance = SP.GetNoisyDiffRadiance() / DiffBSDFEstimate;
            float3 DenoiserSpecRadiance = SP.GetNoisySpecRadiance() / SpecBSDFEstimate;

            if (StablePlaneIndex == 0 &&
                g_Frame.ptConsts.stablePlanesSuppressPrimaryIndirectSpecularK != 0.0 &&
                g_Frame.ptConsts.GetActiveStablePlaneCount() > 1)
            {
                bool ShouldSuppress = true;
                for (uint Plane = 1; Plane < g_Frame.ptConsts.GetActiveStablePlaneCount(); ++Plane)
                    ShouldSuppress = ShouldSuppress && StablePlanes.GetBranchID(PixelPos, Plane) != cStablePlaneInvalidBranchID;
                DenoiserSpecRadiance *= ShouldSuppress ? saturate(1.0 - g_Frame.ptConsts.stablePlanesSuppressPrimaryIndirectSpecularK) : 1.0;
            }

            RTXPTClampRadiance(DenoiserDiffRadiance, g_Frame.ptConsts.denoiserRadianceClampK * 16.0);
            RTXPTClampRadiance(DenoiserSpecRadiance, g_Frame.ptConsts.denoiserRadianceClampK * 16.0);

            float SpecHitT = 0.0;
            if (StablePlanes.LoadDominantIndex(PixelPos) == StablePlaneIndex)
                SpecHitT = t_SpecularHitT[PixelPos];

            u_DenoiserNormalRoughness[PixelPos] = RTXPTDenoiserPackNormalAndRoughness(SP.GetNormal(), FinalRoughness);

#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_RELAX_PREPARE_INPUTS
            u_DenoiserDiffRadianceHitDist[PixelPos] = RTXPTDenoiserRelaxPackRadianceHitDist(DenoiserDiffRadiance, 0.0);
            u_DenoiserSpecRadianceHitDist[PixelPos] = RTXPTDenoiserRelaxPackRadianceHitDist(DenoiserSpecRadiance, SpecHitT);
#else
            const float DiffNormHitDist = 0.0;
            const float SpecNormHitDist = RTXPTDenoiserReblurGetNormHitDist(SpecHitT, VirtualViewZ, SP.GetRoughness());
            u_DenoiserDiffRadianceHitDist[PixelPos] = RTXPTDenoiserReblurPackRadianceNormHitDist(DenoiserDiffRadiance, DiffNormHitDist);
            u_DenoiserSpecRadianceHitDist[PixelPos] = RTXPTDenoiserReblurPackRadianceNormHitDist(DenoiserSpecRadiance, SpecNormHitDist);
#endif
        }
    }

    if (!HasSurface)
        u_DenoiserViewspaceZ[PixelPos] = RTXPT_VIEWZ_SKY_MARKER;
#endif
```

- [ ] **Step 4: Add final merge and no-denoiser branches**

Add this final merge branch:

```hlsl
#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_RELAX_FINAL_MERGE || RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_REBLUR_FINAL_MERGE
    const uint StablePlaneIndex = g_Mini.params.x;
    const bool HasValidation    = g_Mini.params.y != 0;

    const bool HasSurface = t_DenoiserViewspaceZ[PixelPos] != RTXPT_VIEWZ_SKY_MARKER;
    const uint SPAddress  = GenericTSPixelToAddress(PixelPos,
                                                    StablePlaneIndex,
                                                    g_Frame.ptConsts.genericTSLineStride,
                                                    g_Frame.ptConsts.genericTSPlaneStride);

    float4 DiffRadiance = 0.0.xxxx;
    float4 SpecRadiance = 0.0.xxxx;

    if (HasSurface)
    {
        float3 DiffBSDFEstimate;
        float3 SpecBSDFEstimate;
        UnpackTwoFp32ToFp16(u_StablePlanesBuffer[SPAddress].DenoiserPackedBSDFEstimate, DiffBSDFEstimate, SpecBSDFEstimate);

        DiffRadiance = t_DenoiserOutDiffRadianceHitDist[PixelPos];
        SpecRadiance = t_DenoiserOutSpecRadianceHitDist[PixelPos];
        RTXPTDenoiserPostDenoiseProcess(DiffBSDFEstimate, SpecBSDFEstimate, DiffRadiance, SpecRadiance);

        u_OutputColor[PixelPos].xyz += max(0.0.xxx, DiffRadiance.rgb + SpecRadiance.rgb);
    }

    if (HasValidation)
    {
        const float4 Validation = t_DenoiserOutValidation[PixelPos];
        if (Validation.a > 0.0)
            u_OutputColor[PixelPos] = float4(Validation.rgb, 1.0);
    }
#endif
```

Add this no-denoiser branch:

```hlsl
#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_NO_DENOISER_FINAL_MERGE
    StablePlanesContext StablePlanes = StablePlanesContext::make(
        u_StablePlanesHeader, u_StablePlanesBuffer, u_StableRadiance, g_Frame.ptConsts);
    u_OutputColor[PixelPos] = float4(StablePlanes.GetAllRadiance(PixelPos), 1.0);
#endif
```

Add this stable-plane debug-viz branch:

```hlsl
#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_STABLE_PLANES_DEBUG_VIZ
    const int DebugPlaneIndex = int(g_Mini.params.x) - 1;
    const uint3 DebugCoord    = StablePlaneDebugVizFourWaySplitCoord(DebugPlaneIndex, PixelPos, uint2(g_Frame.ptConsts.imageWidth, g_Frame.ptConsts.imageHeight));

    StablePlanesContext StablePlanes = StablePlanesContext::make(
        u_StablePlanesHeader, u_StablePlanesBuffer, u_StableRadiance, g_Frame.ptConsts);

    float3 Color = 0.0.xxx;
    if (DebugCoord.z < g_Frame.ptConsts.GetActiveStablePlaneCount())
    {
        const uint BranchID = StablePlanes.GetBranchID(DebugCoord.xy, DebugCoord.z);
        Color = BranchID == cStablePlaneInvalidBranchID ?
            float3(((PixelPos.x + PixelPos.y) & 1u) != 0u, 0.0, 0.0) :
            StablePlaneDebugVizColor(DebugCoord.z);
    }

    u_OutputColor[PixelPos] = float4(Color, 1.0);
#endif
```

- [ ] **Step 5: Preserve existing P4 branches**

Keep the existing HDR test and edge detection branches unchanged except for any resource declaration moves required by the new structure:

```hlsl
#if RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_HDR_TEST
    float3 ExistingColor = u_ProcessedOutputColor[PixelPos].rgb;
    if (length(float2(PixelPos.xy) - float2(800.0, 500.0)) < 100.0)
        ExistingColor.z += 10.0;
    u_ProcessedOutputColor[PixelPos] = float4(ExistingColor, 1.0);
#elif RTXPT_POST_PROCESS_MODE == RTXPT_POST_PROCESS_EDGE_DETECTION
    const int OffX = 1;
    const int OffY = 1;

    const float3 S00 = LoadLDR(PixelPos, int2(-OffX, -OffY));
    const float3 S01 = LoadLDR(PixelPos, int2(0, -OffY));
    const float3 S02 = LoadLDR(PixelPos, int2(OffX, -OffY));
    const float3 S10 = LoadLDR(PixelPos, int2(-OffX, 0));
    const float3 S12 = LoadLDR(PixelPos, int2(OffX, 0));
    const float3 S20 = LoadLDR(PixelPos, int2(-OffX, OffY));
    const float3 S21 = LoadLDR(PixelPos, int2(0, OffY));
    const float3 S22 = LoadLDR(PixelPos, int2(OffX, OffY));

    const float3 SobelX = S00 + 2.0 * S10 + S20 - S02 - 2.0 * S12 - S22;
    const float3 SobelY = S00 + 2.0 * S01 + S02 - S20 - 2.0 * S21 - S22;
    const float3 EdgeSqr = SobelX * SobelX + SobelY * SobelY;
    const float  Threshold = g_Params.EdgeDetectionThreshold;
    const float3 EdgeColor = 1.0.xxx - (EdgeSqr > Threshold.xxx * Threshold.xxx);
    SaveLDR(PixelPos, saturate(EdgeColor));
#endif
```

- [ ] **Step 6: Verify the deferred DLSS-RR marker exists**

Add this exact comment near the mode definitions:

```hlsl
// TODO(RTXPT-Realtime-DLSS-RR): DLSSRRDenoiserPrepareInputs is deferred to the DLSS-RR phase.
```

Run:

```powershell
rg -n "TODO\\(RTXPT-Realtime-DLSS-RR\\).*DLSSRRDenoiserPrepareInputs|RTXPT_POST_PROCESS_RELAX_PREPARE_INPUTS|RTXPT_POST_PROCESS_NO_DENOISER_FINAL_MERGE" DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh
```

Expected: all three patterns are found.

- [ ] **Step 7: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh
git commit -m "feat(rtxpt): port G7 post-process shader variants"
```

---

### Task 4: Extend `RTXPTPostProcessPass` C++ API

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp`

- [ ] **Step 1: Add G7 public types to the header**

In `RTXPTPostProcessPass.hpp`, include the frame constants and render targets:

```cpp
#include "RTXPTFrameConstants.hpp"
#include "RTXPTRealtimeSettings.hpp"
#include "RTXPTRenderTargets.hpp"
```

Add these declarations above `RTXPTPostProcessPassStats`:

```cpp
enum class RTXPTPostProcessPassId : Uint32
{
    HdrTest = 0,
    EdgeDetection,
    StablePlanesDebugViz,
    RelaxDenoiserPrepareInputs,
    ReblurDenoiserPrepareInputs,
    RelaxDenoiserFinalMerge,
    ReblurDenoiserFinalMerge,
    NoDenoiserFinalMerge,
    Count
};

struct RTXPTDenoiserPostProcessAttribs
{
    ITextureView*               pMergeOutputUAV = nullptr;
    const RTXPTRenderTargets*   pRenderTargets  = nullptr;
    SampleMiniConstants         MiniConstants   = {};
    RTXPTNrdMethod              Method          = RTXPTNrdMethod::REBLUR;
    Uint32                      PlaneIndex      = 0;
    bool                        InitOutput      = false;
    bool                        HasValidation   = false;
};
```

Extend `RTXPTPostProcessPassStats`:

```cpp
struct RTXPTPostProcessPassStats
{
    bool   Ready                         = false;
    bool   LastHdrTestExecuted           = false;
    bool   LastEdgeDetectionExecuted     = false;
    bool   LastStablePlanesDebugExecuted = false;
    bool   LastDenoiserPrepareExecuted   = false;
    bool   LastDenoiserFinalMergeExecuted = false;
    bool   LastNoDenoiserMergeExecuted   = false;
    Uint32 HdrTestDispatchCount          = 0;
    Uint32 EdgeDetectionDispatchCount    = 0;
    Uint32 StablePlanesDebugDispatchCount = 0;
    Uint32 DenoiserPrepareDispatchCount  = 0;
    Uint32 DenoiserFinalMergeDispatchCount = 0;
    Uint32 NoDenoiserMergeDispatchCount  = 0;
};
```

Add public methods:

```cpp
bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, IBuffer* pFrameConstants, bool ComputeSupported);
bool RunStablePlanesDebugViz(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs);
bool RunDenoiserPrepare(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs);
bool RunDenoiserFinalMerge(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs);
bool RunNoDenoiserFinalMerge(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs);
```

- [ ] **Step 2: Replace the two-PSO fields with pass-state array**

Replace existing `m_HdrTestPSO`, `m_EdgeDetectionPSO`, `m_HdrTestSRB`, and `m_EdgeDetectionSRB` fields with:

```cpp
struct PassState
{
    RefCntAutoPtr<IPipelineState>         PSO;
    RefCntAutoPtr<IShaderResourceBinding> SRB;
};

bool CreatePostProcessPSO(IRenderDevice*                 pDevice,
                          const ShaderCreateInfo&        BaseShaderCI,
                          RTXPTPostProcessPassId         Pass);
bool DispatchPass(IDeviceContext*                         pContext,
                  RTXPTPostProcessPassId                  Pass,
                  const RTXPTDenoiserPostProcessAttribs&  Attribs);

std::array<PassState, static_cast<size_t>(RTXPTPostProcessPassId::Count)> m_Passes;
RefCntAutoPtr<IBuffer>                                                    m_FrameConstants;
RefCntAutoPtr<IBuffer>                                                    m_MiniConstants;
```

- [ ] **Step 3: Add pass-name and mode-macro helpers in the cpp**

Add these helpers in the anonymous namespace of `RTXPTPostProcessPass.cpp`:

```cpp
const char* GetPassName(RTXPTPostProcessPassId Pass)
{
    switch (Pass)
    {
        case RTXPTPostProcessPassId::HdrTest: return "RTXPT HDR post-process test";
        case RTXPTPostProcessPassId::EdgeDetection: return "RTXPT LDR edge detection";
        case RTXPTPostProcessPassId::StablePlanesDebugViz: return "RTXPT stable planes debug viz";
        case RTXPTPostProcessPassId::RelaxDenoiserPrepareInputs: return "RTXPT RELAX denoiser prepare inputs";
        case RTXPTPostProcessPassId::ReblurDenoiserPrepareInputs: return "RTXPT REBLUR denoiser prepare inputs";
        case RTXPTPostProcessPassId::RelaxDenoiserFinalMerge: return "RTXPT RELAX denoiser final merge";
        case RTXPTPostProcessPassId::ReblurDenoiserFinalMerge: return "RTXPT REBLUR denoiser final merge";
        case RTXPTPostProcessPassId::NoDenoiserFinalMerge: return "RTXPT no-denoiser final merge";
        default: return "RTXPT unknown post-process pass";
    }
}

const char* GetModeMacro(RTXPTPostProcessPassId Pass)
{
    switch (Pass)
    {
        case RTXPTPostProcessPassId::HdrTest: return "RTXPT_POST_PROCESS_HDR_TEST";
        case RTXPTPostProcessPassId::EdgeDetection: return "RTXPT_POST_PROCESS_EDGE_DETECTION";
        case RTXPTPostProcessPassId::StablePlanesDebugViz: return "RTXPT_POST_PROCESS_STABLE_PLANES_DEBUG_VIZ";
        case RTXPTPostProcessPassId::RelaxDenoiserPrepareInputs: return "RTXPT_POST_PROCESS_RELAX_PREPARE_INPUTS";
        case RTXPTPostProcessPassId::ReblurDenoiserPrepareInputs: return "RTXPT_POST_PROCESS_REBLUR_PREPARE_INPUTS";
        case RTXPTPostProcessPassId::RelaxDenoiserFinalMerge: return "RTXPT_POST_PROCESS_RELAX_FINAL_MERGE";
        case RTXPTPostProcessPassId::ReblurDenoiserFinalMerge: return "RTXPT_POST_PROCESS_REBLUR_FINAL_MERGE";
        case RTXPTPostProcessPassId::NoDenoiserFinalMerge: return "RTXPT_POST_PROCESS_NO_DENOISER_FINAL_MERGE";
        default: return "0";
    }
}

constexpr Uint32 kThreadGroupSize = 8;
```

- [ ] **Step 4: Update initialization**

Change `Initialize` to accept `IBuffer* pFrameConstants`, store it, and create all pass variants:

```cpp
bool RTXPTPostProcessPass::Initialize(IRenderDevice*  pDevice,
                                      IEngineFactory* pEngineFactory,
                                      IBuffer*        pFrameConstants,
                                      bool            ComputeSupported)
{
    Reset();

    if (!ComputeSupported)
    {
        DEV_ERROR("RTXPT post-process pass requires compute shader support");
        return false;
    }
    if (pDevice == nullptr || pEngineFactory == nullptr || pFrameConstants == nullptr)
    {
        DEV_ERROR("RTXPT post-process pass requires a device, engine factory, and frame constants");
        return false;
    }

    m_FrameConstants = pFrameConstants;

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PostProcessing;shaders\\PathTracer", &pShaderSourceFactory);
    if (!pShaderSourceFactory)
        return false;

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.ShaderType            = SHADER_TYPE_COMPUTE;
    ShaderCI.SourceLanguage             = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler             = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags               = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.FilePath                   = "PostProcessing/RTXPTPostProcess.csh";
    ShaderCI.EntryPoint                 = "main";
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    BufferDesc ConstantsDesc;
    ConstantsDesc.Name           = "RTXPT post-process constants";
    ConstantsDesc.Size           = sizeof(RTXPTPostProcessConstants);
    ConstantsDesc.BindFlags      = BIND_UNIFORM_BUFFER;
    ConstantsDesc.Usage          = USAGE_DYNAMIC;
    ConstantsDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    pDevice->CreateBuffer(ConstantsDesc, nullptr, &m_Constants);
    if (!m_Constants)
        return false;

    for (Uint32 Index = 0; Index < static_cast<Uint32>(RTXPTPostProcessPassId::Count); ++Index)
    {
        if (!CreatePostProcessPSO(pDevice, ShaderCI, static_cast<RTXPTPostProcessPassId>(Index)))
        {
            Reset();
            return false;
        }
    }

    m_Stats.Ready = true;
    return true;
}
```

- [ ] **Step 5: Bind static and dynamic variables**

Inside `CreatePostProcessPSO`, use `GetPassName(Pass)` and `GetModeMacro(Pass)`. Add these variables to the resource layout:

```cpp
ResourceLayout
    .AddVariable(SHADER_TYPE_COMPUTE, "g_PostProcessConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "g_Const", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "g_MiniConst", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_LdrColorScratch", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_Depth", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_MotionVectors", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_SpecularHitT", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_DenoiserOutDiffRadianceHitDist", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_DenoiserOutSpecRadianceHitDist", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_DenoiserOutValidation", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_DenoiserViewspaceZ", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "t_DenoiserDisocclusionThresholdMix", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_ProcessedOutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_PostTonemapOutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_StableRadiance", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_StablePlanesHeader", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_StablePlanesBuffer", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserViewspaceZ", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserMotionVectors", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserNormalRoughness", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserDiffRadianceHitDist", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserSpecRadianceHitDist", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_DenoiserDisocclusionThresholdMix", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_COMPUTE, "u_CombinedHistoryClampRelax", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

Bind static variables:

```cpp
SetStaticVariable(State.PSO, SHADER_TYPE_COMPUTE, "g_PostProcessConstants", m_Constants, false);
SetStaticVariable(State.PSO, SHADER_TYPE_COMPUTE, "g_Const", m_FrameConstants, false);
SetStaticVariable(State.PSO, SHADER_TYPE_COMPUTE, "g_MiniConst", m_MiniConstants, false);
```

Create a `m_MiniConstants` dynamic uniform buffer with `sizeof(SampleMiniConstants)` just like `RTXPTRayTracingPass` does.

- [ ] **Step 6: Implement `DispatchPass` resource binding**

In `DispatchPass`, set `pContext->SetRenderTargets(0, nullptr, nullptr, RESOURCE_STATE_TRANSITION_MODE_TRANSITION)`, update mini constants, bind by resource requirement, commit, and dispatch:

```cpp
const RTXPTRenderTargets& RenderTargets = *Attribs.pRenderTargets;
const bool IsPrepare =
    Pass == RTXPTPostProcessPassId::RelaxDenoiserPrepareInputs ||
    Pass == RTXPTPostProcessPassId::ReblurDenoiserPrepareInputs;
const bool IsFinalMerge =
    Pass == RTXPTPostProcessPassId::RelaxDenoiserFinalMerge ||
    Pass == RTXPTPostProcessPassId::ReblurDenoiserFinalMerge;
const bool IsNoDenoiser = Pass == RTXPTPostProcessPassId::NoDenoiserFinalMerge;
const bool IsDebugViz   = Pass == RTXPTPostProcessPassId::StablePlanesDebugViz;

const bool Bound =
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_SpecularHitT", RenderTargets.GetSpecularHitTSRV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_DenoiserOutDiffRadianceHitDist", RenderTargets.GetDenoiserOutDiffRadianceHitDistSRV(Attribs.PlaneIndex), IsFinalMerge) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_DenoiserOutSpecRadianceHitDist", RenderTargets.GetDenoiserOutSpecRadianceHitDistSRV(Attribs.PlaneIndex), IsFinalMerge) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_DenoiserOutValidation", RenderTargets.GetDenoiserOutValidationSRV(), IsFinalMerge && Attribs.HasValidation) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_DenoiserViewspaceZ", RenderTargets.GetDenoiserViewspaceZSRV(), IsFinalMerge) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "t_DenoiserDisocclusionThresholdMix", RenderTargets.GetDenoiserDisocclusionThresholdMixSRV(), IsFinalMerge) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_OutputColor", Attribs.pMergeOutputUAV, IsPrepare || IsFinalMerge || IsNoDenoiser || IsDebugViz) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_StableRadiance", RenderTargets.GetStableRadianceUAV(), IsPrepare || IsNoDenoiser || IsDebugViz) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_StablePlanesHeader", RenderTargets.GetStablePlanesHeaderUAV(), IsPrepare || IsNoDenoiser || IsDebugViz) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_StablePlanesBuffer", RenderTargets.GetStablePlanesBufferUAV(), IsPrepare || IsFinalMerge || IsNoDenoiser || IsDebugViz) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserViewspaceZ", RenderTargets.GetDenoiserViewspaceZUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserMotionVectors", RenderTargets.GetDenoiserMotionVectorsUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserNormalRoughness", RenderTargets.GetDenoiserNormalRoughnessUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserDiffRadianceHitDist", RenderTargets.GetDenoiserDiffRadianceHitDistUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserSpecRadianceHitDist", RenderTargets.GetDenoiserSpecRadianceHitDistUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_DenoiserDisocclusionThresholdMix", RenderTargets.GetDenoiserDisocclusionThresholdMixUAV(), IsPrepare) &&
    SetSRBVariable(State.SRB, SHADER_TYPE_COMPUTE, "u_CombinedHistoryClampRelax", RenderTargets.GetCombinedHistoryClampRelaxUAV(), IsPrepare);
```

Dispatch dimensions:

```cpp
DispatchComputeAttribs DispatchAttribs;
DispatchAttribs.ThreadGroupCountX = (RenderTargets.GetRenderWidth() + kThreadGroupSize - 1u) / kThreadGroupSize;
DispatchAttribs.ThreadGroupCountY = (RenderTargets.GetRenderHeight() + kThreadGroupSize - 1u) / kThreadGroupSize;
DispatchAttribs.ThreadGroupCountZ = 1;
pContext->DispatchCompute(DispatchAttribs);
```

- [ ] **Step 7: Implement public G7 methods**

Use the selected NRD method to choose pass IDs:

```cpp
bool RTXPTPostProcessPass::RunDenoiserPrepare(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs)
{
    m_Stats.LastDenoiserPrepareExecuted = false;
    const RTXPTPostProcessPassId Pass =
        Attribs.Method == RTXPTNrdMethod::RELAX ?
        RTXPTPostProcessPassId::RelaxDenoiserPrepareInputs :
        RTXPTPostProcessPassId::ReblurDenoiserPrepareInputs;
    const bool Executed = DispatchPass(pContext, Pass, Attribs);
    m_Stats.LastDenoiserPrepareExecuted = Executed;
    if (Executed)
        ++m_Stats.DenoiserPrepareDispatchCount;
    return Executed;
}
```

Add the remaining public methods with explicit pass IDs:

```cpp
bool RTXPTPostProcessPass::RunDenoiserFinalMerge(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs)
{
    m_Stats.LastDenoiserFinalMergeExecuted = false;
    const RTXPTPostProcessPassId Pass =
        Attribs.Method == RTXPTNrdMethod::RELAX ?
        RTXPTPostProcessPassId::RelaxDenoiserFinalMerge :
        RTXPTPostProcessPassId::ReblurDenoiserFinalMerge;
    const bool Executed = DispatchPass(pContext, Pass, Attribs);
    m_Stats.LastDenoiserFinalMergeExecuted = Executed;
    if (Executed)
        ++m_Stats.DenoiserFinalMergeDispatchCount;
    return Executed;
}

bool RTXPTPostProcessPass::RunNoDenoiserFinalMerge(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs)
{
    m_Stats.LastNoDenoiserMergeExecuted = false;
    const bool Executed = DispatchPass(pContext, RTXPTPostProcessPassId::NoDenoiserFinalMerge, Attribs);
    m_Stats.LastNoDenoiserMergeExecuted = Executed;
    if (Executed)
        ++m_Stats.NoDenoiserMergeDispatchCount;
    return Executed;
}

bool RTXPTPostProcessPass::RunStablePlanesDebugViz(IDeviceContext* pContext, const RTXPTDenoiserPostProcessAttribs& Attribs)
{
    m_Stats.LastStablePlanesDebugExecuted = false;
    const bool Executed = DispatchPass(pContext, RTXPTPostProcessPassId::StablePlanesDebugViz, Attribs);
    m_Stats.LastStablePlanesDebugExecuted = Executed;
    if (Executed)
        ++m_Stats.StablePlanesDebugDispatchCount;
    return Executed;
}
```

- [ ] **Step 8: Preserve existing P4 methods**

Update `RunHdrTest` and `RunEdgeDetection` to use `m_Passes[HdrTest]` and `m_Passes[EdgeDetection]` while keeping their current behavior and stats.

- [ ] **Step 9: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds through `RTXPTPostProcessPass.cpp`. If shader compilation fails on unused reflected variables, lower those bindings to `Required=false` like `RTXPTDenoisingGuidesBaker.cpp` does for optimized-out variables.

- [ ] **Step 10: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp
git commit -m "feat(rtxpt): add G7 post-process dispatch API"
```

---

### Task 5: Attach G7 Pass to the Post-Process Pipeline

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`

- [ ] **Step 1: Include and store `RTXPTPostProcessPass`**

In `RTXPTPostProcessPipeline.hpp`, add:

```cpp
#include "RTXPTPostProcessPass.hpp"
```

Extend stats:

```cpp
bool RealtimeMergeStageReady     = false;
bool LastRealtimeFinalMergeReady = false;
```

Add public methods:

```cpp
bool RunDenoiserPrepare(IDeviceContext*         pContext,
                        const RTXPTRenderTargets& RenderTargets,
                        RTXPTNrdMethod         Method,
                        Uint32                 PlaneIndex,
                        bool                   InitOutput);

bool RunDenoiserFinalMerge(IDeviceContext*          pContext,
                           const RTXPTRenderTargets& RenderTargets,
                           RTXPTNrdMethod          Method,
                           Uint32                  PlaneIndex,
                           bool                    HasValidation);

bool RunNoDenoiserFinalMerge(IDeviceContext*           pContext,
                             const RTXPTRenderTargets& RenderTargets);
```

Change initialize signature:

```cpp
bool Initialize(IRenderDevice*  pDevice,
                IEngineFactory* pEngineFactory,
                ISwapChain*     pSwapChain,
                IBuffer*        pFrameConstants,
                bool            ComputeSupported);
```

Add private member:

```cpp
RTXPTPostProcessPass m_PostProcessPass;
```

- [ ] **Step 2: Initialize/reset the pass**

In `Reset()`:

```cpp
m_PostProcessPass.Reset();
```

In `Initialize()`:

```cpp
m_Stats.RealtimeMergeStageReady =
    m_PostProcessPass.Initialize(pDevice, pEngineFactory, pFrameConstants, ComputeSupported);
if (!m_Stats.RealtimeMergeStageReady)
{
    DEV_ERROR("RTXPT realtime post-process pass failed to initialize");
    return false;
}
```

- [ ] **Step 3: Implement wrapper methods**

Add this helper near the top of `RTXPTPostProcessPipeline.cpp`:

```cpp
RTXPTDenoiserPostProcessAttribs MakeRealtimePostProcessAttribs(const RTXPTRenderTargets& RenderTargets)
{
    RTXPTDenoiserPostProcessAttribs Attribs;
    Attribs.pRenderTargets  = &RenderTargets;
    Attribs.pMergeOutputUAV = RenderTargets.GetAccumulationOutputUAV();
    return Attribs;
}
```

Implement no-denoiser:

```cpp
bool RTXPTPostProcessPipeline::RunNoDenoiserFinalMerge(IDeviceContext*           pContext,
                                                       const RTXPTRenderTargets& RenderTargets)
{
    RTXPTDenoiserPostProcessAttribs Attribs = MakeRealtimePostProcessAttribs(RenderTargets);
    const bool Executed = m_PostProcessPass.RunNoDenoiserFinalMerge(pContext, Attribs);
    m_Stats.LastRealtimeFinalMergeReady = Executed;
    if (!Executed)
        DEV_ERROR("RTXPT no-denoiser final merge failed");
    return Executed;
}
```

Implement prepare:

```cpp
bool RTXPTPostProcessPipeline::RunDenoiserPrepare(IDeviceContext*           pContext,
                                                  const RTXPTRenderTargets& RenderTargets,
                                                  RTXPTNrdMethod            Method,
                                                  Uint32                    PlaneIndex,
                                                  bool                      InitOutput)
{
    RTXPTDenoiserPostProcessAttribs Attribs = MakeRealtimePostProcessAttribs(RenderTargets);
    Attribs.Method                  = Method;
    Attribs.PlaneIndex              = PlaneIndex;
    Attribs.InitOutput              = InitOutput;
    Attribs.MiniConstants.params.x  = PlaneIndex;
    Attribs.MiniConstants.params.y  = InitOutput ? 1u : 0u;
    return m_PostProcessPass.RunDenoiserPrepare(pContext, Attribs);
}
```

Implement final merge:

```cpp
bool RTXPTPostProcessPipeline::RunDenoiserFinalMerge(IDeviceContext*           pContext,
                                                     const RTXPTRenderTargets& RenderTargets,
                                                     RTXPTNrdMethod            Method,
                                                     Uint32                    PlaneIndex,
                                                     bool                      HasValidation)
{
    RTXPTDenoiserPostProcessAttribs Attribs = MakeRealtimePostProcessAttribs(RenderTargets);
    Attribs.Method                  = Method;
    Attribs.PlaneIndex              = PlaneIndex;
    Attribs.HasValidation           = HasValidation;
    Attribs.MiniConstants.params.x  = PlaneIndex;
    Attribs.MiniConstants.params.y  = HasValidation ? 1u : 0u;

    const bool Executed = m_PostProcessPass.RunDenoiserFinalMerge(pContext, Attribs);
    m_Stats.LastRealtimeFinalMergeReady = Executed;
    if (!Executed)
        DEV_ERROR("RTXPT denoiser final merge failed");
    return Executed;
}
```

- [ ] **Step 4: Update validation**

In `ValidateRenderTargets`, require realtime resources only when they were requested:

```cpp
const bool RealtimeResourcesValid =
    !RenderTargets.AreRealtimeRenderTargetsRequested() ||
    (RenderTargets.HasRealtimeRenderTargets() &&
     RenderTargets.GetCombinedHistoryClampRelaxUAV() != nullptr &&
     RenderTargets.GetAccumulationOutputUAV() != nullptr);
```

Include it in `m_Stats.ResourcesValid`.

- [ ] **Step 5: Update `CreatePhase4Passes` call site later in Task 6**

Do not leave the build broken: Task 6 updates `RTXPTSample.cpp` to pass `m_FrameConstantsCB`.

- [ ] **Step 6: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp
git commit -m "feat(rtxpt): attach G7 post-process pipeline"
```

---

### Task 6: Wire Realtime Final Merge and Presentation

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Update the pipeline initialization call**

In `CreatePhase4Passes`, change the pipeline initialization call to pass `m_FrameConstantsCB`:

```cpp
m_PostProcessPipeline.Initialize(m_pDevice,
                                 m_pEngineFactory,
                                 m_pSwapChain,
                                 m_FrameConstantsCB,
                                 m_FeatureCaps.ComputeShaders);
```

- [ ] **Step 2: Add sample helper declarations**

In `RTXPTSample.hpp`, add:

```cpp
bool RunRealtimePostProcess();
bool RunRealtimeNoDenoiserFinalMerge();
bool PresentRealtimeFinalOutput();
```

- [ ] **Step 3: Implement no-denoiser final merge**

In `RTXPTSample.cpp`, add:

```cpp
bool RTXPTSample::RunRealtimeNoDenoiserFinalMerge()
{
    if (!m_RenderTargets.HasRealtimeRenderTargets())
    {
        RecordRealtimePathTraceStatus("Realtime render targets are missing for final merge");
        return false;
    }

    const bool MergeOk =
        m_PostProcessPipeline.RunNoDenoiserFinalMerge(m_pImmediateContext, m_RenderTargets);
    m_LastRealtimeFinalMergeReady = MergeOk;
    if (!MergeOk)
        RecordRealtimePathTraceStatus("NoDenoiserFinalMerge dispatch failed");
    return MergeOk;
}
```

- [ ] **Step 4: Implement final output presentation**

Add:

```cpp
bool RTXPTSample::PresentRealtimeFinalOutput()
{
    const RTXPTSuperResolutionSettings DisabledSuperResolution;
    const RTXPTSuperResolutionFrameDesc FrameDesc =
        m_PostProcessPipeline.ResolveSuperResolutionFrameDesc(DisabledSuperResolution,
                                                              m_RenderTargets.GetDisplayWidth(),
                                                              m_RenderTargets.GetDisplayHeight(),
                                                              m_RenderTargets.GetProcessedOutputColorFormat(),
                                                              HasRealtimeResetFlag(m_CurrentFrameRealtimeReset, RTXPT_REALTIME_RESET_TAA_SR_HISTORY),
                                                              m_LastElapsedTimeSeconds);

    if (FrameDesc.Enabled)
    {
        const bool SROk = m_PostProcessPipeline.RunSuperResolution(m_pImmediateContext,
                                                                   m_RenderTargets,
                                                                   FrameDesc,
                                                                   m_CameraNearPlane,
                                                                   m_CameraFarPlane,
                                                                   m_CameraVerticalFov);
        if (!SROk)
        {
            RecordRealtimePathTraceStatus("Realtime super-resolution pass failed");
            return false;
        }
    }

    RTXPTBloomParameters BloomParams;
    BloomParams.Enabled   = m_ReferenceUI.EnableBloom;
    BloomParams.Radius    = std::clamp(m_ReferenceUI.BloomRadius, 0.0f, 64.0f);
    BloomParams.Intensity = std::clamp(m_ReferenceUI.BloomIntensity, 0.0f, 0.1f);

    if (!m_PostProcessPipeline.RunPreToneMapping(m_pImmediateContext, m_RenderTargets, BloomParams))
    {
        RecordRealtimePathTraceStatus("Realtime pre-tone post-process failed");
        return false;
    }

    if (!m_PostProcessPipeline.RunToneMapping(m_pImmediateContext,
                                             m_RenderTargets,
                                             m_ReferenceUI.ToneMapping,
                                             m_ReferenceUI.EnableToneMapping))
    {
        RecordRealtimePathTraceStatus("Realtime tone mapping failed");
        return false;
    }

    ITextureView* pPresentationSRV = m_RenderTargets.GetPresentationSRV();
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pPresentationSRV))
    {
        RecordRealtimePathTraceStatus("Realtime presentation blit failed");
        return false;
    }

    return true;
}
```

- [ ] **Step 5: Implement realtime post-process routing**

Add:

```cpp
bool RTXPTSample::RunRealtimePostProcess()
{
    // G8 replaces this branch with prepare -> NRD -> final merge when standalone NRD is available.
    const bool UseStandaloneDenoiser = false;

    if (UseStandaloneDenoiser)
    {
        RecordRealtimePathTraceStatus("Standalone NRD final merge is deferred to G8");
        return false;
    }

    if (!RunRealtimeNoDenoiserFinalMerge())
        return false;

    return PresentRealtimeFinalOutput();
}
```

- [ ] **Step 6: Replace the realtime fallback clear**

In `RunRealtimePathTraceOnly`, replace:

```cpp
// FILL_STABLE_PLANES writes stable-plane radiance storage, not final OutputColor.
// G7/G9 will replace this fallback with NoDenoiserFinalMerge or NRD final merge.
ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
return true;
```

with:

```cpp
if (!RunRealtimePostProcess())
{
    ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
    return false;
}

return true;
```

- [ ] **Step 7: Update debug UI status text**

Replace:

```cpp
ImGui::Text("Realtime final merge: pending G7/G9");
```

with:

```cpp
ImGui::Text("Realtime final merge: %s", m_LastRealtimeFinalMergeReady ? "ready" : "not dispatched");
```

- [ ] **Step 8: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds and no call sites use the old `RTXPTPostProcessPipeline::Initialize` signature.

- [ ] **Step 9: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): route realtime through G7 final merge"
```

---

### Task 7: Add G8 Handoff Calls for Prepare and Final Merge

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`

- [ ] **Step 1: Add a G8 handoff comment and call shape**

In `RunRealtimePostProcess`, replace the current standalone branch body with this call shape while keeping `UseStandaloneDenoiser = false` until G8 enables NRD:

```cpp
if (UseStandaloneDenoiser)
{
    for (Uint32 PlaneIndex = 0; PlaneIndex < static_cast<Uint32>(m_RealtimeUI.StablePlanesActiveCount); ++PlaneIndex)
    {
        const bool InitOutput = PlaneIndex == 0;
        if (!m_PostProcessPipeline.RunDenoiserPrepare(m_pImmediateContext,
                                                      m_RenderTargets,
                                                      m_RealtimeUI.NRDMethod,
                                                      PlaneIndex,
                                                      InitOutput))
        {
            RecordRealtimePathTraceStatus("Denoiser prepare dispatch failed");
            return false;
        }

        // G8 inserts RTXPTNrdIntegration::Dispatch() here for this stable plane.

        const bool HasValidation = m_RenderTargets.GetDenoiserOutValidationSRV() != nullptr;
        if (!m_PostProcessPipeline.RunDenoiserFinalMerge(m_pImmediateContext,
                                                         m_RenderTargets,
                                                         m_RealtimeUI.NRDMethod,
                                                         PlaneIndex,
                                                         HasValidation))
        {
            RecordRealtimePathTraceStatus("Denoiser final merge dispatch failed");
            return false;
        }
    }

    return PresentRealtimeFinalOutput();
}
```

- [ ] **Step 2: Keep standalone NRD disabled until G8**

Keep:

```cpp
const bool UseStandaloneDenoiser = false;
```

Do not change `kRTXPTStandaloneNrdAvailable = false` in G7.

- [ ] **Step 3: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds; `RunDenoiserPrepare` and `RunDenoiserFinalMerge` are compiled and reachable but not executed while NRD is disabled.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp
git commit -m "feat(rtxpt): add G7 NRD handoff call shape"
```

---

### Task 8: Update Mapping Documentation

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add a G7 mapping section**

Append:

```markdown
## Realtime G7 PostProcess Denoiser Prepare and Final Merge

| RTXPT-fork source | Diligent port | Notes |
|---|---|---|
| `ProcessingPasses/PostProcess.h::ComputePassType` | `src/RTXPTPostProcessPass.hpp::RTXPTPostProcessPassId` | Diligent enum mirrors stable-plane debug, RELAX/REBLUR prepare, RELAX/REBLUR final merge, and no-denoiser final merge. |
| `ProcessingPasses/PostProcess.cpp::PostProcess` | `src/RTXPTPostProcessPass.cpp::Initialize` | Creates one Diligent compute PSO/SRB per G7 mode using `RTXPT_POST_PROCESS_MODE`. |
| `ProcessingPasses/PostProcess.cpp::Apply` | `src/RTXPTPostProcessPass.cpp::DispatchPass` | Binds `SampleConstants`, `SampleMiniConstants`, stable-plane resources, denoiser input/output textures, validation SRV, and merge work UAV. |
| `ProcessingPasses/PostProcess.hlsl::DENOISER_PREPARE_INPUTS` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` prepare modes | Writes NRD input resources and initializes merge work output from `StableRadiance` on the first processed plane. |
| `ProcessingPasses/PostProcess.hlsl::DENOISER_FINAL_MERGE` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` final merge modes | Reads per-plane NRD outputs, remodulates with stable-plane BSDF estimates, and adds radiance into the merge work target. |
| `ProcessingPasses/PostProcess.hlsl::NO_DENOISER_FINAL_MERGE` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` no-denoiser mode | Combines stable radiance plus noisy stable-plane radiance into the same downstream merge target. |
| `ProcessingPasses/PostProcess.hlsl::STABLE_PLANES_DEBUG_VIZ` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` stable-plane debug mode | Provides lightweight stable-plane debug output without porting RTXPT-fork `ShaderDebug`. |
| `NRD/DenoiserNRD.hlsli` | `assets/shaders/PostProcessing/RTXPTDenoiserNRD.hlsli` | Wrapper compiles before G8 and can forward to NRD headers after the NRD dependency gate exists. |
```

- [ ] **Step 2: Verify mapping references**

Run:

```powershell
rg -n "Realtime G7|RTXPTPostProcessPassId|RTXPTDenoiserNRD" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: all three references are present.

- [ ] **Step 3: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map G7 post-process port"
```

---

### Task 9: Verification

**Files:**

- No source edits unless verification finds a defect.

- [ ] **Step 1: Build RTXPT**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds with no errors from `RTXPTPostProcessPass.cpp` or `RTXPTPostProcess.csh`.

- [ ] **Step 2: Run targeted source checks**

Run:

```powershell
rg -n "RunNoDenoiserFinalMerge|RunDenoiserPrepare|RunDenoiserFinalMerge|RTXPT_POST_PROCESS_NO_DENOISER_FINAL_MERGE|TODO\\(RTXPT-Realtime-DLSS-RR\\)" DiligentSamples/Samples/RTXPT
```

Expected: each G7 symbol appears in source.

- [ ] **Step 3: Run no-NRD smoke**

Run:

```powershell
build\x64\Debug\DiligentSamples\Samples\RTXPT\Debug\RTXPT.exe
```

Expected:

- The sample opens.
- Reference mode still renders.
- Realtime mode no longer shows the old dark fallback from `RunRealtimePathTraceOnly`.
- With standalone NRD disabled, realtime presents a tone-mapped image produced by `NoDenoiserFinalMerge`.
- The debug UI line reads `Realtime final merge: ready` after a realtime frame.
- `NRD availability` still says the G8 disabled reason.

- [ ] **Step 4: Smoke resize and toggles**

In the sample UI:

- Toggle `Realtime Mode` on/off.
- Toggle `Use standalone denoiser (NRD)` and confirm it remains disabled with the G8 reason.
- Change `Active stable planes`.
- Resize the window.
- Move the camera.

Expected:

- No fallback clear appears unless a real dispatch/build error occurs.
- Realtime target status remains allocated.
- `CombinedHistoryClampRelax` exists when realtime resources are requested.
- Reference mode behavior is unchanged after returning from realtime mode.

- [ ] **Step 5: Run format validation if available**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentCore-ValidateFormatting
```

Expected: formatting target passes. If the target is unavailable in this checkout, run the repository's documented format validation command or state that it could not be run.

- [ ] **Step 6: Final commit**

If Task 9 required fixes:

```bash
git add DiligentSamples/Samples/RTXPT
git commit -m "fix(rtxpt): stabilize G7 post-process verification"
```

If Task 9 required no fixes, do not create an empty commit.

---

## Self-Review

Spec coverage:

- `RELAXDenoiserPrepareInputs`: Task 3 shader branch, Task 4 pass API, Task 5 pipeline wrapper, Task 7 G8 call shape.
- `REBLURDenoiserPrepareInputs`: Task 3 shader branch, Task 4 pass API, Task 5 pipeline wrapper, Task 7 G8 call shape.
- `RELAXDenoiserFinalMerge`: Task 3 shader branch, Task 4 pass API, Task 5 pipeline wrapper, Task 7 G8 call shape.
- `REBLURDenoiserFinalMerge`: Task 3 shader branch, Task 4 pass API, Task 5 pipeline wrapper, Task 7 G8 call shape.
- `NoDenoiserFinalMerge`: Task 3 shader branch, Task 4 pass API, Task 5 pipeline wrapper, Task 6 realtime route.
- `StablePlanesDebugViz`: Task 3 shader branch and Task 4 dispatch API.
- DLSS-RR deferred marker: Task 3 Step 6.
- Prepare stable-plane iteration through mini constants: Task 3 Step 3 and Task 5 Step 3.
- Initialize merge output with `StableRadiance` on first plane: Task 3 Step 3 and Task 5 Step 3.
- Write denoiser inputs and `CombinedHistoryClampRelax`: Task 3 Step 3 and Task 1 Step 3.
- RELAX/REBLUR packing: Task 2 helper wrapper and Task 3 prepare branch.
- Sky/no-surface view-Z sentinel: Task 2 and Task 3 Step 3.
- Final merge per-plane diffuse/spec reads: Task 3 Step 4 and Task 4 Step 6.
- Remodulation through NRD post-denoise process: Task 2 and Task 3 Step 4.
- Validation/debug output when validation resource exists: Task 3 Step 4 and Task 4 Step 6.
- Standalone denoiser off realtime image: Task 6 and Task 9.
- Standalone denoiser on prepare -> NRD -> final merge call shape: Task 7, with actual NRD dispatch deferred to G8 per spec phase split.
- Merge result consumed by AA/SR/tone mapping: Task 5 uses `GetAccumulationOutputUAV()` and Task 6 calls SR/pre-tone/tone/presentation.

Placeholder scan:

- The required `TODO(RTXPT-Realtime-DLSS-RR)` marker is intentional and names the deferred path.
- No unspecified open-ended implementation placeholders are used.

Type consistency:

- C++ pass IDs use `RTXPTPostProcessPassId`.
- Public dispatch uses `RTXPTDenoiserPostProcessAttribs`.
- Pipeline wrappers use `RTXPTNrdMethod`, `PlaneIndex`, `InitOutput`, and `HasValidation`.
- Shader resource names match C++ dynamic variable names.
