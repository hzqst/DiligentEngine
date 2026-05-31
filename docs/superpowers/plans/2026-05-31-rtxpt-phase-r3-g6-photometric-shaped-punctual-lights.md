# RTXPT Phase R3.G6 Photometric Shaped Punctual Lights Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current raw `color * intensity` punctual-light evaluator with RTXPT-consistent photometric units, finite solid-angle sampling for punctual lights, smooth spot shaping, range attenuation, and matching light-proxy weights.

**Architecture:** This phase builds on the post-G5 direct-light RIS/WRS path. CPU light upload converts glTF and RTXPT scene-json lights into a compact Diligent-native `PolymorphicLightInfo` layout that stores RTXPT-style type, radiance/flux, finite radius, direction, range, cone softness, and directional solid angle. HLSL light sampling then returns `radiance / (selectionPdf * solidAnglePdf)` for analytic lights, so sphere and directional lights are sampled with finite PDFs while emissive-triangle MIS and the G5 proxy selector remain unchanged.

**Tech Stack:** C++17 in `DiligentSamples/Samples/RTXPT/src`, HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, Diligent structured buffers/SRVs/static raygen bindings, Dear ImGui for controls. There is no shader unit-test harness; verification is source checks, layout guards, build checks when requested, and manual D3D12/Vulkan visual checks when requested.

---

## Context You Need Before Starting

**Current baseline:** R0, R0.5, R1, R2, and R3.G5 have landed. The current direct-light path is:

- `RTXPTLights` uploads analytic lights into `StructuredBuffer<PolymorphicLightInfo> t_Lights`.
- `RTXPTLights::UploadLightProxyBuffer` builds a global G5 proxy CDF over analytic lights plus one emissive bucket.
- `PathTracer/Lighting/LightSampler.hlsli` samples the proxy table, evaluates analytic lights through `EvalAnalyticLight`, and uses RIS/WRS inside `PathTracer::SampleDirectLightNEE`.
- `EvalAnalyticLight` currently treats point/spot lights as point deltas with `1 / distance^2` and squared spot-cone falloff. Directional lights use a large distance sentinel and radiance `color * intensity`.
- `m_LightIntensityScale` exists as a UI/debug multiplier. After this phase it must default to `1.0` and no longer be needed for ordinary scenes to look correctly exposed.

**RTXPT-fork reference anchors** (read-only):

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/PolymorphicLight.hlsli:93-259` - sphere-light solid-angle sampling and power.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/PolymorphicLight.hlsli:267-328` - point-light flux convention.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/PolymorphicLight.hlsli:331-395` - directional finite angular-size sampling.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Lighting/LightShaping.hlsli` - smooth spot shaping and shaping flux factor.
- `D:/RTXPT-fork/Rtxpt/RTXDI/PrepareLightsPass.cpp:240-344` - CPU conversion from Donut point/spot/directional lights into RTXPT polymorphic-light units.

## Scope

This plan implements G6 only:

- Change the Diligent-native `PolymorphicLightInfo` layout, still 64 bytes, to carry unit-aware punctual-light data.
- Convert point and spot lights to finite sphere lights using a configurable/default small radius, matching the RTXPT finite-PDF path instead of pure point deltas.
- Convert directional lights to finite angular-size lights with radiance = irradiance / solidAngle.
- Add smooth spot shaping (`smoothstep(cosOuter, cosOuter + softness, cosTheta)`) and shaping flux factor for proxy weights.
- Replace hard range cutoff with glTF-compatible smooth range attenuation.
- Make analytic RIS proposal weights use the same photometric quantity as the shader evaluator.
- Keep the G5 proxy table, emissive-triangle branch, and environment NEE behavior intact.
- Keep `lightIntensityScale` as a debug multiplier with default `1.0`; it is not part of the physical unit conversion.

This plan does not implement:

- R4 HDR environment-map importance sampling.
- RTXPT-fork's packed RGB8/log-radiance light encoding.
- RTXPT-fork's full `LightsBaker`, local light grid, temporal feedback, IES profiles, or NEE-AT.
- A scene-authored light-radius UI. Scene JSON may provide `radius`; glTF/KHR punctual lights use the default finite radius.
- BSDF-side MIS for analytic sphere lights. The sphere lights introduced here are analytic proxies, not visible geometry, so `sampleableByBSDF` stays false.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp` - rename and document `PolymorphicLightInfo` fields while preserving 64-byte size.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - mirror the new light layout and define light-type constants.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/Geometry.hlsli` - shared geometry helpers used by light sampling.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightShaping.hlsli` - RTXPT-style spot shaping helpers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli` - replace point/directional evaluator with point/sphere/directional sampling and shaping.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli` - include analytic solid-angle pdf in the proposal pdf.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp` - convert CPU light data into the new units and compute proxy weights from analytic power.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - relabel the analytic-light multiplier as debug-only and keep it defaulted to one.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register new HLSL helper headers.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record the G6 mapping and Diligent-specific divergences.

## Cross-Cutting Contracts

