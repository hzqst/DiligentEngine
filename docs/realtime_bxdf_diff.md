# Reference BxDF Path Differences

Last updated: 2026-06-08

This note records the current source-level differences between the Diligent RTXPT
reference path and upstream `D:/RTXPT-fork`. It does not prove runtime shader
state, code generation behavior, or image parity.

## Reference PathState Spine

The local reference variant now uses the same state-transport spine shape as
upstream RTXPT:

- raygen initializes `PathState`;
- primary and visibility rays carry `PathPayload`;
- closest-hit calls `PathTracer::HandleHit`;
- miss calls `PathTracer::HandleMiss`;
- final reference output is committed by `PathTracer::CommitPixel`.

Reference mode still has Diligent-specific exits:

- raw HDR radiance is accumulated in `PathState::GetL().rgb` and written to the
  output color UAV;
- reference `PathPayload` uses fp32 lanes for radiance, throughput, BSDF pdf,
  firefly K, and MIS state, while realtime BUILD/FILL keep the 5xuint4 packed
  payload;
- primary depth is captured once through reference-only `PathState` accessors
  and written to the existing depth UAV;
- screen motion vectors remain zero for reference;
- Russian roulette uses the reference `minBounceCount` and throughput
  correction semantics;
- the unified reference raygen keeps an explicit loop safety ceiling;
- stable-plane storage, stable-plane resolve, denoiser radiance, and spec-hit
  guide export are not used by reference.

The Diligent bridge remains intentionally different from upstream Donut.
`LoadCurrentSurfaceData` is the shared local surface construction path and
continues to source material, lighting, environment, and geometry state from the
Diligent scene adapters.

## Remaining Bridge Differences

- Local `PathTracerBridge.hlsli` uses Diligent resource bindings and scene
  adapter helpers rather than Donut bridge calls.
- Local `LoadCurrentSurfaceData` constructs `StablePlaneShadingData`,
  `ActiveBSDF`, and Diligent-specific PSD/material metadata from the current
  Diligent scene buffers.
- Emissive-hit MIS uses the local emissive triangle list and a precomputed
  solid-angle PDF carried in `SurfaceData::emissiveLightPdf`.
- Reference branches inside the shared spine preserve the current Diligent
  reference estimator: full-sample NEE, emissive/environment MIS, firefly
  filtering, camera jitter, diffuse-bounce classification, `minBounceCount`
  Russian roulette, volume absorption, and nested-dielectric rejection.
- Realtime BUILD/FILL stable-plane side effects remain guarded away from
  reference mode.

## 2026-06-08 Baseline After Reference Black-Screen Fix

Current confirmed runtime state:

- Reference mode no longer renders all black.
- Diagnostic color writes and raw payload color probes have been removed.
- Opaque, diffuse, specular, emissive, NEE, and environment contributions are
  visible in the Cornell-style BxDF test scene.
- The known remaining visual gap is that transmission/refraction contribution
  is still missing.

Important implementation notes for this baseline:

- Reference closest-hit now keeps the upstream-like shape locally: it loads
  `SurfaceData` and handles the reference hit in the same function, avoiding
  the extra `HandleHit(path, surfaceData, ...)` call for reference mode.
- Reference mode applies the upstream Russian-roulette throughput correction
  before scatter via `UpdatePathThroughput(path, path.GetThpRuRuCorrection().xxx)`.
- Reference ray tracing PSO uses recursion depth 2, matching the fact that
  reference closest-hit can trace visibility rays for NEE.
- Reference payload transport uses a 7xuint4 payload layout to carry fp32
  radiance/throughput, BSDF pdf, firefly K, MIS state, and reference counters.
  BUILD/FILL realtime variants keep the compact 5xuint4 payload layout.
- Nested-dielectric and volume blocks are runtime-gated by
  `nestedDielectricsQuality` where relevant, but transmission parity is not yet
  restored.

Debugging lesson preserved for future shader checks:

- Diligent Debug builds compile DXC shaders with `-Zi -Od -Qembed_debug -Zpr`.
  Ordinary optimized DXC smoke commands can miss validation failures that only
  appear under the runtime Debug flags.
