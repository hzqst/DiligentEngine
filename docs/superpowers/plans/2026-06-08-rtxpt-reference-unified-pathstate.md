# RTXPT Reference Unified PathState Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route the Diligent RTXPT reference variant through the local `PathState` / `PathPayload` / `PathTracer::HandleHit` / `PathTracer::HandleMiss` spine while preserving the current reference image contract.

**Architecture:** Keep one mode-aware `PathTracer::WorkingContext` and one shared state-transport spine. Reference mode gets explicit branches for raw HDR radiance, one-shot primary depth capture, zero motion vectors, MIS/firefly/reference NEE behavior, Russian roulette, bounded termination, and stable-plane side-effect suppression; realtime build/fill stable-plane behavior remains isolated behind mode guards.

**Tech Stack:** HLSL 6.5 DXR shaders, Diligent Engine RTXPT sample, C++ RT PSO/resource binding.

---

## Plan Boundaries

This plan intentionally does not execute or prescribe baseline capture, screenshot capture, image-diff comparison, automated runtime smoke, automated static validation, or build commands. Those checks belong to a separate human/CI gate outside this implementation plan.

The implementation still must be reviewable. Each task below includes manual review notes, but no executable verification step.

Do not revert unrelated dirty work. At plan creation time the worktree had `M DiligentSamples` and `?? docs/realtime_bxdf_diff.md`.

Keep `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp::MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 40)` through this plan.

Do not bind stable-plane UAVs for the reference variant.

Do not switch a reference hit/miss shader to `PathPayload` while reference raygen still traces `RTXPTMaterialHitPayload`; perform the payload switch as one source checkpoint.

## File Structure

- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`: expose source-compatible output UAV names to reference, build `WorkingContext` for all modes, keep stable-plane UAV declarations non-reference only.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`: make `WorkingContext` mode-aware, add reference output fields, add emissive-hit PDF to `SurfaceData`.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli`: add reference-only primary-depth accessors that reuse `stableBranchID` for depth and `stablePlaneIndex` as the reference-only captured flag.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`: compile the spine under reference, add reference accumulation/commit/MIS/firefly/NEE/diffuse-bounce/Russian-roulette behavior, and guard stable-plane side effects.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`: make `LoadCurrentSurfaceData` available to reference and realtime, compute emissive-hit solid-angle PDF, then switch all variants to `PathPayload`.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`: switch reference miss to unpack/handle/pack `PathPayload`.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerVisibilityMiss.rmiss`: switch reference visibility miss to terminate `PathPayload`.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`: first normalize reference output names while the flattened loop stays active, then replace reference raygen with the unified `PathState` loop.
- Modify `docs/realtime_bxdf_diff.md`: record that reference now uses the local unified spine and list remaining Diligent bridge differences.
- Modify `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`: record any remaining reference-specific branches in shared spine functions.

---

### Task 1: Source-Compatible Reference Output Context

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Expose `PathTracerTypes.hlsli` and common output UAV names to reference**

In `PathTracerBridge.hlsli`, include `PathTracerTypes.hlsli` unconditionally and declare the common output resources for all modes:

```hlsl
#include "PathTracerTypes.hlsli"

VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_OutputColor;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>  u_Depth;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4> u_MotionVectors;

#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
VK_IMAGE_FORMAT("r32ui")   RWTexture2D<uint>         u_Throughput;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>        u_SpecularHitT;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_StableRadiance;
VK_IMAGE_FORMAT("r32ui")   RWTexture2DArray<uint>    u_StablePlanesHeader;
RWStructuredBuffer<StablePlane>                      u_StablePlanesBuffer;
#endif
```

- [ ] **Step 2: Make `WorkingContext` mode-aware**

In `PathTracerTypes.hlsli`, replace `WorkingContext` with:

```hlsl
    struct WorkingContext
    {
        RWTexture2D<float4> OutputColor;
        RWTexture2D<float>  Depth;
        RWTexture2D<float4> MotionVectors;
        PathTracerConstants PtConsts;

#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
        StablePlanesContext StablePlanes;
#endif
    };
```

- [ ] **Step 3: Return a mode-aware context**

In `PathTracerBridge.hlsli`, define `GetWorkingContext()` for all modes:

