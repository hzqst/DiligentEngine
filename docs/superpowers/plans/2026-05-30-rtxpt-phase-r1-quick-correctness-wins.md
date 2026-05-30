# RTXPT Phase R1 — Quick Correctness & Quality Wins Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the DiligentEngine RTXPT reference path tracer three steps closer to RTXPT-fork parity by adding an adaptive firefly filter (G1), running next-event estimation at every path bounce (G2), and replacing the single chained PRNG with stateless per-(pixel, vertex, sample) seeding (G3).

**Architecture:** All three changes are localized to the already-RTXPT-aligned `PathTracer/` shader tree plus the C++ sample settings/UI. G1 adds a runtime-gated soft cap (`signal *= threshold·K / average(signal)`) carried by a raygen-local `fireflyFilterK` that shrinks as the path spreads; the filter is disabled by uploading a zero threshold so converged output is provably unchanged. G2 is a default change to the NEE bounce budget. G3 reseeds a stateless generator per path vertex and per effect (camera/NEE/scatter/RR), decorrelating bounces and dimensions without changing any estimator's expected value. Full Sobol/Owen low-discrepancy sampling stays deferred to Phase R5 (G9).

**Tech Stack:** HLSL (DXC, ray-tracing pipeline) under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/`; C++ (Diligent sample framework, Dear ImGui) under `DiligentSamples/Samples/RTXPT/src/`. No automated shader-test harness exists; correctness guards are compile-time `static_assert`s plus documented manual GPU verification (run on explicit user request only, per the workspace rule).

---

## Context You Need Before Starting

**Phase R0.5 has already landed.** The reference path tracer shaders were renamed/reorganized into a `PathTracer/`-style tree with RTXPT-fork-aligned symbol names. The spec's "Touches" lists use pre-R0.5 names (`RTXPTReference.rgen`, `RTXPTRandom.hlsli`, `RTXPTBSDF.hlsli`, `RTXPTShaderShared.hlsli`); the **current** equivalents this plan edits are:

| Spec name (pre-R0.5) | Current path (edit these) |
|---|---|
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTRandom.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli` |
| `RTXPTBSDF.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` |
| `RTXPTShaderShared.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` |
| path-tracer helpers / NEE | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`, `.../PathTracerHelpers.hlsli` |

**The submodule.** `DiligentSamples` is a git submodule on branch `RTXPT`. All edits below are inside the submodule. Commit **inside** `DiligentSamples/` (its working tree), not the umbrella repo. The umbrella repo only tracks the submodule pointer; do not bump it as part of these task commits unless asked.

**RTXPT-fork reference anchors** (read-only, for cross-checking the math): `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerHelpers.hlsli:194-219` (firefly K + filter), `.../PathTracer.hlsli:327-333,477-481,654-657` (K update + emission filtering), `.../PathTracerNEE.hlsli:245-254` (NEE dampening), `.../Utils/StatelessSampleGenerators.hlsli:38-48,191-199` (stateless seeding), `.../Utils/SampleGenerators.hlsli:16-23` (`SampleGeneratorEffectSeed`).

## File Structure (what each task touches)

- `PathTracer/PathTracerShared.h` — GPU mirror of the settings struct. Task 1 adds `fireflyFilterThreshold` (replaces a padding word; size stays 48 bytes).
- `src/RTXPTSample.hpp` — C++ mirror of the settings struct + `static_assert(sizeof==48)`; UI-state struct; `m_MaxNEEBounces` default. Tasks 1, 7.
- `src/RTXPTSample.cpp` — `UpdateFrameConstants` (uploads settings), `UpdateUI` (ImGui panel). Tasks 1, 6, 10.
- `PathTracer/PathTracerHelpers.hlsli` — math helpers. Task 2 adds the firefly-filter functions.
- `PathTracer/Rendering/Materials/BxDF.hlsli` — `SampleBSDF`. Task 3 surfaces the chosen-lobe probability `lobeP`.
- `PathTracer/PathTracerSample.rgen` — the raygen N-bounce loop. Tasks 3, 4, 9.
- `PathTracer/PathTracer.hlsli` — `SampleAnalyticNEE` / `SampleEnvironmentNEE`. Task 5 firefly-dampens NEE.
- `PathTracer/Utils/SampleGenerators.hlsli` — PRNG. Task 8 adds the stateless generator + effect seeds.
- `RTXPT_FORK_MAPPING.md` — symbol correspondence doc. Task 10 adds rows.

## Cross-Cutting Contract (settings layout) — keep in lockstep

`PathTracerConstants` is mirrored in C++ (`RTXPTSample.hpp`, `static_assert(sizeof(PathTracerConstants)==48)`) and HLSL (`PathTracerShared.h`), and embedded in `SampleConstants` (`static_assert(sizeof(SampleConstants)==208)`). Task 1 grows it by **reusing an existing padding word** (`_padding0 → fireflyFilterThreshold`), so **both byte sizes stay unchanged (48 / 208) and both `static_assert`s are untouched**. Field order and offsets must remain identical between the two files.

## Verification Note (read once)

There is no shader unit-test runner. Per the workspace rule (global `CLAUDE.md`) and the spec's verification strategy, **do not auto-run build/test/run commands** — list them so the user can run them on request. Each task's "Verify" steps are: (a) the compile-time `static_assert`s that the C++ build enforces, and (b) a manual GPU check the user runs when they choose. The phase's primary acceptance test is **unbiasedness**: with the firefly filter disabled (FF threshold → 0) and seeding changes in place, the *converged* accumulation image must match the pre-R1 converged image (only per-sample noise differs).

---

### Task 1: Add `fireflyFilterThreshold` to the settings contract (CPU + GPU), inert for now

Adds one float to the shared settings struct by repurposing a padding word, and uploads it each frame from the existing UI state. Nothing consumes it yet — this keeps the struct change isolated and the build green.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h:20-24`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp:70-75`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp:282-283`

