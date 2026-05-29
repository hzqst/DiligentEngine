# RTXPT Reference Path Tracer Completion Design

## Summary

This design defines the work required to bring the DiligentEngine `RTXPT` sample's **reference path tracer** to parity with the original RTXPT-fork's reference mode (`PATH_TRACER_MODE_REFERENCE`). It is a follow-on to the umbrella port design (`docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`) and the completed Phase 5.1–5.4 plans.

The current reference path tracer (after `docs/superpowers/plans/2026-05-29-rtxpt-phase5-4-reference-nee-mis.md`) is architecturally sound and unbiased: a raygen-driven N-bounce loop (`MaxRecursionDepth = 1`), shadow rays that reuse the radiance hit group + miss shader, NEE for analytic punctual lights, cosine-hemisphere environment NEE with power-heuristic MIS, a two-lobe (Lambert + GGX) BSDF, Russian roulette, and progressive accumulation. A full source comparison against RTXPT-fork reference mode identified a set of feature and convergence-quality gaps. This spec captures **every one of those gaps as a goal** and organizes them into ordered, runnable phases.

* The source code of RTXPT can be found at `D:/RTXPT-fork`. Reference-mode behavior referenced below is from `PATH_TRACER_MODE_REFERENCE` only (not the realtime stable-plane / RTXDI track).

## Relationship To The Umbrella Spec And Scope Boundary

The umbrella spec enumerates nine shader dependency layers under Phase 5 and reserves Phase 5.5–5.8 for the **realtime / advanced track** (stable planes, RTXDI/ReSTIR, NRD denoising, NVAPI/SER/OMM/DLSS). This spec is **orthogonal to that track**: it completes the *reference* path tracer (the Phase 5.4 deliverable) and does not advance the realtime track.

This spec therefore maps onto the umbrella spec's deferral lists:

- **"Initial light support" → "Basic emissive mesh extraction"** (currently unimplemented as a light source) — Phase R2.
- **"Later light support" → `LightsBaker` / Environment map baker / NEE feedback / light proxy generation** — Phase R3 / R4 port the *sampling math* (RIS/WRS, env importance sampling), not NVIDIA's full baker infrastructure.
- **"Later material support" → Nested dielectrics / Transmission / Advanced BSDF parameters** — Phase R5 / R6.

In-code, this spec resolves the existing structured markers:

- `TODO(RTXPT-Port Phase 5.3)` (transmission / nested dielectrics / `ALPHA_MODE_BLEND`) → Phase R6.
- `TODO(RTXPT-Port Phase 5.4)` (emissive area-light NEE + MIS, light RIS, HDR env-map MIS) → Phases R2 / R3 / R4.

## Confirmed Requirements And Decisions

- **One spec, phased.** A single "Reference Path Tracer Completion" spec; all gaps are goals, organized into ordered phases R1–R7. Each phase later spawns its own implementation plan (via the `writing-plans` skill) when scheduled.
- **Exact RTXPT reference-mode parity is the fidelity target**, including the convergence-quality gaps: low-discrepancy (Sobol/Owen-scrambled, stateless per-(pixel, vertex, sample)) sampling, bounded-VNDF GGX sampling, Frostbite/Disney energy-conserving diffuse, and multi-scatter specular energy compensation.
- **Scope excludes the realtime track** (RTXDI/ReSTIR, stable planes, NRD, DLSS/Streamline, OMM, SER) and the unported advanced BSDF lobes (sheen, clearcoat, anisotropy).
- **Runnable increments preserved.** Every phase must keep the `RTXPT` sample launching and rendering a valid result on both D3D12 and Vulkan, with new behavior toggleable and unfinished work behind `TODO(RTXPT-Port ...)` markers. This matches the umbrella spec's incremental-delivery and open-work-registry policy.
- **Unbiasedness is a hard invariant.** Toggling any new estimator off must converge to the same image as with it on (it only changes variance). This is the primary per-phase verification.
- **Port the math, not the framework.** Where RTXPT relies on heavy CPU bakers (`LightsBaker`, `EnvMapBaker`) or NVRHI/Donut abstractions, the goal is to reproduce the *sampling and weighting math* with a Diligent-native, minimal data path — not to port NVIDIA's baker classes verbatim.

