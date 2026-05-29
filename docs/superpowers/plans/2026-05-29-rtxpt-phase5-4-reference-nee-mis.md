# RTXPT Phase 5.4 Reference Direct Lighting (NEE + MIS) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the reference path tracer real direct lighting — add Next-Event Estimation (NEE) with shadow/visibility rays over the already-loaded analytic punctual lights (directional / point / spot), plus environment (procedural-sky) NEE combined with BSDF sampling through multiple-importance sampling (MIS, power heuristic) — so directly-lit surfaces, shadows, and sky lighting converge far faster than the current BSDF-only path.

**Architecture:** The reference path tracer already traces every bounce from **raygen** (closest-hit / miss only fill a payload; `MaxRecursionDepth = 1`). This plan keeps that structure: NEE is done entirely in raygen. The key insight is that a **visibility (shadow) ray needs no new ray type, PSO, SBT, or payload** — it reuses the existing radiance hit group and miss shader with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`. The miss shader clears `HitFlag`, so a ray that escapes = "visible"; alpha-masked geometry still casts correct shadows because the existing any-hit runs the alpha test. The only C++ change is **moving the `g_Lights` binding from the miss stage to the raygen stage** (raygen now samples lights; the miss shader no longer needs them once its sun-disk hack is removed). New per-frame settings (enable flags + intensity scales) ride in a grown `RTXPTPathTracerSettings`. Two small standalone HLSL headers are added — `RTXPTEnvironment.hlsli` (the procedural sky, shared by raygen NEE and the miss shader so both estimators see identical radiance) and `RTXPTLightSampling.hlsli` (pure punctual-light decode math) — plus three math helpers in `RTXPTBSDF.hlsli` (`RTXPTPowerHeuristic`, `RTXPTSpecularProbability`, reused by both the sampler and NEE so their pdfs agree).

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs (`AddVariable`, `GetStaticVariableByName`/`Set` — confirmed per-stage in `Tutorial21_RayTracing`), HLSL 6.5 ray tracing shaders compiled by DXC (`TraceRay` with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`, cosine-hemisphere environment sampling, power-heuristic MIS, Cook-Torrance GGX from Phase 5.3b), `DiligentTools` GLTF punctual lights (`RTXPTLightData`: directional / point / spot already uploaded by `RTXPTLights`), Dear ImGui.

---

## Scope Note: Phase 5 Sub-Plan Series And Renumbering

Phase 5 from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md` enumerates nine shader dependency layers. Completed plans:

- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-1-shader-bridge.md` — layers 2-3 (shared declarations and scene/material bridge).
- `docs/superpowers/plans/2026-05-28-rtxpt-phase5-2-reference-path-tracer.md` — layer 4 (reference path tracer core: BSDF-only path tracing + accumulation).
- `docs/superpowers/plans/2026-05-29-rtxpt-phase5-3-material-alpha-anyhit.md` — core of layer 5 (textured base color + emissive, alpha-test any-hit).
- `docs/superpowers/plans/2026-05-29-rtxpt-phase5-3b-ggx-bsdf-normal-maps.md` — deferred layer-5 core (metallic-roughness GGX BSDF, normal mapping, Russian roulette).

This plan (**Phase 5.4: Reference Direct Lighting**) completes the reference path tracer's lighting by implementing the spec's **"Initial light support"** (`Resource And Asset Strategy`): *Environment light, Directional light, Simple analytic lights*. It is the prerequisite the advanced realtime track (RTXDI/ReSTIR) builds on, and matches RTXPT reference mode's `HandleNEE` / `HandleHit` / `HandleMiss` direct-lighting structure (RTXPT reference mode does NEE independently of the realtime stable-plane/RTXDI track).

**Numbering note (this plan renumbers the remaining sub-phases).** The earlier in-code TODOs labelled NEE as "Phase 5.5", but NEE is direct-lighting for the reference path tracer, not the realtime/RTXDI track. This plan promotes direct lighting to its own **Phase 5.4** stage and re-targets / removes those "Phase 5.5" NEE markers. The realtime/advanced track shifts down by one:

- Phase 5.5: Stable planes and realtime mode (layer 6) — its own plan in a later session.
- Phase 5.6: RTXDI / ReSTIR shader bridge and passes (layer 7).
- Phase 5.7: NRD, denoising guides, post-process (layer 8).
- Phase 5.8: NVAPI, SER, OMM, DLSS-related shader variants (layer 9).

This plan **defers** (each kept as a structured `TODO(RTXPT-Port Phase 5.4)` marker in code, to be resolved by a later lighting refinement plan):

