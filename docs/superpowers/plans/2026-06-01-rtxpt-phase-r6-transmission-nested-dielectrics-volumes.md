# RTXPT Phase R6 Transmission, Nested Dielectrics, and Volumes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the DiligentEngine RTXPT reference path tracer to RTXPT-fork reference-mode parity for G10: rough dielectric reflection/transmission, nested dielectric priority handling, homogeneous volume absorption, and stochastic alpha-blend visibility.

**Architecture:** Keep the Diligent raygen-driven, `MaxRecursionDepth = 1` reference path tracer. Grow the material and hit-payload contracts so closest-hit can return two-sided material state, port RTXPT-fork's BSDF transmission math into `Rendering/Materials/BxDF.hlsli`, and add a raygen-local `InteriorList` that tracks dielectric media across path segments. Volume absorption is a throughput multiplier applied before shading each hit, matching RTXPT-fork's reference-mode `HandleHit` flow.

**Tech Stack:** HLSL 6.5 ray tracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`, C++17 sample code under `DiligentSamples/Samples/RTXPT/src`, Diligent ray-tracing PSO creation, DiligentTools GLTF material attributes, Dear ImGui, CMake shader source registration. `DiligentSamples` and `DiligentTools` are git submodules; implementation commits in this plan are made inside the submodules whose files are touched.

---

## Context You Need Before Starting

Phase R0.5 and R5 have landed, so use the current RTXPT-fork-aligned paths:

| Spec name | Current path |
|---|---|
| `RTXPTReference.rgen` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` |
| `RTXPTReference.rchit` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` |
| `RTXPTReference.rahit` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit` |
| `RTXPTBSDF.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` |
| `RTXPTShaderShared.hlsli` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` |
| C++ material mirror | `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` |
| C++ frame constants mirror | `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` |

Current baseline:

- `MaterialPTData` is 96 bytes and only carries base color, emission, metallic/roughness, normal, and emissive/alpha-test flags.
- `PathPayload` is 80 bytes and only carries a single shading normal, opaque material data, emission, and emissive-light MIS pdf.
- `PathTracerClosestHit.rchit` flips the shading normal toward the incoming ray and explicitly behaves as single-sided opaque shading.
- `PathTracerAnyHit.rahit` only supports `ALPHA_MODE_MASK`; the open R6 marker mentions transmission, nested dielectrics, and `ALPHA_MODE_BLEND`.
- `BxDF.hlsli` has the R5 Frostbite diffuse + BVNDF GGX reflection model, but no transmission lobes, no relative eta, and no diffuse/specular transmission weights.
- `RTXPTSceneGraph` already parses sidecar `EnableTransmission`, `TransmissionFactor`, `IoR`, and `ThinSurface`, but `RTXPTMaterials` does not upload them to the GPU.
- DiligentTools GLTF already exposes `Material::TransmissionShaderAttribs` and `Material::VolumeShaderAttribs`; it does not expose `KHR_materials_ior`, so per-material IoR comes from the RTXPT sidecar JSON until DiligentTools grows that field.

RTXPT-fork anchors to read before coding:

- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:385-607` - `SpecularReflectionTransmissionMicrofacet` eval/sample/pdf and refraction Jacobian.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:620-702` - `StandardBSDFData` transmission/eta fields and `SetEta`.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/BxDF.hlsli:737-970` - `FalcorBSDF` lobe probabilities and mixture pdf.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/LobeType.hlsli:18-42` - reflection/transmission lobe bit layout.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/InteriorList.hlsli:24-247` - 2-slot nested dielectric priority stack.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNestedDielectrics.hlsli:18-128` - false-hit rejection, outside-IoR update, and stack update after transmission.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli:535-546` - volume absorption before nested-dielectric false-hit rejection.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Volumes/HomogeneousVolumeSampler.hlsli:117-135` - homogeneous transmittance evaluation.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerBridgeDonut.hlsli:742-792,855-887` - glTF transmission -> BSDF data, `updateOutsideIoR`, `loadIoR`, and volume attenuation conversion.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Materials/MaterialPT.h:17-77` - material GPU fields for transmission, IoR, nested priority, and volume.
- `D:/RTXPT-fork/Rtxpt/Materials/MaterialsBaker.h:142-185` and `MaterialsBaker.cpp:537-585` - CPU-side material flags and field upload.
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp:1017` and `SampleUI.h` nearby defaults - nested dielectric quality UI labels and tooltip.

Do not copy NVIDIA file headers, large comments, or wholesale source blocks. Port the behavior, names, constants, and control flow into Diligent-owned code.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp` - add R6 material sidecar fields for diffuse transmission, nested priority, and volume attenuation.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp` - parse those sidecar fields.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` - grow `MaterialPTData`, add R6 flag bits and helper declarations.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` - upload GLTF/sidecar transmission, volume, alpha-blend, thin-surface, and nested-priority data.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp` - mark alpha-blend geometry non-opaque so any-hit can run.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp` - add any-hit / alpha-blend stats if the implementation surfaces them in the debug UI.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - enable the nested dielectric UI and pass an any-hit requirement into the RT pass.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp` - compile any-hit when either material textures or stochastic alpha blend need it; grow `MaxPayloadSize`.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - mirror material/payload layout and R6 flags.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli` - add transmission/volume/IoR/material-header helpers.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli` - add diffuse/specular transmission lobes, eta, dielectric Fresnel, and mixture pdf.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit` - emit two-sided payload data instead of opaque-only shading state.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit` - add stochastic alpha-blend rejection.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - add interior-list state, false-hit rejection, outside-IoR update, absorption, and transmission-stack update after scatter.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli` - include R6 headers and expose helper functions used by raygen.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/InteriorList.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/HomogeneousVolumeData.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Volumes/HomogeneousVolumeSampler.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register new shader include files.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - record R6 mappings and intentional Diligent-native divergences.

## Cross-Cutting Contracts

- **Material layout:** `MaterialPTData` grows from 96 to 144 bytes. C++ and HLSL mirrors must update together, with `static_assert`s for size and offsets.
- **Payload layout:** `PathPayload` grows from 80 to 144 bytes. `RTXPTRayTracingPass::Initialize` must set `MaxPayloadSize = sizeof(float) * 36`, and the comment in `PathTracerShared.h` must match.
- **Any-hit activation:** Any-hit is required when either material textures are enabled or any material/geometry uses stochastic alpha blend. The any-hit shader must compile both with and without `ENABLE_MATERIAL_TEXTURES`.
- **Nested quality:** UI values match RTXPT-fork: `0=Off`, `1=Fast`, `2=Quality`. Fast allows up to 4 rejected dielectric hits and falls back to non-nested behavior after the limit; Quality allows up to 16 and terminates pathological loops after the limit.
- **Interior-list state:** The interior list is raygen-local, not payload-local. Rejected false hits must not consume a path bounce or a diffuse-bounce count.
- **Volume absorption:** Absorption is applied only while `InteriorList` is non-empty and before shading the hit segment, matching RTXPT-fork reference mode. No scattering is introduced in R6; `sigmaS` remains zero.
- **BSDF pdf:** `EvalBSDF` and `SampleBSDF` must share the same mixture pdf for reflection and transmission so direct-light, emissive, and environment MIS stay consistent.
- **Scope boundary:** This phase does not add clearcoat, sheen, anisotropy, OMM, SER, RTXDI, stable planes, volume scattering, or texture-coordinate transform support.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repo
- Verify: `DiligentSamples`
- Verify: `DiligentTools`
- Verify: RTXPT-fork reference anchors

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
git -C DiligentTools status --short --branch
```

Expected: branch lines are present. Existing unrelated changes may be present. Do not overwrite dirty files without reading them first.

- [ ] **Step 2: Confirm current R6 markers and baseline layout**

Run:

```powershell
rg -n "Phase R6|Phase 5\\.3|transmission|Nested Dielectrics|ALPHA_MODE_BLEND|PathPayload|MaterialPTData" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `RTXPTSample.hpp`, `RTXPTSample.cpp`, `PathTracerSample.rgen`, `PathTracerAnyHit.rahit`, `PathTracerShared.h`, and material files.

- [ ] **Step 3: Confirm the reference anchors are available**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Materials\BxDF.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Materials\InteriorList.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\PathTracerNestedDielectrics.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\Shaders\PathTracer\Rendering\Volumes\HomogeneousVolumeSampler.hlsli
```

Expected: all four commands print `True`.

- [ ] **Step 4: Commit nothing**

Expected: no commit in Task 0. This task only establishes the starting point.

---

### Task 1: Grow Material and Sidecar Contracts

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`

- [ ] **Step 1: Add sidecar material fields**

In `RTXPTSceneGraph.hpp`, extend `RTXPTMaterialExtension` after the existing R6 fields:

```cpp
    float  DiffuseTransmissionFactor              = 0.0f;
    float  ThicknessFactor                        = 0.0f;
    float3 VolumeAttenuationColor                 = float3{1, 1, 1};
    float  VolumeAttenuationDistance              = 3.402823466e+38f;
    int    NestedPriority                         = 14;
```

- [ ] **Step 2: Parse sidecar material fields**

In `ParseRTXPTMaterialExtension`, after `Ext.TransmissionFactor` and `Ext.IoR`, add:

```cpp
    Ext.DiffuseTransmissionFactor = ReadRTXPTOptionalFloat(Json, "DiffuseTransmissionFactor", Ext.DiffuseTransmissionFactor);
    Ext.ThicknessFactor           = ReadRTXPTOptionalFloat(Json, "ThicknessFactor", Ext.ThicknessFactor);
    Ext.VolumeAttenuationDistance = ReadRTXPTOptionalFloat(Json, "VolumeAttenuationDistance", Ext.VolumeAttenuationDistance);
    Ext.NestedPriority            = std::clamp(Json.value("NestedPriority", Ext.NestedPriority), 0, 14);

    float VolumeColor[3] = {Ext.VolumeAttenuationColor.x, Ext.VolumeAttenuationColor.y, Ext.VolumeAttenuationColor.z};
    if (ReadRTXPTFloatArray(Json, "VolumeAttenuationColor", VolumeColor, 3))
        Ext.VolumeAttenuationColor = float3{VolumeColor[0], VolumeColor[1], VolumeColor[2]};
```

If `std::clamp` is not visible in this file, add the existing local include style for `<algorithm>`.

- [ ] **Step 3: Grow the C++ GPU material record**

In `RTXPTMaterials.hpp`, replace the tail of `MaterialPTData` after `normalScale` with:

```cpp
    float transmissionFactor        = 0.0f; // offset 80
    float diffuseTransmissionFactor = 0.0f; // offset 84
    float ior                       = 1.5f; // offset 88
    float thicknessFactor           = 0.0f; // offset 92

    float3 volumeAttenuationColor    = float3{1, 1, 1};       // offset 96
    float  volumeAttenuationDistance = 3.402823466e+38f;      // offset 108

    Uint32 transmissionTextureIndex = 0;   // offset 112
    float  transmissionTextureSlice = 0.0f; // offset 116
    Uint32 thicknessTextureIndex    = 0;   // offset 120
    float  thicknessTextureSlice    = 0.0f; // offset 124

    Uint32 nestedPriority = 14; // offset 128; RTXPT-fork PTMaterial::kMaterialMaxNestedPriority.
    Uint32 _paddingR6_0   = 0;
    float  _paddingR6_1   = 0.0f;
    float  _paddingR6_2   = 0.0f;
};
static_assert(sizeof(MaterialPTData) == 144, "MaterialPTData layout must match PathTracer/PathTracerShared.h");
static_assert(offsetof(MaterialPTData, transmissionFactor) == 80,
              "MaterialPTData transmissionFactor offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(MaterialPTData, volumeAttenuationColor) == 96,
              "MaterialPTData volumeAttenuationColor offset must match PathTracer/PathTracerShared.h");
static_assert(offsetof(MaterialPTData, nestedPriority) == 128,
              "MaterialPTData nestedPriority offset must match PathTracer/PathTracerShared.h");
```

Keep the existing offset asserts for metallic/normal fields.

- [ ] **Step 4: Add C++ material flags**

In `RTXPTMaterials.hpp`, append these bits after `kMaterialFlag_EmissiveAreaLight`:

```cpp
constexpr Uint32 kMaterialFlag_HasTransmission        = 0x40u;
constexpr Uint32 kMaterialFlag_HasTransmissionTexture = 0x80u;
constexpr Uint32 kMaterialFlag_HasVolume              = 0x100u;
constexpr Uint32 kMaterialFlag_HasThicknessTexture    = 0x200u;
constexpr Uint32 kMaterialFlag_ThinSurface            = 0x400u;
constexpr Uint32 kMaterialFlag_AlphaBlend             = 0x800u;
```

Add helper declarations near the existing material helper declarations:

```cpp
bool RTXPTMaterialIsAlphaBlended(const GLTF::Material&         Material,
                                 const RTXPTMaterialExtension* pExtension);
bool RTXPTMaterialNeedsAnyHit(const GLTF::Material&         Material,
                              const RTXPTMaterialExtension* pExtension,
                              bool                          HasBaseColorTexture);
```

- [ ] **Step 5: Mirror material layout and flags in HLSL**

In `PathTracerShared.h`, replace the `MaterialPTData` tail after `normalScale` with the HLSL mirror:

```hlsl
    float transmissionFactor;        // offset 80
    float diffuseTransmissionFactor; // offset 84
    float ior;                       // offset 88
    float thicknessFactor;           // offset 92

    float3 volumeAttenuationColor;    // offset 96
    float  volumeAttenuationDistance; // offset 108

    uint  transmissionTextureIndex; // offset 112
    float transmissionTextureSlice; // offset 116
    uint  thicknessTextureIndex;    // offset 120
    float thicknessTextureSlice;    // offset 124

    uint  nestedPriority; // offset 128
    uint  _paddingR6_0;
    float _paddingR6_1;
    float _paddingR6_2;
};
```

Update the comment above `MaterialPTData` from 96 bytes to 144 bytes, and append matching HLSL flag constants:

```hlsl
static const uint kMaterialFlagHasTransmission        = 0x40u;
static const uint kMaterialFlagHasTransmissionTexture = 0x80u;
static const uint kMaterialFlagHasVolume              = 0x100u;
static const uint kMaterialFlagHasThicknessTexture    = 0x200u;
static const uint kMaterialFlagThinSurface            = 0x400u;
static const uint kMaterialFlagAlphaBlend             = 0x800u;
```

- [ ] **Step 6: Upload GLTF transmission and volume fields**

In `FillMaterialPTDataFromGLTF`, after normal texture handling, add:

```cpp
    Data.ior = 1.5f;

    if (Material.Transmission != nullptr)
    {
        Data.transmissionFactor = std::clamp(Material.Transmission->Factor, 0.0f, 1.0f);
        if (Data.transmissionFactor > 0.0f)
            Data.flags |= kMaterialFlag_HasTransmission;

        const int TransmissionTextureId = Material.GetTextureId(GLTF::DefaultTransmissionTextureAttribId);
        if (TransmissionTextureId >= 0)
        {
            Data.flags |= kMaterialFlag_HasTransmissionTexture;
            Data.transmissionTextureIndex = static_cast<Uint32>(TransmissionTextureId);
            Data.transmissionTextureSlice = Material.GetTextureAttrib(GLTF::DefaultTransmissionTextureAttribId).TextureSlice;
        }
    }

    if (Material.Volume != nullptr)
    {
        Data.flags |= kMaterialFlag_HasVolume;
        Data.thicknessFactor           = std::max(Material.Volume->ThicknessFactor, 0.0f);
        Data.volumeAttenuationColor    = Material.Volume->AttenuationColor;
        Data.volumeAttenuationDistance = std::max(Material.Volume->AttenuationDistance, 0.0f);

        const int ThicknessTextureId = Material.GetTextureId(GLTF::DefaultThicknessTextureAttribId);
        if (ThicknessTextureId >= 0)
        {
            Data.flags |= kMaterialFlag_HasThicknessTexture;
            Data.thicknessTextureIndex = static_cast<Uint32>(ThicknessTextureId);
            Data.thicknessTextureSlice = Material.GetTextureAttrib(GLTF::DefaultThicknessTextureAttribId).TextureSlice;
        }
    }

    if (Material.Attribs.AlphaMode == GLTF::Material::ALPHA_MODE_BLEND)
        Data.flags |= kMaterialFlag_AlphaBlend;
```

- [ ] **Step 7: Apply RTXPT sidecar overrides**

In `RTXPTMaterials::Upload(IRenderDevice*, const RTXPTSceneGraphData&)`, inside the `if (pExtension != nullptr && pExtension->Loaded)` block, after roughness assignment, add:

```cpp
                Data.transmissionFactor        = std::clamp(Ext.TransmissionFactor, 0.0f, 1.0f);
                Data.diffuseTransmissionFactor = std::clamp(Ext.DiffuseTransmissionFactor, 0.0f, 1.0f);
                Data.ior                       = std::max(Ext.IoR, 1.0f);
                Data.thicknessFactor           = std::max(Ext.ThicknessFactor, 0.0f);
                Data.volumeAttenuationColor    = Ext.VolumeAttenuationColor;
                Data.volumeAttenuationDistance = std::max(Ext.VolumeAttenuationDistance, 0.0f);
                Data.nestedPriority            = static_cast<Uint32>(std::clamp(Ext.NestedPriority, 0, 14));

                if (Ext.EnableTransmission || Data.transmissionFactor > 0.0f || Data.diffuseTransmissionFactor > 0.0f)
                    Data.flags |= kMaterialFlag_HasTransmission;
                else
                    Data.flags &= ~kMaterialFlag_HasTransmission;

                if (Ext.ThinSurface)
                    Data.flags |= kMaterialFlag_ThinSurface;
                else
                    Data.flags &= ~kMaterialFlag_ThinSurface;

                if (Data.volumeAttenuationDistance < 3.402823466e+38f ||
                    Data.volumeAttenuationColor.x != 1.0f ||
                    Data.volumeAttenuationColor.y != 1.0f ||
                    Data.volumeAttenuationColor.z != 1.0f)
                    Data.flags |= kMaterialFlag_HasVolume;
```

- [ ] **Step 8: Add material bridge helpers**

