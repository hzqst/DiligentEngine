# RTXPT Phase R7 Shadow and AA Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the DiligentEngine RTXPT reference path tracer to RTXPT-fork reference-mode parity for G11: robust face-normal ray origins, grazing-angle shadow fadeout, and thin-lens depth of field with RTXPT-style camera sampling.

**Architecture:** Keep the Diligent raygen-driven, `MaxRecursionDepth = 1` reference path tracer and the R0.5 `PathTracer/` shader layout. Port RTXPT-fork's ray-origin and camera-ray helpers into `PathTracerHelpers.hlsli`, pass the extra surface/material data that grazing fadeout needs through the closest-hit payload, and add a top-level `PathTracerCameraData` constant block to `SampleConstants` so the raygen can use the same `ComputeRayThinlens` flow while preserving the existing Diligent frame-constant layering.

**Tech Stack:** HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, C++17 sample code under `DiligentSamples/Samples/RTXPT/src`, Diligent ray-tracing PSO payload sizing, Dear ImGui, and RTXPT sidecar material JSON. `DiligentSamples` is a git submodule; implementation commits in this plan are made inside `DiligentSamples/`.

---

## Context You Need Before Starting

Phase R6 has landed in the working tree this plan was written against. Use the current RTXPT-fork-aligned paths:

| Spec name | Current path |
|---|---|
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTReference.rchit` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` |
| `RTXPTShaderShared.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` |
| C++ frame constants mirror | `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` |
| C++ material mirror | `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` |
| Mapping doc | `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` |

Current baseline:

- `PathTracerHelpers.hlsli` has MIS and firefly helpers, but no RTXPT-fork `ComputeRayOrigin`, no grazing fadeout helper, and no camera ray helper.
- `PathTracer::MakeVisibilityOrigin` offsets by a scalar bias and chooses a side from the face normal. RTXPT-fork chooses the face-normal side from the shading normal and then calls `ComputeRayOrigin`.
- `PathPayload` is 144 bytes. It carries `worldNormal` and `faceNormal`, but not the pre-normal-map vertex normal or `shadowNoLFadeout`.
- `MaterialPTData` is 144 bytes and has unused tail padding at offsets 132, 136, and 140. Use that padding for `shadowNoLFadeout` rather than growing the material record.
- `PathTracerSample.rgen` builds primary rays through `viewProjInv`, so it cannot apply aperture/focal-distance depth of field.
- `RTXPTReferenceUIState` has `AccumulationAA`, but no `CameraAperture` or `CameraFocalDistance` controls.

RTXPT-fork anchors to read before coding:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerHelpers.hlsli:29-60` - robust `ComputeRayOrigin` and `ComputeLowGrazingAngleFalloff`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerHelpers.hlsli:126-150` - `ComputeRayThinlens`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli:166-213` - visibility ray origin and fadeout call site.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Scene/ShadingData.hlsli:42-57` - `vertexN` and `shadowNoLFadeout` fields used by fadeout.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Materials/MaterialPT.h:54` - material `ShadowNoLFadeout` field.
- `D:/RTXPT-fork/Rtxpt/Materials/MaterialsBaker.cpp:287,588` - material UI clamp range and upload clamp.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h:19-44,110-133` - `PathTracerCameraData` and `BridgeCamera`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerBridgeDonut.hlsli:545-553` - camera sample generation and `ComputeRayThinlens` usage.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:176-177` and `SampleUI.cpp:685-689` - DoF defaults and UI labels: `Aperture`, `Focal Distance`.

Do not copy NVIDIA file headers, large comments, or wholesale source blocks. Port the behavior, names, constants, and control flow into Diligent-owned code.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp` - add RTXPT sidecar `ShadowNoLFadeout`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp` - parse `ShadowNoLFadeout` from `.material.json`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` - reuse material padding for `shadowNoLFadeout`, add offset guard.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` - upload clamped `ShadowNoLFadeout`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` - add `PathTracerCameraData`, grow `SampleConstants`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - add DoF UI state.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - fill camera constants, expose DoF UI, reset accumulation on DoF changes.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` - grow `MaxPayloadSize` for the R7 payload extension.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - mirror material, payload, and camera layout.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli` - expose `loadShadowNoLFadeout`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` - output vertex normal and material fadeout in the payload.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli` - add robust ray origin, grazing fadeout, disk sampling, and thin-lens ray helpers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` - use robust visibility origins and fadeout in direct/environment NEE.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - use `ComputeRayThinlens`, robust scatter origins, and new NEE parameters.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record R7 mappings and intentional Diligent-native divergences.

## Cross-Cutting Contracts

