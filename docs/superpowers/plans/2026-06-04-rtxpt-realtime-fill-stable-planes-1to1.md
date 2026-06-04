# RTXPT Realtime Fill Stable Planes 1:1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore RTXPT-fork `FILL_STABLE_PLANES` direct-light NEE, BSDF scatter continuation, and stable-plane branch tracking in the Diligent RTXPT realtime path so ordinary lit geometry is no longer black in realtime mode.

**Architecture:** Keep the existing reference raygen path intact. Add the missing realtime state-machine pieces around `PathState`: a dedicated visibility ray path, a fill-mode BSDF surface contract, direct-light NEE accumulation into `PathState::L`, scatter continuation, `StablePlanesOnScatter`, and stricter fill-variant light-resource binding. The final realtime image continues to come from stable-plane storage through no-denoiser or NRD final merge, not from direct reference output.

**Tech Stack:** C++17, Diligent Engine ray tracing PSO/SBT APIs, HLSL 6.5 DXR shaders, CMake/Visual Studio build, D3D12 and Vulkan RT validation.

---

## File Structure

- Create `docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-source-map.md`: focused source map from RTXPT-fork symbols to Diligent files.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h`: add primary and visibility ray-type constants shared by raygen, visibility helpers, and SBT binding.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`: add a second miss entry for visibility rays.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`: move reusable NEE helpers out of the reference-only block, add fill-mode visibility tracing, add fill-mode NEE and scatter helpers, and replace the stopgap termination.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`: switch primary `TraceRay` calls to the shared ray-type constants.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`: fill realtime `SurfaceData` with the same material inputs needed by `SampleBSDF` and NEE.
- Modify `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`: extend `BSDFSample`, `ActiveBSDF`, and `SurfaceData` to carry fill-mode scatter state; reuse the existing `NEEBSDFMISInfo` packed MIS state.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`: create/bind the visibility miss shader, bind hit groups for two ray types, and require light resources for `FillStablePlanes`.
- Modify `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`: expose per-variant dispatch stats and binding state in the debug UI.

Work inside the submodule root for shader/C++ commits:

```powershell
cd D:\DiligentEngine-hzqst\DiligentSamples
```

Work inside the superproject root for docs and submodule-pointer commits:

```powershell
cd D:\DiligentEngine-hzqst
```

---

### Task 1: Source Contract Map

**Files:**
- Create: `docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-source-map.md`
- Read: `docs/superpowers/specs/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1-spec.md`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerStablePlanes.hlsli`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Read: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Confirm the current missing behavior by static checks**

Run from the superproject root:

```powershell
rg -n "Conservative Task 4 stopgap|currently do not run direct-light NEE|StablePlanesOnScatter\(path" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src
```

Expected before implementation:

```text
DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli:<line>:        // Conservative Task 4 stopgap...
DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp:<line>:            // Stable-plane BUILD/FILL variants currently do not run direct-light NEE...
DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli:<line>:    inline void StablePlanesOnScatter(...)
```

There should be no call to `StablePlanesOnScatter(path, bs, workingContext)` from `PathTracer.hlsli`.

- [ ] **Step 2: Create the source map document**

Create `docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-source-map.md` with this content:

```markdown
# RTXPT Realtime Fill Stable Planes Source Map

## Purpose

This map records the source-to-port contract for restoring RTXPT-fork `PATH_TRACER_MODE_FILL_STABLE_PLANES` behavior in Diligent RTXPT.

## Core Flow

| RTXPT-fork source | Diligent target | Notes |
|---|---|---|
| `Rtxpt/Shaders/PathTracerSample.hlsl::RAYGEN_ENTRY` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen::main` | Keep `FirstHitFromVBuffer -> while(path.isActive()) -> CommitPixel` for realtime variants. |
| `PathTracer.hlsli::HandleHit` | `PathTracer.hlsli::HandleHit` | Restore emission, stable-plane handling, scatter, NEE, termination ordering. |
| `PathTracer.hlsli::GenerateScatterRay` | `PathTracer.hlsli::GenerateScatterRay` | Use Diligent `SampleBSDF` and write `PathState` fields. |
| `PathTracerNEE.hlsli::HandleNEE` | `PathTracer.hlsli::HandleNEE` | Use existing Diligent `SampleDirectLightNEE` and `SampleEnvironmentNEE` helpers. |
| `PathTracerStablePlanes.hlsli::StablePlanesOnScatter` | existing `PathTracerStablePlanes.hlsli::StablePlanesOnScatter` | Call after fill-mode scatter. |
| `StablePlanes.hlsli::CommitDenoiserRadiance` | existing `StablePlanes.hlsli::CommitDenoiserRadiance` | Preserve storage contract. |

## Required Divergences

- Diligent keeps `RTXPTRayTracingPass` and Diligent PSO/SBT ownership instead of Donut/NVRHI pipeline objects.
- Diligent uses `PathTracerSample.rgen`, `.rchit`, `.rmiss`, and `.rahit` standalone shader files instead of a single Donut pipeline package.
- Visibility rays use a Diligent SBT ray type with `RTXPT_VISIBILITY_RAY_INDEX` and a `visibilityMain` miss entry to avoid mutating primary `PathState`.
- DLSS-RR, SER, OMM, and RTXDI final shading are outside this fill-path repair.

## Acceptance Links

- Spec: `docs/superpowers/specs/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1-spec.md`
- Implementation plan: `docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1.md`
```

