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
- `DiligentSamples/Samples/RTXPT/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`:
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

### External Texture Override Of glTF Textures (Confirmed Behavior)

This is not limited to `convergence-test.scene.json`. The shipped scenes
**Bistro** (`bistro-programmer-art.scene.json`) and **ABeautifulGame** author
external texture-path objects (`BaseTexture`, `NormalTexture`,
`OcclusionRoughnessMetallicTexture`, etc.) in their `.material.json` files, and
their glTF models also carry embedded glTF textures. Today the external objects
are ignored, so these scenes render with glTF textures. After this change, a
material with both a glTF texture and an authored external texture for the same
slot switches to the external texture.

Decision: the override is the desired behavior. It matches RTXPT-fork, where the
`.material.json` is the authoritative material source. The external DDS paths
referenced by Bistro resolve under the Diligent assets root (verified:
`Models/Bistro/objects/.../*.dds` exist), so the override is expected to be
visually equivalent for those scenes while routing them through the external
texture path.

Because shipped scenes now depend on external loads, the override must be
fail-safe so it cannot regress a scene that previously rendered through glTF
textures:

- When an external texture object is authored for a slot, loads successfully,
  and the enable switch is on: the external binding replaces the glTF binding
  for that slot.
- When an external texture object is authored but its load fails, and the slot
  also has a valid glTF texture binding: keep the existing glTF binding for that
  slot instead of dropping to factor-only. Dropping to factor-only here would
  make a previously textured Bistro/ABeautifulGame material lose its texture,
  which is a regression. (This refines D7's failure handling, which otherwise
  clears the flag.)
- When an external texture object is authored, its load fails, and there is no
  glTF texture for that slot (the `convergence-test` case): clear the flag and
  use factor-only fallback, as in D7.

## Implementation Scope

In-scope implementation files:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.{hpp,cpp}`
- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`. Hosts the
  `.material.json` parse/load loop in `RTXPTScene::LoadScene`, which the
  `RTXPT_ENABLE_MATERIAL_EXTENSION` "loaded" gate wraps. See D0.
- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.{hpp,cpp}`
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp` and
  `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`. These classify materials
  (any-hit / alpha-test / alpha-blend / emissive-area-light) using glTF-only
  texture queries and must account for external material-JSON texture state. See
  D8.
- The single call site in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  if `RTXPTMaterials::Upload` needs the assets root as an explicit argument.
- `DiligentSamples/Samples/RTXPT/shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`
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

### D0 - Master Switch: `RTXPT_ENABLE_MATERIAL_EXTENSION`

Add a compile-time master switch that controls whether the entire RTXPT
`.material.json` material extension is loaded and used. Default: enabled (`1`).

Define it in `RTXPTSceneGraph.hpp`, which is visible to both `RTXPTScene.cpp`
(via `RTXPTScene.hpp`) and `RTXPTMaterials.cpp`:

```cpp
#ifndef RTXPT_ENABLE_MATERIAL_EXTENSION
#    define RTXPT_ENABLE_MATERIAL_EXTENSION 1
#endif
```

It can be overridden from CMake, e.g.
`target_compile_definitions(RTXPT PRIVATE RTXPT_ENABLE_MATERIAL_EXTENSION=0)`.

This gates the whole extension, not just the external textures added by this
spec. Semantics when set to `0`:

- The `.material.json` discovery/parse loop in `RTXPTScene::LoadScene` is
  compiled out, so no material JSON files are read (no extension file I/O). Each
  material's extension entry remains at its default state with `Loaded = false`.
- `RTXPTGetMaterialExtension` returns `nullptr`, so every consumer
  (`RTXPTMaterials::Upload`, the classification helpers) falls back to pure glTF
  material behavior — covering scalar overrides, enable switches, and the
  external material textures from this spec.

The gate is applied at two points so the intent is explicit: the parse/load loop
("loaded") and an early-out in `RTXPTGetMaterialExtension` ("used"). Because a
default extension already reports `Loaded = false`, the existing consumers
require no further changes for the switch to take effect.

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

Resource dimension is preverified, so the implementation keeps the current
`Texture2DArray` shader contract. `MaterialBridge.hlsli` declares:

```hlsl
Texture2DArray t_BindlessTextures[MATERIAL_TEXTURE_COUNT];
```

and `CreateMaterialTextureView` requests `RESOURCE_DIM_TEX_2D_ARRAY`. External
DDS/PNG textures load through `CreateTextureFromFile`, which creates
`RESOURCE_DIM_TEX_2D` resources (`TextureLoaderImpl.cpp`). Diligent's texture
view validation explicitly allows a `RESOURCE_DIM_TEX_2D_ARRAY` view over a
`RESOURCE_DIM_TEX_2D` resource (`TextureBase.cpp`, the `RESOURCE_DIM_TEX_2D`
case permits a `TEX_2D` or `TEX_2D_ARRAY` view). A single-slice 2D-array view of
the loaded 2D texture is therefore valid on D3D12 and Vulkan, matches what the
glTF path already produces, and preserves the `Texture2DArray` shader contract.

