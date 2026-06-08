# RTXPT Reference Unified PathState Design

Date: 2026-06-08

## Purpose

Move the Diligent RTXPT reference variant toward the upstream RTXPT reference
execution model by authoring a reference path through the local `PathState`
spine.

This is not a direct reuse of the current realtime spine. The current shader
tree has two mutually exclusive compile universes:

- reference mode compiles a flattened raygen loop and a material-hit payload;
- realtime build/fill modes compile `PathState`, `PathPayload`, stable planes,
  `PathTracer::HandleHit`, `PathTracer::HandleMiss`, and `CommitPixel`.

The design therefore explicitly chooses this route:

- make the spine compile in `PATH_TRACER_MODE_REFERENCE`;
- add reference-specific exits for radiance, depth, and motion outputs;
- keep stable-plane writes disabled for reference;
- preserve the current reference behavior first;
- only then remove the flattened raygen loop.

## Background

`docs/realtime_bxdf_diff.md` records that local reference mode currently does
not call local `PathTracer::HandleHit`. Instead, closest-hit fills
`RTXPTMaterialHitPayload`, and reference raygen reconstructs BSDF, NEE, MIS,
nested-dielectric, volume, and path termination behavior from payload fields.

The current reference output is visually correct after recent BxDF fixes. This
spec treats that output as the baseline. The remaining realtime smooth-glass
issue is still diagnosed as a realtime Fill stable-plane routing or shader
codegen problem, so this migration must not obscure that diagnosis.

## Review Corrections

The reviewed draft understated several facts about the local code. This revision
makes them design constraints:

1. Current `PathTracer.hlsli` excludes `PathState.hlsli`, `StablePlanes.hlsli`,
   and `PathTracerStablePlanes.hlsli` when `PATH_TRACER_MODE_REFERENCE`.
2. Current `HandleHit`, `HandleMiss`, `GenerateScatterRay`,
   `AccumulatePathRadiance`, and `CommitPixel` live in the non-reference
   branch.
3. Current `CommitPixel` has no reference `u_Output` path.
4. Current `AccumulatePathRadiance` only writes build stable radiance or fill
   path `L`; reference needs an explicit radiance accumulator.
5. Current `GenerateScatterRay` includes realtime/stable-plane side effects
   such as `StablePlanesOnScatter` and specular-hit-distance guide export.
6. Switching payloads before opening the reference spine would not compile.

## Goals

1. Make reference mode compile the `PathState`/`PathPayload` spine.
2. Add `PATH_TRACER_MODE_REFERENCE` behavior to the spine functions that need a
   reference output path.
3. Keep reference radiance accumulation writing raw HDR to `u_Output`.
4. Preserve reference depth and screen-motion-vector output contracts.
5. Preserve current reference sampling quality before deleting the flattened
   loop.
6. Make reference closest-hit call local `PathTracer::HandleHit`.
7. Make reference miss call local `PathTracer::HandleMiss`.
8. Keep Diligent scene, material, lighting, environment, and binding adapters.
9. Update source-diff documentation after implementation.

## Non-Goals

- Do not route reference through stable-plane storage.
- Do not add a stable-plane resolve pass for reference.
- Do not port the upstream Donut bridge wholesale.
- Do not make local `Bridge::loadSurface` byte-identical to upstream
  `PathTracerBridgeDonut.hlsli`.
- Do not introduce RTXDI/ReSTIR final shading parity.
- Do not solve the current realtime smooth-glass Fill issue as part of this
  migration.
- Do not shrink RT payload size in the first pass.
- Do not intentionally replace reference quality features with realtime
  approximations.

## Current-State Constraints

Relevant code constraints:

- `PathTracer.hlsli` includes `PathState.hlsli` and stable-plane headers only
  outside reference mode.
- `PathTracerSample.rgen` has a flattened reference branch and a separate
  non-reference `PathState` branch.
- `PathTracerClosestHit.rchit` uses `RTXPTMaterialHitPayload` for reference and
  `PathPayload` for realtime variants.
- `PathTracerMiss.rmiss` and visibility miss shaders follow the same payload
  split.
- `PathTracerTypes.hlsli::WorkingContext` currently contains `OutputColor`,
  `PtConsts`, and `StablePlanes`; reference needs output views but must not
  require stable-plane resources.
- `RTXPTRayTracingPass.cpp` currently uses a conservative 160-byte
  `MaxPayloadSize`.

The first implementation must work with these constraints instead of assuming
the current realtime functions are already reference-ready.

## Target Architecture

The desired end state is:

```text
Reference raygen
  -> PathTracer::EmptyPathInitialize
  -> PathTracer::SetupPathPrimaryRay
  -> PathTracer::StartPixel
  -> nextHit
       -> TraceRay with PathPayload
       -> closest-hit or miss
       -> PathTracer::HandleHit or PathTracer::HandleMiss
  -> postProcessHit
  -> ValidateNaNs
  -> PathTracer::CommitPixel
       -> u_Output + u_Depth + u_ScreenMotionVectors
```

