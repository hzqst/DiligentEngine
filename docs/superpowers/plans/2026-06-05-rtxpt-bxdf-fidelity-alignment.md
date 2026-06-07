# RTXPT BxDF Fidelity Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align Diligent RTXPT BxDF behavior with RTXPT-fork's component BxDF model while preserving the current Diligent raygen and stable-plane call sites.

**Architecture:** Keep Diligent shader paths, resource bindings, ray payloads, and public wrappers. Refactor `Rendering/Materials/BxDF.hlsli` from compact free functions into RTXPT-fork-shaped components, keep `MakeStandardBSDFData`, `EvalBSDF`, and `SampleBSDF` as compatibility adapters, and route stable-plane delta exploration through the same `FalcorBSDF` component state.

**Tech Stack:** HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, C++17/C++20 DiligentSamples host code for build and UI status, DXC through Diligent ray-tracing PSOs, CMake Visual Studio builds on Windows. `DiligentSamples` is a git submodule; implementation commits in this plan are made inside `DiligentSamples/`.

---

## Context You Need Before Starting

This plan implements `docs/superpowers/specs/2026-06-05-rtxpt-bxdf-fidelity-alignment-spec.md`.

Current Diligent baseline:

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` already has Frostbite diffuse helper math, BVNDF helpers, specular transmission code, lobe bit constants, `EvalBSDF`, and `SampleBSDF`.
- The file is still a compact free-function implementation. It does not define `DiffuseReflectionFrostbite`, `DiffuseTransmissionLambert`, `SpecularReflectionMicrofacet`, `SpecularReflectionTransmissionMicrofacet`, or `FalcorBSDF`.
- `MaterialHeader` currently exposes nested priority, active lobes, and thin-surface bits only.
- `PathTracerTypes.hlsli::ActiveBSDF::evalDeltaLobes` currently initializes zero lobes and returns no delta exploration data.
- `PathTracer.hlsli::MakeBSDFSample` currently maps specular transmission to delta index `1`, which does not match the RTXPT-fork convention.
- `PathTracerClosestHit.rchit` already builds `StablePlaneShadingData`, `StablePlaneMaterialState`, and `ActiveBSDF`, so this plan uses those integration points instead of replacing the path-tracer state machine.

Reference anchors to read before each implementation task:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:157-245` - diffuse reflection and diffuse transmission components.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:273-380` - specular reflection microfacet component.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:385-614` - specular reflection/transmission microfacet component.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:615-702` - `StandardBSDFData` accessors.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:709-1053` - `FalcorBSDF` mixture, sampling, pdf, and delta export.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/StandardBSDF.hlsli:34-240` - world/local wrapper and delta-lobe conversion.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Scene/Material/MaterialData.hlsli:23-85` - full material header bit layout.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/LobeType.hlsli:23-41` - lobe values and aliases.

Predecessor dependency:

- The material JSON external texture-loading spec is `docs/superpowers/specs/2026-06-05-rtxpt-material-json-texture-loading-spec.md`.
- Tasks 1 through 7 can proceed before that fix.
- Final visual acceptance on `convergence-test.scene.json` must wait until external material texture inputs are confirmed to match RTXPT-fork.

Do not copy NVIDIA file headers, large comments, or wholesale source blocks. Port the behavior, names, constants, and control flow into Diligent-owned shader files.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli` - add RTXPT-fork-compatible non-delta reflection/transmission aliases.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli` - add PSD exclude, PSD block-motion-vector, and dominant-delta-lobe bit accessors.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` - introduce component structs, `FalcorBSDF`, local delta-lobe export, accessors, and compatibility wrappers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli` - extend stable-plane material state and delegate `ActiveBSDF::evalDeltaLobes` to BxDF.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` - set active lobe and PSD state during stable-plane surface construction.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` - fix sampled delta-lobe index convention for fill-stable-plane scattering.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - keep reference raygen wrappers working and run source checks against the default active-lobe path.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record the BxDF component and semantic mappings.

## Cross-Cutting Contracts

- **Wrapper compatibility:** Existing calls remain valid:

```hlsl
StandardBSDFData bsdfData = MakeStandardBSDFData(...);
EvalBSDF(bsdfData, wo, wi, specProb, f, pdf);
SampleBSDF(bsdfData, wo, preGeneratedSamples, wi, weight, pdf, lobe, lobeP);
```

- **Direction naming:** Diligent wrappers receive `wo` as the current view/outgoing-from-surface vector and `wi` as the evaluated or sampled scatter direction. Inside `FalcorBSDF`, map Diligent `wo` to reference-style `wiLocal` and Diligent `wi` to reference-style `woLocal`.
- **Active lobes:** Default to `kLobeTypeAll` for current materials. `FalcorBSDF` must gate all probabilities by `MaterialHeader::getActiveLobes()`.
- **PSD state:** `MaterialHeader` and `StablePlaneMaterialState` expose `isPSDExclude`, `isPSDBlockMotionVectorsAtSurface`, and `getPSDDominantDeltaLobeP1`.
- **Delta index order:** Transmission delta lobe index is `0`; reflection delta lobe index is `1`.
- **Delta pdf:** Sampled delta events return `pdf = 0.0` after mixture sampling.
- **Source organization:** Keep the implementation in `BxDF.hlsli` for this plan. Do not introduce a Donut compatibility layer or new root build configuration.
- **Shader includes:** Use Diligent-relative include paths and traditional include guards.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: `D:/RTXPT-fork`
- Verify: `docs/superpowers/specs/2026-06-05-rtxpt-material-json-texture-loading-spec.md`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Read dirty files before editing them.

- [ ] **Step 2: Confirm reference anchors exist**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Materials\BxDF.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Materials\StandardBSDF.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Scene\Material\MaterialData.hlsli
```

Expected: all three commands print `True`.

- [ ] **Step 3: Confirm the external texture-loading predecessor is visible**

Run:

```powershell
Test-Path docs\superpowers\specs\2026-06-05-rtxpt-material-json-texture-loading-spec.md
rg -n "external material texture|Material textures loaded|TextureCount|convergence-test" docs/superpowers DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets
```

Expected: the spec path prints `True`. The `rg` command shows the predecessor spec and current material texture status/reporting points. Record whether a separate implementation plan or code change already exists before using visual comparison as final evidence.

- [ ] **Step 4: Capture current BxDF and stable-plane anchors**

Run:

```powershell
rg -n "struct StandardBSDFData|GetBSDFLobeProbabilities|EvalBSDF|SampleBSDF|evalDeltaLobes|MakeBSDFSample|MaterialHeader|setActiveLobes|getActiveLobes" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `BxDF.hlsli`, `PathTracerTypes.hlsli`, `PathTracer.hlsli`, and `MaterialData.hlsli`.

- [ ] **Step 5: Commit nothing**

Expected: no commit in Task 0. This task only establishes the starting point.

---

### Task 1: Add Lobe Aliases And MaterialHeader PSD Bits

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli`

- [ ] **Step 1: Add missing lobe aliases**

In `LobeType.hlsli`, insert these aliases after `kLobeTypeTransmission` and before `kLobeTypeAll`:

```hlsl
static const uint kLobeTypeNonDeltaReflection   = 0x03u;
static const uint kLobeTypeNonDeltaTransmission = 0x30u;
```

Keep all existing bit values unchanged.

- [ ] **Step 2: Extend `MaterialHeader` bit layout**

In `MaterialData.hlsli`, replace the constant block in `struct MaterialHeader` with:

```hlsl
    static const uint kNestedPriorityBits          = 4u;
    static const uint kLobeTypeBits                = 8u;
    static const uint kPSDDominantDeltaLobeP1Bits  = 4u;

    static const uint kNestedPriorityOffset = 0u;
    static const uint kLobeTypeOffset       = kNestedPriorityOffset + kNestedPriorityBits;
    static const uint kThinSurfaceFlagOffset = kLobeTypeOffset + kLobeTypeBits;
    static const uint kPSDExcludeFlagOffset = kThinSurfaceFlagOffset + 1u;
    static const uint kPSDBlockMotionVectorsAtSurfaceFlagOffset = kPSDExcludeFlagOffset + 1u;
    static const uint kPSDDominantDeltaLobeP1Offset = kPSDBlockMotionVectorsAtSurfaceFlagOffset + 1u;
```

- [ ] **Step 3: Add PSD accessors**

In `MaterialData.hlsli`, add these methods immediately after `isThinSurface()`:

```hlsl
    void setPSDExclude(bool psdExclude)
    {
        packedData = PACK_BITS(1u, kPSDExcludeFlagOffset, packedData, psdExclude ? 1u : 0u);
    }

    bool isPSDExclude()
    {
        return (packedData & (1u << kPSDExcludeFlagOffset)) != 0u;
    }

    void setPSDBlockMotionVectorsAtSurface(bool blockMotionVectorsAtSurface)
    {
        packedData = PACK_BITS(1u, kPSDBlockMotionVectorsAtSurfaceFlagOffset, packedData, blockMotionVectorsAtSurface ? 1u : 0u);
    }

    bool isPSDBlockMotionVectorsAtSurface()
    {
        return (packedData & (1u << kPSDBlockMotionVectorsAtSurfaceFlagOffset)) != 0u;
    }

    void setPSDDominantDeltaLobeP1(uint dominantDeltaLobeP1)
    {
        packedData = PACK_BITS(kPSDDominantDeltaLobeP1Bits,
                               kPSDDominantDeltaLobeP1Offset,
                               packedData,
                               dominantDeltaLobeP1);
    }

    uint getPSDDominantDeltaLobeP1()
    {
        return EXTRACT_BITS(kPSDDominantDeltaLobeP1Bits, kPSDDominantDeltaLobeP1Offset, packedData);
    }
```

- [ ] **Step 4: Verify lobe and header symbols**

Run:

```powershell
rg -n "kLobeTypeNonDeltaReflection|kLobeTypeNonDeltaTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli
rg -n "kPSDExcludeFlagOffset|setPSDExclude|isPSDExclude|setPSDBlockMotionVectorsAtSurface|isPSDBlockMotionVectorsAtSurface|setPSDDominantDeltaLobeP1|getPSDDominantDeltaLobeP1" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli
```

Expected: all requested aliases and accessors are present.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add BxDF lobe and PSD header bits" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Add StandardBSDFData Accessors And Header Facade

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`

- [ ] **Step 1: Include `MaterialData.hlsli`**

At the top of `BxDF.hlsli`, keep `SampleGenerators.hlsli` and `LobeType.hlsli`, and add the material header include between them:

```hlsl
#include "../../Utils/SampleGenerators.hlsli"
#include "../../Scene/Material/MaterialData.hlsli"
#include "LobeType.hlsli"
```

- [ ] **Step 2: Extend `StandardBSDFData` with active-lobe and PSD state**

Replace the current `StandardBSDFData` struct with this field layout and accessor surface:

```hlsl
struct StandardBSDFData
{
    float3 N;
    float3 diffuse;
    float3 specular;
    float3 transmission;
    float  roughness;
    float  alpha;
    float  eta;
    float  metallic;
    float  diffuseTransmission;
    float  specularTransmission;
    bool   thinSurface;
    uint   activeLobes;
    uint   psdExclude;
    uint   psdBlockMotionVectorsAtSurface;
    uint   psdDominantDeltaLobeP1;

    float3 Diffuse() { return diffuse; }
    float3 Specular() { return specular; }
    float3 Transmission() { return transmission; }
    float  Roughness() { return roughness; }
    float  Metallic() { return metallic; }
    float  Eta() { return eta; }
    float  DiffuseTransmission() { return diffuseTransmission; }
    float  SpecularTransmission() { return specularTransmission; }

    void SetEta(float value)
    {
        eta = value;
    }

