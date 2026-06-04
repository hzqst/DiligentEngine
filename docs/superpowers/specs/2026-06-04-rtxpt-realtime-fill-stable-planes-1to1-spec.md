# RTXPT Realtime Fill Stable Planes 1:1 Parity Spec

## Summary

This spec defines the work required to bring the current Diligent RTXPT realtime
`PATH_TRACER_MODE_FILL_STABLE_PLANES` path to behavioral parity with
`D:/RTXPT-fork/Rtxpt`.

The immediate bug signal is that switching the Diligent sample to realtime mode
produces a mostly black image: sky and emissive surfaces remain visible, while
ordinary lit scene geometry becomes black. The root cause is not tone mapping or
material loading. The current Diligent realtime path has the stable-plane
resource skeleton, but it does not yet run the original `FILL_STABLE_PLANES`
direct-light NEE, BSDF scatter, and `StablePlanesOnScatter` continuation flow.

The target behavior is exact RTXPT-fork realtime path-tracer semantics:

```text
PathTracePrePass / BUILD_STABLE_PLANES
  -> writes stable planes, stable radiance, depth, motion, throughput

PathTrace / FILL_STABLE_PLANES
  -> starts from plane 0
  -> runs normal noisy path tracing after the stable plane
  -> evaluates direct lighting with NEE
  -> generates BSDF scatter rays
  -> calls StablePlanesOnScatter after scatter
  -> commits noisy radiance into StablePlane::PackedNoisyRadianceAndSpecAvg

NoDenoiserFinalMerge or NRD FinalMerge
  -> merges stable radiance + noisy layer radiance into OutputColor
```

## Source Anchors

Reference RTXPT-fork anchors:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerSample.hlsl`: raygen entry,
  `FirstHitFromVBuffer`, `nextHit`, `postProcessHit`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli`:
  `AccumulatePathRadiance`, `CommitPixel`, `HandleHit`, `HandleMiss`,
  `GenerateScatterRay`, direct-light `HandleNEE` integration, termination rules.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerStablePlanes.hlsli`:
  `StablePlanesHandleHit`, `StablePlanesOnScatter`, stable-branch tracking.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli`:
  direct-light next-event estimation and MIS details.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`:
  `CommitDenoiserRadiance`, `GetAllRadiance`, branch IDs and stable-plane
  storage contract.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightSampler.hlsli`:
  light candidate generation, RIS/WRS helpers, emissive and analytic light
  sampling.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli`:
  BSDF sampling, lobe IDs, sampled-lobe probability, mixture pdf.

Current Diligent anchors:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`:
  `DispatchPathTracePrePass`, `PathTrace`, `DispatchPathTraceLoop`,
  `RunRealtimeNoDenoiserFinalMerge`, `Denoise`, `RunRealtimePostProcess`.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}`:
  path-trace variants, shader macros, static and dynamic binding layout.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`:
  current reference raygen and realtime `FILL_STABLE_PLANES` raygen entry.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`:
  current realtime stable-plane hit/miss glue and temporary hit termination.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli`:
  current stable-plane storage and branch tracking helpers.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`:
  current stable-plane buffer/radiance accumulation contract.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`:
  no-denoiser and denoiser final merge paths.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`:
  naming and source-mapping rules for future upstream re-ports.

## Current Failure Mode

In Diligent realtime mode, `RTXPTSample::PathTrace` selects stable-plane tracing:

```text
RealtimeMode == true
  -> DispatchPathTracePrePass(BuildStablePlanes)
  -> DispatchPathTraceLoop(FillStablePlanes)
  -> RunRealtimePostProcess()
