# RTXPT Phase R3.G5 Light Importance Sampling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add reference-mode light importance sampling for the DiligentEngine RTXPT path tracer: replace the current uniform analytic/emissive light picks with a configurable RIS/WRS selector (`NEECandidateSamples`, `NEEFullSamples`) while keeping the estimator unbiased and preserving the current uniform mode as a debug/fallback path.

**Architecture:** The current R2 renderer already has the data G5 needs: `RTXPTLights` uploads analytic punctual lights, the emissive-triangle build pass fills `t_EmissiveTriangles`, and raygen already does NEE plus emissive BSDF-hit MIS. G5 adds a compact Diligent-native global light-proxy table over analytic lights plus one optional emissive bucket, then uses a new `PathTracer/Lighting/LightSampler.hlsli` helper to draw N candidates, run weighted reservoir sampling, and visibility-test the selected full samples. The same proxy pdf is used on the BSDF-hit emissive side, so toggling between uniform and Power+ sampling changes variance only, not the converged result.

**Tech Stack:** C++17 in `DiligentSamples/Samples/RTXPT/src`, HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, Diligent structured buffers/SRVs/static raygen bindings, Dear ImGui for controls. There is no shader unit-test harness; verification is layout guards, targeted source checks, and manual GPU runs only when explicitly requested.

---

## Context You Need Before Starting

**Current baseline:** R0, R0.5, R1, and R2 have landed. The current shader layout is already the post-R0.5 `PathTracer/` tree. R2 added emissive-triangle NEE and MIS, so the current direct-light path is:

- Analytic lights: `PathTracer::SampleAnalyticNEE` in `PathTracer/PathTracer.hlsli`, selecting one analytic light uniformly from `Bridge::getLightCount()`.
- Emissive triangles: `PathTracer::SampleEmissiveNEE` in `PathTracer/PathTracer.hlsli`, selecting one triangle uniformly from `Bridge::getEmissiveTriangleCount()`.
- BSDF-hit emissive MIS: `PathTracerSample.rgen` multiplies the closest-hit-provided `payload.emissiveLightPdf` by the old uniform triangle selection pdf (`1 / emissiveCount`).
- UI placeholders: `RTXPTSample.cpp` shows "Sampling technique", "Candidate samples", "Full samples", and "MIS Type" disabled in the "NEE settings" block.

**Spec name to current path map:**

| Spec-era name | Current path for this plan |
|---|---|
| `RTXPTLightSampling.hlsli` / RIS header | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli` (new) |
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTLightSampling.hlsli` light decode | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli` |
| `RTXPTSceneBridge.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli` |
| shared CPU/GPU constants | `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` and `.../PathTracer/PathTracerShared.h` |
| light inventory / proxy data | `DiligentSamples/Samples/RTXPT/src/RTXPTLights.{hpp,cpp}` |
| raygen bindings | `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` |
| UI | `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` |

**RTXPT-fork reference anchors** (read-only):

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli:52-161` - `NEEWeightedReservoirSampler` and `GenerateLightSample`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli:282-314` - candidate/full-sample loop.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightSampler.hlsli:229-344` - light-selection pdfs and BSDF-side MIS.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:153-155` - `NEEType`, `NEECandidateSamples = 5`, `NEEFullSamples = 1`.

## Scope

This plan implements G5 only:

- Add frame constants for `NEEType`, `NEECandidateSamples`, `NEEFullSamples`, and `NEEMISType` (carried for UI parity; the shader uses the "Full" path in this plan).
- Add a compact `RTXPTLightProxy` structured buffer built by `RTXPTLights`.
- Add `Lighting/LightSampler.hlsli` with a Diligent-native global proxy selector and WRS helper.
- Replace the separate uniform analytic/emissive NEE calls with one direct-light RIS/WRS path.
- Update emissive BSDF-hit MIS to use the same light-selection pdf as the new NEE path.
- Enable the UI controls for `Uniform` and `Power+`, candidate count, and full sample count.

This plan intentionally does not implement:

- G6 photometric/shaped punctual-light units. The existing `lightIntensityScale` and `EvalAnalyticLight` energy model remain unchanged.
- R4 HDR environment-map importance sampling. Procedural-sky environment NEE stays as it is.
- RTXPT-fork's full `LightsBaker`, temporal feedback, local sampling, or NEE-AT. The `NEE-AT` UI option remains visible but disabled.
- Selectable approximate MIS modes. `NEEMISType` remains visible but disabled; the implementation uses the full MIS path.
- Payload growth. The current 80-byte payload already carries `emissiveLightPdf`; the selection pdf is reconstructed from the proxy table in raygen.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - grows `PathTracerConstants` to 64 bytes and defines `RTXPTLightProxy`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - mirrors the 64-byte constants layout.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp` - adds CPU mirror of `RTXPTLightProxy`, proxy-buffer accessor, stats, and private helper declarations.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp` - builds and uploads the proxy CDF buffer after analytic lights and emissive triangle counts are known.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli` - declares `t_LightProxies` and adds proxy accessors.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli` - proxy selection, candidate generation, WRS, and emissive selection-pdf helpers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` - replaces uniform analytic/emissive NEE helpers with the shared direct-light RIS/WRS path.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - calls the new direct-light sampler and uses the new emissive MIS pdf.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` - adds/binds `t_LightProxies` for raygen.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - uploads G5 settings and enables the UI controls.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - registers the new HLSL header.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - records the new light sampler mapping and divergence from the full RTXPT-fork baker/local sampler.

## Cross-Cutting Contracts

- **Settings layout:** `PathTracerConstants` grows from 48 to 64 bytes in C++ and HLSL. `SampleConstants` grows from 208 to 224 bytes. The four new scalar fields make the struct 16 scalar words, so the constant-buffer layout stays 16-byte aligned.
- **Proxy layout:** `RTXPTLightProxy` is a 16-byte structured-buffer element mirrored in C++ and HLSL: `prefixWeight`, `weight`, `index`, `kind`. The proxy table is ordered as all analytic-light proxies followed by one emissive-bucket proxy when `emissiveTriangleCount > 0`.
- **Proxy count:** Raygen derives the real proxy count as `analyticLightCount + (emissiveTriangleCount > 0 ? 1 : 0)`. The C++ side still uploads one dummy proxy when the real count is zero so `t_LightProxies` is never null.
- **Emissive pdf:** Closest-hit continues to return only the triangle's solid-angle pdf in `payload.emissiveLightPdf`. Raygen reconstructs the full light pdf as `emissiveBucketSelectionPdf / emissiveTriangleCount * payload.emissiveLightPdf`. No per-triangle light-index table is introduced.
- **MIS sample count:** `NEEFullSamples` participates in MIS (`fullSamples * lightPdf`); `NEECandidateSamples` participates in the reservoir correction and does not multiply the BSDF-side full MIS pdf.
- **Bindings:** `t_LightProxies`, `t_Lights`, and `t_EmissiveTriangles` are raygen-only STATIC variables. Closest-hit stays free of direct-light sampler resources.
- **Backends:** No backend-specific code is introduced. The new buffer is a standard structured SRV and must work on D3D12 and Vulkan.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: `DiligentSamples/Samples/RTXPT`

- [ ] **Step 1: Confirm working-tree state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Unrelated dirty files may exist, but do not overwrite them. If any of the files in this plan are already modified, inspect them before editing and preserve user changes.

- [ ] **Step 2: Confirm the G5 header does not already exist**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
```

