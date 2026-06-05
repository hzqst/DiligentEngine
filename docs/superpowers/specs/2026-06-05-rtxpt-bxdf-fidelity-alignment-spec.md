# RTXPT BxDF Fidelity Alignment Spec

## Summary

This spec defines the work required to align the current Diligent RTXPT BxDF
implementation with the original RTXPT-fork material and BSDF behavior.

The immediate visual signal is the bottom-row sphere mismatch in
`convergence-test.scene.json`. The material JSON asset data has already been
confirmed to match the original scene, and a separate first-priority spec covers
external material texture loading. That texture-loading fix should be completed
first so scene inputs are comparable, but it must not be treated as proof that
all BxDF differences are resolved.

The target strategy is source-level and behavior-level porting of the original
RTXPT-fork BxDF components, not continued local guessing around individual
colors, probabilities, or roughness thresholds. The Diligent implementation
should preserve Diligent shader include paths, binding constraints, and existing
raygen/closest-hit integration points while restoring the original BxDF
component model:

- `DiffuseReflectionFrostbite`
- `DiffuseTransmissionLambert`
- `SpecularReflectionMicrofacet`
- `SpecularReflectionTransmissionMicrofacet`
- `FalcorBSDF`
- active-lobe gating from `MaterialHeader`
- delta-lobe handling and `evalDeltaLobes`
- original `eval`, `evalPdf`, `sample`, `weight`, `pdf`, and `lobeP` semantics

## Problem Evidence

The current Diligent RTXPT port lives under:

```text
DiligentSamples/Samples/RTXPT
```

The reference RTXPT-fork source lives under:

```text
D:/RTXPT-fork
```

Known evidence:

- In `convergence-test.scene.json`, the bottom-row spheres differ visually
  between the Diligent port and the original RTXPT-fork reference.
- Scene assets and the ConvergenceTest material JSON have been checked against
  the original source and are not the primary material-data discrepancy.
- A separate external material texture-loading issue can still change scene
  inputs, so final BxDF visual validation must run after texture inputs are
  correct.
- Current Diligent `Rendering/Materials/BxDF.hlsli` is a compact rewrite. It
  contains standalone `StandardBSDFData`, lobe probability helpers,
  Fresnel/GGX helper functions, `EvalBSDF`, and `SampleBSDF`.
- The original RTXPT-fork `Rendering/Materials/BxDF.hlsli` is a much larger
  component implementation with named diffuse, transmission, microfacet, mixed
  `FalcorBSDF`, active-lobe, delta-lobe, and PSD-related behavior.
- Current Diligent `Scene/Material/MaterialData.hlsli` keeps only nested
  priority, active lobes, and thin-surface state in `MaterialHeader`; the
  original material header also carries `PSDExclude`,
  `PSDBlockMotionVectorsAtSurface`, and `PSDDominantDeltaLobeP1`.

## Source Anchors

Current Diligent anchors:

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`
  defines the simplified Diligent BxDF implementation and public
  `EvalBSDF`/`SampleBSDF` helpers used by reference and stable-plane paths.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli`
  defines Diligent lobe bit constants. The main bit layout already matches the
  RTXPT-fork layout.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli`
  defines the current simplified `MaterialHeader`.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
  shows reference raygen usage of `MakeStandardBSDFData`, `EvalBSDF`,
  `SampleBSDF`, lobe masks, sampled `pdf`, `lobeP`, firefly state, and nested
  dielectric scatter updates.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
  contains the stable-plane path-state integration, including
  `GenerateScatterRay`, `HandleNEE`, `AccumulatePathRadiance`, `CommitPixel`,
  and the current `BSDFSample` adapter around `SampleBSDF`.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
  converts hit material data into `SurfaceData`, `StablePlaneShadingData`, and
  `ActiveBSDF` for build/fill stable-plane modes.

Reference RTXPT-fork anchors:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli`
  defines the source BxDF components, mixed `FalcorBSDF`, active-lobe gating,
  sampling probabilities, mixture pdf rules, and `evalDeltaLobes`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/Microfacet.hlsli`
  defines GGX NDF, VNDF/BVNDF sampling/pdf helpers, Smith masking, and the
  specular multiple-scattering approximation used by the BxDF.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/Fresnel.hlsli`
  defines Schlick, generalized Schlick, dielectric Fresnel, and conductor
  Fresnel helpers.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/StandardBSDF.hlsli`
  wraps `FalcorBSDF` for `eval`, `sample`, `evalPdf`, `getLobes`,
  spec/diff estimates, reference sampling, and delta-lobe export.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/LobeType.hlsli`
  defines the reference lobe bit layout, including the named
  `NonDeltaReflection` and `NonDeltaTransmission` aliases.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerBridgeDonut.hlsli`
  builds `ShadingData`, sets material header flags, sets active lobes, creates
  `StandardBSDFData::make(...)`, and returns `SurfaceData`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Scene/Material/MaterialData.hlsli`
  defines the full reference `MaterialHeader` layout and PSD accessors.