- Emissive-triangle **area-light NEE + MIS** (emissive surfaces are still gathered by BSDF sampling only — there is no emissive light list yet; that needs RTXPT's `LightsBaker`).
- HDR **environment-map IBL** with environment-map importance sampling (the environment is still the procedural sky; env NEE uses a cosine-hemisphere sampler).
- **Light importance sampling / RIS** for many lights (this plan uniformly selects one analytic light per NEE call).
- Proper **photometric unit handling** for punctual lights (this plan uses raw `color * intensity` with inverse-square / cone falloff and a user-tunable intensity scale).

## Baseline

Current state of `DiligentSamples/Samples/RTXPT` (submodule `HEAD = 87002ff1 fix(rtxpt): handle normal-map handedness and layout guards`, top-level `HEAD = 7a421b0`):

- `RTXPTReference.rgen` traces an N-bounce loop from raygen: per bounce it issues one `TraceRay(g_TLAS, RAY_FLAG_NONE, 0xFF, 0, 1, 0, Ray, Payload)`, accumulates `Payload.Emission` (sky on miss, emissive on hit) into `PathRadiance`, builds an `RTXPTSurface` from the payload, importance-samples the GGX BSDF (`RTXPTSampleBSDF`), applies Russian roulette past `MinBounces`, and blends into RGBA32F `g_AccumColor` (tone-mapped via `ToneMapACES` into RGBA8 `g_OutputColor`). It declares `g_TLAS`, `g_OutputColor`, `g_AccumColor` and includes `RTXPTSceneBridge.hlsli` + `RTXPTRandom.hlsli` + `RTXPTBSDF.hlsli`. **There is no direct lighting / NEE: the analytic lights contribute nothing.** Trailing markers: `TODO(... Phase 5.3)` transmission, `TODO(... Phase 5.5)` NEE, `TODO(... Phase 6)` tone mapping.
- `RTXPTReference.rmiss` includes `RTXPTSceneBridge.hlsli`, writes a procedural sky gradient into `Payload.Emission`, sets `Payload.HitFlag = 0u`, and **adds a placeholder sun-disk tint** read from the first directional light via `Bridge::GetLight(0)` (this is the only consumer of `g_Lights` outside raygen-to-be). Trailing `TODO(... Phase 5.5)` sun-disk marker.
- `RTXPTReference.rchit` fills the payload (world pos/normal, normal-mapped, base color, metallic/roughness, emissive). Trailing `TODO(... Phase 5.5)` NEE marker. **Unchanged by this plan except its TODO marker** (NEE lives in raygen, not closest-hit).
- `RTXPTReference.rahit` is the alpha-test any-hit (compiled into the hit group only when material textures are enabled): it `IgnoreHit()`s alpha-masked texels and otherwise accepts. **Unchanged by this plan** — it already makes shadow rays alpha-correct.
- `RTXPTSceneBridge.hlsli` declares the globals `g_FrameConstants`, `g_SubInstanceData`, `g_Lights`, `g_VertexBuffer`, `g_IndexBuffer`, the hit helpers (under `RTXPT_ENABLE_HIT_BRIDGE`), and — **outside that guard, so callable from raygen** — `Bridge::GetLightCount()` and `Bridge::GetLight(uint)`.
- `RTXPTLights.cpp` uploads a `StructuredBuffer<RTXPTLightData>` (64 bytes/entry): `ColorIntensity` (rgb + intensity in `.a`), `PositionRange` (xyz + range in `.w`), `DirectionType` (xyz + type in `.w`; **`LightTypeToShaderValue`: 0 = Directional, 1 = Point, 2 = Spot, -1 = disabled**), `SpotAngles` (inner `.x`, outer `.y`, radians). Direction convention (from `MakeLightData` and the rmiss sun-disk): `DirectionType.xyz` is the light **emission** direction, so `-DirectionType.xyz` points from a surface **toward** the light. When the scene has no lights, one disabled default light (type `-1`) is uploaded so the SRV is never null.
- `RTXPTRayTracingPass.cpp` builds the RT PSO. `g_FrameConstants` + `g_TLAS` are STATIC for `SHADER_TYPE_RAY_GEN`; `g_Materials` / `g_SubInstanceData` / `g_VertexBuffer` / `g_IndexBuffer` for the hit stages; **`g_Lights` is STATIC for `SHADER_TYPE_RAY_MISS`** (declared at the `AddVariable` chain and bound via `SetStatic(SHADER_TYPE_RAY_MISS, "g_Lights", pLightsView)`). `MaxRecursionDepth = 1`, `MaxPayloadSize = sizeof(float)*16` (64 bytes), `MaxAttributeSize = sizeof(float)*2`. SBT: `BindRayGenShader("Main")`, `BindMissShader("PrimaryMiss", 0)`, `BindHitGroupForTLAS(m_TLAS, 0, "PrimaryHit")`. Diligent binds STATIC ray-tracing variables **per stage** — confirmed by `DiligentSamples/Tutorials/Tutorial21_RayTracing/src/Tutorial21_RayTracing.cpp` (it calls `GetStaticVariableByName(stage, "g_ConstantsCB")->Set(...)` once for each of raygen/miss/closest-hit). A STATIC variable can only be fetched for a stage whose compiled shader actually references it.
- `RTXPTShaderShared.hlsli` mirrors `RTXPTPathTracerSettings` (currently `MaxBounces`, `AccumulationFrame`, `ResetAccumulation`, `MinBounces` — 16 bytes) embedded in `RTXPTFrameConstants` (176 bytes), and `RTXPTPathTracerPayload` (64 bytes: `WorldPos`, `HitDistance`, `WorldNormal`, `HitFlag`, `BaseColor`, `Metallic`, `Emission`, `Roughness`).
- `RTXPTBSDF.hlsli` provides `RTXPT_PI`, `RTXPT_INV_PI`, `RTXPT_MIN_ROUGHNESS`, `RTXPTSurface`, `RTXPTMakeSurface`, `RTXPTFresnelSchlick`, `RTXPTDistributionGGX`, `RTXPTVisibilitySmithGGX`, `RTXPTLuminance`, `RTXPTEvalBSDF(S, Wo, Wi, SpecProb, out FTimesNoL, out Pdf)`, `RTXPTSampleBSDF(S, Wo, inout Rng, out Wi, out Weight, out Pdf)`. The sampler computes its lobe-selection probability (`SpecProb`) inline — this plan factors it out so NEE can reproduce the exact same BSDF pdf for MIS.
- `RTXPTRandom.hlsli` provides `RTXPTRandom`, `NextFloat`, `NextFloat2`, `BuildOrthonormalBasis`, `SampleCosineHemisphere(Rand, Normal, out Pdf)` (pdf = cos/PI).
- `RTXPTSample.hpp` owns `RTXPTPathTracerSettings` (C++ mirror, `static_assert(sizeof == 16)`) and `RTXPTFrameConstants` (`static_assert(sizeof == 176)`), plus members `m_MaxBounces = 4`, `m_MinBounces = 3`. `UpdateFrameConstants` fills `MaxBounces` / `AccumulationFrame` / `ResetAccumulation` / `MinBounces`. `UpdateUI` has "Max bounces" / "Min bounces (RR start)" sliders + a "Reset accumulation" button, and a `TODO(... Phase 5.5)` NEE text line.
- `CMakeLists.txt` registers the shader sources in order: `RTXPTBSDF.hlsli`, then `RTXPTReference.rgen`, etc.

This plan assumes the top-level repository starts clean and the `DiligentSamples` submodule is at the state above.

---

## Scope

This plan implements:

- Grow `RTXPTPathTracerSettings` (C++ + HLSL mirror) from 16 to **32 bytes**: add `EnableNEE`, `EnableEnvNEE` (uint), `EnvIntensity`, `LightIntensityScale` (float). `RTXPTFrameConstants` grows 176 → **192 bytes**. Update both `static_assert`s.
- Add `RTXPTEnvironment.hlsli` (`RTXPTEvalSky`) — the procedural sky factored out so the miss shader and raygen environment NEE evaluate identical radiance. Register in `CMakeLists.txt`.
- Add `RTXPTLightSampling.hlsli` (`RTXPTLightSample` + `RTXPTEvalAnalyticLight`) — pure decode of directional / point / spot punctual lights into `{ Wi, Distance, Radiance, Valid }`. Register in `CMakeLists.txt`.
- Extend `RTXPTBSDF.hlsli`: `RTXPTPowerHeuristic(pdfA, pdfB)` (β=2), `RTXPTSpecularProbability(S, Wo)` (extracted from the sampler), and refactor `RTXPTSampleBSDF` to call it (behavior identical).
- Atomic NEE landing: rewrite `RTXPTReference.rgen` to add a visibility helper, analytic-light NEE (no MIS — delta lights), environment NEE with MIS, and BSDF-side environment MIS (weight the BSDF-sampled sky hit); rewrite `RTXPTReference.rmiss` to use `RTXPTEvalSky` and drop the sun-disk / `g_Lights` usage; **move the `g_Lights` STATIC binding from `SHADER_TYPE_RAY_MISS` to `SHADER_TYPE_RAY_GEN`** in `RTXPTRayTracingPass.cpp`; re-target the resolved `Phase 5.5` markers in `rgen` / `rmiss` / `rchit`.
- Wire the new settings into `RTXPTSample` (members, `UpdateFrameConstants`, UI checkboxes + sliders) and re-target the sample TODO.

This plan intentionally does **not**:

- Add emissive-triangle area-light NEE, light importance sampling / RIS, or HDR environment-map IBL (kept as `Phase 5.4` deferral markers).
- Change the RT-pass payload size, `MaxRecursionDepth`, `MaxAttributeSize`, the hit-group / SBT layout, or add a second ray type — shadow rays reuse hit group 0 + miss 0.
- Touch `RTXPTReference.rchit` / `RTXPTReference.rahit` logic (only the `rchit` TODO comment is re-targeted).
- Implement transmission / nested dielectrics / `ALPHA_MODE_BLEND` (still `Phase 5.3` markers).
- Run automated builds or runtime execution; build/runtime steps are listed for explicit user request only.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` — grow `RTXPTPathTracerSettings` (32 B) + `RTXPTFrameConstants` (192 B); add NEE members.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli` — mirror the 32-byte settings struct.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli` — `RTXPTEvalSky`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli` — `RTXPTLightSample` + `RTXPTEvalAnalyticLight`.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` — register the two new headers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli` — add `RTXPTPowerHeuristic` / `RTXPTSpecularProbability`; refactor `RTXPTSampleBSDF`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen` — NEE + MIS + shadow rays.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss` — use `RTXPTEvalSky`, drop sun disk / `g_Lights`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit` — re-target the resolved NEE TODO comment only.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` — move `g_Lights` STATIC binding from miss to raygen.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` — wire NEE settings + UI; re-target sample TODO.

---

### Task 0: Phase 5.4 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [ ] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected: branch line `## RTXPT...origin/RTXPT` and no staged/modified files under `DiligentSamples/Samples/RTXPT` or `docs/superpowers/plans`. Unrelated files may be left untouched.

- [ ] **Step 2: Confirm DiligentSamples Phase 5.3b state**

Run:

```powershell
git -C DiligentSamples status --short --branch
git -C DiligentSamples log --oneline -n 3
```

Expected: clean working tree; the most recent commit is `87002ff1 fix(rtxpt): handle normal-map handedness and layout guards`.

- [ ] **Step 3: Confirm the new headers do not yet exist**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli
```

Expected: `False` for both. If either exists, inspect it before overwriting and preserve unrelated work.

- [ ] **Step 4: Confirm the resolved-by-this-plan TODO markers are present**

Run:

```powershell
rg -n "Phase 5.5" DiligentSamples/Samples/RTXPT
```

Expected exactly four matches, which this plan re-targets to `Phase 5.4`:

```text
assets/shaders/RTXPTReference.rchit  : Add NEE shadow rays toward analytic and environment lights.
assets/shaders/RTXPTReference.rgen   : Add explicit light sampling and MIS once the lighting baker is restored.
assets/shaders/RTXPTReference.rmiss  : Replace the placeholder sun disk with environment map / NEE-driven sun sampling ...
src/RTXPTSample.cpp                  : add explicit light sampling and MIS once the lighting baker is restored.
```

---

### Task 1: Grow The Path-Tracer Settings With NEE Controls

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`

Context: the settings sub-struct grows from 4 to 8 fields (16 → 32 bytes). All new fields default to "NEE on, intensities 1.0" so the renderer lights up correctly even before the UI is wired (Task 6). `RTXPTFrameConstants` becomes 192 bytes (64 + 64 + 16 + 16 + 32). The C++ `static_assert`s are the build-time guard that the layout matches the HLSL mirror.

- [ ] **Step 1: Grow the C++ settings + frame-constants structs**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, locate:

```cpp
struct RTXPTPathTracerSettings
{
    Uint32 MaxBounces        = 4;
    Uint32 AccumulationFrame = 0;
    Uint32 ResetAccumulation = 1;
    Uint32 MinBounces        = 0;
};
static_assert(sizeof(RTXPTPathTracerSettings) == 16, "RTXPTPathTracerSettings layout must match RTXPTShaderShared.hlsli");
```

Replace it with:

```cpp
struct RTXPTPathTracerSettings
{
    Uint32 MaxBounces        = 4;
    Uint32 AccumulationFrame = 0;
    Uint32 ResetAccumulation = 1;
    Uint32 MinBounces        = 0;

    Uint32 EnableNEE           = 1;    // Non-zero enables next-event estimation (direct light sampling).
    Uint32 EnableEnvNEE        = 1;    // Non-zero adds environment (sky) NEE with MIS alongside analytic lights.
    float  EnvIntensity        = 1.0f; // Scales the procedural-sky environment radiance.
    float  LightIntensityScale = 1.0f; // Scales analytic (punctual) light radiance.
};
static_assert(sizeof(RTXPTPathTracerSettings) == 32, "RTXPTPathTracerSettings layout must match RTXPTShaderShared.hlsli");
```

Then, in the same file, locate:

```cpp
static_assert(sizeof(RTXPTFrameConstants) == 176, "RTXPTFrameConstants layout must match RTXPTShaderShared.hlsli");
```

Replace it with:

```cpp
static_assert(sizeof(RTXPTFrameConstants) == 192, "RTXPTFrameConstants layout must match RTXPTShaderShared.hlsli");
```

- [ ] **Step 2: Mirror the 32-byte settings struct in HLSL**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli`, locate:

```hlsl
// Mirrors Diligent::RTXPTPathTracerSettings (the new sub-struct embedded in RTXPTFrameConstants).
struct RTXPTPathTracerSettings
{
    uint MaxBounces;        // Maximum number of secondary bounces; 0 means primary-ray only.
    uint AccumulationFrame; // 0-based index of the sample being added this frame.
    uint ResetAccumulation; // Non-zero means raygen should overwrite the accumulation buffer instead of blending.
    uint MinBounces;        // Reserved for Phase 5.3 Russian roulette; ignored by Phase 5.2.
};
```

Replace it with:

```hlsl
// Mirrors Diligent::RTXPTPathTracerSettings (the sub-struct embedded in RTXPTFrameConstants; total size 32 bytes).
struct RTXPTPathTracerSettings
{
    uint MaxBounces;        // Maximum number of secondary bounces; 0 means primary-ray only.
    uint AccumulationFrame; // 0-based index of the sample being added this frame.
    uint ResetAccumulation; // Non-zero means raygen should overwrite the accumulation buffer instead of blending.
    uint MinBounces;        // Russian-roulette start bounce.

    uint  EnableNEE;           // Non-zero enables next-event estimation (direct light sampling) at each hit.
    uint  EnableEnvNEE;        // Non-zero adds environment (sky) NEE with MIS in addition to analytic lights.
    float EnvIntensity;        // Scales the procedural-sky environment radiance.
    float LightIntensityScale; // Scales analytic (punctual) light radiance.
};
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the settings growth**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.4 NEE path-tracer settings" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the sample header and the shared HLSL header.

---

### Task 2: Add The Shared Procedural-Sky Environment Helper

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

Context: the sky gradient currently lives inline in `RTXPTReference.rmiss`. Both the miss shader (BSDF-sampled sky) and raygen environment NEE must evaluate the **same** radiance for MIS to be unbiased, so factor it into a tiny dependency-free header. The directional/analytic "sun" is now a delta light handled by NEE — it is intentionally **not** part of the sky (a delta light has no visible disk; keeping the old sun-disk would double-count it in reflections).

- [ ] **Step 1: Create `RTXPTEnvironment.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli`:

```hlsl
#ifndef RTXPT_ENVIRONMENT_HLSLI
#define RTXPT_ENVIRONMENT_HLSLI

// Procedural sky used as the reference path tracer's environment light. Punctual/directional lights are
// delta lights sampled separately via NEE, so they are intentionally NOT baked into the sky (that would
// double-count them in reflections). Shared by the miss shader and raygen environment NEE so both
// estimators evaluate identical radiance, which is required for unbiased MIS.
float3 RTXPTEvalSky(float3 Dir)
{
    const float  T       = saturate(Dir.y * 0.5 + 0.5);
    const float3 Horizon = float3(0.48, 0.58, 0.68);
    const float3 Zenith  = float3(0.05, 0.08, 0.14);
    return lerp(Horizon, Zenith, T);
}

#endif // RTXPT_ENVIRONMENT_HLSLI
```

- [ ] **Step 2: Register `RTXPTEnvironment.hlsli` in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, locate:

```cmake
    assets/shaders/RTXPTBSDF.hlsli
    assets/shaders/RTXPTReference.rgen
```

Replace it with:

```cmake
    assets/shaders/RTXPTBSDF.hlsli
    assets/shaders/RTXPTEnvironment.hlsli
    assets/shaders/RTXPTReference.rgen
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the environment helper**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.4 shared procedural-sky environment helper" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the new environment header and the CMake registration.

---

### Task 3: Add The Analytic Light Sampling Helper

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

Context: pure decode math for one punctual light into a sampling result. No ray-tracing intrinsics here (those live in raygen, which owns `g_TLAS`). Type encoding matches `LightTypeToShaderValue` in `RTXPTLights.cpp` (0 = Directional, 1 = Point, 2 = Spot, < 0 = disabled). The "toward the light" direction `-DirectionType.xyz` matches the `RTXPTReference.rmiss` convention. Point/spot radiance includes inverse-square falloff and (for spot) a smooth cone attenuation; directional uses a large distance sentinel.

- [ ] **Step 1: Create `RTXPTLightSampling.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli`:

```hlsl
#ifndef RTXPT_LIGHT_SAMPLING_HLSLI
#define RTXPT_LIGHT_SAMPLING_HLSLI

#include "RTXPTShaderShared.hlsli"

// Result of evaluating one analytic (punctual) light toward a shading point.
struct RTXPTLightSample
{
    float3 Wi;       // unit direction from the shading point toward the light
    float  Distance; // distance to the light (1e16 sentinel for directional lights at infinity)
    float3 Radiance; // incident radiance/irradiance, already including inverse-square + spot-cone falloff
    bool   Valid;    // false for disabled lights, zero range, or points outside the spot cone
};

// Decode a punctual light (type stored in DirectionType.w: 0 = directional, 1 = point, 2 = spot; < 0 = disabled).
// Direction convention matches RTXPTReference.rmiss / RTXPTLights.cpp: -DirectionType.xyz points from the
// surface toward the light, and DirectionType.xyz is the light's emission direction.
RTXPTLightSample RTXPTEvalAnalyticLight(RTXPTLightData Light, float3 P)
{
    RTXPTLightSample S;
    S.Wi       = float3(0.0, 1.0, 0.0);
    S.Distance = 1e16;
    S.Radiance = float3(0.0, 0.0, 0.0);
    S.Valid    = false;

    const float  Type      = Light.DirectionType.w;
    const float3 Color     = Light.ColorIntensity.rgb;
    const float  Intensity = Light.ColorIntensity.a;
    if (Type < -0.5 || Intensity <= 0.0)
        return S;

    if (Type < 0.5)
    {
        // Directional: delta light at infinity, no distance falloff.
        S.Wi       = normalize(-Light.DirectionType.xyz);
        S.Distance = 1e16;
        S.Radiance = Color * Intensity;
        S.Valid    = true;
        return S;
    }

    // Point / spot: positional with inverse-square falloff.
    const float3 ToLight = Light.PositionRange.xyz - P;
    const float  Dist    = length(ToLight);
    if (Dist <= 1e-5)
        return S;

    const float Range = Light.PositionRange.w;
    if (Range > 0.0 && Dist > Range)
        return S;

    S.Wi            = ToLight / Dist;
    S.Distance      = Dist;
    float3 Radiance = Color * Intensity / (Dist * Dist);

    if (Type > 1.5)
    {
        // Spot cone attenuation. SpotAngles.x = inner half-angle, .y = outer half-angle (radians).
        // CosAngle compares the light's emission direction against the light->surface direction (-Wi).
        const float CosAngle = dot(normalize(Light.DirectionType.xyz), -S.Wi);
        const float CosInner = cos(Light.SpotAngles.x);
        const float CosOuter = cos(Light.SpotAngles.y);
        const float Atten    = saturate((CosAngle - CosOuter) / max(CosInner - CosOuter, 1e-4));
        if (Atten <= 0.0)
            return S;
        Radiance *= Atten * Atten; // squared for a smoother cone edge
    }

    S.Radiance = Radiance;
    S.Valid    = true;
    return S;
}

#endif // RTXPT_LIGHT_SAMPLING_HLSLI
```

- [ ] **Step 2: Register `RTXPTLightSampling.hlsli` in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, locate:

```cmake
    assets/shaders/RTXPTEnvironment.hlsli
    assets/shaders/RTXPTReference.rgen
```

Replace it with:

```cmake
    assets/shaders/RTXPTEnvironment.hlsli
    assets/shaders/RTXPTLightSampling.hlsli
    assets/shaders/RTXPTReference.rgen
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the light sampling helper**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.4 analytic punctual-light sampling helper" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the new light-sampling header and the CMake registration.

---

### Task 4: Add MIS + Lobe-Probability Helpers To The BSDF

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`

Context: NEE needs the **same** BSDF pdf the sampler would produce for a given direction, so MIS weights are consistent. The sampler computes its lobe-selection probability inline; this task extracts it into `RTXPTSpecularProbability` (used by both the sampler and NEE) and adds the power-heuristic MIS weight. The refactor of `RTXPTSampleBSDF` is behavior-preserving.

- [ ] **Step 1: Insert `RTXPTPowerHeuristic` + `RTXPTSpecularProbability` after `RTXPTLuminance`**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`, locate:

```hlsl
float RTXPTLuminance(float3 C)
{
    return dot(C, float3(0.2126, 0.7152, 0.0722));
}

// Evaluate f(Wo,Wi) * NoL and the single-sample MIS pdf for the given (unit, away-from-surface) directions.
```

Replace it with:

```hlsl
float RTXPTLuminance(float3 C)
{
    return dot(C, float3(0.2126, 0.7152, 0.0722));
}

// Power heuristic (beta = 2) for combining two single-sample estimators in MIS.
float RTXPTPowerHeuristic(float PdfA, float PdfB)
{
    const float A2 = PdfA * PdfA;
    const float B2 = PdfB * PdfB;
    return A2 / max(A2 + B2, 1e-7);
}

// Lobe-selection probability used by both the BSDF sampler and NEE evaluation so their pdfs agree (MIS).
float RTXPTSpecularProbability(RTXPTSurface S, float3 Wo)
{
    const float  NoV     = max(dot(S.N, Wo), 0.0);
    const float3 Fapprox = RTXPTFresnelSchlick(S.F0, NoV);
    const float  SpecLum = RTXPTLuminance(Fapprox);
    const float  DiffLum = RTXPTLuminance(S.DiffuseAlbedo * (1.0 - Fapprox));
    return clamp(SpecLum / max(SpecLum + DiffLum, 1e-4), 0.1, 0.9);
}

// Evaluate f(Wo,Wi) * NoL and the single-sample MIS pdf for the given (unit, away-from-surface) directions.
```

- [ ] **Step 2: Refactor `RTXPTSampleBSDF` to use `RTXPTSpecularProbability`**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli`, locate:

```hlsl
    // Pick the lobe from the Fresnel-weighted specular vs diffuse luminance, clamped so neither lobe starves.
    const float3 Fapprox = RTXPTFresnelSchlick(S.F0, NoV);
    const float  SpecLum = RTXPTLuminance(Fapprox);
    const float  DiffLum = RTXPTLuminance(S.DiffuseAlbedo * (1.0 - Fapprox));
    const float  SpecProb = clamp(SpecLum / max(SpecLum + DiffLum, 1e-4), 0.1, 0.9);
```

Replace it with:

```hlsl
    // Pick the lobe from the Fresnel-weighted specular vs diffuse luminance (shared with NEE so pdfs agree).
    const float SpecProb = RTXPTSpecularProbability(S, Wo);
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit the BSDF helpers**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTBSDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.4 MIS power heuristic and shared lobe probability" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the BSDF header.

---

### Task 5: Land NEE — Raygen, Miss Shader, And The g_Lights Binding (Atomic)

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

Context: these four changes are **coupled and must land together** to keep the repo buildable+runnable. Diligent binds STATIC ray-tracing variables per stage, and a STATIC variable can only be fetched for a stage whose compiled shader references it. So: raygen must start referencing `g_Lights` (rgen rewrite) **and** the C++ must declare/bind `g_Lights` for `SHADER_TYPE_RAY_GEN` in the same commit; simultaneously the miss shader must stop referencing `g_Lights` (drop the sun disk) so the old miss-stage binding is no longer required. Shadow rays reuse the radiance hit group (index 0) + miss shader (index 0) with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`; the miss shader clears `HitFlag` so a ray that escapes is "visible", and the existing alpha-test any-hit keeps masked-geometry shadows correct. All rays are traced from raygen, so `MaxRecursionDepth = 1` is unchanged.

- [ ] **Step 1: Rewrite the raygen shader**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rgen` with:

```hlsl
#include "RTXPTSceneBridge.hlsli"
#include "RTXPTRandom.hlsli"
#include "RTXPTBSDF.hlsli"
#include "RTXPTEnvironment.hlsli"
#include "RTXPTLightSampling.hlsli"

RaytracingAccelerationStructure                g_TLAS;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4>   g_OutputColor;
VK_IMAGE_FORMAT("rgba32f") RWTexture2D<float4> g_AccumColor;

static float3 ToneMapACES(float3 X)
{
    // Krzysztof Narkowicz fitted ACES curve. Matches Phase 5.1 visual mood when the accumulation buffer
    // is converted back to rgba8 for the blit pass.
    const float A = 2.51;
    const float B = 0.03;
    const float C = 2.43;
    const float D = 0.59;
    const float E = 0.14;
    return saturate((X * (A * X + B)) / (X * (C * X + D) + E));
}

// Trace a visibility (shadow) ray. Reuses the radiance hit group + miss shader instead of adding a second
// ray type: ACCEPT_FIRST_HIT_AND_END_SEARCH + SKIP_CLOSEST_HIT means only the alpha-test any-hit (for
// masked geometry) and the miss shader run. The miss shader clears HitFlag, so a missed ray = visible.
bool RTXPTTraceVisibility(float3 Origin, float3 Dir, float TMax)
{
    RTXPTPathTracerPayload Shadow;
    Shadow.WorldPos    = float3(0.0, 0.0, 0.0);
    Shadow.HitDistance = 0.0;
    Shadow.WorldNormal = float3(0.0, 1.0, 0.0);
    Shadow.HitFlag     = 1u; // assume occluded; the miss shader sets this to 0 when the ray escapes.
    Shadow.BaseColor   = float3(0.0, 0.0, 0.0);
    Shadow.Metallic    = 0.0;
    Shadow.Emission    = float3(0.0, 0.0, 0.0);
    Shadow.Roughness   = 1.0;

    RayDesc Ray;
    Ray.Origin    = Origin;
    Ray.Direction = Dir;
    Ray.TMin      = 1e-3;
    Ray.TMax      = TMax;

    TraceRay(g_TLAS,
             RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER,
             0xFF, 0, 1, 0, Ray, Shadow);

    return Shadow.HitFlag == 0u;
}

// Next-event estimation for one uniformly-selected analytic (punctual) light. Delta lights cannot be hit
// by BSDF sampling, so no MIS is needed - only the light-sampling estimator contributes.
float3 RTXPTSampleAnalyticNEE(RTXPTSurface Surface, float3 Wo, float3 HitPos, float3 ShadowOrigin,
                              inout RTXPTRandom Rng, float LightScale)
{
    const uint LightCount = Bridge::GetLightCount();
    if (LightCount == 0u)
        return float3(0.0, 0.0, 0.0);

    const uint           Index = min(LightCount - 1u, (uint)(NextFloat(Rng) * float(LightCount)));
    const RTXPTLightData Light = Bridge::GetLight(Index);

    const RTXPTLightSample LS = RTXPTEvalAnalyticLight(Light, HitPos);
    if (!LS.Valid)
        return float3(0.0, 0.0, 0.0);

    const float SpecProb = RTXPTSpecularProbability(Surface, Wo);
    float3 FTimesNoL;
    float  BsdfPdf;
    RTXPTEvalBSDF(Surface, Wo, LS.Wi, SpecProb, FTimesNoL, BsdfPdf);
    if (dot(FTimesNoL, FTimesNoL) <= 0.0)
        return float3(0.0, 0.0, 0.0);

    // Stop just short of the light so finite point/spot lights are not self-occluded by floating error.
    const float TMax = min(LS.Distance, 1e16) * 0.999;
    if (!RTXPTTraceVisibility(ShadowOrigin, LS.Wi, TMax))
        return float3(0.0, 0.0, 0.0);

    // Divide by the 1/LightCount selection pdf (i.e. multiply by LightCount). Delta light: no solid-angle pdf.
    return FTimesNoL * LS.Radiance * LightScale * float(LightCount);
}

// Next-event estimation for the environment (procedural sky), combined with BSDF sampling via MIS. The sky
// is sampled with a cosine-hemisphere distribution (pdf = NoL / PI) around the shading normal.
float3 RTXPTSampleEnvNEE(RTXPTSurface Surface, float3 Wo, float3 ShadowOrigin,
                         inout RTXPTRandom Rng, float EnvIntensity)
{
    const float2 Rand = NextFloat2(Rng);
    float        EnvPdf;
    const float3 Wi = SampleCosineHemisphere(Rand, Surface.N, EnvPdf);
    if (EnvPdf <= 0.0)
        return float3(0.0, 0.0, 0.0);

    const float SpecProb = RTXPTSpecularProbability(Surface, Wo);
    float3 FTimesNoL;
    float  BsdfPdf;
    RTXPTEvalBSDF(Surface, Wo, Wi, SpecProb, FTimesNoL, BsdfPdf);
    if (dot(FTimesNoL, FTimesNoL) <= 0.0)
        return float3(0.0, 0.0, 0.0);

    // The sky is only visible if the ray escapes the scene (misses all geometry).
    if (!RTXPTTraceVisibility(ShadowOrigin, Wi, 1e16))
        return float3(0.0, 0.0, 0.0);

    const float3 Sky  = RTXPTEvalSky(Wi) * EnvIntensity;
    const float  MisW = RTXPTPowerHeuristic(EnvPdf, BsdfPdf);
    return FTimesNoL * Sky * MisW / EnvPdf;
}

[shader("raygeneration")]
void main()
{
    const uint2 Pixel      = DispatchRaysIndex().xy;
    const uint2 Dimensions = DispatchRaysDimensions().xy;

    const uint  FrameSeed = asuint(g_FrameConstants.ViewportSize_FrameIdx.w);
    RTXPTRandom Rng       = RTXPTRandom_Init(Pixel, FrameSeed);

    // Jitter inside the pixel - gives free anti-aliasing across accumulated samples.
    const float2 Jitter = NextFloat2(Rng);
    const float2 UV     = (float2(Pixel) + Jitter) / float2(Dimensions);
    const float2 NDC    = UV * 2.0 - 1.0;

    const float4 WorldPos4 = mul(float4(NDC, 1.0, 1.0), g_FrameConstants.ViewProjInv);
    const float3 Origin    = g_FrameConstants.CameraPosition_Time.xyz;
    float3       RayOrigin = Origin;
    float3       RayDir    = normalize(WorldPos4.xyz / WorldPos4.w - Origin);

    const uint  MaxBounces          = max(g_FrameConstants.PathTracer.MaxBounces, 1u);
    const bool  EnableNEE           = g_FrameConstants.PathTracer.EnableNEE != 0u;
    const bool  EnableEnvNEE        = EnableNEE && (g_FrameConstants.PathTracer.EnableEnvNEE != 0u);
    const float EnvIntensity        = g_FrameConstants.PathTracer.EnvIntensity;
    const float LightIntensityScale = g_FrameConstants.PathTracer.LightIntensityScale;

    float3 Throughput   = float3(1.0, 1.0, 1.0);
    float3 PathRadiance = float3(0.0, 0.0, 0.0);

    // MIS bookkeeping for the BSDF-sampled vertex that generated the current ray.
    // PrevBsdfPdf == 0 marks "no preceding BSDF sample" (the primary camera ray) -> emission weight 1.
    float  PrevBsdfPdf   = 0.0;
    float3 PrevNormal    = float3(0.0, 0.0, 0.0);
    bool   PrevDidEnvNEE = false;

    [loop]
    for (uint Bounce = 0u; Bounce < MaxBounces; ++Bounce)
    {
        RayDesc Ray;
        Ray.Origin    = RayOrigin;
        Ray.Direction = RayDir;
        Ray.TMin      = 1e-3;
        Ray.TMax      = 10000.0;

        RTXPTPathTracerPayload Payload;
        Payload.WorldPos    = float3(0.0, 0.0, 0.0);
        Payload.HitDistance = -1.0;
        Payload.WorldNormal = float3(0.0, 1.0, 0.0);
        Payload.HitFlag     = 0u;
        Payload.BaseColor   = float3(0.0, 0.0, 0.0);
        Payload.Emission    = float3(0.0, 0.0, 0.0);
        Payload.Metallic    = 0.0;
        Payload.Roughness   = 1.0;

        // RAY_FLAG_NONE lets the alpha-test any-hit shader run for non-opaque (alpha-masked) geometry.
        TraceRay(g_TLAS, RAY_FLAG_NONE, 0xFF, 0, 1, 0, Ray, Payload);

        if (Payload.HitFlag == 0u)
        {
            // Environment (sky). When the previous vertex performed environment NEE, MIS-weight this
            // BSDF-sampled sky hit against that environment-sampling estimator to avoid double counting.
            float MisW = 1.0;
            if (PrevDidEnvNEE)
            {
                const float EnvPdf = max(dot(PrevNormal, RayDir), 0.0) * RTXPT_INV_PI;
                MisW               = RTXPTPowerHeuristic(PrevBsdfPdf, EnvPdf);
            }
            PathRadiance += Throughput * Payload.Emission * EnvIntensity * MisW;
            break;
        }

        // Emissive surface hit: no emissive-light NEE sampler competes, so add it with full weight.
        PathRadiance += Throughput * Payload.Emission;

        const float3 Wo      = -RayDir;
        RTXPTSurface  Surface = RTXPTMakeSurface(Payload.WorldNormal, Payload.BaseColor, Payload.Metallic, Payload.Roughness);

        // Offset the shadow / next-bounce origin along the shading normal to avoid self-intersection.
        const float  Bias         = max(1e-4, 1e-3 * Payload.HitDistance);
        const float3 ShadowOrigin = Payload.WorldPos + Surface.N * Bias;

        // Direct lighting (next-event estimation) at this vertex.
        if (EnableNEE)
        {
            PathRadiance += Throughput * RTXPTSampleAnalyticNEE(Surface, Wo, Payload.WorldPos, ShadowOrigin, Rng, LightIntensityScale);
            if (EnableEnvNEE)
                PathRadiance += Throughput * RTXPTSampleEnvNEE(Surface, Wo, ShadowOrigin, Rng, EnvIntensity);
        }

        // Importance-sample the GGX BSDF for the next direction. Weight = f * NoL / pdf.
        float3 NextDir;
        float3 Weight;
        float  BsdfPdf;
        if (!RTXPTSampleBSDF(Surface, Wo, Rng, NextDir, Weight, BsdfPdf))
            break;

        Throughput *= Weight;

        // Russian roulette once we are past MinBounces - unbiased early termination of dim paths.
        if (Bounce >= g_FrameConstants.PathTracer.MinBounces)
        {
            const float Survive = clamp(max(Throughput.x, max(Throughput.y, Throughput.z)), 0.05, 1.0);
            if (NextFloat(Rng) > Survive)
                break;
            Throughput /= Survive;
        }

        // Carry MIS state for weighting the next ray's sky hit.
        PrevBsdfPdf   = BsdfPdf;
        PrevNormal    = Surface.N;
        PrevDidEnvNEE = EnableEnvNEE;

        RayOrigin = ShadowOrigin;
        RayDir    = NextDir;
    }

    // Blend into the accumulation buffer. ResetAccumulation == 1 means this is the first sample after a reset.
    float3     Accumulated = PathRadiance;
    const uint Reset       = g_FrameConstants.PathTracer.ResetAccumulation;
    const uint Frame       = max(g_FrameConstants.PathTracer.AccumulationFrame, 1u);
    if (Reset == 0u)
    {
        const float4 Previous = g_AccumColor[Pixel];
        const float  InvN     = 1.0 / float(Frame);
        Accumulated           = Previous.rgb + (PathRadiance - Previous.rgb) * InvN;
    }
    g_AccumColor[Pixel] = float4(Accumulated, 1.0);

    // OutputColor is the rgba8 image consumed by the existing blit/compute chain.
    g_OutputColor[Pixel] = float4(ToneMapACES(Accumulated), 1.0);
}

// TODO(RTXPT-Port Phase 5.3): Add transmission / nested dielectrics to the BSDF (currently opaque diffuse + GGX specular).
// TODO(RTXPT-Port Phase 5.4): Add emissive-triangle area lights, light importance sampling / RIS, and HDR environment-map MIS (NEE currently uses uniform light selection + a procedural-sky cosine env sampler).
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

- [ ] **Step 2: Rewrite the miss shader to use the shared sky and drop g_Lights**

Replace the entire contents of `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rmiss` with:

```hlsl
#include "RTXPTShaderShared.hlsli"
#include "RTXPTEnvironment.hlsli"

[shader("miss")]
void main(inout RTXPTPathTracerPayload Payload)
{
    Payload.WorldPos    = float3(0.0, 0.0, 0.0);
    Payload.HitDistance = -1.0;
    Payload.WorldNormal = float3(0.0, 1.0, 0.0);
    Payload.HitFlag     = 0u;
    Payload.BaseColor   = float3(0.0, 0.0, 0.0);
    Payload.Emission    = RTXPTEvalSky(WorldRayDirection());
    Payload.Metallic    = 0.0;
    Payload.Roughness   = 1.0;
}

// TODO(RTXPT-Port Phase 5.4): Replace the procedural sky with an importance-sampled HDR environment map (EnvMapBaker) and add environment-map MIS.
```

- [ ] **Step 3: Re-target the resolved closest-hit NEE TODO**

In `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTReference.rchit`, locate:

```hlsl
// TODO(RTXPT-Port Phase 5.5): Add NEE shadow rays toward analytic and environment lights.
```

Replace it with:

```hlsl
// TODO(RTXPT-Port Phase 5.4): Emissive surfaces are gathered by BSDF sampling only; add emissive-triangle area-light NEE + MIS once an emissive light list exists.
```

- [ ] **Step 4: Move the g_Lights binding from miss to raygen (resource layout)**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`, locate:

```cpp
        .AddVariable(SHADER_TYPE_RAY_MISS, "g_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

Replace it with:

```cpp
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_Lights", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

- [ ] **Step 5: Move the g_Lights binding from miss to raygen (Set call)**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`, locate:

```cpp
    m_Stats.LightBridgeBound    = SetStatic(SHADER_TYPE_RAY_MISS, "g_Lights", pLightsView);
```

Replace it with:

```cpp
    m_Stats.LightBridgeBound    = SetStatic(SHADER_TYPE_RAY_GEN, "g_Lights", pLightsView);
```

- [ ] **Step 6: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/RTXPTReference.rgen Samples/RTXPT/assets/shaders/RTXPTReference.rmiss Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 7: Commit the NEE landing**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/assets/shaders/RTXPTReference.rgen Samples/RTXPT/assets/shaders/RTXPTReference.rmiss Samples/RTXPT/assets/shaders/RTXPTReference.rchit Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add phase 5.4 NEE direct lighting with shadow rays and MIS" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the raygen, miss, closest-hit, and RT-pass files.

---

### Task 6: Wire NEE Settings And UI Into The Sample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

Context: the settings defaults from Task 1 already turn NEE on, so the renderer is correct after Task 5 even before this task. This task adds the runtime toggles and intensity scales, each resetting accumulation on change so the converged image restarts cleanly.

- [ ] **Step 1: Add the NEE members**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, locate:

```cpp
    Uint32                      m_MaxBounces                = 4;
    Uint32                      m_MinBounces                = 3;
    int                         m_SelectedSceneCamera       = -1;
```

Replace it with:

```cpp
    Uint32                      m_MaxBounces                = 4;
    Uint32                      m_MinBounces                = 3;
    bool                        m_EnableNEE                 = true;
    bool                        m_EnableEnvNEE              = true;
    float                       m_EnvIntensity              = 1.0f;
    float                       m_LightIntensityScale       = 1.0f;
    int                         m_SelectedSceneCamera       = -1;
```

- [ ] **Step 2: Feed the NEE members into the frame constants**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate (in `UpdateFrameConstants`):

```cpp
    m_LastFrameConstants.PathTracer.MaxBounces        = m_MaxBounces;
    m_LastFrameConstants.PathTracer.AccumulationFrame = m_AccumulationFrame;
    m_LastFrameConstants.PathTracer.ResetAccumulation = m_ResetAccumulationPending ? 1u : 0u;
    m_LastFrameConstants.PathTracer.MinBounces        = m_MinBounces;
```

Replace it with:

```cpp
    m_LastFrameConstants.PathTracer.MaxBounces          = m_MaxBounces;
    m_LastFrameConstants.PathTracer.AccumulationFrame   = m_AccumulationFrame;
    m_LastFrameConstants.PathTracer.ResetAccumulation   = m_ResetAccumulationPending ? 1u : 0u;
    m_LastFrameConstants.PathTracer.MinBounces          = m_MinBounces;
    m_LastFrameConstants.PathTracer.EnableNEE           = m_EnableNEE ? 1u : 0u;
    m_LastFrameConstants.PathTracer.EnableEnvNEE        = m_EnableEnvNEE ? 1u : 0u;
    m_LastFrameConstants.PathTracer.EnvIntensity        = m_EnvIntensity;
    m_LastFrameConstants.PathTracer.LightIntensityScale = m_LightIntensityScale;
```

- [ ] **Step 3: Add the NEE UI controls**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate (in `UpdateUI`):

```cpp
    int MinBouncesUI = static_cast<int>(m_MinBounces);
    if (ImGui::SliderInt("Min bounces (RR start)", &MinBouncesUI, 0, 16))
    {
        m_MinBounces = static_cast<Uint32>(MinBouncesUI);
        RequestAccumulationReset("Min bounces changed");
    }
    if (ImGui::Button("Reset accumulation"))
        RequestAccumulationReset("User reset");
```

Replace it with:

```cpp
    int MinBouncesUI = static_cast<int>(m_MinBounces);
    if (ImGui::SliderInt("Min bounces (RR start)", &MinBouncesUI, 0, 16))
    {
        m_MinBounces = static_cast<Uint32>(MinBouncesUI);
        RequestAccumulationReset("Min bounces changed");
    }
    if (ImGui::Checkbox("Next-event estimation (NEE)", &m_EnableNEE))
        RequestAccumulationReset("NEE toggled");
    if (ImGui::Checkbox("Environment NEE + MIS", &m_EnableEnvNEE))
        RequestAccumulationReset("Environment NEE toggled");
    if (ImGui::SliderFloat("Light intensity scale", &m_LightIntensityScale, 0.0f, 10.0f))
        RequestAccumulationReset("Light intensity changed");
    if (ImGui::SliderFloat("Environment intensity", &m_EnvIntensity, 0.0f, 5.0f))
        RequestAccumulationReset("Environment intensity changed");
    if (ImGui::Button("Reset accumulation"))
        RequestAccumulationReset("User reset");
```

- [ ] **Step 4: Re-target the resolved sample NEE TODO**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`, locate:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.5): add explicit light sampling and MIS once the lighting baker is restored.");
```

Replace it with:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 5.4): NEE samples analytic + procedural-sky lights with MIS; add emissive area lights, light RIS, and HDR env-map IBL.");
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit the sample wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 5.4 NEE toggles and intensity controls into the sample" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing the sample header and source.

---

### Task 7: Phase 5.4 Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [ ] **Step 1: Confirm the resolved markers are gone and the deferred markers remain**

Run:

```powershell
rg -uuu -n "Phase 5.5" DiligentSamples/Samples/RTXPT
```

Expected: **no output** (all four `Phase 5.5` NEE markers were re-targeted to `Phase 5.4`).

Run:

```powershell
rg -uuu -n "TODO\(RTXPT-Port Phase 5.4" DiligentSamples/Samples/RTXPT
```

Expected four matches (deferred lighting refinements):

```text
RTXPTReference.rmiss : HDR environment map (EnvMapBaker) + environment-map MIS
RTXPTReference.rchit : emissive-triangle area-light NEE + MIS
RTXPTReference.rgen  : emissive area lights, light importance sampling / RIS, HDR environment-map MIS
RTXPTSample.cpp      : add emissive area lights, light RIS, and HDR env-map IBL
```

(The pre-existing `Phase 5.3` transmission / `ALPHA_MODE_BLEND` markers, the `Phase 5` compiler-flag marker, the `Phase 4` UI marker, and the `Phase 6` tone-mapping marker are untouched by this plan and still appear under their own searches.)

- [ ] **Step 2: Confirm the new headers exist and are registered**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTEnvironment.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTLightSampling.hlsli
rg -n "RTXPTEnvironment.hlsli|RTXPTLightSampling.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: `True` for both files and two matches in `CMakeLists.txt`.

- [ ] **Step 3: Confirm the layout static_asserts and binding move**

Run:

```powershell
rg -n "sizeof\(RTXPTPathTracerSettings\) == 32|sizeof\(RTXPTFrameConstants\) == 192" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
rg -n "total size 32 bytes" DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTShaderShared.hlsli
rg -n "SHADER_TYPE_RAY_GEN, \"g_Lights\"" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
rg -n "SHADER_TYPE_RAY_MISS, \"g_Lights\"" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: two matches for the C++ `static_assert`s, one for the HLSL mirror comment, **two** matches for `SHADER_TYPE_RAY_GEN, "g_Lights"` (the `AddVariable` declaration and the `SetStatic` call), and **no** matches for `SHADER_TYPE_RAY_MISS, "g_Lights"`.