Expected:

```text
False
```

- [ ] **Step 3: Confirm the current R3 placeholders are still present**

Run:

```powershell
rg -n "Phase R3|NEECandidateSamples|NEEFullSamples|NEEMISType|Sampling technique" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected facts:

```text
RTXPTSample.cpp has disabled UI controls for Sampling technique, Candidate samples, Full samples, and MIS Type.
PathTracerSample.rgen has a combined Phase R3/R4 TODO about light importance sampling and HDR env-map importance sampling.
```

- [ ] **Step 4: Confirm current constants sizes**

Run:

```powershell
rg -n "static_assert\\(sizeof\\(PathTracerConstants\\)|static_assert\\(sizeof\\(SampleConstants\\)" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```

Expected current values before Task 1:

```text
PathTracerConstants == 48
SampleConstants == 208
```

---

### Task 1: Grow Path-Tracer Constants For G5 Controls

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`

Context: `RTXPTReferenceUIState` already has `NEEType`, `NEECandidateSamples`, `NEEFullSamples`, and `NEEMISType`. This task carries them into the GPU frame constants. The shader uses `NEEType`, `NEECandidateSamples`, and `NEEFullSamples`; `NEEMISType` is uploaded for layout/UI parity but the control stays disabled in G5.

- [ ] **Step 1: Replace the C++ `PathTracerConstants` tail**

In `RTXPTSample.hpp`, replace the current `PathTracerConstants` struct with this exact layout:

```cpp
struct PathTracerConstants
{
    Uint32 bounceCount       = 4;
    Uint32 sampleIndex       = 0;
    Uint32 resetAccumulation = 1;
    Uint32 minBounceCount    = 0;

    Uint32 NEEEnabled            = 1;    // Non-zero enables next-event estimation (direct light sampling).
    Uint32 environmentNEEEnabled = 1;    // bit 0 enables environment NEE; bits 1..31 pack emissive triangle count (G4).
    float  environmentIntensity  = 1.0f; // Scales the procedural-sky environment radiance.
    float  lightIntensityScale   = 1.0f; // Scales analytic (punctual) light radiance.

    Uint32 maxNEEBounceCount  = 16; // NEE budget clamp; default covers the full bounce budget.
    Uint32 analyticLightCount = 0;  // CPU-side valid analytic lights; the dummy binding light is not sampled.
    Uint32 NEEType            = 1;  // G5: 0=Uniform, 1=Power+, 2=NEE-AT (deferred/disabled in UI).
    Uint32 NEECandidateSamples = 5; // G5: RIS candidate count per full sample.

    Uint32 NEEFullSamples          = 1;    // G5: visibility-tested full samples.
    Uint32 NEEMISType              = 0;    // G5 UI parity: 0=Full; approximate modes remain disabled in this plan.
    float  fireflyFilterThreshold  = 0.0f; // G1 adaptive firefly filter; 0 disables the filter.
    float  exposureScale           = 1.0f; // Scene camera exposure multiplier before in-raygen ACES.
};
static_assert(sizeof(PathTracerConstants) == 64, "PathTracerConstants layout must match PathTracer/PathTracerShared.h");
```

Then update:

```cpp
static_assert(sizeof(SampleConstants) == 224, "SampleConstants layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Mirror the same layout in HLSL**

In `PathTracerShared.h`, replace `struct PathTracerConstants` with:

```hlsl
struct PathTracerConstants
{
    uint bounceCount;       // Maximum number of secondary bounces; 0 means primary-ray only.
    uint sampleIndex;       // 0-based index of the sample being added this frame.
    uint resetAccumulation; // Non-zero means raygen should overwrite the accumulation buffer instead of blending.
    uint minBounceCount;    // Russian-roulette start bounce.

    uint  NEEEnabled;            // Non-zero enables next-event estimation (direct light sampling) at each hit.
    uint  environmentNEEEnabled; // bit 0: environment NEE enabled; bits 1..31: emissive triangle count (G4).
    float environmentIntensity;  // Scales the procedural-sky environment radiance.
    float lightIntensityScale;   // Scales analytic (punctual) light radiance.

    uint maxNEEBounceCount;   // Limits NEE work to the first N path bounces.
    uint analyticLightCount;  // CPU-side count of valid analytic lights; the dummy light is not sampled.
    uint NEEType;             // G5: 0=Uniform, 1=Power+, 2=NEE-AT (deferred/disabled in UI).
    uint NEECandidateSamples; // G5: RIS candidate count per full sample.

