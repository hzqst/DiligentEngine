# KHR_materials_transmission Must Not Force AlphaMode BLEND

Status: **issue identified, proposed fix not yet applied**

Scope:

- DiligentTools glTF asset loading
- DiligentFX PBR / GLTFViewer transmission rendering
- RTXPT material classification and ray tracing geometry flags

## Summary

The DiligentFX glTF loader currently changes every material that carries
`KHR_materials_transmission` to `ALPHA_MODE_BLEND`.

This conflates two different glTF concepts:

- `alphaMode` describes alpha-as-coverage, i.e. whether the material covers the surface.
- `KHR_materials_transmission` describes physically-based transmission through a surface that
  still exists.

According to the Khronos extension documentation, `alphaMode` is not the mechanism for
physically-based transparency. When alpha-as-coverage is not being used, a transmissive material
should keep `alphaMode = "OPAQUE"` even though it is visually transparent.

Source:
<https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_materials_transmission/README.md>

## Current Behavior

The loader first reads the authored glTF `alphaMode`:

`DiligentTools/AssetLoader/src/GLTFLoader.cpp`

```cpp
auto alpha_mode_it = gltf_mat.additionalValues.find("alphaMode");
if (alpha_mode_it != gltf_mat.additionalValues.end())
{
    const tinygltf::Parameter& param = alpha_mode_it->second;
    if (param.string_value == "BLEND")
    {
        Mat.Attribs.AlphaMode = Material::ALPHA_MODE_BLEND;
    }
    if (param.string_value == "MASK")
    {
        Mat.Attribs.AlphaMode   = Material::ALPHA_MODE_MASK;
        Mat.Attribs.AlphaCutoff = 0.5f;
    }
}
```

Later, when `KHR_materials_transmission` is present, it overwrites that semantic alpha mode:

```cpp
auto ext_it = gltf_mat.extensions.find("KHR_materials_transmission");
if (ext_it != gltf_mat.extensions.end())
{
    Mat.Attribs.AlphaMode = Material::ALPHA_MODE_BLEND;

    Mat.Transmission = std::make_unique<Material::TransmissionShaderAttribs>();
    ...
}
```

After this assignment, downstream code cannot distinguish:

- an asset that explicitly requested `alphaMode = "BLEND"` for alpha-as-coverage;
- an opaque-coverage dielectric that merely has physical transmission.

## Why This Is Incorrect

`KHR_materials_transmission` is a BSDF / material-energy property. It is not a coverage property.
A glass pane with transmission still covers the triangle; rays should hit the surface and then be
reflected, refracted, absorbed, or scattered by the material model.

Forcing `ALPHA_MODE_BLEND` has several incorrect side effects:

1. It changes the loaded material semantics away from the authored glTF file.
2. It makes base-color alpha participate in alpha-blend paths even when the asset did not request
   alpha-as-coverage.
3. It routes transmissive surfaces through renderer logic intended for transparent coverage.
4. In RTXPT, it can make transmissive geometry non-opaque and route rays through stochastic
   alpha-blend any-hit handling, so the closest-hit transmission BSDF may rarely execute.

This is especially harmful for glass assets that use low opacity values in sidecar material data:
the stochastic any-hit path treats the value as a pass-through probability rather than a physical
transmission parameter.

## Affected Paths

### DiligentTools AssetLoader

`DiligentTools/AssetLoader/src/GLTFLoader.cpp` mutates `Mat.Attribs.AlphaMode` inside the
`KHR_materials_transmission` block.

The loader should preserve the authored glTF `alphaMode` and store transmission only in
`Mat.Transmission`.

### DiligentFX PBR / GLTFViewer

`DiligentFX/PBR/src/GLTF_PBR_Renderer.cpp` currently buckets draw calls by
`Material.Attribs.AlphaMode`:

```cpp
const int AlphaMode = Material.Attribs.AlphaMode;
m_RenderLists[AlphaMode].emplace_back(primitive, *pNode);
```

The PBR shader also uses `AlphaMode == BLEND` to decide whether to preserve alpha and enter the
transparent-composition path:

```hlsl
if (BasicAttribs.AlphaMode == PBR_ALPHA_MODE_BLEND)
{
    OutColor.rgb *= BaseColor.a;
    ...
}

if (BasicAttribs.AlphaMode != PBR_ALPHA_MODE_BLEND)
{
    OutColor.a = 1.0;
}
```