- [ ] **Step 1: Replace the first padding word in the HLSL settings struct**

In `PathTracerShared.h`, the `PathTracerConstants` struct currently ends:

```hlsl
    uint maxNEEBounceCount;  // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    uint analyticLightCount; // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    uint _padding0;
    uint _padding1;
};
```

Replace those four lines with:

```hlsl
    uint  maxNEEBounceCount;     // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    uint  analyticLightCount;    // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float fireflyFilterThreshold; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter entirely.
    uint  _padding1;
};
```

- [ ] **Step 2: Replace the matching padding word in the C++ settings struct**

In `RTXPTSample.hpp`, `PathTracerConstants` currently ends:

```cpp
    Uint32 maxNEEBounceCount  = 1; // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    Uint32 analyticLightCount = 0; // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    Uint32 _padding0          = 0;
    Uint32 _padding1          = 0;
};
static_assert(sizeof(PathTracerConstants) == 48, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

Replace with:

```cpp
    Uint32 maxNEEBounceCount      = 1;    // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
    Uint32 analyticLightCount     = 0;    // CPU-side count of valid analytic lights; the uploaded dummy light is not sampled.
    float  fireflyFilterThreshold = 0.0f; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter (set from UI each frame).
    Uint32 _padding1              = 0;
};
static_assert(sizeof(PathTracerConstants) == 48, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

(`float` and `Uint32` are both 4 bytes, so the size stays 48; the `static_assert` line is unchanged.)

- [ ] **Step 3: Upload the threshold from the UI state each frame**

In `RTXPTSample.cpp`, `UpdateFrameConstants`, after this line:

```cpp
    m_LastFrameConstants.ptConsts.analyticLightCount    = m_Lights.GetStats().LightCount;
```

add:

```cpp
    // G1: a disabled firefly filter uploads a zero threshold, so the soft cap is a no-op and the
    // converged image is identical to the filter-on image (only per-sample variance differs).
    m_LastFrameConstants.ptConsts.fireflyFilterThreshold =
        m_ReferenceUI.ReferenceFireflyFilterEnabled ? m_ReferenceUI.ReferenceFireflyFilterThreshold : 0.0f;
```

- [ ] **Step 4: Verify the struct contract compiles**

The guard is compile-time. When the user requests a build, the two `static_assert`s in `RTXPTSample.hpp` (`sizeof(PathTracerConstants)==48`, `sizeof(SampleConstants)==208`) must compile. No runtime change is expected this task (no shader reads the field yet).

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h \
        Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): add fireflyFilterThreshold to PathTracerConstants (R1/G1)"
```

---

### Task 2: Add the firefly-filter math helpers

Ports RTXPT-fork's `ComputeNewScatterFireflyFilterK`, `FireflyFilter`, and `FireflyFilterShort` into the helpers header. Uses numeric literals for π (not `K_PI`) and the `acos`/`sqrt` intrinsics (not `FastACos`/`FastSqrt`), because `PathTracerHelpers.hlsli` is included **before** `BxDF.hlsli` (which defines `K_PI`) in `PathTracer.hlsli`; pulling those names in here would be a use-before-definition or a redefinition.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`

- [ ] **Step 1: Append the helpers before the include guard's `#endif`**

The file currently ends:

```hlsl
// Power heuristic for MIS (Veach). Matches RTXPT-fork PathTracerHelpers.hlsli signature.
float PowerHeuristic(float nf, float fPdf, float ng, float gPdf)
{
    const float f  = nf * fPdf;
    const float g  = ng * gPdf;
    const float f2 = f * f;
    const float g2 = g * g;
    return f2 / max(f2 + g2, 1e-7);
}

#endif // __PATH_TRACER_HELPERS_HLSLI__
```

Insert the following block between the `PowerHeuristic` closing brace and the `#endif`:

```hlsl

// Mean of the RGB channels. RTXPT uses Average() (not luminance) for the firefly soft cap to
// avoid a hue shift toward blue when clamping. (Self-contained: pi literals avoid coupling to
// BxDF.hlsli's K_PI, which is included after this header.)
float Average(float3 v)
{
    return (v.x + v.y + v.z) * 0.3333333333333333;
}

// Ray-cone spread angle implied by a scatter pdf (RTXPT ComputeRayConeSpreadAngleExpansionByScatterPDF,
// growthFactor folded to 1.0). A delta lobe (pdf==0 sentinel) has zero spread.
float ComputeRayConeSpreadAngleExpansionByScatterPDF(float bsdfScatterPdf)
{
    const float twoPi = 6.28318530717958647692;
    return 2.0 * acos(max(-1.0, 1.0 - (1.0 / bsdfScatterPdf) / twoPi));
}

// Adaptive firefly-filter K (RTXPT ComputeNewScatterFireflyFilterK): K shrinks as the path
// spreads. `currentK` is the path's running K, `bouncePdf` the scatter pdf (0 == delta event),
// `lobeP` the probability of the lobe that was sampled.
float ComputeNewScatterFireflyFilterK(float currentK, float bouncePdf, float lobeP)
{
    const float minK  = 0.00001;
    const float angle = (bouncePdf == 0.0) ? 0.0 : ComputeRayConeSpreadAngleExpansionByScatterPDF(bouncePdf);
    const float k     = 32.0;             // empirical
    float       p     = k / (k + angle * angle);
    p *= sqrt(lobeP);                     // sqrt behaves better empirically
    return max(minK, currentK * p);
}

// Soft-cap a vector signal to threshold*K of its own average (RTXPT FireflyFilter).
float3 FireflyFilter(float3 signalIn, float threshold, float fireflyFilterK)
{
    const float fft  = threshold * fireflyFilterK;
    const float maxR = Average(signalIn);
    if (maxR > fft)
        signalIn = signalIn / maxR * fft;
    return signalIn;
}

// Scalar dampening factor for the same soft cap (RTXPT FireflyFilterShort); multiply a radiance by it.
float FireflyFilterShort(float signalAverage, float threshold, float fireflyFilterK)
{
    const float fft = threshold * fireflyFilterK;
    return (signalAverage > fft) ? (fft / signalAverage) : 1.0;
}
```