```hlsl
PathTracer::WorkingContext GetWorkingContext()
{
    PathTracer::WorkingContext ret;
    ret.PtConsts      = g_Const.ptConsts;
    ret.OutputColor   = u_OutputColor;
    ret.Depth         = u_Depth;
    ret.MotionVectors = u_MotionVectors;

#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
    ret.StablePlanes  = StablePlanesContext::make(u_StablePlanesHeader,
                                                  u_StablePlanesBuffer,
                                                  u_StableRadiance,
                                                  g_Const.ptConsts);
#endif
    return ret;
}
```

- [ ] **Step 4: Keep flattened reference raygen active but write source-compatible outputs**

In `PathTracerSample.rgen`, remove reference-only declarations of `u_Output` and `u_ScreenMotionVectors`. Keep `u_Depth` only through `PathTracerBridge.hlsli`.

Replace the current final writes with:

```hlsl
    u_OutputColor[pixel]   = float4(pathRadiance, 1.0);
    u_Depth[pixel]         = primaryDepth;
    u_MotionVectors[pixel] = float4(0.0, 0.0, 0.0, 0.0);
```

- [ ] **Step 5: Commit this source checkpoint**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "refactor(rtxpt): expose reference output context" --trailer "Co-Authored-By: GPT 5.5"
```

Manual review notes: reference still uses the old flattened raygen and old material-hit payload after this task; stable-plane UAV fields remain non-reference only.

---

### Task 2: Compile The PathState Spine Under Reference

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli`

- [ ] **Step 1: Include `PathState` and `PathPayload` for reference**

In `PathTracer.hlsli`, replace the reference-excluding include block with:

```hlsl
#include "PathState.hlsli"
#include "PathPayload.hlsli"

#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
#    include "PathTracerStablePlanes.hlsli"
#endif
```

- [ ] **Step 2: Move shared spine functions out of the non-reference-only branch**

Use this guard map while restructuring the large reference/non-reference split:

```text
All modes:
  HasFinishedSurfaceBounces
  EmptyPathInitialize
  StartPixel
  SetupPathPrimaryRay
  UpdatePathThroughput
  ShouldCollectGISecondaryRadiance
  UpdateSurfaceOutsideIoR
  HandleNestedDielectrics
  AccumulatePathRadiance
  AccumulateNEERadiance
  CommitPixel
  HandleRussianRoulette
  UpdatePathTravelled
  HandleMiss
  HandleHit

Reference only, temporary until final cleanup:
  RTXPTMaterialHitPayload MakeEmptyPayload

Non-build modes, reference + fill:
  MakeBSDFSample
  UpdateNestedDielectricsOnScatterTransmission
  HandleNEE
  GenerateScatterRay

Reference only:
  ComputeBSDFEnvMISWeight
  ReferenceShouldRunNEE
  ApplyReferenceFireflyFilter

Build/fill only:
  PathTracerStablePlanes.hlsli include
  StablePlanesHandleMiss
  StablePlanesHandleHit

Fill only:
  StablePlanesOnScatter
  Bridge::ExportSpecHitTStart
  Bridge::ExportSpecHitTStop
```

- [ ] **Step 3: Add reference primary-depth state**

In `PathState.hlsli`, add these accessors after `stablePlaneIndex`:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
    void InitReferencePrimaryDepth(float depth)
    {
        stableBranchID   = asuint(depth);
        stablePlaneIndex = 0u;
    }

    bool HasReferencePrimaryDepth() { return stablePlaneIndex != 0u; }

    void CaptureReferencePrimaryDepth(float depth)
    {
        if (!HasReferencePrimaryDepth())
        {
            stableBranchID   = asuint(depth);
            stablePlaneIndex = 1u;
        }
    }

    float GetReferencePrimaryDepth() { return asfloat(stableBranchID); }
#endif
```

In `EmptyPathInitialize`, initialize this state:

```hlsl
        path.setStablePlaneIndex(0);
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        path.InitReferencePrimaryDepth(g_Const.ptConsts.camera.FarZ);
#else
        path.stableBranchID = 1u;