```

The build pass can accumulate stable emissive and environment radiance. The final
merge then outputs `StablePlanes.GetAllRadiance(PixelPos)`, so emissive surfaces
and sky remain visible.

Ordinary surfaces are black because their radiance is not self-emission. They
need direct-light NEE or BSDF-scattered continuation paths to produce lighting.
The current Diligent `FILL_STABLE_PLANES` path does not yet generate those
contributions:

- `PathTracer.hlsli` accumulates `surfaceEmission` and environment miss radiance.
- The hit path has a temporary termination block after stable-plane handling.
- `RTXPTRayTracingPass.cpp` explicitly treats light, feedback, and emissive
  resources as optional for realtime variants because current stable-plane
  variants do not run direct-light NEE.

This means realtime final merge has stable radiance but almost no noisy surface
lighting to merge.

## Confirmed Requirements

- Realtime `FILL_STABLE_PLANES` must be ported as the original RTXPT algorithm,
  not replaced with a reference-mode fallback or a single-bounce diffuse hack.
- Reference mode must keep its current behavior and output path.
- `BUILD_STABLE_PLANES` remains the stable-radiance and stable-plane construction
  pass. It must not start accumulating noisy NEE radiance.
- `FILL_STABLE_PLANES` must run NEE and BSDF scatter after loading the base
  stable plane, then deposit noisy radiance into stable-plane storage.
- `StablePlanesOnScatter` must be called after successful BSDF scatter in fill
  mode, matching RTXPT-fork ordering.
- Realtime variants must bind and use the light bridge resources required by NEE:
  `t_Lights`, `t_LightingControl`, `t_LightProxyCounters`,
  `t_LightSamplingProxies`, `t_LocalSamplingBuffer`,
  `u_FeedbackTotalWeight`, `u_FeedbackCandidates`, and
  `t_EmissiveTriangles` where reflected by the shader.
- The implementation must remain Diligent-native: no Donut/NVRHI compatibility
  layer is introduced.
- Shader structure and naming should stay close to RTXPT-fork where the ported
  algorithm has a direct source analog.
- D3D12 and Vulkan are both target backends for verification.

## Non-Goals

- DLSS-RR and Streamline integration.
- SER, OMM, NVAPI-specific shader extensions, or RTXPT-fork performance
  shortcuts that require unavailable platform features.
- A broad rewrite of the already-working reference path tracer.
- Replacing stable-plane realtime mode with reference path tracing presented
  through the realtime post-process path.
- Full RTXDI/ReSTIR final shading parity unless a small binding or helper is
  required so the direct-light NEE path has the same resource contract as
  RTXPT-fork.

## Design

### D1 - Split Reference Raygen From Stable-Plane Continuation Logic

Current Diligent has both a standalone reference raygen loop and a stable-plane
`PathState` loop. The parity target should keep the reference path intact, then
make the stable-plane path follow the RTXPT-fork state-machine flow.

`PATH_TRACER_MODE_REFERENCE` continues to:

- initialize camera ray state,
- run the existing reference loop,
- write `u_Output`/`u_OutputColor`,
- write primary depth and motion-vector guides.

`PATH_TRACER_MODE_FILL_STABLE_PLANES` must:

- initialize `PathState`,
- call `FirstHitFromVBuffer(path, 0, workingContext)`,
- loop with `nextHit(path, tMinMax, workingContext)` and `postProcessHit`,
- let closest-hit/miss handling advance `PathState`,
- call `PathTracer::CommitPixel(path, workingContext)` at the end.

The implementation should not duplicate the current reference raygen's
float-local path-radiance loop inside fill mode. Fill mode owns radiance through
`PathState::L` and `StablePlanesContext::CommitDenoiserRadiance`.

### D2 - Restore Hit Handling Order From RTXPT-fork

The Diligent `PathTracer::HandleHit` fill path must be reworked to match the
source ordering:

1. Update travelled distance and vertex index.
2. Apply volume transmittance and nested dielectric acceptance if active in the
   current port.
3. Accumulate emissive or analytic-light-proxy surface emission with MIS.
4. Determine `pathStopping = path.isTerminatingAtNextBounce()`.
5. Call `StablePlanesHandleHit(...)` before throughput/scatter updates.
6. If `pathStopping` or the stable-plane handler terminated the path, stop.
7. Apply Russian roulette correction where supported by the current port state.
8. Generate a BSDF scatter ray for non-build modes.
9. Evaluate direct-light NEE for non-build modes.
10. Accumulate NEE radiance with the same stable-plane/noisy-radiance rules as
    RTXPT-fork.
11. On successful scatter, call `StablePlanesOnScatter(path, bs, workingContext)`
    for fill mode.
12. Terminate only when scatter failed, the bounce limit is reached, or Russian
    roulette terminated the path.

The current temporary unconditional `path.terminate()` after the stable-plane
hit block is removed for fill mode. Build mode may still terminate or return
early after stable-plane construction as in RTXPT-fork.

### D3 - Port Direct-Light NEE Into PathTracer.hlsli

Realtime fill must use the same NEE helpers as reference where possible, with
RTXPT-fork-compatible function boundaries:

- `HandleNEE` or equivalent helper computes one direct-light contribution for
  the current surface before scatter continuation.
- Analytic light sampling uses the existing Diligent `LightSampler`/`PolymorphicLight`
  layer already ported for reference mode.
- Emissive-triangle NEE uses `t_EmissiveTriangles` and MIS against BSDF-hit
  emissive surfaces.
- Environment NEE uses the existing env-map sampler and importance map resources
  when available.
- Feedback resources are updated only where the current Diligent `LightsBaker`
  contract already supports the corresponding RTXPT-fork behavior.

Radiance accumulation in fill mode follows this invariant:

```text
if path is on a stable branch:
  skip stable radiance that BUILD_STABLE_PLANES already captured