- [ ] **Step 2: Verify it compiles where included**

These functions are header-only and unused until Tasks 4–5, so the guard is "the shaders that include `PathTracerHelpers.hlsli` (via `PathTracer.hlsli`) still compile." When the user requests a build, confirm `PathTracerSample.rgen` and the hit shaders still build with no redefinition errors (`Average`/`FireflyFilter`/etc. are new names not used elsewhere — verified by grep before this task).

- [ ] **Step 3: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli
git commit -m "feat(rtxpt): add adaptive firefly-filter helpers (R1/G1)"
```

---

### Task 3: Surface the chosen-lobe probability `lobeP` from `SampleBSDF`

The firefly K update needs the probability of the lobe that was actually sampled. Add an `out float lobeP` to `SampleBSDF` and update its single caller so the shader still compiles. `lobeP` is unused this task (consumed in Task 4).

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli:105-147`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen:99-103`

- [ ] **Step 1: Extend the `SampleBSDF` signature and initialize the new out-param**

In `BxDF.hlsli`, change the signature and the out-param initializers. Replace:

```hlsl
bool SampleBSDF(StandardBSDFData bsdfData, float3 wo, inout SampleGenerator sg,
                out float3 wi, out float3 weight, out float pdf)
{
    wi     = float3(0.0, 0.0, 0.0);
    weight = float3(0.0, 0.0, 0.0);
    pdf    = 0.0;
```

with:

```hlsl
bool SampleBSDF(StandardBSDFData bsdfData, float3 wo, inout SampleGenerator sg,
                out float3 wi, out float3 weight, out float pdf, out float lobeP)
{
    wi     = float3(0.0, 0.0, 0.0);
    weight = float3(0.0, 0.0, 0.0);
    pdf    = 0.0;
    lobeP  = 0.0;
```

- [ ] **Step 2: Record the lobe probability in each lobe branch**

Still in `SampleBSDF`, the lobe-selection block currently reads:

```hlsl
    if (lobe < specProb)
    {
        // GGX half-vector (NDF) sampling in the local frame, then reflect Wo about H.
        const float a    = bsdfData.alpha;
```

Insert `lobeP = specProb;` as the first statement of the `if` body:

```hlsl
    if (lobe < specProb)
    {
        lobeP = specProb;
        // GGX half-vector (NDF) sampling in the local frame, then reflect Wo about H.
        const float a    = bsdfData.alpha;
```

And the diffuse branch currently reads:

```hlsl
    else
    {
        // Cosine-weighted diffuse hemisphere sample.
        float pdfUnused;
```

Insert `lobeP = 1.0 - specProb;`:

```hlsl
    else
    {
        lobeP = 1.0 - specProb;
        // Cosine-weighted diffuse hemisphere sample.
        float pdfUnused;
```

- [ ] **Step 3: Update the raygen call site to pass a `lobeP` local**

In `PathTracerSample.rgen`, replace:

```hlsl
        float3 nextDir;
        float3 weight;
        float  pdf;
        if (!SampleBSDF(bsdfData, wo, sg, nextDir, weight, pdf))
            break;
```

with:

```hlsl
        float3 nextDir;
        float3 weight;
        float  pdf;
        float  lobeP;
        if (!SampleBSDF(bsdfData, wo, sg, nextDir, weight, pdf, lobeP))
            break;
```

- [ ] **Step 4: Verify it compiles**

Guard: when the user requests a build, `PathTracerSample.rgen` must compile with the 7-arg `SampleBSDF`. No behavior change (the converged image is identical — `lobeP` is computed but unused). Grep confirms `SampleBSDF` has no other caller.

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "feat(rtxpt): return sampled-lobe probability from SampleBSDF (R1/G1)"
```

---

### Task 4: Carry `fireflyFilterK` in raygen; filter surface + environment emission

Initializes the per-path firefly K to 1.0, applies the soft cap to surface emission (BSDF hits) and environment emission (misses), and shrinks K after each scatter using the scatter pdf + lobe probability. The filter is gated on `ffThreshold != 0`, so a zero threshold (Task 1, filter disabled) makes every change a no-op.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen:39-48` (state init), `:72-81` (emission), `:105` (K update)

- [ ] **Step 1: Initialize the K and read the threshold once**

In `PathTracerSample.rgen`, the per-path state init currently reads:

```hlsl
    float3 throughput   = float3(1.0, 1.0, 1.0);
    float3 pathRadiance = float3(0.0, 0.0, 0.0);

    float  prevBsdfPdf   = 0.0;
    float3 prevNormal    = float3(0.0, 1.0, 0.0);
    bool   prevDidEnvNEE = false;
    const uint maxBounces    = max(g_Const.ptConsts.bounceCount, 1u);
    const uint maxNEEBounces = min(g_Const.ptConsts.maxNEEBounceCount, maxBounces);
    const bool enableNEE     = g_Const.ptConsts.NEEEnabled != 0u;
    const bool enableEnvNEE  = enableNEE && g_Const.ptConsts.environmentNEEEnabled != 0u;
```

Replace with:

```hlsl
    float3 throughput   = float3(1.0, 1.0, 1.0);
    float3 pathRadiance = float3(0.0, 0.0, 0.0);

    // G1: per-path firefly-filter K starts at 1.0 and shrinks as the path spreads (set after each scatter).
    float fireflyFilterK = 1.0;

    float  prevBsdfPdf   = 0.0;
    float3 prevNormal    = float3(0.0, 1.0, 0.0);
    bool   prevDidEnvNEE = false;
    const uint  maxBounces    = max(g_Const.ptConsts.bounceCount, 1u);
    const uint  maxNEEBounces = min(g_Const.ptConsts.maxNEEBounceCount, maxBounces);
    const bool  enableNEE     = g_Const.ptConsts.NEEEnabled != 0u;
    const bool  enableEnvNEE  = enableNEE && g_Const.ptConsts.environmentNEEEnabled != 0u;
    const float ffThreshold   = g_Const.ptConsts.fireflyFilterThreshold; // 0 disables the firefly filter.
```

- [ ] **Step 2: Filter environment emission on a miss**

The miss block currently reads:

```hlsl
        if (payload.hitFlag == 0u)
        {
            const float  misWeight   = PathTracer::ComputeBSDFEnvMISWeight(prevDidEnvNEE, prevBsdfPdf, prevNormal, rayDir);
            const float3 envRadiance = payload.emission * g_Const.ptConsts.environmentIntensity;
            pathRadiance += throughput * envRadiance * misWeight;
            break;
        }
```

Replace with:

```hlsl
        if (payload.hitFlag == 0u)
        {
            const float  misWeight   = PathTracer::ComputeBSDFEnvMISWeight(prevDidEnvNEE, prevBsdfPdf, prevNormal, rayDir);
            const float3 envRadiance = payload.emission * g_Const.ptConsts.environmentIntensity;
            float3       environmentEmission = envRadiance * misWeight;
            if (ffThreshold != 0.0)
                environmentEmission = FireflyFilter(environmentEmission, ffThreshold, fireflyFilterK);
            pathRadiance += throughput * environmentEmission;
            break;
        }
```

- [ ] **Step 3: Filter surface emission on a BSDF hit**

The surface-emission accumulation currently reads:

```hlsl
        // Accumulate emissive surfaces hit by BSDF sampling. Area-light NEE is deferred to a later lighting pass.
        pathRadiance += throughput * payload.emission;
```

Replace with:

```hlsl
        // Accumulate emissive surfaces hit by BSDF sampling. Area-light NEE is deferred to a later lighting pass.
        float3 surfaceEmission = payload.emission;
        if (ffThreshold != 0.0)
            surfaceEmission = FireflyFilter(surfaceEmission, ffThreshold, fireflyFilterK);
        pathRadiance += throughput * surfaceEmission;
```

- [ ] **Step 4: Shrink K after the scatter**

The post-scatter throughput update currently reads:

```hlsl
        throughput *= weight;

        // Russian roulette once we are past minBounceCount - unbiased early termination of dim paths.
```

Replace with:

```hlsl
        throughput *= weight;

        // G1: shrink the firefly K by the spread implied by this scatter (pdf) and the sampled lobe (lobeP).
        fireflyFilterK = ComputeNewScatterFireflyFilterK(fireflyFilterK, pdf, lobeP);

        // Russian roulette once we are past minBounceCount - unbiased early termination of dim paths.
```

- [ ] **Step 5: Verify (compile + unbiasedness)**

Compile guard: `PathTracerSample.rgen` builds (uses `FireflyFilter`/`ComputeNewScatterFireflyFilterK` from Task 2). Manual GPU check (on user request): with the FF UI toggle still disabled (Task 6 not yet done → `ReferenceFireflyFilterEnabled` defaults true, so Task 1 uploads threshold=5.0 already — the filter is **live** after this task). Confirm bright emissive/specular sparkle is visibly reduced versus pre-R1. Then, to confirm unbiasedness, temporarily force `m_ReferenceUI.ReferenceFireflyFilterEnabled=false` (or wait for Task 6) and verify the converged image matches pre-R1.

- [ ] **Step 6: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "feat(rtxpt): apply firefly filter to surface + environment emission (R1/G1)"
```

---

### Task 5: Firefly-dampen the NEE contributions

Applies the soft cap to analytic and environment NEE radiance, mirroring RTXPT-fork's `PathTracerNEE.hlsli:245-254`: recompute a NEE-specific K from the current path K and the sampling pdf, then dampen the local radiance (`f·Li`) before the importance weights and path throughput. Threading the path K through the NEE functions is a coupled signature change, so the raygen call sites are updated in the same task.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli:54-104`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen:91-97`

- [ ] **Step 1: Dampen analytic NEE**

In `PathTracer.hlsli`, replace the whole `SampleAnalyticNEE` function:

```hlsl
    float3 SampleAnalyticNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                             float3 wo, inout SampleGenerator sg)
    {
        const uint lightCount = Bridge::getLightCount();
        if (lightCount == 0u || g_Const.ptConsts.lightIntensityScale <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const uint lightIndex = min(uint(sampleNext1D(sg) * float(lightCount)), lightCount - 1u);

        LightSample light = EvalAnalyticLight(Bridge::getLight(lightIndex), hitPos);
        if (!light.valid)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, light.dir, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, light.dir, light.distance))
            return float3(0.0, 0.0, 0.0);

        return f * light.radiance * g_Const.ptConsts.lightIntensityScale * float(lightCount);
    }
```

with:

```hlsl
    float3 SampleAnalyticNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                             float3 wo, inout SampleGenerator sg, float fireflyFilterK)
    {
        const uint lightCount = Bridge::getLightCount();
        if (lightCount == 0u || g_Const.ptConsts.lightIntensityScale <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const uint lightIndex = min(uint(sampleNext1D(sg) * float(lightCount)), lightCount - 1u);

        LightSample light = EvalAnalyticLight(Bridge::getLight(lightIndex), hitPos);
        if (!light.valid)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, light.dir, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, light.dir, light.distance))
            return float3(0.0, 0.0, 0.0);

        // Local radiance f*Li before the uniform-selection reweight (x lightCount).
        const float3 fLi = f * light.radiance * g_Const.ptConsts.lightIntensityScale;

        // G1: dampen NEE fireflies. Our analytic lights are deltas with no solid-angle pdf, so we use the
        // BSDF pdf toward the light as the spread proxy (RTXPT uses SelectionPdf*SolidAnglePdf for area lights).
        float damp = 1.0;
        const float ffThreshold = g_Const.ptConsts.fireflyFilterThreshold;
        if (ffThreshold != 0.0)
        {
            const float neeK = ComputeNewScatterFireflyFilterK(fireflyFilterK, bsdfPdf, 1.0);
            damp = FireflyFilterShort(Average(fLi), ffThreshold, neeK);
        }

        return fLi * float(lightCount) * damp;
    }
```

- [ ] **Step 2: Dampen environment NEE**

Still in `PathTracer.hlsli`, replace the whole `SampleEnvironmentNEE` function:

```hlsl
    float3 SampleEnvironmentNEE(StandardBSDFData bsdfData, float3 visibilityOrigin,
                                float3 wo, inout SampleGenerator sg)
    {
        if (g_Const.ptConsts.environmentIntensity <= 0.0)
            return float3(0.0, 0.0, 0.0);

        float envPdf;
        const float3 wi = sampleCosineHemisphere(sampleNext2D(sg), bsdfData.N, envPdf);
        if (envPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, wi, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, wi, kVisibilityRayTMax))
            return float3(0.0, 0.0, 0.0);

        const float3 envRadiance = EnvMap::Eval(wi) * g_Const.ptConsts.environmentIntensity;
        const float  misWeight   = PowerHeuristic(1.0, envPdf, 1.0, bsdfPdf);
        return f * envRadiance * (misWeight / envPdf);
    }
```

with:

```hlsl
    float3 SampleEnvironmentNEE(StandardBSDFData bsdfData, float3 visibilityOrigin,
                                float3 wo, inout SampleGenerator sg, float fireflyFilterK)
    {
        if (g_Const.ptConsts.environmentIntensity <= 0.0)
            return float3(0.0, 0.0, 0.0);

        float envPdf;
        const float3 wi = sampleCosineHemisphere(sampleNext2D(sg), bsdfData.N, envPdf);
        if (envPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        const float specProb = getSpecularProbability(bsdfData, wo);
        float3      f;
        float       bsdfPdf;
        EvalBSDF(bsdfData, wo, wi, specProb, f, bsdfPdf);
        if (bsdfPdf <= 0.0)
            return float3(0.0, 0.0, 0.0);

        if (!TraceVisibilityRay(visibilityOrigin, wi, kVisibilityRayTMax))
            return float3(0.0, 0.0, 0.0);

        const float3 envRadiance = EnvMap::Eval(wi) * g_Const.ptConsts.environmentIntensity;
        const float  misWeight   = PowerHeuristic(1.0, envPdf, 1.0, bsdfPdf);

        // Local radiance f*Li before the MIS / 1-over-pdf importance weights.
        const float3 fLi = f * envRadiance;

        // G1: dampen NEE fireflies using the env-sampling pdf as the spread proxy.
        float damp = 1.0;
        const float ffThreshold = g_Const.ptConsts.fireflyFilterThreshold;
        if (ffThreshold != 0.0)
        {
            const float neeK = ComputeNewScatterFireflyFilterK(fireflyFilterK, envPdf, 1.0);
            damp = FireflyFilterShort(Average(fLi), ffThreshold, neeK);
        }

        return fLi * damp * (misWeight / envPdf);
    }
```

- [ ] **Step 3: Pass the path K from the raygen call sites**

In `PathTracerSample.rgen`, the NEE block currently reads:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sg);
            if (enableEnvNEE)
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sg);
        }
```

Replace with:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sg, fireflyFilterK);
            if (enableEnvNEE)
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sg, fireflyFilterK);
        }
```

- [ ] **Step 4: Verify (compile + unbiasedness)**

Compile guard: raygen builds with the 6-arg NEE functions. Manual GPU check (on user request): NEE highlights (e.g. a strong punctual light grazing a glossy surface) show fewer sparkles; with the filter disabled the converged image is unchanged.

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli \
        Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "feat(rtxpt): firefly-dampen analytic + environment NEE (R1/G1)"
```

---

### Task 6: Enable the firefly UI controls (G1 runnable milestone)

Removes the `BeginDisabled`/placeholder-tooltip scaffolding around the "FireflyFilter (reference *)" checkbox and "FF Threshold" input, wiring both through `ResetOnChange` so editing them restarts accumulation. The toggle already drives the uploaded threshold (Task 1, Step 3).

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp:532-547`

- [ ] **Step 1: Replace the disabled firefly block with live controls**

In `UpdateUI`, replace this block:

```cpp
            // Adaptive firefly filter: Phase R1 (G1). Each control gets its own
            // BeginDisabled/EndDisabled scope so IsItemHovered() in PlaceholderTooltip()
            // attaches a tooltip to that specific item rather than only the last one.
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("FireflyFilter (reference *)", &m_ReferenceUI.ReferenceFireflyFilterEnabled);
            ImGui::EndDisabled();
            PlaceholderTooltip("Adaptive firefly filter lands in Phase R1.");
            if (m_ReferenceUI.ReferenceFireflyFilterEnabled)
            {
                ImGui::Indent(Indent);
                ImGui::BeginDisabled(true);
                ImGui::InputFloat("FF Threshold", &m_ReferenceUI.ReferenceFireflyFilterThreshold, 0.1f, 0.2f, "%.5f");
                ImGui::EndDisabled();
                PlaceholderTooltip("Adaptive firefly filter threshold lands in Phase R1.");
                ImGui::Unindent(Indent);
            }
```

with:

```cpp
            // Adaptive firefly filter (G1, live). Disabling it uploads a zero threshold so the soft
            // cap is a no-op and the converged image is identical to the filter-on image.
            ResetOnChange(ImGui::Checkbox("FireflyFilter (reference *)", &m_ReferenceUI.ReferenceFireflyFilterEnabled),
                          "Firefly filter toggled");
            if (m_ReferenceUI.ReferenceFireflyFilterEnabled)
            {
                ImGui::Indent(Indent);
                ResetOnChange(ImGui::InputFloat("FF Threshold", &m_ReferenceUI.ReferenceFireflyFilterThreshold, 0.1f, 0.2f, "%.5f"),
                              "Firefly threshold changed");
                ImGui::Unindent(Indent);
            }
```

- [ ] **Step 2: Verify (build + UI)**

Compile guard: `RTXPTSample.cpp` builds (`ResetOnChange`/`PlaceholderTooltip` lambdas already exist; `PlaceholderTooltip` is still used by other placeholder controls, so no unused-variable warning). Manual UI check (on user request): the FireflyFilter checkbox and FF Threshold input are now interactive; toggling/editing them restarts accumulation; unchecking the box and letting the image converge reproduces the pre-R1 converged image (unbiasedness).

- [ ] **Step 3: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): enable firefly-filter UI controls (R1/G1)"
```

---

### Task 7: NEE at all bounces — fix the `MaxNEEBounces` default (G2)

Changes the default NEE bounce budget so next-event estimation runs at every surface vertex up to the bounce limit, restoring indirect direct-lighting (analytic delta lights can never be hit by BSDF sampling, so without NEE past bounce 0 secondary surfaces receive zero analytic/sky direct light). The control stays as an optional clamp; only its default changes.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp:70` (struct default), `:157` (member default)

- [ ] **Step 1: Raise the GPU-mirror struct default**

In `RTXPTSample.hpp`, `PathTracerConstants`, change:

```cpp
    Uint32 maxNEEBounceCount      = 1;    // Limits NEE work to the first N path bounces to avoid TDR-heavy dispatches.
```

to:

```cpp
    Uint32 maxNEEBounceCount      = 16;   // Default covers the full bounce budget (NEE at every vertex); a lower value is an optional perf/TDR clamp.
```

- [ ] **Step 2: Raise the authoritative member default**

`UpdateFrameConstants` uploads from `m_MaxNEEBounces` (the source of truth), so change its initializer too. In the `RTXPTSample` member list, change:

```cpp
    Uint32                      m_MaxNEEBounces            = 1;
```

to:

```cpp
    Uint32                      m_MaxNEEBounces            = 16;
```

- [ ] **Step 3: Verify (build + behavior)**

Compile guard: builds unchanged (value-only edit). Manual GPU check (on user request): in a scene where a punctual or sky light directly lights surface A and surface A indirectly lights shadowed surface B, B now receives that bounce light (it was black past bounce 0 before). The "NEE bounces" slider defaults to 16 and clamps against "Max bounces" in raygen (`min(maxNEEBounceCount, maxBounces)`). Converged output is unchanged where it was already correct (more bounces only adds previously-missing energy; it does not double-count, because BSDF sampling can never hit a delta light).

- [ ] **Step 4: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSample.hpp
git commit -m "feat(rtxpt): run NEE at all bounces by default (R1/G2)"
```

---

### Task 8: Add the stateless per-(pixel, vertex, sample) generator + effect seeds (G3)

Adds a stateless generator constructor and per-effect decorrelation salts to the sample-generator header, mirroring RTXPT-fork's `UniformSampleSequenceGenerator::make` and `SampleGeneratorEffectSeed`. The existing `SampleGenerator` struct and `sampleNext1D/2D` are reused unchanged; only the seeding changes. Full Sobol/Owen low-discrepancy sampling remains deferred to Phase R5 (G9).

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli:39-52`

- [ ] **Step 1: Add effect-seed constants and the stateless constructor**

In `SampleGenerators.hlsli`, after the existing `SampleGenerator_make` function:

```hlsl
SampleGenerator SampleGenerator_make(uint2 pixelPos, uint frameSeed)
{
    SampleGenerator sg;
    const uint      PixelSeed = (pixelPos.x << 16) | (pixelPos.y & 0xffffu);
    sg.State                  = Hash32Combine(Hash32(frameSeed), PixelSeed);
    return sg;
}
```

insert:

```hlsl

// Per-effect decorrelation salts. Mirror RTXPT-fork's SampleGeneratorEffectSeed
// (D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/SampleGenerators.hlsli:16).
static const uint kSampleEffect_Base                = 0u;
static const uint kSampleEffect_ScatterBSDF         = 1u;
static const uint kSampleEffect_NextEventEstimation = 2u;
static const uint kSampleEffect_NEELightSampler     = 3u;
static const uint kSampleEffect_RussianRoulette     = 6u;

// Stateless per-(pixel, vertex, sample) seeding (G3): each path vertex + effect draws a decorrelated
// sequence, so bounces and dimensions no longer share one forward hash chain. Mirrors RTXPT-fork's
// UniformSampleSequenceGenerator::make (StatelessSampleGenerators.hlsli:191). Full Sobol/Owen
// low-discrepancy sampling is deferred to Phase R5 (G9).
SampleGenerator SampleGenerator_makeStateless(uint2 pixelPos, uint vertexIndex, uint sampleIndex, uint effectSeed)
{
    SampleGenerator sg;
    const uint baseHash = Hash32Combine(Hash32(vertexIndex + 0x035F9F29u), (pixelPos.x << 16) | (pixelPos.y & 0xffffu));
    uint       h        = Hash32Combine(baseHash, effectSeed);
    h                   = Hash32Combine(h, sampleIndex);
    sg.State            = h;
    return sg;
}
```

- [ ] **Step 2: Verify it compiles**

Header-only; unused until Task 9. Guard: shaders including `SampleGenerators.hlsli` still build (new names `kSampleEffect_*` and `SampleGenerator_makeStateless` collide with nothing — grep before this task confirmed). `SampleGenerator_make` is intentionally kept (still referenced by the mapping doc; harmless inline function).

- [ ] **Step 3: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli
git commit -m "feat(rtxpt): add stateless per-vertex sample generator + effect seeds (R1/G3)"
```

---

### Task 9: Raygen uses per-vertex / per-effect generators (G3 runnable milestone)

Replaces the single per-(pixel, frame) chained generator with stateless generators seeded per path vertex and per effect, keyed by the accumulation `sampleIndex`. The camera ray uses vertex 0 / Base; each bounce `b` uses vertex `b+1` with distinct seeds for light selection, env NEE, BSDF scatter, and Russian roulette. This only changes the noise pattern, not any estimator's expected value, so the converged image is unchanged.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` (seed setup `:26-30`, loop body)

- [ ] **Step 1: Replace the camera-ray seed setup**

Near the top of `main`, replace:

```hlsl
    const uint  frameSeed = asuint(g_Const.viewportSizeAndFrameIndex.w);
    SampleGenerator sg    = SampleGenerator_make(pixel, frameSeed);

    // Jitter inside the pixel - gives free anti-aliasing across accumulated samples.
    const float2 jitter = sampleNext2D(sg);
```

with:

```hlsl
    // G3: stateless per-(pixel, vertex, sample) seeding keyed by the accumulation sample index.
    // Each path vertex + effect gets its own decorrelated generator (full Sobol/Owen is Phase R5).
    const uint sampleIndex = g_Const.ptConsts.sampleIndex;

    // Jitter inside the pixel - free anti-aliasing across accumulated samples (vertex 0 / Base effect).
    SampleGenerator sgCamera = SampleGenerator_makeStateless(pixel, 0u, sampleIndex, kSampleEffect_Base);
    const float2    jitter   = sampleNext2D(sgCamera);
```

- [ ] **Step 2: Add the vertex index at the top of the loop body**

The loop body currently begins:

```hlsl
    [loop]
    for (uint bounce = 0u; bounce < maxBounces; ++bounce)
    {
        RayDesc ray;
```

Replace with:

```hlsl
    [loop]
    for (uint bounce = 0u; bounce < maxBounces; ++bounce)
    {
        const uint vertexIndex = bounce + 1u;

        RayDesc ray;
```

- [ ] **Step 3: Seed the NEE generators per vertex**

After Task 5 the NEE block reads:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sg, fireflyFilterK);
            if (enableEnvNEE)
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sg, fireflyFilterK);
        }
