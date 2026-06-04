# RTXPT Realtime G4-G5 PathTrace Pipeline Variants and Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port RTXPT-fork's realtime `PathTrace` body into `DiligentSamples/Samples/RTXPT` by adding REF/BUILD/FILL ray-tracing variants and reproducing `Sample::PathTrace` pass ordering, while excluding standalone `Sample::Denoise` / NRD / final merge implementation.

**Architecture:** Keep the port Diligent-native: `RTXPTRayTracingPass` owns Diligent RT PSO/SRB/SBT variants, `RTXPTRenderTargets` remains the resource owner from G3, and `RTXPTSample` owns frame orchestration. Shader algorithm files stay close to RTXPT-fork for stable-plane path tracing, while Diligent bridge code translates source resource names and scene/material/light access to the existing Diligent sample buffers.

**Tech Stack:** C++17, HLSL 6.5/DXC, Diligent `IRenderDevice`/`IDeviceContext`/ray-tracing PSO/SBT APIs, Diligent UAV barriers through `StateTransitionDesc`, ImGui status UI, RTXPT-fork reference source under `D:/RTXPT-fork/Rtxpt`, PowerShell + `rg` verification.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`, goals `G4 - Realtime Ray-Tracing Pipeline Variants` and `G5 - PathTrace Orchestration`.
- G1/G2/G3 state is already present in this checkout:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp` owns realtime UI/settings and reset flags.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` and `assets/shaders/PathTracer/PathTracerShared.h` already expose `sampleBaseIndex`, `invSubSampleCount`, stable-plane controls, generic tiled-storage strides, current/previous camera/view fields.
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}` already allocate stable-plane resources, throughput, `SpecularHitT`, scratch, and NRD-oriented textures.
- Current G4/G5 gaps:
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` owns one reference PSO/SRB/SBT only.
  - `RTXPTRayTracingPass::Trace()` binds only `u_Output`, `u_Depth`, and `u_ScreenMotionVectors`.
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h` does not define `PATH_TRACER_MODE_REFERENCE`, `PATH_TRACER_MODE_BUILD_STABLE_PLANES`, or `PATH_TRACER_MODE_FILL_STABLE_PLANES`.
  - `PathState.hlsli`, `PathPayload.hlsli`, `PathTracerTypes.hlsli`, `StablePlanes.hlsli`, and `PathTracerStablePlanes.hlsli` are absent from the Diligent shader tree.
  - `RTXPTSample::Render()` still returns a disabled realtime fallback immediately when `m_RealtimeUI.RealtimeMode` is true.
- Original RTXPT-fork invariants:
  - `D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp`: `CreateRTPipelines()` creates exactly three `PathTracerSample.hlsl` variants for `PATH_TRACER_MODE_REFERENCE`, `PATH_TRACER_MODE_BUILD_STABLE_PLANES`, and `PATH_TRACER_MODE_FILL_STABLE_PLANES`.
  - `D:/RTXPT-fork/Rtxpt/Sample.cpp:2438`: `Sample::PathTrace()` does realtime BUILD pre-pass, `VBufferExport`, `LightsBaker.UpdateEnd`, FILL/REFERENCE sub-sample loop, optional RTXDI final hooks, denoising guide bake, and optional stable-plane debug visualization.
  - `D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h`: `SampleMiniConstants` is a 64-byte `uint4 x 4` block; in `PathTrace`, `params.x` carries `subSampleIndex`.
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`: path-tracer mode values are `REFERENCE = 0`, `BUILD_STABLE_PLANES = 1`, `FILL_STABLE_PLANES = 2`.

## Scope Boundaries

- This plan implements G4/G5 only: ray-tracing variants and `PathTrace` orchestration.
- This plan does not implement NRD, standalone `Denoise`, denoiser prepare/final merge, or `NoDenoiserFinalMerge`; those are G6-G9/G7 responsibilities.
- This plan must not present realtime FILL output as final HDR color. In realtime mode, `FILL_STABLE_PLANES` writes noisy radiance into stable-plane storage; final `OutputColor` production remains gated until merge/denoise plans run.
- `Denoising Guides Bake` is kept as an ordered G5 hook. The compute pass bodies belong to G6, but G5 must define the call point and status so later G6 work plugs in without reshaping `PathTrace`.
- RTXDI/ReSTIR final hooks are preserved as disabled, explicitly reported hooks. This plan does not claim parity for RTXDI-enabled settings until a separate RTXDI plan exists.
- Reference mode must continue to dispatch the reference variant and then use existing accumulation, tone mapping, and presentation behavior.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
  - Adds CPU-side `SampleMiniConstants` for per-dispatch sub-sample data.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
  - Mirrors `SampleMiniConstants`.
  - Renames the current material-hit payload to avoid conflicting with RTXPT-fork's packed `PathPayload`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h`
  - Adds path-tracer mode constants and a default `PATH_TRACER_MODE`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
  - Adds `g_MiniConst`.
  - Adds source-compatible Diligent resource names for output, motion, depth, throughput, stable planes, stable radiance, and specular hit distance.
  - Adds Diligent equivalents of source bridge helpers consumed by stable-plane code.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathPayload.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli`
  - Ports RTXPT-fork stable-plane path-state and payload logic under Apache-compatible local headers by manual translation.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
  - Adds realtime build/fill branches and keeps reference behavior intact.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
  - Dispatches reference logic for `PATH_TRACER_MODE_REFERENCE`.
  - Dispatches source-compatible stable-plane build/fill logic for realtime variants.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit`
  - Uses the renamed material-hit payload for reference behavior and the packed source `PathPayload` for stable-plane variants.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/ExportVisibilityBuffer.csh`
  - Ports live RTXPT-fork `VBufferExport` debug behavior. It does not revive upstream's disabled `#if 0` export body.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.cpp`
  - Dedicated compute pass for the G5 `VBufferExport` timing anchor.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
  - Replaces single RT state with three variants and adds dispatch input objects.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Adds Diligent `PathTrace` orchestration and realtime status.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`
  - Extends `UpdateEnd()` signature to accept current depth and motion-vector views, preserving source call semantics even while the current Diligent LightsBaker only uses them for future feedback paths.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Registers new C++ and shader files.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  - Records G4/G5 source-to-port mapping and explicitly marks Denoise/NRD/final merge as out of this plan.

---

### Task 0: Baseline Preflight and Source Contract

**Files:**
- Verify: top-level repository
- Verify: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/Sample.cpp`
- Verify: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`

- [ ] **Step 1: Confirm workspace state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files are preserved; do not revert user changes.

- [ ] **Step 2: Confirm source G4/G5 anchors**

Run:

```powershell
rg -n "CreateRTPipelines|PATH_TRACER_MODE_REFERENCE|PATH_TRACER_MODE_BUILD_STABLE_PLANES|PATH_TRACER_MODE_FILL_STABLE_PLANES|void Sample::PathTrace|PathTracePrePass|VBufferExport|LightsBaker|SampleMiniConstants" D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp D:/RTXPT-fork/Rtxpt/Sample.cpp D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h
```

Expected:

```text
AdvancedSample.cpp contains the three CreateVariant calls.
Sample.cpp contains Sample::PathTrace, PathTracePrePass, VBufferExport, LightsBaker::UpdateEnd, and the subSampleIndex loop.
Config.h contains the three PATH_TRACER_MODE values.
SampleConstantBuffer.h contains struct SampleMiniConstants.
```

- [ ] **Step 3: Confirm Diligent G4/G5 gaps**

Run:

```powershell
rg -n "PATH_TRACER_MODE|PathState|PathPayload|StablePlanes|SampleMiniConstants|m_ptPipelineBuildStablePlanes|VBufferExport|RealtimeMode\\)" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected before implementation: stable-plane resources may appear in render-target/status code, but there is no three-variant RT pass, no `SampleMiniConstants`, no source `PathState`/`PathPayload` files, and `RTXPTSample::Render()` still clears and returns when realtime mode is selected.

- [ ] **Step 4: No commit for preflight**

No source changes are made in Task 0. Do not create a commit for this task.

### Task 1: Add Path-Tracer Mode and Mini-Constant Contracts

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/SampleConstantBuffer.h`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`