In `MaterialBridge.hlsli`, under the textured path, add:

```hlsl
    float getTransmission(MaterialPTData material, float2 uv)
    {
        float transmission = material.transmissionFactor;
        if ((material.flags & kMaterialFlagHasTransmissionTexture) != 0u)
            transmission *= sampleMaterialTexture(material.transmissionTextureIndex, material.transmissionTextureSlice, uv).r;
        return saturate(transmission);
    }

    float getThickness(MaterialPTData material, float2 uv)
    {
        float thickness = material.thicknessFactor;
        if ((material.flags & kMaterialFlagHasThicknessTexture) != 0u)
            thickness *= sampleMaterialTexture(material.thicknessTextureIndex, material.thicknessTextureSlice, uv).g;
        return max(thickness, 0.0);
    }
```

Under the non-textured fallback, add:

```hlsl
    float getTransmission(MaterialPTData material, float2 uv) { return saturate(material.transmissionFactor); }
    float getThickness(MaterialPTData material, float2 uv) { return max(material.thicknessFactor, 0.0); }
```

After the `#endif` for textured/fallback helpers, add:

```hlsl
    float loadIoR(uint materialID)
    {
        MaterialPTData material = getMaterial(materialID);
        return max(material.ior, 1.0);
    }

    bool isThinSurface(MaterialPTData material)
    {
        return (material.flags & kMaterialFlagThinSurface) != 0u || (material.flags & kMaterialFlagHasTransmission) == 0u;
    }
```

- [ ] **Step 9: Verify material contract symbols**

Run:

```powershell
rg -n "transmissionFactor|diffuseTransmissionFactor|volumeAttenuation|thicknessFactor|nestedPriority|kMaterialFlag_HasTransmission|kMaterialFlagHasTransmission|getTransmission|getThickness|loadIoR|isThinSurface" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in C++ material/scene graph files, `PathTracerShared.h`, and `MaterialBridge.hlsli`.

- [ ] **Step 10: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): upload R6 transmission material data" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 2: Add RTXPT-Fork Interior and Volume Shader Headers

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/InteriorList.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/HomogeneousVolumeData.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Volumes/HomogeneousVolumeSampler.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create lobe type bit flags**

Create `Rendering/Materials/LobeType.hlsli` with:

```hlsl
#ifndef __LOBE_TYPE_HLSLI__
#define __LOBE_TYPE_HLSLI__

static const uint kLobeTypeNone                    = 0x00u;
static const uint kLobeTypeDiffuseReflection       = 0x01u;
static const uint kLobeTypeSpecularReflection      = 0x02u;
static const uint kLobeTypeDeltaReflection         = 0x04u;
static const uint kLobeTypeDiffuseTransmission     = 0x10u;
static const uint kLobeTypeSpecularTransmission    = 0x20u;
static const uint kLobeTypeDeltaTransmission       = 0x40u;
static const uint kLobeTypeDiffuse                 = 0x11u;
static const uint kLobeTypeSpecular                = 0x22u;
static const uint kLobeTypeDelta                   = 0x44u;
static const uint kLobeTypeNonDelta                = 0x33u;
static const uint kLobeTypeReflection              = 0x0fu;
static const uint kLobeTypeTransmission            = 0xf0u;
static const uint kLobeTypeAll                     = 0xffu;

#endif // __LOBE_TYPE_HLSLI__
```

- [ ] **Step 2: Create material header helpers**

Create `Scene/Material/MaterialData.hlsli` with:

```hlsl
#ifndef __MATERIAL_DATA_HLSLI__
#define __MATERIAL_DATA_HLSLI__

#define EXTRACT_BITS(bits, offset, value) (((value) >> (offset)) & ((1u << (bits)) - 1u))
#define PACK_BITS(bits, offset, flags, value) ((((value) & ((1u << (bits)) - 1u)) << (offset)) | ((flags) & (~(((1u << (bits)) - 1u) << (offset)))))

struct MaterialHeader
{
    uint packedData;

    static const uint kNestedPriorityBits = 4;
    static const uint kLobeTypeBits       = 8;
    static const uint kNestedPriorityOffset = 0;
    static const uint kLobeTypeOffset       = kNestedPriorityOffset + kNestedPriorityBits;
    static const uint kThinSurfaceFlagOffset = kLobeTypeOffset + kLobeTypeBits;

    static MaterialHeader make()
    {
        MaterialHeader header;
        header.packedData = 0u;
        return header;
    }

    void setNestedPriority(uint priority) { packedData = PACK_BITS(kNestedPriorityBits, kNestedPriorityOffset, packedData, priority); }
    uint getNestedPriority() { return EXTRACT_BITS(kNestedPriorityBits, kNestedPriorityOffset, packedData); }

    void setActiveLobes(uint activeLobes) { packedData = PACK_BITS(kLobeTypeBits, kLobeTypeOffset, packedData, activeLobes); }
    uint getActiveLobes() { return EXTRACT_BITS(kLobeTypeBits, kLobeTypeOffset, packedData); }

    void setThinSurface(bool thinSurface) { packedData = PACK_BITS(1u, kThinSurfaceFlagOffset, packedData, thinSurface ? 1u : 0u); }
    bool isThinSurface() { return (packedData & (1u << kThinSurfaceFlagOffset)) != 0u; }
};

#undef PACK_BITS
#undef EXTRACT_BITS

#endif // __MATERIAL_DATA_HLSLI__
```

- [ ] **Step 3: Create the 2-slot interior list**

Create `Rendering/Materials/InteriorList.hlsli` with the RTXPT-fork-compatible 2-slot behavior:

```hlsl
#ifndef __INTERIOR_LIST_HLSLI__
#define __INTERIOR_LIST_HLSLI__

#ifndef INTERIOR_LIST_SLOT_COUNT
#define INTERIOR_LIST_SLOT_COUNT 2
#endif

struct InteriorList
{
    static const uint kNoMaterial           = 0xffffffffu;
    static const uint kMaterialBits         = 28u;
    static const uint kNestedPriorityBits   = 4u;
    static const uint kMaterialOffset       = 0u;
    static const uint kNestedPriorityOffset = kMaterialOffset + kMaterialBits;
    static const uint kMaterialMask         = ((1u << kMaterialBits) - 1u) << kMaterialOffset;
    static const uint kMaxNestedPriority    = (1u << kNestedPriorityBits) - 1u;

    uint2 slots;

    static InteriorList make()
    {
        InteriorList list;
        list.slots = uint2(0u, 0u);
        return list;
    }

    uint makeSlot(uint materialID, uint nestedPriority)
    {
        return (nestedPriority << kNestedPriorityOffset) | (materialID & kMaterialMask);
    }

    bool isSlotActive(uint slot) { return slot != 0u; }
    bool isEmpty() { return !isSlotActive(slots.x); }
    uint getSlotNestedPriority(uint slot) { return slot >> kNestedPriorityOffset; }
    uint getSlotMaterialID(uint slot) { return slot & kMaterialMask; }
    uint getTopNestedPriority() { return getSlotNestedPriority(slots.x); }
    uint getTopMaterialID() { return isSlotActive(slots.x) ? getSlotMaterialID(slots.x) : kNoMaterial; }
    uint getNextMaterialID() { return isSlotActive(slots.y) ? getSlotMaterialID(slots.y) : kNoMaterial; }

    bool isTrueIntersection(uint nestedPriority)
    {
        return nestedPriority == 0u || nestedPriority >= getTopNestedPriority();
    }

    void sortSlots()
    {
        if (slots.x < slots.y)
        {
            uint tmp = slots.x;
            slots.x = slots.y;
            slots.y = tmp;
        }
    }

    void handleIntersection(uint materialID, uint nestedPriority, bool entering)
    {
        if (nestedPriority == 0u)
            nestedPriority = kMaxNestedPriority;

        if (entering && slots.x == 0u)
            slots.x = makeSlot(materialID, nestedPriority);
        else if (!entering && isSlotActive(slots.x) && getSlotMaterialID(slots.x) == materialID)
            slots.x = 0u;
        else if (entering && slots.y == 0u)
            slots.y = makeSlot(materialID, nestedPriority);
        else if (!entering && isSlotActive(slots.y) && getSlotMaterialID(slots.y) == materialID)
            slots.y = 0u;

        sortSlots();
    }
};

#endif // __INTERIOR_LIST_HLSLI__
```

- [ ] **Step 4: Create homogeneous volume data and sampler**

Create `Scene/Material/HomogeneousVolumeData.hlsli`:

```hlsl
#ifndef __HOMOGENEOUS_VOLUME_DATA_HLSLI__
#define __HOMOGENEOUS_VOLUME_DATA_HLSLI__

struct HomogeneousVolumeData
{
    float3 sigmaA;
    float3 sigmaS;
    float  g;
};

#endif // __HOMOGENEOUS_VOLUME_DATA_HLSLI__
```

Create `Rendering/Volumes/HomogeneousVolumeSampler.hlsli`:

```hlsl
#ifndef __HOMOGENEOUS_VOLUME_SAMPLER_HLSLI__
#define __HOMOGENEOUS_VOLUME_SAMPLER_HLSLI__