    uint  NEEFullSamples;         // G5: visibility-tested full samples.
    uint  NEEMISType;             // G5 UI parity: 0=Full; approximate modes remain disabled in this plan.
    float fireflyFilterThreshold; // G1 adaptive firefly filter: soft-cap level; 0 disables the filter entirely.
    float exposureScale;          // Scene camera exposure multiplier applied before the in-raygen ACES curve.
};
```

- [ ] **Step 3: Run the layout-source check**

Run:

```powershell
rg -n "PathTracerConstants|NEECandidateSamples|NEEFullSamples|NEEMISType|SampleConstants\\) == 224" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
```

Expected: the new fields appear in both files in the same order; C++ `PathTracerConstants` asserts 64 and `SampleConstants` asserts 224.

- [ ] **Step 4: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
git -C DiligentSamples commit -m "feat(rtxpt): add light importance sampling constants" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Add The Global Light Proxy Buffer

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`

Context: G5 does not port RTXPT-fork's full `LightsBaker`. It adds a minimal global CDF table that lets raygen draw from either equal-weight proxies (`Uniform`) or emitted-power proxies (`Power+`). The table has one proxy per analytic light and one optional emissive bucket; the bucket then samples a triangle uniformly.

- [ ] **Step 1: Add proxy constants and HLSL struct**

In `PathTracerShared.h`, after `PolymorphicLightInfo`, add:

```hlsl
static const uint kLightProxyKindAnalytic       = 0u;
static const uint kLightProxyKindEmissiveBucket = 1u;

// Mirrors Diligent::RTXPTLightProxy in RTXPTLights.hpp (16 bytes).
// prefixWeight is the inclusive cumulative weight after this proxy.
struct RTXPTLightProxy
{
    float prefixWeight;
    float weight;
    uint  index;
    uint  kind;
};
```

- [ ] **Step 2: Add the C++ mirror and buffer ownership**

In `RTXPTLights.hpp`, add the C++ mirror after `PolymorphicLightInfo`:

```cpp
constexpr Uint32 kLightProxyKind_Analytic       = 0u;
constexpr Uint32 kLightProxyKind_EmissiveBucket = 1u;

struct RTXPTLightProxy
{
    float  prefixWeight = 0.0f;
    float  weight       = 0.0f;
    Uint32 index        = 0;
    Uint32 kind         = kLightProxyKind_Analytic;
};
static_assert(sizeof(RTXPTLightProxy) == 16, "RTXPTLightProxy layout must match PathTracer/PathTracerShared.h");
```

Extend `RTXPTLightStats`:

```cpp
struct RTXPTLightStats
{
    Uint32      LightCount            = 0;
    Uint32      EmissiveTriangleCount = 0;
    Uint32      LightProxyCount       = 0;
    float       LightProxyTotalWeight = 0.0f;
    std::string LastError;
};
```

Add accessors and private state:

```cpp
IBuffer* GetLightProxyBuffer() const { return m_LightProxyBuffer; }
Uint32   GetLightProxyCount() const { return m_Stats.LightProxyCount; }

bool UploadLightProxyBuffer(IRenderDevice* pDevice);

RefCntAutoPtr<IBuffer>       m_LightProxyBuffer;
std::vector<PolymorphicLightInfo> m_AnalyticLights;
float                        m_EmissiveProxyWeight = 0.0f;
```

Keep the existing public API otherwise unchanged.

- [ ] **Step 3: Reset and remember analytic light data**

In `RTXPTLights::Reset`, release the new buffer and clear the cached vectors:

```cpp
void RTXPTLights::Reset()
{
    m_LightBuffer.Release();
    m_EmissiveTriangleBuffer.Release();
    m_LightProxyBuffer.Release();
    m_AnalyticLights.clear();
    m_EmissiveProxyWeight = 0.0f;
    m_Stats = {};
}
```

In `UploadLightBuffer`, before adding the disabled dummy light, preserve the real analytic list and keep `LightCount` equal to that real analytic count. Do not recompute `LightCount` from `Lights.size()` after the dummy light is appended:

```cpp
m_AnalyticLights = Lights;
m_Stats.LightCount = static_cast<Uint32>(m_AnalyticLights.size());
```

Then keep the existing dummy-light upload behavior for `m_LightBuffer`; the dummy only keeps the SRV non-null and must not appear in `m_AnalyticLights`, `LightCount`, or the proxy table.

- [ ] **Step 4: Accumulate a rough emissive proxy weight while counting triangles**

In `RTXPTLights.cpp`, add a helper in the anonymous namespace:

```cpp
float MaxRGB(const float3& V)
{
    return std::max(V.x, std::max(V.y, V.z));
}

float GetMaterialEmissionMagnitude(const GLTF::Material& Material, const RTXPTMaterialExtension* pExtension)
{
    if (pExtension != nullptr && pExtension->Loaded)
        return MaxRGB(pExtension->EmissiveFactor);

    return MaxRGB(Material.Attribs.EmissiveFactor);
}
```

Inside `UploadEmissiveTriangles`, set `m_EmissiveProxyWeight = 0.0f;` before the loops. When an NEE-eligible primitive contributes `TriangleCount`, also add:

```cpp
const Uint32 TriangleCount = GetPrimitiveTriangleCount(Primitive);
EmissiveTriangleCount += TriangleCount;
m_EmissiveProxyWeight += static_cast<float>(TriangleCount) *
    std::max(1e-6f, GetMaterialEmissionMagnitude(Model.Materials[Primitive.MaterialId], pExtension));
```

This is only the proposal weight. It does not change emitted radiance, so it cannot bias the estimator.

- [ ] **Step 5: Build and upload the proxy CDF**

Add `RTXPTLights::UploadLightProxyBuffer`:

```cpp
bool RTXPTLights::UploadLightProxyBuffer(IRenderDevice* pDevice)
{
    m_LightProxyBuffer.Release();
    if (pDevice == nullptr)
    {
        m_Stats.LastError = "RTXPT light proxy buffer requires a render device";
        LOG_ERROR_MESSAGE(m_Stats.LastError.c_str());
        return false;
    }

    std::vector<RTXPTLightProxy> Proxies;
    Proxies.reserve(m_AnalyticLights.size() + (m_Stats.EmissiveTriangleCount > 0 ? 1u : 0u));

    float Prefix = 0.0f;
    for (Uint32 LightIndex = 0; LightIndex < static_cast<Uint32>(m_AnalyticLights.size()); ++LightIndex)
    {
        const PolymorphicLightInfo& Light = m_AnalyticLights[LightIndex];
        const float3 Radiance = float3{Light.colorIntensity.x, Light.colorIntensity.y, Light.colorIntensity.z} *
            std::max(Light.colorIntensity.w, 0.0f);
        const float Weight = std::max(1e-6f, MaxRGB(Radiance));
        Prefix += Weight;
        Proxies.push_back(RTXPTLightProxy{Prefix, Weight, LightIndex, kLightProxyKind_Analytic});
    }

    if (m_Stats.EmissiveTriangleCount > 0)
    {
        const float Weight = std::max(1e-6f, m_EmissiveProxyWeight);
        Prefix += Weight;
        Proxies.push_back(RTXPTLightProxy{Prefix, Weight, 0u, kLightProxyKind_EmissiveBucket});
    }

    m_Stats.LightProxyCount       = static_cast<Uint32>(Proxies.size());
    m_Stats.LightProxyTotalWeight = Prefix;
    if (Proxies.empty())
        Proxies.emplace_back();

    BufferDesc Desc;
    Desc.Name              = "RTXPT light proxy buffer";
    Desc.Usage             = USAGE_IMMUTABLE;
    Desc.BindFlags         = BIND_SHADER_RESOURCE;
    Desc.Mode              = BUFFER_MODE_STRUCTURED;
    Desc.ElementByteStride = sizeof(RTXPTLightProxy);
    Desc.Size              = sizeof(RTXPTLightProxy) * Proxies.size();

    BufferData Data{Proxies.data(), Desc.Size};
    pDevice->CreateBuffer(Desc, &Data, &m_LightProxyBuffer);

    VERIFY(m_LightProxyBuffer, "Failed to create RTXPT light proxy buffer");
    return m_LightProxyBuffer != nullptr;
}
```

At the end of `UploadEmissiveTriangles`, after `UploadEmissiveTriangleBuffer(...)` succeeds, call `UploadLightProxyBuffer(pDevice)` and return the conjunction. This keeps the proxy table synchronized with the analytic-light upload and the emissive-triangle count.

- [ ] **Step 6: Add shader bridge accessors**

In `PathTracerBridge.hlsli`, add the new SRV near the other globals:

```hlsl
StructuredBuffer<RTXPTLightProxy>       t_LightProxies;
```

Add accessors in `namespace Bridge`:

```hlsl
uint getLightProxyCount()
{
    return getLightCount() + ((getEmissiveTriangleCount() > 0u) ? 1u : 0u);
}

RTXPTLightProxy getLightProxy(uint index)
{
    return t_LightProxies[index];
}

float getLightProxyTotalWeight()
{
    const uint proxyCount = getLightProxyCount();
    return proxyCount > 0u ? getLightProxy(proxyCount - 1u).prefixWeight : 0.0;
}
```

- [ ] **Step 7: Run the proxy-source checks**

Run:

```powershell
rg -n "RTXPTLightProxy|t_LightProxies|LightProxyCount|LightProxyTotalWeight|UploadLightProxyBuffer|getLightProxy" DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli
```

Expected: C++ and HLSL both define the 16-byte proxy type; `RTXPTLights` owns and uploads the buffer; `PathTracerBridge.hlsli` exposes the proxy table to raygen.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTLights.hpp Samples/RTXPT/src/RTXPTLights.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): add light sampling proxy buffer" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Add The RIS/WRS Light Sampler Header

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

Context: keep raw light evaluation in `PolymorphicLight.hlsli`. The new header owns proposal selection, WRS, and selection-pdf helpers. It must not call `TraceRay`; visibility stays in `PathTracer.hlsli`.

- [ ] **Step 1: Create `LightSampler.hlsli` with the candidate and reservoir types**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`:

```hlsl
#ifndef __LIGHT_SAMPLER_HLSLI__
#define __LIGHT_SAMPLER_HLSLI__

#include "../PathTracerBridge.hlsli"
#include "../Rendering/Materials/BxDF.hlsli"
#include "../Utils/SampleGenerators.hlsli"
#include "PolymorphicLight.hlsli"

struct DirectLightSample
{
    float3 dir;
    float  distance;
    float3 radianceOverPdf; // incident radiance divided by proposal pdf.
    float  proposalPdf;     // selection pdf * solid-angle pdf; for delta analytic lights, solid-angle pdf is 1.
    float3 bsdfF;
    float  bsdfPdf;
    uint   kind;
    uint   index;
    bool   valid;
    bool   sampleableByBSDF;
};

DirectLightSample DirectLightSample_make_empty()
{
    DirectLightSample s;
    s.dir              = float3(0.0, 1.0, 0.0);
    s.distance         = 0.0;
    s.radianceOverPdf  = float3(0.0, 0.0, 0.0);
    s.proposalPdf      = 0.0;
    s.bsdfF            = float3(0.0, 0.0, 0.0);
    s.bsdfPdf          = 0.0;
    s.kind             = kLightProxyKindAnalytic;
    s.index            = 0u;
    s.valid            = false;
    s.sampleableByBSDF = false;
    return s;
}

struct NEEWeightedReservoirSampler
{
    DirectLightSample candidate;
    float             weightSum;
    float             candidateWeight;

    static NEEWeightedReservoirSampler make()
    {
        NEEWeightedReservoirSampler r;
        r.candidate       = DirectLightSample_make_empty();
        r.weightSum       = 0.0;
        r.candidateWeight = 0.0;
        return r;
    }

    void Add(float randomValue, DirectLightSample sample, float weight)
    {
        if (!sample.valid || weight <= 0.0)
            return;

        weightSum += weight;
        if (randomValue < saturate(weight / weightSum))
        {
            candidate       = sample;
            candidateWeight = weight;
        }
    }

    float CandidateProbability()
    {
        return weightSum > 0.0 ? candidateWeight / weightSum : 0.0;
    }
};
```

- [ ] **Step 2: Add proxy sampling helpers**

Continue the same file with:

```hlsl
uint SamplePowerProxyIndex(float rnd)
{
    const uint proxyCount = Bridge::getLightProxyCount();
    if (proxyCount == 0u)
        return 0u;

    const float totalWeight = Bridge::getLightProxyTotalWeight();
    if (totalWeight <= 0.0)
        return 0u;

    const float target = rnd * totalWeight;
    uint lo = 0u;
    uint hi = proxyCount;
    [loop]
    while (lo + 1u < hi)
    {
        const uint mid = (lo + hi) >> 1u;
        if (target <= Bridge::getLightProxy(mid).prefixWeight)
            hi = mid;
        else
            lo = mid;
    }

    return target <= Bridge::getLightProxy(lo).prefixWeight ? lo : min(lo + 1u, proxyCount - 1u);
}