- **Light layout:** `PolymorphicLightInfo` remains 64 bytes. It changes from `colorIntensity/positionRange/directionType/spotAngles` to `colorType/positionRadius/directionRange/shaping`. Update both C++ and HLSL static assertions together.
- **Type encoding:** Store RTXPT-fork type IDs as float values in `colorType.w`: sphere=0, triangle=1, directional=2, environment=3, point=4, environmentQuad=5. Only sphere, directional, and point appear in the analytic-light buffer in this phase.
- **Finite punctual lights:** glTF and scene-json point/spot lights use `max(authoredRadius, kDefaultPunctualLightRadius)` and therefore become sphere lights by default. A zero-radius point path remains in HLSL for safety but CPU upload should not generate it for ordinary point/spot lights.
- **Photometric conversion:** Sphere radiance is `color * intensity / (PI * radius^2)`, mirroring RTXPT's projected-area conversion. Point flux is `color * intensity`. Directional radiance is `color * intensity / solidAngle`.
- **Range attenuation:** The shader uses the glTF loader's recommended smooth range attenuation `saturate(1 - (distance / range)^4)` when `range > 0`.
- **Proposal pdf:** Analytic direct-light proposal pdf becomes `selectionPdf * light.solidAnglePdf`. Point lights have `solidAnglePdf = 1`; sphere and directional lights have finite PDFs.
- **Proxy weight:** CPU proxy weights must use analytic power/irradiance in the same units as shader sampling, not raw color intensity. This keeps G5 power sampling consistent after G6.
- **No payload growth:** All new light data is in `t_Lights` and raygen-local sampling state. `PathPayload` and `MaxPayloadSize` do not change.
- **Backends:** No backend-specific path is introduced. The light buffer remains a standard structured SRV for D3D12 and Vulkan.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples/Samples/RTXPT`

- [ ] **Step 1: Confirm working-tree state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Unrelated dirty files may exist. If any file listed in this plan is dirty, inspect it before editing and preserve user changes.

- [ ] **Step 2: Confirm G5 baseline files exist**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
rg -n "SampleDirectLightNEE|GenerateDirectLightCandidate|RTXPTLightProxy" DiligentSamples/Samples/RTXPT
```

Expected facts:

```text
LightSampler.hlsli exists.
PathTracer.hlsli contains SampleDirectLightNEE.
LightSampler.hlsli contains GenerateDirectLightCandidate.
RTXPTLights.hpp and PathTracerShared.h contain RTXPTLightProxy.
```

- [ ] **Step 3: Confirm current raw-light evaluator**

Run:

```powershell
rg -n "colorIntensity|positionRange|directionType|spotAngles|EvalAnalyticLight|Light intensity scale" DiligentSamples/Samples/RTXPT/src/RTXPTLights.* DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected facts:

```text
PolymorphicLightInfo uses colorIntensity, positionRange, directionType, and spotAngles.
PolymorphicLight.hlsli contains EvalAnalyticLight.
RTXPTSample.cpp exposes the "Light intensity scale" slider.
```

- [ ] **Step 4: Commit preflight snapshot only if requested**

No commit is required for a clean preflight. If the user requests a pre-change checkpoint, run:

```powershell
git status --short
```

Expected: no files from this plan have been changed yet.

---

### Task 1: Update Shared Punctual-Light Layout

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`

- [ ] **Step 1: Replace the C++ `PolymorphicLightInfo` layout**

In `RTXPTLights.hpp`, replace `struct PolymorphicLightInfo` with:

```cpp
struct PolymorphicLightInfo
{
    // rgb: radiance for sphere/directional lights, flux for zero-radius point lights.
    // w: RTXPT-fork PolymorphicLightType value stored as a float.
    float4 colorType = float4{0, 0, 0, -1};

    // xyz: world-space center/position. w: finite sphere radius.
    float4 positionRadius = float4{0, 0, 0, 0};

    // xyz: normalized primary direction for spot/directional lights. w: point/spot range; 0 means infinite.
    float4 directionRange = float4{0, -1, 0, 0};

    // x: cos outer cone angle; y: cone softness in cosine space; z: minimum falloff; w: directional solid angle.
    float4 shaping = float4{-1, 0, 0, 0};
};
static_assert(sizeof(PolymorphicLightInfo) == 64, "PolymorphicLightInfo layout must match PathTracer/PathTracerShared.h");
```

- [ ] **Step 2: Replace the HLSL `PolymorphicLightInfo` layout and add type constants**

In `PathTracerShared.h`, replace the existing `PolymorphicLightInfo` block with:

```hlsl
static const uint kPolymorphicLightTypeSphere         = 0u;
static const uint kPolymorphicLightTypeTriangle       = 1u;
static const uint kPolymorphicLightTypeDirectional    = 2u;
static const uint kPolymorphicLightTypeEnvironment    = 3u;
static const uint kPolymorphicLightTypePoint          = 4u;
static const uint kPolymorphicLightTypeEnvironmentQuad = 5u;

// Mirrors Diligent::PolymorphicLightInfo in RTXPTLights.hpp.
struct PolymorphicLightInfo
{
    // rgb: radiance for sphere/directional lights, flux for zero-radius point lights.
    // w: RTXPT-fork PolymorphicLightType value stored as a float.
    float4 colorType;

    // xyz: world-space center/position. w: finite sphere radius.
    float4 positionRadius;

    // xyz: normalized primary direction for spot/directional lights. w: point/spot range; 0 means infinite.
    float4 directionRange;

    // x: cos outer cone angle; y: cone softness in cosine space; z: minimum falloff; w: directional solid angle.
    float4 shaping;
};
```

- [ ] **Step 3: Run layout source checks**

Run:

```powershell
rg -n "colorIntensity|positionRange|directionType|spotAngles" DiligentSamples/Samples/RTXPT/src/RTXPTLights.* DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
rg -n "static_assert\\(sizeof\\(PolymorphicLightInfo\\) == 64" DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp
```

Expected:

```text
The old field-name search returns no matches in RTXPTLights.* or PathTracerShared.h.
The static_assert still reports PolymorphicLightInfo == 64.
```