- The matching RTXPT closest-hit smoke command is recorded in Serena memory
  `suggested_commands`.

Suggested next handoff focus:

- Compare upstream transmission/nested-dielectric scatter flow against the local
  `LoadCurrentSurfaceData`, `HandleNestedDielectrics`, `GenerateScatterRay`, and
  material bridge transmission fields.
- Keep the current visible reference output as the baseline and avoid adding
  persistent color-marker diagnostics.

## 2026-06-08 Transmission / Nested-Dielectric Scatter-Flow Comparison

This section records a source-level comparison of the transmission and
nested-dielectric scatter flow between the local Diligent port and upstream
`D:/RTXPT-fork`. It is a static read of the shaders/C++; it does not prove
runtime shader state or image parity.

### Files compared

| Local (`DiligentSamples/.../PathTracer`) | Upstream (`Rtxpt/Shaders/...`) |
| --- | --- |
| `PathTracer.hlsli` `GenerateScatterRay` / `HandleNestedDielectrics` / `UpdateSurfaceOutsideIoR` / `UpdateNestedDielectricsOnScatterTransmission` | `PathTracer/PathTracer.hlsli` `GenerateScatterRay`; `PathTracer/PathTracerNestedDielectrics.hlsli` |
| `PathTracerNestedDielectrics.hlsli` `ComputeOutsideIoR` / `GetMaxRejectedDielectricHits` | `PathTracer/PathTracerNestedDielectrics.hlsli` `ComputeOutsideIoR` (`kMaxRejectedDielectricHits`) |
| `PathTracerClosestHit.rchit` `LoadCurrentSurfaceData` + reference `HandleHit` | `PathTracerBridgeDonut.hlsli::loadSurface` + `PathTracer.hlsli::HandleHit` |
| `Rendering/Materials/BxDF.hlsli` `MakeStandardBSDFData` / `SpecularReflectionTransmissionMicrofacet` / `FalcorBSDF` / `SampleBSDF` / `evalDeltaLobes` | `Rendering/Materials/BxDF.hlsli` (`StandardBSDFData`, same structs) |
| `Rendering/Materials/InteriorList.hlsli` | `Rendering/Materials/InteriorList.hlsli` |
| `Rendering/Materials/MaterialBridge.hlsli` + `src/RTXPTMaterials.cpp` | `PathTracerBridgeDonut.hlsli` + Falcor material system |

### Verdict: the named reference scatter spine is a faithful, functionally-correct port

The four areas called out for this handoff (`LoadCurrentSurfaceData`,
`HandleNestedDielectrics`, `GenerateScatterRay`, material-bridge transmission
fields) line up with upstream. Concretely:

- **eta convention matches.** Local `UpdateSurfaceOutsideIoR`
  (`PathTracer.hlsli:312`) sets
  `eta = frontFacing ? outsideIoR/interiorIoR : interiorIoR/outsideIoR`, identical
  to upstream `Bridge::updateOutsideIoR` (`PathTracerBridgeDonut.hlsli:855`). The
  local adds `max(.,1.0)` clamps; harmless for IoR ≥ 1. `MakeStandardBSDFData`
  (`BxDF.hlsli:109`) seeds the same eta at load time, and `interiorIoR` is
  correctly threaded through `SurfaceData::make` (`PathTracerTypes.hlsli:198`).
- **transmission-side ray offset matches.** Local `GenerateScatterRay`
  (`PathTracer.hlsli:484`) offsets the next origin along
  `isTransmission ? -faceNCorrected : +faceNCorrected`, equivalent to upstream
  `shadingData.computeNewRayOrigin(bs.isLobe(Reflection))`. `kBSDFLobeTransmission`
  (= `kLobeTypeTransmission` = `0xf0`) includes delta transmission `0x40`, so
  smooth (delta) glass offsets on the correct side. Lobe-type bit layout in
  `LobeType.hlsli` is byte-identical to upstream.