#endif
```

Manual review note: `CaptureReferencePrimaryDepth` must be one-shot and must not depend on `path.getVertexIndex() == 1u`. The old flattened branch captured depth from the first primary `TraceRay` hit before nested-dielectric rejection could continue the path.

- [ ] **Step 4: Guard stable-plane start from reference**

In `StartPixel`, use:

```hlsl
    inline void StartPixel(const PathState path, const WorkingContext workingContext)
    {
#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE || defined(__INTELLISENSE__)
        workingContext.StablePlanes.StartPixel(path.GetPixelPos());
#endif

#if PATH_TRACER_MODE == PATH_TRACER_MODE_BUILD_STABLE_PLANES
        Bridge::ExportSurfaceInit(path.GetPixelPos());
#endif
    }
```

- [ ] **Step 5: Add reference accumulation and commit**

In `AccumulatePathRadiance`, add the reference branch:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_BUILD_STABLE_PLANES
        workingContext.StablePlanes.AccumulateStableRadiance(path.GetPixelPos(), radiance);
#elif PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        float4 newL = path.GetL();
        newL.rgb += radiance;
        path.SetL(newL);
#elif PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES
        if (!stablePlaneOnBranch)
        {
            float4 newL = float4(radiance, specularRadianceAvg) * Bridge::getNoisyRadianceAttenuation();
            path.SetL(path.GetL() + newL);
        }
#else
#    error Unsupported PATH_TRACER_MODE.
#endif
```

In `CommitPixel`, add the reference branch:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_BUILD_STABLE_PLANES
        // Stable radiance and stable-plane data are committed incrementally while tracing.
#elif PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        const uint2 pixelPos = path.GetPixelPos();
        workingContext.OutputColor[pixelPos]   = float4(path.GetL().rgb, 1.0);
        workingContext.Depth[pixelPos]         = path.GetReferencePrimaryDepth();
        workingContext.MotionVectors[pixelPos] = float4(0.0, 0.0, 0.0, 0.0);
#elif PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES
        workingContext.StablePlanes.CommitDenoiserRadiance(path);
#else
#    error Unsupported PATH_TRACER_MODE.
#endif
```

- [ ] **Step 6: Guard stable-plane miss/scatter side effects**

Guard `StablePlanesHandleMiss`, `StablePlanesHandleHit`, `StablePlanesOnScatter`, and spec-hit guide export so reference mode cannot call them.

The `StablePlanesOnScatter` and spec-hit guide export block in `GenerateScatterRay` must be wrapped in:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES || defined(__INTELLISENSE__)
        // existing spec-hit guide export and StablePlanesOnScatter body
#endif
```

- [ ] **Step 7: Commit this source checkpoint**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli
git commit -m "refactor(rtxpt): compile reference PathState spine" --trailer "Co-Authored-By: GPT 5.5"
```

Manual review notes: the old flattened reference raygen remains active after this task; payload switching has not started.

---

### Task 3: Preserve Reference Quality In The Shared Spine

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Add emissive-hit PDF to `SurfaceData`**

In `PathTracerTypes.hlsli::SurfaceData`, add:

```hlsl
        float emissiveLightPdf;
```

Update `SurfaceData::make` to accept and assign `emissiveLightPdf` as its final argument.

- [ ] **Step 2: Compute emissive solid-angle PDF in `LoadCurrentSurfaceData`**

In `PathTracerClosestHit.rchit::LoadCurrentSurfaceData`, add `float emissiveLightPdf = 0.0;` and compute it after `surfaceEmission = Bridge::getEmission(material, texCoord);`:

```hlsl
        if ((material.flags & kMaterialFlagEmissiveAreaLight) != 0u)
        {
            const float3 wp0   = mul(ObjectToWorld3x4(), float4(V0.position, 1.0));
            const float3 wp1   = mul(ObjectToWorld3x4(), float4(V1.position, 1.0));
            const float3 wp2   = mul(ObjectToWorld3x4(), float4(V2.position, 1.0));
            const float3 ng    = cross(wp1 - wp0, wp2 - wp0);
            const float  ngLen = length(ng);
            const float  area  = 0.5 * ngLen;
            if (area > 1e-9)
            {
                const float3 normal   = ng / ngLen;
                const float  cosTheta = abs(dot(normal, -rayDir));
                if (cosTheta > 2e-9)
                    emissiveLightPdf = min(kMaxSolidAnglePdf, (1.0 / area) * (RayTCurrent() * RayTCurrent()) / cosTheta);
            }
        }
