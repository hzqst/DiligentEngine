# RTXPT Material JSON Texture Loading Spec

## Summary

This spec defines the minimum Diligent-native fix for loading external texture
references authored in RTXPT `.material.json` files.

The immediate regression signal is `convergence-test.scene.json`: its
`ConvergenceTest.gltf` contains no `textures`, `images`, or `samplers` arrays,
but its RTXPT material JSON files contain external texture objects. For example,
`ConvergenceTest.Floor.material.json` references
`Models\living_room\textures\wood4.dds` through `BaseTexture.path`, and that DDS
file exists under the Diligent RTXPT assets root. Because the current Diligent
port only reads material scalar factors and enable switches, the floor loses its
wood texture even though the scene and material JSON are byte-identical to
`D:/RTXPT-fork/Assets`.

The target fix is to parse RTXPT material texture descriptors, load their
textures into the same material bindless table used by glTF textures, and write
the corresponding `MaterialPTData` texture indices and flags.

## Problem Evidence

- Current Diligent RTXPT sample path:
  `DiligentSamples/Samples/RTXPT`.
- Reference RTXPT-fork path:
  `D:/RTXPT-fork`.
- `DiligentSamples/Samples/RTXPT/assets/convergence-test.scene.json` matches
  `D:/RTXPT-fork/Assets/convergence-test.scene.json`.
- `DiligentSamples/Samples/RTXPT/assets/Materials/ConvergenceTest.Floor.material.json`
  matches `D:/RTXPT-fork/Assets/Materials/ConvergenceTest.Floor.material.json`.
- `DiligentSamples/Samples/RTXPT/assets/Models/ConvergenceTest/ConvergenceTest.gltf`
  has no `textures`, `images`, or `samplers` keys, so `GLTF::Model::GetTextureCount()`
  is zero for this asset.
- The floor material JSON contains:

```json
"BaseTexture": {
  "NormalMap": false,
  "path": "Models\\living_room\\textures\\wood4.dds",
  "sRGB": true
}
```

- `DiligentSamples/Samples/RTXPT/assets/Models/living_room/textures/wood4.dds`
  exists.
- Current Diligent material upload only appends SRVs from
  `GLTF::Model::GetTextureCount()`, so scene JSON external textures never enter
  `RTXPTMaterials::m_TextureBindings`.
- Current `MaterialPTData` texture indices are first populated from glTF texture
  IDs and then remapped through the glTF-only texture remap. For ConvergenceTest
  this leaves all material texture flags cleared.

## Source Anchors

Current Diligent anchors:

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`:
  `RTXPTScene::LoadScene` resolves `m_AssetsRoot`, loads glTF model assets,
  finds `.material.json` candidates under `AssetsRoot/Materials`, and stores
  parsed `RTXPTMaterialExtension` records in `RTXPTSceneGraphData`.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.{hpp,cpp}`:
  `RTXPTMaterialExtension` currently stores scalar factors and texture enable
  switches; `ParseRTXPTMaterialExtension` does not parse `BaseTexture`,
  `OcclusionRoughnessMetallicTexture`, `NormalTexture`, `EmissiveTexture`, or
  `TransmissionTexture` objects.
- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.{hpp,cpp}`:
  `FillMaterialPTDataFromGLTF`, `AppendTextureViews`, `RemapMaterialTextureIndices`,
  and `RTXPTMaterials::Upload(IRenderDevice*, const RTXPTSceneGraphData&)` build
  material GPU data and the bindless texture SRV table.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`:
  `t_BindlessTextures[MATERIAL_TEXTURE_COUNT]` is sampled through
  `MaterialPTData` texture indices when `ENABLE_MATERIAL_TEXTURES` is defined.
- `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`:
  `LoadSourceTexture` already uses `CreateTextureFromFile` with paths resolved
  relative to the RTXPT assets root.