else:
  accumulate noisy radiance into path.L scaled by invSubSampleCount
```

Specular average is preserved for denoiser input. If the first scatter after a
stable plane is diffuse, specular average is zero; otherwise it follows the
RTXPT-fork heuristic for first and early delta-only bounces.

### D4 - Restore BSDF Scatter Contract

The fill path needs a BSDF scatter object compatible with
`StablePlanesOnScatter`:

- sampled ray direction,
- throughput/weight multiplier,
- pdf,
- sampled lobe bitmask,
- sampled-lobe probability,
- delta-lobe index via `bs.getDeltaLobeIndex()` or an equivalent helper.

The current Diligent `SampleBSDF` returns `lobe` and `lobeP` in the reference
path. Fill mode must package that information into the stable-plane scatter
contract without changing reference-mode math.

Scatter updates must:

- update ray origin using the hit face normal for reflection/transmission
  side selection,
- multiply path throughput by the BSDF weight,
- carry nested dielectric state where currently supported,
- update diffuse-bounce counters,
- update firefly-filter state if the current realtime settings enable it,
- store MIS/scatter pdf information used by emissive/environment BSDF-hit MIS.

### D5 - Re-enable Light Bridge Binding For Realtime Variants

`RTXPTRayTracingPass::Initialize` currently allows realtime variants to omit
light bridge resources because DXC strips them from the incomplete shader. Once
fill mode runs NEE, realtime variants must bind those resources when reflected.

Binding policy:

- Reference variant: light bridge resources remain required.
- BuildStablePlanes variant: light bridge resources remain optional unless the
  shader reflects them through a stable-radiance or surface-emission path.
- FillStablePlanes variant: light bridge resources become required for resources
  used by direct-light NEE.
- Missing reflected resources fail initialization with a precise status message
  naming the variant and resource.

The stats shown in the ImGui debug panel should no longer report realtime
`LightBridgeBound` as fallback when fill mode actually reflects and binds the
resource set.

### D6 - Preserve Stable-Plane Output And Merge Contract

Fill mode does not write final color directly. It writes:

- noisy radiance and specular average into
  `StablePlane::PackedNoisyRadianceAndSpecAvg`,
- specular hit distance into `u_SpecularHitT` when the dominant denoising layer
  path reaches the same stop conditions as RTXPT-fork,
- guide data consumed by denoising guide bake and NRD prepare/final merge.

`NoDenoiserFinalMerge` remains a valid first acceptance target:

```text
OutputColor = StableRadiance + sum(valid StablePlane noisy radiance)
```

NRD prepare/final merge then consumes the same stable-plane noisy-radiance data.
The implementation must not special-case no-denoiser output in a way that hides
missing stable-plane noisy radiance from NRD.

## Implementation Phases

### Phase F1 - Source Diff And Contract Map

Create a focused implementation map from RTXPT-fork to Diligent for these
symbols:

- `PathTracer::HandleHit`
- `PathTracer::HandleMiss`
- `PathTracer::GenerateScatterRay`
- `PathTracer::HandleNEE`
- `PathTracer::AccumulatePathRadiance`
- `StablePlanesOnScatter`
- `StablePlanesHandleHit`
- `StablePlanesContext::CommitDenoiserRadiance`

The map records exact Diligent equivalents, intentional divergences, and files
to edit. This phase does not change behavior.

### Phase F2 - Fill-Mode Scatter State

Add or adapt the Diligent HLSL scatter data shape so fill mode can call
`StablePlanesOnScatter` with the same information RTXPT-fork expects. Preserve
the reference path's existing `SampleBSDF` call signatures unless a shared
helper can be introduced without changing output.

Acceptance:

- `PATH_TRACER_MODE_FILL_STABLE_PLANES` compiles with scatter state available.
- Reference mode renders unchanged.
- Fill mode still renders stable radiance even before NEE is restored.

### Phase F3 - Direct-Light NEE In Fill Mode

Port the direct-light NEE call into `PathTracer::HandleHit` for non-build modes.
Use the same light sampling, environment sampling, emissive-triangle MIS, and
firefly-filter inputs as the reference path where the current Diligent port
already has them.

Acceptance:

- A realtime scene lit by analytic lights shows non-emissive surfaces.
- Sky and emissive surfaces remain visible.
- No-denoiser final merge shows stable radiance plus noisy direct lighting.

### Phase F4 - BSDF Scatter Continuation

Replace the fill-mode temporary hit termination with BSDF scatter continuation.
After a successful scatter, update path state and call `StablePlanesOnScatter`.
Preserve build-mode early-return behavior.

Acceptance:

- Realtime mode can trace beyond the base stable plane.
- Secondary light contribution appears where bounce settings permit it.
- Stable-plane branch IDs remain valid and no-denoiser merge remains finite.

### Phase F5 - Realtime Light Bridge Binding

Update `RTXPTRayTracingPass` so `FillStablePlanes` binds the reflected light,
feedback, environment, and emissive resources required by NEE. Keep
`BuildStablePlanes` optional for resources it does not reflect.

Acceptance:

- Initialization fails clearly if a required reflected fill-mode resource is
  unavailable.
- ImGui ray-tracing stats report light bridge resources as bound when realtime
  fill uses them.
- D3D12 and Vulkan shader reflection both accept the binding layout.

### Phase F6 - Parity Validation And Regression Guards

Add focused validation scenes or test hooks for:

- emissive-only scene: sky/emissive still match the current visible path,
- analytic-light scene: ordinary geometry is not black in realtime mode,
- indirect bounce scene: secondary contribution appears with bounce count above
  one,
- reference mode: output path and accumulation remain unchanged,
- realtime no-denoiser final merge: finite HDR output with no NaNs or infinities.

Acceptance:

- D3D12 build and launch smoke pass.
- Vulkan build and launch smoke pass where available.
- Shader compile logs contain no missing reflected resource errors.
- Debug UI confirms `BuildStablePlanes` and `FillStablePlanes` dispatch counts
  increase in realtime mode.

## Validation Strategy

Minimum verification commands and checks for the implementation plan:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Manual smoke checks:

- Launch RTXPT in D3D12 realtime mode.
- Disable standalone denoiser and use no-denoiser final merge.
- Use an analytic-light scene with non-emissive geometry.
- Confirm non-emissive surfaces are lit.
- Toggle back to reference mode and confirm reference output still renders.
- Repeat on Vulkan if the local build has Vulkan RT support configured.

Shader-level debug checks:

- Enable stable-plane debug view and confirm valid plane occupancy.
- Inspect debug UI for `OutputColor`, `StableRadiance`, `StablePlanesHeader`,
  and `StablePlanesBuffer` availability.
- Confirm `LightBridgeBound`, `LightsBakerBridgeBound`, and
  `EmissiveLightBridgeBound` are not fallback in a realtime fill run that uses
  direct-light NEE.

Failure signatures to catch:

- ordinary geometry still black: NEE/scatter radiance is not reaching
  `PackedNoisyRadianceAndSpecAvg`;
- only primary direct light visible: scatter continuation or bounce counters are
  wrong;
- emissive/sky double-bright: fill mode is accumulating stable radiance already
  captured by build mode;
- speckled infinite output: firefly/radiance clamp, pdf, or lobe probability is
  invalid;
- shader reflection misses light resources: fill-mode macro path still strips
  NEE code or binding stages are incomplete.

## Risks And Mitigations

- Risk: direct copy of RTXPT-fork `HandleHit` conflicts with the Diligent
  reference raygen loop.
  Mitigation: port the stable-plane state-machine path separately and keep
  reference-mode behavior isolated.

- Risk: build mode starts using NEE resources accidentally.
  Mitigation: gate NEE and scatter code on
  `PATH_TRACER_MODE != PATH_TRACER_MODE_BUILD_STABLE_PLANES`.

- Risk: stable radiance is double-counted.
  Mitigation: keep the fill-mode `AccumulatePathRadiance` branch condition:
  accumulate noisy radiance only when `!stablePlaneOnBranch`.

- Risk: light bridge binding becomes too strict for variants where DXC strips a
  resource.
  Mitigation: make strictness variant-specific: required for reflected fill-mode
  NEE resources, optional for build resources not reflected.

- Risk: Vulkan and D3D12 reflect HLSL resources differently.
  Mitigation: use the existing `SetStaticForStages` found-any pattern, but make
  fill-mode required resources precise and log the variant/resource name.

- Risk: implementation accidentally turns realtime into reference path tracing.
  Mitigation: final merge must consume stable-plane storage, not direct
  reference output, and dispatch stats must show build and fill variants.

## Acceptance Criteria

The spec is satisfied when:

- realtime mode no longer renders ordinary lit geometry black;
- sky and emissive surfaces remain correct and are not double-counted;
- `FILL_STABLE_PLANES` performs direct-light NEE and BSDF scatter continuation;
- `StablePlanesOnScatter` is called after fill-mode scatter and stable branch
  tracking remains valid;
- fill-mode noisy radiance is committed through
  `StablePlanesContext::CommitDenoiserRadiance`;
- realtime variants bind and use the required light bridge resources;
- no-denoiser final merge produces a finite HDR image from stable-plane data;
- reference mode still renders through the reference path and accumulation;
- D3D12 and Vulkan shader creation do not report missing required resources.

## Relationship To Existing Specs And Plans

This spec narrows and strengthens the realtime path-tracing portion of
`docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`.
It should be implemented before treating NRD visual quality as meaningful,
because NRD can only denoise radiance that `FILL_STABLE_PLANES` actually
generates.

Existing plans under `docs/superpowers/plans/2026-06-03-rtxpt-realtime-g4-g5-pathtrace-pipeline-variants-and-orchestration.md`
and later realtime denoise plans should be revised or superseded if they assume
the current stable-plane fill skeleton is already behaviorally complete.
