# RTXPT Realtime PathTrace and Denoise Port Spec

## Summary

This spec defines the work required to port the original RTXPT-fork realtime path-tracing flow into
`DiligentSamples/Samples/RTXPT`, with the same observable behavior as RTXPT-fork for realtime
`PathTrace` and standalone `Denoise`.

The target source is `D:/RTXPT-fork/Rtxpt`. The central realtime call chain is:

```text
AdvancedPathTracer::SampleRenderCode
  -> optional RTXDI BeginFrame
  -> Sample::PathTrace
  -> Sample::Denoise

Sample::RenderScene frame flow then continues:
  -> Sample::PostProcessAA
  -> PostProcessPreToneMapping
  -> ToneMappingPass
  -> PostProcessPostToneMapping
  -> final blit
```

`RealtimeAA == 3` is the DLSS-RR path. That path is explicitly deferred in this spec: resource names
and TODO anchors should be reserved where they prevent rework, but DLSS-RR input preparation,
Streamline resource tagging, and `EvaluateDLSSRR` are not implemented by this phase. The standalone
NRD path is not deferred: REBLUR and RELAX integration must be wired when NRD is available, and the
sample must compile out cleanly with a visible disabled reason when NRD is not available.

## Source Anchors

Reference source files and functions:

- `D:/RTXPT-fork/Rtxpt/AdvancedSample.cpp`: `AdvancedPathTracer::SampleRenderCode`,
  `CreateRTPipelines`.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp`: `CreateRenderPasses`, `UpdatePathTracerConstants`,
  frame render orchestration around render-target recreation and binding-set rebuild,
  `PathTrace`, `Denoise`, `PostProcessAA`.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.{h,cpp}`: realtime render targets,
  stable-plane buffers, denoiser inputs/outputs, TAA/DLSS resources.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.{h,cpp,hlsl}`: denoiser prepare,
  denoiser final merge, no-denoiser final merge, stable-plane debug visualization,
  DLSS-RR prepare path.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/DenoisingGuidesBaker.{h,cpp,hlsl}`:
  specular-hit-distance smoothing, average layer radiance, denoiser guide debug views.
- `D:/RTXPT-fork/Rtxpt/NRD/NrdIntegration.{h,cpp}` and `NRD/NrdConfig.{h,cpp}`:
  NRD instance creation, pipeline creation, resource mapping, REBLUR/RELAX settings.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Config.h`: `PATH_TRACER_MODE_REFERENCE`,
  `PATH_TRACER_MODE_BUILD_STABLE_PLANES`, `PATH_TRACER_MODE_FILL_STABLE_PLANES`,
  `cStablePlaneCount`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`,
  `PathTracerStablePlanes.hlsli`, `PathTracer.hlsli`, `PathState.hlsli`,
  `PathPayload.hlsli`, `PathTracerTypes.hlsli`.
- `D:/RTXPT-fork/Rtxpt/SampleUI.{h,cpp}`: `RealtimeMode`, `RealtimeAA`,
  `ActualUseStandaloneDenoiser`, NRD settings, stable-plane controls, debug views.

Current Diligent port anchors:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}`: current reference-only
  orchestration, UI, accumulation reset, render-target update.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}`: HDR output,
  accumulation, post-process targets, depth, motion vectors, temporal feedback, super-resolution
  split.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}`: current raygen
  dispatch for the reference path.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}`:
  accumulation, pre-tone-mapping, tone mapping, super-resolution scheduling.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` and
  `PathTracerShared.h`: current reference raygen and frame constants.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`: existing source-to-port
  mapping and divergence rules.

## Current Diligent State

The Diligent RTXPT sample is currently a reference-path-tracing pipeline:

```text
RTXPTRayTracingPass::Trace
  -> RTXPTPostProcessPipeline::RunAccumulation
  -> RTXPTPostProcessPipeline::RunPreToneMapping
  -> RTXPTPostProcessPipeline::RunToneMapping
  -> RTXPTBlitPass
```