This explains why the loader may have forced `ALPHA_MODE_BLEND`: the raster renderer appears to
use the same field both for glTF alpha coverage and for transmission compositing. That is the core
design problem. The fix should split semantic alpha mode from renderer-side pass/composite mode.

`DiligentSamples/Samples/GLTFViewer/src/GLTFViewer.cpp` enables transmission through
`GLTF_PBR_Renderer::CreateInfo::EnableTransmission`. If the loader stops forcing `ALPHA_MODE_BLEND`,
GLTFViewer may need renderer-side changes so transmissive materials still render in the intended
transmission/composition path.

### RTXPT

RTXPT currently treats `Material.Attribs.AlphaMode == ALPHA_MODE_BLEND` as alpha-blended material
state. Because the loader forces this state for transmission, transmissive materials can become
alpha-blended even when the original glTF material was opaque coverage.

This affects:

- material flags;
- any-hit selection;
- ray tracing geometry opacity flags;
- whether the closest-hit transmission BSDF is consistently reached.

See also:
`DiligentSamples/Samples/RTXPT/docs/transmissive_materials_issue.md`

## Proposed Fix

### 1. Preserve Authored glTF AlphaMode in the Loader

Remove the forced assignment from the `KHR_materials_transmission` block:

```cpp
- Mat.Attribs.AlphaMode = Material::ALPHA_MODE_BLEND;
```

The extension block should only create and populate `Mat.Transmission`:

```cpp
Mat.Transmission = std::make_unique<Material::TransmissionShaderAttribs>();

const tinygltf::Value& TransExt = ext_it->second;
LoadExtensionTexture(gltf_model, *this, TransExt, MatBuilder, TransmissionTextureName);
LoadExtensionParameter(TransExt, "transmissionFactor", Mat.Transmission->Factor);
```

Expected loader behavior:

- No authored `alphaMode`, with `KHR_materials_transmission`: keep `ALPHA_MODE_OPAQUE`.
- Authored `alphaMode = "OPAQUE"`, with `KHR_materials_transmission`: keep `ALPHA_MODE_OPAQUE`.
- Authored `alphaMode = "MASK"`, with `KHR_materials_transmission`: keep `ALPHA_MODE_MASK`.
- Authored `alphaMode = "BLEND"`, with `KHR_materials_transmission`: keep `ALPHA_MODE_BLEND`.

### 2. Add Renderer-Side Transmission Classification

If DiligentFX needs transmissive materials to be rendered after opaque primitives, blended with the
background, or processed through OIT, that should be expressed as renderer state, not as glTF
`alphaMode`.

Introduce a derived render classification such as:

```cpp
enum class PBRRenderSurfaceClass
{
    Opaque,
    AlphaMask,
    AlphaBlendCoverage,
    TransmissionComposite
};
```

or add explicit material/render flags:

```cpp
RENDER_SURFACE_FLAG_ALPHA_BLEND_COVERAGE
RENDER_SURFACE_FLAG_TRANSMISSION_COMPOSITE
```

The derived classification can be computed from:

- authored `Material.Attribs.AlphaMode`;
- `Material.Transmission != nullptr`;
- `transmissionFactor > 0`;
- presence of a transmission texture;
- `GLTF_PBR_Renderer::CreateInfo::EnableTransmission`.

The important rule is:

```text
Material.Attribs.AlphaMode controls alpha-as-coverage only.
Renderer-side transmission classification controls transmission compositing only.
```

### 3. Update DiligentFX PBR Shader Inputs

Do not use `BasicAttribs.AlphaMode == PBR_ALPHA_MODE_BLEND` as the only way to keep
transmission alpha or enter the transmission compositing path.

Instead, pass an explicit flag to the shader, for example:

```hlsl
bool UseAlphaCoverageBlend;
bool UseTransmissionComposite;
```

or encode equivalent bits in the material/render attribs.

Then split the logic:

```hlsl
if (UseAlphaCoverageBlend)
{
    // Alpha-as-coverage behavior. BaseColor.a is coverage.
    OutColor.rgb *= BaseColor.a;
}

if (UseTransmissionComposite)
{
    // Physical transmission compositing. Transmission controls output alpha/transmittance.
    OutColor.a = 1.0 - Shading.Transmission;
}
else if (!UseAlphaCoverageBlend)
{
    OutColor.a = 1.0;
}
```

This keeps transmissive materials visually compatible in GLTFViewer without corrupting the glTF
semantic alpha mode.