```

Pass `emissiveLightPdf` into `SurfaceData::make`.

- [ ] **Step 3: Add reference NEE/firefly helpers**

In `PathTracer.hlsli`, after `ComputeBSDFEnvMISWeight`, add:

```hlsl
    inline bool ReferenceShouldRunNEE(const PathState preScatterPath, const WorkingContext workingContext)
    {
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        const uint maxBounces    = max(workingContext.PtConsts.bounceCount, 1u);
        const uint maxNEEBounces = min(workingContext.PtConsts.maxNEEBounceCount, maxBounces);
        return preScatterPath.getVertexIndex() <= maxNEEBounces;
#else
        return true;
#endif
    }

    inline float3 ApplyReferenceFireflyFilter(float3 radiance, const PathState path, const WorkingContext workingContext)
    {
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        const float ffThreshold = workingContext.PtConsts.fireflyFilterThreshold;
        if (ffThreshold != 0.0)
            radiance = FireflyFilter(radiance, ffThreshold, path.GetFireflyFilterK());
#endif
        return radiance;
    }
```

Use `ReferenceShouldRunNEE` in `HandleNEE` after the `fullSamples` check.

- [ ] **Step 4: Preserve environment and emissive-hit MIS**

In `HandleMiss`, apply reference environment MIS and firefly filtering before the stable-plane miss branch:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        const NEEBSDFMISInfo prevMISInfo = NEEBSDFMISInfo::Unpack16bit(path.GetPackedMISInfo());
        const bool didEnvNEE =
            prevMISInfo.LightSamplingEnabled &&
            ((workingContext.PtConsts.environmentNEEEnabled & 1u) != 0u);
        environmentEmission *= ComputeBSDFEnvMISWeight(didEnvNEE, path.GetBsdfScatterPdf(), rayDir);
        environmentEmission = ApplyReferenceFireflyFilter(environmentEmission, path, workingContext);
#endif
```

In `HandleHit`, capture reference depth immediately after `UpdatePathTravelled(...)` and before nested-dielectric rejection can return:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        path.CaptureReferencePrimaryDepth(rayTCurrent);
#endif
```

Then apply emissive-hit MIS and firefly filtering before emission accumulation:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        float3 referenceSurfaceEmission = surfaceEmission;
        const NEEBSDFMISInfo prevMISInfo = NEEBSDFMISInfo::Unpack16bit(path.GetPackedMISInfo());
        const bool didEmissiveNEE =
            prevMISInfo.LightSamplingEnabled &&
            prevMISInfo.FullSamples > 0u &&
            Bridge::getEmissiveTriangleCount() > 0u;
        if (didEmissiveNEE && path.GetBsdfScatterPdf() > 0.0 && surfaceData.emissiveLightPdf > 0.0)
        {
            referenceSurfaceEmission *= ComputeBSDFMISForEmissiveTriangle(path.GetPixelPos(),
                                                                          path.GetBsdfScatterPdf(),
                                                                          surfaceData.emissiveLightPdf,
                                                                          prevMISInfo.FullSamples);
        }
        referenceSurfaceEmission = ApplyReferenceFireflyFilter(referenceSurfaceEmission, path, workingContext);
#else
        float3 referenceSurfaceEmission = surfaceEmission;
#endif
```

Use `referenceSurfaceEmission` in the existing emission accumulation block.

- [ ] **Step 5: Preserve reference diffuse-bounce classification**

In `GenerateScatterRay`, use the reference classification under reference mode:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        const bool isDiffuseBounce =
            ((lobe & kBSDFLobeDiffuseReflection) != 0u) ||
            (((lobe & kBSDFLobeTransmission) == 0u) &&
             surfaceData.bsdf.standardData.roughness > kSpecularRoughnessThreshold);
        if (isDiffuseBounce)
            path.incrementCounter(PackedCounters::DiffuseBounces);