- [ ] **Step 3: Commit the source map**

Run from the superproject root:

```powershell
git add docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-source-map.md docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1.md
git commit -m "docs(rtxpt): map realtime fill stable planes parity" -m "Co-Authored-By: GPT 5.5"
```

Expected: a root commit containing only docs changes.

---

### Task 2: Add Visibility Ray Type And Miss Entry

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Test: `cmake --build build\x64\Debug --config Debug --target DiligentSamples`

- [ ] **Step 1: Add ray-type constants**

Append this block to `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h` after the `PATH_TRACER_MODE` default:

```hlsl
#define RTXPT_PRIMARY_RAY_INDEX    0
#define RTXPT_VISIBILITY_RAY_INDEX 1
#define RTXPT_HIT_GROUP_STRIDE     2
```

- [ ] **Step 2: Add a visibility miss entry**

In `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`, keep the existing `main` entry and add this second entry after it:

```hlsl
[shader("miss")]
void visibilityMain(inout ActiveRayPayload Payload)
{
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
    Payload.hitFlag = 0u;
#else
    PathState path = PathPayload::unpack(Payload);
    path.terminate();
    Payload = PathPayload::pack(path);
#endif
}
```

This entry marks visibility as unblocked on miss without running `PathTracer::HandleMiss`.

- [ ] **Step 3: Replace primary TraceRay index literals in the reference raygen**

In `PathTracerSample.rgen`, replace the reference path `TraceRay` indices:

```hlsl
                 0,
                 1,
                 0,
```

with:

```hlsl
                 RTXPT_PRIMARY_RAY_INDEX,
                 RTXPT_HIT_GROUP_STRIDE,
                 RTXPT_PRIMARY_RAY_INDEX,
```

- [ ] **Step 4: Replace primary TraceRay index literals in realtime `nextHit`**

In `PathTracerSample.rgen::nextHit`, replace:

```hlsl
             0,
             1,
             0,
```

with:

```hlsl
             RTXPT_PRIMARY_RAY_INDEX,
             RTXPT_HIT_GROUP_STRIDE,
             RTXPT_PRIMARY_RAY_INDEX,
```

- [ ] **Step 5: Add a fill-safe visibility payload helper**

In `PathTracer.hlsli`, add this helper in `namespace PathTracer` before direct-light sampling helpers:

```hlsl
#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE
    inline PathPayload MakeVisibilityPayload(uint2 pixelPos)
    {
        PathState visibilityPath = EmptyPathInitialize(pixelPos, 0.0);
        visibilityPath.setActive();
        visibilityPath.clearHit();
        visibilityPath.SetL(float4(0.0, 0.0, 0.0, 0.0));
        visibilityPath.SetThp(float3(0.0, 0.0, 0.0));
        return PathPayload::pack(visibilityPath);
    }
#endif
```

- [ ] **Step 6: Make `TraceVisibilityRay` use the visibility ray type**

Replace the existing reference-only `TraceVisibilityRay` with this variant-gated function:

```hlsl
    bool TraceVisibilityRay(float3 origin, float3 dir, float tMax)
    {
        if (tMax <= kVisibilityRayTMin)
            return false;

        RayDesc ray;
        ray.Origin    = origin;
        ray.Direction = dir;
        ray.TMin      = kVisibilityRayTMin;
        ray.TMax      = tMax;

#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
        RTXPTMaterialHitPayload payload = MakeEmptyPayload(1u);
        TraceRay(t_SceneBVH,
                 RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER,
                 0xFF,
                 RTXPT_VISIBILITY_RAY_INDEX,
                 RTXPT_HIT_GROUP_STRIDE,
                 RTXPT_VISIBILITY_RAY_INDEX,
                 ray,
                 payload);
        return payload.hitFlag == 0u;
#else
        PathPayload payload = MakeVisibilityPayload(Bridge::getPixelPosition());
        TraceRay(t_SceneBVH,
                 RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER,
                 0xFF,
                 RTXPT_VISIBILITY_RAY_INDEX,
                 RTXPT_HIT_GROUP_STRIDE,
                 RTXPT_VISIBILITY_RAY_INDEX,
                 ray,
                 payload);
        return !PathPayload::unpack(payload).isActive();
#endif
    }
```

- [ ] **Step 7: Create and bind the visibility miss shader in C++**