#include "../../Scene/Material/HomogeneousVolumeData.hlsli"

struct HomogeneousVolumeSampler
{
    static float3 evalTransmittance(float3 sigmaT, float distance)
    {
        return exp(-max(distance, 0.0) * sigmaT);
    }

    static float3 evalTransmittance(HomogeneousVolumeData vd, float distance)
    {
        return evalTransmittance(vd.sigmaA + vd.sigmaS, distance);
    }
};

#endif // __HOMOGENEOUS_VOLUME_SAMPLER_HLSLI__
```

- [ ] **Step 5: Add volume bridge helper**

In `MaterialBridge.hlsli`, include the new volume data header:

```hlsl
#include "../../Scene/Material/HomogeneousVolumeData.hlsli"
```

Add this helper in the `Bridge` namespace after `loadIoR`:

```hlsl
    HomogeneousVolumeData loadHomogeneousVolumeData(uint materialID)
    {
        HomogeneousVolumeData volume;
        volume.sigmaS = float3(0.0, 0.0, 0.0);
        volume.sigmaA = float3(0.0, 0.0, 0.0);
        volume.g      = 0.0;

        MaterialPTData material = getMaterial(materialID);
        if ((material.flags & kMaterialFlagHasVolume) == 0u)
            return volume;

        const float3 attenuationColor = clamp(material.volumeAttenuationColor, 1e-7, 1.0);
        const float  attenuationDistance = max(material.volumeAttenuationDistance, 1e-30);
        volume.sigmaA = -log(attenuationColor) / attenuationDistance.xxx;
        return volume;
    }
```

- [ ] **Step 6: Create nested dielectric helpers**

Create `PathTracerNestedDielectrics.hlsli` with:

```hlsl
#ifndef __PATH_TRACER_NESTED_DIELECTRICS_HLSLI__
#define __PATH_TRACER_NESTED_DIELECTRICS_HLSLI__

#include "Rendering/Materials/InteriorList.hlsli"
#include "Rendering/Materials/LobeType.hlsli"

namespace PathTracer
{
    float ComputeOutsideIoR(InteriorList interiorList, uint materialID, bool entering)
    {
        uint outsideMaterialID = interiorList.getTopMaterialID();
        if (!entering && outsideMaterialID == materialID)
            outsideMaterialID = interiorList.getNextMaterialID();

        if (outsideMaterialID == InteriorList::kNoMaterial)
            return 1.0;

        return Bridge::loadIoR(outsideMaterialID);
    }

    uint GetMaxRejectedDielectricHits(uint nestedQuality)
    {
        return nestedQuality == 2u ? 16u : 4u;
    }

    bool HandleNestedDielectrics(PathPayload payload,
                                 uint nestedQuality,
                                 inout InteriorList interiorList,
                                 inout uint rejectedHits,
                                 out float outsideIoR)
    {
        const bool entering = payload.frontFacing != 0u;
        outsideIoR = ComputeOutsideIoR(interiorList, payload.materialID, entering);

        if (nestedQuality == 0u || payload.thinSurface != 0u)
            return true;

        const uint maxRejectedHits = GetMaxRejectedDielectricHits(nestedQuality);
        if (rejectedHits < maxRejectedHits && !interiorList.isTrueIntersection(payload.nestedPriority))
        {
            ++rejectedHits;
            interiorList.handleIntersection(payload.materialID, payload.nestedPriority, entering);
            return false;
        }

        if (nestedQuality == 2u && rejectedHits >= maxRejectedHits && !interiorList.isTrueIntersection(payload.nestedPriority))
            return false;

        return true;
    }

    void UpdateNestedDielectricsOnScatterTransmission(PathPayload payload, uint lobe, inout InteriorList interiorList)
    {
        if ((lobe & kLobeTypeTransmission) == 0u || payload.thinSurface != 0u)
            return;

        interiorList.handleIntersection(payload.materialID, payload.nestedPriority, payload.frontFacing != 0u);
    }
}

#endif // __PATH_TRACER_NESTED_DIELECTRICS_HLSLI__
```

Task 4 adds the `PathPayload` fields referenced here.

- [ ] **Step 7: Include R6 helpers**

In `PathTracer.hlsli`, add includes after the existing includes:

```hlsl
#include "Rendering/Volumes/HomogeneousVolumeSampler.hlsli"
#include "PathTracerNestedDielectrics.hlsli"
```

If include ordering creates a dependency cycle, move `PathTracerNestedDielectrics.hlsli` after `BxDF.hlsli` is included through the existing path.

- [ ] **Step 8: Register new shader headers**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the new headers near their neighboring shader files:

```cmake
    assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli
    assets/shaders/PathTracer/Rendering/Materials/InteriorList.hlsli
    assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli
    assets/shaders/PathTracer/Scene/Material/HomogeneousVolumeData.hlsli
    assets/shaders/PathTracer/Rendering/Volumes/HomogeneousVolumeSampler.hlsli
    assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli
```

- [ ] **Step 9: Verify new header symbols**

Run:

```powershell
rg -n "kLobeTypeTransmission|InteriorList|MaterialHeader|HomogeneousVolumeData|HomogeneousVolumeSampler|ComputeOutsideIoR|HandleNestedDielectrics|UpdateNestedDielectricsOnScatterTransmission|loadHomogeneousVolumeData" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "PathTracerNestedDielectrics.hlsli|InteriorList.hlsli|HomogeneousVolumeSampler.hlsli" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: all requested symbols and CMake registrations are present.

- [ ] **Step 10: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/LobeType.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/InteriorList.hlsli Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/MaterialData.hlsli Samples/RTXPT/assets/shaders/PathTracer/Scene/Material/HomogeneousVolumeData.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Volumes/HomogeneousVolumeSampler.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli Samples/RTXPT/CMakeLists.txt
git -C DiligentSamples commit -m "feat(rtxpt): add R6 nested dielectric helpers" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 3: Port Transmission Lobes Into the BSDF

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`

- [ ] **Step 1: Include lobe types and replace local lobe constants**

At the top of `BxDF.hlsli`, after the sample-generator include, add:

```hlsl
#include "LobeType.hlsli"
```

Replace the R5 lobe constants with aliases to the new RTXPT-fork-compatible bit layout:

```hlsl
static const uint kBSDFLobeDiffuseReflection    = kLobeTypeDiffuseReflection;
static const uint kBSDFLobeSpecularReflection   = kLobeTypeSpecularReflection;
static const uint kBSDFLobeDeltaReflection      = kLobeTypeDeltaReflection;
static const uint kBSDFLobeDiffuseTransmission  = kLobeTypeDiffuseTransmission;
static const uint kBSDFLobeSpecularTransmission = kLobeTypeSpecularTransmission;
static const uint kBSDFLobeDeltaTransmission    = kLobeTypeDeltaTransmission;
static const uint kBSDFLobeDelta                = kLobeTypeDelta;
static const uint kBSDFLobeTransmission         = kLobeTypeTransmission;
```

- [ ] **Step 2: Extend `StandardBSDFData`**

Replace the struct with:

```hlsl
struct StandardBSDFData
{
    float3 N;
    float3 diffuse;
    float3 specular;
    float3 transmission;
    float  roughness;
    float  alpha;
    float  eta; // incident IoR / transmitted IoR.
    float  metallic;
    float  diffuseTransmission;
    float  specularTransmission;
    bool   thinSurface;
};
```

Replace `MakeStandardBSDFData` with this signature:

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
```

Use this initialization body:

```hlsl
{
    const float r     = saturate(roughness);
    const float alpha = r * r;
    const float safeMaterialIoR = max(materialIoR, 1.0);
    const float safeOutsideIoR  = max(outsideIoR, 1.0);
    const float f0Term          = (safeMaterialIoR - 1.0) / max(safeMaterialIoR + 1.0, 1e-6);
    const float dielectricF0    = f0Term * f0Term;

    StandardBSDFData bsdfData;
    bsdfData.N                    = N;
    bsdfData.roughness            = r;
    bsdfData.alpha                = (alpha < kMinGGXAlpha) ? 0.0 : max(alpha, kMinGGXAlpha);
    bsdfData.diffuse              = baseColor * (1.0 - metallic);
    bsdfData.specular             = lerp(float3(dielectricF0, dielectricF0, dielectricF0), baseColor, metallic);
    bsdfData.transmission         = baseColor;
    bsdfData.eta                  = frontFacing ? safeOutsideIoR / safeMaterialIoR : safeMaterialIoR / safeOutsideIoR;
    bsdfData.metallic             = saturate(metallic);
    bsdfData.diffuseTransmission  = saturate(diffuseTransmissionFactor) * (1.0 - bsdfData.metallic);
    bsdfData.specularTransmission = saturate(transmissionFactor) * (1.0 - bsdfData.metallic);
    bsdfData.thinSurface          = thinSurface;
    return bsdfData;
}
```

- [ ] **Step 3: Add dielectric Fresnel helper**

Add this helper after the existing Schlick helpers:

```hlsl
float evalFresnelDielectric(float eta, float cosThetaI, out float cosThetaT)
{
    cosThetaI = clamp(cosThetaI, -1.0, 1.0);
    const float sin2ThetaI = max(0.0, 1.0 - cosThetaI * cosThetaI);
    const float sin2ThetaT = eta * eta * sin2ThetaI;
    if (sin2ThetaT >= 1.0)
    {
        cosThetaT = 0.0;
        return 1.0;
    }

    cosThetaT = sqrt(max(0.0, 1.0 - sin2ThetaT));
    const float rs = (cosThetaI - eta * cosThetaT) / max(cosThetaI + eta * cosThetaT, 1e-7);
    const float rp = (eta * cosThetaI - cosThetaT) / max(eta * cosThetaI + cosThetaT, 1e-7);
    return saturate(0.5 * (rs * rs + rp * rp));
}

float evalFresnelDielectric(float eta, float cosThetaI)
{
    float cosThetaT;
    return evalFresnelDielectric(eta, cosThetaI, cosThetaT);
}
```

- [ ] **Step 4: Add lobe probability helper**

Add this struct and helper before `EvalBSDF`:

```hlsl
struct BSDFLobeProbabilities
{
    float pDiffuseReflection;
    float pDiffuseTransmission;
    float pSpecularReflection;
    float pSpecularTransmission;
};

BSDFLobeProbabilities GetBSDFLobeProbabilities(StandardBSDFData bsdfData, float3 wo)
{
    const float NdotV = saturate(dot(bsdfData.N, wo));
    const float3 fApprox = evalFresnelSchlick(bsdfData.specular, float3(1.0, 1.0, 1.0), NdotV);
    const float diffuseWeight = luminance(bsdfData.diffuse);
    const float specularWeight = luminance(fApprox);
    const float dielectricBSDF = (1.0 - bsdfData.metallic) * (1.0 - bsdfData.specularTransmission);
    const float metallicBRDF = bsdfData.metallic * (1.0 - bsdfData.specularTransmission);

    BSDFLobeProbabilities p;
    p.pDiffuseReflection = diffuseWeight * dielectricBSDF * (1.0 - bsdfData.diffuseTransmission);
    p.pDiffuseTransmission = diffuseWeight * dielectricBSDF * bsdfData.diffuseTransmission;
    p.pSpecularReflection = specularWeight * (metallicBRDF + dielectricBSDF);
    p.pSpecularTransmission = bsdfData.specularTransmission;

    const float norm = p.pDiffuseReflection + p.pDiffuseTransmission + p.pSpecularReflection + p.pSpecularTransmission;
    if (norm > 0.0)
    {
        const float invNorm = 1.0 / norm;
        p.pDiffuseReflection *= invNorm;
        p.pDiffuseTransmission *= invNorm;
        p.pSpecularReflection *= invNorm;
        p.pSpecularTransmission *= invNorm;
    }
    return p;
}
```

Keep `getSpecularProbability` for R5/R4 call sites temporarily; Task 5 updates call sites to the full probability helper where needed.

- [ ] **Step 5: Add diffuse transmission evaluation and sampling**

Add these helpers before `EvalBSDF`:

```hlsl
float3 EvalDiffuseTransmission(StandardBSDFData bsdfData, float3 wiLocal, float3 woLocal, out float pdf)
{
    pdf = 0.0;
    if (wiLocal.z <= kMinCosTheta || woLocal.z >= -kMinCosTheta)
        return float3(0.0, 0.0, 0.0);

    const float cosThetaT = -woLocal.z;
    pdf = cosThetaT * K_1_PI;
    const float3 albedo = bsdfData.thinSurface ? bsdfData.transmission : sqrt(max(bsdfData.transmission, 0.0));
    return albedo * (cosThetaT * K_1_PI);
}

float3 SampleDiffuseTransmission(StandardBSDFData bsdfData, float2 rand, out float3 wi, out float pdf)
{
    float localPdf;
    float3 localWi = sampleCosineHemisphere(rand, float3(0.0, 0.0, 1.0), localPdf);
    localWi.z = -localWi.z;
    pdf = localPdf;
    wi = localWi;
    const float3 albedo = bsdfData.thinSurface ? bsdfData.transmission : sqrt(max(bsdfData.transmission, 0.0));
    return albedo;
}
```

This helper works in local space. Convert between local/world in `SampleBSDF`.

- [ ] **Step 6: Add specular reflection/transmission eval/pdf helpers**

Add local-space helpers equivalent to RTXPT-fork's `SpecularReflectionTransmissionMicrofacet`:

```hlsl
float3 EvalSpecularReflectionTransmission(StandardBSDFData bsdfData, float3 wiLocal, float3 woLocal)
{
    if (min(wiLocal.z, abs(woLocal.z)) < kMinCosTheta || bsdfData.alpha == 0.0)
        return float3(0.0, 0.0, 0.0);

    const bool isReflection = woLocal.z > 0.0;
    float actualEta = (bsdfData.thinSurface && !isReflection) ? 1.0 : bsdfData.eta;

    float3 h = normalize(woLocal + wiLocal * (isReflection ? 1.0 : actualEta));
    h *= sign(h.z);

    const float wiDotH = dot(wiLocal, h);
    const float woDotH = dot(woLocal, h);
    const float D = evalNdfGGX(bsdfData.alpha, h.z);
    const float G = evalVisibilitySmithGGXCorrelated(bsdfData.alpha, wiLocal.z, abs(woLocal.z)) *
        4.0 * wiLocal.z * abs(woLocal.z);
    const float F = evalFresnelDielectric(actualEta, wiDotH);

    if (isReflection)
        return F * D * G * 0.25 / max(wiLocal.z, 1e-7);

    const float sqrtDenom = woDotH + actualEta * wiDotH;
    const float t = actualEta * actualEta * wiDotH * woDotH / max(wiLocal.z * sqrtDenom * sqrtDenom, 1e-7);
    const float3 albedo = bsdfData.thinSurface ? bsdfData.transmission : sqrt(max(bsdfData.transmission, 0.0));
    return albedo * (1.0 - F) * D * G * abs(t);
}

float EvalPdfSpecularReflectionTransmission(StandardBSDFData bsdfData, float3 wiLocal, float3 woLocal)
{
    if (min(wiLocal.z, abs(woLocal.z)) < kMinCosTheta || bsdfData.alpha == 0.0)
        return 0.0;

    const bool isReflection = woLocal.z > 0.0;
    float actualEta = (bsdfData.thinSurface && !isReflection) ? 1.0 : bsdfData.eta;
    float3 h = normalize(woLocal + wiLocal * (isReflection ? 1.0 : actualEta));
    h *= sign(h.z);

    const float wiDotH = dot(wiLocal, h);
    const float woDotH = dot(woLocal, h);
    const float F = evalFresnelDielectric(actualEta, wiDotH);
    float pdf = evalPdfGGX_BVNDF(bsdfData.alpha, wiLocal, h);

    if (isReflection)
    {
        if (woDotH <= 0.0)
            return 0.0;
        pdf *= wiDotH / woDotH;
    }
    else
    {
        if (woDotH > 0.0)
            return 0.0;
        const float sqrtDenom = woDotH + actualEta * wiDotH;
        pdf *= wiDotH * 4.0;
        pdf *= abs(woDotH) / max(sqrtDenom * sqrtDenom, 1e-7);
    }

    if (bsdfData.specularTransmission > 0.0)
        pdf *= isReflection ? F : (1.0 - F);

    return clamp(pdf, 0.0, 3.402823466e+38);
}
```

- [ ] **Step 7: Update `EvalBSDF` for reflection and transmission**

Replace `EvalBSDF` with a version that:

- builds `woLocal` and `wiLocal` from `bsdfData.N`;
- gets `BSDFLobeProbabilities p = GetBSDFLobeProbabilities(bsdfData, wo)`;
- evaluates diffuse/specular reflection only when `wiLocal.z > 0`;
- evaluates diffuse/specular transmission only when `wiLocal.z < 0`;
- computes `pdf` as the same mixture used by `SampleBSDF`.

The public signature stays:

```hlsl
void EvalBSDF(StandardBSDFData bsdfData, float3 wo, float3 wi, float specProb, out float3 f, out float pdf)
```

Inside the function, ignore `specProb` after all call sites move to lobe probabilities:

```hlsl
    (void)specProb;
```

Keep this output convention:

```hlsl
    f = diffuseReflection + diffuseTransmission + specularReflection + specularTransmission;
    pdf = p.pDiffuseReflection * pdfDiffuseReflection +
          p.pDiffuseTransmission * pdfDiffuseTransmission +
          p.pSpecularReflection * pdfSpecularReflection +
          p.pSpecularTransmission * pdfSpecularTransmission;
```

- [ ] **Step 8: Update `SampleBSDF` to sample four lobe families**

Replace `SampleBSDF` internals so `preGeneratedSample.z` selects the cumulative probabilities:

```hlsl
const BSDFLobeProbabilities p = GetBSDFLobeProbabilities(bsdfData, wo);
const float uSelect = preGeneratedSample.z;

if (uSelect < p.pDiffuseReflection)
{
    // cosine reflection, lobe = kBSDFLobeDiffuseReflection
}
else if (uSelect < p.pDiffuseReflection + p.pDiffuseTransmission)
{
    // cosine transmission, lobe = kBSDFLobeDiffuseTransmission
}
else if (uSelect < p.pDiffuseReflection + p.pDiffuseTransmission + p.pSpecularReflection)
{
    // existing BVNDF specular reflection, lobe = delta/specular reflection
}
else if (p.pSpecularTransmission > 0.0)
{
    // rough or delta dielectric transmission, lobe = delta/specular transmission
}
```