- [ ] **Step 1: Add CPU `SampleMiniConstants`**

In `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`, add this struct after `SampleConstants`:

```cpp
struct SampleMiniConstants
{
    uint4 params  = {};
    uint4 params1 = {};
    uint4 params2 = {};
    uint4 params3 = {};
};
static_assert(sizeof(SampleMiniConstants) == 64, "SampleMiniConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Add shader `SampleMiniConstants`**

In `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`, add this struct after `SampleConstants`:

```hlsl
struct SampleMiniConstants
{
    uint4 params;
    uint4 params1;
    uint4 params2;
    uint4 params3;
};
```

- [ ] **Step 3: Add shader mode constants**

Replace the body of `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h` with:

```hlsl
#ifndef __CONFIG_H__
#define __CONFIG_H__

// Path-tracer compile-time configuration. Values mirror
// D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h.
#define PATH_TRACER_MODE_REFERENCE              0
#define PATH_TRACER_MODE_BUILD_STABLE_PLANES    1
#define PATH_TRACER_MODE_FILL_STABLE_PLANES     2

#ifndef PATH_TRACER_MODE
#    define PATH_TRACER_MODE PATH_TRACER_MODE_REFERENCE
#endif

#ifndef ENABLE_DEBUG_VIZUALISATIONS
#    define ENABLE_DEBUG_VIZUALISATIONS 0
#endif

#ifndef ENABLE_DEBUG_DELTA_TREE_VIZUALISATION
#    define ENABLE_DEBUG_DELTA_TREE_VIZUALISATION 0
#endif

// ENABLE_MATERIAL_TEXTURES and MATERIAL_TEXTURE_COUNT are supplied by C++ when the
// bindless material-texture table exists.

#endif // __CONFIG_H__
```

- [ ] **Step 4: Verify contracts**

Run:

```powershell
rg -n "struct SampleMiniConstants|PATH_TRACER_MODE_REFERENCE|PATH_TRACER_MODE_BUILD_STABLE_PLANES|PATH_TRACER_MODE_FILL_STABLE_PLANES" DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h
```

Expected: both C++ and HLSL define `SampleMiniConstants`; `Config.h` defines the three path-tracer mode constants and default mode.

- [ ] **Step 5: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Config.h
git commit -m "feat(rtxpt): add realtime pathtrace mode constants"
```

### Task 2: Split Current Material-Hit Payload from RTXPT-fork Packed Path Payload

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathPayload.hlsli`

- [ ] **Step 1: Rename the current reference material payload**

In `PathTracerShared.h`, rename the existing reference payload struct:

```hlsl
struct RTXPTMaterialHitPayload
{
    float3 worldPos;
    float  hitDistance;

    float3 worldNormal;
    uint   hitFlag;

    float3 faceNormal;
    uint   materialID;

    float3 baseColor;
    float  metallic;

    float3 emission;
    float  roughness;

    float emissiveLightPdf;
    float ior;
    float transmissionFactor;
    float diffuseTransmissionFactor;

    float3 transmissionColor;
    float  volumeAttenuationDistance;

    float3 volumeAttenuationColor;
    uint   materialFlags;

    uint  nestedPriority;
    uint  frontFacing;
    uint  thinSurface;
    float alpha;