## Goals

Each goal lists the current DiligentEngine state, the RTXPT-fork reference behavior it must match (with `file:line` anchors under `D:/RTXPT-fork/Rtxpt/Shaders/`), and a concrete success criterion. Goal IDs are stable handles for the implementation plans.

### Quick correctness & quality wins (Phase R1)

**G1 — Adaptive firefly filter.**
- Current: none. The accumulation buffer integrates raw HDR; the only nonlinearity is the ACES `saturate` at output (`DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`).
- RTXPT: an adaptive, per-path firefly filter, **on by default** in reference mode (`ReferenceFireflyFilterEnabled`, `SampleUI.h:128/212`; `RTXPT_FIREFLY_FILTER` guard). K-factor `ComputeNewScatterFireflyFilterK` shrinks as the path spreads (`PathTracer/PathTracerHelpers.hlsli:189-219`); applied to NEE radiance (`PathTracer/PathTracerNEE.hlsli:245-254`), surface emission (`PathTracer/PathTracer.hlsli:654-657`), and environment emission (`PathTracer/PathTracer.hlsli:477-481`). Soft cap: `signal *= threshold·K / average(signal)` when above the threshold.
- Success: a per-path filter K carried in the path state, applied to NEE / emissive / environment contributions, with a UI toggle + threshold; bright specular/NEE sparkle is suppressed and converged output is unchanged when the filter is disabled.

**G2 — NEE at all bounces (fix the `MaxNEEBounces` default).**
- Current: `MaxNEEBounces` defaults to `1` (`DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`), so `UseNEE = EnableNEE && Bounce < 1` runs NEE only at the primary hit (`RTXPTReference.rgen`). Because analytic lights are delta lights (the BSDF can never sample them), secondary bounces receive **zero** analytic-light contribution — indirect illumination from punctual/sky lights past bounce 0 is lost.
- RTXPT: NEE runs at every surface vertex up to the bounce limit; the bounce budget is governed by `HasFinishedSurfaceBounces` (total `getMaxBounceLimit` and `getMaxDiffuseBounceLimit`, `PathTracer/PathTracer.hlsli:40-45`, `PathTracerBridgeDonut.hlsli:525-541`), not by a separate NEE-only cap.
- Success: NEE default covers the full bounce budget (matching RTXPT); a shadowed surface lit by bounce light off a directly-lit surface receives that indirect light. The `MaxNEEBounces` control may remain as an optional performance/TDR clamp, but its default must not suppress indirect direct-lighting.

**G3 — Decorrelated low-discrepancy-ready seeding.**
- Current: a single forward `Hash32` chain seeded per (pixel, frame) and advanced sequentially across all bounces and dimensions (`DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTRandom.hlsli`). This correlates dimensions and bounces and converges slowly.
- RTXPT: stateless, decorrelated generators keyed by (pixel/pathID, vertexIndex, sampleIndex) (`Utils/StatelessSampleGenerators.hlsli`, `Utils/SampleGenerators.hlsli`); the sample index is `sampleBaseIndex + subSampleIndex`.
- Success (R1 portion): replace the single chained PRNG with a stateless per-(pixel, vertex, sample) generator so each bounce/dimension draws a decorrelated sequence. (Full Sobol/Owen low-discrepancy lands in G9 alongside the BSDF fidelity work.)

### Emissive-triangle area lights (Phase R2)