```

Replace with:

```hlsl
        const bool useNEE = enableNEE && bounce < maxNEEBounces;
        if (useNEE)
        {
            SampleGenerator sgNEELight = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEELightSampler);
            pathRadiance += throughput * PathTracer::SampleAnalyticNEE(bsdfData, payload.worldPos, visibilityOrigin, wo, sgNEELight, fireflyFilterK);
            if (enableEnvNEE)
            {
                SampleGenerator sgEnvNEE = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NextEventEstimation);
                pathRadiance += throughput * PathTracer::SampleEnvironmentNEE(bsdfData, visibilityOrigin, wo, sgEnvNEE, fireflyFilterK);
            }
        }
```

- [ ] **Step 4: Seed the BSDF-scatter generator per vertex**

After Task 3 the scatter call reads:

```hlsl
        float3 nextDir;
        float3 weight;
        float  pdf;
        float  lobeP;
        if (!SampleBSDF(bsdfData, wo, sg, nextDir, weight, pdf, lobeP))
            break;
```

Replace with:

```hlsl
        float3          nextDir;
        float3          weight;
        float           pdf;
        float           lobeP;
        SampleGenerator sgScatter = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_ScatterBSDF);
        if (!SampleBSDF(bsdfData, wo, sgScatter, nextDir, weight, pdf, lobeP))
            break;