- [ ] **Step 4: Confirm the DiligentSamples log shows the Phase 5.4 commits**

Run:

```powershell
git -C DiligentSamples log --oneline -n 7
```

Expected (most recent first):

```text
feat(rtxpt): wire phase 5.4 NEE toggles and intensity controls into the sample
feat(rtxpt): add phase 5.4 NEE direct lighting with shadow rays and MIS
feat(rtxpt): add phase 5.4 MIS power heuristic and shared lobe probability
feat(rtxpt): add phase 5.4 analytic punctual-light sampling helper
feat(rtxpt): add phase 5.4 shared procedural-sky environment helper
feat(rtxpt): add phase 5.4 NEE path-tracer settings
fix(rtxpt): handle normal-map handedness and layout guards
```

- [ ] **Step 5: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: exit code 0. If the build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 6: Optional D3D12 runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with D3D12 on a standalone-RT-capable machine. Expected ImGui facts:

```text
TraceRays pass: ready
Light bridge: bound
Lights: N (N >= 1)
Accumulation target: active (RGBA32F)
TraceRays executed: yes; TraceRays count increases every frame
```

Expected visual result vs Phase 5.3b: surfaces now show **direct lighting with hard shadows** cast from the scene's punctual lights (directional sun / point / spot), instead of the previous ambient-only BSDF look. The image converges noticeably **faster** than 5.3b because direct light is now explicitly sampled rather than found only by chance BSDF bounces. UI controls:

- Toggling **"Next-event estimation (NEE)"** off reverts to the pure BSDF path tracer (much noisier, slower convergence) and the converged image must match (NEE is unbiased).
- Toggling **"Environment NEE + MIS"** off removes the environment-sampling estimator; the sky still lights surfaces via BSDF bounces, and the converged image must be unchanged.
- **"Light intensity scale"** brightens/darkens the analytic-light direct contribution; **"Environment intensity"** scales the sky. If the scene's punctual lights use large photometric units and the image is over-bright at scale 1.0, reduce the slider (the deferred `Phase 5.4` photometric-units marker tracks proper handling).
- Moving any slider/toggle, or "Reset accumulation", restarts convergence.

- [ ] **Step 7: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT-capable machine. Expected facts and visuals match the D3D12 run. If `Light bridge: bound` is shown but `Lights: 1` with no real scene lights, the single uploaded light is the disabled default (type `-1`); `RTXPTEvalAnalyticLight` returns `Valid = false` for it, so there is no analytic direct lighting and only environment NEE + BSDF sampling contribute — the sample must still render and converge.

If standalone ray tracing shaders are unavailable, expected:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
```

and the sample clears the swapchain via `ClearFallback`.

- [ ] **Step 8: Commit the top-level submodule pointer and plan**

After all `DiligentSamples` Phase 5.4 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-29-rtxpt-phase5-4-reference-nee-mis.md
git commit -m "feat(samples): plan and add RTXPT phase 5.4 reference NEE and MIS direct lighting" -m "Co-Authored-By: GPT 5.5"
```

