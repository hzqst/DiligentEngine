# RTXPT Phase R5 BSDF Fidelity and Low-Discrepancy Sampler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the DiligentEngine RTXPT reference path tracer's BSDF model and BSDF sampler to RTXPT-fork reference-mode parity for G8 and G9: bounded-VNDF GGX, Frostbite diffuse, multi-scatter specular energy compensation, diffuse-bounce limiting, and Sobol/Owen low-discrepancy BSDF samples.

**Architecture:** Keep the raygen-driven, one-recursion-level Diligent path-tracer architecture. Port the RTXPT-fork math into the already-aligned `PathTracer/` shader tree: `BxDF.hlsli` remains the BSDF facade, a new `Utils/StatelessSampleGenerators.hlsli` carries the Sobol/Owen sequence generator, and `PathTracerSample.rgen` pre-generates BSDF sample dimensions exactly like RTXPT-fork. Match RTXPT-fork's control surface: `diffuseBounceCount` is a runtime frame constant, while `EnableLDSamplerForBSDF` is compiled as `RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF` and recreates the RT PSO when changed.

**Tech Stack:** HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, C++17 sample code under `DiligentSamples/Samples/RTXPT/src`, Diligent ray-tracing PSO shader macros, Dear ImGui, CMake shader source registration. `DiligentSamples` is a git submodule; implementation commits in this plan are made inside `DiligentSamples/`.

---

## Context You Need Before Starting

Phase R0.5 has landed, so use the current RTXPT-fork-aligned paths, not the older names from the spec:

| Spec name | Current path |
|---|---|
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTRandom.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli` |
| `RTXPTBSDF.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` |
| `RTXPTShaderShared.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` |
| C++ frame constants mirror | `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` |

Current baseline:

- `BxDF.hlsli` has a two-lobe Lambert + GGX model, NDF half-vector sampling, no `lobe` output, no delta lobe collapse, and no multi-scatter specular compensation.
- `SampleGenerators.hlsli` has the R1 stateless uniform generator and explicitly says full Sobol/Owen is deferred to R5.
- `RTXPTSample.cpp` already shows `Max diffuse bounces` and `Enable LD sampler for BSDF`, but both controls are disabled placeholders.
- `PathTracerConstants` has no `diffuseBounceCount`. Its C++/HLSL size is 64 bytes and `SampleConstants` is 352 bytes.
- `RTXPTRayTracingPass.cpp` already has `ShaderMacroHelper`, so R5 can add the LD macro without changing the shader-loading model.

RTXPT-fork anchors to read before coding:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:43-52` - `GGXSamplingBVNDF`, `kMinGGXAlpha`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:151-190` - Frostbite diffuse evaluation.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:250-372` - `MultiScatterSpecularApprox`, `SpecularReflectionMicrofacet`, BVNDF sample/pdf path.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:709-968` - `FalcorBSDF` lobe probabilities, `sample`, and `evalPdf`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/Microfacet.hlsli:97-213` - bounded-VNDF pdf and sample functions.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/NoiseAndSequences.hlsli:49-260` - reference hash, `Hash32ToFloat`, Sobol, Owen scrambling helpers.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/SampleGenerators.hlsli:16-76` - effect seeds, LD cutover macro, `sampleNext*`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli:16-159` - `SampleGeneratorVertexBase`, `SampleSequenceGenerator`, `UniformSampleSequenceGenerator`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli:40-45,250-270,350-374` - diffuse-bounce termination/counting and BSDF sample generation.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1037,1500` - LD sampler macro and `diffuseBounceCount` upload.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:171,183` and `SampleUI.cpp:829-830,1061` - UI defaults and controls.

Do not copy NVIDIA file headers, large comments, or wholesale source blocks. Port the behavior, names, constants, and control flow into Diligent-owned code.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` - add `diffuseBounceCount`, keep constants 16-byte aligned, update size guards.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - mirror the same constants layout.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - add a small RT-pass settings dirty flag if needed for macro recreation.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - upload `diffuseBounceCount`, enable the R5 UI controls, and recreate the RT PSO when the LD macro changes.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` - accept the LD macro flag and define `RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` - port Frostbite diffuse, BVNDF sampling/pdf, multi-scatter specular compensation, lobe IDs, and pre-generated sample input.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli` - align reference-mode hash-to-float conversion and add `sampleNext3D/4D` helpers.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli` - RTXPT-fork-style Sobol/Owen sample sequence generator for BSDF sample blocks.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - pre-generate BSDF samples, choose LD/uniform path, track diffuse bounces, and call the new `SampleBSDF` signature.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register the new HLSL include.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record R5 sampler/BSDF mappings and the intentional Diligent-native divergences.