#else
        const bool isDiffuseBounce =
            ((lobe & (kBSDFLobeDiffuseReflection | kBSDFLobeDiffuseTransmission)) != 0u) ||
            surfaceData.bsdf.standardData.roughness > kSpecularRoughnessThreshold;
        if (isDiffuseBounce &&
            !(((lobe & kBSDFLobeDiffuseTransmission) != 0u) && ((path.getVertexIndex() % 2u) == 1u)))
            path.incrementCounter(PackedCounters::DiffuseBounces);
#endif
```

- [ ] **Step 6: Preserve Russian roulette and `minBounceCount`**

Replace the current `HandleRussianRoulette` stub with:

```hlsl
    inline bool HandleRussianRoulette(inout PathState path,
                                      const PathState preScatterPath,
                                      const WorkingContext workingContext)
    {
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        if (preScatterPath.getVertexIndex() <= workingContext.PtConsts.minBounceCount)
            return true;

        SampleGenerator sgRR = SampleGenerator_makeStateless(preScatterPath.GetPixelPos(),
                                                             preScatterPath.getVertexIndex(),
                                                             Bridge::getSampleIndex(),
                                                             kSampleEffect_RussianRoulette);
        const float3 thp     = path.GetThp();
        const float  survive = clamp(max(thp.x, max(thp.y, thp.z)), 0.05, 1.0);
        if (sampleNext1D(sgRR) > survive)
        {
            path.terminate();
            return false;
        }

        path.SetThp(thp / survive);
#endif
        return true;
    }
```

In `HandleHit`, after scatter succeeds and after `shouldTerminate` is computed, use:

```hlsl
        if (shouldTerminate)
        {
            path.setTerminateAtNextBounce();
        }
        else if (!HandleRussianRoulette(path, preScatterPath, workingContext))
        {
            return;
        }
```

- [ ] **Step 7: Commit this source checkpoint**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git commit -m "refactor(rtxpt): preserve reference quality in shared spine" --trailer "Co-Authored-By: GPT 5.5"
```

Manual review notes: this task must preserve full-sample NEE, emissive/environment MIS, firefly filtering, one-shot primary depth, reference diffuse-bounce classification, volume/nested dielectric behavior, and Russian roulette throughput correction.

---

### Task 4: Switch Reference To PathPayload In One Source Checkpoint

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerVisibilityMiss.rmiss`

- [ ] **Step 1: Switch visibility rays to `PathPayload`**

Make `MakeVisibilityPayload` available for all modes:

```hlsl
    inline PathPayload MakeVisibilityPayload(uint2 pixelPos)
    {
        PathState visibilityPath = EmptyPathInitialize(pixelPos, 0.0);
        visibilityPath.setActive();
        visibilityPath.clearHit();
#if PATH_TRACER_MODE != PATH_TRACER_MODE_BUILD_STABLE_PLANES
        visibilityPath.SetL(float4(0.0, 0.0, 0.0, 0.0));
#endif
        visibilityPath.SetThp(float3(0.0, 0.0, 0.0));
        return PathPayload::pack(visibilityPath);
    }
```

Use the packed path payload in `TraceVisibilityRay` for reference and realtime.

- [ ] **Step 2: Switch closest-hit to shared `PathPayload`**

In `PathTracerClosestHit.rchit`, include `PathTracer.hlsli` for every mode and use:

```hlsl
using ActiveRayPayload = PathPayload;

[shader("closesthit")]
void main(inout ActiveRayPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
    PathState path = PathPayload::unpack(Payload);
    PathTracer::WorkingContext workingContext = GetWorkingContext();
    PathTracer::HandleHit(path, Attributes, workingContext);
    Payload = PathPayload::pack(path);
}
```

Move `MakeFallbackTangent`, `LoadCurrentSurfaceData`, and the attributes overload of `HandleHit` out of the old non-reference-only branch.

- [ ] **Step 3: Switch primary miss to shared `PathPayload`**

Replace `PathTracerMiss.rmiss` with:

```hlsl
#include "Config.h"
#include "PathTracer.hlsli"

using ActiveRayPayload = PathPayload;

[shader("miss")]
void main(inout ActiveRayPayload Payload)
{
    PathState path = PathPayload::unpack(Payload);
    PathTracer::WorkingContext workingContext = GetWorkingContext();
    PathTracer::HandleMiss(path, WorldRayOrigin(), WorldRayDirection(), RayTCurrent(), workingContext);
    Payload = PathPayload::pack(path);
}
```

- [ ] **Step 4: Switch visibility miss to shared `PathPayload`**

Replace `PathTracerVisibilityMiss.rmiss` with:

```hlsl
#include "Config.h"
#include "PathState.hlsli"
#include "PathPayload.hlsli"