    void SetRoughness(float value)
    {
        roughness = saturate(value);
        const float newAlpha = roughness * roughness;
        alpha = (newAlpha < kMinGGXAlpha) ? 0.0 : max(newAlpha, kMinGGXAlpha);
    }
};
```

- [ ] **Step 3: Add a helper that reconstructs `MaterialHeader` from `StandardBSDFData`**

Add this helper after `StandardBSDFData`:

```hlsl
MaterialHeader MakeMaterialHeader(StandardBSDFData bsdfData)
{
    MaterialHeader header = MaterialHeader::make();
    header.setActiveLobes(bsdfData.activeLobes == 0u ? kLobeTypeAll : bsdfData.activeLobes);
    header.setThinSurface(bsdfData.thinSurface);
    header.setPSDExclude(bsdfData.psdExclude != 0u);
    header.setPSDBlockMotionVectorsAtSurface(bsdfData.psdBlockMotionVectorsAtSurface != 0u);
    header.setPSDDominantDeltaLobeP1(bsdfData.psdDominantDeltaLobeP1);
    return header;
}
```

- [ ] **Step 4: Replace `MakeStandardBSDFData` with full and compatibility overloads**

Keep the existing public 10-argument and 4-argument call shapes, but route both through a full overload that accepts active-lobe and PSD state:

```hlsl
StandardBSDFData MakeStandardBSDFData(float3 N,
                                      float3 baseColor,
                                      float  metallic,
                                      float  roughness,
                                      float  materialIoR,
                                      float  outsideIoR,
                                      float  transmissionFactor,
                                      float  diffuseTransmissionFactor,
                                      bool   thinSurface,
                                      bool   frontFacing,
                                      uint   activeLobes,
                                      bool   psdExclude,
                                      bool   psdBlockMotionVectorsAtSurface,
                                      uint   psdDominantDeltaLobeP1)
{
    const float r               = saturate(roughness);
    const float m               = saturate(metallic);
    const float safeMaterialIoR = max(materialIoR, 1.0);
    const float safeOutsideIoR  = max(outsideIoR, 1.0);
    const float f0Sqrt          = (safeMaterialIoR - 1.0) / max(safeMaterialIoR + 1.0, 1e-4);
    const float dielectric      = f0Sqrt * f0Sqrt;
    const float alpha           = r * r;

    StandardBSDFData bsdfData;
    bsdfData.N                              = N;
    bsdfData.diffuse                        = baseColor * (1.0 - m);
    bsdfData.specular                       = lerp(float3(dielectric, dielectric, dielectric), baseColor, m);
    bsdfData.transmission                   = baseColor;
    bsdfData.roughness                      = r;
    bsdfData.alpha                          = (alpha < kMinGGXAlpha) ? 0.0 : max(alpha, kMinGGXAlpha);
    bsdfData.eta                            = frontFacing ? safeOutsideIoR / safeMaterialIoR : safeMaterialIoR / safeOutsideIoR;
    bsdfData.metallic                       = m;
    bsdfData.diffuseTransmission            = saturate(diffuseTransmissionFactor) * (1.0 - m);
    bsdfData.specularTransmission           = saturate(transmissionFactor) * (1.0 - m);
    bsdfData.thinSurface                    = thinSurface;
    bsdfData.activeLobes                    = activeLobes == 0u ? kLobeTypeAll : activeLobes;
    bsdfData.psdExclude                     = psdExclude ? 1u : 0u;
    bsdfData.psdBlockMotionVectorsAtSurface = psdBlockMotionVectorsAtSurface ? 1u : 0u;
    bsdfData.psdDominantDeltaLobeP1         = psdDominantDeltaLobeP1;
    return bsdfData;
}
```

Then keep the current 10-argument overload by adding:

```hlsl
StandardBSDFData MakeStandardBSDFData(float3 N,
                                      float3 baseColor,
                                      float  metallic,
                                      float  roughness,
                                      float  materialIoR,
                                      float  outsideIoR,
                                      float  transmissionFactor,
                                      float  diffuseTransmissionFactor,
                                      bool   thinSurface,
                                      bool   frontFacing)
{
    return MakeStandardBSDFData(N,
                                baseColor,
                                metallic,
                                roughness,
                                materialIoR,
                                outsideIoR,
                                transmissionFactor,
                                diffuseTransmissionFactor,
                                thinSurface,
                                frontFacing,
                                kLobeTypeAll,
                                false,
                                false,
                                0u);
}
```

Keep the 4-argument overload and route it through the 10-argument overload.

- [ ] **Step 5: Verify the new data contract**

Run:

```powershell
rg -n "activeLobes|psdExclude|psdBlockMotionVectorsAtSurface|psdDominantDeltaLobeP1|MakeMaterialHeader|SetEta|SetRoughness" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "MakeStandardBSDFData\\(" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: `BxDF.hlsli` exposes active/PSD state and all existing call sites still resolve to one of the overloads.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add StandardBSDF header state" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Refactor Diffuse Components

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`

- [ ] **Step 1: Add `DiffuseReflectionFrostbite`**

Replace the free `evalDiffuseFrostbiteWeight` usage with a component that keeps the existing Diligent helper math but exposes the reference-shaped methods:

```hlsl
struct DiffuseReflectionFrostbite
{
    float3 albedo;
    float  roughness;

    float3 eval(const float3 wi, const float3 wo)
    {
        if (min(wi.z, wo.z) < kMinCosTheta)
            return float3(0.0, 0.0, 0.0);

        return evalWeight(wi, wo) * K_1_PI * wo.z;
    }

    bool sample(const float3 wi, out float3 wo, out float pdf, out float3 weight, out uint lobe, out float lobeP, float3 preGeneratedSample)
    {
        wo   = sampleCosineHemisphereLocal(preGeneratedSample.xy, pdf);
        lobe = kLobeTypeDiffuseReflection;

        if (min(wi.z, wo.z) < kMinCosTheta)
        {
            weight = float3(0.0, 0.0, 0.0);
            lobeP  = 0.0;
            return false;
        }

        weight = evalWeight(wi, wo);
        lobeP  = 1.0;
        return true;
    }

    float evalPdf(const float3 wi, const float3 wo)
    {
        if (min(wi.z, wo.z) < kMinCosTheta)
            return 0.0;

        return K_1_PI * wo.z;
    }

    float3 evalWeight(float3 wi, float3 wo)
    {
        const float3 h            = normalize(wi + wo);
        const float  woDotH       = dot(wo, h);
        const float  energyBias   = lerp(0.0, 0.5, roughness);
        const float  energyFactor = lerp(1.0, 1.0 / 1.51, roughness);
        const float  fd90         = energyBias + 2.0 * woDotH * woDotH * roughness;
        const float  wiScatter    = evalFresnelSchlick(1.0, fd90, wi.z);
        const float  woScatter    = evalFresnelSchlick(1.0, fd90, wo.z);
        return albedo * wiScatter * woScatter * energyFactor;
    }
};
```

- [ ] **Step 2: Add `DiffuseTransmissionLambert`**

Add the transmission component immediately after `DiffuseReflectionFrostbite`:

```hlsl
struct DiffuseTransmissionLambert
{
    float3 albedo;

    float3 eval(const float3 wi, const float3 wo)
    {
        if (min(wi.z, -wo.z) < kMinCosTheta)
            return float3(0.0, 0.0, 0.0);

        return K_1_PI * albedo * -wo.z;
    }

    bool sample(const float3 wi, out float3 wo, out float pdf, out float3 weight, out uint lobe, out float lobeP, float3 preGeneratedSample)
    {
        wo   = sampleCosineHemisphereLocal(preGeneratedSample.xy, pdf);
        wo.z = -wo.z;
        lobe = kLobeTypeDiffuseTransmission;

        if (min(wi.z, -wo.z) < kMinCosTheta)
        {
            weight = float3(0.0, 0.0, 0.0);
            lobeP  = 0.0;
            return false;
        }

        weight = albedo;
        lobeP  = 1.0;
        return true;
    }

    float evalPdf(const float3 wi, const float3 wo)
    {
        if (min(wi.z, -wo.z) < kMinCosTheta)
            return 0.0;

        return K_1_PI * -wo.z;
    }
};
```

- [ ] **Step 3: Remove duplicate free diffuse evaluators**

Delete `EvalDiffuseTransmission` and `SampleDiffuseTransmission` after the component methods compile conceptually into the same behavior. Keep `GetTransmissionAlbedo`, `sampleCosineHemisphereLocal`, and the Fresnel helpers because the components and specular paths use them.

- [ ] **Step 4: Verify diffuse component symbols**

Run:

```powershell
rg -n "struct DiffuseReflectionFrostbite|struct DiffuseTransmissionLambert|evalWeight|kLobeTypeDiffuseReflection|kLobeTypeDiffuseTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "EvalDiffuseTransmission|SampleDiffuseTransmission|evalDiffuseFrostbiteWeight" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
```

Expected: the first command finds the component methods. The second command prints no removed free diffuse helpers.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): componentize diffuse BxDF lobes" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Refactor Specular Components

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`

- [ ] **Step 1: Add `SpecularReflectionMicrofacet` shell**

Add this component before `FalcorBSDF` is introduced:

```hlsl
struct SpecularReflectionMicrofacet
{
    float3 albedo;
    float  alpha;
    uint   activeLobes;

    bool hasLobe(uint lobe)
    {
        return (activeLobes & lobe) != 0u;
    }
};
```