    float3 vertexNormal;
    float  shadowNoLFadeout;
};
```

Do not leave a `struct PathPayload` material-hit declaration in `PathTracerShared.h`; the name `PathPayload` is reserved for RTXPT-fork's packed path-state payload.

- [ ] **Step 2: Update reference shader users**

Replace material payload references in the current reference path:

```powershell
rg -n "\\bPathPayload\\b" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit
```

For the current reference implementation, replace each material-hit payload type with `RTXPTMaterialHitPayload`. Example:

```hlsl
RTXPTMaterialHitPayload payload = PathTracer::MakeEmptyPayload(1u);
```

and:

```hlsl
[shader("closesthit")]
void main(inout RTXPTMaterialHitPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
```

- [ ] **Step 3: Rename the reference helper return type**

In `PathTracer.hlsli`, update `PathTracer::MakeEmptyPayload`:

```hlsl
RTXPTMaterialHitPayload MakeEmptyPayload(uint hitFlag)
{
    RTXPTMaterialHitPayload payload;
    payload.worldPos    = float3(0.0, 0.0, 0.0);
    payload.hitDistance = -1.0;
    payload.worldNormal = float3(0.0, 1.0, 0.0);
    payload.hitFlag     = hitFlag;
    payload.faceNormal  = float3(0.0, 1.0, 0.0);
    payload.materialID  = 0u;
    payload.baseColor   = float3(0.0, 0.0, 0.0);
    payload.emission    = float3(0.0, 0.0, 0.0);
    payload.metallic    = 0.0;
    payload.roughness   = 1.0;
    payload.emissiveLightPdf = 0.0;
    payload.ior = 1.5;
    payload.transmissionFactor = 0.0;
    payload.diffuseTransmissionFactor = 0.0;
    payload.transmissionColor = float3(1.0, 1.0, 1.0);
    payload.volumeAttenuationDistance = 3.402823466e+38;
    payload.volumeAttenuationColor = float3(1.0, 1.0, 1.0);
    payload.materialFlags = 0u;
    payload.nestedPriority = 14u;
    payload.frontFacing = 1u;
    payload.thinSurface = 1u;
    payload.alpha = 1.0;
    payload.vertexNormal = payload.worldNormal;
    payload.shadowNoLFadeout = 0.0;
    return payload;
}
```

- [ ] **Step 4: Verify `PathPayload` name is free**

Run:

```powershell
rg -n "\\bPathPayload\\b" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected after Task 2: no matches except comments that mention the upcoming packed source payload. If comments mention it, they must say `PathPayload.hlsli` and must not refer to the current material-hit payload.

- [ ] **Step 5: Build reference mode**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build exits 0. Reference mode still has one RT PSO and compiles with `RTXPTMaterialHitPayload`.

- [ ] **Step 6: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit
git commit -m "refactor(rtxpt): reserve PathPayload for realtime path state"
```

### Task 3: Port Stable-Plane Shader State and Bridge Resources

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathPayload.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathPayload.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathState.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerTypes.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerStablePlanes.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/Bindings/ShaderResourceBindings.hlsli`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerBridgeDonut.hlsli`

- [ ] **Step 1: Create `PathPayload.hlsli`**

Translate `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathPayload.hlsli` into the Diligent shader tree. Keep the include guard and packed payload shape:

```hlsl
#ifndef __PATH_PAYLOAD_HLSLI__
#define __PATH_PAYLOAD_HLSLI__

#include "Config.h"

struct PathPayload
{
    uint4 packed[5];

#ifdef PATH_STATE_DEFINED
    static PathPayload pack(const PathState path);
    static PathState unpack(const PathPayload p);
#endif
};

// The pack/unpack bodies match RTXPT-fork and use PathState fields.

#endif // __PATH_PAYLOAD_HLSLI__
```

The implementation body must preserve the source field order:

```text
packed[0] = path.PackOriginId
packed[1] = path.PackDirSceneLength
packed[2].xy = path.pack23
packed[2].zw = path.imageXformPacked for BUILD, path.pack45 otherwise
packed[3].xy = interior slots when enabled
packed[3].z = path.packedCounters
packed[3].w = path.stableBranchID
packed[4].x = path.rayCone.widthSpreadAngleFP16
packed[4].y = path.pack0
packed[4].z = path.pack1
packed[4].w = path.flagsAndVertexIndex
```

- [ ] **Step 2: Create `PathState.hlsli`**

Translate the RTXPT-fork `PathState` file, preserving:

```text
kVertexIndexBitCount = 10
kStablePlaneIndexBitOffset = 14 + kVertexIndexBitCount
PathFlags stable-plane bits:
  stablePlaneIndexBit0
  stablePlaneIndexBit1
  stablePlaneOnPlane
  stablePlaneOnBranch
  stablePlaneBaseScatterDiff
  exportSpecHitTQueued
  stablePlaneOnDominantBranch
PackedCounters:
  DiffuseBounces
  RejectedHits
  BouncesFromStablePlane
PathState fields:
  PackOriginId
  PackDirSceneLength
  pack23
  imageXformPacked for BUILD or pack45 otherwise
  interiorList when enabled
  packedCounters
  stableBranchID
  rayCone
  pack0
  pack1
  flagsAndVertexIndex
```

Resolve include differences to existing Diligent files:

```hlsl
#include "Config.h"
#include "PathTracerHelpers.hlsli"
#include "PathTracerShared.h"
#include "Rendering/Materials/InteriorList.hlsli"
#include "Utils/SampleGenerators.hlsli"
```

When an upstream include path is not present in Diligent, move only the required helper into `PathTracerHelpers.hlsli` with the same function name used by source stable-plane code.

- [ ] **Step 3: Create `StablePlanes.hlsli`**

Translate the source `StablePlane` struct exactly enough to match G3's CPU ABI:

```hlsl
struct StablePlane
{
    float3 RayOrigin;
    float  LastRayTCurrent;
    float3 RayDir;
    float  SceneLength;
    uint3  PackedThpAndMVs;
    uint   VertexIndexAndRoughness;
    uint3  DenoiserPackedBSDFEstimate;
    uint   PackedNormal;
    uint2  PackedNoisyRadianceAndSpecAvg;
    uint   FlagsAndVertexIndex;
    uint   PackedCounters;
};
```

Preserve these constants:

```hlsl
static const uint cStablePlaneMaxVertexIndex   = 15;
static const uint cStablePlaneInvalidBranchID  = 0xFFFFFFFF;
static const uint cStablePlaneEnqueuedBranchID = 0xFFFFFFFE;
static const uint cStablePlaneJustStartedID    = 0;
```

Use existing G2 stride fields:

```hlsl
uint PixelToAddress(uint2 pixelPos, uint planeIndex)
{
    return GenericTSPixelToAddress(pixelPos,
                                   planeIndex,
                                   PTConstants.genericTSLineStride,
                                   PTConstants.genericTSPlaneStride);
}
```

- [ ] **Step 4: Add bridge resources and mini constants**

In `PathTracerBridge.hlsli`, after the existing global resources, add:

```hlsl
ConstantBuffer<SampleMiniConstants> g_MiniConst;

VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_OutputColor;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>        u_Depth;
VK_IMAGE_FORMAT("rg16f")   RWTexture2D<float2>       u_MotionVectors;
VK_IMAGE_FORMAT("r32ui")   RWTexture2D<uint>         u_Throughput;
VK_IMAGE_FORMAT("r32f")    RWTexture2D<float>        u_SpecularHitT;
VK_IMAGE_FORMAT("rgba16f") RWTexture2D<float4>       u_StableRadiance;
VK_IMAGE_FORMAT("r32ui")   RWTexture2DArray<uint>    u_StablePlanesHeader;
RWStructuredBuffer<StablePlane>                      u_StablePlanesBuffer;
```

Keep the old names only where reference diagnostics still need them. Runtime binding in Task 5 must use source names for normal execution.

- [ ] **Step 5: Add source-compatible bridge helpers**

Add helpers consumed by source stable-plane code:

```hlsl
namespace Bridge
{
    uint getSampleIndex()
    {
        return g_Const.ptConsts.sampleBaseIndex + g_MiniConst.params.x;
    }

    float getNoisyRadianceAttenuation()
    {
        return g_Const.ptConsts.invSubSampleCount;
    }

    uint2 getPixelPosition()
    {
        return DispatchRaysIndex().xy;
    }
}
```

Also add Diligent versions of the source export helpers:

```hlsl
namespace Bridge
{
    void ExportSurfaceInit(uint2 pixelPos)
    {
        u_Depth[pixelPos]        = 0.0;
        u_MotionVectors[pixelPos] = float2(0.0, 0.0);
        u_Throughput[pixelPos]   = 0u;
        u_SpecularHitT[pixelPos] = 0.0;
    }

    void ExportNonSurface(uint2 pixelPos)
    {
        u_Depth[pixelPos]        = 0.0;
        u_MotionVectors[pixelPos] = float2(0.0, 0.0);
        u_Throughput[pixelPos]   = 0u;
    }
}
```

For `ExportSurface`, reuse existing Diligent hit data and previous-frame view constants to write:

```text
u_Depth[pixel] = current dominant surface view-space or camera-linear depth expected by later passes.
u_MotionVectors[pixel] = screen motion vector computed from current and previous view/projection.
u_Throughput[pixel] = packed throughput helper matching RTXPT-fork's R11G11B10-style use.
```

Do not write `u_OutputColor` from BUILD mode.

- [ ] **Step 6: Create `PathTracerTypes.hlsli` and `PathTracerStablePlanes.hlsli`**

Translate the source files with Diligent include paths. Preserve these source-facing functions:

```text
PathTracer::WorkingContext
GetWorkingContext()
PathTracer::StablePlanesOnScatter
PathTracer::StablePlanesHandleHit
PathTracer::StablePlanesHandleMiss
StablePlanesContext::StartPixel
StablePlanesContext::CommitDenoiserRadiance
StablePlanesContext::GetAllRadiance
StablePlanesContext::LoadDominantIndex
StablePlanesContext::LoadStablePlane
StablePlanesContext::FindNextToExplore
StablePlanesContext::ExplorationStart
```

Where source code references RTXDI-only resources, keep the branch compiled out by `PT_USE_RESTIR_DI == 0` and `PT_USE_RESTIR_GI == 0` in this plan.

- [ ] **Step 7: Register new shader files in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new shader files to `SHADERS` near other `PathTracer` files:

```cmake
    assets/shaders/PathTracer/PathPayload.hlsli
    assets/shaders/PathTracer/PathState.hlsli
    assets/shaders/PathTracer/PathTracerTypes.hlsli
    assets/shaders/PathTracer/StablePlanes.hlsli
    assets/shaders/PathTracer/PathTracerStablePlanes.hlsli
```

- [ ] **Step 8: Verify source shader files exist and names match**

Run:

```powershell
rg -n "struct PathPayload|struct PathState|struct StablePlane|StablePlanesContext|CommitDenoiserRadiance|PathTracerStablePlanes" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: all five new files are discovered and contain the source-compatible symbol names.

- [ ] **Step 9: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathPayload.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): port realtime stable-plane shader state"
```

### Task 4: Port Raygen Build/Fill Control Flow

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerSample.hlsl`
- Read: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Include mode config first**

At the top of `PathTracerSample.rgen`, include `Config.h` before mode-sensitive logic:

```hlsl
#include "Config.h"
```

Keep diagnostic blocks guarded by `RTXPT_SCREEN_PATTERN_DIAGNOSTIC` and `RTXPT_MINIMAL_TRACE_RAY_DIAGNOSTIC`.

- [ ] **Step 2: Route reference mode to current reference raygen**

Wrap the current non-diagnostic raygen body in:

```hlsl
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE

[shader("raygeneration")]
void main()
{
    // Existing Diligent reference raygen body using RTXPTMaterialHitPayload.
}

#else

// Realtime source-compatible raygen body from Step 3.

#endif
```

Expected behavior: reference mode continues to write `u_OutputColor`, `u_Depth`, and `u_MotionVectors` through the existing reference algorithm.

- [ ] **Step 3: Add realtime raygen body**

For BUILD/FILL modes, port the source `RAYGEN_ENTRY` flow into `main()`:

```hlsl
#include "PathTracerTypes.hlsli"
#include "PathPayload.hlsli"
#include "PathState.hlsli"
#include "StablePlanes.hlsli"
#include "PathTracerStablePlanes.hlsli"

float2 FirstHitFromVBuffer(inout PathState path,
                           const uint basePlaneIndex,
                           const PathTracer::WorkingContext workingContext)
{
    const uint2 pixelPos = path.GetPixelPos();
    float2 tMinMax = float2(0.0, kMaxRayTravel);

    StablePlane sp = workingContext.StablePlanes.LoadStablePlane(pixelPos, basePlaneIndex);
    uint stableBranchID = workingContext.StablePlanes.GetBranchID(pixelPos, basePlaneIndex);

    float sceneLength     = sp.SceneLength;
    float lastRayTCurrent = sp.LastRayTCurrent;
    uint  vertexIndex     = sp.VertexIndexAndRoughness >> 16;

    float3 thp;
    float3 dummy;
    UnpackTwoFp32ToFp16(sp.PackedThpAndMVs, thp, dummy);

    bool isMiss = false;
    if (!isfinite(sceneLength))
    {
        sceneLength = kMaxRayTravel;
        isMiss = true;
    }
    else
    {
        tMinMax.x = lastRayTCurrent * 0.99;
        tMinMax.y = lastRayTCurrent * 1.01;
        sceneLength -= lastRayTCurrent;
    }

    path.setVertexIndex(vertexIndex - 1);
    path.SetDir(sp.RayDir);
    path.SetOrigin(sp.RayOrigin);
    path.setFlag(PathFlags::stablePlaneOnPlane, true);
    path.setFlag(PathFlags::stablePlaneOnBranch, true);
    path.setStablePlaneIndex(basePlaneIndex);
    path.stableBranchID = stableBranchID;
    path.SetThp(thp);
    path.SetL(float4(0.0, 0.0, 0.0, 0.0));

    const uint dominantSPIndex = workingContext.StablePlanes.LoadDominantIndex(pixelPos);
    path.setFlag(PathFlags::stablePlaneOnDominantBranch, dominantSPIndex == basePlaneIndex);
    path.setCounter(PackedCounters::BouncesFromStablePlane, 0);

    if (PathTracer::HasFinishedSurfaceBounces(path.getVertexIndex() + 1,
                                              path.getCounter(PackedCounters::DiffuseBounces)))
        path.setTerminateAtNextBounce();

    PathTracer::UpdatePathTravelledLengthOnly(path, sceneLength);

    if (isMiss)
        PathTracer::HandleMiss(path, path.GetOrigin(), path.GetDir(), sceneLength, workingContext);

    return tMinMax;
}

[shader("raygeneration")]
void main()
{
    const uint2 pixelPos = DispatchRaysIndex().xy;

    PathState path = PathTracer::EmptyPathInitialize(pixelPos,
                                                     g_Const.ptConsts.camera.PixelConeSpreadAngle);
    PathTracer::SetupPathPrimaryRay(path, Bridge::computeCameraRay(pixelPos));

    PathTracer::WorkingContext workingContext = GetWorkingContext();
    PathTracer::StartPixel(path, workingContext);

#if PATH_TRACER_MODE == PATH_TRACER_MODE_FILL_STABLE_PLANES
    float2 tMinMax = FirstHitFromVBuffer(path, 0u, workingContext);
#else
    float2 tMinMax = float2(0.0, kMaxRayTravel);
#endif

    while (path.isActive())
    {
        nextHit(path, tMinMax, workingContext);
        postProcessHit(path, workingContext);
    }

    PathTracer::CommitPixel(path, workingContext);
}
```

The helper bodies `nextHit`, `postProcessHit`, `ValidateNaNs`, and build-mode exploration must follow `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerSample.hlsl` with Diligent `TraceRay(t_SceneBVH, ...)` naming.

- [ ] **Step 4: Port mode-sensitive `PathTracer.hlsli` branches**

Add source branch behavior while keeping reference mode isolated:

```hlsl
#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE
#    include "PathState.hlsli"
#    include "StablePlanes.hlsli"
#    include "PathTracerStablePlanes.hlsli"
#endif
```

For realtime modes, preserve these source semantics:

```text
BUILD:
  StartPixel clears stable radiance, stable plane headers, depth/motion/throughput/specular-hit defaults.
  AccumulatePathRadiance writes stable emissive/environment radiance into StableRadiance.
  CommitPixel does not write OutputColor.
  Russian roulette and noisy NEE are disabled.
FILL:
  FirstHitFromVBuffer restores base stable plane 0.
  AccumulatePathRadiance only accumulates noisy radiance off the stable branch.
  CommitPixel calls StablePlanesContext::CommitDenoiserRadiance.
  Specular-hit distance is captured for the dominant denoising layer.
REFERENCE:
  Current Diligent reference path still writes OutputColor directly.
```

- [ ] **Step 5: Adapt hit/miss/any-hit shader payloads per mode**

In `.rchit`, `.rmiss`, and `.rahit`, use conditional payload type selection:

```hlsl
#include "Config.h"

#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
using ActiveRayPayload = RTXPTMaterialHitPayload;
#else
#    include "PathState.hlsli"
#    include "PathPayload.hlsli"
using ActiveRayPayload = PathPayload;
#endif
```

For realtime modes, closest-hit must unpack `PathPayload`, call source-compatible hit handling, then repack:

```hlsl
[shader("closesthit")]
void main(inout ActiveRayPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
#if PATH_TRACER_MODE == PATH_TRACER_MODE_REFERENCE
    // Existing material-hit payload fill.
#else
    PathState path = PathPayload::unpack(Payload);
    PathTracer::WorkingContext workingContext = GetWorkingContext();
    PathTracer::HandleHit(path, Attributes, workingContext);
    Payload = PathPayload::pack(path);
#endif
}
```

Miss mode mirrors the source behavior:

```hlsl
#if PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE
PathState path = PathPayload::unpack(Payload);
PathTracer::WorkingContext workingContext = GetWorkingContext();
PathTracer::HandleMiss(path, WorldRayOrigin(), WorldRayDirection(), kMaxRayTravel, workingContext);
Payload = PathPayload::pack(path);
#endif
```

Any-hit keeps Diligent alpha-test/alpha-blend rejection logic and works with both payload types because it does not need to modify payload fields.

- [ ] **Step 6: Verify shader symbols**

Run:

```powershell
rg -n "PATH_TRACER_MODE|FirstHitFromVBuffer|CommitDenoiserRadiance|StablePlanesHandleHit|StablePlanesHandleMiss|ActiveRayPayload|RTXPTMaterialHitPayload|PathPayload::pack" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: raygen has mode branches and source flow, hit/miss shaders select active payload by mode, and stable-plane commit functions are referenced.

- [ ] **Step 7: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit
git commit -m "feat(rtxpt): add realtime pathtrace shader variants"
```

### Task 5: Upgrade `RTXPTRayTracingPass` to Three RT Variants

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp`

- [ ] **Step 1: Add variant enums and dispatch structs**

In `RTXPTRayTracingPass.hpp`, add:

```cpp
enum class RTXPTPathTraceVariant : Uint32
{
    Reference          = 0,
    BuildStablePlanes  = 1,
    FillStablePlanes   = 2,
    Count
};

struct RTXPTRayTracingDispatch
{
    ITextureView*        pOutputColorUAV          = nullptr;
    ITextureView*        pDepthUAV                = nullptr;
    ITextureView*        pMotionVectorsUAV        = nullptr;
    ITextureView*        pThroughputUAV           = nullptr;
    ITextureView*        pSpecularHitTUAV         = nullptr;
    ITextureView*        pStableRadianceUAV       = nullptr;
    ITextureView*        pStablePlanesHeaderUAV   = nullptr;
    IBufferView*         pStablePlanesBufferUAV   = nullptr;
    const SampleMiniConstants* pMiniConstants     = nullptr;
    Uint32               Width                    = 0;
    Uint32               Height                   = 0;
};
```

Add per-variant stats:

```cpp
struct RTXPTRayTracingVariantStats
{
    bool   Ready             = false;
    bool   LastTraceExecuted = false;
    Uint32 TraceCount        = 0;
};
```

- [ ] **Step 2: Replace single PSO/SRB/SBT fields**

In `RTXPTRayTracingPass`, replace:

```cpp
RefCntAutoPtr<IPipelineState>         m_PSO;
RefCntAutoPtr<IShaderResourceBinding> m_SRB;
RefCntAutoPtr<IShaderBindingTable>    m_SBT;
```

with:

```cpp
struct VariantState
{
    RefCntAutoPtr<IPipelineState>         PSO;
    RefCntAutoPtr<IShaderResourceBinding> SRB;
    RefCntAutoPtr<IShaderBindingTable>    SBT;
    RTXPTRayTracingVariantStats           Stats;
};