Expected: one top-level commit that records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] **Spec coverage.** This plan implements the spec's named **"Initial light support"** (`Resource And Asset Strategy`): *Environment light, Directional light, Simple analytic lights*. It resolves the in-code `Phase 5.5` NEE markers and matches RTXPT reference mode's `HandleNEE` / `HandleHit` / `HandleMiss` direct-lighting structure (`D:/RTXPT-fork` `rendering_pipeline` / `PathTrace` memories: reference mode does NEE + MIS and accumulation, independent of the realtime stable-plane/RTXDI track). The remaining "Later light support" (LightsBaker, emissive area lights, RTXDI/ReGIR) and HDR env-map IBL stay deferred as `Phase 5.4` markers and future sub-phases (5.5 stable planes, 5.6 RTXDI, 5.7 NRD, 5.8 NVAPI/SER/OMM/DLSS), with the renumbering documented in the Scope Note.
- [x] **Runnable increments.** Each task is a focused, buildable commit. Tasks 1-4 are purely additive (new struct fields with benign defaults, two unused standalone headers, two new BSDF helpers, and a behavior-preserving `RTXPTSampleBSDF` refactor) and leave the old BSDF-only renderer running. Task 5 is **atomic by necessity** — Diligent binds STATIC RT variables per stage and only for stages whose shader references the resource, so the raygen `g_Lights` usage, the miss-shader drop of `g_Lights`, and the C++ `g_Lights` stage move must land together; combined they keep the repo buildable and runnable (raygen declares+binds+uses `g_Lights`; the miss shader no longer references it). After Task 5 the defaults already enable NEE; Task 6 only adds runtime controls. The accumulation/blit chain, payload size (64 B), `MaxRecursionDepth` (1), `MaxAttributeSize`, hit-group/SBT layout, and the closest-hit/any-hit logic are all unchanged.
- [x] **No new ray type / proven pattern.** Shadow rays reuse hit group 0 + miss 0 with `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER` and the existing 64-byte payload; the miss shader clears `HitFlag` (escape = visible) and the existing alpha-test any-hit keeps masked-geometry shadows correct. All rays are traced from raygen, so recursion depth stays 1. The per-stage STATIC binding move is the pattern confirmed in `DiligentSamples/Tutorials/Tutorial21_RayTracing/src/Tutorial21_RayTracing.cpp` (`GetStaticVariableByName(stage, "g_ConstantsCB")->Set(...)` once per stage).
- [x] **MIS correctness.** Delta lights (directional/point/spot) are sampled by NEE only (no BSDF overlap) and weighted by the inverse selection probability `* LightCount`. The environment is sampled by two estimators — cosine-hemisphere env NEE (pdf `NoL/PI`) and BSDF sampling — combined with the power heuristic: the env-NEE term uses `RTXPTPowerHeuristic(EnvPdf, BsdfPdf)` and the BSDF-sampled sky hit on the next iteration uses `RTXPTPowerHeuristic(PrevBsdfPdf, EnvPdf(dir))`, where `EnvPdf(dir) = max(dot(PrevNormal, RayDir), 0) * RTXPT_INV_PI`. The primary ray (`PrevBsdfPdf == 0`, `PrevDidEnvNEE == false`) and emissive-surface hits get weight 1. `RTXPTSpecularProbability` is shared by the sampler and NEE so the BSDF pdf used in MIS exactly matches the sampler's. With `EnableNEE` off the renderer degrades to the original unbiased BSDF-only path; with `EnableEnvNEE` off the sky reverts to BSDF-only weight 1.
- [x] **Single source of truth.** `RTXPTPathTracerSettings` is defined once in C++ (`RTXPTSample.hpp`, `static_assert(sizeof == 32)`; `RTXPTFrameConstants` `static_assert(sizeof == 192)`) and mirrored once in HLSL (`RTXPTShaderShared.hlsli`) with matching field order (`MaxBounces`, `AccumulationFrame`, `ResetAccumulation`, `MinBounces`, `EnableNEE`, `EnableEnvNEE`, `EnvIntensity`, `LightIntensityScale`). The procedural sky lives once in `RTXPTEnvironment.hlsli` (used by both the miss shader and raygen env NEE). The light decode lives once in `RTXPTLightSampling.hlsli`.
- [x] **Type/name consistency.** `RTXPTEvalAnalyticLight` (Task 3) returns `RTXPTLightSample { Wi, Distance, Radiance, Valid }`, consumed by `RTXPTSampleAnalyticNEE` (Task 5). `RTXPTPowerHeuristic` / `RTXPTSpecularProbability` (Task 4) are consumed by the raygen NEE helpers and the BSDF sampler (Task 4/5). `RTXPTEvalSky` (Task 2) is consumed by the miss shader and `RTXPTSampleEnvNEE` (Task 5). `RTXPTTraceVisibility` (Task 5) is consumed by both NEE helpers. The HLSL settings fields `EnableNEE` / `EnableEnvNEE` / `EnvIntensity` / `LightIntensityScale` match the C++ members `m_EnableNEE` / `m_EnableEnvNEE` / `m_EnvIntensity` / `m_LightIntensityScale` (Task 6) and the `PathTracer.*` writes (Task 6 Step 2). `Bridge::GetLightCount` / `Bridge::GetLight` (already outside `RTXPT_ENABLE_HIT_BRIDGE`) are called from raygen.
- [x] **No placeholders.** Every code step shows complete code; every command shows expected output. The only `TODO(...)` strings are the intentional, structured open-work markers required by the spec's TODO policy.
- [x] **House style honored.** Verification avoids build/runtime execution unless the user explicitly asks (per `CLAUDE.md`); each task is a single-purpose commit (Task 5 is intentionally multi-file because the change is atomic) using the established `Co-Authored-By: GPT 5.5` trailer that matches every prior RTXPT commit and Phase 5.x plan; copyright dates stay `2026`; the obsolete sun-disk hack is removed rather than left dead, and the deferred work is preserved with structured markers.