- `DiligentTools/TextureLoader/interface/TextureLoader.h` and
  `DiligentTools/TextureLoader/interface/TextureUtilities.h`:
  `TextureLoadInfo::IsSRGB` and `CreateTextureFromFile` provide the existing
  Diligent texture loading path, including DDS support.

Reference RTXPT-fork anchors:

- `D:/RTXPT-fork/Rtxpt/Materials/MaterialsBaker.h`:
  `PTTexture` stores `LocalPath`, `sRGB`, `NormalMap`, and loaded texture state;
  `PTMaterial` owns `BaseTexture`, `OcclusionRoughnessMetallicTexture`,
  `NormalTexture`, `EmissiveTexture`, and `TransmissionTexture`, plus the
  matching enable switches.
- `D:/RTXPT-fork/Rtxpt/Materials/MaterialsBaker.cpp`:
  `PTMaterial::FromJson` calls `loadTexture(input, ..., "BaseTexture")` for all
  five texture slots, resolves the authored path against `mediaPath`, searches
  for a DDS replacement for PNG inputs, and loads through the texture cache with
  the authored `sRGB` flag.
- `D:/RTXPT-fork/Rtxpt/Materials/MaterialsBaker.cpp`:
  `PTMaterial::FillData` sets texture flags only when the texture is loaded and
  the matching enable switch is true, then calls `GetBindlessTextureIndex` to
  write GPU texture indices.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Materials/MaterialPT.h`:
  `PTMaterialData` contains texture index fields and flags for base/diffuse,
  metal-rough/specular, emissive, normal, and transmission textures.

## Current Diligent State

`RTXPTScene::LoadScene` already discovers material JSON files through candidate
paths rooted at `AssetsRoot/Materials`, and every glTF material receives an
entry in `RTXPTSceneGraphData::MaterialExtensions`.

`ParseRTXPTMaterialExtension` currently reads values such as
`BaseOrDiffuseColor`, `EmissiveColor`, `Metalness`, `Roughness`,
`TransmissionFactor`, `DiffuseTransmissionFactor`, volume parameters,
`EnableBaseTexture`, `EnableEmissiveTexture`, `EnableNormalTexture`,
`EnableOcclusionRoughnessMetallicTexture`, `EnableTransmission`,
`ThinSurface`, and `SkipRender`.

It does not read texture object fields:

- `BaseTexture.path`, `BaseTexture.sRGB`, `BaseTexture.NormalMap`
- `OcclusionRoughnessMetallicTexture.path`,
  `OcclusionRoughnessMetallicTexture.sRGB`,
  `OcclusionRoughnessMetallicTexture.NormalMap`
- `NormalTexture.path`, `NormalTexture.sRGB`, `NormalTexture.NormalMap`
- `EmissiveTexture.path`, `EmissiveTexture.sRGB`, `EmissiveTexture.NormalMap`
- `TransmissionTexture.path`, `TransmissionTexture.sRGB`,
  `TransmissionTexture.NormalMap`

`RTXPTMaterials::AppendTextureViews` appends one SRV per texture returned by the
glTF model. In scene-graph upload, it runs once per `RTXPTModelAsset` before any
material JSON extension is applied. It does not load or append external material
textures.

`MaterialBridge.hlsli` is already capable of sampling base color, emissive,
metallic-roughness, normal, transmission, and thickness textures when their
`MaterialPTData` flags and indices are valid. The missing piece is CPU-side
population of those flags and indices for `.material.json` texture objects.

## Reference Behavior

RTXPT-fork treats `.material.json` texture objects as first-class material
texture inputs:

1. `PTMaterial::FromJson` parses each texture object into a `PTTexture`.
2. Empty or missing texture objects leave that slot unloaded.
3. Non-empty `path` values are resolved relative to `mediaPath`, which maps to
   the RTXPT assets root.
4. PNG texture paths prefer a neighboring `.dds` file when it exists; otherwise
   the authored path is used.
5. The authored `sRGB` flag is passed to the texture loader.
6. `PTMaterial::FillData` sets each texture-use flag only when both conditions
   are true:
   the texture loaded successfully, and the matching `Enable*Texture` switch is
   enabled.
7. Transmission texture use is additionally gated by `EnableTransmission`.
8. The bindless descriptor index is written into the GPU material record; if the
   texture is unavailable, the flag is removed and the index is set invalid.

## Target Behavior

Diligent RTXPT must load external material JSON texture objects with the same
effective material semantics as RTXPT-fork while preserving the current glTF
texture path.

Required behavior:

- A `.material.json` texture object with a valid `path` creates a Diligent
  texture and SRV.
- The texture SRV is appended to `RTXPTMaterials::m_TextureBindings`, so
  `RTXPTMaterials::GetTextureCount()` includes both glTF textures and external
  material JSON textures.
- `MaterialPTData` texture indices refer to the final bindless binding index,
  not the original glTF texture ID or an unresolved local descriptor ID.
- `BaseTexture` drives `baseColorTextureIndex`, `baseColorTextureSlice`, and
  `kMaterialFlag_HasBaseColorTexture`.
- `OcclusionRoughnessMetallicTexture` drives
  `metallicRoughnessTextureIndex`, `metallicRoughnessTextureSlice`, and
  `kMaterialFlag_HasMetallicRoughnessTexture`. The current Diligent shader bridge
  samples roughness from `.g` and metallic from `.b`; occlusion remains unused
  unless a later shader change consumes it.
- `NormalTexture` drives `normalTextureIndex`, `normalTextureSlice`, and
  `kMaterialFlag_HasNormalTexture`.
- `EmissiveTexture` drives `emissiveTextureIndex`, `emissiveTextureSlice`, and
  `kMaterialFlag_HasEmissiveTexture`.
- `TransmissionTexture` drives `transmissionTextureIndex`,
  `transmissionTextureSlice`, and `kMaterialFlag_HasTransmissionTexture` only
  when transmission is enabled for the material.
- `EnableBaseTexture`, `EnableOcclusionRoughnessMetallicTexture`,
  `EnableNormalTexture`, `EnableEmissiveTexture`, `EnableTransmissionTexture`,
  and `EnableTransmission` are respected.
- Missing or failed external textures clear only the affected texture flag and
  leave scalar/factor fallback behavior active.
- If no external texture object is authored for a slot, the existing glTF texture
  path remains valid and is still controlled by the matching enable switch.
- Authored paths are resolved relative to the RTXPT assets root, not the material
  JSON file directory.
- Authored backslashes are normalized before file existence checks and loading.
- PNG paths prefer a neighboring `.dds` file when present, matching RTXPT-fork;
  paths that already point at `.dds` load directly; otherwise the authored path
  is used as the fallback.
- The authored `sRGB` flag is honored through `TextureLoadInfo::IsSRGB`.
  Color-like textures such as base and emissive should load as sRGB when the JSON
  says `true`; data textures such as ORM, normal, and transmission should remain
  linear when the JSON says `false`.

## Implementation Scope

In-scope implementation files:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.{hpp,cpp}`
- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.{hpp,cpp}`
- The single call site in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  if `RTXPTMaterials::Upload` needs the assets root as an explicit argument.
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`
  only if SRV dimension compatibility requires a shader resource declaration
  adjustment.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt` only if the existing RTXPT
  target does not already link the texture loader symbols required by
  `CreateTextureFromFile`.

Out-of-scope behavior:

- Rewriting the material shader model or BxDF implementation.
- Adding a Donut/NVRHI compatibility layer.
- Changing `.material.json` file format or asset content.
- Generating new textures.
- Changing scene JSON or glTF files.
- Adding occlusion shading behavior beyond the current metallic-roughness bridge
  unless a separate material-fidelity spec requires it.

## Detailed Design

### D1 - Add Material JSON Texture Descriptors

Add a small descriptor type near `RTXPTMaterialExtension`, for example:

```cpp
struct RTXPTMaterialTextureDesc
{
    std::string LocalPath;
    bool        HasPath   = false;
    bool        SRGB      = false;
    bool        NormalMap = false;
};
```

Extend `RTXPTMaterialExtension` with one descriptor per RTXPT material texture
slot:

- `BaseTexture`
- `OcclusionRoughnessMetallicTexture`
- `NormalTexture`
- `EmissiveTexture`
- `TransmissionTexture`

Also add `EnableTransmissionTexture`, because RTXPT-fork gates transmission
texture use separately from `EnableTransmission`.

The descriptor should store the authored local path exactly enough for reporting
and deduplication, but path resolution should happen later against the assets
root. This keeps `ParseRTXPTMaterialExtension` independent from global scene
state.

### D2 - Parse Texture Objects From `.material.json`

Add a helper in `RTXPTSceneGraph.cpp` that reads a named object:

```text
ReadMaterialTexture(Json, "BaseTexture") -> RTXPTMaterialTextureDesc
```

Parsing rules:

- Missing, null, or non-object texture values produce `HasPath = false`.
- Empty `path` produces `HasPath = false` and should add or log a warning when
  the surrounding code already has a warning channel available.
- `path` is stored as a string after slash normalization.
- `sRGB` defaults to `false` when absent.
- `NormalMap` defaults to `false` when absent.
- No texture file I/O happens during JSON parse.

Parse `EnableTransmissionTexture` with default `true`, matching RTXPT-fork.

### D3 - Resolve Assets-Root Relative Texture Paths

External texture loading needs the RTXPT assets root. The smallest interface
change is to pass it into material upload:

```cpp
bool RTXPTMaterials::Upload(IRenderDevice* pDevice,
                            const RTXPTSceneGraphData& SceneData,
                            const std::string& AssetsRoot);