Required behavior:

- Delta reflection and delta transmission both return `pdf = 0.0` as the existing firefly/MIS sentinel.
- Specular transmission uses `evalFresnelDielectric(bsdfData.eta, wiDotH, cosThetaT)`.
- Thin-surface transmission uses `actualEta = 1.0` for transmission direction and skips volume stack updates later.
- Non-delta transmission calls `EvalBSDF` to obtain the mixture pdf and returns `weight = f / pdf`.
- The function returns false if the sampled world direction is invalid or non-finite.

- [ ] **Step 9: Verify BSDF transmission symbols**

Run:

```powershell
rg -n "kBSDFLobeDiffuseTransmission|kBSDFLobeSpecularTransmission|kBSDFLobeDeltaTransmission|kBSDFLobeTransmission|evalFresnelDielectric|BSDFLobeProbabilities|EvalDiffuseTransmission|EvalSpecularReflectionTransmission|EvalPdfSpecularReflectionTransmission|diffuseTransmission|specularTransmission|eta|thinSurface" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
```

Expected: all requested symbols are present in `BxDF.hlsli`.

- [ ] **Step 10: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): port R6 transmission BSDF lobes" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 4: Expand Payload and Closest-Hit Two-Sided Surface Data

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Grow `PathPayload`**

In `PathTracerShared.h`, replace the `PathPayload` comment and struct with:

```hlsl
// Reference path tracer payload. Size is 144 bytes (36 floats); keep RTXPTRayTracingPass::Initialize
// MaxPayloadSize in sync when this changes.
struct PathPayload
{
    float3 worldPos;
    float  hitDistance;

    float3 worldNormal; // Shading normal oriented against the incoming ray.
    uint   hitFlag;

    float3 faceNormal; // Geometric face normal oriented against the incoming ray.
    uint   materialID;

    float3 baseColor;
    float  metallic;

    float3 emission;
    float  roughness;

    float emissiveLightPdf;
    float ior;
    float transmissionFactor;
    float diffuseTransmissionFactor;

    float3 transmissionColor;
    float  volumeAttenuationDistance;

    float3 volumeAttenuationColor;
    uint   materialFlags;

    uint  nestedPriority;
    uint  frontFacing;
    uint  thinSurface;
    float alpha;
};
```

- [ ] **Step 2: Initialize the expanded payload**

In `PathTracer.hlsli::MakeEmptyPayload`, initialize the new fields:

```hlsl
        payload.faceNormal                  = float3(0.0, 1.0, 0.0);
        payload.materialID                  = 0u;
        payload.ior                         = 1.5;
        payload.transmissionFactor          = 0.0;
        payload.diffuseTransmissionFactor   = 0.0;
        payload.transmissionColor           = float3(1.0, 1.0, 1.0);
        payload.volumeAttenuationDistance   = 3.402823466e+38;
        payload.volumeAttenuationColor      = float3(1.0, 1.0, 1.0);
        payload.materialFlags               = 0u;
        payload.nestedPriority              = 14u;
        payload.frontFacing                 = 1u;
        payload.thinSurface                 = 1u;
        payload.alpha                       = 1.0;
```

- [ ] **Step 3: Update RT payload size**

In `RTXPTRayTracingPass.cpp`, replace the payload-size comment and value:

```cpp
    // PathPayload = 9 * float4 = 144 bytes (R6 transmission/nested-dielectric surface data).
    PSOCreateInfo.MaxPayloadSize = static_cast<Uint32>(sizeof(float) * 36);
```

- [ ] **Step 4: Emit two-sided closest-hit fields**

In `PathTracerClosestHit.rchit`, replace the single-sided normal flip block:

```hlsl
        // Flip the shading normal to face the camera (single-sided shading; transmission is deferred).
        if (dot(WorldNormal, RayDir) > 0.0)
            WorldNormal = -WorldNormal;
```

with:

```hlsl
        const bool frontFacing = dot(geometricNormal, RayDir) < 0.0;
        float3 FaceNormal = frontFacing ? geometricNormal : -geometricNormal;
        if (dot(WorldNormal, FaceNormal) < 0.0)
            WorldNormal = -WorldNormal;
```

After normal-map perturbation, replace the camera-facing flip with a face-normal consistency correction:

```hlsl
                if (dot(WorldNormal, FaceNormal) < 0.0)
                    WorldNormal = -WorldNormal;
```

- [ ] **Step 5: Fill R6 payload material fields**

After `BaseColor`, `Metallic`, and `Roughness` are resolved, fill:

```hlsl
        const float transmission = Bridge::getTransmission(material, texCoord);

        Payload.materialID                = subInstance.MaterialID;
        Payload.faceNormal                = FaceNormal;
        Payload.ior                       = max(material.ior, 1.0);
        Payload.transmissionFactor        = transmission;
        Payload.diffuseTransmissionFactor = saturate(material.diffuseTransmissionFactor);
        Payload.transmissionColor         = BaseColor;
        Payload.volumeAttenuationDistance = material.volumeAttenuationDistance;
        Payload.volumeAttenuationColor    = material.volumeAttenuationColor;
        Payload.materialFlags             = material.flags;
        Payload.nestedPriority            = min(material.nestedPriority, 14u);
        Payload.frontFacing               = frontFacing ? 1u : 0u;
        Payload.thinSurface               = Bridge::isThinSurface(material) ? 1u : 0u;
        Payload.alpha                     = Bridge::getBaseColor(material, texCoord).a;
```

If this duplicates `Bridge::getBaseColor`, keep the earlier result as a `float4 BaseColorWithAlpha` and reuse it:

```hlsl
        const float4 BaseColorWithAlpha = Bridge::getBaseColor(material, texCoord);
        BaseColor = BaseColorWithAlpha.rgb;
        Payload.alpha = BaseColorWithAlpha.a;
```

- [ ] **Step 6: Preserve fallback payload values**

Before the `if (Bridge::hasSubInstanceTable() && Bridge::hasMaterialTable())` block, initialize fallback values:

```hlsl
    float3 FaceNormal = -WorldRayDirection();
    uint   MaterialID = 0u;
    bool   FrontFacing = true;
```

At the bottom, always assign:

```hlsl
    Payload.faceNormal  = normalize(FaceNormal);
    Payload.materialID  = MaterialID;
    Payload.frontFacing = FrontFacing ? 1u : 0u;
```

Inside the material path, set `MaterialID = subInstance.MaterialID;` and `FrontFacing = frontFacing;`.

- [ ] **Step 7: Verify payload and closest-hit fields**

Run:

```powershell
rg -n "faceNormal|materialID|transmissionFactor|diffuseTransmissionFactor|transmissionColor|volumeAttenuation|nestedPriority|frontFacing|thinSurface|MaxPayloadSize|36" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
rg -n "single-sided shading|transmission is deferred|sizeof\\(float\\) \\* 20" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: first command finds the R6 payload plumbing. Second command has no stale single-sided/deferred comments or old payload-size value.

- [ ] **Step 8: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): return R6 surface data from closest hit" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 5: Integrate Nested Dielectrics and Volume Absorption in Raygen

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli`

- [ ] **Step 1: Add nested-dielectric quality to frame constants**

In `RTXPTFrameConstants.hpp`, replace the R5 padding tail:

```cpp
    Uint32 diffuseBounceCount     = 2; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    Uint32 _paddingR5_0           = 0;
    Uint32 _paddingR5_1           = 0;
    Uint32 _paddingR5_2           = 0;
```

with:

```cpp
    Uint32 diffuseBounceCount       = 2; // R5/G9: max diffuse bounces and BSDF LD sampling window.
    Uint32 nestedDielectricsQuality = 1; // R6/G10: 0=Off, 1=Fast, 2=Quality.
    Uint32 _paddingR6_0             = 0;
    Uint32 _paddingR6_1             = 0;
```

Mirror the same field names in `PathTracerShared.h`.

- [ ] **Step 2: Upload the UI value**

In `RTXPTSample.cpp::UpdateFrameConstants`, after `diffuseBounceCount`, add:

```cpp
    m_LastFrameConstants.ptConsts.nestedDielectricsQuality =
        static_cast<Uint32>(std::clamp(m_ReferenceUI.NestedDielectricsQuality, 0, 2));
```

- [ ] **Step 3: Enable the nested dielectric UI**

In `RTXPTSample.cpp::UpdateUI`, replace the disabled nested dielectric block with:

```cpp
        if (ResetOnChange(ImGui::Combo("Nested Dielectrics", &m_ReferenceUI.NestedDielectricsQuality, "Off\0Fast\0Quality\0\0"),
                          "Nested dielectric quality changed"))
        {
            m_ReferenceUI.NestedDielectricsQuality = std::clamp(m_ReferenceUI.NestedDielectricsQuality, 0, 2);
        }
        if (ImGui::IsItemHovered())
            ImGui::SetTooltip("Priority-based nested dielectrics. Fast allows fewer false-hit rejections; Quality allows more.");
```