float GetProxySelectionPdf(uint proxyIndex, bool usePowerSampling)
{
    const uint proxyCount = Bridge::getLightProxyCount();
    if (proxyCount == 0u)
        return 0.0;

    if (!usePowerSampling)
        return 1.0 / float(proxyCount);

    const float totalWeight = Bridge::getLightProxyTotalWeight();
    return totalWeight > 0.0 ? Bridge::getLightProxy(proxyIndex).weight / totalWeight : 0.0;
}

uint SampleProxyIndex(float rnd, bool usePowerSampling)
{
    const uint proxyCount = Bridge::getLightProxyCount();
    if (proxyCount == 0u)
        return 0u;

    if (!usePowerSampling)
        return min(uint(rnd * float(proxyCount)), proxyCount - 1u);

    return SamplePowerProxyIndex(rnd);
}

float GetEmissiveTriangleSelectionPdf(bool usePowerSampling)
{
    const uint emissiveCount = Bridge::getEmissiveTriangleCount();
    const uint proxyCount    = Bridge::getLightProxyCount();
    if (emissiveCount == 0u || proxyCount == 0u)
        return 0.0;

    const uint  emissiveProxyIndex = proxyCount - 1u;
    const float bucketPdf          = GetProxySelectionPdf(emissiveProxyIndex, usePowerSampling);
    return bucketPdf / float(emissiveCount);
}
```

- [ ] **Step 3: Add candidate generation**

Still in `LightSampler.hlsli`, add a candidate generator that wraps analytic and emissive samples in the same pdf contract:

```hlsl
DirectLightSample GenerateDirectLightCandidate(StandardBSDFData bsdfData, float3 hitPos, float3 wo,
                                               inout SampleGenerator sg, bool usePowerSampling)
{
    DirectLightSample sample = DirectLightSample_make_empty();

    const uint proxyCount = Bridge::getLightProxyCount();
    if (proxyCount == 0u)
        return sample;

    const uint proxyIndex = SampleProxyIndex(sampleNext1D(sg), usePowerSampling);
    const RTXPTLightProxy proxy = Bridge::getLightProxy(proxyIndex);
    const float proxyPdf = GetProxySelectionPdf(proxyIndex, usePowerSampling);
    if (proxyPdf <= 0.0)
        return sample;

    if (proxy.kind == kLightProxyKindAnalytic)
    {
        LightSample analytic = EvalAnalyticLight(Bridge::getLight(proxy.index), hitPos);
        if (!analytic.valid)
            return sample;

        sample.dir             = analytic.dir;
        sample.distance        = analytic.distance;
        sample.proposalPdf     = proxyPdf;
        sample.radianceOverPdf = analytic.radiance * g_Const.ptConsts.lightIntensityScale / proxyPdf;
        sample.kind            = kLightProxyKindAnalytic;
        sample.index           = proxy.index;
        sample.valid           = true;
        sample.sampleableByBSDF = false;
    }
    else if (proxy.kind == kLightProxyKindEmissiveBucket)
    {
        const uint triCount = Bridge::getEmissiveTriangleCount();
        if (triCount == 0u)
            return sample;

        const uint triIndex = min(uint(sampleNext1D(sg) * float(triCount)), triCount - 1u);
        const EmissiveTriangle tri = Bridge::getEmissiveTriangle(triIndex);

        const float3 ng = cross(tri.edge1.xyz, tri.edge2.xyz);
        const float  ngLen = length(ng);
        if (ngLen <= 0.0)
            return sample;

        const float  area = 0.5 * ngLen;
        const float3 normal = ng / ngLen;
        const float3 bary = SampleTriangleUniform(sampleNext2D(sg));
        const float3 P = tri.base.xyz + tri.edge1.xyz * bary.y + tri.edge2.xyz * bary.z;
        const float3 toLight = P - hitPos;
        const float  distSq = max(1e-9, dot(toLight, toLight));
        const float  dist = sqrt(distSq);
        const float3 wi = toLight / dist;
        const float  cosTheta = abs(dot(normal, -wi));
        if (cosTheta <= 2e-9)
            return sample;

        const float selectionPdf = proxyPdf / float(triCount);
        const float solidAnglePdf = min(kMaxSolidAnglePdf, pdfAtoW(1.0 / area, dist, cosTheta));
        const float proposalPdf = selectionPdf * solidAnglePdf;
        if (proposalPdf <= 0.0)
            return sample;

        sample.dir              = wi;
        sample.distance         = dist * 0.9985;
        sample.proposalPdf      = proposalPdf;
        sample.radianceOverPdf  = tri.radiance.rgb / proposalPdf;
        sample.kind             = kLightProxyKindEmissiveBucket;
        sample.index            = triIndex;
        sample.valid            = true;
        sample.sampleableByBSDF = true;
    }

    const float specProb = getSpecularProbability(bsdfData, wo);
    EvalBSDF(bsdfData, wo, sample.dir, specProb, sample.bsdfF, sample.bsdfPdf);
    if (sample.bsdfPdf <= 0.0 || !sample.valid)
        return DirectLightSample_make_empty();

    return sample;
}
```

- [ ] **Step 4: Add the RIS target helper**

Add:

```hlsl
float EvalDirectLightCandidateWeight(DirectLightSample sample)
{
    if (!sample.valid || sample.proposalPdf <= 0.0 || sample.bsdfPdf <= 0.0)
        return 0.0;

    // Matches RTXPT-fork's cheap target: incident radiance over proposal pdf, weighted by the BSDF pdf.
    return max(sample.radianceOverPdf.x, max(sample.radianceOverPdf.y, sample.radianceOverPdf.z)) * sample.bsdfPdf;
}