```

- [ ] **Step 5: Seed the Russian-roulette generator per vertex**

The Russian-roulette block currently reads:

```hlsl
        if (bounce >= g_Const.ptConsts.minBounceCount)
        {
            const float survive = clamp(max(throughput.x, max(throughput.y, throughput.z)), 0.05, 1.0);
            if (sampleNext1D(sg) > survive)
                break;
            throughput /= survive;
        }
```

Replace with:

```hlsl
        if (bounce >= g_Const.ptConsts.minBounceCount)
        {
            SampleGenerator sgRR     = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_RussianRoulette);
            const float     survive  = clamp(max(throughput.x, max(throughput.y, throughput.z)), 0.05, 1.0);
            if (sampleNext1D(sgRR) > survive)
                break;
            throughput /= survive;
        }
```

- [ ] **Step 6: Confirm the old `sg` is fully gone**

Grep the file to ensure no orphaned reference to the removed single generator remains:

```bash
cd DiligentSamples
grep -n "\bsg\b" Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: **no matches** (every use is now `sgCamera` / `sgNEELight` / `sgEnvNEE` / `sgScatter` / `sgRR`). If `SampleGenerator_make` or `frameSeed` still appears, the Step 1 replacement was missed.

- [ ] **Step 7: Verify (compile + unbiasedness)**