- [ ] **Step 4: Convert the raygen bounce loop to a manual loop**

In `PathTracerSample.rgen`, replace:

```hlsl
    [loop]
    for (uint bounce = 0u; bounce < maxBounces || terminateAtNextEndpoint; ++bounce)
```

with:

```hlsl
    uint bounce = 0u;
    uint rejectedDielectricHits = 0u;
    InteriorList interiorList = InteriorList::make();

    [loop]
    while (bounce < maxBounces || terminateAtNextEndpoint)
```

At the end of the loop, after `rayDir = nextDir;`, add:

```hlsl
        ++bounce;
```

When `payload.hitFlag == 0u`, keep the existing `break`.

- [ ] **Step 5: Apply volume absorption before shading each hit**

After the hit check and before surface emission accumulation, add:

```hlsl
        if (!interiorList.isEmpty())
        {
            const HomogeneousVolumeData volume = Bridge::loadHomogeneousVolumeData(interiorList.getTopMaterialID());
            throughput *= HomogeneousVolumeSampler::evalTransmittance(volume, payload.hitDistance);
        }
```

This matches RTXPT-fork's absorption-only behavior. No volume scattering path is added in R6.

- [ ] **Step 6: Reject false nested-dielectric hits**

After volume absorption and before surface emission accumulation, add:

```hlsl
        float outsideIoR = 1.0;
        const bool acceptedDielectricHit =
            PathTracer::HandleNestedDielectrics(payload,
                                                g_Const.ptConsts.nestedDielectricsQuality,
                                                interiorList,
                                                rejectedDielectricHits,
                                                outsideIoR);
        if (!acceptedDielectricHit)
        {
            if (g_Const.ptConsts.nestedDielectricsQuality == 2u &&
                rejectedDielectricHits >= PathTracer::GetMaxRejectedDielectricHits(g_Const.ptConsts.nestedDielectricsQuality))
                break;

            const float bias = max(1e-4, 1e-3 * payload.hitDistance);
            rayOrigin = payload.worldPos - payload.faceNormal * bias;
            continue;
        }
```

This `continue` must not increment `bounce`; with the manual loop from Step 4, it preserves RTXPT-fork's "decrement vertex index" behavior.

- [ ] **Step 7: Build BSDF data with outside IoR**

Replace the current `MakeStandardBSDFData` call:

```hlsl
        StandardBSDFData bsdfData = MakeStandardBSDFData(payload.worldNormal, payload.baseColor, payload.metallic, payload.roughness);
```

with:

```hlsl
        StandardBSDFData bsdfData =
            MakeStandardBSDFData(payload.worldNormal,
                                 payload.baseColor,
                                 payload.metallic,
                                 payload.roughness,
                                 payload.ior,
                                 outsideIoR,
                                 payload.transmissionFactor,
                                 payload.diffuseTransmissionFactor,
                                 payload.thinSurface != 0u,
                                 payload.frontFacing != 0u);
```

- [ ] **Step 8: Use face normal for visibility and scatter origins**

Replace:

```hlsl
        const float3 visibilityOrigin = payload.worldPos + bsdfData.N * bias;
```

with:

```hlsl
        const float3 visibilityOrigin = payload.worldPos + payload.faceNormal * bias;
```

For transmitted scatter directions, the next ray must start on the opposite side of the face normal. After `SampleBSDF`, add:

```hlsl
        const bool isTransmission = (lobe & kBSDFLobeTransmission) != 0u;
        const float3 scatterOrigin = payload.worldPos + (isTransmission ? -payload.faceNormal : payload.faceNormal) * bias;
```

Replace the later `rayOrigin = visibilityOrigin;` with:

```hlsl
        rayOrigin = scatterOrigin;
```

- [ ] **Step 9: Update the interior stack after transmission scatter**

After `throughput *= weight;`, add:

```hlsl
        PathTracer::UpdateNestedDielectricsOnScatterTransmission(payload, lobe, interiorList);
```

Keep diffuse-bounce counting active only for non-transmission diffuse reflection:

```hlsl
        const bool isDiffuseBounce =
            ((lobe & kBSDFLobeDiffuseReflection) != 0u) ||
            (((lobe & kBSDFLobeTransmission) == 0u) && bsdfData.roughness > kSpecularRoughnessThreshold);
```

- [ ] **Step 10: Verify raygen R6 state**

Run:

```powershell
rg -n "nestedDielectricsQuality|InteriorList::make|rejectedDielectricHits|loadHomogeneousVolumeData|evalTransmittance|HandleNestedDielectrics|outsideIoR|UpdateNestedDielectricsOnScatterTransmission|kBSDFLobeTransmission|scatterOrigin" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "BeginDisabled\\(true\\).*Nested|transmission is deferred|single-sided shading" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: first command finds the full R6 raygen path. Second command has no stale disabled nested-dielectric UI or single-sided/deferred comments.

- [ ] **Step 11: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli
git -C DiligentSamples commit -m "feat(rtxpt): integrate nested dielectric path state" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 6: Add Stochastic Alpha-Blend Visibility

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit`

- [ ] **Step 1: Implement material helpers**

In `RTXPTMaterials.cpp`, add:

```cpp
bool RTXPTMaterialIsAlphaBlended(const GLTF::Material& Material, const RTXPTMaterialExtension* pExtension)
{
    const bool ExtensionTransmission =
        pExtension != nullptr && pExtension->Loaded &&
        (pExtension->EnableTransmission || pExtension->TransmissionFactor > 0.0f || pExtension->DiffuseTransmissionFactor > 0.0f);
    return Material.Attribs.AlphaMode == GLTF::Material::ALPHA_MODE_BLEND || ExtensionTransmission;
}

bool RTXPTMaterialNeedsAnyHit(const GLTF::Material&         Material,
                              const RTXPTMaterialExtension* pExtension,
                              bool                          HasBaseColorTexture)
{
    return RTXPTMaterialIsAlphaTested(Material, pExtension, HasBaseColorTexture) ||
        RTXPTMaterialIsAlphaBlended(Material, pExtension);
}
```

- [ ] **Step 2: Track any-hit geometry during AS build**

In `RTXPTAccelerationStructures.hpp`, add this stat after `AlphaTestedGeometryCount`:

```cpp
    Uint32      AlphaBlendedGeometryCount = 0;
```

In `RTXPTAccelerationStructures.cpp`, replace the `MaterialAlphaTested` vector with:

```cpp
        std::vector<Uint8> MaterialNeedsAnyHit(Model.Materials.size(), Uint8{0});
        std::vector<Uint8> MaterialAlphaTested(Model.Materials.size(), Uint8{0});
        std::vector<Uint8> MaterialAlphaBlended(Model.Materials.size(), Uint8{0});
```

Inside the material loop:

```cpp
            const bool IsAlphaTested = RTXPTMaterialIsAlphaTested(Material, pExtension, HasBaseColorTexture);
            const bool IsAlphaBlended = RTXPTMaterialIsAlphaBlended(Material, pExtension);
            MaterialAlphaTested[MatIdx]  = IsAlphaTested ? Uint8{1} : Uint8{0};
            MaterialAlphaBlended[MatIdx] = IsAlphaBlended ? Uint8{1} : Uint8{0};
            MaterialNeedsAnyHit[MatIdx]  = (IsAlphaTested || IsAlphaBlended) ? Uint8{1} : Uint8{0};
```

Replace geometry flag selection:

```cpp
                const bool GeometryNeedsAnyHit = Primitive.MaterialId < MaterialNeedsAnyHit.size() &&
                    MaterialNeedsAnyHit[Primitive.MaterialId] != 0;
                BuildData.Flags = GeometryNeedsAnyHit ? RAYTRACING_GEOMETRY_FLAG_NONE : RAYTRACING_GEOMETRY_FLAG_OPAQUE;
```

Keep the existing alpha-tested counter and add the alpha-blend counter:

```cpp
                const bool GeometryAlphaBlended = Primitive.MaterialId < MaterialAlphaBlended.size() &&
                    MaterialAlphaBlended[Primitive.MaterialId] != 0;
                if (GeometryAlphaBlended)
                    ++m_Stats.AlphaBlendedGeometryCount;
```

- [ ] **Step 3: Compile any-hit when alpha blend needs it**

In `RTXPTRayTracingPass.hpp`, add a `bool EnableAnyHit` parameter immediately after `EnableMaterialTextures`.

In `RTXPTRayTracingPass.cpp`, compute:

```cpp
    const bool UseAnyHit = FullPathTracer && EnableAnyHit;
```

Create `pAnyHit` when `UseAnyHit`, and add the hit group with `pAnyHit` when `UseAnyHit`.

Keep the material texture macro independent:

```cpp
    if (UseTextures)
        Macros.Add("ENABLE_MATERIAL_TEXTURES", 1);
```

At the end of initialization:

```cpp
    m_Stats.AnyHitEnabled = UseAnyHit;
```

In `RTXPTSample.cpp::CreatePhase4Passes`, pass:

```cpp
    const bool EnableAnyHit = EnableMaterialTextures || m_AccelerationStructures.GetStats().AlphaBlendedGeometryCount > 0;
```