## Cross-Cutting Contracts

- **Frame constants layout:** `PathTracerConstants` grows from 64 to 80 bytes by adding `diffuseBounceCount` plus explicit padding. `SampleConstants` grows from 352 to 368 bytes. Update C++ and HLSL mirrors together.
- **LD toggle parity:** `EnableLDSamplerForBSDF` is a shader macro, matching RTXPT-fork. Toggling it recreates the ray-tracing pass and resets accumulation.
- **BSDF sample contract:** `SampleBSDF` no longer consumes `inout SampleGenerator`; it consumes a pre-generated sample triplet and returns `lobe`, `lobeP`, `pdf`, `wi`, and `weight`. This mirrors RTXPT-fork's `FalcorBSDF::sample` separation between sample generation and BxDF sampling.
- **MIS consistency:** `EvalBSDF` and `SampleBSDF` must use the same mixture pdf. Direct-light, emissive, and environment MIS call sites continue to rely on `EvalBSDF`, so a pdf mismatch here is a phase failure.
- **Diffuse bounce count:** `diffuseBounceCount` controls both max diffuse bounces and the LD window. Total path length is still capped by `bounceCount`.
- **Delta events:** Near-mirror GGX lobes collapse to a delta event when `alpha < kMinGGXAlpha`. The returned `pdf` is zero for delta reflection, which the existing firefly and MIS helpers already treat as a delta sentinel.
- **Backend scope:** no new resources, bindings, payload fields, SBT records, ray types, or recursion depth changes are required.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: current R4/R5 baseline

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Do not overwrite dirty files without reading them first.

- [ ] **Step 2: Confirm the current R5 placeholders**

Run:

```powershell
rg -n "DiffuseBounceCount|EnableLDSamplerForBSDF|Phase R5|low-discrepancy|Sobol|Owen|SampleBSDF" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `RTXPTSample.hpp`, `RTXPTSample.cpp`, `SampleGenerators.hlsli`, `PathTracerSample.rgen`, and `BxDF.hlsli`.

- [ ] **Step 3: Confirm the reference anchors are available**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Materials\BxDF.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Utils\StatelessSampleGenerators.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Utils\NoiseAndSequences.hlsli
```

Expected: all three commands print `True`.

- [ ] **Step 4: Commit nothing**

Expected: no commit in Task 0. This task only establishes the starting point.

---

### Task 1: Wire R5 Controls and Constants

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Grow the C++ frame constants**

In `RTXPTFrameConstants.hpp`, replace the tail of `PathTracerConstants` with:

```cpp
    Uint32 NEEFullSamples         = 1;    // G5: visibility-tested full samples.
    Uint32 NEEMISType             = 0;    // G5 UI parity: 0=Full; approximate modes remain disabled.
    float  fireflyFilterThreshold = 0.0f; // G1 adaptive firefly filter; 0 disables the filter.
    float  exposureScale          = 1.0f; // Scene camera exposure multiplier before in-raygen ACES.

    Uint32 diffuseBounceCount = 2; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    Uint32 _paddingR5_0       = 0;
    Uint32 _paddingR5_1       = 0;
    Uint32 _paddingR5_2       = 0;
};
static_assert(sizeof(PathTracerConstants) == 80, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

Then update the `SampleConstants` guard:

```cpp
static_assert(sizeof(SampleConstants) == 368, "SampleConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Mirror the HLSL constants layout**

In `PathTracerShared.h`, apply the same tail fields:

```hlsl
    uint  NEEFullSamples;         // G5: visibility-tested full samples.
    uint  NEEMISType;             // G5 UI parity: 0=Full; approximate modes remain disabled.
    float fireflyFilterThreshold; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter entirely.
    float exposureScale;          // Scene camera exposure multiplier applied before the in-raygen ACES curve.

    uint diffuseBounceCount; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    uint _paddingR5_0;
    uint _paddingR5_1;
    uint _paddingR5_2;
};
```

Update the comment above `PathTracerConstants` from 64 bytes to 80 bytes and the `SampleConstants` comment from 352 bytes to 368 bytes.

- [ ] **Step 3: Upload `diffuseBounceCount` each frame**

In `RTXPTSample.cpp`, inside `UpdateFrameConstants`, after assigning `NEEMISType`, add:

```cpp
    m_LastFrameConstants.ptConsts.diffuseBounceCount =
        static_cast<Uint32>(std::clamp(m_ReferenceUI.DiffuseBounceCount, 0, 16));
```

- [ ] **Step 4: Track when the RT PSO must be recreated**

In `RTXPTSample.hpp`, add this member near the existing dirty flags:

```cpp
    bool                           m_RayTracingPassSettingsDirty = false;
```

In `RTXPTSample.cpp::Update`, after the lights-baker dirty handling and before `if (RecreatePhase4Passes)`, add:

```cpp
    if (m_RayTracingPassSettingsDirty)
    {
        RecreatePhase4Passes           = true;
        m_RayTracingPassSettingsDirty  = false;
    }
```

- [ ] **Step 5: Enable the diffuse-bounce UI control**

In `UpdateUI`, replace the disabled `Max diffuse bounces` block with:

```cpp
            int DiffuseBouncesUI = m_ReferenceUI.DiffuseBounceCount;
            if (ResetOnChange(ImGui::SliderInt("Max diffuse bounces", &DiffuseBouncesUI, 0, 16),
                              "Max diffuse bounces changed"))
                m_ReferenceUI.DiffuseBounceCount = std::clamp(DiffuseBouncesUI, 0, 16);
```

- [ ] **Step 6: Enable the LD sampler UI control**

In `UpdateUI`, replace the disabled `Enable LD sampler for BSDF` block with:

```cpp
        if (ResetOnChange(ImGui::Checkbox("Enable LD sampler for BSDF", &m_ReferenceUI.EnableLDSamplerForBSDF),
                          "BSDF LD sampler toggled"))
            m_RayTracingPassSettingsDirty = true;
```

- [ ] **Step 7: Add the LD macro to the RT pass**

In `RTXPTRayTracingPass.hpp`, add a `bool EnableLDSamplerForBSDF` parameter to `Initialize` immediately after `EnableMaterialTextures`.

In `RTXPTRayTracingPass.cpp`, add the same parameter in the definition and add this macro after the material-texture macro block:

```cpp
    Macros.Add("RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF", EnableLDSamplerForBSDF ? 1 : 0);
```

In `RTXPTSample.cpp::CreatePhase4Passes`, pass `m_ReferenceUI.EnableLDSamplerForBSDF` to every `m_RayTracingPass.Initialize(...)` call at the matching parameter position.

- [ ] **Step 8: Verify control/constant plumbing**

Run:

```powershell
rg -n "diffuseBounceCount|RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF|m_RayTracingPassSettingsDirty|Enable LD sampler for BSDF|Max diffuse bounces" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in the C++/HLSL constants mirrors, `RTXPTSample`, and `RTXPTRayTracingPass`.

- [ ] **Step 9: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire R5 BSDF sampler controls" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Port the RTXPT-Fork BSDF Model

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`

- [ ] **Step 1: Add RTXPT-fork-aligned constants and lobe bits**

At the top of `BxDF.hlsli`, keep `K_PI` / `K_1_PI` and replace the roughness-only floor with:

```hlsl
static const float kMinCosTheta = 1e-6;
static const float kMinGGXAlpha = 0.0064;

static const uint kBSDFLobeDiffuseReflection  = 0x01u;
static const uint kBSDFLobeSpecularReflection = 0x02u;
static const uint kBSDFLobeDeltaReflection    = 0x04u;
static const uint kBSDFLobeDelta              = kBSDFLobeDeltaReflection;
```

- [ ] **Step 2: Preserve roughness in `StandardBSDFData`**

Replace the struct and constructor with a version that stores perceptual roughness separately from alpha:

```hlsl
struct StandardBSDFData
{
    float3 N;
    float3 diffuse;
    float3 specular;
    float  roughness;
    float  alpha;
};

StandardBSDFData MakeStandardBSDFData(float3 N, float3 baseColor, float metallic, float roughness)
{
    const float r = saturate(roughness);
    const float alpha = r * r;

    StandardBSDFData bsdfData;
    bsdfData.N        = N;
    bsdfData.roughness = r;
    bsdfData.alpha    = (alpha < kMinGGXAlpha) ? 0.0 : max(alpha, kMinGGXAlpha);
    bsdfData.diffuse  = baseColor * (1.0 - metallic);
    bsdfData.specular = lerp(float3(0.04, 0.04, 0.04), baseColor, metallic);
    return bsdfData;
}
```

- [ ] **Step 3: Add Frostbite diffuse and multi-scatter helpers**

Add these helpers before `EvalBSDF`:

```hlsl
float3 evalDiffuseFrostbiteWeight(float3 albedo, float roughness, float3 wi, float3 wo)
{
    const float3 h = normalize(wi + wo);
    const float  woDotH = saturate(dot(wo, h));
    const float  energyBias = lerp(0.0, 0.5, roughness);
    const float  energyFactor = lerp(1.0, 1.0 / 1.51, roughness);
    const float  fd90 = energyBias + 2.0 * woDotH * woDotH * roughness;
    const float  wiScatter = evalFresnelSchlick(1.0, fd90, wi.z);
    const float  woScatter = evalFresnelSchlick(1.0, fd90, wo.z);
    return albedo * wiScatter * woScatter * energyFactor;
}

float EmsApprox(float r2, float NdV)
{
    const float r4  = r2 * r2;
    const float nv0 = 0.2 * r2;
    const float nv1 = 0.32 * r2 + 1.94 * r4;
    return lerp(nv0, nv1, NdV);
}

float3 MultiScatterSpecularApprox(float alpha, float NdV, float3 F0)
{
    return 1.0 + F0 * EmsApprox(alpha, NdV);
}
```

If `evalFresnelSchlick` only has the `float3` overload, add this scalar overload beside it:

```hlsl
float evalFresnelSchlick(float f0, float f90, float cosTheta)
{
    const float f = pow(saturate(1.0 - cosTheta), 5.0);
    return f0 + (f90 - f0) * f;
}
```

- [ ] **Step 4: Add BVNDF sample/pdf helpers**

Add the RTXPT-fork bounded-VNDF helpers using Diligent-owned comments and the same math as `Microfacet.hlsli`:

```hlsl
float evalPdfGGX_BVNDF(float alphaValue, float3 wiLocal, float3 hLocal)
{
    const float2 alpha = alphaValue.xx;
    const float  ndf   = evalNdfGGX(alphaValue, hLocal.z);
    const float2 ai    = alpha * wiLocal.xy;
    const float  len2  = dot(ai, ai);
    const float  t     = sqrt(len2 + wiLocal.z * wiLocal.z);
    const float  a     = saturate(min(alpha.x, alpha.y));
    const float  s     = 1.0 + length(wiLocal.xy);
    const float  a2    = a * a;
    const float  s2    = s * s;
    const float  k     = (1.0 - a2) * s2 / max(s2 + a2 * wiLocal.z * wiLocal.z, 1e-7);
    return ndf / max(2.0 * (k * wiLocal.z + t), 1e-7);
}

float3 sampleGGX_BVNDF(float alphaValue, float3 wiLocal, float2 rand)
{
    const float2 alpha = alphaValue.xx;
    const float3 iStd  = normalize(float3(wiLocal.xy * alpha, wiLocal.z));
    const float  phi   = 2.0 * K_PI * rand.x;
    const float  a     = saturate(min(alpha.x, alpha.y));
    const float  s     = 1.0 + length(wiLocal.xy);
    const float  a2    = a * a;
    const float  s2    = s * s;
    const float  k     = (1.0 - a2) * s2 / max(s2 + a2 * wiLocal.z * wiLocal.z, 1e-7);
    const float  b     = (wiLocal.z > 0.0) ? k * iStd.z : iStd.z;
    const float  z     = mad(1.0 - rand.y, 1.0 + b, -b);
    const float  sinTheta = sqrt(saturate(1.0 - z * z));
    const float3 oStd = float3(sinTheta * cos(phi), sinTheta * sin(phi), z);
    const float3 mStd = iStd + oStd;
    return normalize(float3(mStd.xy * alpha, mStd.z));
}
```

- [ ] **Step 5: Replace BSDF evaluation with Frostbite + BVNDF pdf**

Update `EvalBSDF` so it:

- rejects `min(NdotL, NdotV) < kMinCosTheta`;
- transforms `wo` and `wi` into the local basis for BVNDF pdf evaluation;
- uses `evalDiffuseFrostbiteWeight(...) * K_1_PI`;
- multiplies the specular term by `MultiScatterSpecularApprox(bsdfData.alpha, NdotV, bsdfData.specular)`;
- computes the mixture pdf as `specProb * pdfSpecular + (1 - specProb) * pdfDiffuse`;
- returns `pdfSpecular = 0` when `bsdfData.alpha == 0.0`.

The resulting shape should keep the existing public signature:

```hlsl
void EvalBSDF(StandardBSDFData bsdfData, float3 wo, float3 wi, float specProb, out float3 f, out float pdf)
```

- [ ] **Step 6: Change `SampleBSDF` to consume pre-generated samples and return lobe bits**

Replace the `SampleBSDF` signature with:

```hlsl
bool SampleBSDF(StandardBSDFData bsdfData, float3 wo, float3 preGeneratedSample,
                out float3 wi, out float3 weight, out float pdf, out uint lobe, out float lobeP)
```

Required behavior:

- `preGeneratedSample.xy` samples the selected lobe.
- `preGeneratedSample.z` selects diffuse vs specular.
- Specular sampling uses `sampleGGX_BVNDF`.
- Delta specular uses `wi = reflect(-wo, bsdfData.N)`, `pdf = 0.0`, `lobe = kBSDFLobeDeltaReflection`, and `lobeP = specProb`.
- Non-delta specular uses `evalPdfGGX_BVNDF` through `EvalBSDF`.
- Diffuse sampling uses `sampleCosineHemisphere` and returns `kBSDFLobeDiffuseReflection`.
- `weight = f / pdf` for non-delta events.

- [ ] **Step 7: Verify the BSDF API surface**

Run:

```powershell
rg -n "kMinGGXAlpha|sampleGGX_BVNDF|evalPdfGGX_BVNDF|MultiScatterSpecularApprox|evalDiffuseFrostbiteWeight|kBSDFLobe|SampleBSDF" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "SampleBSDF\\([^,]+,[^,]+,\\s*inout SampleGenerator|NDF\\) sampling|Lambertian diffuse" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
```

Expected: first command finds all new helpers. Second command prints no old `inout SampleGenerator` signature and no stale NDF/Lambert-only comments.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): port R5 BSDF fidelity model" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Add the RTXPT-Fork-Style Stateless LD Sampler

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Align hash-to-float conversion with RTXPT reference mode**