## Current Diligent State

The current Diligent `BxDF.hlsli` is usable but not source-faithful. It folds
the material model into a shorter set of free functions:

- `StandardBSDFData` stores `N`, diffuse/specular/transmission colors,
  roughness, `alpha`, `eta`, metallic, diffuse/specular transmission, and
  `thinSurface`.
- `MakeStandardBSDFData(...)` derives diffuse/specular colors, GGX alpha,
  relative eta, and transmission factors directly from closest-hit payload data.
- `GetBSDFLobeProbabilities(...)` computes four mixture weights:
  diffuse reflection, diffuse transmission, specular reflection, and specular
  transmission.
- `EvalBSDF(...)` evaluates diffuse Frostbite-like reflection, diffuse
  transmission, specular reflection, and specular transmission in one function.
- `SampleBSDF(...)` samples one of those four families and returns `wi`,
  `weight`, `pdf`, `lobe`, and `lobeP`.

Important divergences from the reference:

- The file does not define the original component structs
  `DiffuseReflectionFrostbite`, `DiffuseTransmissionLambert`,
  `SpecularReflectionMicrofacet`, `SpecularReflectionTransmissionMicrofacet`,
  or `FalcorBSDF`.
- `MaterialHeader::getActiveLobes()` is not used by the current BxDF mixture.
  Sampling probabilities are driven by material values only.
- There is no `FalcorBSDF::getLobes(data)` behavior to derive possible lobes
  from material parameters and roughness/delta state.
- There is no `evalDeltaLobes` implementation matching RTXPT-fork. Stable-plane
  delta exploration and PSD behavior therefore cannot be considered faithful.
- Delta events are handled locally inside `SampleBSDF`, but the original
  convention that sampled delta events return `pdf = 0` after mixture sampling
  must be matched exactly where stable-plane code and MIS consume it.
- Diligent `LobeType.hlsli` matches the core bit values, but it omits the
  reference aliases `NonDeltaReflection = 0x03u` and
  `NonDeltaTransmission = 0x30u`.
- Current `MaterialHeader` exposes nested priority, active lobes, and
  thin-surface state but omits the PSD fields used by the reference bridge and
  BxDF delta path.

The current Diligent path-tracer integration already has useful adapter points.
Reference raygen directly calls `EvalBSDF` and `SampleBSDF`. Stable-plane fill
mode calls `GenerateScatterRay`, packages sampled data into `BSDFSample`, calls
`HandleNEE`, calls `StablePlanesOnScatter` after a valid scatter, and commits
denoiser radiance through stable-plane storage. The BxDF alignment should use
those integration points instead of rewriting the path-tracer state machine.

## Reference Behavior

The original RTXPT-fork BxDF model is component-based.

`DiffuseReflectionFrostbite`:

- evaluates Frostbite diffuse reflection as `evalWeight * 1/pi * cosTheta`;
- samples cosine hemisphere directions;
- returns the sampled lobe as `DiffuseReflection`;
- returns `weight = eval(...) / pdf`.

`DiffuseTransmissionLambert`:

- evaluates Lambertian diffuse transmission on the opposite hemisphere;
- samples cosine hemisphere directions and flips `z`;
- returns the sampled lobe as `DiffuseTransmission`;
- uses `-wo.z * 1/pi` pdf for transmission.

`SpecularReflectionMicrofacet`:

- uses active-lobe gating for `SpecularReflection` and `DeltaReflection`;
- treats `alpha == 0` as delta reflection when delta support is enabled;
- uses GGX sampling according to the configured mode, currently BVNDF in the
  reference configuration;
- evaluates pdf through `evalPdf(...)` instead of relying on duplicated sampling
  pdf code;
- uses Schlick Fresnel and the Turquin-style multi-scatter approximation.

`SpecularReflectionTransmissionMicrofacet`:

- uses active-lobe gating for reflection/transmission and delta variants;
- evaluates rough specular reflection and transmission through a shared
  microfacet interface;
- uses dielectric Fresnel to choose reflection versus transmission;
- treats thin-surface transmission as an eta-to-1 hack for transmission only;
- applies the transmission Jacobian in `evalPdf`;
- returns delta reflection/transmission with `pdf = 0`.

`FalcorBSDF`:

- owns the four lobe components and the four mixture probabilities;
- derives transmission albedo as either material transmission or its square
  root depending on thin-surface state;
- clamps roughness into delta behavior when `roughness^2 < kMinGGXAlpha`;
- reads `MaterialHeader::getActiveLobes()` to gate lobe probabilities;
- uses `MaterialHeader::isPSDExclude()` to suppress delta PSD export;
- computes probabilities for diffuse reflection, diffuse transmission,
  specular reflection, and specular reflection/transmission;
- after sampling a component, scales `weight`, `pdf`, and `lobeP` by the
  selected mixture probability and adds the reference mixture pdf terms for
  other compatible lobes;
- forces sampled delta-event `pdf` to `0.0`;
- exports two delta lobes in a stable order: transmission index `0`, reflection
  index `1`.

`StandardBSDF`:

- wraps the BxDF for world/local conversions through `ShadingData`;
- exposes `eval`, `sample`, `evalPdf`, `getLobes`, and `evalDeltaLobes`;
- stores `StandardBSDFData` with accessors for diffuse, specular, roughness,
  metallic, eta, transmission, diffuse transmission, and specular transmission.

The reference bridge builds material state in this order:

1. Create `ShadingData`.
2. Fill geometry, normal, tangent, bitangent, face normal, vertex normal, and
   front-facing state.
3. Create `MaterialHeader::make()`.
4. Set nested priority, thin-surface flag, PSD flags, and active lobes.
5. Compute diffuse/specular/roughness/metallic/transmission/eta from material
   inputs.
6. Create `StandardBSDF::make(StandardBSDFData::make(...))`.
7. Return `PathTracer::SurfaceData`.

## Target Behavior

Diligent RTXPT should produce BxDF behavior that matches RTXPT-fork for the
same material inputs, directions, and random samples.

Required behavior:

- Diligent must port or behaviorally mirror the original source components
  named above.
- `FalcorBSDF` mixture probabilities must match reference equations and
  active-lobe gating.
- `EvalBSDF` and `SampleBSDF` compatibility wrappers may remain, but their
  behavior must delegate to the reference-equivalent component model.
- `SampleBSDF` must preserve the current Diligent call shape used by reference
  raygen and stable-plane fill, returning `wi`, `weight`, `pdf`, `lobe`, and
  `lobeP`.
- `EvalBSDF` must evaluate the same mixture and pdf semantics as
  `StandardBSDF::eval`/`evalPdf`, including average specular semantics where
  the stable-plane path needs it.
- Active lobes must be available to BxDF construction. Default active lobes for
  current materials should be `LobeType::All` unless a material or compile-time
  constraint deliberately disables a family.
- Delta lobe export must be implemented where stable-plane logic can consume it.
- `MaterialHeader` or the Diligent equivalent material-state object must expose
  minimum compatible methods for `isPSDExclude`,
  `isPSDBlockMotionVectorsAtSurface`, and `getPSDDominantDeltaLobeP1`.
- The LobeType bit layout must remain compatible with RTXPT-fork:

```text
DiffuseReflection       0x01
SpecularReflection      0x02
DeltaReflection         0x04
DiffuseTransmission     0x10
SpecularTransmission    0x20
DeltaTransmission       0x40
Diffuse                 0x11
Specular                0x22
Delta                   0x44
NonDelta                0x33
Reflection              0x0f
Transmission            0xf0
NonDeltaReflection      0x03
NonDeltaTransmission    0x30
All                     0xff
```

## Implementation Scope

In scope:

- Port or adapt the reference BxDF component structs into Diligent shader code.
- Preserve Diligent include paths, HLSL guard style, resource binding
  constraints, and shader macro constraints.
- Keep current Diligent public call points working:
  `MakeStandardBSDFData`, `EvalBSDF`, `SampleBSDF`, `GenerateScatterRay`, and
  stable-plane `BSDFSample` packaging.
- Add minimum compatibility accessors/fields for active lobes and PSD metadata.
- Align diffuse, specular, transmission, GGX, Fresnel, mixture pdf, sampled
  weight, lobe probability, delta event, and lobe bit semantics.