std::array<VariantState, static_cast<size_t>(RTXPTPathTraceVariant::Count)> m_Variants;
RefCntAutoPtr<IBuffer> m_MiniConstantsCB;
```

Keep shared static resources (`m_TLAS`, `m_IndexBufferView`, bridge stats) outside variant state.

- [ ] **Step 3: Create a mini-constants uniform buffer**

In `Initialize()`, after validating device/context/factory:

```cpp
BufferDesc MiniCBDesc;
MiniCBDesc.Name      = "RTXPT SampleMiniConstants";
MiniCBDesc.Size      = sizeof(SampleMiniConstants);
MiniCBDesc.BindFlags = BIND_UNIFORM_BUFFER;
MiniCBDesc.Usage     = USAGE_DYNAMIC;
MiniCBDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
pDevice->CreateBuffer(MiniCBDesc, nullptr, &m_MiniConstantsCB);
if (!m_MiniConstantsCB)
{
    DEV_ERROR("Failed to create RTXPT SampleMiniConstants buffer");
    return false;
}
```

- [ ] **Step 4: Factor variant creation**

In `RTXPTRayTracingPass.cpp`, add:

```cpp
const char* GetVariantName(RTXPTPathTraceVariant Variant)
{
    switch (Variant)
    {
        case RTXPTPathTraceVariant::Reference:         return "Reference";
        case RTXPTPathTraceVariant::BuildStablePlanes: return "BuildStablePlanes";
        case RTXPTPathTraceVariant::FillStablePlanes:  return "FillStablePlanes";
        default:                                       return "Unknown";
    }
}