Compile guard: raygen builds (uses `SampleGenerator_makeStateless` + `kSampleEffect_*` from Task 8). Manual GPU check (on user request): the **converged** image matches the pre-R1 / pre-Task-9 converged image (reseeding only changes the noise realization, not the expected value); early-sample noise is decorrelated across bounces (less structured streaking). Note `viewportSizeAndFrameIndex.w` (the frame index) is no longer used for seeding — seeding is now keyed by the accumulation `sampleIndex`, matching RTXPT-fork.

- [ ] **Step 8: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git commit -m "feat(rtxpt): decorrelated per-vertex/per-effect seeding in raygen (R1/G3)"
```

---

### Task 10: Update open-work markers + the fork-mapping doc

Retargets the in-code roadmap text that named R1 as pending, and records the new symbol correspondences so future upstream merges stay near-mechanical.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp:761-766` (Status/Debug roadmap)
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` (T-B and T-E tables)

- [ ] **Step 1: Remove the R1 line from the in-UI roadmap**

In `UpdateUI`'s "Status / Debug" section, the roadmap currently reads:

```cpp
        ImGui::TextColored(CategoryColor, "Roadmap (open work):");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R1): adaptive firefly filter, NEE at all bounces, decorrelated seeding.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R3): light importance sampling (RIS/WRS) + photometric units.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R4): HDR environment map with importance sampling + MIS.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R5): VNDF/Frostbite/multi-scatter BSDF + low-discrepancy sampler.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R6): transmission / nested dielectrics / ALPHA_MODE_BLEND.");