- Add targeted validation hooks or tests where feasible.

Out of scope:

- Replacing the Diligent RTXPT path tracer with the Donut/NVRHI bridge.
- Re-porting unrelated RTXPT-fork renderer features.
- Treating the material texture-loading fix as a substitute for BxDF parity.
- Broad scene asset rewrites.
- Changing root build configuration or shader resource schemas beyond what the
  BxDF compatibility surface requires.

## Detailed Design

### D1 - Establish BxDF Source Parity Units

Create or refactor Diligent BxDF code around the same behavioral units as the
reference:

- Fresnel helpers:
  `evalFresnelSchlick`, `evalFresnelDielectric`, and any needed generalized or
  conductor variants.
- Microfacet helpers:
  `evalNdfGGX`, `evalPdfGGX_BVNDF`, `sampleGGX_BVNDF`, Smith masking, and
  `MultiScatterSpecularApprox`.
- Diffuse components:
  `DiffuseReflectionFrostbite` and `DiffuseTransmissionLambert`.
- Specular components:
  `SpecularReflectionMicrofacet` and
  `SpecularReflectionTransmissionMicrofacet`.
- Mixture:
  `FalcorBSDF`.
- Wrapper:
  a Diligent-compatible `StandardBSDF` facade or equivalent free-function
  facade that exactly preserves current call sites.

The implementation may keep all code in `BxDF.hlsli` if that best matches the
current Diligent include layout. If separate files are introduced, they must use
Diligent-relative include paths and avoid adding a Donut compatibility layer.

### D2 - Preserve Diligent Call-Site Compatibility

Current Diligent code calls:

```hlsl
StandardBSDFData bsdfData = MakeStandardBSDFData(...);
EvalBSDF(bsdfData, wo, wi, specProb, f, pdf);
SampleBSDF(bsdfData, wo, samples, wi, weight, pdf, lobe, lobeP);
```

Those wrappers should remain available. Internally they should construct a
reference-equivalent material header and shading frame, then delegate to
`FalcorBSDF` or the new `StandardBSDF` facade.

Stable-plane fill mode currently adapts the result into:

```hlsl
BSDFSample bs = MakeBSDFSample(lobe, pdf, lobeP, weight, wi);
StablePlanesOnScatter(path, bs, workingContext);
```

That adapter must keep working, but `deltaLobeIndex` must follow the reference
ordering: transmission delta lobe index `0`, reflection delta lobe index `1`.
The current Diligent heuristic must be audited against this convention.

### D3 - Restore Active-Lobe Gating

BxDF construction must consume active-lobe state instead of assuming all
material-value-positive lobes are active.

Minimum behavior:

- closest-hit material construction sets active lobes to `All` for ordinary
  RTXPT materials;
- compile-time constraints can mask transmission if transmission support is
  disabled;
- `FalcorBSDF` probability construction gates each lobe family with active
  lobe bits;
- `getLobes(data)` derives the possible lobe set from material values and
  roughness/delta state;
- Diligent wrappers pass the active lobe mask into the BxDF path, even when the
  source call site only has `StandardBSDFData`.

If the existing Diligent `StandardBSDFData` remains a free-function input, it
should gain a default active-lobe mask or be paired with a small header wrapper
so legacy call sites do not silently bypass gating.

### D4 - Restore MaterialHeader/PSD Minimum Compatibility

The original `MaterialHeader` packs:

- nested priority;
- active lobes;
- thin-surface flag;
- PSD exclude flag;
- PSD block-motion-vectors-at-surface flag;
- dominant delta lobe plus one.

Diligent must expose equivalent shader methods on the material-state type used
by BxDF and stable-plane code:

- `setPSDExclude(bool)` / `isPSDExclude()`;
- `setPSDBlockMotionVectorsAtSurface(bool)` /
  `isPSDBlockMotionVectorsAtSurface()`;
- `setPSDDominantDeltaLobeP1(uint)` / `getPSDDominantDeltaLobeP1()`.

If these methods already exist on a Diligent stable-plane-specific material
state, the implementation should map BxDF construction to that state instead of
duplicating state in a second incompatible header.

### D5 - Align Mixture Evaluation And Sampling Semantics

The reference `FalcorBSDF` mixture has observable semantics that must be
preserved:

- Diffuse reflection weight is multiplied by
  `(1 - specTrans) * (1 - diffTrans)`.
- Diffuse transmission weight is multiplied by
  `(1 - specTrans) * diffTrans`.