### 4. Update GLTF_PBR_Renderer Render Lists

Replace direct bucketing by `Material.Attribs.AlphaMode` with bucketing by the derived render
surface class.

Possible draw order:

1. opaque coverage materials;
2. alpha-mask materials;
3. transmission-composite materials, if the selected raster path needs them after opaque;
4. authored alpha-blend coverage materials.

The exact order may depend on the current OIT path. The key requirement is that
`TransmissionComposite` must not require mutating `Material.Attribs.AlphaMode`.

### 5. Update GLTFViewer If Needed

`GLTFViewer` currently enables transmission through renderer create info:

```cpp
RendererCI.EnableTransmission = !m_pDevice->GetDeviceInfo().IsWebGPUDevice();
```

After the loader fix, verify that GLTFViewer still renders transmissive assets correctly. If not,
the viewer should rely on the renderer-side transmission classification above, rather than requiring
the loader to rewrite `alphaMode`.

Potential GLTFViewer-level changes:

- expose or forward a renderer option for transmission compositing;
- ensure debug view and feature toggles enable the new transmission-composite path;
- keep WebGPU texture-limit behavior unchanged unless the new implementation changes texture usage.

### 6. Make RTXPT Robust Against Loader Semantics

RTXPT should classify alpha blending from authored alpha-as-coverage, not from physical
transmission.

After the loader preserves authored `alphaMode`, RTXPT can keep using `ALPHA_MODE_BLEND` to mean
authored alpha coverage. It should not add alpha-blend behavior merely because
`kMaterialFlag_HasTransmission` is set.

Suggested RTXPT rule:

```text
Transmission selects the closest-hit BSDF lobes.
Alpha mask/blend coverage selects any-hit behavior.
```

For transmissive materials without authored alpha coverage:

- keep ray tracing geometry opaque;
- do not request stochastic alpha-blend any-hit;
- let the closest-hit BSDF handle reflection/refraction/transmission.

If a material explicitly combines `KHR_materials_transmission` with authored `alphaMode = "BLEND"`
or `"MASK"`, treat that as real alpha-as-coverage and preserve the corresponding any-hit behavior.

## Tests and Validation

### Loader Unit Tests

Add glTF loader tests for:

1. `KHR_materials_transmission` with no `alphaMode`:
   - expected `Mat.Attribs.AlphaMode == ALPHA_MODE_OPAQUE`;
   - expected `Mat.Transmission != nullptr`.
2. `KHR_materials_transmission` with `alphaMode = "OPAQUE"`:
   - expected `ALPHA_MODE_OPAQUE`.
3. `KHR_materials_transmission` with `alphaMode = "MASK"`:
   - expected `ALPHA_MODE_MASK`;
   - expected cutoff preserved.
4. `KHR_materials_transmission` with `alphaMode = "BLEND"`:
   - expected `ALPHA_MODE_BLEND`.

### GLTFViewer Smoke Tests

Use at least one public glTF asset with `KHR_materials_transmission` and no alpha coverage.

Expected:

- material remains semantically `OPAQUE`;
- transmission still renders through the DiligentFX transmission path;
- no regression in explicit alpha-blend or alpha-mask assets;
- transmission debug view still reports the expected factor/texture.

### RTXPT Validation

Use `assets/kitchen.scene.json` and the `glass-liquid-ice` materials described in
`transmissive_materials_issue.md`.

Expected:

- transmissive glass without authored alpha coverage builds as opaque ray tracing geometry;
- stochastic alpha-blend any-hit is not used for pure transmission;
- closest-hit BSDF executes consistently;
- glass refraction matches the upstream RTXPT-fork behavior more closely.

## Compatibility Notes

This change may alter visual output in GLTFViewer if the current renderer depends on
`ALPHA_MODE_BLEND` to make transmission visible. That dependency should be fixed in DiligentFX by
adding explicit transmission compositing state.

The loader should not keep the current behavior for backward compatibility because it changes the
meaning of loaded glTF material data. Compatibility should be handled at the renderer layer.

## Acceptance Criteria

- Loading `KHR_materials_transmission` no longer changes authored `alphaMode`.
- DiligentFX still has a renderer-side path for transmission compositing when enabled.
- GLTFViewer continues to show transmissive materials correctly.
- RTXPT no longer treats pure transmission as alpha-blended coverage.
- Explicit authored alpha mask/blend materials keep their existing behavior.