Reference mode uses the same state-transport spine as build/fill, but not the
same stable-plane storage exits. Shared functions must be mode-aware:

- reference accumulates radiance into path state for final `u_Output`;
- build accumulates stable radiance and plane data;
- fill accumulates noisy radiance and commits denoiser radiance.

This is compatible with "no stable-plane writes" only if
`AccumulatePathRadiance`, `CommitPixel`, `HandleMiss`, `HandleHit`, and
`GenerateScatterRay` gain explicit reference branches.

## Compile-Universe Refactor

The first code checkpoint must make the spine available to reference while the
old flattened raygen remains active. This keeps the change verifiable and avoids
a big-bang payload switch.

Required shape:

1. Move or widen includes so `PathState`, `PathPayload`, and the shared
   non-stable-plane helpers compile under reference mode.
2. Avoid making reference require stable-plane resources. Either split
   `WorkingContext` fields by mode or provide reference-safe stubs for fields
   that only build/fill use.
3. Guard stable-plane-only operations with
   `PATH_TRACER_MODE != PATH_TRACER_MODE_REFERENCE`.
4. Keep the old reference raygen branch until the new spine compiles and has a
   reference output path.

This checkpoint should compile reference shaders without changing runtime
reference behavior.

## Reference Output Contract

Reference mode must keep writing the same downstream targets:

- `u_Output`: raw HDR radiance for accumulation and tone mapping.
- `u_Depth`: primary-ray depth semantics compatible with current reference
  output.
- `u_ScreenMotionVectors`: current reference writes zero motion vectors; this
  must remain true unless a later spec changes it.

`PathTracer::CommitPixel` must have a reference branch. It may write
`path.GetL().rgb` or another explicit reference radiance field, but the chosen
storage must be documented and validated.

Primary depth cannot be lost when `RTXPTMaterialHitPayload` is removed from the
main path. If `PathState` does not already carry the necessary first-hit depth
semantics, implementation must add a reference-safe way to capture and commit
it without affecting build/fill payload layout.

## Surface And Material Data

`LoadCurrentSurfaceData` remains the Diligent equivalent of upstream
`Bridge::loadSurface`. It should become the single surface construction path
for reference and realtime modes.

It must continue to populate:

- world position;
- shading normal;
- face normal;
- corrected vertex normal;
- tangent and bitangent;
- material ID;
- front-facing state;
- emission;
- shadow `NoL` fadeout;
- nested dielectric priority;
- thin-surface flag;
- active lobes and PSD state;
- `StandardBSDFData`;
- interior IoR;
- local NEE light indices when available.

No new material fields should be routed through `RTXPTMaterialHitPayload`.
Reference material state should come from `SurfaceData`.

## Reference Quality Invariants

The migration must preserve these current reference behaviors unless a later
design explicitly chooses to change them:

1. Full-sample NEE support:
   reference currently uses `min(32u, g_Const.ptConsts.NEEFullSamples)` for
   per-vertex direct-light sampling and emissive-triangle MIS. The first
   unified-spine implementation must not silently downgrade this to the
   realtime single-sample path.
2. Diffuse-bounce classification:
   current reference raygen and current realtime `GenerateScatterRay` classify
   diffuse transmission differently. The first migration must preserve current
   reference termination and low-discrepancy-sampler switching semantics, or it
   must record a deliberate estimator change.
3. Camera jitter:
   reference currently applies full per-pixel random jitter in addition to the
   camera jitter field. The unified path must preserve this.
4. Depth and motion outputs:
   primary depth and zero motion-vector output must remain compatible with the
   current reference branch.
5. Volume and nested-dielectric behavior:
   false-hit rejection, interior IoR, and volume transmittance must remain
   behaviorally equivalent to the current reference baseline before any source
   parity cleanup is attempted.

These are acceptance criteria, not optional risks.

## NEE, MIS, And Emission

Reference NEE/MIS should move out of the flattened raygen, but not by replacing
it with realtime approximations.

The target is a reference-capable `HandleHit` path that can:

1. update path travel;
2. load `SurfaceData`;
3. apply volume transmittance;
4. handle nested dielectrics;
5. accumulate emissive hit contribution with the current reference MIS
   behavior;
6. export primary depth and motion information for reference outputs;
7. generate BSDF scatter;
8. evaluate direct-light and environment NEE with reference sample-count
   semantics;
9. apply Russian roulette and termination through `PathState`;
10. commit final raw HDR radiance through the reference branch of
    `CommitPixel`.

Implementation may keep Diligent-specific helper functions for emissive
triangle MIS, environment NEE, and direct-light sampling. The key requirement is
that reference behavior is reproduced before the flattened loop is removed.