- [ ] **Step 4: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h
git commit -m "refactor(rtxpt): update punctual light data layout"
```

---

### Task 2: Add Geometry And Light-Shaping Helpers

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/Geometry.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightShaping.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create `Utils/Geometry.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/Geometry.hlsli` with:

```hlsl
#ifndef __GEOMETRY_HLSLI__
#define __GEOMETRY_HLSLI__

#include "SampleGenerators.hlsli"

float sq(float v)
{
    return v * v;
}

float2 SampleDiskUniform(float2 rand)
{
    const float angle = 6.28318530717958647692 * rand.x;
    return float2(cos(angle), sin(angle)) * sqrt(rand.y);
}

float3 sphericalDirection(float sinTheta, float cosTheta, float sinPhi, float cosPhi,
                          float3 x, float3 y, float3 z)
{
    return sinTheta * cosPhi * x + sinTheta * sinPhi * y + cosTheta * z;
}

// Uniformly sampled barycentric coordinates inside a triangle.
float3 SampleTriangleUniform(float2 rnd)
{
    const float sqrtx = sqrt(rnd.x);
    return float3(1.0 - sqrtx, sqrtx * (1.0 - rnd.y), sqrtx * rnd.y);
}

// Area-measure to solid-angle-measure pdf conversion.
float pdfAtoW(float pdfA, float distance, float cosTheta)
{
    return pdfA * (distance * distance) / max(cosTheta, 2e-9);
}

#endif // __GEOMETRY_HLSLI__
```

- [ ] **Step 2: Create `Lighting/LightShaping.hlsli`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightShaping.hlsli` with:

```hlsl
#ifndef __LIGHT_SHAPING_HLSLI__
#define __LIGHT_SHAPING_HLSLI__

static const float kMinSpotlightFalloff = 0.0001;

struct LightShaping
{
    float cosConeAngle;
    float3 primaryAxis;
    float cosConeSoftness;
    uint  isSpot;
    float minFalloff;
};

LightShaping LightShaping_make_none()
{
    LightShaping shaping;
    shaping.cosConeAngle    = -1.0;
    shaping.primaryAxis     = float3(0.0, -1.0, 0.0);
    shaping.cosConeSoftness = 0.0;
    shaping.isSpot          = 0u;
    shaping.minFalloff      = 0.0;
    return shaping;
}

float evaluateLightShaping(LightShaping shaping, float3 surfacePosition, float3 lightSamplePosition)
{
    if (shaping.isSpot == 0u)
        return 1.0;

    const float3 lightToSurface = normalize(surfacePosition - lightSamplePosition);
    const float  cosTheta       = dot(shaping.primaryAxis, lightToSurface);
    const float  smoothFalloff  = smoothstep(shaping.cosConeAngle,
                                             shaping.cosConeAngle + shaping.cosConeSoftness,
                                             cosTheta);
    return max(shaping.minFalloff, smoothFalloff);
}

float getShapingFluxFactor(LightShaping shaping)
{
    if (shaping.isSpot == 0u)
        return 1.0;

    float solidAngleOverTwoPi = 1.0 - shaping.cosConeAngle;
    solidAngleOverTwoPi *= lerp(1.0, 0.5, shaping.cosConeSoftness);
    return solidAngleOverTwoPi * 0.5;
}

#endif // __LIGHT_SHAPING_HLSLI__
```

- [ ] **Step 3: Register helper headers in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new headers in the `SHADERS` list next to the existing PathTracer helpers:

```cmake
    assets/shaders/PathTracer/Lighting/LightShaping.hlsli
    assets/shaders/PathTracer/Utils/Geometry.hlsli
```

Place `LightShaping.hlsli` near `Lighting/PolymorphicLight.hlsli` and `Geometry.hlsli` near `Utils/SampleGenerators.hlsli`.

- [ ] **Step 4: Run helper source checks**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/Geometry.hlsli
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightShaping.hlsli
rg -n "LightShaping.hlsli|Geometry.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected:

```text
Both Test-Path commands print True.
CMakeLists.txt lists both new headers exactly once.
```

- [ ] **Step 5: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Utils/Geometry.hlsli DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightShaping.hlsli DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add light shaping helpers"
```

---

### Task 3: Convert CPU Lights Into Photometric Units

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`

- [ ] **Step 1: Add constants and helpers in the anonymous namespace**

In `RTXPTLights.cpp`, after `ReadRTXPTLightIntensity`, add:

```cpp
constexpr float kPolymorphicLightTypeSphere      = 0.0f;
constexpr float kPolymorphicLightTypeDirectional = 2.0f;
constexpr float kPolymorphicLightTypePoint       = 4.0f;

constexpr float kDefaultPunctualLightRadius      = 0.01f;
constexpr float kDefaultDirectionalAngularSize   = 0.53f; // Degrees, roughly the solar angular diameter.
constexpr float kMinimumPunctualLightRadius      = 1.0e-4f;
constexpr float kMinimumDirectionalSolidAngle    = 1.0e-8f;
constexpr float kMinimumProxyWeight              = 1.0e-6f;

float Max3(float X, float Y, float Z)
{
    return std::max(X, std::max(Y, Z));
}

float Luminance(const float3& Color)
{
    return Color.x * 0.2126f + Color.y * 0.7152f + Color.z * 0.0722f;
}

float ClampRadius(float Radius)
{
    return std::max(Radius, kMinimumPunctualLightRadius);
}

float SafeOuterCone(float OuterCone)
{
    return std::clamp(std::abs(OuterCone), 0.001f, PI_F);
}

float ConeSoftness(float InnerCone, float OuterCone)
{
    const float Outer = SafeOuterCone(OuterCone);
    const float Inner = std::clamp(std::abs(InnerCone), 0.0f, Outer);
    return std::clamp(1.0f - Inner / Outer, 0.0f, 1.0f);
}

float ShapingFluxFactor(float CosConeAngle, float CosConeSoftness)
{
    float SolidAngleOverTwoPi = 1.0f - CosConeAngle;
    SolidAngleOverTwoPi *= (1.0f - 0.5f * CosConeSoftness);
    return SolidAngleOverTwoPi * 0.5f;
}

float ReadRTXPTLightRadius(const nlohmann::json& Json)
{
    return ClampRadius(ReadRTXPTOptionalFloat(Json, "radius", kDefaultPunctualLightRadius));
}

float ReadRTXPTDirectionalAngularSizeRadians(const nlohmann::json& Json)
{
    const float Degrees = ReadRTXPTOptionalFloat(Json, "angularSize", kDefaultDirectionalAngularSize);
    return DegreesToRadians(std::clamp(Degrees, 0.001f, 90.0f));
}

float DirectionalSolidAngle(float AngularSizeRadians)
{
    const float HalfAngle = std::max(AngularSizeRadians * 0.5f, 0.00001f);
    return std::max(2.0f * PI_F * (1.0f - std::cos(HalfAngle)), kMinimumDirectionalSolidAngle);
}

float3 NormalizeDirection(const float3& Direction, const float3& Fallback)
{
    const float LenSq = Direction.x * Direction.x + Direction.y * Direction.y + Direction.z * Direction.z;
    if (LenSq <= 1.0e-12f)
        return Fallback;

    const float InvLen = 1.0f / std::sqrt(LenSq);
    return float3{Direction.x * InvLen, Direction.y * InvLen, Direction.z * InvLen};
}

PolymorphicLightInfo MakeSphereLightData(const float3& Color, float Intensity, const float4x4& Transform,
                                         float Range, float Radius, float InnerCone, float OuterCone, bool IsSpot)
{
    const float  ClampedRadius = ClampRadius(Radius);
    const float3 Flux          = Color * std::max(Intensity, 0.0f);
    const float  ProjectedArea = PI_F * ClampedRadius * ClampedRadius;
    const float3 Radiance      = Flux / std::max(ProjectedArea, 1.0e-8f);
    const float3 Direction     = NormalizeDirection(float3{-Transform._31, -Transform._32, -Transform._33}, float3{0.0f, -1.0f, 0.0f});

    PolymorphicLightInfo Data;
    Data.colorType      = float4{Radiance.x, Radiance.y, Radiance.z, kPolymorphicLightTypeSphere};
    Data.positionRadius = float4{Transform._41, Transform._42, Transform._43, ClampedRadius};
    Data.directionRange = float4{Direction.x, Direction.y, Direction.z, std::max(Range, 0.0f)};
    Data.shaping        = float4{-1.0f, 0.0f, 0.0f, 0.0f};

    if (IsSpot)
    {
        const float Outer    = SafeOuterCone(OuterCone);
        const float Softness = ConeSoftness(InnerCone, OuterCone);
        Data.shaping.x      = std::cos(Outer);
        Data.shaping.y      = Softness;
        Data.shaping.z      = OuterCone < 0.0f ? 0.0001f : 0.0f;
    }

    return Data;
}

PolymorphicLightInfo MakeDirectionalLightData(const float3& Color, float Intensity, const float4x4& Transform,
                                              float AngularSizeRadians)
{
    const float3 Direction  = NormalizeDirection(float3{-Transform._31, -Transform._32, -Transform._33}, float3{0.0f, -1.0f, 0.0f});
    const float  SolidAngle = DirectionalSolidAngle(AngularSizeRadians);
    const float3 Radiance   = Color * std::max(Intensity, 0.0f) / SolidAngle;

    PolymorphicLightInfo Data;
    Data.colorType      = float4{Radiance.x, Radiance.y, Radiance.z, kPolymorphicLightTypeDirectional};
    Data.positionRadius = float4{0.0f, 0.0f, 0.0f, 0.0f};
    Data.directionRange = float4{Direction.x, Direction.y, Direction.z, 0.0f};
    Data.shaping        = float4{-1.0f, 0.0f, 0.0f, SolidAngle};
    return Data;
}

float ComputeAnalyticLightProxyWeight(const PolymorphicLightInfo& Light)
{
    const float3 Color{Light.colorType.x, Light.colorType.y, Light.colorType.z};
    const float  Type = Light.colorType.w;

    if (std::abs(Type - kPolymorphicLightTypeSphere) < 0.5f)
    {
        const float Radius     = ClampRadius(Light.positionRadius.w);
        const float Area       = 4.0f * PI_F * Radius * Radius;
        const float ShapeFlux  = Light.shaping.x > -0.999f ? ShapingFluxFactor(Light.shaping.x, Light.shaping.y) : 1.0f;
        return std::max(kMinimumProxyWeight, Area * PI_F * Luminance(Color) * ShapeFlux);
    }

    if (std::abs(Type - kPolymorphicLightTypePoint) < 0.5f)
        return std::max(kMinimumProxyWeight, 4.0f * PI_F * Luminance(Color));

    if (std::abs(Type - kPolymorphicLightTypeDirectional) < 0.5f)
        return std::max(kMinimumProxyWeight, Max3(Color.x, Color.y, Color.z) * std::max(Light.shaping.w, kMinimumDirectionalSolidAngle));

    return kMinimumProxyWeight;
}
```

- [ ] **Step 2: Replace `MakeLightData`**

Replace the current `MakeLightData(const GLTF::Light& Light, const float4x4& NodeTransform)` with:

```cpp
PolymorphicLightInfo MakeLightData(const GLTF::Light& Light, const float4x4& NodeTransform)
{
    const float3 Color{Light.Color.x, Light.Color.y, Light.Color.z};
    switch (Light.Type)
    {
        case GLTF::Light::TYPE::DIRECTIONAL:
            return MakeDirectionalLightData(Color, Light.Intensity, NodeTransform,
                                            DegreesToRadians(kDefaultDirectionalAngularSize));

        case GLTF::Light::TYPE::POINT:
            return MakeSphereLightData(Color, Light.Intensity, NodeTransform, Light.Range,
                                       kDefaultPunctualLightRadius, 0.0f, PI_F, false);

        case GLTF::Light::TYPE::SPOT:
            return MakeSphereLightData(Color, Light.Intensity, NodeTransform, Light.Range,
                                       kDefaultPunctualLightRadius, Light.InnerConeAngle,
                                       Light.OuterConeAngle, true);

        default:
        {
            PolymorphicLightInfo Disabled;
            Disabled.colorType.w = -1.0f;
            return Disabled;
        }
    }
}
```

- [ ] **Step 3: Replace scene-json light construction**

Inside `RTXPTLights::Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData)`, replace the manual `PolymorphicLightInfo Light;` field assignment block with:

```cpp
        const float3 Color3{Color[0], Color[1], Color[2]};
        const float  Intensity = ReadRTXPTLightIntensity(LightMeta.RawJson);

        PolymorphicLightInfo Light;
        if (LightMeta.Type == "DirectionalLight")
        {
            Light = MakeDirectionalLightData(Color3, Intensity, LightMeta.GlobalTransform,
                                             ReadRTXPTDirectionalAngularSizeRadians(LightMeta.RawJson));
        }
        else if (LightMeta.Type == "PointLight")
        {
            Light = MakeSphereLightData(Color3, Intensity, LightMeta.GlobalTransform,
                                        ReadRTXPTOptionalFloat(LightMeta.RawJson, "range", 0.0f),
                                        ReadRTXPTLightRadius(LightMeta.RawJson),
                                        0.0f,
                                        PI_F,
                                        false);
        }
        else if (LightMeta.Type == "SpotLight")
        {
            Light = MakeSphereLightData(Color3, Intensity, LightMeta.GlobalTransform,
                                        ReadRTXPTOptionalFloat(LightMeta.RawJson, "range", 0.0f),
                                        ReadRTXPTLightRadius(LightMeta.RawJson),
                                        ReadRTXPTSpotAngleRadians(LightMeta.RawJson, "innerAngle", "innerConeAngle"),
                                        ReadRTXPTSpotAngleRadians(LightMeta.RawJson, "outerAngle", "outerConeAngle", 45.0f),
                                        true);
        }
        else
        {
            continue;
        }
```

Keep the existing `float Color[3]` read just above this replacement.

- [ ] **Step 4: Replace analytic proxy weighting**

In `UploadLightProxyBuffer`, replace the analytic-light weight calculation:

```cpp
        const float3 Radiance = float3{Light.colorIntensity.x, Light.colorIntensity.y, Light.colorIntensity.z} *
            std::max(Light.colorIntensity.w, 0.0f);
        const float Weight = std::max(1e-6f, MaxRGB(Radiance));
```

with:

```cpp
        const float Weight = ComputeAnalyticLightProxyWeight(Light);
```

- [ ] **Step 5: Replace dummy light initialization**

In `UploadLightBuffer`, replace the dummy-light fields with:

```cpp
        PolymorphicLightInfo Default;
        Default.colorType = float4{0, 0, 0, -1.0f};
        Lights.emplace_back(Default);
```

- [ ] **Step 6: Remove stale helper only if unused**

Run:

```powershell
rg -n "MaxRGB\\(" DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp
```

Expected: if `MaxRGB` has no callers after Step 4, delete the `MaxRGB` helper. Keep `GetMaterialEmissionMagnitude` by changing it to use `Max3`:

```cpp
return Max3(pExtension->EmissiveFactor.x, pExtension->EmissiveFactor.y, pExtension->EmissiveFactor.z);
```

and:

```cpp
return Max3(Material.Attribs.EmissiveFactor.x, Material.Attribs.EmissiveFactor.y, Material.Attribs.EmissiveFactor.z);
```

- [ ] **Step 7: Run CPU source checks**

Run:

```powershell
rg -n "colorIntensity|positionRange|directionType|spotAngles" DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp
rg -n "MakeSphereLightData|MakeDirectionalLightData|ComputeAnalyticLightProxyWeight|kDefaultPunctualLightRadius" DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp
```

Expected:

```text
The old field-name search returns no matches in RTXPTLights.cpp.
The new helper search reports all four helper names.
```

- [ ] **Step 8: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp
git commit -m "feat(rtxpt): convert punctual lights to photometric units"
```

---

### Task 4: Replace Analytic-Light HLSL Sampling

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli`

- [ ] **Step 1: Replace `PolymorphicLight.hlsli`**

Replace the entire file with:

```hlsl
#ifndef __POLYMORPHIC_LIGHT_HLSLI__
#define __POLYMORPHIC_LIGHT_HLSLI__

#include "../PathTracerShared.h"
#include "../Utils/Geometry.hlsli"
#include "LightShaping.hlsli"

static const float kDistantLightDistance = 100000.0;
static const float kLightPdfEpsilon      = 2e-9;

struct LightSample
{
    float3 dir;
    float  distance;
    float3 radiance;
    float  solidAnglePdf;
    bool   valid;
    bool   sampleableByBSDF;
};

LightSample LightSample_make_empty()
{
    LightSample ls;
    ls.dir              = float3(0.0, 1.0, 0.0);
    ls.distance         = 0.0;
    ls.radiance         = float3(0.0, 0.0, 0.0);
    ls.solidAnglePdf    = 0.0;
    ls.valid            = false;
    ls.sampleableByBSDF = false;
    return ls;
}

bool tryNormalize(float3 v, out float3 dir)
{
    const float lenSq = dot(v, v);
    if (lenSq <= 1e-12)
    {
        dir = float3(0.0, 0.0, 0.0);
        return false;
    }

    dir = v * rsqrt(lenSq);
    return true;
}

uint DecodeLightType(PolymorphicLightInfo light)
{
    return (uint)(light.colorType.w + 0.5);
}

LightShaping UnpackLightShaping(PolymorphicLightInfo light)
{
    LightShaping shaping = LightShaping_make_none();
    if (light.shaping.x <= -0.999)
        return shaping;

    float3 primaryAxis;
    if (!tryNormalize(light.directionRange.xyz, primaryAxis))
        return shaping;

    shaping.cosConeAngle    = light.shaping.x;
    shaping.primaryAxis     = primaryAxis;
    shaping.cosConeSoftness = max(light.shaping.y, 0.0);
    shaping.isSpot          = 1u;
    shaping.minFalloff      = max(light.shaping.z, 0.0);
    return shaping;
}

float EvaluateRangeAttenuation(float distanceToCenter, float range)
{
    if (range <= 0.0)
        return 1.0;
    if (distanceToCenter >= range)
        return 0.0;

    const float d  = saturate(distanceToCenter / range);
    const float d2 = d * d;
    return saturate(1.0 - d2 * d2);
}

float3 ApplyCommonPunctualTerms(PolymorphicLightInfo light, float3 surfacePos,
                                float3 lightSamplePosition, float3 radiance)
{
    const float distanceToCenter = length(light.positionRadius.xyz - surfacePos);
    const float rangeAttenuation = EvaluateRangeAttenuation(distanceToCenter, light.directionRange.w);
    if (rangeAttenuation <= 0.0)
        return float3(0.0, 0.0, 0.0);

    const float shaping = evaluateLightShaping(UnpackLightShaping(light), surfacePos, lightSamplePosition);
    return radiance * (rangeAttenuation * shaping);
}

LightSample CalcPointLightSample(PolymorphicLightInfo light, float3 surfacePos)
{
    const float3 toLight = light.positionRadius.xyz - surfacePos;
    const float  distSq  = dot(toLight, toLight);
    if (distSq <= 1e-8)
        return LightSample_make_empty();

    LightSample ls;
    ls.dir              = toLight * rsqrt(distSq);
    ls.distance         = sqrt(distSq);
    ls.radiance         = ApplyCommonPunctualTerms(light, surfacePos, light.positionRadius.xyz,
                                                   max(light.colorType.rgb, float3(0.0, 0.0, 0.0)) / distSq);
    ls.solidAnglePdf    = 1.0;
    ls.valid            = max(ls.radiance.x, max(ls.radiance.y, ls.radiance.z)) > 0.0;
    ls.sampleableByBSDF = false;
    return ls;
}

LightSample CalcSphereLightSample(PolymorphicLightInfo light, float2 random, float3 surfacePos)
{
    const float3 lightVector    = light.positionRadius.xyz - surfacePos;
    const float  lightDistance2 = dot(lightVector, lightVector);
    const float  radius         = max(light.positionRadius.w, 1e-4);
    const float  radius2        = radius * radius;
    if (lightDistance2 <= radius2)
        return LightSample_make_empty();

    const float lightDistance = sqrt(lightDistance2);
    const float sinThetaMax2  = radius2 / lightDistance2;
    const float cosThetaMax   = sqrt(max(0.0, 1.0 - sinThetaMax2));
    const float phi           = 6.28318530717958647692 * random.x;
    const float cosTheta      = lerp(cosThetaMax, 1.0, random.y);
    const float sinTheta      = sqrt(max(0.0, 1.0 - cosTheta * cosTheta));
    const float sinTheta2     = sinTheta * sinTheta;

    const float ds       = lightDistance * cosTheta - sqrt(max(1e-10, radius2 - lightDistance2 * sinTheta2));
    const float cosAlpha = (lightDistance2 + radius2 - ds * ds) / (2.0 * lightDistance * radius);
    const float sinAlpha = sqrt(max(0.0, 1.0 - cosAlpha * cosAlpha));

    const float3 sampleSpaceNormal = normalize(lightVector);
    float3       sampleSpaceTangent;
    float3       sampleSpaceBitangent;
    BranchlessONB(sampleSpaceNormal, sampleSpaceTangent, sampleSpaceBitangent);

    float sinPhi;
    float cosPhi;
    sincos(phi, sinPhi, cosPhi);

    const float3 radiusVector = sphericalDirection(sinAlpha, cosAlpha, sinPhi, cosPhi,
                                                   -sampleSpaceTangent, -sampleSpaceBitangent, -sampleSpaceNormal);
    const float3 samplePos    = light.positionRadius.xyz + radius * radiusVector;
    const float3 toSample     = samplePos - surfacePos;
    const float  sampleDist2  = dot(toSample, toSample);
    if (sampleDist2 <= 1e-8)
        return LightSample_make_empty();

    const float solidAnglePdf = 1.0 / max(2.0 * 3.14159265358979323846 * (1.0 - cosThetaMax), kLightPdfEpsilon);

    LightSample ls;
    ls.dir              = toSample * rsqrt(sampleDist2);
    ls.distance         = sqrt(sampleDist2);
    ls.radiance         = ApplyCommonPunctualTerms(light, surfacePos, samplePos,
                                                   max(light.colorType.rgb, float3(0.0, 0.0, 0.0)));
    ls.solidAnglePdf    = solidAnglePdf;
    ls.valid            = max(ls.radiance.x, max(ls.radiance.y, ls.radiance.z)) > 0.0;
    ls.sampleableByBSDF = false;
    return ls;
}

LightSample CalcDirectionalLightSample(PolymorphicLightInfo light, float2 random, float3 surfacePos)
{
    float3 direction;
    if (!tryNormalize(light.directionRange.xyz, direction))
        return LightSample_make_empty();

    const float solidAngle   = max(light.shaping.w, kLightPdfEpsilon);
    const float cosHalfAngle = saturate(1.0 - solidAngle / (2.0 * 3.14159265358979323846));
    const float sinHalfAngle = sqrt(max(0.0, 1.0 - cosHalfAngle * cosHalfAngle));
    const float2 diskSample  = SampleDiskUniform(random);

    float3 tangent;
    float3 bitangent;
    BranchlessONB(direction, tangent, bitangent);

    const float3 distantDirectionSample = normalize(direction +
                                                    tangent * diskSample.x * sinHalfAngle +
                                                    bitangent * diskSample.y * sinHalfAngle);
    const float3 samplePos = surfacePos - distantDirectionSample * kDistantLightDistance;

    LightSample ls;
    ls.dir              = normalize(samplePos - surfacePos);
    ls.distance         = kDistantLightDistance;
    ls.radiance         = max(light.colorType.rgb, float3(0.0, 0.0, 0.0));
    ls.solidAnglePdf    = 1.0 / solidAngle;
    ls.valid            = max(ls.radiance.x, max(ls.radiance.y, ls.radiance.z)) > 0.0;
    ls.sampleableByBSDF = false;
    return ls;
}

LightSample SampleAnalyticLight(PolymorphicLightInfo light, float2 random, float3 surfacePos)
{
    const uint type = DecodeLightType(light);
    if (type == kPolymorphicLightTypeSphere)
        return CalcSphereLightSample(light, random, surfacePos);
    if (type == kPolymorphicLightTypePoint)
        return CalcPointLightSample(light, surfacePos);
    if (type == kPolymorphicLightTypeDirectional)
        return CalcDirectionalLightSample(light, random, surfacePos);

    return LightSample_make_empty();
}

#endif // __POLYMORPHIC_LIGHT_HLSLI__
```