**G4 — Emissive-triangle area-light NEE + MIS.**
- Current: emissive surfaces are gathered **only** by chance BSDF hits, added with full weight (`PathRadiance += Throughput * Payload.Emission`, `RTXPTReference.rgen`). No emissive light list, no area-light NEE, no MIS. The closest-hit already reads the emissive material factor/texture (`RTXPTReference.rchit`).
- RTXPT: emissive geometry becomes `TriangleLight` entries; NEE uniformly samples the triangle by area and converts area→solid-angle pdf (`PolymorphicLight.hlsli:399-521`, `pdfAtoW`); a BSDF ray that hits an emissive triangle is MIS-weighted against the area-light NEE estimator via the triangle's `neeTriangleLightIndex` (`PathTracer/PathTracer.hlsli:592-634`, `ComputeBSDFMISForEmissiveTriangle`). The two estimators combine with the balance heuristic.
- Success: an emissive triangle light list is extracted (CPU side, in `RTXPTLights`/`RTXPTScene`) and bound to the shader; NEE samples emissive triangles with a shadow ray and area→solid-angle pdf; emissive BSDF hits are MIS-weighted against it; a scene lit primarily by emissive meshes converges dramatically faster and matches the BSDF-only converged result.

### Light importance sampling & units (Phase R3)

**G5 — Light importance sampling (RIS / weighted reservoir).**
- Current: uniform single-light selection, `index = min(uint(rand·count), count-1)`, contribution scaled by `×LightCount` (`RTXPTReference.rgen`).
- RTXPT: RIS over `NEECandidateSamples` candidates (default 5) producing `NEEFullSamples` visibility-tested samples (default 1) via a weighted reservoir sampler; global selection is power/importance-proportional from a proxy table (`PathTracer/PathTracerNEE.hlsli:88-161`, `GenerateLightSample`/`NEEWeightedReservoirSampler`; defaults `PathTracerShared.h:84-85`). RIS target ≈ unshadowed contribution × BSDF pdf.
- Success: an N-candidate RIS/WRS light selector with configurable candidate/full-sample counts; for many-light scenes, NEE noise drops substantially versus uniform selection; converged output matches uniform selection (unbiased).

**G6 — Photometric / shaped punctual-light units.**
- Current: raw `color · intensity` with inverse-square + squared-cone falloff and a user intensity slider (`DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli`); directional uses a large distance sentinel.
- RTXPT: punctual lights are sampled through `PolymorphicLight` with proper units and light shaping (`PolymorphicLight.hlsli` point/spot/directional; `evaluateLightShaping`). Point lights are modeled as small spheres with solid-angle sampling rather than pure deltas.
- Success: punctual-light radiance uses RTXPT-consistent units and cone/shaping falloff so intensities no longer require a manual scale slider to look correct; spot cones and ranges match RTXPT behavior.

### Environment-map IBL (Phase R4)

**G7 — HDR environment map with importance sampling + MIS.**
- Current: a hard-coded procedural sky gradient (`DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli`), sampled for NEE with a cosine hemisphere and power-heuristic MIS against the BSDF (`RTXPTReference.rgen`); the miss shader evaluates the same gradient (`RTXPTReference.rmiss`).
- RTXPT: an HDR environment map is loaded and importance-sampled. Reference-mode NEE samples it via an equal-area octahedral quadtree (`EnvironmentQuadLight`, `PolymorphicLight.hlsli:562-640`, solid-angle pdf `NodeDim²/4π`); the BSDF-side miss evaluates `EnvMap::EvalLocal` (`Lighting/EnvMap.hlsli:84-87`) with a diffuse-bounce MIP offset, and the two are combined with balance-heuristic MIS (`PathTracer/PathTracer.hlsli:454-472`, env-light lookup by direction). A direct MIP-descent importance sampler also exists (`EnvMap.hlsli:172-253`).
- Success: the sample can load an HDR environment map (procedural sky remains a fallback); env NEE importance-samples the map (equal-area or MIP-descent) instead of cosine-hemisphere; env↔BSDF MIS uses the importance-sampling pdf; a sky-lit scene converges faster and matches the BSDF-only converged result.

### BSDF fidelity & sampler (Phase R5)