- **Material layout:** `MaterialPTData` stays 144 bytes. Replace the float padding at offset 136 with `shadowNoLFadeout`; keep offset 128 `nestedPriority` unchanged.
- **Payload layout:** `PathPayload` grows from 144 bytes to 160 bytes by adding `float3 vertexNormal` and `float shadowNoLFadeout`. Update `RTXPTRayTracingPass::Initialize` to `sizeof(float) * 40`.
- **Frame constants layout:** `PathTracerConstants` stays 80 bytes. `SampleConstants` grows from 368 bytes to 480 bytes by adding a 112-byte `PathTracerCameraData` top-level field. This mirrors RTXPT-fork's camera struct, while the field placement remains Diligent-native.
- **Shadow visibility rays:** Continue to reuse hit group 0 + miss 0 with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`; no new ray type and `MaxRecursionDepth` remains 1.
- **DoF defaults:** `CameraAperture` defaults to `0.0f`, so the default image remains pinhole. `CameraFocalDistance` defaults to `10000.0f`, matching RTXPT-fork UI defaults.
- **Sidecar compatibility:** Existing material JSON files without `ShadowNoLFadeout` keep the default `0.0f`, so fadeout is opt-in per material just like RTXPT-fork.
- **Scope boundary:** R7 does not add stable planes, TAA, NRD, DLSS, realtime-mode camera history, material editor UI, ray cones, or texture LOD changes.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: RTXPT-fork reference anchors

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Do not overwrite dirty files without reading them first.

- [ ] **Step 2: Confirm R6 baseline is present**

Run:

```powershell
rg -n "PathPayload.*144 bytes|MaxPayloadSize.*36|MaterialPTData.*144|nestedDielectricsQuality|InteriorList|transmissionFactor|diffuseTransmissionFactor|PathTracerNestedDielectrics" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `PathTracerShared.h`, `RTXPTRayTracingPass.cpp`, `RTXPTFrameConstants.hpp`, `RTXPTMaterials.hpp`, and R6 shader files.

- [ ] **Step 3: Confirm the RTXPT-fork anchors are available**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\PathTracerHelpers.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\PathTracerNEE.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\PathTracerShared.h
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracerBridgeDonut.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Materials\MaterialsBaker.cpp
```

Expected: all commands print `True`.

- [ ] **Step 4: Commit nothing**

Expected: no commit in Task 0. This task only establishes the starting point.

---

### Task 1: Add Shadow Fadeout Material and Payload Contracts

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Add the sidecar material field**

In `RTXPTSceneGraph.hpp`, add `ShadowNoLFadeout` after `NestedPriority`:

```cpp
    int    NestedPriority                          = 14;
    float  ShadowNoLFadeout                        = 0.0f;
    bool   SkipRender                              = false;
```

- [ ] **Step 2: Parse the sidecar material field**

In `RTXPTSceneGraph.cpp::ParseRTXPTMaterialExtension`, after `NestedPriority`, add:

```cpp
    Ext.ShadowNoLFadeout = ReadRTXPTOptionalFloat(Json, "ShadowNoLFadeout", Ext.ShadowNoLFadeout);
```

Do not clamp here; upload code owns the RTXPT-fork clamp range.

- [ ] **Step 3: Reuse material padding for `shadowNoLFadeout`**

In `RTXPTMaterials.hpp`, replace the `MaterialPTData` tail after `nestedPriority` with:

```cpp
    // RTXPT-fork authored priority: 0 is the special highest-priority value; 14 is the default/max authored value.
    Uint32 nestedPriority   = 14;   // offset 128
    Uint32 _paddingR7_0     = 0;    // offset 132
    float  shadowNoLFadeout = 0.0f; // offset 136
    float  _paddingR7_1     = 0.0f; // offset 140
};
static_assert(sizeof(MaterialPTData) == 144, "MaterialPTData layout must match PathTracer/PathTracerShared.h");
static_assert(offsetof(MaterialPTData, shadowNoLFadeout) == 136,
              "MaterialPTData shadowNoLFadeout offset must match PathTracer/PathTracerShared.h");
```

Keep all existing offset `static_assert`s, including `nestedPriority == 128`.

- [ ] **Step 4: Upload the clamped fadeout value**

In `RTXPTMaterials.cpp::RTXPTMaterials::Upload(IRenderDevice*, const RTXPTSceneGraphData&)`, inside the sidecar extension block after `Data.nestedPriority`, add:

```cpp
                Data.shadowNoLFadeout = std::clamp(Ext.ShadowNoLFadeout, 0.0f, 0.25f);
