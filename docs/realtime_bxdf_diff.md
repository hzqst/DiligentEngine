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