- [ ] **Step 2: Move rough/delta reflection behavior into the component**

Add `eval`, `sample`, and `evalPdf` methods to `SpecularReflectionMicrofacet` with these concrete mappings:

- `eval` uses the current rough reflection math from `EvalBSDF`: GGX NDF, correlated visibility, Schlick Fresnel, and `MultiScatterSpecularApprox`.
- `sample` uses the current delta branch for `alpha == 0.0` and the current `sampleGGX_BVNDF` rough branch.
- `evalPdf` uses `evalPdfGGX_BVNDF(alpha, wi, h)` and returns `0.0` for delta or inactive lobes.
- `hasLobe(kLobeTypeSpecularReflection)` gates rough reflection.
- `hasLobe(kLobeTypeDeltaReflection)` gates delta reflection.

The method signatures must be:

```hlsl
float3 eval(const float3 wi, const float3 wo)
bool sample(const float3 wi, out float3 wo, out float pdf, out float3 weight, out uint lobe, out float lobeP, float3 preGeneratedSample)
float evalPdf(const float3 wi, const float3 wo)
```

- [ ] **Step 3: Add `SpecularReflectionTransmissionMicrofacet` shell**

Add this component after `SpecularReflectionMicrofacet`:

```hlsl
struct SpecularReflectionTransmissionMicrofacet
{
    float3 transmissionAlbedo;
    float  alpha;
    float  eta;
    uint   activeLobes;
    bool   isThinSurface;

    bool hasLobe(uint lobe)
    {
        return (activeLobes & lobe) != 0u;
    }
};
```

- [ ] **Step 4: Move rough/delta reflection-transmission behavior into the component**

Move the current `EvalSpecularReflectionTransmission` and `EvalPdfSpecularReflectionTransmission` bodies into `SpecularReflectionTransmissionMicrofacet::eval` and `SpecularReflectionTransmissionMicrofacet::evalPdf`, replacing `bsdfData` field reads with component fields:

- `bsdfData.alpha` becomes `alpha`.
- `bsdfData.eta` becomes `eta`.
- `bsdfData.thinSurface` becomes `isThinSurface`.
- `GetTransmissionAlbedo(bsdfData)` becomes `transmissionAlbedo`.
- Reflection requires `hasLobe(kLobeTypeSpecularReflection)` for rough and `hasLobe(kLobeTypeDeltaReflection)` for delta.
- Transmission requires `hasLobe(kLobeTypeSpecularTransmission)` for rough and `hasLobe(kLobeTypeDeltaTransmission)` for delta.

Add a `sample` method with this signature:

```hlsl
bool sample(const float3 wi, out float3 wo, out float pdf, out float3 weight, out uint lobe, out float lobeP, float3 preGeneratedSample)
```

The sample method keeps these observable semantics:

- Use `preGeneratedSample.z` to choose reflection versus transmission using dielectric Fresnel.
- Use thin-surface eta override for transmission only.
- For delta reflection, return `lobe = kLobeTypeDeltaReflection` and `pdf = 0.0`.
- For delta transmission, return `lobe = kLobeTypeDeltaTransmission` and `pdf = 0.0`.
- For rough events, set `weight = eval(wi, wo) / pdf`.
- Set `lobeP` to the local Fresnel branch probability before `FalcorBSDF` multiplies by the component mixture probability.

- [ ] **Step 5: Remove old free specular-transmission helpers**

After the component methods are in place, delete the free functions:

```hlsl
float3 EvalSpecularReflectionTransmission(StandardBSDFData bsdfData, float3 wiLocal, float3 woLocal)
float EvalPdfSpecularReflectionTransmission(StandardBSDFData bsdfData, float3 wiLocal, float3 woLocal)
```

- [ ] **Step 6: Verify specular component symbols**

Run:

```powershell
rg -n "struct SpecularReflectionMicrofacet|struct SpecularReflectionTransmissionMicrofacet|hasLobe|kLobeTypeDeltaReflection|kLobeTypeDeltaTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "EvalSpecularReflectionTransmission|EvalPdfSpecularReflectionTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
```

Expected: component symbols are present and old free specular-transmission helper names are gone.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): componentize specular BxDF lobes" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Implement FalcorBSDF Mixture And Wrappers

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Add `FalcorBSDF` fields and construction**

Add `FalcorBSDF` after the four components:

```hlsl
struct FalcorBSDF
{
    DiffuseReflectionFrostbite diffuseReflection;
    DiffuseTransmissionLambert diffuseTransmission;
    SpecularReflectionMicrofacet specularReflection;
    SpecularReflectionTransmissionMicrofacet specularReflectionTransmission;

    float diffTrans;
    float specTrans;
    float pDiffuseReflection;
    float pDiffuseTransmission;
    float pSpecularReflection;
    float pSpecularReflectionTransmission;
    bool  psdExclude;
};
```

Add `__init` and `make` methods that set component fields exactly from `StandardBSDFData` and `MaterialHeader`:

```hlsl
    void __init(const MaterialHeader mtl, float3 N, float3 V, const StandardBSDFData data)
    {
        const bool isThinSurface = mtl.isThinSurface();
        const float3 dataTransmission = data.Transmission();
        const float3 transmissionAlbedo = isThinSurface ? dataTransmission : sqrt(max(dataTransmission, float3(0.0, 0.0, 0.0)));
        const float dataRoughness = data.Roughness();

        diffuseReflection.albedo   = data.Diffuse();
        diffuseReflection.roughness = dataRoughness;
        diffuseTransmission.albedo = transmissionAlbedo;

        float alpha = dataRoughness * dataRoughness;
        if (alpha < kMinGGXAlpha)
            alpha = 0.0;
        else
            alpha = max(alpha, kMinGGXAlpha);

        const uint activeLobes = mtl.getActiveLobes();
        psdExclude = mtl.isPSDExclude();

        specularReflection.albedo      = data.Specular();
        specularReflection.alpha       = alpha;
        specularReflection.activeLobes = activeLobes;

        specularReflectionTransmission.transmissionAlbedo = transmissionAlbedo;
        specularReflectionTransmission.alpha = data.Eta() == 1.0 ? 0.0 : alpha;
        specularReflectionTransmission.eta = data.Eta();
        specularReflectionTransmission.activeLobes = activeLobes;
        specularReflectionTransmission.isThinSurface = isThinSurface;

        diffTrans = data.DiffuseTransmission();
        specTrans = data.SpecularTransmission();

        const float metallicBRDF  = data.Metallic() * (1.0 - specTrans);
        const float dielectricBSDF = (1.0 - data.Metallic()) * (1.0 - specTrans);
        const float specularBSDF  = specTrans;
        const float diffuseWeight = luminance(data.Diffuse());
        const float specularWeight = luminance(evalFresnelSchlick(data.Specular(), float3(1.0, 1.0, 1.0), saturate(dot(V, N))));

        pDiffuseReflection = (activeLobes & kLobeTypeDiffuseReflection) != 0u ? diffuseWeight * dielectricBSDF * (1.0 - diffTrans) : 0.0;
        pDiffuseTransmission = (activeLobes & kLobeTypeDiffuseTransmission) != 0u ? diffuseWeight * dielectricBSDF * diffTrans : 0.0;
        pSpecularReflection = (activeLobes & (kLobeTypeSpecularReflection | kLobeTypeDeltaReflection)) != 0u ? specularWeight * (metallicBRDF + dielectricBSDF) : 0.0;
        pSpecularReflectionTransmission =
            (activeLobes & (kLobeTypeSpecularReflection | kLobeTypeDeltaReflection | kLobeTypeSpecularTransmission | kLobeTypeDeltaTransmission)) != 0u ? specularBSDF : 0.0;

        float normFactor = pDiffuseReflection + pDiffuseTransmission + pSpecularReflection + pSpecularReflectionTransmission;
        if (normFactor > 0.0)
        {
            normFactor = 1.0 / normFactor;
            pDiffuseReflection *= normFactor;
            pDiffuseTransmission *= normFactor;
            pSpecularReflection *= normFactor;
            pSpecularReflectionTransmission *= normFactor;
        }
    }
```