**G8 — BSDF model parity (VNDF, Frostbite diffuse, multi-scatter).**
- Current: Lambert diffuse + GGX specular, two lobes, **NDF half-vector sampling**, height-correlated Smith visibility, Fresnel-luminance lobe-selection probability (`DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`). No multi-scatter compensation; plain Lambert diffuse.
- RTXPT: `FalcorBSDF` (`Rendering/Materials/BxDF.hlsli:709`) with **bounded-VNDF GGX sampling** (default `GGXSamplingBVNDF`, `BxDF.hlsli:43`, `Microfacet.hlsli`), **Frostbite/Disney energy-conserving diffuse** (default `DiffuseBrdfFrostbite`, `BxDFConfig.hlsli:25`, `BxDF.hlsli:157`), and **Turquin multi-scatter specular energy compensation** (`MultiScatterSpecularApprox`, `BxDF.hlsli:261`). Lobe-selection weights and the MIS-consistent mixture pdf are built in `FalcorBSDF::__init`/`sample`/`evalPdf` (`BxDF.hlsli:791-962`); `kMinGGXAlpha = 0.0064` collapses near-mirror lobes to delta events.
- Success: GGX specular uses bounded-VNDF sampling (with matching `evalPdf`); diffuse uses the energy-conserving Frostbite model; multi-scatter compensation restores energy at high roughness/metalness; lobe selection and pdf stay MIS-consistent. Variance and MIS balance match RTXPT to within sampler differences; converged output is unchanged versus the current model (energy-compensation aside).

**G9 — Low-discrepancy (Sobol/Owen) sampling.**
- Current: hash-chain PRNG (see G3).
- RTXPT: Sobol/Owen-scrambled low-discrepancy sequences for the first diffuse bounce, uniform thereafter (`PathTracer/PathTracer.hlsli:360-369`, `Utils/SampleGenerators.hlsli`).
- Success: low-discrepancy sequences drive primary-ray and early-bounce sampling, switching to uniform after the configured diffuse-bounce count; primary-effect noise (AA, first-bounce GI) converges measurably faster; remains unbiased.

### Transmission & volumes (Phase R6)

**G10 — Transmission, nested dielectrics, and volume absorption.**
- Current: opaque single-sided shading only; the shading normal is flipped to face the camera and transmission is skipped (`RTXPTReference.rchit`); the any-hit handles only `ALPHA_MODE_MASK` (`RTXPTReference.rahit`); BSDF has no transmission lobe.
- RTXPT: `FalcorBSDF` adds rough-dielectric specular reflection+transmission (`SpecularReflectionTransmissionMicrofacet`, `BxDF.hlsli:385-598`) with Fresnel-driven reflect/refract and the correct refraction Jacobian; an `InteriorList` priority stack handles nested dielectrics (`Rendering/Materials/InteriorList.hlsli`, `PathTracerNestedDielectrics.hlsli`); homogeneous volume absorption (Beer-Lambert) is applied while traversing a medium (`PathTracer/PathTracer.hlsli:535-546`, `Rendering/Volumes/HomogeneousVolumeSampler.hlsli`). glTF `KHR_materials_transmission` + per-material IoR feed the BSDF (`PathTracerBridgeDonut.hlsli:742-792`).
- Success: glass/refractive materials transmit and refract correctly; nested dielectrics resolve via the interior-list priority stack; per-medium absorption tints transmitted paths over distance. Resolves the `Phase 5.3` markers. (`ALPHA_MODE_BLEND` stochastic transparency may be included here or kept as a sub-item.)

### Polish (Phase R7, optional)

**G11 — Shadow/AA polish.**
- Current: shadow-ray origin offset along the *shading* normal; no grazing-angle shadow fadeout; no depth of field.
- RTXPT: shadow-ray origin via `ComputeRayOrigin` along the *face* normal (`PathTracer/PathTracerNEE.hlsli:166-182`); grazing-angle shadow fadeout (`ComputeLowGrazingAngleFalloff`, `PathTracerHelpers.hlsli`); thin-lens camera with aperture for DoF (`PathTracerBridgeDonut.hlsli` `ComputeRayThinlens`).
- Success: reduced shadow acne/leak on normal-mapped surfaces via face-normal offset; optional shadow-terminator fadeout; optional thin-lens DoF control.

