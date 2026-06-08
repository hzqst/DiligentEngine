# RTXPT Reference Unified PathState Design

Date: 2026-06-08

## Purpose

Move the Diligent RTXPT reference ray tracing variant toward the upstream
RTXPT reference execution model without replacing the Diligent scene, material,
lighting, or resource binding bridge.

The target is option B from the design discussion:

- `PATH_TRACER_MODE_REFERENCE` uses `PathPayload` and `PathState`.
- Reference closest-hit and miss shaders update the path through local
  `PathTracer::HandleHit` and `PathTracer::HandleMiss`.
- Reference raygen drives the same `nextHit`, `postProcessHit`, and
  `CommitPixel` style loop used by the upstream reference path.
- Diligent-native bridge adaptation remains in place for material data,
  geometry fetches, environment maps, light resources, and render target
  bindings.

## Background

`docs/realtime_bxdf_diff.md` records the current difference:

- upstream reference mode still traces through unified
  `PathTracer::HandleHit`;
- local reference mode uses a flattened raygen loop;
- local reference closest-hit fills `RTXPTMaterialHitPayload`;
- local reference raygen reconstructs BSDF, NEE, MIS, nested dielectric, and
  path state behavior from payload fields.

The current local reference output is visually correct after recent BxDF work.
The remaining realtime smooth-glass issue is currently diagnosed as a realtime
Fill stable-plane routing or shader-codegen problem, not as a broad BxDF source
parity gap. This migration therefore must not expand the realtime regression
surface while it improves reference source structure.

## Goals

1. Make reference closest-hit call local `PathTracer::HandleHit`.
2. Make reference miss call local `PathTracer::HandleMiss`.
3. Make reference raygen carry `PathState` instead of local radiance,
   throughput, nested dielectric, MIS, and BSDF scatter variables.
4. Reuse the same local `SurfaceData`, `StablePlaneShadingData`, `ActiveBSDF`,
   and `GenerateScatterRay` path for reference, build, and fill variants.
5. Preserve the reference output contract: ray tracing writes raw HDR radiance
   to `u_Output`; accumulation, tone mapping, bloom, and presentation remain
   outside raygen.
6. Keep Diligent bridge/resource ownership intact.
7. Update source-diff documentation after the migration.

## Non-Goals

- Do not port upstream Donut scene bridge wholesale.
- Do not make Diligent `Bridge::loadSurface` byte-identical to
  `PathTracerBridgeDonut.hlsli`.
- Do not introduce RTXDI/ReSTIR final shading parity.
- Do not solve the current realtime smooth-glass Fill routing issue as part of
  this migration.
- Do not remove Diligent material, light, environment, or skinned-geometry
  resource adaptation.
- Do not shrink RT payload size in the first implementation pass unless all
  variants and shader reflection prove it is safe.

## Current State

Relevant local files:

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
  contains the flattened reference loop and a separate stable-plane
  `PathState` loop.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
  uses `RTXPTMaterialHitPayload` for reference and `PathPayload` for realtime
  variants.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerMiss.rmiss`
  and `PathTracerVisibilityMiss.rmiss` have the same payload split.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
  already has local `PathState` hit, miss, NEE, scatter, and commit logic.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
  already defines local `SurfaceData`, `StablePlaneShadingData`, and
  `ActiveBSDF`.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` sets one
  conservative `MaxPayloadSize` for all variants.

Important constraints:

- The old material-hit payload is 160 bytes.
- The packed realtime `PathPayload` is smaller, but using the larger
  `MaxPayloadSize` remains a conservative PSO setting.
- Existing reference visual correctness is a regression baseline.
- Existing realtime opaque, diffuse, and metal behavior must not regress.

## Target Architecture

All path tracing variants use a common shader-state spine:

```text
PathState
  -> PathPayload::pack
  -> TraceRay
  -> closest-hit or miss
  -> PathPayload::unpack
  -> PathTracer::HandleHit or PathTracer::HandleMiss
  -> PathPayload::pack
  -> raygen loop
  -> PathTracer::CommitPixel
```

Reference mode remains distinct only where it must:

- no stable-plane storage writes;
- no Build/Fill stable-plane split behavior;
- no realtime denoiser guide export except existing reference surface exports;
- no ReSTIR/RTXDI-only path branches.

The local material bridge continues to produce `SurfaceData`. The upstream
shape is matched at the execution level, while the Diligent bridge remains the
source of geometry, materials, textures, light metadata, and environment data.

## Reference Raygen Design

The reference branch of `PathTracerSample.rgen` should:

1. compute the pixel and camera ray;
2. initialize `PathState` with `PathTracer::EmptyPathInitialize`;
3. call `PathTracer::SetupPathPrimaryRay`;
4. create `PathTracer::WorkingContext`;
5. call `PathTracer::StartPixel`;
6. loop while the path is active:
   - call `nextHit(path, tMinMax, workingContext)`;
   - call `postProcessHit(path, workingContext)`;
   - call `ValidateNaNs(path, workingContext)` if validation is enabled;
7. call `PathTracer::CommitPixel(path, workingContext)`.

The reference raygen should stop owning these local state variables:

- `throughput`;
- `pathRadiance`;
- `fireflyFilterK`;
- `prevBsdfPdf`;
- `prevDidEnvNEE`;
- `prevDidEmissiveNEE`;
- `diffuseBounces`;
- `terminateAtNextEndpoint`;
- reference-local `InteriorList`.

Their behavior should be represented by `PathState` and helpers in
`PathTracer.hlsli`.

## Hit And Miss Shader Design

For `PATH_TRACER_MODE_REFERENCE`, `PathTracerClosestHit.rchit` should use
`PathPayload` as the active payload. The closest-hit body should:

1. unpack `PathPayload` to `PathState`;
2. build `SurfaceData` through local `LoadCurrentSurfaceData`;
3. call local `PathTracer::HandleHit`;
4. repack the updated state to `PathPayload`.

The reference closest-hit should stop filling `RTXPTMaterialHitPayload` as the
main execution result.

For reference miss shaders, miss handling should mirror realtime:

1. unpack `PathPayload`;
2. call local `PathTracer::HandleMiss`;
3. repack `PathPayload`.

Visibility rays should continue to produce the visibility behavior expected by
local NEE helpers. If a visibility shader currently relies on the material-hit
payload shape, the implementation plan must isolate and preserve that contract
before removing or bypassing `RTXPTMaterialHitPayload`.

## Surface And Material Data

`LoadCurrentSurfaceData` is the local equivalent of upstream
`Bridge::loadSurface` for Diligent resources. It should remain the single path
for reference and realtime surface construction.

The helper should continue to populate:

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

No new payload fields should be added for reference material data. Reference
material state should come from `SurfaceData`, not from a separate hit payload.

## NEE, MIS, And Emission

Reference NEE/MIS should move from the flattened raygen ownership to
`PathTracer::HandleHit` where possible.

The target ordering is:

1. update path travel;
2. load surface data;
3. apply volume transmittance through `PathState` throughput;
4. handle nested dielectrics;
5. accumulate emissive hit contribution with current local MIS support;
6. export surface data for reference guide/depth/motion outputs;
7. generate BSDF scatter through `GenerateScatterRay`;
8. evaluate direct-light and environment NEE through local Diligent helpers;
9. apply Russian roulette and termination through `PathState`;
10. commit final pixel through `CommitPixel`.

The implementation may keep Diligent-specific helper functions for emissive
triangle MIS, environment NEE, and direct-light sampling. The important design
constraint is that reference no longer owns a duplicate copy of those state
transitions in raygen.

## Payload And Pipeline Contract

The first implementation should keep
`RTXPTRayTracingPass.cpp::MaxPayloadSize = sizeof(float) * 40`.

Rationale:

- it already covers the old 160-byte reference payload;
- it also covers the smaller packed `PathPayload`;
- keeping it avoids a simultaneous PSO contract change while changing shader
  state flow;
- payload-size reduction can be a later cleanup after reference and realtime
  validation pass.

If the old `RTXPTMaterialHitPayload` becomes unused after migration, removing it
or shrinking payload size should be treated as a separate low-risk cleanup.

## Documentation Updates

After the code migration, update `docs/realtime_bxdf_diff.md`:

- replace the statement that local reference does not call
  `PathTracer::HandleHit`;
- record that local reference now uses local unified `PathState` and
  `HandleHit`;
- keep explicit notes that local bridge/resource adaptation still differs from
  upstream Donut `Bridge::loadSurface`;
- keep the current realtime smooth-glass diagnosis separate unless new evidence
  appears.

The mapping document may need a short note if the material-hit payload is no
longer part of the reference main path.

## Verification Plan

Static checks:

- `rg` confirms the reference path no longer uses `RTXPTMaterialHitPayload` as
  the main ray payload.
- `rg` confirms reference closest-hit and miss paths call
  `PathTracer::HandleHit` and `PathTracer::HandleMiss`.
- `rg` confirms reference raygen uses `PathState`, `nextHit`,
  `postProcessHit`, and `CommitPixel`.

Build checks:

- build the RTXPT sample target or the closest available CMake target;
- confirm reference, build-stable-planes, and fill-stable-planes shader
  variants compile and RT PSOs initialize.

Runtime checks:

- load `convergence-test.scene.json`;
- run reference mode and compare against the current known-good visual baseline;
- run realtime mode and confirm opaque, diffuse, and metal behavior do not
  regress;
- confirm the existing realtime smooth-glass issue is not widened into other
  material classes;
- record visual observations and any remaining known differences in docs.

## Risks

1. `PathState` reference-mode macro branches may not yet cover every behavior
   that the flattened raygen currently implements.
2. Moving MIS and NEE state into `PathState` may change noise patterns or
   convergence even when the final estimator remains valid.
3. Visibility rays may still rely on compatibility payload behavior and need a
   narrow preservation path.
4. The current realtime smooth-glass issue is likely codegen or Fill routing
   related; sharing more reference code with realtime must not obscure that
   diagnosis.
5. `CommitPixel` and surface export behavior must preserve the reference
   post-processing pipeline contract.

## Acceptance Criteria

- Reference mode compiles with `PathPayload` and `PathState`.
- Reference closest-hit calls local `PathTracer::HandleHit`.
- Reference miss calls local `PathTracer::HandleMiss`.
- Reference raygen no longer contains the duplicate flattened BxDF loop.
- Reference mode writes raw HDR radiance to the same downstream output target.
- `convergence-test.scene.json` reference rendering remains visually acceptable
  relative to the current baseline.
- Realtime opaque, diffuse, and metal behavior do not regress.
- Documentation clearly states which differences were removed and which
  Diligent bridge differences intentionally remain.

## Implementation Sequencing

The implementation plan should split this work into small checkpoints:

1. Switch reference shader payload selection to `PathPayload`.
2. Make reference closest-hit and miss call local hit/miss handlers.
3. Replace flattened reference raygen with the `PathState` loop.
4. Restore or adjust reference `CommitPixel` and surface export behavior.
5. Resolve compile errors and macro gaps.
6. Run static/build/runtime verification.
7. Update documentation.

Each checkpoint should be reversible and have a targeted verification command
or observation before moving to the next one.