The current shader path writes raw HDR radiance, primary depth, and zero screen motion vectors. It
does not build stable planes, does not maintain the RTXPT-fork `PathState`/`PathPayload` realtime
state machine, does not export denoiser guide buffers, and does not run NRD. The UI exposes a
Reference/Realtime mode combo, but the realtime mode is marked out of scope. `PathTracerConstants`
contains the reference subset and lacks fields such as `sampleBaseIndex`, `perPixelJitterAAScale`,
`invSubSampleCount`, `texLODBias`, `preExposedGrayLuminance`, `denoisingEnabled`,
`_activeStablePlaneCount`, stable-plane stride fields, and NRD/DLSS-RR clamp fields.

The render-target graph already includes several Phase 6 post-processing resources: HDR
`OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, render/display dimensions,
`Depth`, `ScreenMotionVectors`, temporal feedback, and a super-resolution wrapper. It does not yet
include the stable-plane and NRD-specific resources required by realtime denoising.

## Confirmed Decisions

- Port the RTXPT-fork realtime behavior through Diligent-native owners, not by introducing a
  Donut/NVRHI compatibility layer.
- Preserve RTXPT-fork names and shader structure for the algorithm layer where this makes future
  upstream re-ports mechanical, while keeping Diligent sample lifecycle, Diligent object ownership,
  Apache headers, and local build style.
- Treat standalone NRD as a required target path. Compile-time or runtime capability gating is
  allowed only so unsupported configurations still build and report a clear disabled reason.
- Keep `RealtimeAA == 3` / DLSS-RR out of this implementation. Add only narrow TODO markers and
  resource reservations needed to avoid later churn.
- Keep reference mode behavior intact. Realtime mode must not regress reference accumulation,
  tone mapping, or presentation.

## Considered Approaches

Recommended approach: Diligent-native port of the original realtime pipeline. Add focused
Diligent pass classes for stable-plane tracing, denoising guides, post-process denoiser prepare/merge,
NRD, TAA/SR handoff, and realtime orchestration. Shader algorithms should be ported close to
RTXPT-fork, while CPU API calls are expressed through Diligent interfaces.

Alternative: Donut/NVRHI shim layer. This would make copying source code easier initially, but it
would duplicate engine abstractions inside a Diligent sample and create long-term maintenance debt.

Alternative: minimal realtime path without stable planes. This would produce a moving realtime image,
but it would not satisfy the 1:1 target because RTXPT-fork realtime denoising depends on stable planes,
denoising guide resources, final merge passes, and NRD.

## Goals

### G1 - Realtime State and UI Parity

Add a realtime settings model mirroring the RTXPT-fork controls that affect this flow:

- `RealtimeMode`, `RealtimeSamplesPerPixel`, `RealtimeAA`, `StandaloneDenoiser`.
- `ActualUseStandaloneDenoiser()`: true only when `RealtimeMode && RealtimeAA < 3 && StandaloneDenoiser`.
- `StablePlanesActiveCount`, `StablePlanesMaxVertexDepth`,
  `AllowPrimarySurfaceReplacement`, `StablePlanesSplitStopThreshold`,
  `StablePlanesSuppressPrimaryIndirectSpecular`, `StablePlanesAntiAliasingFallthrough`.
- realtime firefly filter controls, denoiser radiance clamp, reset realtime caches.
- NRD controls: mode `REBLUR`/`RELAX`, disocclusion thresholds, alternate disocclusion mix,
  REBLUR settings, RELAX settings.
- `RealtimeAA == 3` controls are visible only as disabled or TODO-bound controls.

Success criteria:

- Reference mode still defaults to the existing reference pipeline.
- Realtime mode selects the realtime path only when explicitly enabled.
- UI changes reset only the histories they invalidate: accumulation, realtime caches, NRD history,
  TAA/SR history, or render-target recreation.

### G2 - Realtime Frame Constants and Sample Indexing

Extend `SampleConstants` and `PathTracerConstants` to match the realtime fields consumed by the
source shaders:

- render dimensions: `imageWidth`, `imageHeight`.
- sample identity: `sampleBaseIndex`, `frameIndex`, `invSubSampleCount`.
- realtime jitter and texture LOD: `perPixelJitterAAScale`, `texLODBias`.
- exposure/denoiser values: `preExposedGrayLuminance`, `fireflyFilterThreshold`,
  `denoisingEnabled`, `denoiserRadianceClampK`.
- stable-plane controls and storage strides:
  `_activeStablePlaneCount`, `maxStablePlaneVertexDepth`,
  `allowPrimarySurfaceReplacement`, `stablePlanesSplitStopThreshold`,
  `stablePlanesSuppressPrimaryIndirectSpecularK`,
  `stablePlanesAntiAliasingFallthrough`, `genericTSLineStride`,
  `genericTSPlaneStride`.
- previous camera/view data required by motion vectors and NRD common settings.

For `RealtimeAA == 3`, keep `DLSSRRBrightnessClampK` and RR micro-jitter as
`TODO(RTXPT-Realtime-DLSS-RR)` if the fields are needed by shared layout, but do not execute the
DLSS-RR path.

Success criteria:

- C++ and HLSL shared layouts are synchronized with `static_assert`s.
- Realtime `sampleBaseIndex` follows RTXPT-fork semantics:
  `m_sampleIndex * ActualSamplesPerPixel()`.
- Realtime resets do not reuse reference accumulation sample indices.

### G3 - Realtime Render Targets

Extend `RTXPTRenderTargets` with the RTXPT-fork realtime/denoiser resources:

| Resource | Format target | Size | Purpose |
|---|---|---|---|
| `StableRadiance` | `RGBA16_FLOAT` | render | stable emissive/sky radiance |
| `StablePlanesHeader` | `R32_UINT`, array size 4 | render | branch IDs and first-hit/dominant index |
| `StablePlanesBuffer` | structured `StablePlane` | generic TS storage | stable-plane payload data |
| `Throughput` | `R32_UINT` | render | packed throughput helper |
| `SpecularHitT` | `R32_FLOAT` | render | specular hit distance guide |
| `ScratchFloat1` | `R32_FLOAT` | render | ping-pong for guide filtering |
| `DenoiserViewspaceZ` | `R32_FLOAT` | render | NRD `IN_VIEWZ` |
| `DenoiserMotionVectors` | `RGBA16_FLOAT` or closest supported | render | NRD `IN_MV` |
| `DenoiserNormalRoughness` | packed normal/roughness compatible with NRD | render | NRD `IN_NORMAL_ROUGHNESS` |
| `DenoiserDiffRadianceHitDist` | `RGBA16_FLOAT` | render | NRD diffuse input |
| `DenoiserSpecRadianceHitDist` | `RGBA16_FLOAT` | render | NRD specular input |
| `DenoiserDisocclusionThresholdMix` | `R8_UNORM` | render | NRD disocclusion mix |
| `DenoiserOutDiffRadianceHitDist[3]` | `RGBA16_FLOAT` | render | per-plane NRD diffuse output |
| `DenoiserOutSpecRadianceHitDist[3]` | `RGBA16_FLOAT` | render | per-plane NRD specular output |
| `DenoiserOutValidation` | `RGBA8_UNORM` optional | render | NRD validation debug output |
| `DenoiserAvgLayerRadianceHalfRes` | `RGBA16_FLOAT` | half render | average layer radiance/debug/RR guide |

DLSS-RR-specific guide resources (`RRDiffuseAlbedo`, `RRSpecAlbedo`, `RRNormalsAndRoughness`,
`RRSpecMotionVectors`, `RRTransparencyLayer`) should be reserved only if they are cheap and useful for
future work. They must be marked `TODO(RTXPT-Realtime-DLSS-RR)` and not required by the NRD path.

Success criteria:

- Resize/recreate logic releases NRD instances and realtime resources when render size changes.
- Resources have SRV/UAV accessors where required by post-process and NRD.
- Unsupported formats fail closed with a clear status message instead of presenting stale data.

### G4 - Realtime Ray-Tracing Pipeline Variants

Port the RTXPT-fork realtime variants:

- `PATH_TRACER_MODE_REFERENCE` remains the current reference path.
- `PATH_TRACER_MODE_BUILD_STABLE_PLANES` builds stable planes and stable radiance.
- `PATH_TRACER_MODE_FILL_STABLE_PLANES` runs noisy path tracing, tracks stable branches, and commits
  denoiser radiance into stable-plane storage.

Required shader layers:

- `PathState.hlsli`, `PathPayload.hlsli`, `PathTracerTypes.hlsli`.
- `StablePlanes.hlsli`, `PathTracerStablePlanes.hlsli`.
- realtime branches inside `PathTracer.hlsli` for stable-plane build/fill, miss handling,
  scatter handling, `CommitDenoiserRadiance`, and specular-hit-distance capture.
- Diligent bridge equivalents for resource bindings currently provided by
  `ShaderResourceBindings.hlsli` and `PathTracerBridgeDonut.hlsli`.

Success criteria:

- `RTXPTRayTracingPass` or a dedicated realtime pass can create and dispatch all three variants.
- `MaxPayloadSize`, SBT/hit-group configuration, and shader macros are correct for each variant.
- Reference mode output is unchanged when realtime mode is disabled.

### G5 - `PathTrace` Orchestration

Implement the Diligent equivalent of `Sample::PathTrace`:

```text
if RealtimeMode:
  PathTracePrePass:
    dispatch BUILD_STABLE_PLANES
    write Depth, ScreenMotionVectors, Throughput, SpecularHitT, StableRadiance,
    StablePlanesHeader, StablePlanesBuffer
  VBufferExport:
    export visibility/stable-plane data required by downstream passes