Add:

```hlsl
    static FalcorBSDF make(const MaterialHeader mtl, float3 N, float3 V, const StandardBSDFData data)
    {
        FalcorBSDF ret;
        ret.__init(mtl, N, V, data);
        return ret;
    }
```

- [ ] **Step 2: Add `FalcorBSDF::getLobes`**

Add:

```hlsl
    static uint getLobes(const StandardBSDFData data)
    {
        const float alpha = data.Roughness() * data.Roughness();
        const bool  isDelta = alpha < kMinGGXAlpha;
        const float diffTransValue = data.DiffuseTransmission();
        const float specTransValue = data.SpecularTransmission();

        uint lobes = isDelta ? kLobeTypeDeltaReflection : kLobeTypeSpecularReflection;
        if (any(data.Diffuse() > 0.0) && specTransValue < 1.0)
        {
            if (diffTransValue < 1.0)
                lobes |= kLobeTypeDiffuseReflection;
            if (diffTransValue > 0.0)
                lobes |= kLobeTypeDiffuseTransmission;
        }
        if (specTransValue > 0.0)
            lobes |= isDelta ? kLobeTypeDeltaTransmission : kLobeTypeSpecularTransmission;

        return lobes;
    }
```

- [ ] **Step 3: Add `FalcorBSDF::eval` and `evalPdf`**

Add `eval` returning `float4`, with `.w = Average(specular)`:

```hlsl
    float4 eval(const float3 wi, const float3 wo)
    {
        float3 diffuse = float3(0.0, 0.0, 0.0);
        float3 specular = float3(0.0, 0.0, 0.0);

        if (pDiffuseReflection > 0.0)
            diffuse += (1.0 - specTrans) * (1.0 - diffTrans) * diffuseReflection.eval(wi, wo);
        if (pDiffuseTransmission > 0.0)
            diffuse += (1.0 - specTrans) * diffTrans * diffuseTransmission.eval(wi, wo);
        if (pSpecularReflection > 0.0)
            specular += (1.0 - specTrans) * specularReflection.eval(wi, wo);
        if (pSpecularReflectionTransmission > 0.0)
            specular += specTrans * specularReflectionTransmission.eval(wi, wo);

        return float4(diffuse + specular, Average(specular));
    }

    float evalPdf(const float3 wi, const float3 wo)
    {
        float pdf = 0.0;
        if (pDiffuseReflection > 0.0)
            pdf += pDiffuseReflection * diffuseReflection.evalPdf(wi, wo);
        if (pDiffuseTransmission > 0.0)
            pdf += pDiffuseTransmission * diffuseTransmission.evalPdf(wi, wo);
        if (pSpecularReflection > 0.0)
            pdf += pSpecularReflection * specularReflection.evalPdf(wi, wo);
        if (pSpecularReflectionTransmission > 0.0)
            pdf += pSpecularReflectionTransmission * specularReflectionTransmission.evalPdf(wi, wo);
        return pdf;
    }
```

- [ ] **Step 4: Add `FalcorBSDF::sample` with reference mixture pdf semantics**

Replace `GetBSDFLobeProbabilities` and the current monolithic `SampleBSDF` selection logic with `FalcorBSDF::sample`. The branch order and pdf additions must be:

```hlsl
diffuseReflection branch:
    weight /= pDiffuseReflection;
    weight *= (1.0 - specTrans) * (1.0 - diffTrans);
    pdf *= pDiffuseReflection;
    lobeP *= pDiffuseReflection;
    if (pSpecularReflection > 0.0) pdf += pSpecularReflection * specularReflection.evalPdf(wi, wo);
    if (pSpecularReflectionTransmission > 0.0) pdf += pSpecularReflectionTransmission * specularReflectionTransmission.evalPdf(wi, wo);

diffuseTransmission branch:
    weight /= pDiffuseTransmission;
    weight *= (1.0 - specTrans) * diffTrans;
    pdf *= pDiffuseTransmission;
    lobeP *= pDiffuseTransmission;
    if (pSpecularReflectionTransmission > 0.0) pdf += pSpecularReflectionTransmission * specularReflectionTransmission.evalPdf(wi, wo);

specularReflection branch:
    weight /= pSpecularReflection;
    weight *= (1.0 - specTrans);
    pdf *= pSpecularReflection;
    lobeP *= pSpecularReflection;
    if (pDiffuseReflection > 0.0) pdf += pDiffuseReflection * diffuseReflection.evalPdf(wi, wo);
    if (pSpecularReflectionTransmission > 0.0) pdf += pSpecularReflectionTransmission * specularReflectionTransmission.evalPdf(wi, wo);

specularReflectionTransmission branch:
    weight /= pSpecularReflectionTransmission;
    weight *= specTrans;
    pdf *= pSpecularReflectionTransmission;
    lobeP *= pSpecularReflectionTransmission;
    if (pDiffuseReflection > 0.0) pdf += pDiffuseReflection * diffuseReflection.evalPdf(wi, wo);
    if (pDiffuseTransmission > 0.0) pdf += pDiffuseTransmission * diffuseTransmission.evalPdf(wi, wo);
    if (pSpecularReflection > 0.0) pdf += pSpecularReflection * specularReflection.evalPdf(wi, wo);
```

After branch evaluation, add:

```hlsl
        if (!valid || ((lobe & kLobeTypeDelta) != 0u))
            pdf = 0.0;
```

- [ ] **Step 5: Replace wrapper internals**

Keep the public wrapper signatures and replace their bodies with local frame conversion plus `FalcorBSDF`.

`EvalBSDF`:

```hlsl
void EvalBSDF(StandardBSDFData bsdfData, float3 wo, float3 wi, float specProb, out float3 f, out float pdf)
{
    f   = float3(0.0, 0.0, 0.0);
    pdf = 0.0;

    float3 tangent;
    float3 bitangent;
    BranchlessONB(bsdfData.N, tangent, bitangent);

    const float3 viewLocal = float3(dot(wo, tangent), dot(wo, bitangent), dot(bsdfData.N, wo));
    const float3 scatterLocal = float3(dot(wi, tangent), dot(wi, bitangent), dot(bsdfData.N, wi));
    if (viewLocal.z < kMinCosTheta || abs(scatterLocal.z) < kMinCosTheta)
        return;

    const MaterialHeader mtl = MakeMaterialHeader(bsdfData);
    FalcorBSDF bsdf = FalcorBSDF::make(mtl, bsdfData.N, wo, bsdfData);
    const float4 value = bsdf.eval(viewLocal, scatterLocal);
    f = value.rgb;
    pdf = bsdf.evalPdf(viewLocal, scatterLocal);
}
```

`specProb` remains a compatibility parameter and does not alter reference-equivalent behavior.

`SampleBSDF`:

```hlsl
bool SampleBSDF(StandardBSDFData bsdfData, float3 wo, float3 preGeneratedSample,
                out float3 wi, out float3 weight, out float pdf, out uint lobe, out float lobeP)
{
    wi     = float3(0.0, 0.0, 0.0);
    weight = float3(0.0, 0.0, 0.0);
    pdf    = 0.0;
    lobe   = 0u;
    lobeP  = 0.0;

    float3 tangent;
    float3 bitangent;
    BranchlessONB(bsdfData.N, tangent, bitangent);

    const float3 viewLocal = float3(dot(wo, tangent), dot(wo, bitangent), dot(bsdfData.N, wo));
    if (viewLocal.z < kMinCosTheta)
        return false;

    float3 scatterLocal;
    const MaterialHeader mtl = MakeMaterialHeader(bsdfData);
    FalcorBSDF bsdf = FalcorBSDF::make(mtl, bsdfData.N, wo, bsdfData);
    const bool valid = bsdf.sample(viewLocal, scatterLocal, pdf, weight, lobe, lobeP, preGeneratedSample);
    if (!valid)
        return false;

    wi = normalize(tangent * scatterLocal.x + bitangent * scatterLocal.y + bsdfData.N * scatterLocal.z);
    return IsFiniteVector(wi) && IsFiniteVector(weight) && IsFiniteScalar(pdf);
}
```

- [ ] **Step 6: Verify wrappers and removed compact probability helper**

Run:

```powershell
rg -n "struct FalcorBSDF|pDiffuseReflection|pDiffuseTransmission|pSpecularReflectionTransmission|getLobes\\(|float4 eval\\(|float evalPdf\\(|bool sample\\(" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "GetBSDFLobeProbabilities|BSDFLobeProbabilities" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "EvalBSDF\\(|SampleBSDF\\(" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: `FalcorBSDF` methods and wrappers are present. The old compact probability helper names are gone. Existing call sites still call `EvalBSDF` and `SampleBSDF`.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): route BxDF wrappers through FalcorBSDF" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Restore Delta-Lobe Export And Stable-Plane Indexing

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`

- [ ] **Step 1: Add BxDF-local delta lobe type**

In `BxDF.hlsli`, add this type before `FalcorBSDF`:

```hlsl
static const uint cBxDFMaxDeltaLobes = 2u;

struct BxDFDeltaLobe
{
    float3 dir;
    float3 thp;
    uint   transmission;
    float  probability;

    static BxDFDeltaLobe make()
    {
        BxDFDeltaLobe ret;
        ret.dir          = float3(0.0, 0.0, 1.0);
        ret.thp          = float3(0.0, 0.0, 0.0);
        ret.transmission = 0u;
        ret.probability  = 0.0;
        return ret;
    }
};
```

- [ ] **Step 2: Add `FalcorBSDF::evalDeltaLobes`**

Add the method to `FalcorBSDF`:

```hlsl
    void evalDeltaLobes(const float3 wi, out BxDFDeltaLobe deltaLobes[cBxDFMaxDeltaLobes], out uint deltaLobeCount, out float nonDeltaPart)
    {
        deltaLobeCount = cBxDFMaxDeltaLobes;
        for (uint i = 0u; i < cBxDFMaxDeltaLobes; ++i)
            deltaLobes[i] = BxDFDeltaLobe::make();

        nonDeltaPart = pDiffuseReflection + pDiffuseTransmission;
        if (specularReflection.alpha > 0.0)
            nonDeltaPart += pSpecularReflection;
        if (specularReflectionTransmission.alpha > 0.0)
            nonDeltaPart += pSpecularReflectionTransmission;

        if ((pSpecularReflection + pSpecularReflectionTransmission) == 0.0 || psdExclude)
            return;

        BxDFDeltaLobe deltaTransmission = BxDFDeltaLobe::make();
        BxDFDeltaLobe deltaReflection = BxDFDeltaLobe::make();
        deltaTransmission.transmission = 1u;
        deltaReflection.transmission = 0u;
        deltaReflection.dir = float3(-wi.x, -wi.y, wi.z);

        if (specularReflection.alpha == 0.0 && specularReflection.hasLobe(kLobeTypeDeltaReflection))
        {
            deltaReflection.probability = pSpecularReflection;
            deltaReflection.thp = (1.0 - pSpecularReflectionTransmission) *
                evalFresnelSchlick(specularReflection.albedo, float3(1.0, 1.0, 1.0), wi.z);
        }

        if (specularReflectionTransmission.alpha == 0.0)
        {
            const bool hasReflection = specularReflectionTransmission.hasLobe(kLobeTypeDeltaReflection);
            const bool hasTransmission = specularReflectionTransmission.hasLobe(kLobeTypeDeltaTransmission);
            if (hasReflection || hasTransmission)
            {
                float cosThetaT;
                float F = evalFresnelDielectric(specularReflectionTransmission.eta, wi.z, cosThetaT);

                if (hasReflection)
                {
                    const float localProbability = pSpecularReflectionTransmission * F;
                    deltaReflection.thp += float3(localProbability, localProbability, localProbability);
                    deltaReflection.probability += localProbability;
                }

                if (hasTransmission)
                {
                    float actualEta = specularReflectionTransmission.eta;
                    if (specularReflectionTransmission.isThinSurface)
                    {
                        actualEta = 1.0;
                        F = evalFresnelDielectric(actualEta, wi.z, cosThetaT);
                    }

                    const float localProbability = pSpecularReflectionTransmission * (1.0 - F);
                    deltaTransmission.dir = float3(-wi.x * actualEta, -wi.y * actualEta, -cosThetaT);
                    deltaTransmission.thp = specularReflectionTransmission.transmissionAlbedo * localProbability;
                    deltaTransmission.probability = localProbability;
                }
            }
        }

        deltaLobes[0] = deltaTransmission;
        deltaLobes[1] = deltaReflection;
    }
```

- [ ] **Step 3: Extend stable-plane material state**

In `PathTracerTypes.hlsli`, replace `StablePlaneMaterialState` with:

```hlsl
    struct StablePlaneMaterialState
    {
        uint flags;
        uint nestedPriority;
        uint activeLobes;
        uint psdExclude;
        uint psdBlockMotionVectorsAtSurface;
        uint psdDominantDeltaLobeP1;

        bool isPSDExclude() { return psdExclude != 0u; }
        bool isPSDBlockMotionVectorsAtSurface() { return psdBlockMotionVectorsAtSurface != 0u; }
        uint getPSDDominantDeltaLobeP1() { return psdDominantDeltaLobeP1; }
        bool isThinSurface() { return (flags & kMaterialFlagThinSurface) != 0u; }
        uint getNestedPriority() { return nestedPriority; }
        uint getActiveLobes() { return activeLobes == 0u ? kLobeTypeAll : activeLobes; }
    };
```

- [ ] **Step 4: Delegate `ActiveBSDF::evalDeltaLobes`**

Replace the current zero-lobe stub in `PathTracerTypes.hlsli` with:

```hlsl
        void evalDeltaLobes(StablePlaneShadingData shadingData,
                            out DeltaLobe deltaLobes[cMaxDeltaLobes],
                            out uint deltaLobeCount,
                            out float nonDeltaPart)
        {
            for (uint i = 0u; i < cMaxDeltaLobes; ++i)
                deltaLobes[i] = DeltaLobe::make();

            const MaterialHeader mtl = MakeMaterialHeader(standardData);
            FalcorBSDF bsdf = FalcorBSDF::make(mtl, shadingData.N, shadingData.V, standardData);

            BxDFDeltaLobe localLobes[cBxDFMaxDeltaLobes];
            uint localCount;
            const float3 viewLocal = float3(dot(shadingData.V, shadingData.T),
                                            dot(shadingData.V, shadingData.B),
                                            dot(shadingData.V, shadingData.N));
            bsdf.evalDeltaLobes(viewLocal, localLobes, localCount, nonDeltaPart);

            deltaLobeCount = min(localCount, cMaxDeltaLobes);
            for (uint i = 0u; i < deltaLobeCount; ++i)
            {
                deltaLobes[i].dir = normalize(shadingData.T * localLobes[i].dir.x +
                                               shadingData.B * localLobes[i].dir.y +
                                               shadingData.N * localLobes[i].dir.z);
                deltaLobes[i].thp = localLobes[i].thp;
                deltaLobes[i].transmission = localLobes[i].transmission;
                deltaLobes[i].probability = localLobes[i].probability;
            }
        }
```

- [ ] **Step 5: Fix sampled delta-lobe index convention**

In `PathTracer.hlsli::MakeBSDFSample`, replace the `deltaLobeIndex` assignment with:

```hlsl
        bs.deltaLobeIndex = (lobe & kBSDFLobeDeltaTransmission) != 0u ? 0u :
            (((lobe & kBSDFLobeDeltaReflection) != 0u) ? 1u : 0u);
```

- [ ] **Step 6: Verify delta export and index symbols**

Run:

```powershell
rg -n "cBxDFMaxDeltaLobes|struct BxDFDeltaLobe|evalDeltaLobes|deltaLobes\\[0\\]|deltaLobes\\[1\\]" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli
rg -n "deltaLobeIndex = .*kBSDFLobeDeltaTransmission|kBSDFLobeDeltaReflection" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
```

Expected: BxDF exports transmission at slot `0` and reflection at slot `1`; `MakeBSDFSample` uses the same convention.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): restore BxDF delta lobe export" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Wire Active Lobes And PSD State At Surface Construction

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Initialize stable-plane material state with active lobes and PSD defaults**

In `PathTracerClosestHit.rchit`, replace the current `StablePlaneMaterialState mtl` initialization block with:

```hlsl
        StablePlaneMaterialState mtl;
        mtl.flags                            = materialFlags;
        mtl.nestedPriority                   = nestedPriority;
        mtl.activeLobes                      = kLobeTypeAll;
        mtl.psdExclude                       = 0u;
        mtl.psdBlockMotionVectorsAtSurface   = 0u;
        mtl.psdDominantDeltaLobeP1           = 0u;
```

- [ ] **Step 2: Pass active-lobe and PSD state into `MakeStandardBSDFData`**

In `PathTracerClosestHit.rchit`, replace the `bsdf.standardData = MakeStandardBSDFData(...)` call with the full overload:

```hlsl
        bsdf.standardData = MakeStandardBSDFData(worldNormal,
                                                 baseColor,
                                                 metallic,
                                                 roughness,
                                                 ior,
                                                 1.0,
                                                 transmissionFactor,
                                                 diffuseTransmissionFactor,
                                                 thinSurface,
                                                 frontFacing,
                                                 mtl.getActiveLobes(),
                                                 mtl.isPSDExclude(),
                                                 mtl.isPSDBlockMotionVectorsAtSurface(),
                                                 mtl.getPSDDominantDeltaLobeP1());
```

Keep the existing `ior` and nested dielectric follow-up path unchanged.

- [ ] **Step 3: Confirm reference raygen uses default active lobes**

In `PathTracerSample.rgen`, keep the existing `MakeStandardBSDFData(payload.worldNormal, ...)` call shape. The default overload from Task 2 must provide `kLobeTypeAll`, no PSD exclusion, no motion-vector block, and dominant delta `0`.

- [ ] **Step 4: Verify construction points**

Run:

```powershell
rg -n "activeLobes|psdExclude|psdBlockMotionVectorsAtSurface|MakeStandardBSDFData\\(worldNormal" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit
rg -n "MakeStandardBSDFData\\(payload.worldNormal" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: closest-hit stable-plane construction passes explicit state; reference raygen still uses the compatibility overload.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit
git -C DiligentSamples commit -m "feat(rtxpt): wire BxDF active lobe state" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 8: Mapping And Source-Level Contract Checks

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: all BxDF-touched shader files

- [ ] **Step 1: Update BxDF mapping table**

In `RTXPT_FORK_MAPPING.md`, extend the Materials mapping section with these rows:

```markdown
| `DiffuseReflectionFrostbite` | `DiffuseReflectionFrostbite` | Frostbite diffuse reflection component; Diligent wrapper maps world `wo` to local reference `wi` |
| `DiffuseTransmissionLambert` | `DiffuseTransmissionLambert` | Opposite-hemisphere Lambertian diffuse transmission |
| `SpecularReflectionMicrofacet` | `SpecularReflectionMicrofacet` | GGX BVNDF reflection, active-lobe gated, delta reflection when roughness falls below `kMinGGXAlpha` |
| `SpecularReflectionTransmissionMicrofacet` | `SpecularReflectionTransmissionMicrofacet` | GGX dielectric reflection/transmission, thin-surface eta override for transmission only |
| `FalcorBSDF` | `FalcorBSDF` | Four-component mixture with active-lobe gating, reference mixture pdf additions, sampled delta `pdf = 0` |
| `FalcorBSDF::evalDeltaLobes` | `ActiveBSDF::evalDeltaLobes` bridge | Exports transmission delta slot `0` and reflection delta slot `1` for stable-plane branch IDs |
| `MaterialHeader` PSD accessors | `Scene/Material/MaterialData.hlsli` plus `StablePlaneMaterialState` | Minimum PSD compatibility surface: exclude, block motion vectors at surface, dominant delta lobe plus one |
```

- [ ] **Step 2: Run source-level contract checks**

Run:

```powershell
rg -n "struct DiffuseReflectionFrostbite|struct DiffuseTransmissionLambert|struct SpecularReflectionMicrofacet|struct SpecularReflectionTransmissionMicrofacet|struct FalcorBSDF|MakeMaterialHeader|evalDeltaLobes" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "kLobeTypeNonDeltaReflection|kLobeTypeNonDeltaTransmission|kLobeTypeAll" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli
rg -n "isPSDExclude|isPSDBlockMotionVectorsAtSurface|getPSDDominantDeltaLobeP1" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli
rg -n "deltaLobes\\[0\\] = deltaTransmission|deltaLobes\\[1\\] = deltaReflection|kBSDFLobeDeltaTransmission\\) != 0u \\? 0u" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: each command finds the corresponding implementation points.

