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