In `RTXPTRayTracingPass.cpp`, add a second miss shader object next to `pMiss`:

```cpp
        RefCntAutoPtr<IShader> pVisibilityMiss;
        if (!ScreenPatternDiagnostic)
        {
            ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_MISS;
            ShaderCI.Desc.Name       = "RTXPT path trace visibility miss";
            ShaderCI.FilePath        = "PathTracer/PathTracerMiss.rmiss";
            ShaderCI.EntryPoint      = "visibilityMain";
            pDevice->CreateShader(ShaderCI, &pVisibilityMiss);
        }
```

Then update the shader verification condition:

```cpp
        VERIFY(pRayGen && (ScreenPatternDiagnostic || (pMiss && pVisibilityMiss && pClosestHit && (!UseAnyHit || pAnyHit))),
               "Failed to create RTXPT path tracing shaders");
        if (!pRayGen || (!ScreenPatternDiagnostic && (!pMiss || !pVisibilityMiss || !pClosestHit || (UseAnyHit && !pAnyHit))))
```

Then add the general shader to the PSO:

```cpp
            PSOCreateInfo.AddGeneralShader("PrimaryMiss", pMiss);
            PSOCreateInfo.AddGeneralShader("VisibilityMiss", pVisibilityMiss);
```

Then bind both miss entries and both hit-group ray-type slots:

```cpp
            State.SBT->BindMissShader("PrimaryMiss", RTXPT_PRIMARY_RAY_INDEX);
            State.SBT->BindMissShader("VisibilityMiss", RTXPT_VISIBILITY_RAY_INDEX);
            State.SBT->BindHitGroupForTLAS(m_TLAS, RTXPT_PRIMARY_RAY_INDEX, "PrimaryHit");
            State.SBT->BindHitGroupForTLAS(m_TLAS, RTXPT_VISIBILITY_RAY_INDEX, "PrimaryHit");
```

Use matching C++ constants near `GetVariantModeMacro`:

```cpp
constexpr Uint32 kRTXPTPrimaryRayIndex    = 0;
constexpr Uint32 kRTXPTVisibilityRayIndex = 1;
```

The C++ snippet should use `kRTXPTPrimaryRayIndex` and `kRTXPTVisibilityRayIndex`; the HLSL uses the macros from `Config.h`.

- [ ] **Step 8: Build to verify visibility infrastructure compiles**

Run from the superproject root:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: shaders compile and the RTXPT sample target builds. If CMake target names differ in this checkout, use the generated RTXPT sample target shown by `cmake --build build\x64\Debug --config Debug --target help | Select-String RTXPT`.

- [ ] **Step 9: Commit visibility infrastructure**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/assets/shaders/PathTracer/Config.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git commit -m "feat(rtxpt): add realtime visibility ray path" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Extend Fill-Mode Surface And BSDF Contract

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Test: `cmake --build build\x64\Debug --config Debug --target DiligentSamples`

- [ ] **Step 1: Extend `BSDFSample`**

Replace `BSDFSample` in `PathTracerTypes.hlsli` with:

```hlsl
struct BSDFSample
{
    uint   lobe;
    uint   deltaLobeIndex;
    float  pdf;
    float  lobeP;
    float3 weight;
    float3 wi;

    uint getDeltaLobeIndex() { return deltaLobeIndex; }
    bool isLobe(uint testLobe) { return (lobe & testLobe) != 0u; }
};
```

- [ ] **Step 2: Extend `StablePlaneShadingData`**

Add these fields to `StablePlaneShadingData` in `PathTracerTypes.hlsli`:

```hlsl
        float3 faceNCorrected;
        float3 vertexN;
        float  shadowNoLFadeout;
        float3 emission;
```

- [ ] **Step 3: Store `StandardBSDFData` in `ActiveBSDF`**

Replace the current `ActiveBSDF` body with:

```hlsl
    struct ActiveBSDF
    {
        StablePlaneBSDFData data;
        StandardBSDFData    standardData;

        void evalDeltaLobes(StablePlaneShadingData shadingData,
                            out DeltaLobe deltaLobes[cMaxDeltaLobes],
                            out uint deltaLobeCount,
                            out float nonDeltaPart)
        {
            for (uint i = 0; i < cMaxDeltaLobes; ++i)
                deltaLobes[i] = DeltaLobe::make();
            deltaLobeCount = 0u;
            nonDeltaPart   = 1.0;
        }

        void estimateSpecDiffBSDF(out float3 diffBSDFEstimate, out float3 specBSDFEstimate, float3 normal, float3 view)
        {
            diffBSDFEstimate = max(standardData.diffuse, 0.04.xxx);
            specBSDFEstimate = max(standardData.specular, 0.04.xxx);
        }
    };
```

- [ ] **Step 4: Add material inputs to fill-mode closest hit**

In `PathTracerClosestHit.rchit::LoadCurrentSurfaceData`, add local variables near the existing defaults:

```hlsl
        float3 baseColor = float3(1.0, 1.0, 1.0);
        float  metallic  = 0.0;
        float  transmissionFactor = 0.0;
        float  diffuseTransmissionFactor = 0.0;
        bool   thinSurface = true;
        float  shadowNoLFadeout = 0.0;
        float3 vertexNormal = worldNormal;
```

Inside the material-table block, after `metalRough` is loaded, set:

```hlsl
            metallic                  = metalRough.x;
            const float4 baseColorWithAlpha = Bridge::getBaseColor(material, texCoord);
            baseColor                 = baseColorWithAlpha.rgb;
            transmissionFactor        = Bridge::getTransmission(material, texCoord);
            diffuseTransmissionFactor = Bridge::getDiffuseTransmission(material, texCoord);
            thinSurface               = Bridge::isThinSurface(material);
            shadowNoLFadeout          = Bridge::loadShadowNoLFadeout(materialID);
            vertexNormal              = worldNormal;
```

- [ ] **Step 5: Fill the new shading and BSDF fields**

In the same function, after existing `shadingData` assignments, add:

```hlsl
        shadingData.faceNCorrected    = faceNormal;
        shadingData.vertexN           = vertexNormal;
        shadingData.shadowNoLFadeout  = shadowNoLFadeout;
        shadingData.emission          = surfaceEmission;
```

Then replace:

```hlsl
        ActiveBSDF bsdf;
        bsdf.data.roughness = roughness;
```

with:

```hlsl
        ActiveBSDF bsdf;
        bsdf.data.roughness = roughness;
        bsdf.standardData = MakeStandardBSDFData(worldNormal,
                                                 baseColor,
                                                 metallic,
                                                 roughness,
                                                 ior,
                                                 1.0,
                                                 transmissionFactor,
                                                 diffuseTransmissionFactor,
                                                 thinSurface,
                                                 frontFacing);
```

- [ ] **Step 6: Build to verify the contract compiles**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: no HLSL compile errors for `SurfaceData`, `ActiveBSDF`, or `StandardBSDFData`.

- [ ] **Step 7: Commit fill-mode material contract**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit
git commit -m "feat(rtxpt): carry bsdf data in realtime hits" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Port Direct-Light NEE To Fill Mode

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Test: `cmake --build build\x64\Debug --config Debug --target DiligentSamples`

- [ ] **Step 1: Move reusable direct-light helpers out of the reference-only block**

In `PathTracer.hlsli`, the following helpers must be available for both reference and realtime variants:

```hlsl
MakeVisibilityOrigin
ComputeShadowNoLFadeout
ComputeLightVsBSDFMISForLightSample
SampleEnvironmentNEE
ComputeBSDFMISForEmissiveTriangle
SampleDirectLightNEE
```

Keep `MakeEmptyPayload` reference-only. Keep `ComputeBSDFEnvMISWeight` reference-only unless the fill-mode miss path starts using it later.

- [ ] **Step 2: Add `HandleNEE` for `PathState`**

Add this function in `PathTracer.hlsli` inside the non-reference section, after `ShouldCollectGISecondaryRadiance`:

```hlsl
    inline NEEResult HandleNEE(const PathState preScatterPath,
                               const SurfaceData surfaceData,
                               inout SampleGenerator sgDirect,
                               inout SampleGenerator sgEnv,
                               const WorkingContext workingContext)
    {
        NEEResult result = NEEResult::empty();

        const uint maxBounces    = max(workingContext.PtConsts.bounceCount, 1u);
        const uint maxNEEBounces = min(workingContext.PtConsts.maxNEEBounceCount, maxBounces);
        const bool enableNEE     = workingContext.PtConsts.NEEEnabled != 0u;
        const bool useNEE        = enableNEE && preScatterPath.getVertexIndex() <= maxNEEBounces;
        const uint fullSamples   = min(32u, workingContext.PtConsts.NEEFullSamples);
        if (!useNEE || fullSamples == 0u)
            return result;

        result.BSDFMISInfo.LightSamplingEnabled = true;
        result.BSDFMISInfo.LightSamplingIsSSC   = false;
        result.BSDFMISInfo.CandidateSamples     = min(NEEBSDFMISInfo::SampleCountLimit(), max(1u, workingContext.PtConsts.NEECandidateSamples));
        result.BSDFMISInfo.FullSamples          = min(NEEBSDFMISInfo::SampleCountLimit(), fullSamples);

        const float3 wo = surfaceData.shadingData.V;
        bool sampledEmissive = false;
        float3 directRadiance = SampleDirectLightNEE(surfaceData.bsdf.standardData,
                                                     surfaceData.shadingData.posW,
                                                     surfaceData.shadingData.faceNCorrected,
                                                     surfaceData.shadingData.vertexN,
                                                     surfaceData.shadingData.shadowNoLFadeout,
                                                     wo,
                                                     preScatterPath.GetPixelPos(),
                                                     sgDirect,
                                                     preScatterPath.GetFireflyFilterK(),
                                                     sampledEmissive);
        directRadiance *= preScatterPath.GetThp();

        const bool enableEnvNEE = (workingContext.PtConsts.environmentNEEEnabled & 1u) != 0u;
        if (enableEnvNEE)
        {
            float3 envRadiance = SampleEnvironmentNEE(surfaceData.bsdf.standardData,
                                                      surfaceData.shadingData.posW,
                                                      surfaceData.shadingData.faceNCorrected,
                                                      surfaceData.shadingData.vertexN,
                                                      surfaceData.shadingData.shadowNoLFadeout,
                                                      wo,
                                                      sgEnv,
                                                      preScatterPath.GetFireflyFilterK());
            directRadiance += preScatterPath.GetThp() * envRadiance;
        }

        if (any(directRadiance > 0.0))
        {
            const float specAvg = preScatterPath.hasFlag(PathFlags::stablePlaneBaseScatterDiff) ? 0.0 : Average(directRadiance);
            result.AccumulateRadiance(directRadiance, specAvg);
        }

        return result;
    }
```