- Specular reflection weight is multiplied by `(1 - specTrans)`.
- Specular reflection/transmission weight is multiplied by `specTrans`.
- Sampling probabilities are normalized after active-lobe gating.
- Sampled component `pdf` and `lobeP` are multiplied by the chosen component
  probability.
- Mixture pdf terms from other non-zero compatible components are added exactly
  as in the reference code, including the reference omissions/comments where the
  omitted terms are always zero for the corresponding hemisphere.
- Delta samples return `pdf = 0.0` after sampling.
- `weight` must remain the throughput multiplier expected by both reference
  raygen and stable-plane path tracing.

The existing Diligent `EvalBSDF` signature includes `specProb`; that parameter
should either be removed only after all call sites are updated or kept as a
compatibility parameter that does not alter reference-equivalent behavior.

### D6 - Align Delta-Lobe And Stable-Plane Semantics

Implement reference-equivalent `evalDeltaLobes` behavior:

- return two delta lobe slots;
- initialize unused delta lobes to zero;
- compute `nonDeltaPart` from diffuse probabilities plus rough specular
  probabilities;
- return early when all specular delta probabilities are zero or PSD is
  excluded;
- export delta transmission at index `0` and delta reflection at index `1`;
- compute reflection/transmission directions and probabilities from the same
  Fresnel and eta rules as the sampled delta path;
- use thin-surface eta override for transmission only.

Stable-plane logic that uses `BSDFSample::getDeltaLobeIndex()` or equivalent
Diligent fields must match that index ordering.

### D7 - Align Shading Data Construction

`PathTracerClosestHit.rchit` should keep Diligent-native material table access,
but the resulting BxDF inputs should match the reference bridge:

- diffuse is `lerp(baseColor, 0, metallic)`;
- specular is `lerp(F0, baseColor, metallic)`;
- roughness is the original material roughness before alpha remapping;
- metallic is the material metalness;
- specular transmission is material transmission multiplied by
  `(1 - metallic)`;
- diffuse transmission is material diffuse transmission multiplied by
  `(1 - metallic)`;
- transmission color is base color;
- eta is outside IoR divided by material IoR when entering, and material IoR
  divided by outside IoR when exiting non-thin surfaces;
- thin-surface state and front-facing state are visible to the BxDF.

Existing Diligent nested dielectric code may update eta after surface loading;
the BxDF data layout must preserve that update path.

### D8 - Coordinate With Material Texture Loading

Validation order matters:

1. Complete the material-json external texture-loading fix first.
2. Use debug views or material dumps to confirm base color, metallic,
   roughness, normal, transmission, diffuse transmission, emission, and IoR
   inputs match the original scene.
3. Run BxDF micro tests and visual comparisons.
4. Attribute remaining bottom-row sphere differences to BxDF/shading behavior
   only after input textures and scalar factors are known-good.

The BxDF implementation should still proceed from source parity; it should not
wait for the texture fix to define the design.

## Testing/Validation

Build and shader validation:

- Build the RTXPT sample target with shaders enabled.
- Compile `PATH_TRACER_MODE_REFERENCE`.
- Compile `PATH_TRACER_MODE_BUILD_STABLE_PLANES`.
- Compile `PATH_TRACER_MODE_FILL_STABLE_PLANES`.
- Run on D3D12 first; repeat Vulkan when Vulkan RT is configured locally.

Suggested command:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Micro validation where feasible:

- CPU-side or shader micro tests for `evalFresnelDielectric` over entering,
  exiting, eta-equals-one, and total internal reflection cases.
- GGX BVNDF `sampleGGX_BVNDF` and `evalPdfGGX_BVNDF` deterministic checks for
  a fixed set of view vectors, roughness values, and samples.
- Diffuse Frostbite reflection checks for cosine hemisphere pdf and finite
  `weight = eval / pdf`.
- Diffuse transmission checks for opposite-hemisphere pdf and lobe ID.
- Specular reflection/transmission checks for rough and delta cases, including
  `pdf = 0` for delta samples.
- `FalcorBSDF` probability checks for opaque dielectric, metallic, diffuse
  transmission, specular transmission, thin surface, active-lobe masked, and
  roughness-below-delta-threshold materials.
- `evalDeltaLobes` checks for lobe count, index order, probabilities,
  `nonDeltaPart`, PSD exclusion, and thin-surface transmission direction.

Scene visual validation:

- Run `convergence-test.scene.json` after material texture inputs are confirmed.
- Compare the bottom-row materials against RTXPT-fork with the same camera,
  accumulation count, bounce settings, NEE settings, exposure, and tonemapping.
- Capture reference and Diligent images at matched sample counts.
- Compare HDR output numerically before tonemapping where possible; keep an
  LDR screenshot for human inspection.
- Use a thresholded difference image and per-sphere region statistics rather
  than relying only on visual memory.

Debug views and runtime sanity:

- Add or use debug views for sampled lobe ID, active lobe mask, roughness,
  metallic, diffuse transmission, specular transmission, eta, and thin surface.
- Expose a lobe-ID view that distinguishes diffuse reflection, diffuse
  transmission, specular reflection, specular transmission, delta reflection,
  and delta transmission.
- Verify `StablePlanesOnScatter` receives the expected lobe mask, `pdf`,
  `lobeP`, and delta index.
- Check for NaN, infinity, negative pdf, zero pdf on non-delta events, and
  extreme firefly regressions.
- Confirm energy sanity on white furnace-style scenes or equivalent closed
  material tests where available.

Reference comparison procedure:

1. Run RTXPT-fork at a fixed resolution, fixed random seed/sample index policy
   if available, and fixed sample count.
2. Run Diligent RTXPT with the same scene, material textures, camera, bounce
   count, NEE settings, and output format.
3. Save pre-tonemap HDR buffers if possible.
4. Compare per-material regions, especially the bottom-row spheres.
5. Inspect lobe debug output for the same camera rays.
6. Record any remaining mismatch as either input-data, BxDF, NEE/MIS,
   geometry-normal, or post-processing difference.

## Risks/Open Questions

- Texture loading may explain part of the visual mismatch. The mitigation is to
  finish texture loading first and validate BxDF after material inputs match.
- The original BxDF contains NVIDIA proprietary source. The implementation must
  follow the project's accepted RTXPT porting approach and keep Diligent-native
  file organization and include paths.
- The current Diligent reference path and stable-plane path share the simplified
  BxDF wrappers. The migration should preserve wrapper signatures until both
  paths are verified.
- Active-lobe and PSD fields may already exist in a Diligent stable-plane
  material state separate from `MaterialHeader`. The implementation should map
  to one canonical material-state path for BxDF construction.
- `specProb` currently appears in Diligent `EvalBSDF` call sites but the
  simplified implementation recomputes mixture probabilities internally. The
  port should decide whether to preserve it as a compatibility no-op or remove
  it with all call sites updated.
- Diligent's closest-hit tangent/normal construction may still differ from the
  Donut bridge. If BxDF parity passes micro tests but scene images differ, the
  next investigation should compare shading frames and normal-map sampling.
- Reference `FalcorBSDF` contains intentional comments around omitted mixture
  pdf terms. These should be ported as behavior, not "corrected" during the
  fidelity pass.

## Acceptance Criteria

This spec is satisfied when:

- Diligent BxDF code contains or behaviorally mirrors the reference
  `DiffuseReflectionFrostbite`, `DiffuseTransmissionLambert`,
  `SpecularReflectionMicrofacet`,
  `SpecularReflectionTransmissionMicrofacet`, and `FalcorBSDF` components.
- `EvalBSDF` and `SampleBSDF` wrappers remain usable by current Diligent
  reference and fill-stable-plane call sites.
- Active-lobe gating affects mixture probabilities exactly as in RTXPT-fork.
- The LobeType bit layout includes the RTXPT-fork-compatible aliases and keeps
  all existing bit values stable.
- Material header or material-state compatibility includes PSD exclude,
  PSD block-motion-vectors-at-surface, and dominant-delta-lobe fields.
- Sampled `weight`, `pdf`, `lobe`, and `lobeP` match reference semantics for
  diffuse, specular, transmission, and delta cases.
- Delta samples return `pdf = 0.0`, and delta lobe export uses transmission
  index `0` and reflection index `1`.
- `StablePlanesOnScatter` receives sampled lobe data compatible with the
  reference stable-plane logic.
- Shader compile/build succeeds for RTXPT reference, build-stable-planes, and
  fill-stable-planes modes.
- After material texture inputs are correct, the convergence-test bottom-row
  sphere comparison no longer shows BxDF-attributable differences beyond the
  documented tolerance.
- Validation reports include image comparison, lobe/debug-view checks,
  NaN/firefly sanity, and any remaining open mismatch category.