## Non-Goals

- The realtime / advanced track: stable planes, RTXDI/ReSTIR DI+GI, ReGIR, NRD denoising, DLSS/DLSS-RR/Streamline, TAA. (Umbrella spec Phase 5.5–5.8.)
- A verbatim port of NVIDIA's CPU bakers (`LightsBaker`, `EnvMapBaker`, OMM baker) or the NVRHI/Donut bridge layers. Only the GPU-side sampling/weighting math is reproduced, on a Diligent-native data path.
- Advanced BSDF lobes not used by reference shading: sheen, clearcoat, anisotropy, Oren-Nayar.
- SER (Shader Execution Reordering), OMM (Opacity Micro-Maps), and NVAPI shader extensions.
- Moving tone mapping out of raygen into a dedicated post-process chain (tracked separately as `Phase 6`).

## Phase Design

Phases are ordered by dependency and risk: cheap, high-visibility wins first; then a light list (prerequisite for emissive MIS and RIS); then environment IBL; then BSDF/sampler fidelity; then the largest material change (transmission); then optional polish. Each phase is an independently runnable increment and gets its own implementation plan when scheduled.

### Phase R1: Quick Correctness & Quality Wins
- Goals: G1 (firefly filter), G2 (NEE at all bounces), G3 (decorrelated seeding).
- Touches: `RTXPTReference.rgen`, `RTXPTRandom.hlsli`, `RTXPTBSDF.hlsli`/path-state helpers, `RTXPTShaderShared.hlsli` + `RTXPTSample.hpp/.cpp` (settings/UI).
- Runnable milestone: sample renders with visibly less sparkle and correct indirect direct-lighting; firefly filter and NEE-bounce controls in the UI; toggling firefly off converges to the same image.

### Phase R2: Emissive-Triangle Area Lights
- Goal: G4.
- Touches: `RTXPTLights` / `RTXPTScene` (emissive triangle extraction + buffer), `RTXPTSceneBridge.hlsli` (light-list access), `RTXPTReference.rgen` (area-light NEE), `RTXPTReference.rchit` (per-hit triangle-light identity for MIS), settings/UI, RT-pass bindings.
- Runnable milestone: emissive-mesh-lit scenes converge fast with correct shadows; emissive BSDF hits MIS-weighted; converged image matches BSDF-only.

### Phase R3: Light Importance Sampling & Units
- Goals: G5 (RIS/WRS), G6 (photometric/shaped units).
- Touches: a new light-sampling/RIS header, `RTXPTLightSampling.hlsli`, `RTXPTReference.rgen`, light buffer/proxy data in `RTXPTLights`, settings/UI.
- Runnable milestone: many-light scenes converge faster; intensities correct without a manual scale; unbiased versus uniform selection.

### Phase R4: HDR Environment-Map IBL
- Goal: G7.
- Touches: env-map loading (`RTXPTScene`/assets), an env importance-sampling header, `RTXPTEnvironment.hlsli`, `RTXPTReference.rmiss`, `RTXPTReference.rgen`, settings/UI, RT-pass bindings.
- Runnable milestone: loads an HDR env map (procedural sky fallback retained); importance-sampled env NEE + MIS; sky-lit scenes converge faster and match BSDF-only.

### Phase R5: BSDF Fidelity & Low-Discrepancy Sampler
- Goals: G8 (VNDF/Frostbite/multi-scatter), G9 (Sobol/Owen).
- Touches: `RTXPTBSDF.hlsli` (or a refactor into microfacet/diffuse/sampler sub-headers), `RTXPTRandom.hlsli`/sampler header, `RTXPTReference.rgen`.
- Runnable milestone: improved energy accuracy at high roughness/metalness and faster primary-effect convergence; pdfs stay MIS-consistent; unbiased.