LightsBaker.UpdateEnd(...)

PathTrace:
  dispatch FILL_STABLE_PLANES when realtime, REFERENCE when reference
  loop subSampleIndex in ActualSamplesPerPixel()

optional RTXDI:
  Execute DI/GI/fused final hooks when the RTXDI path is available and enabled

Denoising Guides Bake:
  DenoiseSpecHitT
  ComputeAvgLayerRadiance
  optional debug visualization

optional StablePlanesDebugViz
```

RTXDI/ReSTIR is part of the original `PathTrace` body. If RTXDI is not implemented in the current
phase, the realtime orchestration must keep the hook and report it as disabled, not silently pretend
strict RTXPT-fork parity for RTXDI-enabled settings.

Success criteria:

- Realtime mode produces `OutputColor` only through stable-plane merge/denoise/no-denoise paths,
  not by the reference raygen.
- `LightsBaker.UpdateEnd` sees current `Depth` and `ScreenMotionVectors`.
- Multiple realtime samples per frame are separated through push constants or an equivalent
  Diligent constant path matching `SampleMiniConstants`.

### G6 - Denoising Guides Baker

Port `DenoisingGuidesBaker` into Diligent-native compute passes:

- `DenoiseSpecHitT`: ping-pong filter `SpecularHitT` with `ScratchFloat1`.
- `ComputeAvgLayerRadiance`: write `DenoiserAvgLayerRadianceHalfRes`.
- `RenderDebugViz`: support denoiser guide debug views when debug visualization is enabled.

Success criteria:

- Guide passes run after realtime `PathTrace` and before standalone NRD or no-denoiser final merge.
- Dispatch dimensions match render size or half render size exactly as in RTXPT-fork.
- Guide passes share the same frame constants and stable-plane bindings as denoiser prepare/merge.

### G7 - PostProcess Denoiser Prepare and Final Merge

Port the required compute variants from `ProcessingPasses/PostProcess.hlsl`:

- `RELAXDenoiserPrepareInputs`.
- `REBLURDenoiserPrepareInputs`.
- `RELAXDenoiserFinalMerge`.
- `REBLURDenoiserFinalMerge`.
- `NoDenoiserFinalMerge`.
- `StablePlanesDebugViz`.

`DLSSRRDenoiserPrepareInputs` is not implemented here. Leave a narrow
`TODO(RTXPT-Realtime-DLSS-RR)` marker that names the deferred path.

NRD prepare input behavior:

- Iterate the selected stable plane through mini constants.
- Initialize `OutputColor` with `StableRadiance` on the first processed plane.
- Write `DenoiserViewspaceZ`, `DenoiserMotionVectors`, `DenoiserNormalRoughness`,
  `DenoiserDiffRadianceHitDist`, `DenoiserSpecRadianceHitDist`,
  `DenoiserDisocclusionThresholdMix`, and `CombinedHistoryClampRelax`.
- Pack RELAX inputs with RELAX front-end helpers and REBLUR inputs with REBLUR front-end helpers.
- Mark sky/no-surface pixels with the same view-Z sentinel behavior.

Final merge behavior:

- Read per-plane NRD diffuse/spec outputs.
- Remodulate with the stable-plane BSDF estimates through `DenoiserNRD::PostDenoiseProcess`.
- Add merged radiance into the input/output work texture.
- Support validation/debug output when the validation resource exists.

No-denoiser behavior:

- Combine all available stable-plane radiance and stable radiance into `OutputColor`.
- Feed `OutputColor` into the same downstream AA/SR/tone-mapping chain as the NRD path.

Success criteria:

- Standalone denoiser off still produces a realtime image through `NoDenoiserFinalMerge`.
- Standalone denoiser on produces a realtime image through prepare -> NRD -> final merge.
- The merge result is the only producer consumed by AA/SR for realtime mode.

### G8 - NRD Integration

Add a Diligent-native `RTXPTNrdIntegration` layer equivalent to `NrdIntegration`:

- Create an NRD instance with `REBLUR_DIFFUSE_SPECULAR` or `RELAX_DIFFUSE_SPECULAR`.
- Create per-pipeline compute shaders, Diligent PSOs, samplers, permanent pool textures,
  transient pool textures, and a volatile constant buffer sized from NRD descriptors.
- Convert NRD texture formats to Diligent `TEXTURE_FORMAT`.
- Map NRD resources to `RTXPTRenderTargets`:
  - `IN_MV` -> `DenoiserMotionVectors`
  - `IN_NORMAL_ROUGHNESS` -> `DenoiserNormalRoughness`
  - `IN_VIEWZ` -> `DenoiserViewspaceZ`
  - `IN_SPEC_RADIANCE_HITDIST` -> `DenoiserSpecRadianceHitDist`
  - `IN_DIFF_RADIANCE_HITDIST` -> `DenoiserDiffRadianceHitDist`
  - `IN_DISOCCLUSION_THRESHOLD_MIX` -> `DenoiserDisocclusionThresholdMix`
  - `OUT_SPEC_RADIANCE_HITDIST` -> `DenoiserOutSpecRadianceHitDist[plane]`
  - `OUT_DIFF_RADIANCE_HITDIST` -> `DenoiserOutDiffRadianceHitDist[plane]`
  - `OUT_VALIDATION` -> `DenoiserOutValidation`
  - permanent/transient pool -> integration-owned textures
- Populate NRD common settings from current and previous camera/view:
  world/view/projection matrices, jitter, previous jitter, motion-vector scale,
  frame index, disocclusion thresholds, denoising range, validation flag,
  fixed deterministic time delta for no-window captures, and accumulation mode reset.

Dependency policy:

- Add a CMake/configuration gate for NRD. The sample must build without NRD.
- When NRD is unavailable, the UI must disable standalone denoiser with a clear reason and realtime
  must fall back to `NoDenoiserFinalMerge`.
- When NRD is available, both RELAX and REBLUR paths must initialize and run.

Success criteria:

- Switching RELAX/REBLUR destroys stale NRD instances and recreates passes.
- Render-target resize destroys stale NRD instances.
- Reset realtime caches clears NRD history through NRD accumulation mode.
- NRD dispatches consume only valid Diligent texture views and report unsupported formats early.

### G9 - Realtime `Denoise` Orchestration

Implement the Diligent equivalent of `Sample::Denoise`:

```text
if !ActualUseStandaloneDenoiser():
  return