#endif // __LIGHT_SAMPLER_HLSLI__
```

- [ ] **Step 5: Include and register the header**

In `PathTracer.hlsli`, add:

```hlsl
#include "Lighting/LightSampler.hlsli"
```

after the existing `PolymorphicLight.hlsli` include.

In `CMakeLists.txt`, add:

```cmake
    assets/shaders/PathTracer/Lighting/LightSampler.hlsli
```

near the other `PathTracer/Lighting/*` shader headers.

- [ ] **Step 6: Source-check the new header**

Run:

```powershell
rg -n "NEEWeightedReservoirSampler|GenerateDirectLightCandidate|GetEmissiveTriangleSelectionPdf|LightSampler.hlsli" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: the new file exists, is included by `PathTracer.hlsli`, and is listed in `CMakeLists.txt`.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add RIS light sampler shader helpers" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Replace Uniform Direct-Light NEE With RIS/WRS And Bind The Proxy Buffer

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

Context: this is the atomic runtime switch. Once raygen references `t_LightProxies`, the RT PSO must bind it in the same commit.

- [ ] **Step 1: Add the unified direct-light NEE helper**

In `PathTracer.hlsli`, keep `TraceVisibilityRay`, `SampleEnvironmentNEE`, and `ComputeBSDFEnvMISWeight`. Replace the old `SampleAnalyticNEE` and `SampleEmissiveNEE` helpers with:

```hlsl
float ComputeLightVsBSDFMISForLightSample(DirectLightSample sample, uint fullSamples)
{
    if (!sample.sampleableByBSDF || sample.bsdfPdf <= 0.0 || sample.proposalPdf <= 0.0)
        return 1.0;

    const float lightPdf = sample.proposalPdf * float(max(fullSamples, 1u));
    return PowerHeuristic(1.0, lightPdf, 1.0, sample.bsdfPdf);
}

float ComputeBSDFMISForEmissiveTriangle(float bsdfPdf, float emissiveSolidAnglePdf, uint fullSamples)
{
    if (bsdfPdf <= 0.0 || emissiveSolidAnglePdf <= 0.0)
        return 1.0;

    const bool usePowerSampling = g_Const.ptConsts.NEEType == 1u;
    const float selectionPdf = GetEmissiveTriangleSelectionPdf(usePowerSampling);
    if (selectionPdf <= 0.0)
        return 1.0;

    const float lightPdf = selectionPdf * emissiveSolidAnglePdf * float(max(fullSamples, 1u));
    return PowerHeuristic(1.0, bsdfPdf, 1.0, lightPdf);
}

float3 SampleDirectLightNEE(StandardBSDFData bsdfData, float3 hitPos, float3 visibilityOrigin,
                            float3 wo, inout SampleGenerator sg, float fireflyFilterK,
                            out bool sampledEmissive)
{
    sampledEmissive = false;

    const uint fullSamples = min(32u, g_Const.ptConsts.NEEFullSamples);
    if (fullSamples == 0u || Bridge::getLightProxyCount() == 0u)
        return float3(0.0, 0.0, 0.0);

    const bool usePowerSampling = g_Const.ptConsts.NEEType == 1u;
    const uint candidateSamples = max(1u, min(32u, g_Const.ptConsts.NEECandidateSamples));

    float3 result = float3(0.0, 0.0, 0.0);
    [loop]
    for (uint sampleIndex = 0u; sampleIndex < fullSamples; ++sampleIndex)
    {
        NEEWeightedReservoirSampler wrs = NEEWeightedReservoirSampler::make();

        [loop]
        for (uint candidateIndex = 0u; candidateIndex < candidateSamples; ++candidateIndex)
        {
            DirectLightSample candidate = GenerateDirectLightCandidate(bsdfData, hitPos, wo, sg, usePowerSampling);
            wrs.Add(sampleNext1D(sg), candidate, EvalDirectLightCandidateWeight(candidate));
        }

        DirectLightSample picked = wrs.candidate;
        const float candidateProbability = wrs.CandidateProbability();
        if (!picked.valid || candidateProbability <= 0.0)
            continue;

        if (!TraceVisibilityRay(visibilityOrigin, picked.dir, picked.distance))
            continue;

        const float wrsScale = 1.0 / (candidateProbability * float(candidateSamples));
        const float misWeight = ComputeLightVsBSDFMISForLightSample(picked, fullSamples);
        float3 contribution = picked.bsdfF * picked.radianceOverPdf * (wrsScale * misWeight / float(fullSamples));

        const float ffThreshold = g_Const.ptConsts.fireflyFilterThreshold;
        if (ffThreshold != 0.0)
        {
            const float neeK = ComputeNewScatterFireflyFilterK(fireflyFilterK, picked.proposalPdf, 1.0);
            contribution *= FireflyFilterShort(Average(contribution), ffThreshold, neeK);
        }

        sampledEmissive = sampledEmissive || picked.kind == kLightProxyKindEmissiveBucket;
        result += contribution;
    }

    return result;
}
```

Keep `SampleEnvironmentNEE` unchanged; environment importance sampling is R4.

- [ ] **Step 2: Update raygen calls**

In `PathTracerSample.rgen`, replace the current per-bounce direct-light block:

```hlsl
SampleGenerator sgNEELight = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEELightSampler);
pathRadiance += throughput * PathTracer::SampleAnalyticNEE(...);
SampleGenerator sgEmissive = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEEEmissive);
pathRadiance += throughput * PathTracer::SampleEmissiveNEE(...);
```

with:

```hlsl
bool sampledEmissiveNEE = false;
SampleGenerator sgNEELight = SampleGenerator_makeStateless(pixel, vertexIndex, sampleIndex, kSampleEffect_NEELightSampler);
pathRadiance += throughput * PathTracer::SampleDirectLightNEE(bsdfData, payload.worldPos, visibilityOrigin,
                                                               wo, sgNEELight, fireflyFilterK, sampledEmissiveNEE);
```

Keep the existing environment NEE block after it.

- [ ] **Step 3: Update emissive BSDF-hit MIS**

In `PathTracerSample.rgen`, replace the old uniform pdf branch:

```hlsl
const float lightPdf = (1.0 / float(emissiveCount)) * payload.emissiveLightPdf;
surfaceEmission *= PowerHeuristic(1.0, prevBsdfPdf, 1.0, lightPdf);
```

with:

```hlsl
const uint fullSamples = min(32u, g_Const.ptConsts.NEEFullSamples);
surfaceEmission *= PathTracer::ComputeBSDFMISForEmissiveTriangle(prevBsdfPdf, payload.emissiveLightPdf, fullSamples);
```

Set the previous-state flag at the end of the bounce from the helper output:

```hlsl
prevDidEmissiveNEE = useNEE && sampledEmissiveNEE;
```

This keeps BSDF-hit emission full-weight when no emissive NEE sample was possible, and it uses `fullSamples * lightPdf` when the light sampler competes with the BSDF.

- [ ] **Step 4: Add the RT pass parameter and stats bit**

In `RTXPTRayTracingPass.hpp`, extend `RTXPTRayTracingPassStats`:

```cpp
bool LightProxyBridgeBound = false;
```

Add a parameter after `pLightBuffer`:

```cpp
IBuffer* pLightProxyBuffer,
```

Update the implementation signature in `RTXPTRayTracingPass.cpp` with the same parameter order.

- [ ] **Step 5: Bind `t_LightProxies` as a raygen STATIC resource**

In `RTXPTRayTracingPass.cpp`, when building the resource layout for `FullPathTracer`, add:

```cpp
.AddVariable(SHADER_TYPE_RAY_GEN, "t_LightProxies", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
```

Create the SRV view:

```cpp
IDeviceObject* pLightProxyView = pLightProxyBuffer != nullptr ? pLightProxyBuffer->GetDefaultView(BUFFER_VIEW_SHADER_RESOURCE) : nullptr;
```

Bind it:

```cpp
m_Stats.LightProxyBridgeBound = SetStatic(SHADER_TYPE_RAY_GEN, "t_LightProxies", pLightProxyView, "light proxy buffer");
```

Include `LightProxyBridgeBound` in the final bridge-bound check. In the diagnostic/non-full path, set it to `true` with the other bridge booleans.

- [ ] **Step 6: Pass the buffer from `RTXPTSample.cpp`**

In both `m_RayTracingPass.Initialize(...)` calls in `RTXPTSample::CreatePhase4Passes`, pass:

```cpp
m_Lights.GetLightProxyBuffer(),
```

immediately after `m_Lights.GetLightBuffer()`.

- [ ] **Step 7: Run source checks**

Run:

```powershell
rg -n "SampleDirectLightNEE|ComputeBSDFMISForEmissiveTriangle|t_LightProxies|LightProxyBridgeBound|GetLightProxyBuffer" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src
```

Expected: raygen calls `SampleDirectLightNEE`, the old uniform analytic/emissive helper calls are gone from `PathTracerSample.rgen`, and `t_LightProxies` is bound in the raygen stage.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): sample direct lights with RIS reservoirs" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Wire Frame Constants And Enable The UI Controls

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`

Context: the UI placeholders already exist. This task makes the implemented controls live and keeps the non-G5 options disabled.

- [ ] **Step 1: Upload G5 settings**

In `RTXPTSample::UpdateFrameConstants`, after `analyticLightCount`, add:

```cpp
m_LastFrameConstants.ptConsts.NEEType             = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEType, 0, 1));
m_LastFrameConstants.ptConsts.NEECandidateSamples = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEECandidateSamples, 1, 32));
m_LastFrameConstants.ptConsts.NEEFullSamples      = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEFullSamples, 0, 32));
m_LastFrameConstants.ptConsts.NEEMISType          = static_cast<Uint32>(std::clamp(m_ReferenceUI.NEEMISType, 0, 0));
```

The `NEEMISType` clamp to `0` is deliberate for G5: the "Full" estimator is the only implemented mode.

- [ ] **Step 2: Make the sampling mode live for Uniform and Power+**

In the "NEE settings" block of `UpdateUI`, replace the disabled combo with a live two-choice combo:

```cpp
const char* SamplingTechniqueItems = "Uniform\0Power+\0\0";
if (ResetOnChange(ImGui::Combo("Sampling technique", &m_ReferenceUI.NEEType, SamplingTechniqueItems), "NEE sampling technique changed"))
    m_ReferenceUI.NEEType = std::clamp(m_ReferenceUI.NEEType, 0, 1);
```

Immediately after it, keep `NEE-AT` visible as a disabled text row:

```cpp
ImGui::BeginDisabled(true);
ImGui::TextUnformatted("NEE-AT");
ImGui::EndDisabled();
PlaceholderTooltip("NEE-AT requires RTXPT-fork's feedback/local sampling path and remains deferred.");
```

- [ ] **Step 3: Make candidate/full sample counts live and accumulation-resetting**

Replace the disabled `InputInt` blocks with:

```cpp
if (ResetOnChange(ImGui::InputInt("Candidate samples", &m_ReferenceUI.NEECandidateSamples, 1), "NEE candidate count changed"))
    m_ReferenceUI.NEECandidateSamples = std::clamp(m_ReferenceUI.NEECandidateSamples, 1, 32);

if (ResetOnChange(ImGui::InputInt("Full samples", &m_ReferenceUI.NEEFullSamples, 1), "NEE full sample count changed"))
    m_ReferenceUI.NEEFullSamples = std::clamp(m_ReferenceUI.NEEFullSamples, 0, 32);
```

If `NEEFullSamples` is set to 0, the direct-light sampler returns zero and BSDF-hit emissive MIS treats light sampling as disabled.

- [ ] **Step 4: Keep MIS Type disabled**

Leave `MIS Type` disabled but update the tooltip:

```cpp
ImGui::BeginDisabled(true);
ImGui::Combo("MIS Type", &m_ReferenceUI.NEEMISType, "Full\0ApproxInRealtime\0Approximate\0\0");
ImGui::EndDisabled();
PlaceholderTooltip("G5 uses the full light-vs-BSDF MIS path; approximate MIS modes remain deferred.");
```

- [ ] **Step 5: Add proxy status readouts**

In the status/debug section near the existing light stats, add:

```cpp
ImGui::Text("Light proxies: %u", m_Lights.GetStats().LightProxyCount);
ImGui::Text("Light proxy weight: %.3f", m_Lights.GetStats().LightProxyTotalWeight);
ImGui::Text("Light proxy bridge: %s", RTPassStats.LightProxyBridgeBound ? "bound" : "missing");
```

- [ ] **Step 6: Update the roadmap text**

Replace:

```cpp
ImGui::TextWrapped("TODO(RTXPT-Port Phase R3): light importance sampling (RIS/WRS) + photometric units.");
```

with:

```cpp
ImGui::TextWrapped("TODO(RTXPT-Port Phase R3.G6): photometric / shaped punctual-light units.");
```

G5 is no longer open once this plan lands; G6 remains as the R3 follow-up.

- [ ] **Step 7: Run UI/source checks**

Run:

```powershell
rg -n "Sampling technique|Candidate samples|Full samples|NEE-AT|Light proxies|Phase R3.G6|NEECandidateSamples|NEEFullSamples" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```

Expected: sampling technique and counts are live controls; `NEE-AT` and `MIS Type` are still visibly disabled; the roadmap no longer says RIS/WRS is open.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTSample.hpp
git -C DiligentSamples commit -m "feat(rtxpt): expose light importance sampling controls" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Mapping Doc, TODO Markers, And Verification

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Verify: all files touched by Tasks 1-5

- [ ] **Step 1: Update the mapping doc lighting rows**

In `RTXPT_FORK_MAPPING.md`, under `### T-D. Lighting Layer`, add:

```markdown
| `PathTracer/Lighting/LightSampler.hlsli` `NEEWeightedReservoirSampler` / `GenerateLightSample` | `PathTracer/Lighting/LightSampler.hlsli` `NEEWeightedReservoirSampler` / `GenerateDirectLightCandidate` / `SampleDirectLightNEE` | R3/G5 ports the RIS/WRS math over a compact Diligent global proxy table |
| `LightSampler::SampleGlobal` proxy table | `RTXPTLights::UploadLightProxyBuffer` + `StructuredBuffer<RTXPTLightProxy> t_LightProxies` | global-only CDF; no `LightsBaker`, feedback, local tiles, or NEE-AT in G5 |
```

Update the existing `TriangleLight::CalcSample` row from "uniform triangle selection (RIS is R3)" to:

```markdown
| `TriangleLight::CalcSample` | emissive bucket branch inside `GenerateDirectLightCandidate` | R2 stores `EmissiveTriangle`; R3 samples it through the RIS/WRS proxy table, still with uniform per-triangle selection inside the emissive bucket |
```

- [ ] **Step 2: Update the divergence note**

In the divergence section, replace:

```markdown
Textured-emissive triangles are excluded from NEE (BSDF-only) for now. Selection is
uniform (RIS/WRS is Phase R3). TODO: align one-sided + double-sided baker semantics.
```

with:

```markdown
Textured-emissive triangles are excluded from NEE (BSDF-only) for now. R3/G5 samples
constant emitters through a global RIS/WRS proxy table with one emissive bucket, not
RTXPT-fork's full `LightsBaker`/local-feedback system. TODO: align one-sided +
double-sided baker semantics.
```

- [ ] **Step 3: Split the shader TODO marker**

At the end of `PathTracerSample.rgen`, replace the combined R3/R4 marker:

```hlsl
// TODO(RTXPT-Port Phase R3/R4): Add light importance sampling / RIS (uniform analytic + emissive selection today) and HDR environment-map importance sampling + MIS (procedural-sky cosine env sampler today).
```

with:

```hlsl
// TODO(RTXPT-Port Phase R4): Add HDR environment-map importance sampling + MIS (procedural-sky cosine env sampler today).
```

G5's RIS/WRS work is now implemented; R4 stays open.

- [ ] **Step 4: Check for stale G5-open markers**

Run:

```powershell
rg -n "RIS/WRS|light importance sampling|Phase R3/R4|Phase R3\\):" DiligentSamples/Samples/RTXPT
```

Expected:

```text
Only mapping-doc history/description rows and the R3.G6 photometric-units roadmap marker remain. No runtime TODO should say RIS/WRS is still unimplemented.
```

- [ ] **Step 5: Check formatting whitespace**

Run:

```powershell
git -C DiligentSamples diff --check
```

Expected:

```text
no output
```

- [ ] **Step 6: Optional build/runtime verification (run only on explicit user request)**

```powershell
cmake --build build\x64\Debug --config Debug
```

Expected if run: the sample builds with the new constants layout and raygen proxy binding.

Manual GPU checks if the user asks for runtime verification:

```text
1. Launch Samples/RTXPT on D3D12.
2. Use a many-light scene or add multiple scene-json punctual lights.
3. Compare "Sampling technique: Uniform" vs "Power+" at the same sample count.
4. Expected: Power+ has lower many-light NEE noise; long accumulation converges to the same image.
5. Repeat on Vulkan.
```

- [ ] **Step 7: Final commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "docs(rtxpt): record light importance sampling mapping" -m "Co-Authored-By: GPT 5.5"
```

---

## Self-Review

- [x] **Spec coverage:** Covers G5's N-candidate RIS/WRS selector, default candidate/full sample counts, proxy-based Power+ selection, UI controls, many-light variance target, and unbiasedness via carried proposal pdfs. G6, R4, NEE-AT, and approximate MIS are explicitly out of scope.
- [x] **No accidental placeholders:** The only `TODO` text in this plan is a structured `TODO(RTXPT-Port ...)` marker being updated according to the repository open-work policy.
- [x] **Type consistency:** `PathTracerConstants` is 64 bytes in C++ and HLSL; `SampleConstants` is 224 bytes; `RTXPTLightProxy` is 16 bytes in C++ and HLSL; raygen derives proxy count from `analyticLightCount` and packed emissive count.
- [x] **MIS consistency:** NEE-side light samples use `proposalPdf` plus reservoir correction; emissive BSDF-hit MIS reconstructs the same proposal pdf and multiplies by `NEEFullSamples`. Candidate count affects WRS correction only.

## Execution Handoff

Implement task-by-task from `Task 0` through `Task 6`. Prefer one commit per task as written. If shader compilation fails after Task 4, first inspect the new `t_LightProxies` static binding and the include order around `PathTracerBridge.hlsli` / `LightSampler.hlsli`.
