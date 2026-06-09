# Reference BxDF Path Differences

Last updated: 2026-06-09

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
  filtering, camera jitter, `minBounceCount` Russian roulette, volume
  absorption, and nested-dielectric rejection.
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

3. **Reference diffuse-bounce classification was intentionally narrower.** Synced
   on 2026-06-08: reference `GenerateScatterRay` now uses the same classification
   as upstream/realtime: diffuse reflection, diffuse transmission, or roughness
   above `kSpecularRoughnessThreshold` count as diffuse-like. Diffuse
   transmission keeps the upstream/realtime every-other-vertex increment guard to
   avoid overdarkening diffuse volumes.

4. **`GenerateScatterRay` omitted upstream scatter side effects.** Synced on
   2026-06-08: non-delta scatter now expands `path.rayCone` spread angle using
   `ComputeRayConeSpreadAngleExpansionByScatterPDF`, clamped to `2*pi`, and every
   valid scatter sets `PathFlags::enableThreadReorder`.

5. **Reference used an inlined single-overload `GenerateScatterRay`.** Synced on
   2026-06-08: local now matches the upstream split shape with a BSDF-sample
   consumer overload plus a sampling overload. The local `BSDFSample.wi` naming and
   `MakeBSDFSample.deltaLobeIndex` FILL fix are preserved.

### Remaining confirmed divergences from upstream (with effect assessment)

No confirmed divergences remain in the compared reference `GenerateScatterRay`
scatter-flow slice after the 2026-06-08 syncs above.

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

## 2026-06-08 Realtime BUILD/FILL Stable-Plane Delta-Tree Transmission Comparison

This section follows up on suspect #1 above ("realtime is a different code path
than reference"). It is a source-level comparison of the realtime stable-plane
delta-tree transmission flow between the local Diligent port and upstream
`D:/RTXPT-fork`. As before, it is a static read; it does not prove runtime
shader state or image parity.

### Files compared (stable-plane path)

| Local (`DiligentSamples/.../PathTracer`) | Upstream (`Rtxpt/Shaders/PathTracer/...`) |
| --- | --- |
| `PathTracerStablePlanes.hlsli` `SplitDeltaPath` / `StablePlanesHandleHit` (BUILD) / `StablePlanesOnScatter` (FILL) | `PathTracerStablePlanes.hlsli` (same functions) |
| `StablePlanes.hlsli` branch-ID helpers + `StablePlane` payload pack/unpack | `StablePlanes.hlsli` (same) |
| `Rendering/Materials/BxDF.hlsli` `FalcorBSDF::evalDeltaLobes` + `getLobes` | `Rendering/Materials/BxDF.hlsli` `evalDeltaLobes` |
| `PathTracerTypes.hlsli` `ActiveBSDF::evalDeltaLobes` wrapper + `BSDFSample::getDeltaLobeIndex` | `Rendering/Materials/IBSDF.hlsli` `BSDFSample::getDeltaLobeIndex`; `StandardBSDF::evalDeltaLobes` wrapper |
| `PathTracer.hlsli` `MakeBSDFSample` (`deltaLobeIndex` assignment) | `IBSDF.hlsli` `getDeltaLobeIndex` (computed on the fly) |

### Verdict: the realtime stable-plane delta-tree transmission path is a faithful, functionally-correct port

Every transmission-relevant element of the BUILD/FILL delta-tree path lines up
with upstream:

- **Delta-lobe enumeration order matches.** Local `FalcorBSDF::evalDeltaLobes`
  (`BxDF.hlsli:871`–`872`) emits `deltaLobes[0] = deltaTransmission;
  deltaLobes[1] = deltaReflection;`, byte-for-byte the upstream order
  (`BxDF.hlsli:1051`–`1052`) and its convention comment ("the index must match
  `BSDFSample::getDeltaLobeIndex()`"). The delta-transmission `dir`/`thp`/
  `probability` math (`-wi*eta`, `-cosThetaT`, thin-surface `eta=1` hack,
  `pSpecularReflectionTransmission*(1-F)`) is identical.
- **`getDeltaLobeIndex` mapping matches the order.** Local `MakeBSDFSample`
  (`PathTracer.hlsli:375`–`376`) sets `deltaLobeIndex = (lobe & Delta)==0 ?
  0xFFFFFFFF : ((lobe & Transmission)==0 ? 1 : 0)`. Upstream
  `BSDFSample::getDeltaLobeIndex` (`IBSDF.hlsli:55`–`60`) computes the same:
  non-delta → `0xFFFFFFFF`, delta-transmission → `0`, delta-reflection → `1`.
  So a FILL delta-transmission scatter advances `stableBranchID` with the same
  index (`0`) that BUILD's `SplitDeltaPath` used for `deltaLobes[0]`. The
  previously-documented FILL `deltaLobeIndex` fix is verified correct against
  upstream — it is the exact equivalent of upstream's on-the-fly computation,
  including the `0xFFFFFFFF` non-delta sentinel that cleanly drops the path off
  the stable branch.
- **Branch-ID arithmetic matches.** `StablePlanes.hlsli`
  `StablePlanesAdvanceBranchID` (`<<2 | deltaLobeID`),
  `StablePlanesVertexIndexFromBranchID` (`firstbithigh/2+1`),
  `StablePlaneIsOnPlane`, and `StablePlaneIsOnStablePath` are line-for-line
  identical to upstream. `cStablePlaneMaxVertexIndex` (15) and the invalid/
  enqueued/just-started sentinels match.
- **`SplitDeltaPath` transmission handling matches.** Local
  (`PathTracerStablePlanes.hlsli:29`–`48`) sets the new origin via
  `computeNewRayOrigin(lobe.transmission == 0)` (view-side bool, same as
  upstream `bs.isLobe(Reflection)` semantics), flags `setScatterTransmission`,
  and on a non-thin transmit calls
  `interiorList.handleIntersection(materialID, nestedPriority, frontFacing)` +
  `setInsideDielectricVolume` under `RTXPT_NESTED_DIELECTRICS_QUALITY > 0` —
  identical to upstream (`:36`–`60`).
- **BUILD split/reuse logic matches.** `StablePlanesHandleHit`
  (`PathTracerStablePlanes.hlsli:89`–`231`) reproduces upstream's
  `evalDeltaLobes` call, `deltaLobeCount = max(cMaxDeltaLobes-1, …)` clamp
  (`cMaxDeltaLobes == 3` on both sides, so the floor is 2 — equal),
  `nonZeroDeltaLobes` pruning, `allowPSR` / `canReuseExisting` gating,
  `StoreExplorationStart` enqueue, and `lobeForReuse` continuation. Upstream's
  `USE_THP_PRIORITIZED_PRUNING` block is compiled out (`0`), so the active
  pruning (`min(nonZeroDeltaLobeCount, availablePlaneCount + canReuseExisting)`)
  is the same.
- **FILL `StablePlanesOnScatter` matches.** Local (`:236`–`284`) advances the
  branch ID with `bs.getDeltaLobeIndex()`, walks the planes for on-plane /
  on-path membership, commits denoiser radiance, and increments
  `BouncesFromStablePlane` exactly as upstream (`:327`–`379`). The
  `stablePlaneBaseScatterDiff` mask `kLobeTypeDiffuseReflection |
  kLobeTypeDiffuseTransmission` = `0x11` equals upstream `(uint)LobeType::Diffuse`
  = `0x11`.

### The one structural divergence (faithful)

Local splits upstream's single `DeltaLobe` into two structs:
`BxDFDeltaLobe[cBxDFMaxDeltaLobes = 2]` (BxDF-internal, `BxDF.hlsli:691`–`709`)
and `DeltaLobe[cMaxDeltaLobes = 3]` (stable-plane facing,
`PathTracerTypes.hlsli:11`–`29`). Upstream uses one `DeltaLobe[cMaxDeltaLobes = 3]`
throughout. The `ActiveBSDF::evalDeltaLobes` wrapper
(`PathTracerTypes.hlsli:95`–`123`) bridges them and is faithful:

- it zero-inits all 3 `DeltaLobe`s, builds the `FalcorBSDF`, transforms the view
  to tangent-local (`viewLocal`), and calls the BxDF `evalDeltaLobes`;
- it copies `deltaLobeCount = min(localCount, cMaxDeltaLobes) = min(2, 3) = 2`
  lobes **in order** (`deltaLobes[i] ← localLobes[i]`), preserving the
  `[0]=transmission, [1]=reflection` convention and all four fields
  (`dir`/`thp`/`transmission`/`probability`);
- it transforms each lobe `dir` from tangent-local back to world space
  (`normalize(T*x + B*y + N*z)`). This is the same local→world step upstream
  performs in its `StandardBSDF::evalDeltaLobes` wrapper, and it is required
  because `SplitDeltaPath` consumes `lobe.dir` in world space (`SetDir(lobe.dir)`,
  `MatrixRotateFromTo(lobe.dir, rayDir)` with world-space `rayDir`).

The two struct definitions are field-identical, so no transmission data is lost
or reordered across the split.

### Cosmetic / harmless differences (no behavioral effect)

1. **MV-transform guard spelling.** `SplitDeltaPath` uses
   `if (newPath.GetMotionVectorSceneLength() == 0)` (local) vs upstream's
   `if (!newPath.GetMotionVectorSceneLength()!=0)`. By C/HLSL precedence the
   upstream form is `(!len) != 0`, which is also true iff `len == 0`. The two are
   identical, and the block only updates the motion-vector image transform — it
   does not touch transmission throughput or direction.
2. **Exploration-payload marshalling.** Local `StoreExplorationStart` packs via
   `PathPayload::pack` → `PathPayload::toArray` into
   `uint4[RTXPT_PATH_PAYLOAD_UINT4_COUNT]` before the store, where upstream
   passes `PathPayload::pack(splitPath).packed` directly. `StablePlane::
   PackCustomPayload`/`UnpackCustomPayload` read `packed[0..4]` on both sides, so
   this requires `RTXPT_PATH_PAYLOAD_UINT4_COUNT == 5` in BUILD mode (the realtime
   compact payload). Worth a one-time confirmation, but it is a marshalling
   adaptation, not a logic change.

### Updated conclusion: suspect #1 is cleared — the defect is a shared, pre-lobe root cause

With both the reference scatter spine (previous section) **and** the realtime
BUILD/FILL stable-plane delta-tree now confirmed faithful, the transmission
symptom is no longer plausibly explained by either mode's scatter-flow logic.
The remaining explanation that fits "transmission broken in *both* modes" is a
defect **upstream of BSDF lobe construction or in throughput accounting**, shared
by both code paths:

- **Most likely — the transmission lobe is never constructed.** Both paths gate
  the delta/spec transmission lobe on `getLobes` (`BxDF.hlsli:789`–`808`):
  `kLobeTypeDeltaTransmission`/`kLobeTypeSpecularTransmission` is only added when
  `data.SpecularTransmission() > 0`. `SpecularTransmission` is
  `saturate(transmissionFactor) * (1 - metallic)` set in `MakeStandardBSDFData`.
  If `transmissionFactor` reaches the shader as `0` (material not authored with
  `KHR_materials_transmission` / the RTXPT transmission extension, or `metallic`
  ≈ 1), neither delta enumeration (BUILD) nor `SampleBSDF` (REFERENCE/FILL) ever
  selects a transmission lobe — identical symptom in both modes for one data
  reason. **This is now the top suspect.**
- **Secondary — eta degeneracy.** If `eta` collapses to `1.0` (outside IoR ==
  interior IoR, e.g. both defaulting to 1.5, or the bridge feeding a wrong IoR),
  `evalDeltaLobes` still emits a delta-transmission lobe but with
  `deltaTransmission.dir = -wi` (no bending) — glass renders as an invisible
  pass-through rather than a refractor, which can read as "transmission missing".
- **Tertiary — throughput zeroed by volume absorption**, as already noted for the
  reference path (`Bridge::loadHomogeneousVolumeData`), applies equally to FILL.

### Revised next checks

- **Promote the material-data dump to the primary action.** Capture the resolved
  `StandardBSDFData` at a glass hit (or dump `MaterialPTData` at load) and confirm
  `SpecularTransmission() > 0`, `metallic < 1`, and `eta != 1.0` actually reach
  `FalcorBSDF::__init`/`getLobes`. If `specTrans == 0` or `eta == 1`, the fix is in
  the material pipeline (`RTXPTMaterials.cpp` / `MaterialBridge.hlsli` /
  glTF authoring), not in either scatter path.
- De-prioritize further stable-plane scatter-flow diffing — this round found no
  transmission-relevant divergence there.
- Keep the current visible reference output as the baseline; avoid persistent
  color-marker diagnostics.

## 2026-06-08 Material Pipeline transmissionFactor / metallic / ior → specTrans / eta Verification

This section traces the material data flow end-to-end
(`RTXPTMaterials.cpp` → `MaterialBridge.hlsli` → `MakeStandardBSDFData` →
`FalcorBSDF::__init`/`getLobes`) to confirm whether `specTrans` and `eta` are
correctly populated. Static read only.

### Verdict: the pipeline is a faithful, correct port — `specTrans`/`eta` are populated correctly given sane inputs

- **C++ record is layout-safe and populated.** `MaterialPTData`
  (`RTXPTMaterials.hpp:50`–`137`) has `static_assert`-pinned offsets (sizeof 144,
  `transmissionFactor@80`, `diffuseTransmissionFactor@84`, `ior@88`). The glTF
  fill (`RTXPTMaterials.cpp:63`–`147`) writes `metallicFactor`/`roughnessFactor`,
  a hard-coded `ior = 1.5`, and — **only if `Material.Transmission != nullptr`** —
  `transmissionFactor` plus `kMaterialFlag_HasTransmission`. The scene-graph /
  extension path (`:334`–`450`) additionally overrides `metallicFactor`,
  `transmissionFactor`, `diffuseTransmissionFactor`, `ior = max(Ext.IoR, 1.0)`,
  `nestedPriority`, and the `ThinSurface`/`HasVolume` flags.
- **Bridge reads are correct.** `MaterialBridge.hlsli`:
  `getTransmission` → `saturate(transmissionFactor)` (`:86`/`:116`),
  `getMetallicRoughness().x` → `metallicFactor` (`:61`/`:114`),
  `loadIoR` → `max(ior, 1.0)` (`:121`),
  `isThinSurface` → `thinFlag || !HasTransmission` (`:150`).
- **`MakeStandardBSDFData` math is faithful to upstream.** `BxDF.hlsli:109`–`112`:
  - `specularTransmission = saturate(transmissionFactor) * (1 - metallic)`
  - `diffuseTransmission  = saturate(diffuseTransmissionFactor) * (1 - metallic)`
  - `eta = frontFacing ? safeOutsideIoR/safeMaterialIoR : safeMaterialIoR/safeOutsideIoR`

  The `(1 - metallic)` factor is **not** a local invention: upstream
  `PathTracerBridgeDonut.hlsli:745`–`746` applies the identical
  `transmission * (1 - metalness)` (and `diffuseTransmission * (1 - metalness)`),
  explicitly citing the `KHR_materials_transmission` "transparent metals" rule.
  The eta formula matches upstream `Bridge::updateOutsideIoR`
  (`PathTracerBridgeDonut.hlsli:860`: `frontFacing ? IoR/interiorIoR :
  interiorIoR/IoR`); the local load-time seed uses `outsideIoR = 1.0` and is then
  refined at runtime by `UpdateSurfaceOutsideIoR` (verified equivalent in the
  earlier section).
- **`__init` consumes them faithfully.** `BxDF.hlsli:756`–`769`:
  `specTrans = data.SpecularTransmission()`, the
  `metallicBRDF`/`dielectricBSDF`/`specularBSDF = specTrans` partition, and
  `specularReflectionTransmission.{eta, alpha=(eta==1)?0:alpha}` are a line-for-line
  port. `getLobes` (`:804`) adds `kLobeTypeDeltaTransmission`/
  `kLobeTypeSpecularTransmission` **iff `SpecularTransmission() > 0`**.

So there is **no port defect** in the material → `specTrans`/`eta` plumbing.
`specTrans` and `eta` are correctly populated whenever the inputs are sane
(`transmissionFactor > 0`, `metallic < 1`, `materialIoR ≠ outsideIoR`).

### The conditions that zero `specTrans` (or degenerate `eta`) are data-driven, not code bugs

1. **`metallic == 1` → `specTrans == 0` (by design, faithful).** Both the glTF 2.0
   default and the `MaterialPTData` C++ default for `metallicFactor` are **1.0**
   (`RTXPTMaterials.hpp:60`). A glass material that does not explicitly author
   `metallicFactor = 0` (via `pbrMetallicRoughness` or the RTXPT extension
   `MetallicFactor`) gets `specTrans = transmission * (1 - 1) = 0` and **never
   selects a transmission lobe — identically in upstream** (transparent metals are
   opaque per the glTF spec). This is the **single highest-probability cause** of
   "transmission missing in both modes", and it is a scene-authoring issue rather
   than a port regression.
2. **`transmissionFactor == 0` reaching the shader.** In the glTF-only
   `Upload(Model)` path, `transmissionFactor`/`HasTransmission` are set **only when
   `Material.Transmission != nullptr`** (`RTXPTMaterials.cpp:108`) — i.e. only if
   the Diligent GLTF loader actually parsed `KHR_materials_transmission`. If the
   loader leaves `Material.Transmission` null (or the material isn't authored with
   transmission), `transmissionFactor` stays `0`, `HasTransmission` is clear, and
   `isThinSurface` then returns `true` (so no nested-dielectric volume either) —
   reading as opaque in both modes.
3. **`eta` degeneracy is unlikely with defaults.** With `materialIoR = 1.5` and a
   vacuum outside (`1.0`), front-facing `eta = 1/1.5 ≈ 0.667 ≠ 1`, so the
   `eta == 1` → no-bending / `alpha = 0` path is not triggered. `eta == 1` would
   require an authored `ior == 1.0` or a broken interior list feeding
   `outsideIoR == materialIoR`.

### Secondary observation (not a transmission-loss cause)

The glTF-only `Upload(Model)` path never sets `kMaterialFlag_ThinSurface`,
`diffuseTransmissionFactor`, `nestedPriority` (stays struct-default `14`), or a
non-`1.5` `ior`; only the `Upload(SceneData)` extension path does. For a single
solid glass this is still consistent (`thinSurface = false` because
`HasTransmission` is set and the flag is clear; `ior = 1.5`), but if the test
scene needs an authored IoR or thin-walled glass, it must go through the
extension path.

### Decisive next step

The plumbing is verified faithful, so the remaining question is purely runtime
data. **Dump `metallicFactor`, `transmissionFactor`, `ior`, and `flags` for the
test glass material** (either log `MaterialPTData` at upload in
`RTXPTMaterials::Upload`, or read back `StandardBSDFData.SpecularTransmission()`/
`Eta()`/`Metallic()` at a glass hit). Expected for working glass:
`transmissionFactor > 0`, `metallic ≈ 0`, `ior ≈ 1.5`, `HasTransmission` set. If
`metallic == 1` or `transmissionFactor == 0`, the fix is in scene authoring /
the GLTF loader's `KHR_materials_transmission` support — not in any shader.

## 2026-06-08 convergence-test Glass Regression: Material Ruled Out, Static Transmission Path Verified Faithful

The user confirmed (a) the Diligent GLTF loader **does** parse
`KHR_materials_transmission` (populates `Material.Transmission`), and (b)
reference-mode glass transmission/refraction **worked at commit
`cf829f32` ("fix: Add WorkingDirectory")** and is now black. This makes it a
**shader-path regression**, not a material-data problem.

### Scene material data is ideal (material hypotheses fully ruled out)

`convergence-test.scene.json` → `Models/ConvergenceTest/ConvergenceTest.gltf`.
The transmissive materials are authored cleanly, e.g. material 17 "Smooth Glass":

```json
"KHR_materials_transmission": { "transmissionFactor": 1 },
"pbrMetallicRoughness": { "baseColorFactor": [1,1,1,1], "metallicFactor": 0, "roughnessFactor": 0.0 },
"doubleSided": true
```

No `KHR_materials_volume`, no `KHR_materials_ior`. Through the (verified)
pipeline this yields `specularTransmission = 1·(1-0) = 1`, `eta = 1/1.5 ≈ 0.667`
(≠ 1), and **no** volume absorption. So `metallic==1`, volume-absorption, and
`eta==1` are all ruled out for this scene.

### The entire current transmission shader path was traced and is faithful/correct

For this exact glass, every transmission-relevant step was read and the values
hand-traced:

- `FalcorBSDF::__init` normalizes `pSpecularReflectionTransmission → 1` (100% of
  the lobe budget goes to spec-refl-transmission), so the transmission lobe is
  selected.
- `SpecularReflectionTransmissionMicrofacet::sample` returns the correct delta
  refraction `wo = (-wi.x·η, -wi.y·η, -cosθT)`, with TIR handled via the Fresnel
  split; `weight = transmissionAlbedo = sqrt([1,1,1]) = [1,1,1]` (the
  Fresnel/selection-probability factors cancel — unbiased), `pdf = 0` for delta.
- `FalcorBSDF::sample` finalizes `weight = transmissionAlbedo·specTrans /
  pSpecularReflectionTransmission = [1,1,1]`. **This matches the cf829f32 weight
  byte-for-byte** (`transmissionAlbedo·specularTransmission/pSpecularTransmission`).
- `GenerateScatterRay` offsets along `-faceNCorrected`, sets the refracted dir,
  updates the interior list on transmit.
- `HandleNestedDielectrics` / `InteriorList` are faithful; `ComputeOutsideIoR`
  gives `outsideIoR = 1.0` on both faces of a single glass, so `eta` is `0.667`
  entering and `1.5` exiting (correct).
- The reference `PathPayload` **does** carry the interior list (`packed[4].xy`),
  the fp32 throughput (`packed[2]`), and radiance (`packed[3]`).
- `HandleMiss` weights the environment by `ComputeBSDFEnvMISWeight`, which
  returns `1.0` when `prevBsdfPdf <= 0` — i.e. **full weight for the delta
  refracted ray** (no erroneous MIS suppression).
- `HasFinishedSurfaceBounces` / RR are faithful; `bounceCount` defaults to `4`
  (≥ the 3 vertices a glass ball needs: enter → exit → background), and the
  reference raygen loop ceiling (`max(2, bounceCount + maxRejectedHits + 2)`) is
  generous. The RR throughput correction `GetThpRuRuCorrection()` is initialized
  to `1.0` and never reduced (rchit:239 is effectively a no-op).

Conclusion: **the transmission source as written produces a correct, throughput-1
refraction for the convergence-test glass.** No source-level logic divergence was
found that would turn it black.

### Therefore the regression is in the refactor delta, not a spotted source bug

Between `cf829f32` (working) and HEAD there are ~30 commits that **(i)**
componentized the BxDF through `FalcorBSDF` (`componentize … BxDF lobes`,
`route BxDF wrappers through FalcorBSDF`, `gate specular transmission delta
lobes`, `preserve gated Fresnel branch weights`) and **(ii)** rerouted the
reference path from a large self-contained raygen (~440 lines, payload even
carried volume attenuation directly) into the unified `PathPayload` spine
(`compile/preserve/route reference … PathState spine`, `stabilize reference path
baseline`). Because the *current* source reads as faithful and value-correct, the
regression is most likely a **refactor-interaction bug or a DXC miscompilation**
(there is precedent: `fc60217a` fixed a DXC miscompilation of the realtime opaque
path under the Debug flags `-Zi -Od -Qembed_debug -Zpr`) rather than a spotted
logic error.

### Decisive next step: bisect `cf829f32 .. HEAD` (build required)

Static reading has been exhausted; localizing this needs runtime signal:

1. **`git bisect`** in `DiligentSamples` between `cf829f32` (good) and HEAD (bad),
   rendering `convergence-test.scene.json` in reference mode each step and
   checking whether the glass spheres refract. Prime suspect groups, in order:
   (a) the reference spine refactor (`3c87e67d`, `6158342c`, `c499efea`,
   `3d348767`); (b) the BxDF componentization (`cbe687eb` … `970a7378`,
   especially `495f09a0 gate specular transmission delta lobes` /
   `bed8b4af preserve gated Fresnel branch weights`).
2. **Or a targeted runtime probe** (temporary, removed after): at the first glass
   hit, write out the selected `lobe`, `bs.weight`, `bsdf.standardData.eta`, and
   `path.GetThp()` to confirm whether the transmission lobe is selected and where
   the throughput collapses. This pinpoints the failing stage without a full
   bisect.

Building/running was not performed (per project rule: no unprompted builds/tests).

## 2026-06-09 Root Cause Confirmed: DXC `-Od` Miscompiles `HandleHit` (nested `inout`)

The DXC-miscompilation hypothesis from the section above is confirmed. The root
cause of both the reference and realtime transmission/refraction regression is the
DXC build flag `-Od` (disable optimizations):

- Compiled with `-Od`, DXC emits incorrect DXIL for `PathTracer::HandleHit`,
  causing it to malfunction. The symptom is consistent with DXC's
  optimization-disabled DXIL mishandling nested `inout` parameters — `HandleHit`
  and its callees thread deeply nested `inout` state (path state, surface data,
  interior list, BSDF sample), exactly the construct that triggers the bad codegen.
- This matches the precedent already noted above: `fc60217a` fixed a DXC
  miscompilation of the realtime opaque path under the same `-Zi -Od
  -Qembed_debug -Zpr` Debug flags.
- With optimizations enabled (no `-Od`), DXC emits correct DXIL, `HandleHit`
  behaves correctly, and **reference mode now renders identically to upstream
  RTXPT**. The transmission/refraction contribution this note tracked as "missing"
  is restored, and the realtime symptoms are gone as well.

This closes the investigation recorded in this note. Every source-level audit
above — reference scatter spine, realtime BUILD/FILL stable-plane delta-tree, and
the material → `specTrans`/`eta` pipeline — was correct in concluding the port was
faithful. The defect was never in the shader source; it was in `-Od` DXIL codegen.

Follow-up: do not compile the RTXPT ray tracing shaders with `-Od`. The Serena
`suggested_commands` smoke command has been updated to drop `-Od`.