using ActiveRayPayload = PathPayload;

[shader("miss")]
void main(inout ActiveRayPayload Payload)
{
    PathState path = PathPayload::unpack(Payload);
    path.terminate();
    Payload = PathPayload::pack(path);
}
```

- [ ] **Step 5: Replace reference raygen with the unified loop**

Remove the flattened `#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE` raygen branch. Keep one shared raygen with a reference safety ceiling:

```hlsl
[shader("raygeneration")]
void main()
{
    const uint2 pixelPos = DispatchRaysIndex().xy;

    const Ray primaryRay = Bridge::computeCameraRay(pixelPos);
    PathState path = PathTracer::EmptyPathInitialize(pixelPos,
                                                     g_Const.ptConsts.camera.PixelConeSpreadAngle);
    PathTracer::SetupPathPrimaryRay(path, primaryRay);

    PathTracer::WorkingContext workingContext = GetWorkingContext();
    PathTracer::StartPixel(path, workingContext);

#if PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES
    float2 tMinMax = FirstHitFromVBuffer(path, 0u, workingContext);
#elif PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
    float2 tMinMax = float2(primaryRay.tMin, primaryRay.tMax);
#else
    float2 tMinMax = float2(0.0, kMaxRayTravel);
#endif

#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
    const uint referenceMaxLoopIterations =
        max(2u,
            max(workingContext.PtConsts.bounceCount, 1u) +
            PathTracer::GetMaxRejectedDielectricHits(workingContext.PtConsts.nestedDielectricsQuality) +
            2u);
    uint referenceLoopIterations = 0u;
#endif

    while (path.isActive())
    {
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        if (referenceLoopIterations++ >= referenceMaxLoopIterations)
        {
            path.terminate();
            break;
        }
#endif

        nextHit(path, tMinMax, workingContext);
        postProcessHit(path, workingContext);
    }

    ValidateNaNs(path, workingContext);
    PathTracer::CommitPixel(path, workingContext);
}
```

Manual review note: `ValidateNaNs` must remain reference-safe; it should read `path.GetL()` for non-build modes and `path.GetThp()` for all modes, without stable-plane resource access. `Bridge::computeCameraRay` uses `Bridge::getSampleIndex()`, so equal-sample visual comparisons outside this plan may see a different-but-valid noise field if `g_MiniConst.params.x` is non-zero.

- [ ] **Step 6: Commit this source checkpoint**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerVisibilityMiss.rmiss
git commit -m "refactor(rtxpt): route reference through PathPayload spine" --trailer "Co-Authored-By: GPT 5.5"
```

Manual review notes: after this task, reference raygen, closest-hit, miss, and visibility miss should no longer contain `RTXPTMaterialHitPayload`. This is a review note, not an automated plan step.

---

### Task 5: Remove Legacy Reference Payload Code And Update Docs

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `docs/realtime_bxdf_diff.md`
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Remove legacy flattened-reference helpers**

Delete `PathTracer::MakeEmptyPayload(uint hitFlag)` from `PathTracer.hlsli`.

Remove any now-dead flattened reference raygen state from `PathTracerSample.rgen`, including local `pathRadiance`, `prevDidEnvNEE`, `prevDidEmissiveNEE`, `primaryDepthCaptured`, and `RTXPTMaterialHitPayload payload` code.

- [ ] **Step 2: Update `docs/realtime_bxdf_diff.md`**

Add or replace the reference path section with:

```markdown
## Reference PathState Spine

The local reference variant now uses the same state-transport spine shape as upstream RTXPT:

- raygen initializes `PathState`;
- primary and visibility rays carry `PathPayload`;
- closest-hit calls `PathTracer::HandleHit`;
- miss calls `PathTracer::HandleMiss`;
- final reference output is committed by `PathTracer::CommitPixel`.

Reference mode still has Diligent-specific exits:

- raw HDR radiance is accumulated in `PathState::GetL().rgb` and written to the output color UAV;
- primary depth is captured once through reference-only `PathState` accessors and written to the existing depth UAV;
- screen motion vectors remain zero for reference;
- Russian roulette uses the reference `minBounceCount` and throughput correction semantics;
- the unified reference raygen keeps an explicit loop safety ceiling;
- stable-plane storage, stable-plane resolve, denoiser radiance, and spec-hit guide export are not used by reference.

The Diligent bridge remains intentionally different from upstream Donut. `LoadCurrentSurfaceData` is the shared local surface construction path and continues to source material, lighting, environment, and geometry state from the Diligent scene adapters.
```

- [ ] **Step 3: Update `RTXPT_FORK_MAPPING.md`**

Add this entry near the RTXPT path tracing shader mapping:

```markdown
### Reference Unified PathState Spine

Local reference mode is intentionally routed through `PathState`, `PathPayload`, `PathTracer::HandleHit`, and `PathTracer::HandleMiss`.

Remaining intentional fork differences:

- `PathTracerBridge.hlsli` uses Diligent resources and scene adapters instead of upstream Donut bridge calls.
- Reference `CommitPixel` writes raw HDR output, primary depth, and zero screen motion vectors; it does not write stable planes.
- Reference branches inside `HandleHit`, `HandleMiss`, `GenerateScatterRay`, `HandleRussianRoulette`, and `HandleNEE` preserve the local reference estimator: full-sample NEE, emissive/environment MIS, firefly filtering, camera jitter, current diffuse-bounce classification, `minBounceCount` Russian roulette, volumes, and nested dielectrics.
- Reference raygen keeps a safety iteration ceiling in addition to normal path termination.
- `RTXPTRayTracingPass.cpp` keeps the conservative 160-byte RT payload size while `PathPayload` is 80 bytes.
```

- [ ] **Step 4: Commit docs and cleanup**

```bash
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen docs/realtime_bxdf_diff.md DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): record reference PathState parity" --trailer "Co-Authored-By: GPT 5.5"
```

Manual handoff notes: build, shader compile, static search, runtime smoke, screenshot capture, image diff, and regression comparison are intentionally outside this plan.

---

## Manual Review Checklist

Use this checklist outside the plan execution flow. It is not an automated step list.

- Reference no longer depends on `RTXPTMaterialHitPayload` after the payload switch.
- Reference closest-hit calls `PathTracer::HandleHit`.
- Reference miss calls `PathTracer::HandleMiss`.
- Reference visibility miss terminates a `PathPayload`.
- Stable-plane UAVs and stable-plane writes are not reachable from reference mode.
- Reference radiance commits through `path.GetL().rgb`.
- Reference primary depth uses `InitReferencePrimaryDepth`, `CaptureReferencePrimaryDepth`, and `GetReferencePrimaryDepth`.
- Reference motion vectors remain zero.
- Reference NEE uses full-sample semantics and preserves emissive/environment MIS.
- Reference `GenerateScatterRay` preserves the current diffuse-bounce classification.
- Reference `HandleRussianRoulette` preserves `minBounceCount` and throughput correction.
- Reference raygen has a safety iteration ceiling.
- `RTXPTRayTracingPass.cpp::MaxPayloadSize` remains `sizeof(float) * 40`.

## Self-Review

- Spec coverage: the plan covers compile-universe opening, reference output context, stable-plane isolation, full-sample NEE, emissive/environment MIS, firefly filtering, diffuse-bounce classification, camera jitter via `Bridge::computeCameraRay`, one-shot primary depth, zero motion vectors, Russian roulette with `minBounceCount`, bounded reference termination, volume/nested dielectric preservation through `PathState`, visibility payload switching, conservative payload size, and documentation updates.
- User constraint: the plan contains no baseline capture task and no automated verification task.
- Type consistency: `NEEBSDFMISInfo::Unpack16bit`, `PathState::InitReferencePrimaryDepth`, `PathState::CaptureReferencePrimaryDepth`, `PathState::HasReferencePrimaryDepth`, `PathState::GetReferencePrimaryDepth`, `PathPayload::pack`, `PathPayload::unpack`, `WorkingContext.OutputColor`, `WorkingContext.Depth`, and `WorkingContext.MotionVectors` are used consistently across tasks.