- [ ] **Step 3: Preserve NEE MIS info in `PathState`**

In `PathTracer.hlsli::HandleHit`, after `NEEResult neeResult = HandleNEE(...)` is added in Task 5, the plan will call:

```hlsl
        path.SetPackedMISInfo_ThpRuRuCorrection(neeResult.BSDFMISInfo.Pack16bit(), path.GetThpRuRuCorrection());
```

Confirm `PathState` already has `SetPackedMISInfo_ThpRuRuCorrection` and `NEEBSDFMISInfo::Pack16bit`; do not add duplicate storage.

- [ ] **Step 4: Build to catch helper visibility issues**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: the shader build succeeds, and there are no missing symbol errors for `SampleDirectLightNEE`, `SampleEnvironmentNEE`, `TraceVisibilityRay`, or `NEEResult`.

- [ ] **Step 5: Commit NEE helper port**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git commit -m "feat(rtxpt): expose nee helpers to realtime fill" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Add BSDF Scatter Continuation And StablePlanesOnScatter

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Test: `cmake --build build\x64\Debug --config Debug --target DiligentSamples`

- [ ] **Step 1: Add `MakeBSDFSample`**

Add this helper near `UpdatePathThroughput`:

```hlsl
    inline BSDFSample MakeBSDFSample(uint lobe, float pdf, float lobeP, float3 weight, float3 wi)
    {
        BSDFSample bs;
        bs.lobe           = lobe;
        bs.deltaLobeIndex = (lobe & kBSDFLobeSpecularTransmission) != 0u ? 1u : 0u;
        bs.pdf            = pdf;
        bs.lobeP          = lobeP;
        bs.weight         = weight;
        bs.wi             = wi;
        return bs;
    }
```

- [ ] **Step 2: Add `GenerateScatterRay`**

Add this function after `HandleNEE`:

```hlsl
    inline bool GenerateScatterRay(const SurfaceData surfaceData,
                                   inout PathState path,
                                   const WorkingContext workingContext,
                                   out BSDFSample bs)
    {
        bs = MakeBSDFSample(0u, 0.0, 0.0, 0.xxx, 0.xxx);

        const float3 wo = surfaceData.shadingData.V;
        const SampleGeneratorVertexBase sgBase = SampleGeneratorVertexBase::make(path.GetPixelPos(), path.getVertexIndex(), Bridge::getSampleIndex());
        const uint diffuseBounces = path.getCounter(PackedCounters::DiffuseBounces);
        float3 preGeneratedSamples;

#if RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF
        [branch]
        if (diffuseBounces < workingContext.PtConsts.diffuseBounceCount)
            preGeneratedSamples = SampleSequenceGenerator::Generate(3u, sgBase, kSampleEffect_ScatterBSDF).xyz;
        else
#endif
            preGeneratedSamples = UniformSampleSequenceGenerator::Generate(3u, sgBase, kSampleEffect_ScatterBSDF).xyz;

        float3 wi;
        float3 weight;
        float  pdf;
        uint   lobe;
        float  lobeP;
        if (!SampleBSDF(surfaceData.bsdf.standardData, wo, preGeneratedSamples, wi, weight, pdf, lobe, lobeP))
            return false;

        const bool isTransmission = (lobe & kBSDFLobeTransmission) != 0u;
        const float3 scatterOrigin = ComputeRayOrigin(surfaceData.shadingData.posW,
                                                      isTransmission ? -surfaceData.shadingData.faceNCorrected : surfaceData.shadingData.faceNCorrected);

        path.clearScatterEventFlags();
        path.SetOrigin(scatterOrigin);
        path.SetDir(wi);
        path.SetThp(path.GetThp() * weight);
        path.SetFireflyFilterK_BsdfScatterPdf(ComputeNewScatterFireflyFilterK(path.GetFireflyFilterK(), pdf, lobeP), pdf);
        path.setScatterTransmission(isTransmission);
        path.setScatterSpecular((lobe & (kBSDFLobeSpecularReflection | kBSDFLobeSpecularTransmission | kBSDFLobeDeltaReflection | kBSDFLobeDeltaTransmission)) != 0u);
        path.setScatterDelta((lobe & kBSDFLobeDelta) != 0u);
        path.setDeltaOnlyPath(path.isDeltaOnlyPath() && ((lobe & kBSDFLobeDelta) != 0u));

        const bool isDiffuseBounce =
            ((lobe & kBSDFLobeDiffuseReflection) != 0u) ||
            (((lobe & kBSDFLobeTransmission) == 0u) && surfaceData.bsdf.standardData.roughness > kSpecularRoughnessThreshold);
        if (isDiffuseBounce)
            path.incrementCounter(PackedCounters::DiffuseBounces);

        bs = MakeBSDFSample(lobe, pdf, lobeP, weight, wi);
        StablePlanesOnScatter(path, bs, workingContext);
        return true;
    }
```