Required path: reuse the existing `CreateMaterialTextureView` helper for external
textures exactly as for glTF textures, and verify it returns non-null for loaded
external files. There is no need to change the shader's `Texture2DArray`
declaration. Do not migrate the bindless table to a `Texture2D` contract and do
not bind glTF and external textures as different resource dimensions inside the
same `t_BindlessTextures` array.

### D5 - Deduplicate External Texture Bindings

Add a deduplication map as a **local variable inside `RTXPTMaterials::Upload`**,
keyed by:

```text
normalized resolved path + sRGB + NormalMap -> bindless texture index
```

Keeping the map local to `Upload` (not a member) means it is naturally fresh on
every scene rebuild and needs no `Reset()` handling, which avoids reusing stale
bindless indices across scene reloads.

Deduplication prevents repeated material JSON files from loading the same DDS
multiple times. If the same normalized path is requested with conflicting
`sRGB` or `NormalMap` metadata, log a warning and keep the first loaded binding;
this mirrors RTXPT-fork's warning/assert behavior without crashing release
builds.

Lifetime is already covered by the existing `m_TextureViews` ownership and needs
no extra retain vector. `CreateMaterialTextureView` creates a non-default view
via `ITexture::CreateView`, and Diligent's `TextureViewBase` keeps a **strong
reference** to its texture for non-default views (`m_spTexture`,
`RefCntAutoPtr<ITexture>`). Storing the external SRV in the existing member
`m_TextureViews` therefore keeps the underlying `ITexture` alive on every
backend, exactly as it already does for glTF texture views. Do **not** add a
separate `std::vector<RefCntAutoPtr<ITexture>>`; reuse `m_TextureViews` and let
`Reset()` continue to release it.

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

`m_Stats.TextureCount` is currently assigned immediately after the glTF append
loop, before the per-material loop that step 5 uses to append external textures.
Move that assignment to **after** external textures are appended, so the stat
reflects the full bindless table (glTF + external). The runtime is not
functionally affected, because the shader macro `MATERIAL_TEXTURE_COUNT` and the
SRB binding both read the live `GetTextureCount()` (`m_TextureBindings.size()`),
not `m_Stats.TextureCount`; the move only keeps the diagnostic stat accurate.

### D7 - Apply External Texture Descriptors To `MaterialPTData`

For each texture slot:

- If the matching `Enable*Texture` switch is false, clear the matching texture
  flag and leave the index at zero.
- If the texture descriptor has no `path`, leave any existing glTF texture state
  untouched unless the enable switch cleared it.
- If the descriptor has a path and loading succeeds, write the bindless binding
  index, set the slice to `0.0f`, and set the matching flag (replacing any glTF
  binding for that slot).
- If the descriptor has a path but loading fails:
  - if the slot already has a valid glTF texture binding, keep that glTF binding
    (do not regress a previously textured shipped-scene material); see "External
    Texture Override Of glTF Textures";
  - otherwise clear the matching flag and use factor-only fallback.

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

Apply `NormalTextureScale` **only when the `NormalTextureScale` key is present
in the material JSON**. `Data.normalScale` is otherwise sourced from the glTF
material (`Attribs.NormalScale`) and is not touched by the scalar-override path.
Unconditionally writing the parsed value (e.g. defaulting to `1.0` when the key
is absent) would clobber the glTF-authored normal scale for every
extension-loaded material and regress glTF normal maps that use a non-1.0 scale.
Track presence explicitly (for example a `bool HasNormalTextureScale` set during
parse) and only assign `Data.normalScale` when it is true.

### D8 - Update Material Classification Helpers

The material-use classification does **not** live in `RTXPTMaterials.cpp`. It is
computed independently, using glTF-only texture queries, in two other files that
are now in scope for this change:

- `RTXPTAccelerationStructures.cpp` calls `RTXPTMaterialHasBaseColorTexture`,
  `RTXPTMaterialIsAlphaTested`, `RTXPTMaterialNeedsAnyHit`, and
  `RTXPTMaterialIsEmissiveAreaLight` to decide which geometry receives an
  any-hit shader and how alpha-test / alpha-blend flags are set.
- `RTXPTLights.cpp` calls `RTXPTMaterialIsEmissiveAreaLight` to decide emissive
  triangle / area-light extraction.

`RTXPTMaterialHasBaseColorTexture` inspects only `GLTF::Model::GetTexture(...)`,
so a material whose only base-color (or emissive) texture comes from a
`.material.json` object returns `false`. Without a fix, the GPU material buffer
(computed inside `Upload`, which already classifies from final flags) and the
acceleration-structure / lights classification disagree: an external-base-color
`ALPHA_MODE_MASK` material would be flagged alpha-tested in the material buffer
but its geometry would not get an any-hit shader, silently breaking the alpha
test. `convergence-test.scene.json` does not expose this because its floor is
opaque and non-emissive.

Any helper that currently checks only glTF texture IDs must account for final
external texture state:

- Base texture presence affects alpha testing and any-hit requirements.
- Emissive texture presence affects whether constant emissive color should be
  treated as an emissive area light.
- Transmission texture presence is meaningful only when the material has
  transmission enabled.