In `SampleGenerators.hlsli`, replace `UintToFloat01` with the RTXPT-fork reference-mode conversion:

```hlsl
float Hash32ToFloat(uint hash)
{
    return (hash >> 8) / float(1 << 24);
}
```

Then update `sampleNext1D`:

```hlsl
float sampleNext1D(inout SampleGenerator sg)
{
    sg.State = Hash32(sg.State);
    return Hash32ToFloat(sg.State);
}
```

Replace the current `Hash32` constants with the RTXPT-fork reference-mode constants from `NoiseAndSequences.hlsli`:

```hlsl
uint Hash32(uint x)
{
    x ^= x >> 16;
    x *= 0x21f0aaadu;
    x ^= x >> 15;
    x *= 0xf35a2d97u;
    x ^= x >> 15;
    return x;
}
```

This phase owns the sampler, so the reference constants are the right choice even though they change noise patterns outside BSDF sampling. Record the intentional divergence from the earlier R1 hash note in `RTXPT_FORK_MAPPING.md`.

- [ ] **Step 2: Add vector sample helpers**

Add these helpers after `sampleNext2D`:

```hlsl
float3 sampleNext3D(inout SampleGenerator sg)
{
    float3 sample;
    sample.x = sampleNext1D(sg);
    sample.y = sampleNext1D(sg);
    sample.z = sampleNext1D(sg);
    return sample;
}

float4 sampleNext4D(inout SampleGenerator sg)
{
    float4 sample;
    sample.x = sampleNext1D(sg);
    sample.y = sampleNext1D(sg);
    sample.z = sampleNext1D(sg);
    sample.w = sampleNext1D(sg);
    return sample;
}
```