- [ ] **Step 2: Run HLSL source checks**

Run:

```powershell
rg -n "EvalAnalyticLight|colorIntensity|positionRange|directionType|spotAngles" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli
rg -n "SampleAnalyticLight|CalcSphereLightSample|CalcDirectionalLightSample|EvaluateRangeAttenuation|solidAnglePdf" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli
```

Expected:

```text
The old symbol/field search returns no matches.
The new helper search reports all listed names.
```

- [ ] **Step 3: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/PolymorphicLight.hlsli
git commit -m "feat(rtxpt): sample shaped punctual lights"
```

---

### Task 5: Include Analytic Solid-Angle PDFs In Direct-Light RIS

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli`

- [ ] **Step 1: Replace the analytic branch in `GenerateDirectLightCandidate`**

In `LightSampler.hlsli`, replace only the `if (proxy.kind == kLightProxyKindAnalytic)` branch with:

```hlsl
    if (proxy.kind == kLightProxyKindAnalytic)
    {
        const LightSample light = SampleAnalyticLight(Bridge::getLight(proxy.index), sampleNext2D(sg), hitPos);
        if (!light.valid || light.solidAnglePdf <= 0.0)
            return sample;

        const float lightPdf = selectionPdf * light.solidAnglePdf;
        if (lightPdf <= 0.0)
            return sample;

        const float3 radiance = light.radiance * max(g_Const.ptConsts.lightIntensityScale, 0.0);
        const float  bsdfProb = getSpecularProbability(bsdfData, wo);
        float3       f;
        float        bsdfPdf;
        EvalBSDF(bsdfData, wo, light.dir, bsdfProb, f, bsdfPdf);

        sample.dir              = light.dir;
        sample.distance         = light.distance;
        sample.radianceOverPdf  = radiance / lightPdf;
        sample.proposalPdf      = lightPdf;
        sample.bsdfF            = f;
        sample.bsdfPdf          = bsdfPdf;
        sample.kind             = proxy.kind;
        sample.index            = proxy.index;
        sample.valid            = true;
        sample.sampleableByBSDF = light.sampleableByBSDF;
    }
```

- [ ] **Step 2: Keep emissive branch untouched**

Run:

```powershell
rg -n "kLightProxyKindEmissiveBucket|trianglePdf|pdfAtoW|tri.radiance" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
```

Expected: the emissive branch still computes `trianglePdf = selectionPdf * (1.0 / float(triCount)) * solidAnglePdf`.

- [ ] **Step 3: Run analytic-pdf checks**

Run:

```powershell
rg -n "SampleAnalyticLight|light\\.solidAnglePdf|selectionPdf \\* light\\.solidAnglePdf|radianceOverPdf" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
```

Expected: all four patterns are present.

- [ ] **Step 4: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Lighting/LightSampler.hlsli
git commit -m "fix(rtxpt): account for analytic light pdfs in RIS"
```

---

### Task 6: Make The Analytic Light Scale Debug-Only

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`

- [ ] **Step 1: Confirm the default scale is one**

Run:

```powershell
rg -n "m_LightIntensityScale\\s*=\\s*1\\.0f" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```