int GetVariantModeMacro(RTXPTPathTraceVariant Variant)
{
    switch (Variant)
    {
        case RTXPTPathTraceVariant::Reference:         return 0;
        case RTXPTPathTraceVariant::BuildStablePlanes: return 1;
        case RTXPTPathTraceVariant::FillStablePlanes:  return 2;
        default:                                       return 0;
    }
}
```

Then create all three variants with:

```cpp
ShaderMacroHelper Macros;
Macros.Add("PATH_TRACER_MODE", GetVariantModeMacro(Variant));
Macros.Add("RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF", EnableLDSamplerForBSDF ? 1 : 0);
```

Keep material texture and any-hit macros identical across variants.

- [ ] **Step 5: Bind source-compatible dynamic UAV names**

The variant resource layout must use these dynamic names for all normal modes:

```cpp
ResourceLayout
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_Depth", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_MotionVectors", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_Throughput", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_SpecularHitT", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_StableRadiance", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_StablePlanesHeader", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
    .AddVariable(SHADER_TYPE_RAY_GEN, "u_StablePlanesBuffer", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
```

Bind `g_MiniConst` as static for raygen/hit/miss stages that reflect it:

```cpp
SetStaticForStages(ConstStages | HitStages, "g_MiniConst", m_MiniConstantsCB, "mini constants", false);
```

For `Reference`, stable-plane variables may not reflect; use `Required = false`. For BUILD/FILL, missing stable-plane variables is a hard failure.

- [ ] **Step 6: Use correct payload size**

Set:

```cpp
PSOCreateInfo.RayTracingPipeline.MaxRecursionDepth = 1;
PSOCreateInfo.RayTracingPipeline.ShaderRecordSize  = 0;
PSOCreateInfo.MaxAttributeSize = static_cast<Uint32>(sizeof(float) * 2);
PSOCreateInfo.MaxPayloadSize   = static_cast<Uint32>(sizeof(float) * 40);
```

Reason: the reference material-hit payload is 160 bytes and the realtime packed `PathPayload` is 80 bytes. Using the larger size keeps all three variants valid and preserves current reference behavior.

- [ ] **Step 7: Add dispatch method**

Replace `Trace(...)` with:

```cpp
bool Dispatch(IDeviceContext*               pContext,
              RTXPTPathTraceVariant         Variant,
              const RTXPTRayTracingDispatch& Dispatch);
```

Implementation requirements:

```cpp
if (!IsReady(Variant) || pContext == nullptr)
    return false;
if (Dispatch.Width == 0 || Dispatch.Height == 0)
    return false;
if (Dispatch.pOutputColorUAV == nullptr || Dispatch.pDepthUAV == nullptr || Dispatch.pMotionVectorsUAV == nullptr)
    return false;
if (Variant != RTXPTPathTraceVariant::Reference &&
    (Dispatch.pThroughputUAV == nullptr ||
     Dispatch.pSpecularHitTUAV == nullptr ||
     Dispatch.pStableRadianceUAV == nullptr ||
     Dispatch.pStablePlanesHeaderUAV == nullptr ||
     Dispatch.pStablePlanesBufferUAV == nullptr))
    return false;
```

Bind reflected variables by name:

```cpp
SetDynamicRaygen("u_OutputColor", Dispatch.pOutputColorUAV, true);
SetDynamicRaygen("u_Depth", Dispatch.pDepthUAV, true);
SetDynamicRaygen("u_MotionVectors", Dispatch.pMotionVectorsUAV, true);
SetDynamicRaygen("u_Throughput", Dispatch.pThroughputUAV, Variant != RTXPTPathTraceVariant::Reference);
SetDynamicRaygen("u_SpecularHitT", Dispatch.pSpecularHitTUAV, Variant != RTXPTPathTraceVariant::Reference);
SetDynamicRaygen("u_StableRadiance", Dispatch.pStableRadianceUAV, Variant != RTXPTPathTraceVariant::Reference);
SetDynamicRaygen("u_StablePlanesHeader", Dispatch.pStablePlanesHeaderUAV, Variant != RTXPTPathTraceVariant::Reference);
SetDynamicRaygen("u_StablePlanesBuffer", Dispatch.pStablePlanesBufferUAV, Variant != RTXPTPathTraceVariant::Reference);
```

Update mini constants before every dispatch:

```cpp
SampleMiniConstants Mini = Dispatch.pMiniConstants != nullptr ? *Dispatch.pMiniConstants : SampleMiniConstants{};
{
    MapHelper<SampleMiniConstants> Mapped{pContext, m_MiniConstantsCB, MAP_WRITE, MAP_FLAG_DISCARD};
    *Mapped = Mini;
}
```

Trace:

```cpp
pContext->SetPipelineState(State.PSO);
pContext->CommitShaderResources(State.SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

TraceRaysAttribs Attribs;
Attribs.DimensionX = Dispatch.Width;
Attribs.DimensionY = Dispatch.Height;
Attribs.pSBT       = State.SBT;
pContext->TraceRays(Attribs);
```

- [ ] **Step 8: Add UAV barrier helper**

Add a public helper:

```cpp
static void InsertUAVBarrier(IDeviceContext* pContext, IBuffer* pBuffer)
{
    if (pContext == nullptr || pBuffer == nullptr)
        return;
    StateTransitionDesc Barrier{pBuffer,
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                STATE_TRANSITION_FLAG_UPDATE_STATE};
    pContext->TransitionResourceState(Barrier);
}
```

and texture overload:

```cpp
static void InsertUAVBarrier(IDeviceContext* pContext, ITextureView* pView)
{
    if (pContext == nullptr || pView == nullptr || pView->GetTexture() == nullptr)
        return;
    StateTransitionDesc Barrier{pView->GetTexture(),
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                RESOURCE_STATE_UNORDERED_ACCESS,
                                STATE_TRANSITION_FLAG_UPDATE_STATE};
    pContext->TransitionResourceState(Barrier);
}
```

Diligent treats `UNORDERED_ACCESS -> UNORDERED_ACCESS` as a UAV barrier, matching the source `setBufferState(... UnorderedAccess)` ordering intent.

- [ ] **Step 9: Verify variant construction**

Run:

```powershell
rg -n "RTXPTPathTraceVariant|BuildStablePlanes|FillStablePlanes|PATH_TRACER_MODE|u_StableRadiance|u_StablePlanesHeader|g_MiniConst|InsertUAVBarrier" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: three variants are declared, mode macros are emitted, stable-plane bindings exist, mini constants are bound, and UAV barrier helper exists.

- [ ] **Step 10: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build exits 0. If BUILD/FILL shaders fail, fix shader compile errors before moving to orchestration.

- [ ] **Step 11: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git commit -m "feat(rtxpt): create pathtrace ray tracing variants"
```

### Task 6: Add `VBufferExport` Timing Anchor

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/ExportVisibilityBuffer.csh`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Read: `D:/RTXPT-fork/Rtxpt/ProcessingPasses/ExportVisibilityBuffer.hlsl`
- Read: RTXPT-fork memory `VBufferExport`

- [ ] **Step 1: Create debug-only `ExportVisibilityBuffer.csh`**

Create shader:

```hlsl
#include "PathTracer/PathTracerShared.h"
#include "PathTracer/PathTracerBridge.hlsli"

#ifndef NUM_COMPUTE_THREADS_PER_DIM
#    define NUM_COMPUTE_THREADS_PER_DIM 8
#endif

[numthreads(NUM_COMPUTE_THREADS_PER_DIM, NUM_COMPUTE_THREADS_PER_DIM, 1)]
void main(uint3 tid : SV_DispatchThreadID)
{
    const uint2 pixel = tid.xy;
    if (pixel.x >= g_Const.ptConsts.imageWidth || pixel.y >= g_Const.ptConsts.imageHeight)
        return;

    // RTXPT-fork's current live VBufferExport only preserves debug visualization.
    // The actual depth/motion/throughput export is done in BUILD_STABLE_PLANES.
}
```

Do not copy the upstream disabled `#if 0` VBuffer export body into live code.

- [ ] **Step 2: Add C++ pass class**

`RTXPTVBufferExportPass.hpp`:

```cpp
#pragma once

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderResourceBinding.h"

namespace Diligent
{

class RTXPTVBufferExportPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, IBuffer* pFrameConstants, IBuffer* pMiniConstants);
    bool Dispatch(IDeviceContext* pContext, Uint32 Width, Uint32 Height);
    bool IsReady() const { return m_PSO != nullptr && m_SRB != nullptr; }

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
};

} // namespace Diligent
```

Implementation follows existing `RTXPTComputePass` style:

```cpp
ShaderCI.FilePath   = "PathTracer/ExportVisibilityBuffer.csh";
ShaderCI.EntryPoint = "main";
```

Resource layout includes static `g_Const` and `g_MiniConst`.

- [ ] **Step 3: Register files**

In `CMakeLists.txt`, add:

```cmake
    src/RTXPTVBufferExportPass.cpp
```

to `SOURCE`, add:

```cmake
    src/RTXPTVBufferExportPass.hpp
```

to `INCLUDE`, and add:

```cmake
    assets/shaders/PathTracer/ExportVisibilityBuffer.csh
```

to `SHADERS`.

- [ ] **Step 4: Verify**

Run:

```powershell
rg -n "RTXPTVBufferExportPass|ExportVisibilityBuffer|NUM_COMPUTE_THREADS_PER_DIM" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: pass class, shader, and CMake entries exist.

- [ ] **Step 5: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.hpp DiligentSamples/Samples/RTXPT/src/RTXPTVBufferExportPass.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/ExportVisibilityBuffer.csh DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add realtime vbuffer export pass"
```

### Task 7: Implement Diligent `PathTrace` Orchestration

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp:2438-2575`
- Read: RTXPT-fork memories `PathTrace`, `PathTracePrePass`, `VBufferExport`

- [ ] **Step 1: Add sample members and helpers**

In `RTXPTSample.hpp`, include the VBuffer pass:

```cpp
#include "RTXPTVBufferExportPass.hpp"
```

Add members near `m_RayTracingPass`:

```cpp
RTXPTVBufferExportPass m_VBufferExportPass;
bool                   m_LastRealtimePathTraceExecuted = false;
bool                   m_LastRealtimeFinalMergeReady    = false;
std::string            m_RealtimePathTraceStatus;
```

Add private helper declarations:

```cpp
bool PathTrace();
bool DispatchPathTracePrePass(const RTXPTRayTracingDispatch& BaseDispatch);
bool DispatchPathTraceLoop(bool UseStablePlanes, const RTXPTRayTracingDispatch& BaseDispatch);
bool RunRealtimePathTraceOnly();
bool RunReferencePathTraceAndPostProcess();
void RecordRealtimePathTraceStatus(const char* Status);
```

- [ ] **Step 2: Initialize VBuffer pass with frame and mini constants**

Where phase passes are created, initialize:

```cpp
m_VBufferExportPass.Initialize(m_pDevice, m_pEngineFactory, m_FrameConstantsCB, m_RayTracingPass.GetMiniConstantsBuffer());
```

If `RTXPTRayTracingPass` keeps `m_MiniConstantsCB` private, expose:

```cpp
IBuffer* GetMiniConstantsBuffer() const { return m_MiniConstantsCB; }
```

- [ ] **Step 3: Extend `LightsBaker::UpdateEnd` signature**

Change declaration:

```cpp
bool UpdateEnd(IDeviceContext* pContext, ITextureView* pDepthSRV, ITextureView* pMotionVectorsSRV);
```

Implementation:

```cpp
bool RTXPTLightsBaker::UpdateEnd(IDeviceContext* pContext,
                                 ITextureView*   pDepthSRV,
                                 ITextureView*   pMotionVectorsSRV)
{
    if (!m_Stats.Ready)
        return false;

    // Current Diligent LightsBaker only clears/processes feedback resources.
    // Depth and motion-vector views are accepted here to preserve the RTXPT-fork
    // PathTrace call contract for future NEE-AT feedback passes.
    (void)pDepthSRV;
    (void)pMotionVectorsSRV;

    // Existing UpdateEnd body follows unchanged.
}
```

Update all existing call sites:

```cpp
m_LightsBaker.UpdateEnd(m_pImmediateContext,
                        m_RenderTargets.GetDepthSRV(),
                        m_RenderTargets.GetScreenMotionVectorsSRV())
```

- [ ] **Step 4: Build base dispatch object**

In `RTXPTSample.cpp`, add:

```cpp
RTXPTRayTracingDispatch MakePathTraceDispatch(const RTXPTRenderTargets& RenderTargets,
                                              const SampleMiniConstants& MiniConstants)
{
    RTXPTRayTracingDispatch Dispatch;
    Dispatch.pOutputColorUAV        = RenderTargets.GetOutputColorUAV();
    Dispatch.pDepthUAV              = RenderTargets.GetDepthUAV();
    Dispatch.pMotionVectorsUAV      = RenderTargets.GetScreenMotionVectorsUAV();
    Dispatch.pThroughputUAV         = RenderTargets.GetThroughputUAV();
    Dispatch.pSpecularHitTUAV       = RenderTargets.GetSpecularHitTUAV();
    Dispatch.pStableRadianceUAV     = RenderTargets.GetStableRadianceUAV();
    Dispatch.pStablePlanesHeaderUAV = RenderTargets.GetStablePlanesHeaderUAV();
    Dispatch.pStablePlanesBufferUAV = RenderTargets.GetStablePlanesBufferUAV();
    Dispatch.pMiniConstants         = &MiniConstants;
    Dispatch.Width                  = RenderTargets.GetRenderWidth();
    Dispatch.Height                 = RenderTargets.GetRenderHeight();
    return Dispatch;
}
```

- [ ] **Step 5: Implement pre-pass order**

Add:

```cpp
bool RTXPTSample::DispatchPathTracePrePass(const RTXPTRayTracingDispatch& BaseDispatch)
{
    if (!m_RenderTargets.HasRealtimeRenderTargets())
    {
        RecordRealtimePathTraceStatus("Realtime render targets are not allocated");
        return false;
    }

    const bool PrePassOk =
        m_RayTracingPass.Dispatch(m_pImmediateContext,
                                  RTXPTPathTraceVariant::BuildStablePlanes,
                                  BaseDispatch);
    if (!PrePassOk)
    {
        RecordRealtimePathTraceStatus("BUILD_STABLE_PLANES dispatch failed");
        return false;
    }

    RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetStablePlanesBuffer());
    RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetDepthUAV());
    RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetScreenMotionVectorsUAV());
    RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetThroughputUAV());

    const bool VBufferOk =
        m_VBufferExportPass.Dispatch(m_pImmediateContext,
                                     m_RenderTargets.GetRenderWidth(),
                                     m_RenderTargets.GetRenderHeight());
    if (!VBufferOk)
    {
        RecordRealtimePathTraceStatus("VBufferExport dispatch failed");
        return false;
    }

    return true;
}
```

- [ ] **Step 6: Implement main PathTrace loop**

Add:

```cpp
bool RTXPTSample::DispatchPathTraceLoop(bool UseStablePlanes, const RTXPTRayTracingDispatch& BaseDispatch)
{
    const Uint32 SPP = std::max(m_RealtimeUI.ActualSamplesPerPixel(), 1u);
    const RTXPTPathTraceVariant Variant =
        UseStablePlanes ? RTXPTPathTraceVariant::FillStablePlanes : RTXPTPathTraceVariant::Reference;

    for (Uint32 SubSampleIndex = 0; SubSampleIndex < SPP; ++SubSampleIndex)
    {
        if (UseStablePlanes)
        {
            RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetStablePlanesBuffer());
            RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetSpecularHitTUAV());
        }

        SampleMiniConstants MiniConstants = {};
        MiniConstants.params.x = SubSampleIndex;

        RTXPTRayTracingDispatch Dispatch = BaseDispatch;
        Dispatch.pMiniConstants = &MiniConstants;

        const bool TraceOk = m_RayTracingPass.Dispatch(m_pImmediateContext, Variant, Dispatch);
        if (!TraceOk)
        {
            RecordRealtimePathTraceStatus(UseStablePlanes ? "FILL_STABLE_PLANES dispatch failed" : "REFERENCE dispatch failed");
            return false;
        }

        if (UseStablePlanes)
            RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetSpecularHitTUAV());
    }

    if (UseStablePlanes)
        RTXPTRayTracingPass::InsertUAVBarrier(m_pImmediateContext, m_RenderTargets.GetStablePlanesBuffer());

    return true;
}
```

Reference mode uses `SPP = 1` because `RTXPTRealtimeSettings::ActualSamplesPerPixel()` returns 1 when realtime mode is false.

- [ ] **Step 7: Implement `PathTrace()`**

Add:

```cpp
bool RTXPTSample::PathTrace()
{
    m_LastRealtimePathTraceExecuted = false;
    m_LastRealtimeFinalMergeReady   = false;
    RecordRealtimePathTraceStatus("");

    const bool UseStablePlanes = m_RealtimeUI.RealtimeMode;
    SampleMiniConstants MiniConstants = {};
    const RTXPTRayTracingDispatch BaseDispatch = MakePathTraceDispatch(m_RenderTargets, MiniConstants);

    if (UseStablePlanes)
    {
        if (!DispatchPathTracePrePass(BaseDispatch))
            return false;
    }

    const bool LightsOk =
        m_LightsBaker.UpdateEnd(m_pImmediateContext,
                                m_RenderTargets.GetDepthSRV(),
                                m_RenderTargets.GetScreenMotionVectorsSRV());
    if (!LightsOk)
    {
        RecordRealtimePathTraceStatus("LightsBaker.UpdateEnd failed");
        return false;
    }

    if (!DispatchPathTraceLoop(UseStablePlanes, BaseDispatch))
        return false;

    if (UseStablePlanes)
    {
        m_LastRealtimePathTraceExecuted = true;

        // RTXDI final shading and denoising guide bake are ordered here in RTXPT-fork.
        // RTXDI is intentionally disabled in this plan and reported in status UI.
        // G6 plugs DenoisingGuidesBaker here before denoise/final-merge phases.
        RecordRealtimePathTraceStatus("Realtime PathTrace BUILD/FILL completed; final merge is not part of G4/G5");
    }

    return true;
}
```

- [ ] **Step 8: Preserve RTXDI disabled hook**

Add status text in the realtime status UI:

```cpp
if (m_RealtimeUI.RealtimeMode)
{
    ImGui::Text("RTXDI/ReSTIR final hooks: disabled in this port phase");
}
```

Do not set `ptConsts.useReSTIRDI` or `ptConsts.useReSTIRGI` to 1 in G4/G5.

- [ ] **Step 9: Verify source order in code**

Run:

```powershell
rg -n "PathTracePrePass|BuildStablePlanes|VBufferExport|UpdateEnd|FillStablePlanes|SampleMiniConstants|SubSampleIndex|RTXDI|DenoisingGuides" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected order in `PathTrace()`:

```text
BUILD_STABLE_PLANES dispatch
VBufferExport dispatch
LightsBaker.UpdateEnd
FILL_STABLE_PLANES or REFERENCE sub-sample loop
RTXDI disabled status hook
DenoisingGuides future hook/status
```

- [ ] **Step 10: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.hpp DiligentSamples/Samples/RTXPT/src/RTXPTLightsBaker.cpp
git commit -m "feat(rtxpt): orchestrate realtime pathtrace passes"
```

### Task 8: Route `Render()` Through Reference or Realtime PathTrace Safely

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp`

- [ ] **Step 1: Extract current reference post-process flow**

Move the current reference trace + accumulation + pre-tone + tone-map + presentation body into:

```cpp
bool RTXPTSample::RunReferencePathTraceAndPostProcess()
{
    if (!PathTrace())
    {
        if (!m_RayTracingPass.IsReady(RTXPTPathTraceVariant::Reference))
            ClearFallback(float4{1.0f, 0.0f, 1.0f, 1.0f});
        else
            ClearFallback(float4{1.0f, 1.0f, 0.0f, 1.0f});
        return false;
    }

    // Existing accumulation, pre-tone mapping, tone mapping, and blit flow follows unchanged.
    return true;
}
```

The old direct call to `m_RayTracingPass.Trace(...)` is removed.

- [ ] **Step 2: Add realtime pathtrace-only route**

Add:

```cpp
bool RTXPTSample::RunRealtimePathTraceOnly()
{
    if (!m_RenderTargets.HasRealtimeRenderTargets())
    {
        RecordRealtimePathTraceStatus(m_RenderTargets.GetLastFailureReason());
        ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
        return false;
    }

    const bool PathTraceOk = PathTrace();
    if (!PathTraceOk)
    {
        ClearFallback(float4{0.9f, 0.15f, 0.05f, 1.0f});
        return false;
    }

    // FILL_STABLE_PLANES writes stable-plane radiance storage, not final OutputColor.
    // G7/G9 will replace this fallback with NoDenoiserFinalMerge or NRD final merge.
    ClearFallback(float4{0.08f, 0.08f, 0.10f, 1.0f});
    return true;
}
```

- [ ] **Step 3: Update `Render()` top-level branch**

After `EnsureRenderTargets()` succeeds, replace the realtime clear-and-return block with:

```cpp
if (m_RealtimeUI.RealtimeMode)
{
    RunRealtimePathTraceOnly();
    return;
}

RunReferencePathTraceAndPostProcess();
```

Reference mode must still execute post-process and presentation. Realtime mode must not run reference accumulation or present stale `OutputColor`.

- [ ] **Step 4: Add status lines**

In status UI:

```cpp
if (m_RealtimeUI.RealtimeMode)
{
    ImGui::Text("Realtime PathTrace: %s", m_LastRealtimePathTraceExecuted ? "BUILD/FILL dispatched" : "not dispatched");
    ImGui::Text("Realtime final merge: pending G7/G9");
    if (!m_RealtimePathTraceStatus.empty())
        ImGui::TextWrapped("Realtime PathTrace status: %s", m_RealtimePathTraceStatus.c_str());
}
```

- [ ] **Step 5: Verify realtime no longer exits before PathTrace**

Run:

```powershell
rg -n "RealtimeMode\\)|RunRealtimePathTraceOnly|RunReferencePathTraceAndPostProcess|PathTrace\\(|ClearFallback\\(float4\\{0\\.08f" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: realtime mode calls `RunRealtimePathTraceOnly()` and then returns; the old immediate clear-and-return block is gone.

- [ ] **Step 6: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): route realtime mode through pathtrace"
```