```

Remove the R1 line (R1 is now delivered):

```cpp
        ImGui::TextColored(CategoryColor, "Roadmap (open work):");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R2): emissive-triangle area-light NEE + MIS.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R3): light importance sampling (RIS/WRS) + photometric units.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R4): HDR environment map with importance sampling + MIS.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R5): VNDF/Frostbite/multi-scatter BSDF + low-discrepancy sampler.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R6): transmission / nested dielectrics / ALPHA_MODE_BLEND.");
```

(The R2 line was previously absent; adding it keeps the roadmap complete now that R1 is done.)

- [ ] **Step 2: Add T-B mapping rows for the new generator + effect seeds**

In `RTXPT_FORK_MAPPING.md`, table **T-B** ends with the `sampleCosineHemisphere` row:

```markdown
| `SampleCosineHemisphere` | `sampleCosineHemisphere` (style) | RTXPT-fork's local-frame helper differs; keep ours and record divergence |
```

Add two rows immediately after it:

```markdown
| (R1/G3 new) | `SampleGenerator_makeStateless` | stateless per-(pixel,vertex,sample) seed; mirrors RTXPT `UniformSampleSequenceGenerator::make`. Full Sobol/Owen deferred to R5 (G9) |
| (R1/G3 new) | `kSampleEffect_*` | mirrors RTXPT `SampleGeneratorEffectSeed` (Base/ScatterBSDF/NextEventEstimation/NEELightSampler/RussianRoulette) |
```

- [ ] **Step 3: Add a T-E mapping row for the firefly helpers**

In table **T-E**, after the `PowerHeuristic` row:

```markdown
| `RTXPTPowerHeuristic(PdfA, PdfB)` | `PowerHeuristic(nf, fPdf, ng, gPdf)` = RTXPT-fork | call with `(1, pdfA, 1, pdfB)` |
```

add:

```markdown
| (R1/G1 new) | `ComputeNewScatterFireflyFilterK` / `FireflyFilter` / `FireflyFilterShort` / `Average` = RTXPT-fork | runtime-gated by `fireflyFilterThreshold==0` instead of RTXPT's compile-time `RTXPT_FIREFLY_FILTER`; uses `acos`/`sqrt` not `FastACos`/`FastSqrt` |
```

- [ ] **Step 4: Verify**

Doc-only and UI-text-only edits. Compile guard: `RTXPTSample.cpp` still builds. No runtime behavior change.

- [ ] **Step 5: Commit**

```bash
cd DiligentSamples
git add Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): retarget R1 roadmap + map firefly/seeding symbols (R1)"
```

---

## Phase-Level Verification (run on explicit user request only)

Per the workspace rule, do not auto-run these — surface them for the user.

1. **Build (both backends).** Configure + build the sample (`build-x64-Debug.bat` or the documented CMake flow). The `static_assert`s in `RTXPTSample.hpp` enforce the settings-layout contract at compile time. Confirm the `RTXPT` sample launches and renders on **D3D12** and **Vulkan**.
2. **Unbiasedness (primary).** With "FireflyFilter (reference *)" **unchecked**, let the image converge; capture it. Check the box, let it converge again; the two converged images must match (only transient noise differs). This proves G1 only changes variance and that the G3 reseeding did not shift the expected value.
3. **G1 visual delta.** With the filter **on**, bright specular/NEE/emissive sparkle is visibly suppressed at low sample counts versus the pre-R1 build.
4. **G2 behavior.** In a scene with one punctual or sky light, a surface lit only by indirect bounce light off a directly-lit surface is now illuminated (was black past bounce 0 before). Lowering "NEE bounces" below "Max bounces" reintroduces the deficit (confirms the control still clamps).
5. **G3 noise structure.** Early accumulation frames show decorrelated, less-structured noise across bounces; converged output unchanged (see step 2).
6. **clang-format.** The edited C++ files (`RTXPTSample.hpp`, `RTXPTSample.cpp`) must pass DiligentCore clang-format validation (`BuildTools/FormatValidation/validate_format_win.bat` or the `DiligentCore-ValidateFormatting` target). Match the surrounding aligned-declaration style in the structs/UI. Copyright headers already read 2026 — no date bump needed.

## Open-Work / Marker Policy After R1

- **Resolved:** the in-UI `TODO(RTXPT-Port Phase R1)` roadmap line (removed in Task 10). G1/G2/G3 are delivered.
- **Untouched (intentional):** the raygen trailing `TODO(RTXPT-Port Phase R2/R3/R4)` and `Phase R6` markers (later phases). The "FireflyFilter (reference *)" label keeps its `*` to match RTXPT-fork's "reference-mode-only" annotation.
- **Deferred within G3:** full Sobol/Owen low-discrepancy sequences (the `bhos_*` machinery) remain a Phase R5 (G9) item; R1 ships only the stateless uniform reseed. This is recorded in the new T-B mapping row.