```

`RTXPTSample::RebuildSceneDependentResources` can pass
`m_Scene.GetAssetsRoot()` or the sample's current `m_AssetsRoot`.

Resolution rules:

1. Start from `std::filesystem::path{AssetsRoot} / Desc.LocalPath`.
2. Normalize slashes with the same `FileSystem::CorrectSlashes` /
   `FileSystem::SimplifyPath` style already used by RTXPT loaders.
3. If the resolved extension is `.png`, check for the same path with `.dds`.
   Use the DDS path when it exists.
4. If the DDS replacement is not used, load the normalized authored path.

This matches RTXPT-fork's DDS preference without making DDS replacement a broad
filename guessing system.

### D4 - Load External Textures Through Diligent TextureLoader

Use the existing Diligent texture loader path:

```cpp
TextureLoadInfo LoadInfo{"RTXPT material texture"};
LoadInfo.IsSRGB = Desc.SRGB;
CreateTextureFromFile(ResolvedPath.c_str(), LoadInfo, pDevice, &Texture);
```

Create an SRV for the loaded texture and append it to the same bindless material
texture table as glTF texture SRVs.

The implementation must not mix incompatible resource types in one HLSL bindless
array. `MaterialBridge.hlsli` currently declares:

```hlsl
Texture2DArray t_BindlessTextures[MATERIAL_TEXTURE_COUNT];
```

and `CreateMaterialTextureView` currently requests
`RESOURCE_DIM_TEX_2D_ARRAY`. The implementation must therefore choose one
consistent path:

- Preferred path: preserve the current `Texture2DArray` shader contract by
  creating external SRVs that are valid as single-slice `RESOURCE_DIM_TEX_2D_ARRAY`
  views, and verify `CreateMaterialTextureView` returns non-null for loaded
  external files.
- Fallback path: if Diligent `CreateTextureFromFile` cannot expose external
  textures as `Texture2DArray`, change the entire material bindless table to a
  `Texture2D` contract only after confirming glTF textures can also be exposed
  through compatible 2D SRVs. In that case, `baseColorTextureSlice` and related
  slice fields remain zero and are ignored by the shader bridge.

Do not bind glTF textures as one resource type and external textures as another
inside the same `t_BindlessTextures` array.

### D5 - Deduplicate External Texture Bindings

Add an internal map in `RTXPTMaterials::Upload` or `RTXPTMaterials`:

```text
normalized resolved path + sRGB + NormalMap -> bindless texture index
```

Deduplication prevents repeated material JSON files from loading the same DDS
multiple times. If the same normalized path is requested with conflicting
`sRGB` or `NormalMap` metadata, log a warning and keep the first loaded binding;
this mirrors RTXPT-fork's warning/assert behavior without crashing release
builds.

The lifetime model should match current `m_TextureViews` ownership. If an SRV
does not keep the underlying `ITexture` alive on every backend, add an internal
`std::vector<RefCntAutoPtr<ITexture>>` for externally loaded textures.

### D6 - Preserve glTF Texture Remapping

Keep the existing upload order:

1. Append and remap glTF texture views for each model asset.
2. Fill `MaterialPTData` from the glTF material.
3. Remap glTF texture indices with `RemapMaterialTextureIndices`.
4. Apply scalar `.material.json` overrides.
5. Apply external `.material.json` texture descriptors.
6. Recompute alpha-test, alpha-blend, emissive-area-light, transmission, and
   volume flags that depend on the final material texture state.

This preserves the built-in glTF texture path for scenes that do not author
external RTXPT texture objects.

### D7 - Apply External Texture Descriptors To `MaterialPTData`

For each texture slot:

- If the matching `Enable*Texture` switch is false, clear the matching texture
  flag and leave the index at zero.
- If the texture descriptor has no `path`, leave any existing glTF texture state
  untouched unless the enable switch cleared it.
- If the descriptor has a path and loading succeeds, write the bindless binding
  index, set the slice to `0.0f`, and set the matching flag.
- If the descriptor has a path but loading fails, clear the matching flag and
  use factor-only fallback.

Slot mapping:

- `BaseTexture` -> `baseColorTextureIndex`, `baseColorTextureSlice`,
  `kMaterialFlag_HasBaseColorTexture`.
- `OcclusionRoughnessMetallicTexture` ->
  `metallicRoughnessTextureIndex`, `metallicRoughnessTextureSlice`,
  `kMaterialFlag_HasMetallicRoughnessTexture`.
- `NormalTexture` -> `normalTextureIndex`, `normalTextureSlice`,
  `kMaterialFlag_HasNormalTexture`.
- `EmissiveTexture` -> `emissiveTextureIndex`, `emissiveTextureSlice`,
  `kMaterialFlag_HasEmissiveTexture`.
- `TransmissionTexture` -> `transmissionTextureIndex`,
  `transmissionTextureSlice`, `kMaterialFlag_HasTransmissionTexture`, gated by
  both `EnableTransmissionTexture` and effective transmission enablement.

If `NormalTextureScale` is not already represented in
`RTXPTMaterialExtension`, parse and apply it to `Data.normalScale` as a small
adjacent compatibility fix; normal map presence without the authored scale can
change appearance.

### D8 - Update Material Classification Helpers

Any helper that currently checks only glTF texture IDs must account for final
external texture state:

- Base texture presence affects alpha testing and any-hit requirements.
- Emissive texture presence affects whether constant emissive color should be
  treated as an emissive area light.
- Transmission texture presence is meaningful only when the material has
  transmission enabled.

The safest approach is to compute these classifications from the final
`MaterialPTData::flags` after external texture descriptors have been applied,
instead of re-querying only the original glTF material.

## Testing/Validation

Build validation:

- Configure/build target `RTXPT` for the active backend configuration used by
  the workspace.
- If available, build both D3D12 and Vulkan RTXPT sample variants because the
  bindless material table and texture view dimension handling must work on both.

Focused runtime validation:

- Load `convergence-test.scene.json`.
- Confirm `RTXPTMaterials::GetTextureCount()` is greater than zero even though
  `ConvergenceTest.gltf` has no glTF textures.
- Confirm the floor material's `MaterialPTData` has
  `kMaterialFlag_HasBaseColorTexture` set and a valid
  `baseColorTextureIndex`.
- Confirm the floor samples
  `Models/living_room/textures/wood4.dds` and visibly restores the wood floor
  texture.
- Confirm `MATERIAL_TEXTURE_COUNT` matches the runtime material texture binding
  count and `ENABLE_MATERIAL_TEXTURES` is enabled for the path-tracing shaders
  when count is non-zero.
- Run reference mode on `convergence-test.scene.json` and visually compare
  against RTXPT-fork or a known-good screenshot for floor texture presence and
  broadly matching material response.

Negative validation:

- Temporarily point one material texture path to a missing file in a local test
  copy of a material JSON.
- Verify load failure is reported, the sample does not crash, the affected
  texture flag is cleared, and scalar fallback renders.
- Restore the material JSON after the local negative test.

Regression validation:

- Load a glTF scene that uses built-in glTF textures and no external RTXPT
  texture objects.
- Confirm existing glTF textures still render and their indices still remap
  through the bindless table.
- Load a material where an external texture object is absent but the matching
  enable switch is true; confirm the existing glTF texture remains active.
- Load a material where the matching enable switch is false; confirm both glTF
  and external texture paths are disabled for that slot.

## Risks/Open Questions

- Resource dimension compatibility is the highest-risk detail. The current
  shader uses `Texture2DArray`, and the existing Diligent material SRV helper
  requests `RESOURCE_DIM_TEX_2D_ARRAY`. The implementation must prove external
  DDS textures can be exposed through the same SRV dimension or deliberately
  migrate all material texture bindings to a consistent `Texture2D` shader
  contract.
- `TextureLoadInfo::IsSRGB` should be verified with DDS inputs. If the DDS file
  already encodes an explicit sRGB format, the loader may derive the final format
  from file metadata; if it is sRGB-agnostic, the authored JSON flag must control
  the final view format.
- Some `.material.json` files may reference the same path with conflicting
  `sRGB` or `NormalMap` metadata. The implementation should warn and keep
  deterministic behavior.
- The current Diligent shader bridge does not consume an independent occlusion
  texture channel. This spec maps RTXPT's `OcclusionRoughnessMetallicTexture` to
  the existing metallic-roughness field and leaves occlusion shading fidelity to
  a separate material/BxDF parity task.
- If linking `CreateTextureFromFile` from `RTXPTMaterials.cpp` exposes a missing
  direct dependency on `Diligent-TextureLoader`, add that link explicitly to the
  RTXPT target rather than relying on transitive linkage.

## Acceptance Criteria

- `RTXPTMaterialExtension` records parsed descriptors for `BaseTexture`,
  `OcclusionRoughnessMetallicTexture`, `NormalTexture`, `EmissiveTexture`, and
  `TransmissionTexture`, including `path`, `sRGB`, and `NormalMap`.
- `ParseRTXPTMaterialExtension` reads those texture objects and reads
  `EnableTransmissionTexture`.
- External texture paths are resolved relative to the RTXPT assets root, with
  slash normalization and PNG-to-DDS preference followed by authored-path
  fallback.
- External material textures are loaded through Diligent texture loading APIs,
  assigned SRVs, and appended to `RTXPTMaterials::m_TextureBindings`.
- The material bindless shader resource declaration and all bound SRVs use a
  consistent texture dimension contract.
- `MaterialPTData` texture indices and flags are correct for base color, ORM,
  normal, emissive, and transmission texture slots.
- `Enable*Texture` switches disable both glTF and external textures for their
  slots.
- Existing glTF material texture loading continues to work for scenes without
  external RTXPT material texture objects.
- In `convergence-test.scene.json`, material texture count becomes non-zero and
  the floor wood texture from `Models/living_room/textures/wood4.dds` is visible
  in reference mode.
- Missing external texture paths produce a warning/fallback path, not a crash.
- The `RTXPT` target builds after the change.