Required approach: the classification helpers must see the same resolved
external-texture presence that `Upload` uses. Because the final
`MaterialPTData::flags` are computed inside `RTXPTMaterials::Upload` and uploaded
to a GPU buffer, the resolved per-material texture state must be made available
to the acceleration-structure and lights builds. Two acceptable options:

1. Retain the CPU-side `std::vector<MaterialPTData>` (or just the final per-
   material `flags`) inside `RTXPTMaterials`, expose it through an accessor, and
   have `RTXPTAccelerationStructures.cpp` / `RTXPTLights.cpp` classify from the
   final flags instead of re-querying the glTF material.
2. Thread the resolved external-texture-presence bits (per material slot) from
   the scene-graph extension into the existing helpers so they can OR external
   presence with the glTF query.

Option 1 is preferred because it makes `Upload` the single source of truth for
material texture state and removes the duplicated glTF-only logic. Whichever
option is chosen, the same resolved state must drive both the GPU material
buffer and the any-hit / alpha-test / emissive-area-light classification, so
they cannot diverge.

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
  copy of a material JSON, using a slot that has no glTF texture (for example a
  `convergence-test` material) so factor-only fallback is the expected result.
- Verify load failure is reported, the sample does not crash, the affected
  texture flag is cleared, and scalar fallback renders.
- Separately, on a slot that does have a glTF texture (for example a Bistro
  material), point the external path to a missing file and confirm the glTF
  texture is retained rather than dropping to factor-only.
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
- Load the shipped scenes that author external texture paths on top of glTF
  textures (`bistro-programmer-art.scene.json` and an ABeautifulGame scene).
  Confirm they still render correctly through the external texture path, and that
  no material that was textured before the change becomes untextured
  (factor-only) after it.
- For a shipped-scene material whose external texture object exists but fails to
  load while a glTF texture is present for the same slot, confirm the glTF
  texture is retained (not dropped to factor-only), per "External Texture
  Override Of glTF Textures".
- Confirm any-hit / alpha-test / emissive-area-light classification is consistent
  for a material whose only base-color or emissive texture comes from a
  `.material.json` object (D8): the acceleration-structure any-hit decision and
  the material buffer flags must agree.

## Risks/Open Questions

- Resource dimension compatibility is resolved (see D4): `CreateTextureFromFile`
  creates `RESOURCE_DIM_TEX_2D` resources, and Diligent's texture view
  validation allows a `RESOURCE_DIM_TEX_2D_ARRAY` view over a
  `RESOURCE_DIM_TEX_2D` resource. External textures reuse the existing
  `CreateMaterialTextureView` helper and the current `Texture2DArray` shader
  contract is preserved. No `Texture2D` migration is needed. The implementation
  must still verify `CreateMaterialTextureView` returns non-null for each loaded
  external file.
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

- `RTXPT_ENABLE_MATERIAL_EXTENSION` is defined (default `1`) and, when set to
  `0`, disables loading and use of the entire `.material.json` material
  extension: no extension file I/O, `RTXPTGetMaterialExtension` returns
  `nullptr`, and rendering falls back to pure glTF material behavior.
- `RTXPTMaterialExtension` records parsed descriptors for `BaseTexture`,
  `OcclusionRoughnessMetallicTexture`, `NormalTexture`, `EmissiveTexture`, and
  `TransmissionTexture`, including `path`, `sRGB`, and `NormalMap`.
- `ParseRTXPTMaterialExtension` reads those texture objects and reads
  `EnableTransmissionTexture`.
- External texture paths are resolved relative to the RTXPT assets root, with
  slash normalization and PNG-to-DDS preference followed by authored-path
  fallback.
- External material textures are loaded through Diligent texture loading APIs,
  assigned SRVs via `CreateMaterialTextureView`, and appended to
  `RTXPTMaterials::m_TextureBindings`.
- The material bindless table keeps the existing `Texture2DArray` /
  `RESOURCE_DIM_TEX_2D_ARRAY` contract for both glTF and external textures; no
  resource-dimension migration is introduced.
- `MaterialPTData` texture indices and flags are correct for base color, ORM,
  normal, emissive, and transmission texture slots.
- `Enable*Texture` switches disable both glTF and external textures for their
  slots.
- Existing glTF material texture loading continues to work for scenes without
  external RTXPT material texture objects.
- Shipped scenes that author external texture paths on top of glTF textures
  (Bistro, ABeautifulGame) still render correctly, and no material that was
  textured before the change becomes factor-only because of it; an external load
  failure on a slot that has a glTF texture retains the glTF texture.
- Any-hit / alpha-test / emissive-area-light classification in
  `RTXPTAccelerationStructures.cpp` and `RTXPTLights.cpp` is consistent with the
  final material texture state (including external textures), so the
  acceleration structure and the GPU material buffer do not disagree.
- In `convergence-test.scene.json`, material texture count becomes non-zero and
  the floor wood texture from `Models/living_room/textures/wood4.dds` is visible
  in reference mode.
- Missing external texture paths produce a warning/fallback path, not a crash.
- The `RTXPT` target builds after the change.