Expected: `m_LightIntensityScale` defaults to `1.0f`.

- [ ] **Step 2: Relabel the UI slider**

In `RTXPTSample.cpp`, replace:

```cpp
            ResetOnChange(ImGui::SliderFloat("Light intensity scale", &m_LightIntensityScale, 0.0f, 10.0f), "Light intensity changed");
```

with:

```cpp
            ResetOnChange(ImGui::SliderFloat("Analytic light debug scale", &m_LightIntensityScale, 0.0f, 4.0f, "%.2f"),
                          "Analytic light debug scale changed");
```

- [ ] **Step 3: Run UI source checks**

Run:

```powershell
rg -n "Light intensity scale|Analytic light debug scale|m_LightIntensityScale" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
```

Expected:

```text
"Light intensity scale" is absent.
"Analytic light debug scale" is present in RTXPTSample.cpp.
m_LightIntensityScale still defaults to 1.0f in RTXPTSample.hpp.
```

- [ ] **Step 4: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "chore(rtxpt): mark analytic light scale as debug control"
```

---

### Task 7: Update RTXPT Fork Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Update the lighting-layer table**

In section `T-D. Lighting Layer`, add these rows after the existing `RTXPTEvalAnalyticLight` mapping row:

```markdown
| `SampleAnalyticLight(light, random, surfacePos)` | `PolymorphicLight::CalcSample(...)` subset | R3/G6 ports point/sphere/directional sampling math over the port's unpacked `PolymorphicLightInfo` layout |
| `LightShaping.hlsli` | `Lighting/LightShaping.hlsli` subset | Smooth spot shaping and shaping flux factor; IES profiles remain absent |
| `Utils/Geometry.hlsli` | `Utils/Geometry.hlsli` subset | Shared disk/triangle/pdf helpers used by analytic and emissive lights |
```

- [ ] **Step 2: Update the shared-struct field table**

In section `T-H. Shared Struct Field Names`, replace the four old `PolymorphicLightInfo` rows with:

```markdown
| `PolymorphicLightInfo` | `colorType` | rgb stores radiance/flux; w stores RTXPT-fork type id as float |
| `PolymorphicLightInfo` | `positionRadius` | xyz center/position; w finite sphere radius |
| `PolymorphicLightInfo` | `directionRange` | xyz normalized direction; w glTF range |
| `PolymorphicLightInfo` | `shaping` | x cos outer angle; y softness; z minimum falloff; w directional solid angle |
```

- [ ] **Step 3: Add the G6 divergence note**

In the `Divergences` section, add:

```markdown
- R3/G6 keeps a 64-byte unpacked `PolymorphicLightInfo` instead of RTXPT-fork's packed
  RGB8/log-radiance `PolymorphicLightInfoFull`. The sampling units match RTXPT:
  point lights use flux, finite punctual lights use sphere radiance divided by projected
  area, and directional lights use irradiance divided by finite solid angle. The port also
  keeps glTF range attenuation because Diligent's GLTF loader exposes `Light::Range`.
```

- [ ] **Step 4: Run mapping checks**

Run:

```powershell
rg -n "SampleAnalyticLight|LightShaping.hlsli|positionRadius|directionRange|R3/G6 keeps" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: each pattern is present.

- [ ] **Step 5: Commit**

```powershell
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map shaped punctual light port"
```

---

### Task 8: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`
- Verify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Run stale-symbol scan**

Run:

```powershell
rg -n "colorIntensity|positionRange|directionType|spotAngles|EvalAnalyticLight|Light intensity scale" DiligentSamples/Samples/RTXPT/src/RTXPTLights.* DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected:

```text
No matches.
```

- [ ] **Step 2: Run new-symbol scan**

Run:

```powershell
rg -n "colorType|positionRadius|directionRange|shaping|SampleAnalyticLight|CalcSphereLightSample|solidAnglePdf|Analytic light debug scale" DiligentSamples/Samples/RTXPT/src/RTXPTLights.* DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: all new symbols are found in the expected C++ and HLSL files.

- [ ] **Step 3: Run CMake registration scan**

Run:

```powershell
rg -n "LightShaping.hlsli|Geometry.hlsli|PolymorphicLight.hlsli|LightSampler.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: all four shader headers are present in the `SHADERS` list.

- [ ] **Step 4: Run build verification when the user has approved build commands**

If a configured build tree exists and the user has approved build verification, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected:

```text
The RTXPT target builds successfully. Any HLSL compile error mentioning PolymorphicLightInfo, SampleAnalyticLight, or solidAnglePdf must be fixed before continuing.
```

If the build tree does not exist, run only this source check and report that build verification was skipped:

```powershell
Test-Path build\x64\Debug
```

Expected when no build tree exists:

```text
False
```

- [ ] **Step 5: Run manual GPU smoke when the user has approved runtime checks**

Run the RTXPT sample on D3D12 and Vulkan using the existing local workflow for this repository. In each backend:

```text
1. Load a scene with a point light and confirm it renders at debug scale 1.0.
2. Load or author a scene-json spot light with innerAngle < outerAngle and confirm the cone edge is smooth.
3. Set a finite range and confirm illumination fades before the range boundary rather than snapping at the boundary.
4. Switch NEE sampling between Uniform and Power+ and confirm brightness is stable while noise changes.
```

Expected: no manual intensity boost is needed for ordinary punctual-light exposure; spot cones and finite ranges behave smoothly; D3D12 and Vulkan render valid images.

- [ ] **Step 6: Final status check**

Run:

```powershell
git status --short
```

Expected: only intentional uncommitted files remain. If all task commits were made, the working tree is clean except for unrelated user changes that predated this plan.