ensure RTXPTNrdIntegration[0..cStablePlaneCount-1]
select prepare/merge pass by NRD method
resetHistory = ResetRealtimeCaches
initWithStableRadiance = true

for pass = min(StablePlanesActiveCount, cStablePlaneCount) - 1 down to 0:
  PrepareInputs(pass, initWithStableRadiance)
  initWithStableRadiance = false
  RunNrdDenoiserPasses(pass, method settings, resetHistory)
  MergeOutputs(pass)
```

Success criteria:

- Per-plane processing order matches RTXPT-fork: highest active plane down to 0.
- Stable radiance is initialized once, on the first processed plane.
- RELAX and REBLUR select their own prepare/merge variants.
- NRD is never called when `RealtimeAA == 3`.

### G10 - Realtime AA/SR Handoff, Excluding DLSS-RR

Port the post-denoise realtime output handoff:

- `RealtimeAA == 0`: copy or pass `OutputColor` to `ProcessedOutputColor`.
- `RealtimeAA == 1`: use DiligentFX `TemporalAntiAliasing` or a dedicated RTXPT temporal AA pass
  to resolve `OutputColor` into `ProcessedOutputColor` using depth and motion vectors.
- `RealtimeAA == 2`: use the existing `RTXPTSuperResolutionPass` / Diligent `ISuperResolution`
  temporal upscaling path where available, with `OutputColor`, `Depth`, and `ScreenMotionVectors`
  as inputs and `ProcessedOutputColor` as output.
- `RealtimeAA == 3`: leave as `TODO(RTXPT-Realtime-DLSS-RR)`.

For `RealtimeAA == 1` and `RealtimeAA == 2`, camera jitter, previous view state, depth, and motion
vectors must be valid before the path trace dispatch that feeds them.

Success criteria:

- Realtime mode always produces `ProcessedOutputColor` before HDR post-process and tone mapping.
- Standalone NRD may run before TAA/SR/DLSS, matching RTXPT-fork.
- DLSS-RR is visibly unavailable and cannot be selected as an executing path.

## Non-Goals

- Implementing DLSS-RR / Streamline `RealtimeAA == 3`.
- Replacing the existing reference accumulation/tone-mapping path.
- Porting Donut or NVRHI as an engine layer.
- Shipping RTXDI/ReSTIR DI/GI in this spec unless scheduled as an explicit sub-plan. The realtime
  path must preserve hooks and disabled-state reporting for RTXDI-enabled settings.
- SER, OMM, Reflex, DLSS Frame Generation, external photo-mode denoisers, or screenshot tooling.

## Phase Plan

### Phase RT0: Contract Audit and Mapping

- Update `RTXPT_FORK_MAPPING.md` with realtime source-to-destination rows for stable planes,
  denoising guides, NRD, post-process denoiser passes, and realtime AA/SR handoff.
- Add structured TODO markers for only the deferred DLSS-RR path and any intentionally gated RTXDI
  subpath.
- Decide NRD dependency location and build gate.

Acceptance:

- Every original source file listed in this spec has a Diligent owner or explicit gated status.
- No broad "realtime TODO" remains; TODO markers name a concrete deferred path.

### Phase RT1: Settings, Constants, and Render Targets

- Add realtime UI/settings and frame constants.
- Add realtime render targets and accessors.
- Add resize/reset behavior and visible capability status.

Acceptance:

- Sample still renders reference mode.
- Realtime mode can allocate all non-DLSS-RR resources or explain why it cannot.

### Phase RT2: Stable-Plane Shader and RT Pipeline Variants

- Port stable-plane shader headers and realtime `PathTracer.hlsli` branches.
- Add BUILD/FILL ray-tracing variants.
- Wire stable-plane UAV/SRV bindings.

Acceptance:

- Realtime can run BUILD and FILL dispatches without NRD enabled.
- `NoDenoiserFinalMerge` can later consume valid stable-plane data.

### Phase RT3: Denoising Guides and No-Denoiser Final Merge

- Port `DenoisingGuidesBaker`.
- Port `StablePlanesDebugViz` and `NoDenoiserFinalMerge`.
- Route standalone-denoiser-off realtime frames through `OutputColor -> ProcessedOutputColor`.

Acceptance:

- Realtime mode renders without NRD.
- Debug/status UI can report stable-plane and guide resource availability.

### Phase RT4: NRD Prepare, Integration, and Final Merge

- Port REBLUR/RELAX prepare and final merge compute variants.
- Add `RTXPTNrdIntegration`.
- Wire `Denoise` orchestration and NRD reset/recreate behavior.

Acceptance:

- With NRD available, standalone denoiser produces denoised realtime output for REBLUR and RELAX.
- Without NRD, the sample compiles and uses no-denoiser final merge with a visible disabled reason.

### Phase RT5: Realtime AA/SR Handoff

- Implement `RealtimeAA == 0`, `1`, and `2` output handoff.
- Keep `RealtimeAA == 3` disabled with `TODO(RTXPT-Realtime-DLSS-RR)`.

Acceptance:

- Realtime output reaches the existing HDR post-process, tone mapping, LDR post-process, and blit.
- TAA/SR resets and previous-view state behave correctly after camera, resize, and cache resets.

## Cross-Cutting Contracts

- `OutputColor` is raw or merged realtime HDR radiance. It is never tone-mapped.
- `ProcessedOutputColor` is the HDR post-AA/SR result consumed by existing pre-tone mapping and
  tone mapping.
- Stable-plane resources are render-size resources; post-AA/SR output is display-size when
  super-resolution is active.
- NRD prepare inputs are render-size resources.
- Previous view/camera data must be updated after each successful frame and invalidated on resize,
  mode change, shader reload, or scene reset.
- Realtime cache reset must reset stable planes, denoising guides, NRD history, TAA/SR history,
  and any RTXDI feedback that is active.
- C++/HLSL resource names should stay close to RTXPT-fork (`u_DenoiserViewspaceZ`,
  `u_DenoiserMotionVectors`, `u_DenoiserNormalRoughness`, etc.) even if binding slots differ.
- Backend-specific or external-library paths must compile out cleanly on unsupported backends.

## Verification Strategy

Per phase:

1. Build the RTXPT sample target after code changes that affect C++ or HLSL.
2. Run D3D12 smoke tests for reference and realtime modes.
3. Run Vulkan smoke tests for reference and any realtime subset that does not depend on unavailable
   backend features.
4. Verify reference mode image behavior before/after RT phases that touch shared shaders or constants.
5. For realtime without NRD, verify `NoDenoiserFinalMerge` produces a valid HDR image and presentation.
6. For realtime with NRD, verify REBLUR and RELAX both initialize, dispatch, and produce output.
7. Toggle `StandaloneDenoiser`, `RealtimeAA`, `StablePlanesActiveCount`, NRD method, resize, camera move,
   and reset realtime caches; verify the correct histories reset.
8. Source-scan for accidental DLSS-RR execution. `RealtimeAA == 3` may appear only in
   `TODO(RTXPT-Realtime-DLSS-RR)` guards or disabled UI/status code.
9. Compare a fixed scene against RTXPT-fork for pass ordering and resource debug views:
   stable radiance, stable-plane radiance, denoiser guide depth/roughness/albedo/normal/motion,
   NRD diffuse/spec outputs, and final merged HDR.

## Open Questions For Implementation Planning

- NRD source/dependency placement: add an external package/submodule, use an existing installed SDK,
  or vendor the minimum NRD integration into the sample build.
- RTXDI scheduling: whether to implement RTXDI as part of the realtime plan or keep it behind an
  explicit disabled-state hook until a separate RTXDI spec.
- TAA path: whether to use DiligentFX `TemporalAntiAliasing` directly or wrap it in an RTXPT-specific
  pass for closer resource naming and reset semantics.