- **interior-list update on transmit matches.** Local
  `UpdateNestedDielectricsOnScatterTransmission` (`PathTracer.hlsli:384`) mirrors
  upstream `PathTracerNestedDielectrics.hlsli:118`
  (`handleIntersection` + `setInsideDielectricVolume`).
- **false-hit rejection matches.** Local `HandleNestedDielectrics`
  (`PathTracer.hlsli:321`) reproduces upstream's reject path (increment
  `RejectedHits`, push interface, re-offset origin along `-faceNCorrected`,
  `decrementVertexIndex`, return `false`), then updates the outside IoR.
- **BSDF transmission math matches.** `SpecularReflectionTransmissionMicrofacet`
  (`BxDF.hlsli:493`) and `FalcorBSDF::__init/sample/eval/evalPdf`
  (`BxDF.hlsli:711`) are a line-for-line port of upstream `BxDF.hlsli:385/737`:
  same Fresnel split, same delta/rough refraction `wo` (`(eta*wiDotH - cosThetaT)*h - eta*wi`),
  same `transmissionAlbedo = thin ? T : sqrt(T)`, same
  `specularReflectionTransmission.alpha = (eta==1)?0:alpha`, same lobe
  probabilities (`specTrans` drives `pSpecularReflectionTransmission`). The only
  intentional divergence is `evalVisibilitySmithGGXCorrelated` returning
  `G/(4·NoV·NoL)` (reconstructed back to `G` before use), already recorded in
  `RTXPT_FORK_MAPPING.md`.
- **material data is populated.** `RTXPTMaterials.cpp` sets `transmissionFactor`,
  `diffuseTransmissionFactor`, `ior`, `nestedPriority`, and the
  `kMaterialFlag_HasTransmission` / `_ThinSurface` / `_HasVolume` flags from glTF +
  the RTXPT extension (`RTXPTMaterials.cpp:108`,`:355`). `MaterialBridge.hlsli`
  reads them back into `specularTransmission = transmission*(1-metallic)` and
  `diffuseTransmission`. Offsets are guarded by `static_assert`.
- **defaults are nested-dielectric-on.** Macro `RTXPT_NESTED_DIELECTRICS_QUALITY`
  defaults to `1` (`PathTracerHelpers.hlsli:39`), so the nested-dielectric path
  is compiled and enabled by default. Runtime `nestedDielectricsQuality` still
  exists in the constants layout/UI state, but no longer gates the shader-side
  nested-dielectric scatter flow after the 2026-06-08 upstream sync.

A single convex glass object traced through the reference path
(enter → refract → exit) is therefore expected to produce correct refraction even
with the interior list empty, because both the entry and exit faces recompute eta
from `materialIoR` vs the vacuum outside IoR. No defect was found in the reference
transmission scatter math itself.

### Resolved upstream divergences

1. **Runtime gating where upstream uses compile-time.** Synced on 2026-06-08:
   volume absorption, false-hit rejection, and on-transmit interior-list updates
   are now guarded only by `RTXPT_NESTED_DIELECTRICS_QUALITY` plus their local
   thin-surface/empty-list conditions. The shader no longer reads
   `workingContext.PtConsts.nestedDielectricsQuality` to decide these branches.

2. **`GetMaxRejectedDielectricHits` was runtime-derived.** Synced on 2026-06-08:
   local now derives the rejected-hit limit from compile-time
   `RTXPT_NESTED_DIELECTRICS_QUALITY` (`4` for Fast, `16` for Quality, `0` for
   Off), matching upstream's binding-time semantics. `PathTracerSample.rgen`
   consumes the zero-argument helper for the reference loop ceiling.

### Remaining confirmed divergences from upstream (with effect assessment)

1. **Reference diffuse-bounce classification is intentionally narrower.** Local
   reference (`PathTracer.hlsli:501`) counts a bounce as diffuse only when it is a
   diffuse-reflection lobe *or* (rough specular `roughness>0.25` **and not**
   transmission); upstream/realtime also count `DiffuseTransmission` and rough
   transmission. This is the previously-recorded intentional reference divergence;
   it changes how fast a transmissive path exhausts `diffuseBounceCount`, which can
   shorten transmissive paths but does not zero them.