- [ ] **Step 3: Rewrite fill-mode `HandleHit` order**

Replace the body of non-reference `PathTracer::HandleHit` after `StablePlanesHandleHit(...)` with this ordering:

```hlsl
        if (pathStopping || !path.isActive())
        {
            path.terminate();
            return;
        }

#if PATH_TRACER_MODE == PATH_TRACER_MODE_BUILD_STABLE_PLANES
        return;
#else
        const PathState preScatterPath = path;
        BSDFSample bs;
        const bool scatterValid = GenerateScatterRay(surfaceData, path, workingContext, bs);

        SampleGenerator sgDirect = SampleGenerator_makeStateless(preScatterPath.GetPixelPos(),
                                                                 preScatterPath.getVertexIndex(),
                                                                 Bridge::getSampleIndex(),
                                                                 kSampleEffect_NEELightSampler);
        SampleGenerator sgEnv = SampleGenerator_makeStateless(preScatterPath.GetPixelPos(),
                                                              preScatterPath.getVertexIndex(),
                                                              Bridge::getSampleIndex(),
                                                              kSampleEffect_NextEventEstimation);
        NEEResult neeResult = HandleNEE(preScatterPath, surfaceData, sgDirect, sgEnv, workingContext);
        path.SetPackedMISInfo_ThpRuRuCorrection(neeResult.BSDFMISInfo.Pack16bit(), path.GetThpRuRuCorrection());

        float4 neeRadianceAndSpecAvg = neeResult.GetRadianceAndSpecAvg();
        if (any(neeRadianceAndSpecAvg > 0.0))
        {
            const int bouncesFromStablePlane = preScatterPath.getCounter(PackedCounters::BouncesFromStablePlane) + 1;
            float specRadianceAvg = 0.0;
            if (!preScatterPath.hasFlag(PathFlags::stablePlaneBaseScatterDiff))
            {
                const bool pathIsDeltaOnlyPath = preScatterPath.isDeltaOnlyPath();
                const bool specialCondition = (bouncesFromStablePlane == 1) || (pathIsDeltaOnlyPath && bouncesFromStablePlane <= 3);
                specRadianceAvg = specialCondition ? neeRadianceAndSpecAvg.w : Average(neeRadianceAndSpecAvg.rgb);
            }

            AccumulatePathRadiance(workingContext,
                                   path,
                                   neeRadianceAndSpecAvg.rgb,
                                   specRadianceAvg,
                                   false,
                                   ShouldCollectGISecondaryRadiance(preScatterPath));
        }

        if (!scatterValid)
        {
            path.terminate();
            return;
        }

        const bool shouldTerminate = HasFinishedSurfaceBounces(path.getVertexIndex() + 1,
                                                               path.getCounter(PackedCounters::DiffuseBounces));
        if (shouldTerminate)
            path.setTerminateAtNextBounce();
#endif
```

Remove the old block:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES
        if (path.hasFlag(PathFlags::stablePlaneOnDominantBranch))
        {
            Bridge::ExportSpecHitTStart(path);
            Bridge::ExportSpecHitTStop(path);
        }
#endif

        // Conservative Task 4 stopgap...
        path.terminate();