### Phase R6: Transmission, Nested Dielectrics & Volumes
- Goal: G10.
- Touches: BSDF transmission lobe, interior-list + volume headers, `RTXPTReference.rchit`/`rahit` (two-sided shading, blend mode), payload growth, material data (`RTXPTMaterials` transmission/IoR), settings/UI, RT-pass payload size.
- Runnable milestone: glass/refractive materials render correctly; nested dielectrics and absorption work; resolves the `Phase 5.3` markers.

### Phase R7: Shadow/AA Polish (optional)
- Goal: G11.
- Touches: `RTXPTReference.rgen` (shadow-ray origin, fadeout), camera ray generation (DoF), settings/UI.
- Runnable milestone: fewer shadow artifacts on normal-mapped surfaces; optional shadow-terminator fadeout and thin-lens DoF.

## Cross-Cutting Contracts

These shared contracts span multiple phases and must be kept explicit (each phase that touches one calls it out and updates the matching `static_assert`s / bindings):

- **Settings & frame-constants layout.** `RTXPTPathTracerSettings` (currently 48 bytes) is mirrored in C++ (`RTXPTSample.hpp`, `static_assert(sizeof == 48)`) and HLSL (`RTXPTShaderShared.hlsli`); `RTXPTFrameConstants` carries it. New per-phase toggles/parameters grow this struct in lockstep with both `static_assert`s.
- **Payload size.** The reference payload `RTXPTPathTracerPayload` is 64 bytes; `MaxPayloadSize` is set in `RTXPTRayTracingPass`. Per-path state that lives entirely in the raygen loop (throughput, MIS bookkeeping, the firefly K-factor of G1, the nested-dielectric interior list of G10) stays raygen-local and does **not** grow the payload. Transmission (G10) adds per-hit surface fields the closest-hit must return (IoR, transmission color, specular-transmission), which likely requires payload growth — adjust `MaxPayloadSize` and the `static_assert`/comment together when it does.
- **Light-buffer contract.** `RTXPTLights` uploads `StructuredBuffer<RTXPTLightData>` and a CPU `AnalyticLightCount`; `Bridge::GetLightCount()` returns `AnalyticLightCount` so the binding-safety dummy light is excluded from sampling. Emissive (G4) and proxy/RIS (G5) data extend this contract; the dummy-light invariant and per-stage STATIC binding rules (a STATIC variable is bound per stage and only for stages whose compiled shader references it) must be preserved.
- **Hit group / SBT reuse.** Shadow/visibility rays continue to reuse hit group 0 + miss 0 with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`; no new ray type unless a phase explicitly justifies it. `MaxRecursionDepth` stays 1 (all rays issued from raygen).
- **Backends.** Every phase keeps D3D12 and Vulkan first-class; backend-specific paths require explicit capability checks (umbrella spec's capability model).

## Verification Strategy

Per phase:

1. **Unbiasedness (primary).** Toggle the new estimator/feature off and confirm the accumulated image converges to the same result as with it on (only variance/noise differs). For MIS additions, confirm no double-counting (the sum of estimator weights for a shared direction stays ≈ 1).
2. **Convergence/visual delta.** Record the expected qualitative change (e.g. "emissive-lit scene converges in N× fewer samples", "less specular sparkle", "glass refracts").
3. **Layout guards.** C++ `static_assert`s for any grown settings/payload structs compile.
4. **Runnable on both backends.** Sample launches and renders on D3D12 and Vulkan; unsupported sub-features compile out / disable with a visible reason.
5. **Build/runtime steps are listed for explicit user request only** (per the workspace rule: do not auto-run build/test/runtime commands).

## Open-Work / TODO Marker Policy

Unfinished or deferred work stays behind structured `// TODO(RTXPT-Port Phase R<n>): ...` markers, consistent with the umbrella spec's open-work registry. When a phase lands, it re-targets or removes the markers it resolves (e.g. R6 resolves the `Phase 5.3` transmission markers; R2/R3/R4 resolve the `Phase 5.4` markers) and leaves new markers for any sub-item it intentionally defers.