2. **`GenerateScatterRay` omits a couple of upstream side effects.** The local
   port does not expand `path.rayCone` spread angle on non-delta scatter and does
   not set `PathFlags::enableThreadReorder` (cf. upstream `PathTracer.hlsli:292/341`).
   These affect texture LOD / firefly-K growth / thread reordering, not whether a
   transmission lobe is sampled.

3. **Reference uses an inlined single-overload `GenerateScatterRay`** that consumes
   pre-generated samples and calls `SampleBSDF` directly, whereas upstream splits
   into a sampler overload + a `GenerateScatterRay(bs, …)` consumer. Same net math.
   `MakeBSDFSample.deltaLobeIndex` carries the documented FILL fix
   (`PathTracer.hlsli:365`): non-delta → `0xFFFFFFFF`, delta transmission → `0`,
   delta reflection → `1`.

### Where transmission most plausibly breaks (prioritized suspects for next session)

Because the reference scatter math is faithful and on by default, the symptom is
most likely **outside** the four named reference functions:

1. **Realtime is a different code path than reference.** In realtime,
   `PATH_TRACER_MODE_BUILD_STABLE_PLANES` `HandleHit` returns right after
   `StablePlanesHandleHit` (`PathTracer.hlsli:784`–`826`) and **never calls
   `GenerateScatterRay`**. Delta transmission for the primary stable planes is
   instead enumerated by `FalcorBSDF::evalDeltaLobes` (`BxDF.hlsli:810`) inside
   `PathTracerStablePlanes.hlsli`. So "realtime transmission broken" and "reference
   transmission broken" can have *different* root causes. The BUILD/FILL
   delta-tree path (`PathTracerStablePlanes.hlsli`, `StablePlanesOnScatter`,
   branch-ID advance) was **not** part of this comparison and should be diffed
   against upstream `StablePlanes`/`PathTracerStablePlanes.hlsli` next — this is
   the top suspect for the realtime half of the bug.

2. **Scene material actually flagged transmissive?** Confirm the BxDF/Cornell test
   material reaches the shader with `transmissionFactor > 0` and
   `kMaterialFlag_HasTransmission` set (and the intended `thinSurface`/`ior`/
   `nestedPriority`). If the test glass is authored without the RTXPT transmission
   extension and without a glTF `KHR_materials_transmission` factor, the bridge
   reports `specTrans == 0` and the transmission lobe is never selected — which
   would read as "missing transmission" in both modes for an identical (data) reason.

3. **Bounce budget vs surfaces-per-glass.** Each glass interface consumes a vertex;
   with a low `bounceCount`/`diffuseBounceCount` a closed glass object may terminate
   before the refracted path reaches a light. Verify `bounceCount` is high enough
   for enter+exit (+ any internal bounces) before concluding the lobe is dropped.

4. **Throughput/absorption zeroing.** Check `Bridge::loadHomogeneousVolumeData`
   (`MaterialBridge.hlsli:133`): for a `HasVolume` material with a near-black
   `volumeAttenuationColor` or tiny `volumeAttenuationDistance`, `sigmaA` blows up
   and `evalTransmittance` drives throughput to ~0 over the glass path length,
   producing black "glass". Confirm the test material's attenuation is sane.

### Suggested next checks

- Diff `PathTracerStablePlanes.hlsli` + `StablePlanesOnScatter` + branch-ID handling
  against upstream `Shaders/PathTracer/PathTracerStablePlanes.hlsli` and
  `StablePlanes.hlsli` (realtime BUILD/FILL transmission).
- Dump the resolved `MaterialPTData` (transmissionFactor, flags, ior,
  nestedPriority) for the test glass material at load time and confirm
  `specTrans > 0` reaches `FalcorBSDF::__init`.
- Temporarily force `nestedDielectricsQuality` Off vs Fast vs Quality and a
  single convex glass slab to isolate reference refraction from nested-priority
  handling, then re-enable.
- Keep the current visible reference output as the baseline; avoid persistent
  color-marker diagnostics.