### Task 9: Update Mapping and Source Parity Notes

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md` only if implementation discovers a concrete correction to G4/G5 assumptions

- [ ] **Step 1: Add G4/G5 mapping rows**

Add a `Realtime G4-G5` section:

```markdown
## Realtime G4-G5 PathTrace Variants and Orchestration

| RTXPT-fork source | Diligent owner | Status |
|---|---|---|
| `AdvancedSample.cpp::CreateRTPipelines` REF/BUILD/FILL variants | `src/RTXPTRayTracingPass.{hpp,cpp}` `RTXPTPathTraceVariant` | Diligent RT PSO/SRB/SBT variants with `PATH_TRACER_MODE` macros. |
| `Sample.cpp::PathTrace` BUILD pre-pass | `src/RTXPTSample.cpp::DispatchPathTracePrePass` | Realtime-only dispatch to `BuildStablePlanes`, then UAV barriers. |
| `Sample.cpp::PathTrace` `VBufferExport` marker | `src/RTXPTVBufferExportPass.{hpp,cpp}`, `assets/shaders/PathTracer/ExportVisibilityBuffer.csh` | Debug/timing anchor only; primary export stays in BUILD pass as in current RTXPT-fork. |
| `Sample.cpp::PathTrace` `LightsBaker.UpdateEnd(... Depth, MotionVectors)` | `src/RTXPTLightsBaker::UpdateEnd(... pDepthSRV, pMotionVectorsSRV)` | Call-order and data contract preserved; current Diligent feedback implementation accepts but does not yet consume the views. |
| `Sample.cpp::PathTrace` FILL/REFERENCE sub-sample loop | `src/RTXPTSample.cpp::DispatchPathTraceLoop` | Uses `SampleMiniConstants.params.x` for sub-sample index. |
| `Shaders/PathTracer/PathPayload.hlsli` | `assets/shaders/PathTracer/PathPayload.hlsli` | Packed path-state payload for realtime variants. |
| `Shaders/PathTracer/PathState.hlsli` | `assets/shaders/PathTracer/PathState.hlsli` | Stable-plane flags/counters and path state. |
| `Shaders/PathTracer/StablePlanes.hlsli` | `assets/shaders/PathTracer/StablePlanes.hlsli` | Stable-plane buffer/header/radiance logic. |
| `Shaders/PathTracer/PathTracerStablePlanes.hlsli` | `assets/shaders/PathTracer/PathTracerStablePlanes.hlsli` | Build/fill stable-plane hit/miss/scatter logic. |
| `Sample.cpp::PathTrace` RTXDI final hooks | status UI only | Disabled in G4/G5; not silently treated as implemented. |
| `Sample.cpp::Denoise` | G8/G9 plans | Excluded from this plan. |
```

- [ ] **Step 2: Verify no broad realtime claims**

Run:

```powershell
rg -n "Realtime G4-G5|BuildStablePlanes|FillStablePlanes|Denoise|NRD|final merge|RTXDI" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: mapping explicitly states PathTrace is ported, RTXDI is disabled, and Denoise/final merge are out of this plan.

- [ ] **Step 3: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map realtime pathtrace port"
```

### Task 10: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Build RTXPT**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build exits 0. Shader compile errors for all three variants are fixed before continuing.

- [ ] **Step 2: Verify required G4 files and symbols**

Run:

```powershell
rg -n "PATH_TRACER_MODE_REFERENCE|PATH_TRACER_MODE_BUILD_STABLE_PLANES|PATH_TRACER_MODE_FILL_STABLE_PLANES|PathState|PathPayload|StablePlanesContext|CommitDenoiserRadiance|FirstHitFromVBuffer|RTXPTPathTraceVariant|BuildStablePlanes|FillStablePlanes" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: all G4 symbols are present in Diligent code.

- [ ] **Step 3: Verify required G5 order**

Run:

```powershell
rg -n "DispatchPathTracePrePass|VBufferExport|UpdateEnd|DispatchPathTraceLoop|SampleMiniConstants|SubSampleIndex|RunRealtimePathTraceOnly|RunReferencePathTraceAndPostProcess" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: `PathTrace()` order matches source: pre-pass, VBuffer export, lights update end, trace loop, disabled RTXDI/guide hooks.

- [ ] **Step 4: Verify realtime does not use reference raygen output as final color**

Run:

```powershell
rg -n "RunAccumulation|RunPreToneMapping|RunToneMapping|RunRealtimePathTraceOnly|FILL_STABLE_PLANES|final merge" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: reference mode runs accumulation/tone mapping; realtime G4/G5 pathtrace-only route does not run reference accumulation and reports final merge as pending.

- [ ] **Step 5: Smoke run on D3D12**

Run the sample from the existing build output using the local sample invocation pattern for this workspace. If the workspace has no runnable sample binary, record that manual smoke could not be executed and keep the build plus source checks as verification evidence.

Expected when runnable:

```text
Reference mode still renders through existing accumulation/tone mapping.
Realtime mode no longer exits before TraceRays; status reports BUILD/FILL dispatched when shader resources are valid.
Realtime mode displays the dark fallback because final merge is outside G4/G5.
```

- [ ] **Step 6: Inspect diff**

Run:

```powershell
git diff -- DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected:

```text
No unrelated files changed.
No DLSS-RR execution path added.
No standalone Denoise or NRD dispatch added.
Reference accumulation/tone mapping path remains present.
```

- [ ] **Step 7: Final commit if prior task commits were skipped**

If task-level commits were not made, commit the complete G4/G5 change:

```powershell
git add DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "feat(rtxpt): port realtime pathtrace orchestration"
```

## Self-Review

**Spec coverage**

- G4 `PATH_TRACER_MODE_REFERENCE`: Task 4 keeps reference mode isolated and Task 5 creates the reference variant.
- G4 `PATH_TRACER_MODE_BUILD_STABLE_PLANES`: Tasks 3-5 port stable-plane state and create the BUILD variant.
- G4 `PATH_TRACER_MODE_FILL_STABLE_PLANES`: Tasks 3-5 port fill state, `FirstHitFromVBuffer`, and FILL dispatch.
- G4 shader layers: Task 3 creates `PathState.hlsli`, `PathPayload.hlsli`, `PathTracerTypes.hlsli`, `StablePlanes.hlsli`, and `PathTracerStablePlanes.hlsli`.
- G4 Diligent bridge equivalents: Task 3 adds resource names and bridge helper functions.
- G5 `PathTracePrePass`: Task 7 implements BUILD pre-pass and UAV barriers.
- G5 `VBufferExport`: Task 6 creates the pass and Task 7 dispatches it after BUILD.
- G5 `LightsBaker.UpdateEnd`: Task 7 preserves order and extends the Diligent signature to carry depth/motion views.
- G5 `PathTrace` sub-sample loop: Task 7 uses `SampleMiniConstants.params.x` for each sub-sample.
- G5 RTXDI hooks: Task 7 and Task 9 report disabled hooks explicitly.
- G5 denoising guide bake: Task 7 preserves the ordered hook/status, while actual guide pass bodies remain in G6 by spec.

**Placeholder scan**

- No task leaves an unnamed file/function or asks the implementer to invent missing content.
- Denoise, NRD, final merge, DLSS-RR, and RTXDI execution are explicitly out of scope rather than vague future work.
- Where source-sized shader bodies are too large for this document, the plan names exact source files, required symbols, field orders, and destination files.

**Type consistency**

- `SampleMiniConstants` is 64 bytes in C++ and HLSL.
- `RTXPTPathTraceVariant` values match `PATH_TRACER_MODE` macro values.
- `RTXPTMaterialHitPayload` is the reference payload; `PathPayload` is reserved for realtime packed path state.
- `RTXPTRayTracingDispatch` names match `RTXPTRenderTargets` accessor names from G3 and HLSL resource names from Task 3.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-03-rtxpt-realtime-g4-g5-pathtrace-pipeline-variants-and-orchestration.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

This plan is intentionally scoped to G4/G5. It should be followed by separate G6/G7/G8/G9 plans before realtime mode presents a final denoised or no-denoiser image.