- [ ] **Step 3: Create `StatelessSampleGenerators.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli` with:

```hlsl
#ifndef __STATELESS_SAMPLE_GENERATORS_HLSLI__
#define __STATELESS_SAMPLE_GENERATORS_HLSLI__

#include "SampleGenerators.hlsli"

struct SampleGeneratorVertexBase
{
    uint baseHash;
    uint sampleIndex;

    static SampleGeneratorVertexBase make(uint2 pixelCoord, uint vertexIndex, uint sampleIndex)
    {
        SampleGeneratorVertexBase ret;
        ret.sampleIndex = sampleIndex;
        ret.baseHash    = Hash32Combine(Hash32(vertexIndex + 0x035F9F29u),
                                        (pixelCoord.x << 16) | (pixelCoord.y & 0xffffu));
        return ret;
    }
};

// Port these helper functions from RTXPT-fork NoiseAndSequences.hlsli, using local comments:
// bhos_reverse_bits, bhos_owen_hash, bhos_owen_scramble, bhos_sobol.
// Keep SOBOL_MAX_DIMENSIONS at 5, matching RTXPT-fork.

struct UniformSampleSequenceGenerator
{
    static float4 Generate(uint count, SampleGeneratorVertexBase base, uint effectSeed,
                           int subSampleIndex = 0, int subSampleCount = 1)
    {
        count = min(count, 4u);
        float4 retVal = float4(0.0, 0.0, 0.0, 0.0);

        const uint activeIndex = base.sampleIndex * uint(subSampleCount) + uint(subSampleIndex);
        uint currentHash = Hash32Combine(base.baseHash, effectSeed);
        currentHash = Hash32Combine(currentHash, activeIndex);

        [unroll]
        for (uint counter = 0u; counter < count; ++counter)
        {
            currentHash = Hash32(currentHash);
            retVal[counter] = Hash32ToFloat(currentHash);
        }
        return retVal;
    }
};

struct SampleSequenceGenerator
{
    static float4 Generate(uint count, SampleGeneratorVertexBase base, uint effectSeed,
                           int subSampleIndex = 0, int subSampleCount = 1)
    {
        count = min(count, 4u);
        float4 retVal = float4(0.0, 0.0, 0.0, 0.0);

        const uint activeIndex = base.sampleIndex * uint(subSampleCount) + uint(subSampleIndex);
        const uint currentHash = Hash32Combine(base.baseHash, effectSeed);

        [unroll]
        for (uint dimension = 0u; dimension < count; ++dimension)
        {
            const uint shuffleSeed = Hash32Combine(currentHash, 0u);
            const uint dimSeed     = Hash32Combine(currentHash, 1u + dimension);
            const uint shuffledIndex = bhos_owen_scramble(activeIndex, shuffleSeed);
            const uint dimSample =
                (dimension == 0u) ? bhos_reverse_bits(shuffledIndex) : bhos_sobol(shuffledIndex, dimension);
            retVal[dimension] = Hash32ToFloat(bhos_owen_scramble(dimSample, dimSeed));
        }
        return retVal;
    }
};

#endif // __STATELESS_SAMPLE_GENERATORS_HLSLI__
```