```

- [ ] **Step 4: Add a static check for the restored scatter call**

Run:

```powershell
rg -n "StablePlanesOnScatter\(path, bs, workingContext\)|Conservative Task 4 stopgap" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
```

Expected after implementation:

```text
DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli:<line>:        StablePlanesOnScatter(path, bs, workingContext);
```

There must be no `Conservative Task 4 stopgap` match.

- [ ] **Step 5: Build to verify fill-mode scatter compiles**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: `BuildStablePlanes` and `FillStablePlanes` shader variants compile with the static sample-generation form from Step 2.

- [ ] **Step 6: Commit scatter continuation**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git commit -m "feat(rtxpt): continue realtime fill paths after hits" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Require Light Bridge Resources For Fill Variant

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Test: `cmake --build build\x64\Debug --config Debug --target DiligentSamples`

- [ ] **Step 1: Add variant booleans in `CreateVariant`**

Near the existing `ReferenceVariant` declaration in `RTXPTRayTracingPass.cpp`, replace:

```cpp
        const bool ReferenceVariant = Variant == RTXPTPathTraceVariant::Reference;
```

with:

```cpp
        const bool ReferenceVariant              = Variant == RTXPTPathTraceVariant::Reference;
        const bool BuildStablePlanesVariant      = Variant == RTXPTPathTraceVariant::BuildStablePlanes;
        const bool FillStablePlanesVariant       = Variant == RTXPTPathTraceVariant::FillStablePlanes;
        const bool DirectLightResourcesRequired  = ReferenceVariant || FillStablePlanesVariant;
```

- [ ] **Step 2: Make `g_MiniConst` required for fill**

Replace:

```cpp
        const bool MiniConstantsRequired = Variant == RTXPTPathTraceVariant::BuildStablePlanes;
```

with:

```cpp
        const bool MiniConstantsRequired = BuildStablePlanesVariant || FillStablePlanesVariant;
```

- [ ] **Step 3: Require light bridge resources for fill**

Replace the current light binding block:

```cpp
            const bool LightBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_Lights", pLightsView, "light buffer", ReferenceVariant);
            const bool LightsBakerBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightingControl", pLightingControlView, "LightsBaker control buffer", ReferenceVariant) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightProxyCounters", pLightProxyCountersView, "LightsBaker proxy counters", ReferenceVariant) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightSamplingProxies", pLightSamplingProxiesView, "LightsBaker sampling proxies", ReferenceVariant) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LocalSamplingBuffer", pLocalSamplingView, "LightsBaker local sampling buffer", ReferenceVariant) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "u_FeedbackTotalWeight", pFeedbackTotalWeightUAV, "LightsBaker feedback total weight", ReferenceVariant) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "u_FeedbackCandidates", pFeedbackCandidatesUAV, "LightsBaker feedback candidates", ReferenceVariant);
```

with:

```cpp
            const bool LightBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_Lights", pLightsView, "light buffer", DirectLightResourcesRequired);
            const bool LightsBakerBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightingControl", pLightingControlView, "LightsBaker control buffer", DirectLightResourcesRequired) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightProxyCounters", pLightProxyCountersView, "LightsBaker proxy counters", DirectLightResourcesRequired) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LightSamplingProxies", pLightSamplingProxiesView, "LightsBaker sampling proxies", DirectLightResourcesRequired) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_LocalSamplingBuffer", pLocalSamplingView, "LightsBaker local sampling buffer", DirectLightResourcesRequired) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "u_FeedbackTotalWeight", pFeedbackTotalWeightUAV, "LightsBaker feedback total weight", DirectLightResourcesRequired) &&
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "u_FeedbackCandidates", pFeedbackCandidatesUAV, "LightsBaker feedback candidates", DirectLightResourcesRequired);
```

- [ ] **Step 4: Require emissive triangles for fill when reflected**

Replace:

```cpp
            const bool EmissiveLightBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_EmissiveTriangles", pEmissiveView, "emissive triangle buffer", ReferenceVariant);
```

with:

```cpp
            const bool EmissiveLightBridgeBound =
                SetStaticForStages(SHADER_TYPE_RAY_GEN, "t_EmissiveTriangles", pEmissiveView, "emissive triangle buffer", DirectLightResourcesRequired);
```

- [ ] **Step 5: Build to verify binding strictness**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: PSO creation succeeds when the app has light and emissive dummy buffers available. If initialization fails, inspect the log resource name and bind the existing dummy resource produced by `RTXPTLightsBaker` or `RTXPTEmissiveTrianglePass`, not a null object.

- [ ] **Step 6: Commit binding strictness**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git commit -m "feat(rtxpt): require light bridge for realtime fill" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Add Realtime Fill Diagnostics

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Test: launch RTXPT and inspect the Status / Debug panel

- [ ] **Step 1: Add per-variant dispatch stats to the UI**

In `RTXPTSample.cpp` under the Status / Debug section after:

```cpp
        ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