```

`0.25f` matches the RTXPT-fork material upload clamp.

- [ ] **Step 5: Mirror material and payload layout in HLSL**

In `PathTracerShared.h`, add the R7 payload fields at the end of `PathPayload`:

```hlsl
    uint  nestedPriority;
    uint  frontFacing;
    uint  thinSurface;
    float alpha;

    float3 vertexNormal;       // Interpolated vertex normal, corrected for face side, before normal mapping.
    float  shadowNoLFadeout;   // RTXPT-fork MaterialPT::ShadowNoLFadeout.
};
```

Update the payload comment:

```hlsl
// Reference path tracer payload. Size is 160 bytes (40 floats); keep RTXPTRayTracingPass::Initialize
// MaxPayloadSize in sync when this changes.
```

In the HLSL `MaterialPTData` tail, replace the padding with:

```hlsl
    uint  nestedPriority; // offset 128
    uint  _paddingR7_0;
    float shadowNoLFadeout; // offset 136
    float _paddingR7_1;
};
```

- [ ] **Step 6: Add material bridge accessor**

In `MaterialBridge.hlsli`, after `loadIoR`, add:

```hlsl
    float loadShadowNoLFadeout(uint materialID)
    {
        const uint count = getMaterialCount();
        if (materialID >= count)
            return 0.0;

        MaterialPTData material = getMaterial(materialID);
        return clamp(material.shadowNoLFadeout, 0.0, 0.25);
    }
```

- [ ] **Step 7: Initialize the new payload fields**

In `PathTracer.hlsli::PathTracer::MakeEmptyPayload`, after `payload.alpha = 1.0;`, add:

```hlsl
        payload.vertexNormal       = payload.worldNormal;
        payload.shadowNoLFadeout   = 0.0;
```

- [ ] **Step 8: Write vertex normal and fadeout in closest-hit**

In `PathTracerClosestHit.rchit`, introduce a `VertexNormal` local next to `WorldNormal`:

```hlsl
    float3 WorldNormal = -WorldRayDirection();
    float3 VertexNormal = WorldNormal;
    float3 FaceNormal  = -WorldRayDirection();
```

After interpolating `WorldNormal`, before tangent-space normal mapping, capture the unperturbed vertex normal:

```hlsl
        WorldNormal = Bridge::interpolateNormal(V0, V1, V2, Attributes.barycentrics);
        if (dot(WorldNormal, WorldNormal) < 1e-6)
            WorldNormal = FaceNormal;
        if (dot(WorldNormal, FaceNormal) < 0.0)
            WorldNormal = -WorldNormal;
        VertexNormal = WorldNormal;
```

After `Payload.alpha = BaseColorWithAlpha.a;`, add:

```hlsl
        Payload.shadowNoLFadeout = Bridge::loadShadowNoLFadeout(MaterialID);
```

At the final payload write block, add:

```hlsl
    Payload.vertexNormal = normalize(VertexNormal);
```

- [ ] **Step 9: Grow ray-tracing payload size**

In `RTXPTRayTracingPass.cpp`, replace the payload comment and value:

```cpp
    // PathPayload = 10 * float4 = 160 bytes (R7 adds vertex normal + shadow fadeout).
    PSOCreateInfo.MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 40);
```

- [ ] **Step 10: Verify material and payload contract symbols**

Run:

```powershell
rg -n "ShadowNoLFadeout|shadowNoLFadeout|vertexNormal|PathPayload.*160 bytes|MaxPayloadSize.*40|offsetof\\(MaterialPTData, shadowNoLFadeout\\) == 136" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "PathPayload.*144 bytes|MaxPayloadSize.*36|_paddingR6_1" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: the first command finds the R7 fields and size updates. The second command finds no stale R6 payload-size or material-padding references.

- [ ] **Step 11: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add shadow fadeout surface payload" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Port Robust Ray Origins and Grazing Fadeout

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Add robust ray-origin and fadeout helpers**

In `PathTracerHelpers.hlsli`, add these helpers before `PowerHeuristic`:

```hlsl
float ComputeRayOriginComponent(float worldPosition, float faceNormal)
{
    const float originScale = 1.0 / 16.0;
    const float floatScale  = 3.0 / 65536.0;
    const float intScale    = 3.0 * 256.0;

    const int   intOffset = int(faceNormal * intScale);
    const int   intPos    = asint(worldPosition) + (worldPosition < 0.0 ? -intOffset : intOffset);
    const float fpOffset  = worldPosition + faceNormal * floatScale;
    return abs(worldPosition) < originScale ? fpOffset : asfloat(intPos);
}

float3 ComputeRayOrigin(float3 worldPosition, float3 faceNormal)
{
    return float3(ComputeRayOriginComponent(worldPosition.x, faceNormal.x),
                  ComputeRayOriginComponent(worldPosition.y, faceNormal.y),
                  ComputeRayOriginComponent(worldPosition.z, faceNormal.z));
}

float ComputeLowGrazingAngleFalloff(float3 lightDirection, float3 interpolatedGeometryNormal, float falloffFrom, float falloffRange)
{
    return saturate((dot(lightDirection, interpolatedGeometryNormal) - falloffFrom) / max(falloffRange, 1e-6));
}
```

This is the RTXPT-fork behavior, written without relying on a `select()` helper that the Diligent shader tree does not currently define.

- [ ] **Step 2: Make visibility rays start from the robust origin**

In `PathTracer.hlsli`, set:

```hlsl
static const float kVisibilityRayTMin = 0.0;
```

Replace `MakeVisibilityOrigin` with:

```hlsl
    float3 MakeVisibilityOrigin(float3 hitPos, float3 faceNormal, float3 shadingNormal, float3 dir)
    {
        const float side = dot(shadingNormal, dir) >= 0.0 ? 1.0 : -1.0;
        return ComputeRayOrigin(hitPos, faceNormal * side);
    }

    float ComputeShadowNoLFadeout(float3 lightDir, float3 vertexNormal, float shadowNoLFadeout)
    {
        return shadowNoLFadeout > 0.0 ?
            ComputeLowGrazingAngleFalloff(lightDir, vertexNormal, shadowNoLFadeout, 2.0 * shadowNoLFadeout) :
            1.0;
    }
```

This mirrors RTXPT-fork `ComputeVisibilityRay`: the side is chosen by the shading normal, and the offset uses the face normal.

- [ ] **Step 3: Apply fadeout to environment NEE**

Change the `SampleEnvironmentNEE` signature in `PathTracer.hlsli` to:

```hlsl
    float3 SampleEnvironmentNEE(StandardBSDFData bsdfData, float3 hitPos, float3 faceNormal, float3 vertexNormal,
                                float shadowNoLFadeout, float3 wo, inout SampleGenerator sg, float fireflyFilterK)
```

Replace its visibility origin and contribution block with:

```hlsl
        const float3 visibilityOrigin = MakeVisibilityOrigin(hitPos, faceNormal, bsdfData.N, envSample.Dir);
        if (!TraceVisibilityRay(visibilityOrigin, envSample.Dir, kVisibilityRayTMax))
            return float3(0.0, 0.0, 0.0);

        const float misWeight = PowerHeuristic(1.0, envSample.Pdf, 1.0, bsdfPdf);
        const float fadeOut   = ComputeShadowNoLFadeout(envSample.Dir, vertexNormal, shadowNoLFadeout);
        float3      contribution = f * envSample.Le * (fadeOut * misWeight / envSample.Pdf);
```

- [ ] **Step 4: Apply fadeout to direct-light NEE**

Change the `SampleDirectLightNEE` signature in `PathTracer.hlsli` to:

```hlsl
    float3 SampleDirectLightNEE(StandardBSDFData bsdfData, float3 hitPos, float3 faceNormal, float3 vertexNormal,
                                float shadowNoLFadeout, float3 wo, uint2 pixelPos, inout SampleGenerator sg,
                                float fireflyFilterK, out bool sampledEmissive)
```

Replace the visibility origin and contribution block with:

```hlsl
            const float visibilityDistance =
                picked.kind == kLightProxyKindEmissiveBucket ? picked.distance * 0.9985 : picked.distance;
            const float3 visibilityOrigin = MakeVisibilityOrigin(hitPos, faceNormal, bsdfData.N, picked.dir);
            if (!TraceVisibilityRay(visibilityOrigin, picked.dir, visibilityDistance))
                continue;

            const float wrsScale   = 1.0 / (candidateProbability * float(candidateSamples));
            const float misWeight  = ComputeLightVsBSDFMISForLightSample(picked, fullSamples);
            const float fadeOut    = ComputeShadowNoLFadeout(picked.dir, vertexNormal, shadowNoLFadeout);
            float3      contribution = picked.bsdfF * picked.radianceOverPdf * (fadeOut * wrsScale * misWeight / float(fullSamples));
```

- [ ] **Step 5: Pass the new NEE arguments from raygen**

In `PathTracerSample.rgen`, replace the direct-light NEE call with:

```hlsl
            pathRadiance += throughput * PathTracer::SampleDirectLightNEE(bsdfData,
                                                                          payload.worldPos,
                                                                          payload.faceNormal,
                                                                          payload.vertexNormal,
                                                                          payload.shadowNoLFadeout,
                                                                          wo,
                                                                          pixel,
                                                                          sgNEELight,
                                                                          fireflyFilterK,
                                                                          sampledEmissiveNEE);
```

Replace the environment NEE call with:

```hlsl
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData,
                                                                              payload.worldPos,
                                                                              payload.faceNormal,
                                                                              payload.vertexNormal,
                                                                              payload.shadowNoLFadeout,
                                                                              wo,
                                                                              sgEnvNEE,
                                                                              fireflyFilterK);
```

- [ ] **Step 6: Use robust origins for non-NEE path continuation**

In `PathTracerSample.rgen`, replace the nested-dielectric false-hit continuation origin:

```hlsl
            rayOrigin = ComputeRayOrigin(payload.worldPos, -payload.faceNormal);
            continue;
```

Replace the scatter origin:

```hlsl
        const bool   isTransmission = (lobe & kBSDFLobeTransmission) != 0u;
        const float3 scatterOrigin  = ComputeRayOrigin(payload.worldPos, isTransmission ? -payload.faceNormal : payload.faceNormal);
```

- [ ] **Step 7: Verify robust-origin and fadeout symbols**

Run:

```powershell
rg -n "ComputeRayOrigin|ComputeRayOriginComponent|ComputeLowGrazingAngleFalloff|ComputeShadowNoLFadeout|MakeVisibilityOrigin\\(|kVisibilityRayTMin = 0\\.0|shadowNoLFadeout|vertexNormal" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "MakeVisibilityOrigin\\([^,]+,[^,]+,[^,]+\\)|\\+ \\(isTransmission \\? -payload\\.faceNormal : payload\\.faceNormal\\) \\* bias|rayOrigin = payload\\.worldPos - payload\\.faceNormal \\* bias" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: the first command finds the R7 helper and call sites. The second command finds no stale three-argument visibility origin or scalar-bias scatter origins.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "feat(rtxpt): align shadow ray origins with RTXPT" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Add RTXPT-Fork Thin-Lens Camera Data

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Add C++ camera constants mirror**

In `RTXPTFrameConstants.hpp`, before `PathTracerConstants`, add:

```cpp
struct PathTracerCameraData
{
    float3 PosW                 = float3{0, 0, 0};
    float  NearZ                = 0.0f;
    float3 DirectionW           = float3{0, 0, 1};
    float  PixelConeSpreadAngle = 0.0f;
    float3 CameraU              = float3{1, 0, 0};
    float  FarZ                 = 0.0f;
    float3 CameraV              = float3{0, 1, 0};
    float  FocalDistance        = 10000.0f;
    float3 CameraW              = float3{0, 0, 10000.0f};
    float  AspectRatio          = 1.0f;
    Uint32 ViewportWidth        = 1;
    Uint32 ViewportHeight       = 1;
    float  ApertureRadius       = 0.0f;
    float  _padding0            = 0.0f;
    float2 Jitter               = float2{0, 0};
    float  _padding1            = 0.0f;
    float  _padding2            = 0.0f;
};
static_assert(sizeof(PathTracerCameraData) == 112, "PathTracerCameraData layout must match PathTracer/PathTracerShared.h");
```

Add the camera field to `SampleConstants` after `viewportSizeAndFrameIndex`:

```cpp
    float4               viewportSizeAndFrameIndex = float4{0, 0, 0, 0};
    PathTracerCameraData camera                    = {};
    PathTracerConstants  ptConsts                  = {};