Complete the four `bhos_*` helpers from the local RTXPT-fork reference. Keep the same algorithm and constants; write Diligent-owned comments and do not copy the NVIDIA file header.

- [ ] **Step 4: Register the new shader include**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new file next to `SampleGenerators.hlsli`:

```cmake
    assets/shaders/PathTracer/Utils/SampleGenerators.hlsli
    assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli
    assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
```

- [ ] **Step 5: Verify sampler symbols**

Run:

```powershell
rg -n "Hash32ToFloat|sampleNext3D|sampleNext4D" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli
rg -n "SampleGeneratorVertexBase|SampleSequenceGenerator|UniformSampleSequenceGenerator|bhos_owen_scramble|bhos_sobol" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli
rg -n "StatelessSampleGenerators.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: all requested names are present.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add Sobol Owen BSDF sample generator" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Integrate BSDF Samples in Raygen

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Include the stateless sequence generator and define the macro fallback**

Near the production path includes in `PathTracerSample.rgen`, use:

```hlsl
#ifndef RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF
#    define RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF 1
#endif

#include "Utils/StatelessSampleGenerators.hlsli"
#include "PathTracer.hlsli"
```

- [ ] **Step 2: Track diffuse bounces in the path loop**

Before the bounce loop, add:

```hlsl
    uint diffuseBounces = 0u;