## Stable-Plane Isolation

Stable-plane-only behavior must remain isolated from reference:

- no `StablePlanesOnScatter` in reference;
- no `StablePlanesHandleHit` or `StablePlanesHandleMiss` in reference;
- no specular-hit-distance guide export in reference;
- no denoiser radiance commit in reference;
- no new reference bindings for stable-plane UAVs.

Because this migration touches shared functions used by realtime Fill, every
shared-function edit is part of the realtime regression surface and must be
verified accordingly.

## Payload And Pipeline Contract

The first implementation should keep
`RTXPTRayTracingPass.cpp::MaxPayloadSize = sizeof(float) * 40`.

The implementation plan must verify:

- actual `PathPayload` byte size;
- that the conservative 160-byte payload size covers every variant;
- that reference no longer depends on material-hit payload fields for primary
  depth, motion, or material state before removing that payload from the main
  path.

Payload-size reduction or deletion of `RTXPTMaterialHitPayload` is a later
cleanup after all variants pass validation.

## Implementation Sequencing

The plan must avoid non-compiling intermediate checkpoints. Use this order:

1. Make the `PathState` spine compile under reference mode while old reference
   raygen remains active.
2. Add reference branches for radiance accumulation, `CommitPixel`, depth
   export, motion export, and stable-plane side-effect suppression.
3. Add reference-quality NEE, MIS, diffuse-bounce, jitter, volume, and
   nested-dielectric preservation inside the spine.
4. Compile all variants and verify old reference behavior is still active.
5. Switch reference closest-hit and miss shaders to `PathPayload` and local
   hit/miss handlers.
6. Switch reference raygen to the `PathState` loop.
7. Remove or retire the flattened reference loop only after the new path
   compiles and matches the reference baseline.
8. Update `docs/realtime_bxdf_diff.md` and, if needed,
   `RTXPT_FORK_MAPPING.md`.

Each checkpoint must have a build or static verification before moving to the
next one.

## Verification Plan

Static checks:

- `rg` confirms reference spine functions compile in reference mode.
- `rg` confirms stable-plane-only calls are guarded out of reference mode.
- `rg` confirms reference closest-hit and miss paths call
  `PathTracer::HandleHit` and `PathTracer::HandleMiss` only after the spine is
  reference-capable.
- `rg` confirms the flattened reference loop is removed or disabled only at the
  final migration checkpoint.

Build checks:

- build the RTXPT sample target or closest available CMake target;
- confirm reference, build-stable-planes, and fill-stable-planes shader
  variants compile and RT PSOs initialize;
- confirm reference does not require stable-plane UAV bindings.

Reference runtime checks:

- load `convergence-test.scene.json`;
- compare current baseline and unified-spine reference output at a fixed sample
  count;
- use an image metric such as MSE or mean absolute luminance error when a
  capture pipeline is available;
- otherwise record fixed-settings screenshot comparisons and manual
  observations.

Realtime runtime checks:

- confirm opaque, diffuse, and metal realtime behavior do not regress;
- confirm the current smooth-glass failure does not widen into other material
  classes;
- note that touching shared functions may temporarily reduce confidence in the
  current Fill-path diagnosis, so before/after observations must be recorded.

## Risks

1. Opening `PathState` and shared helpers to reference can affect realtime
   because the same functions are used by Fill.
2. Adding mode branches may change compiler optimization and shader-codegen
   behavior around an already suspicious Fill delta-transmission path.
3. Reference output requires explicit `u_Output`, `u_Depth`, and
   `u_ScreenMotionVectors` exits that the current spine does not have.
4. Replacing reference full-sample NEE with realtime single-sample NEE would be
   a quality regression.
5. Changing diffuse-bounce classification could change converged images for
   transmissive diffuse materials.
6. Removing `RTXPTMaterialHitPayload` before replacing primary-depth and motion
   semantics would break the reference output contract.

## Acceptance Criteria

- Reference mode compiles with `PathPayload` and `PathState`.
- Reference mode does not bind or write stable-plane resources.
- Reference `AccumulatePathRadiance` and `CommitPixel` branches write raw HDR
  radiance to the existing reference output path.
- Reference depth and motion-vector outputs remain compatible with the current
  baseline.
- Reference closest-hit calls local `PathTracer::HandleHit`.
- Reference miss calls local `PathTracer::HandleMiss`.
- The flattened reference raygen loop is removed only after the unified path
  passes build and baseline validation.
- Full-sample NEE, diffuse-bounce classification, camera jitter, volume, and
  nested-dielectric semantics are preserved or explicitly documented as
  intentional estimator changes.
- Realtime opaque, diffuse, and metal behavior do not regress.
- Documentation clearly states which source differences were removed and which
  Diligent bridge differences intentionally remain.