- [ ] **Step 3: Run stale-shape checks**

Run:

```powershell
rg -n "GetBSDFLobeProbabilities|BSDFLobeProbabilities|EvalDiffuseTransmission|SampleDiffuseTransmission|EvalSpecularReflectionTransmission|EvalPdfSpecularReflectionTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "deltaLobeIndex = \\(lobe & kBSDFLobeSpecularTransmission\\)" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli
```

Expected: both commands print no matches.

- [ ] **Step 4: Run whitespace checks**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no output and exit code 0.

- [ ] **Step 5: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): record BxDF fidelity mappings" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 9: Build And Visual Validation

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: `DiligentSamples/Samples/RTXPT/assets/convergence-test.scene.json`
- Verify: `D:/RTXPT-fork/Assets/convergence-test.scene.json`

- [ ] **Step 1: Build the RTXPT sample target**

Run from the superproject root:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Expected: build succeeds and ray-tracing shaders compile for the active local backend configuration.

- [ ] **Step 2: Confirm all path-tracer variants compile through the RT pass**

Run:

```powershell
rg -n "PATH_TRACER_MODE_REFERENCE|PATH_TRACER_MODE_BUILD_STABLE_PLANES|PATH_TRACER_MODE_FILL_STABLE_PLANES|TraceRays" DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: all three variants remain defined and use the same BxDF include surface.

- [ ] **Step 3: Confirm material texture predecessor before final image judgment**

Run or inspect the predecessor fix evidence:

```powershell
rg -n "Material textures loaded|Material textures bound|TextureCount|external texture|convergence-test" DiligentSamples/Samples/RTXPT/src docs/superpowers
```

Expected: there is evidence that external material JSON textures can be loaded and bound. If this is not true, record BxDF as source/build-complete but defer final `convergence-test` visual acceptance.

- [ ] **Step 4: Run D3D12 visual comparison**

Run the Diligent RTXPT sample on D3D12 with `convergence-test.scene.json`.

Acceptance checks:

- Material texture count is non-zero for ConvergenceTest after the predecessor fix.
- Reference mode renders without NaN, infinity, black-frame, or explosive firefly regression.
- Bottom-row spheres no longer show BxDF-attributable color/probability/roughness mismatches against RTXPT-fork after matching exposure, tonemapping, sample count, bounce count, NEE settings, and camera.
- Sampled delta events behave as zero-pdf scatter events without breaking firefly filtering.

- [ ] **Step 5: Run realtime stable-plane smoke comparison**

Enable realtime mode and run enough frames to exercise BUILD and FILL stable-plane variants.

Acceptance checks:

- `BuildStablePlanes` and `FillStablePlanes` variants dispatch.
- `StablePlanesOnScatter` receives lobe bits, `pdf`, `lobeP`, and delta index without branch-ID inversion.
- Delta transmission follows branch slot `0`; delta reflection follows branch slot `1`.
- PSD motion-vector block checks call the stable-plane material state method and do not silently always return false unless the material state is false.

- [ ] **Step 6: Run Vulkan smoke verification when Vulkan RT is configured**

Run the same scene on Vulkan RT when local configuration supports it.

Expected: shader translation and runtime execution do not expose HLSL constructs that only compile on D3D12.

- [ ] **Step 7: Record validation evidence**

Update the implementation notes or final handoff with:

- Build command and result.
- Backend and scene used.
- Sample count, bounce count, NEE settings, exposure, tonemapping, and camera.
- Reference and Diligent screenshots or HDR captures.
- Per-sphere comparison notes for the bottom row.
- Remaining mismatch category if any: input-data, BxDF, NEE/MIS, geometry-normal, or post-processing.

Expected: final report separates BxDF source parity from external texture input parity.

---

## Final Phase Verification

Run these before claiming the BxDF fidelity phase is source-complete:

```powershell
rg -n "struct DiffuseReflectionFrostbite|struct DiffuseTransmissionLambert|struct SpecularReflectionMicrofacet|struct SpecularReflectionTransmissionMicrofacet|struct FalcorBSDF|MakeMaterialHeader|evalDeltaLobes|cBxDFMaxDeltaLobes" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
rg -n "kLobeTypeNonDeltaReflection|kLobeTypeNonDeltaTransmission" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli
rg -n "setPSDExclude|isPSDExclude|setPSDBlockMotionVectorsAtSurface|isPSDBlockMotionVectorsAtSurface|setPSDDominantDeltaLobeP1|getPSDDominantDeltaLobeP1" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli
rg -n "activeLobes|psdExclude|psdBlockMotionVectorsAtSurface|psdDominantDeltaLobeP1" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit
rg -n "GetBSDFLobeProbabilities|BSDFLobeProbabilities|EvalDiffuseTransmission|SampleDiffuseTransmission|EvalSpecularReflectionTransmission|EvalPdfSpecularReflectionTransmission|deltaLobeIndex = \\(lobe & kBSDFLobeSpecularTransmission\\)" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
git -C DiligentSamples diff --check -- Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerTypes.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected:

- The first four `rg` commands find the new component, lobe, PSD, and stable-plane state surfaces.
- The stale-shape `rg` command prints no matches.
- `diff --check` prints no whitespace errors.

Run build verification before claiming runtime-complete:

```powershell
cmake --build build\x64\Debug --config Debug --target DiligentSamples
```

Runtime-complete requires D3D12 visual validation and, when locally available, Vulkan RT smoke validation.

## Self-Review

- Spec coverage: component BxDFs are assigned to Tasks 3 through 5; active-lobe and PSD compatibility are assigned to Tasks 1, 2, 6, and 7; delta export and sampled delta index order are assigned to Task 6; stable-plane integration is assigned to Tasks 6 and 7; mapping and validation are assigned to Tasks 8 and 9.
- Scope control: this plan does not replace the path tracer, add a Donut bridge, change root build configuration, add resource schemas, or rewrite scene assets.
- Predecessor dependency: external material texture loading is explicitly checked before final `convergence-test` visual acceptance.
- Compatibility: `MakeStandardBSDFData`, `EvalBSDF`, and `SampleBSDF` remain public wrappers for current reference and fill-stable-plane call sites.
- Naming/style: shader type names match RTXPT-fork where behavior matches, while file organization and includes stay Diligent-native.