```

add:

```cpp
        const RTXPTRayTracingVariantStats& ReferenceStats = m_RayTracingPass.GetVariantStats(RTXPTPathTraceVariant::Reference);
        const RTXPTRayTracingVariantStats& BuildStats     = m_RayTracingPass.GetVariantStats(RTXPTPathTraceVariant::BuildStablePlanes);
        const RTXPTRayTracingVariantStats& FillStats      = m_RayTracingPass.GetVariantStats(RTXPTPathTraceVariant::FillStablePlanes);
        ImGui::Text("Reference variant: %s, dispatches=%u", ReferenceStats.Ready ? "ready" : "missing", ReferenceStats.TraceCount);
        ImGui::Text("BuildStablePlanes variant: %s, dispatches=%u", BuildStats.Ready ? "ready" : "missing", BuildStats.TraceCount);
        ImGui::Text("FillStablePlanes variant: %s, dispatches=%u", FillStats.Ready ? "ready" : "missing", FillStats.TraceCount);
```

- [ ] **Step 2: Add explicit fill-light binding status**

After the existing light bridge status lines, add:

```cpp
        if (m_RealtimeUI.RealtimeMode)
        {
            ImGui::Text("Realtime fill light resources: %s",
                        RTPassStats.LightBridgeBound && RTPassStats.LightsBakerBridgeBound && RTPassStats.EmissiveLightBridgeBound ?
                            "bound" :
                            "missing");
        }
```

- [ ] **Step 3: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: C++ compiles without missing `RTXPTPathTraceVariant` or `RTXPTRayTracingVariantStats` symbols.

- [ ] **Step 4: Commit diagnostics**

Run from `D:\DiligentEngine-hzqst\DiligentSamples`:

```powershell
git add Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "chore(rtxpt): expose realtime fill diagnostics" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 8: Final Verification And Superproject Update

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: superproject git index for `DiligentSamples` submodule pointer if code commits were made

- [ ] **Step 1: Run static regression checks**

Run from the superproject root:

```powershell
rg -n "Conservative Task 4 stopgap|currently do not run direct-light NEE" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src
rg -n "StablePlanesOnScatter\(path, bs, workingContext\)" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
rg -n "VisibilityMiss|visibilityMain|RTXPT_VISIBILITY_RAY_INDEX" DiligentSamples/Samples/RTXPT
```

Expected:

```text
first command: no matches
second command: one match in PathTracer.hlsli
third command: matches in Config.h, PathTracerMiss.rmiss, PathTracer.hlsli, and RTXPTRayTracingPass.cpp
```

- [ ] **Step 2: Run build verification**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: build completes successfully. If the generated target is not `DiligentSamples`, run:

```powershell
cmake --build build\x64\Debug --config Debug --target help | Select-String RTXPT
```

Then build the displayed RTXPT sample target.

- [ ] **Step 3: Run manual D3D12 smoke**

Launch the RTXPT sample using the local run command for this checkout. If the repo has a generated Visual Studio executable, use the Debug RTXPT sample executable under `build\x64\Debug` or `install\x64\Debug`.

Manual checks:

```text
1. Select realtime mode.
2. Disable standalone denoiser so no-denoiser final merge is visible.
3. Load or keep a scene with analytic lights and non-emissive geometry.
4. Confirm ordinary scene geometry is lit instead of black.
5. Confirm sky and emissive objects remain visible without obvious double brightness.
6. Open Status / Debug and confirm BuildStablePlanes and FillStablePlanes dispatch counts increase.
7. Confirm realtime fill light resources read "bound".
8. Switch back to reference mode and confirm reference output still renders.
```

- [ ] **Step 4: Run Vulkan smoke when available**

If the local build exposes a Vulkan RTXPT run mode and Vulkan RT support is available, repeat Step 3 in Vulkan. Expected: shader creation succeeds and realtime mode renders lit non-emissive geometry.

- [ ] **Step 5: Commit superproject pointer**

Run from the superproject root after all submodule commits:

```powershell
git status --short
git add DiligentSamples docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1.md docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-source-map.md
git commit -m "feat(rtxpt): plan realtime fill stable planes parity" -m "Co-Authored-By: GPT 5.5"
```

Expected: root commit records the updated `DiligentSamples` submodule pointer and plan docs.

---

## Self-Review Checklist

- Spec coverage:
  - F1 source map: Task 1.
  - F2 scatter state: Task 3 and Task 5.
  - F3 direct-light NEE: Task 4 and Task 5.
  - F4 BSDF scatter continuation: Task 5.
  - F5 realtime light bridge binding: Task 6.
  - F6 validation and regression guards: Task 7 and Task 8.
- Red-flag scan: this plan contains no banned marker strings or vague missing-code instructions.
- Type consistency:
  - New HLSL constants are `RTXPT_PRIMARY_RAY_INDEX`, `RTXPT_VISIBILITY_RAY_INDEX`, and `RTXPT_HIT_GROUP_STRIDE`.
  - New C++ constants are `kRTXPTPrimaryRayIndex` and `kRTXPTVisibilityRayIndex`.
  - New miss entry is `visibilityMain`; SBT group name is `VisibilityMiss`.
  - Fill scatter uses `BSDFSample bs` and calls `StablePlanesOnScatter(path, bs, workingContext)`.