- [ ] **Step 4: Add stochastic alpha blend in any-hit**

In `PathTracerAnyHit.rahit`, include the sampler helper:

```hlsl
#include "Utils/SampleGenerators.hlsli"
```

Replace the early-return alpha-test body with a shared any-hit body:

```hlsl
    const bool alphaTested  = (material.flags & kMaterialFlagAlphaTested) != 0u;
    const bool alphaBlended = (material.flags & kMaterialFlagAlphaBlend) != 0u;
    if (!alphaTested && !alphaBlended)
        return;

    GeometryVertexData V0;
    GeometryVertexData V1;
    GeometryVertexData V2;
    Bridge::getTriangleVertices(subInstance, PrimitiveIndex(), V0, V1, V2);
    const float2 texCoord = Bridge::interpolateTexCoord(V0, V1, V2, Attributes.barycentrics);

    if (alphaTested && !Bridge::alphaTestPasses(material, texCoord))
        IgnoreHit();

    if (alphaBlended)
    {
        const float alpha = Bridge::getBaseColor(material, texCoord).a;
        const uint seed = Hash32Combine(Hash32Combine(Hash32(PrimitiveIndex() + 0x2c9277b5u), DispatchRaysIndex().x),
                                        Hash32Combine(DispatchRaysIndex().y, g_Const.ptConsts.sampleIndex));
        if (Hash32ToFloat(seed) > saturate(alpha))
            IgnoreHit();
    }
```

- [ ] **Step 5: Verify any-hit alpha-blend path**

Run:

```powershell
rg -n "AlphaBlend|AlphaBlendedGeometryCount|EnableAnyHit|UseAnyHit|kMaterialFlagAlphaBlend|Hash32ToFloat|ALPHA_MODE_BLEND" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
rg -n "Phase 5\\.3|Honor ALPHA_MODE_BLEND" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: first command finds the new path. Second command has no stale open marker for blend-mode support.

- [ ] **Step 6: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit
git -C DiligentSamples commit -m "feat(rtxpt): add stochastic alpha blend visibility" -m "Co-Authored-By: GPT 5.5"
```

---

### Task 7: Mapping, Cleanup, and Verification

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: all R6-touched files

- [ ] **Step 1: Update mapping document**

In `RTXPT_FORK_MAPPING.md`, add an R6 section with these rows:

```markdown
| Diligent port | RTXPT-fork source | R6 notes |
|---|---|---|
| `Rendering/Materials/BxDF.hlsli` transmission helpers | `Rendering/Materials/BxDF.hlsli::SpecularReflectionTransmissionMicrofacet` | Rough dielectric reflection/transmission with Fresnel lobe selection and refraction Jacobian |
| `Rendering/Materials/LobeType.hlsli` | `Rendering/Materials/LobeType.hlsli` | Reflection/transmission lobe bit layout kept compatible with RTXPT-fork |
| `Rendering/Materials/InteriorList.hlsli` | `Rendering/Materials/InteriorList.hlsli` | Two-slot priority stack; raygen-local in Diligent instead of packed into RTXPT `PathState` |
| `PathTracerNestedDielectrics.hlsli` | `PathTracerNestedDielectrics.hlsli` | False-hit rejection, outside-IoR computation, and stack updates after transmission scatter |
| `Rendering/Volumes/HomogeneousVolumeSampler.hlsli` | `Rendering/Volumes/HomogeneousVolumeSampler.hlsli` | Absorption-only Beer-Lambert transmittance; no scattering in R6 |
| `MaterialPTData` R6 fields | `Materials/MaterialPT.h::PTMaterialData` | Diligent layout differs but carries transmission, IoR, nested priority, and attenuation data |
| `PathTracerClosestHit.rchit` two-sided payload | `PathTracerBridgeDonut.hlsli::loadSurface` | Diligent keeps closest-hit payload return style instead of RTXPT-fork `SurfaceData` |
| `PathTracerSample.rgen` interior-list loop | `PathTracer.hlsli::HandleHit` | Diligent raygen loop owns path state; rejected hits do not consume bounce count |
```

- [ ] **Step 2: Remove stale R6 open-work text**

Run:

```powershell
rg -n "Phase R6|Phase 5\\.3|transmission / nested dielectrics|ALPHA_MODE_BLEND|transmission is deferred|single-sided shading" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no stale user-visible roadmap text remains in `RTXPTSample.cpp`, no old any-hit blend marker remains, and no opaque-only comments remain in closest-hit/raygen. Mapping/history references may remain if they describe completed R6 behavior.

- [ ] **Step 3: Run source-level contract checks**

Run:

```powershell
rg -n "sizeof\\(MaterialPTData\\) == 144|offsetof\\(MaterialPTData, transmissionFactor\\) == 80|offsetof\\(MaterialPTData, volumeAttenuationColor\\) == 96|offsetof\\(MaterialPTData, nestedPriority\\) == 128" DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp
rg -n "PathPayload.*144 bytes|MaxPayloadSize.*36|nestedDielectricsQuality|kLobeTypeTransmission|SpecularTransmission|InteriorList|HomogeneousVolumeSampler|HandleNestedDielectrics|UpdateNestedDielectricsOnScatterTransmission" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: all material/payload contract checks and R6 shader symbols are present.

- [ ] **Step 4: Run whitespace checks**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTSceneGraph.hpp Samples/RTXPT/src/RTXPTSceneGraph.cpp Samples/RTXPT/src/RTXPTMaterials.hpp Samples/RTXPT/src/RTXPTMaterials.cpp Samples/RTXPT/src/RTXPTAccelerationStructures.hpp Samples/RTXPT/src/RTXPTAccelerationStructures.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit Samples/RTXPT/assets/shaders/PathTracer/PathTracerAnyHit.rahit Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli Samples/RTXPT/assets/shaders/PathTracer/PathTracerNestedDielectrics.hlsli Samples/RTXPT/CMakeLists.txt Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no output and exit code 0.

- [ ] **Step 5: Build verification when explicitly requested**

Do not auto-run this unless the user asks for build/runtime verification. When requested, run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: the RTXPT sample target builds. The C++ `static_assert`s confirm `MaterialPTData == 144`; the shader compiler accepts the larger payload and new headers.

- [ ] **Step 6: Manual GPU verification when explicitly requested**

Run the RTXPT sample on D3D12 and Vulkan. Acceptance checks:

- Opaque materials render the same as before R6 with `Nested Dielectrics = Off` and transmission factors at zero.
- A simple glass slab with `TransmissionFactor=1`, `IoR=1.5`, and low roughness refracts the background and casts/receives NEE through the transmission lobe.
- Rough glass broadens transmitted highlights and does not produce NaN/Inf pixels.
- Two nested dielectric volumes with different `NestedPriority` values carve correctly: the higher-priority medium wins where volumes overlap.
- `Nested Dielectrics = Fast` and `Quality` both avoid dark false-hit shells; Quality tolerates more rejected hits before termination.
- Volume attenuation tints transmitted paths according to `VolumeAttenuationColor` and path length; setting attenuation color to white restores untinted glass.
- Thin-surface transmissive materials refract without pushing an interior medium onto the stack and do not apply volume absorption.
- `ALPHA_MODE_BLEND` or a sidecar transmissive material with alpha below 1 uses stochastic any-hit; accumulated output converges without hard mask edges.
- D3D12 and Vulkan both launch and render valid images.

- [ ] **Step 7: Commit**

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): record R6 transmission mappings" -m "Co-Authored-By: GPT 5.5"
```

---

## Final Phase Verification

Run these source-level checks before calling the phase implementation source-complete:

```powershell
rg -n "transmissionFactor|diffuseTransmissionFactor|volumeAttenuation|nestedPriority|kLobeTypeTransmission|kBSDFLobeTransmission|evalFresnelDielectric|InteriorList|HandleNestedDielectrics|HomogeneousVolumeSampler|AlphaBlend|EnableAnyHit" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "Phase R6|Phase 5\\.3|transmission is deferred|single-sided shading|Honor ALPHA_MODE_BLEND" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
git -C DiligentSamples diff --check -- Samples/RTXPT
```

Expected:

- First command finds the R6 implementation in material data, BSDF, raygen state, any-hit, and mapping docs.
- Second command has no stale R6 open-work text in source/UI/shaders.
- `diff --check` reports no whitespace errors.

The phase is source-complete only after the direct checks pass. It is runtime-verified only after a requested build/run confirms D3D12 and Vulkan render valid images.

## Self-Review

- G10 coverage: rough dielectric specular reflection/transmission is covered by Task 3; nested dielectric priority handling is covered by Tasks 2 and 5; volume absorption is covered by Tasks 1, 2, and 5; `ALPHA_MODE_BLEND` stochastic transparency is covered by Task 6.
- RTXPT-fork alignment: every R6 algorithm has a reference anchor listed above, and Task 7 records the Diligent-vs-RTXPT mapping.
- Contract coverage: material layout, payload size, any-hit activation, frame constants, shader include registration, and UI enablement each have explicit tasks and verification commands.
- Scope control: no realtime-track features, no stable-plane changes, no volume scattering, no advanced lobes beyond the transmission/diffuse-transmission fields needed to mirror RTXPT-fork's standard BSDF.