```

- [ ] **Step 3: Pre-generate BSDF samples like RTXPT-fork**

Replace the current scatter generator block:

```hlsl
        SampleGenerator sgScatter = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_ScatterBSDF);
        if (!SampleBSDF(bsdfData, wo, sgScatter, nextDir, weight, pdf, lobeP))
            break;
```

with:

```hlsl
        const SampleGeneratorVertexBase sgBase = SampleGeneratorVertexBase::make(pixel, vertexIndex, sampleIndex);
        float3 preGeneratedSamples;

#if RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF
        [branch]
        if (diffuseBounces < g_Const.ptConsts.diffuseBounceCount)
            preGeneratedSamples = SampleSequenceGenerator::Generate(3u, sgBase, kSampleEffect_ScatterBSDF).xyz;
        else
#endif
            preGeneratedSamples = UniformSampleSequenceGenerator::Generate(3u, sgBase, kSampleEffect_ScatterBSDF).xyz;

        uint lobe;
        if (!SampleBSDF(bsdfData, wo, preGeneratedSamples, nextDir, weight, pdf, lobe, lobeP))
            break;
```

- [ ] **Step 4: Enforce `diffuseBounceCount` after sampling**

After `throughput *= weight;`, add:

```hlsl
        if ((lobe & kBSDFLobeDiffuseReflection) != 0u)
        {
            ++diffuseBounces;
            if (diffuseBounces > g_Const.ptConsts.diffuseBounceCount)
                break;
        }
```

Keep the existing total-bounce `for` loop and Russian roulette logic unchanged.

- [ ] **Step 5: Preserve delta-event handling**

Keep this existing line:

```hlsl
        fireflyFilterK = ComputeNewScatterFireflyFilterK(fireflyFilterK, pdf, lobeP);
```

Expected behavior: for delta reflection, `pdf == 0`, so the G1 helper treats the scatter as a zero-spread delta event.

- [ ] **Step 6: Verify raygen uses the new sampler**

Run:

```powershell
rg -n "RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF|SampleGeneratorVertexBase|SampleSequenceGenerator::Generate|UniformSampleSequenceGenerator::Generate|diffuseBounces|kBSDFLobeDiffuseReflection" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
rg -n "SampleBSDF\\([^,]+,[^,]+,\\s*sgScatter|SampleGenerator_makeStateless\\(pixel, vertexIndex, sampleIndex, kSampleEffect_ScatterBSDF\\)" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: first command finds all new sampling path symbols. Second command prints no old scatter-generator call.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "feat(rtxpt): drive BSDF sampling with Sobol Owen sequences" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Mapping, Cleanup, and Verification

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: all R5-touched files

- [ ] **Step 1: Update the mapping document**

In `RTXPT_FORK_MAPPING.md`, update the Utils and Materials sections with these rows or equivalent wording:

```markdown
| `Hash32` / `Hash32Combine` / `Hash32ToFloat` | `Utils/NoiseAndSequences.hlsli` | R5 aligns reference-mode sample conversion; record if any hash constants intentionally remain port-specific |
| `StatelessSampleGenerators.hlsli` | `Utils/StatelessSampleGenerators.hlsli` | BSDF-only Sobol/Owen sample blocks through `SampleSequenceGenerator::Generate` |
| `UniformSampleSequenceGenerator` | `UniformSampleSequenceGenerator` | Uniform fallback for BSDF sample blocks after the diffuse-bounce LD window or when the macro is disabled |
| `kMinGGXAlpha` | `kMinGGXAlpha` | Near-mirror specular collapses to delta reflection |
| `sampleGGX_BVNDF` / `evalPdfGGX_BVNDF` | `Microfacet.hlsli` | Bounded-VNDF GGX reflection sampling and matching pdf |
| `evalDiffuseFrostbiteWeight` | `DiffuseReflectionFrostbite::evalWeight` | Frostbite/Disney energy-conserving diffuse |
| `MultiScatterSpecularApprox` | `MultiScatterSpecularApprox` | Turquin multi-scatter specular compensation |
```

- [ ] **Step 2: Remove R5 placeholder UI text**

Search for the open-work marker text:

```powershell
rg -n "Phase R5|low-discrepancy sampler lands|VNDF/Frostbite/multi-scatter" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no stale user-visible R5 placeholder text remains in `RTXPTSample.cpp`. Mapping/history references may remain if they describe completed R5 behavior.

- [ ] **Step 3: Run whitespace checks**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli Samples/RTXPT/assets/shaders/PathTracer/Utils/SampleGenerators.hlsli Samples/RTXPT/assets/shaders/PathTracer/Utils/StatelessSampleGenerators.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/CMakeLists.txt Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no output and exit code 0.

- [ ] **Step 4: Build verification when explicitly requested**

Do not auto-run this unless the user asks for build/runtime verification. When requested, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the RTXPT sample target builds. The C++ `static_assert`s confirm `PathTracerConstants == 80` and `SampleConstants == 368`.

- [ ] **Step 5: Manual GPU verification when explicitly requested**

Run the sample on D3D12 and Vulkan. Acceptance checks:

- `Max diffuse bounces` is live; changing it resets accumulation.
- `Enable LD sampler for BSDF` is live; changing it recreates the RT PSO and resets accumulation.
- At equal sample counts, primary AA and first-bounce GI noise improve with LD enabled.
- With LD disabled, the image remains unbiased and converges to the same result, with different noise.
- High-roughness/metallic materials no longer darken from missing multi-scatter compensation.
- Very low roughness specular paths behave as delta events without NaN/Inf pixels.
- Direct-light, emissive, and environment MIS remain stable because `EvalBSDF` and `SampleBSDF` agree on pdf.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): record R5 BSDF sampler mappings" -m "Co-Authored-By: GPT 5.5"
```

---

## Final Phase Verification

Run these source-level checks before calling the phase implementation complete:

```powershell
rg -n "diffuseBounceCount|RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF|SampleSequenceGenerator|UniformSampleSequenceGenerator|sampleGGX_BVNDF|evalPdfGGX_BVNDF|MultiScatterSpecularApprox|evalDiffuseFrostbiteWeight|kMinGGXAlpha" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "SampleBSDF\\([^,]+,[^,]+,\\s*inout SampleGenerator|NDF\\) sampling|Lambertian diffuse|low-discrepancy sampler lands in Phase R5" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected:

- First command finds the R5 implementation in constants, sampler, raygen, BSDF, and mapping doc.
- Second command has no stale old-BSDF signature or user-visible R5 placeholder text.

The phase is source-complete only after the direct checks pass. It is runtime-verified only after a requested build/run confirms D3D12 and Vulkan render valid images.

## Self-Review

- G8 coverage: bounded-VNDF sampling/pdf, Frostbite diffuse, multi-scatter specular compensation, lobe selection, mixture pdf, and near-mirror delta events are assigned to Tasks 2 and 4.
- G9 coverage: Sobol/Owen generator, uniform fallback, LD macro, diffuse-bounce window, and UI wiring are assigned to Tasks 1, 3, and 4.
- RTXPT-fork alignment: every algorithm has an anchor listed above; Task 5 updates the mapping doc so future upstream ports know the exact correspondence.
- Scope control: no payload growth, no new RT resources, no new ray type, no backend-specific code, and no realtime-track features.