```

Update:

```cpp
static_assert(sizeof(SampleConstants) == 480, "SampleConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Mirror camera constants in HLSL**

In `PathTracerShared.h`, before `PathTracerConstants`, add:

```hlsl
struct PathTracerCameraData
{
    float3 PosW;
    float  NearZ;
    float3 DirectionW;
    float  PixelConeSpreadAngle;
    float3 CameraU;
    float  FarZ;
    float3 CameraV;
    float  FocalDistance;
    float3 CameraW;
    float  AspectRatio;
    uint2  ViewportSize;
    float  ApertureRadius;
    float  _padding0;
    float2 Jitter;
    float  _padding1;
    float  _padding2;
};
```

Add the field to HLSL `SampleConstants` in the same position and update the comment to 480 bytes:

```hlsl
// Mirrors Diligent::SampleConstants in RTXPTFrameConstants.hpp (must keep order and layout in sync; total size 480 bytes).
struct SampleConstants
{
    float4x4             viewProj;
    float4x4             viewProjInv;
    float4               cameraPositionAndTime;
    float4               viewportSizeAndFrameIndex;
    PathTracerCameraData camera;
    PathTracerConstants  ptConsts;
    RTXPTEnvMapConstants envMap;
};
```

- [ ] **Step 3: Add RTXPT-style DoF UI state**

In `RTXPTSample.hpp::RTXPTReferenceUIState`, add after `EnvironmentMapEnabled`:

```cpp
    float CameraAperture                       = 0.0f;     // Phase R7 (G11): thin-lens aperture radius.
    float CameraFocalDistance                  = 10000.0f; // Phase R7 (G11): thin-lens focal distance.
```

Do not add a separate DoF enable flag. RTXPT-fork uses `CameraAperture == 0.0f` as the pinhole/default path.

- [ ] **Step 4: Add a C++ camera-data builder**

In `RTXPTSample.cpp`, inside the anonymous namespace after `PackEnvironmentNEEAndEmissiveTriangleCount`, add:

```cpp
PathTracerCameraData MakePathTracerCameraData(const FirstPersonCamera& Camera,
                                              Uint32                   Width,
                                              Uint32                   Height,
                                              float                    FocalDistance,
                                              float                    ApertureRadius)
{
    const auto& Proj = Camera.GetProjAttribs();
    const float SafeHeight = static_cast<float>(std::max(Height, Uint32{1}));
    const float AspectRatio = Height > 0 ? static_cast<float>(Width) / static_cast<float>(Height) : 1.0f;
    const float SafeFocalDistance = std::max(FocalDistance, 0.001f);
    const float TanHalfFov = std::tan(Proj.FOV * 0.5f);

    const float3 CameraDir = normalize(Camera.GetWorldAhead());
    const float3 CameraUp  = normalize(Camera.GetWorldUp());
    const float3 CameraW   = CameraDir * SafeFocalDistance;
    const float3 CameraUUnit = normalize(cross(CameraW, CameraUp));
    const float3 CameraVUnit = normalize(cross(CameraUUnit, CameraW));

    PathTracerCameraData Data;
    Data.PosW                 = Camera.GetPos();
    Data.NearZ                = Proj.NearClipPlane;
    Data.DirectionW           = CameraDir;
    Data.PixelConeSpreadAngle = std::atan(2.0f * TanHalfFov / SafeHeight);
    Data.CameraU              = CameraUUnit * (SafeFocalDistance * TanHalfFov * AspectRatio);
    Data.FarZ                 = Proj.FarClipPlane;
    Data.CameraV              = CameraVUnit * (SafeFocalDistance * TanHalfFov);
    Data.FocalDistance        = SafeFocalDistance;
    Data.CameraW              = CameraW;
    Data.AspectRatio          = AspectRatio;
    Data.ViewportWidth        = std::max(Width, Uint32{1});
    Data.ViewportHeight       = std::max(Height, Uint32{1});
    Data.ApertureRadius       = std::max(ApertureRadius, 0.0f);
    return Data;
}
```

`ApplySceneCamera` already sets reference axes so `GetWorldAhead()` matches the Donut/RTXPT forward vector; do not negate it.

- [ ] **Step 5: Upload camera constants each frame**

In `RTXPTSample.cpp::UpdateFrameConstants`, after `viewportSizeAndFrameIndex`, add:

```cpp
    m_LastFrameConstants.camera = MakePathTracerCameraData(m_Camera,
                                                           SCDesc.Width,
                                                           SCDesc.Height,
                                                           m_ReferenceUI.CameraFocalDistance,
                                                           m_ReferenceUI.CameraAperture);
```

- [ ] **Step 6: Add disk sampling and thin-lens helpers in HLSL**

In `PathTracerHelpers.hlsli`, after the grazing fadeout helper, add:

```hlsl
struct CameraRay
{
    float3 origin;
    float3 dir;
    float  tMin;
    float  tMax;
};

float2 SampleConcentricDisk(float2 sample)
{
    const float2 p = 2.0 * sample - 1.0;
    if (dot(p, p) == 0.0)
        return float2(0.0, 0.0);

    float r;
    float theta;
    if (abs(p.x) > abs(p.y))
    {
        r     = p.x;
        theta = 0.7853981633974483 * (p.y / p.x);
    }
    else
    {
        r     = p.y;
        theta = 1.5707963267948966 - 0.7853981633974483 * (p.x / p.y);
    }

    return r * float2(cos(theta), sin(theta));
}

float3 ComputeNonNormalizedRayDirPinhole(PathTracerCameraData data, uint2 pixel, float2 jitter)
{
    const float2 p   = (float2(pixel) + float2(0.5, 0.5) + jitter) / float2(data.ViewportSize);
    const float2 ndc = float2(2.0, -2.0) * p + float2(-1.0, 1.0);
    return ndc.x * data.CameraU + ndc.y * data.CameraV + data.CameraW;
}

CameraRay ComputeRayThinlens(PathTracerCameraData data, uint2 pixel, float2 jitter, float2 sample2D)
{
    CameraRay ray;
    ray.origin = data.PosW;
    ray.dir    = ComputeNonNormalizedRayDirPinhole(data, pixel, jitter);

    const float2 apertureSample = SampleConcentricDisk(sample2D);
    const float3 target = ray.origin + ray.dir;
    if (data.ApertureRadius > 0.0)
    {
        ray.origin += data.ApertureRadius *
            (apertureSample.x * normalize(data.CameraU) + apertureSample.y * normalize(data.CameraV));
    }
    ray.dir = normalize(target - ray.origin);

    const float invCos = 1.0 / max(dot(normalize(data.CameraW), ray.dir), 1e-6);
    ray.tMin = data.NearZ * invCos;
    ray.tMax = data.FarZ * invCos;

    ray.origin += ray.dir * ray.tMin;
    ray.tMax   = max(ray.tMax - ray.tMin, 0.0);
    ray.tMin   = 0.0;
    return ray;
}
```

- [ ] **Step 7: Use thin-lens camera rays in raygen**

In `PathTracerSample.rgen`, replace the current primary-ray setup:

```hlsl
    const float2    jitter   = sampleNext2D(sgCamera);
    const float2    uv       = (float2(pixel) + jitter) / float2(dimensions);
    const float2    ndc      = uv * 2.0 - 1.0;

    const float4 worldPos4 = mul(float4(ndc, 0.0, 1.0), g_Const.viewProjInv);
    const float3 origin    = g_Const.cameraPositionAndTime.xyz;
    float3       rayOrigin = origin;
    float3       rayDir    = normalize(worldPos4.xyz / worldPos4.w - origin);
```

with:

```hlsl
    const float2 jitter         = sampleNext2D(sgCamera);
    const float2 subPixelOffset = jitter - 0.5.xx;
    const float2 cameraDoFSample = sampleNext2D(sgCamera);
    CameraRay    cameraRay      = ComputeRayThinlens(g_Const.camera, pixel, subPixelOffset, cameraDoFSample);

    float3 rayOrigin = cameraRay.origin;
    float3 rayDir    = cameraRay.dir;
```

Inside the path loop, replace primary-ray `TMin`/`TMax` setup:

```hlsl
        ray.TMin      = bounce == 0u ? cameraRay.tMin : 0.0;
        ray.TMax      = bounce == 0u ? cameraRay.tMax : 10000.0;
```

This preserves the old near-plane behavior for the first ray and uses robust origins for later rays.

- [ ] **Step 8: Verify camera data and thin-lens symbols**

Run:

```powershell
rg -n "PathTracerCameraData|CameraAperture|CameraFocalDistance|MakePathTracerCameraData|ComputeRayThinlens|SampleConcentricDisk|cameraDoFSample|cameraRay" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "SampleConstants\\) == 480|PathTracerCameraData\\) == 112|SampleConstants.*368 bytes|viewProjInv.*worldPos4" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: first command finds the new camera path. Second command finds the new size guards and no stale production inverse-VP primary-ray path.

- [ ] **Step 9: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "feat(rtxpt): add thin lens camera rays" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Expose DoF Controls and Reset Accumulation

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add RTXPT-fork DoF controls to the Camera section**

In `RTXPTSample.cpp::UpdateUI`, inside the `Camera` collapsing header, add these controls near the scene-camera selector and before clip-plane controls:

```cpp
        if (ResetOnChange(ImGui::InputFloat("Aperture", &m_ReferenceUI.CameraAperture, 0.001f, 0.01f, "%.4f"),
                          "Camera aperture changed"))
        {
            m_ReferenceUI.CameraAperture = std::clamp(m_ReferenceUI.CameraAperture, 0.0f, 1.0f);
        }

        if (ResetOnChange(ImGui::InputFloat("Focal Distance", &m_ReferenceUI.CameraFocalDistance, 0.1f),
                          "Camera focal distance changed"))
        {
            m_ReferenceUI.CameraFocalDistance = std::clamp(m_ReferenceUI.CameraFocalDistance, 0.001f, 1.0e16f);
        }
```

Use the RTXPT-fork labels exactly: `Aperture` and `Focal Distance`.

- [ ] **Step 2: Clamp the values even when ImGui edits are cancelled**

Still in `UpdateUI`, immediately after the two controls, add:

```cpp
        m_ReferenceUI.CameraAperture      = std::clamp(m_ReferenceUI.CameraAperture, 0.0f, 1.0f);
        m_ReferenceUI.CameraFocalDistance = std::clamp(m_ReferenceUI.CameraFocalDistance, 0.001f, 1.0e16f);
```

This matches RTXPT-fork's runtime clamping and keeps accidental negative values out of the constant buffer.

- [ ] **Step 3: Verify UI symbols**

Run:

```powershell
rg -n "Aperture|Focal Distance|CameraAperture|CameraFocalDistance|Camera aperture changed|Camera focal distance changed" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```

Expected: matches in the UI state struct, camera constant upload, and Camera UI section.

- [ ] **Step 4: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): expose reference camera dof controls" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Update Mapping, Run Final Verification, and Record Acceptance

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: all R7-touched files

- [ ] **Step 1: Add R7 mapping rows**

In `RTXPT_FORK_MAPPING.md`, add a `Phase R7 - Shadow/AA Polish` section near the existing phase mapping notes:

```markdown
## Phase R7 - Shadow/AA Polish

| Diligent port | RTXPT-fork source | R7 notes |
|---|---|---|
| `PathTracerHelpers.hlsli::ComputeRayOrigin` | `PathTracer/PathTracerHelpers.hlsli::ComputeRayOrigin` | Same robust offset algorithm, expressed without RTXPT-fork's `select()` helper |
| `PathTracerHelpers.hlsli::ComputeLowGrazingAngleFalloff` | `PathTracer/PathTracerHelpers.hlsli::ComputeLowGrazingAngleFalloff` | Same direct-light shadow terminator fadeout formula |
| `PathTracer::MakeVisibilityOrigin` | `PathTracer/PathTracerNEE.hlsli::ComputeVisibilityRay` | Diligent keeps visibility tracing in raygen helpers; face-normal side is still selected by the shading normal |
| `PathPayload.vertexNormal` | `Scene/ShadingData.hlsli::vertexN` | Closest-hit payload carries the corrected pre-normal-map vertex normal because Diligent does not materialize `ShadingData` |
| `MaterialPTData.shadowNoLFadeout` | `Materials/MaterialPT.h::ShadowNoLFadeout` | Stored in existing Diligent padding at offset 136; material record stays 144 bytes |
| `PathTracerCameraData` + `ComputeRayThinlens` | `PathTracerShared.h::PathTracerCameraData` + `PathTracerHelpers.hlsli::ComputeRayThinlens` | Same camera basis and thin-lens math; Diligent stores the camera block at top-level `SampleConstants.camera` |
| `RTXPTSample` UI labels `Aperture` / `Focal Distance` | `SampleUI.cpp` camera section | Same labels, defaults, and clamp ranges; aperture 0 is the default pinhole path |
```

- [ ] **Step 2: Verify no stale R7 gap text remains in active code**

Run:

```powershell
rg -n "Phase R7|Shadow/AA polish|shadow-ray origin offset|no grazing-angle|no depth of field|Aperture|Focal Distance|ComputeRayThinlens" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: any `Phase R7` matches describe completed work or mapping notes. There are no active-code comments saying DoF or shadow fadeout is missing.

- [ ] **Step 3: Run layout and formatting checks**

Run:

```powershell
rg -n "sizeof\\(MaterialPTData\\) == 144|offsetof\\(MaterialPTData, shadowNoLFadeout\\) == 136|sizeof\\(PathTracerCameraData\\) == 112|sizeof\\(SampleConstants\\) == 480|PathPayload.*160 bytes|MaxPayloadSize.*40" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: all size/layout guards are present and `diff --check` reports no whitespace errors.

- [ ] **Step 4: Build the RTXPT target**

Run from the superproject root:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the RTXPT sample target builds. The C++ `static_assert`s confirm `MaterialPTData == 144`, `PathTracerCameraData == 112`, and `SampleConstants == 480`; the shader compiler accepts `PathPayload == 160`.

- [ ] **Step 5: Runtime acceptance on D3D12 and Vulkan**

Run the RTXPT sample on D3D12 and Vulkan. Acceptance checks:

- Default `Aperture = 0.0` renders the same pinhole framing as before R7 after accumulation reset.
- Increasing `Aperture` while setting a finite `Focal Distance` visibly defocuses out-of-focus geometry and resets accumulation.
- Normal-mapped surfaces with sidecar `ShadowNoLFadeout > 0.0` show reduced hard triangle-shaped shadow terminators at low grazing angles.
- Materials with no sidecar `ShadowNoLFadeout` keep the previous fadeout-free direct-light behavior.
- Shadow acne/leak from NEE rays is reduced versus the scalar-bias path, especially near normal-mapped or high-scale geometry.
- R6 transmission/nested dielectric scenes still render; the R7 payload growth did not regress closest-hit material data.

- [ ] **Step 6: Add a local sidecar smoke material if needed**

If no existing scene material has non-zero `ShadowNoLFadeout`, temporarily edit one local material sidecar during manual validation:

```json
{
  "ShadowNoLFadeout": 0.05
}
```

Use this only for local runtime smoke testing unless the scene asset is meant to become a committed regression fixture.

- [ ] **Step 7: Commit mapping and final notes**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): record R7 RTXPT-fork mappings" -m "Co-Authored-By: GPT 5.5"
```

- [ ] **Step 8: Final status check**

Run:

```powershell
git -C DiligentSamples status --short --branch
git status --short --branch
```

Expected: only intentional submodule pointer changes remain in the superproject, unless other pre-existing unrelated changes were present before Task 0.

## Final Verification Checklist

Run before declaring R7 complete:

```powershell
rg -n "ComputeRayOrigin|ComputeLowGrazingAngleFalloff|ComputeRayThinlens|PathTracerCameraData|CameraAperture|CameraFocalDistance|ShadowNoLFadeout|shadowNoLFadeout|vertexNormal" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "PathPayload.*144 bytes|MaxPayloadSize.*36|SampleConstants\\) == 368|viewProjInv.*worldPos4|no depth of field|no grazing-angle" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
git -C DiligentSamples diff --check -- Samples/RTXPT
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

- First `rg`: finds all R7 features across C++, HLSL, and the mapping doc.
- Second `rg`: no stale old-layout or old-camera-production-path matches remain.
- `diff --check`: no whitespace errors.
- `cmake --build`: RTXPT target builds.

## Self-Review Result

- Spec coverage: G11 is covered by Task 1/2 for face-normal robust shadow origins and grazing fadeout, Task 3/4 for thin-lens DoF, and Task 5 for RTXPT-fork mapping plus validation.
- Placeholder scan: no unresolved placeholder tokens, empty implementation steps, or unspecified "add tests" steps remain. Validation commands and expected results are explicit.
- Type consistency: `shadowNoLFadeout`, `vertexNormal`, `PathTracerCameraData`, `CameraAperture`, and `CameraFocalDistance` use the same names across tasks. Layout sizes are consistent: `MaterialPTData == 144`, `PathPayload == 160`, `PathTracerCameraData == 112`, `SampleConstants == 480`.
