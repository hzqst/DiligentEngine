# RTXPT Material JSON Texture Loading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Load external texture references authored in RTXPT `.material.json` files (e.g. `convergence-test`'s floor `wood4.dds`) into the same bindless material-texture table used by glTF textures, and write the matching `MaterialPTData` indices/flags.

**Architecture:** Parse the five RTXPT material texture descriptor objects during scene-graph parsing (no I/O). During `RTXPTMaterials::Upload`, after the existing glTF fill + remap + scalar-override passes, resolve each authored path against the assets root, load it through Diligent's `CreateTextureFromFile`, append a single-slice `Texture2DArray`-compatible SRV to the existing bindless table (with per-path dedup), and write the bindless index/flag into `MaterialPTData`. External textures override the glTF binding for the same slot but fall back to the glTF binding if the external load fails. Material classification helpers (any-hit / alpha-test / emissive-area-light) are updated to account for external textures so the GPU material buffer and the acceleration-structure / lights builds stay consistent.

**Tech Stack:** C++17, Diligent Engine (`Diligent-TextureLoader` via `CreateTextureFromFile`, `FileSystem`, `RefCntAutoPtr`), nlohmann::json, CMake/Visual Studio.

**Source spec:** `docs/superpowers/specs/2026-06-05-rtxpt-material-json-texture-loading-spec.md`

---

## Conventions For This Plan

- **No unit-test harness exists for the RTXPT sample.** Per-task verification is a compile of the `RTXPT` target; end-to-end verification is the runtime/visual validation in Task 7 (mirrors the spec's Testing/Validation section).
- **Build command** (bash on Windows, build dir already configured): `cmake --build build/x64/Debug --config Debug --target RTXPT`
- Per the user's project convention, **the executor must not run builds autonomously** — at each "verify it compiles" step, pause and let the user trigger the build, then continue once it is green. Treat the build command as the verification the user runs.
- **Copyright headers already read `Copyright 2026`; today is 2026, so no header date changes are required.**
- All five RTXPT material texture slots and their enable switches:
  | JSON object | enable switch | `MaterialPTData` index / slice | flag |
  |---|---|---|---|
  | `BaseTexture` | `EnableBaseTexture` | `baseColorTextureIndex` / `baseColorTextureSlice` | `kMaterialFlag_HasBaseColorTexture` |
  | `OcclusionRoughnessMetallicTexture` | `EnableOcclusionRoughnessMetallicTexture` | `metallicRoughnessTextureIndex` / `metallicRoughnessTextureSlice` | `kMaterialFlag_HasMetallicRoughnessTexture` |
  | `NormalTexture` | `EnableNormalTexture` | `normalTextureIndex` / `normalTextureSlice` | `kMaterialFlag_HasNormalTexture` |
  | `EmissiveTexture` | `EnableEmissiveTexture` | `emissiveTextureIndex` / `emissiveTextureSlice` | `kMaterialFlag_HasEmissiveTexture` |
  | `TransmissionTexture` | `EnableTransmissionTexture` + effective transmission | `transmissionTextureIndex` / `transmissionTextureSlice` | `kMaterialFlag_HasTransmissionTexture` |

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp` | Scene-graph data types + master switch | Define `RTXPT_ENABLE_MATERIAL_EXTENSION` (default 1); add `RTXPTMaterialTextureDesc`; extend `RTXPTMaterialExtension` with 5 descriptors + `EnableTransmissionTexture` + `NormalTextureScale`/`HasNormalTextureScale`. |
| `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp` | Material JSON parsing | Add `ReadMaterialTexture` helper; parse the 5 texture objects, `EnableTransmissionTexture`, `NormalTextureScale`. |
| `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp` | `.material.json` load | Gate the parse/load loop in `RTXPTScene::LoadScene` with `RTXPT_ENABLE_MATERIAL_EXTENSION` ("loaded"). |
| `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` | Material GPU upload interface | Add `AssetsRoot` param to scene-graph `Upload`; update bindless-table doc comment. |
| `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` | Material GPU upload + classification helpers | Add external-texture resolve/load/dedup; apply descriptors to `MaterialPTData`; move `m_Stats.TextureCount`; update `RTXPTMaterialHasBaseColorTexture` and `RTXPTMaterialIsEmissiveAreaLight` (D8). |
| `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` | Upload call site | Pass `m_Scene.GetAssetsRoot()` to `Upload`. |
| `DiligentSamples/Samples/RTXPT/CMakeLists.txt` | Link config | Verify only; add explicit `Diligent-TextureLoader` link **only if** the build fails to resolve `CreateTextureFromFile`. |

---

## Task 1: Add Master Switch, Descriptor Types, And Extension Fields

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp` (add macro after the include block ~line 16; insert struct before `struct RTXPTMaterialExtension` ~line 71; extend that struct ~lines 79-104)

- [ ] **Step 1: Define the `RTXPT_ENABLE_MATERIAL_EXTENSION` master switch**

Immediately after the include block and before the `namespace Diligent` opening (or just inside it, before `using RTXPTSceneId = Uint32;`), add:

```cpp
// Master switch for the RTXPT .material.json material extension (scalar overrides, enable switches, and
// external material textures). Define as 0 (e.g. via CMake target_compile_definitions) to ignore
// .material.json entirely and render with pure glTF material behavior. Default: enabled.
#ifndef RTXPT_ENABLE_MATERIAL_EXTENSION
#    define RTXPT_ENABLE_MATERIAL_EXTENSION 1
#endif
```

- [ ] **Step 2: Add the `RTXPTMaterialTextureDesc` struct**

Insert this immediately before the `struct RTXPTMaterialExtension` declaration:

```cpp
// One RTXPT .material.json texture object (BaseTexture, NormalTexture, etc.). Path is stored verbatim after
// slash normalization; resolution against the assets root happens later in RTXPTMaterials::Upload.
struct RTXPTMaterialTextureDesc
{
    std::string LocalPath;
    bool        HasPath   = false;
    bool        SRGB      = false;
    bool        NormalMap = false;
};
```

- [ ] **Step 3: Extend `RTXPTMaterialExtension` with texture descriptors and new scalars**

In `struct RTXPTMaterialExtension`, immediately after the existing `bool SkipRender = false;` line and before the struct's closing `};`, add:

```cpp
    bool                     EnableTransmissionTexture = true;
    float                    NormalTextureScale        = 1.0f;
    bool                     HasNormalTextureScale     = false;
    RTXPTMaterialTextureDesc BaseTexture;
    RTXPTMaterialTextureDesc OcclusionRoughnessMetallicTexture;
    RTXPTMaterialTextureDesc NormalTexture;
    RTXPTMaterialTextureDesc EmissiveTexture;
    RTXPTMaterialTextureDesc TransmissionTexture;
```

- [ ] **Step 4: Verify it compiles**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds with no errors (header-only change; macro defaults to 1; no behavior change yet).

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.hpp
git commit -m "feat(rtxpt): add material extension master switch and texture descriptor types"
```

---

## Task 2: Parse Texture Objects From `.material.json`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp` (add helper before `ParseRTXPTMaterialExtension` ~line 38; add parsing before `return Ext;` ~line 94)

- [ ] **Step 1: Add the `ReadMaterialTexture` file-local helper**

Insert this `static` function immediately before `RTXPTMaterialExtension ParseRTXPTMaterialExtension(...)`:

```cpp
// Reads a named RTXPT material texture object. Missing, null, non-object, or empty-path values yield
// HasPath = false. The authored path is stored with backslashes normalized to forward slashes; no file
// I/O happens here.
static RTXPTMaterialTextureDesc ReadMaterialTexture(const nlohmann::json& Json, const char* Key)
{
    RTXPTMaterialTextureDesc Desc;

    const auto It = Json.find(Key);
    if (It == Json.end() || !It->is_object())
        return Desc;

    const std::string Path = ReadRTXPTOptionalString(*It, "path", "");
    if (Path.empty())
        return Desc;

    Desc.LocalPath = Path;
    std::replace(Desc.LocalPath.begin(), Desc.LocalPath.end(), '\\', '/');
    Desc.HasPath   = true;
    Desc.SRGB      = It->value("sRGB", false);
    Desc.NormalMap = It->value("NormalMap", false);
    return Desc;
}
```

(`<algorithm>` for `std::replace` and `RTXPTSceneJson.hpp` for `ReadRTXPTOptionalString` are already included in this file.)

- [ ] **Step 2: Parse the texture objects and new scalars in `ParseRTXPTMaterialExtension`**

In `ParseRTXPTMaterialExtension`, immediately before the final `return Ext;`, add:

```cpp
    Ext.EnableTransmissionTexture = Json.value("EnableTransmissionTexture", Ext.EnableTransmissionTexture);

    Ext.BaseTexture                       = ReadMaterialTexture(Json, "BaseTexture");
    Ext.OcclusionRoughnessMetallicTexture = ReadMaterialTexture(Json, "OcclusionRoughnessMetallicTexture");
    Ext.NormalTexture                     = ReadMaterialTexture(Json, "NormalTexture");
    Ext.EmissiveTexture                   = ReadMaterialTexture(Json, "EmissiveTexture");
    Ext.TransmissionTexture               = ReadMaterialTexture(Json, "TransmissionTexture");

    if (Json.contains("NormalTextureScale"))
    {
        Ext.NormalTextureScale    = ReadRTXPTOptionalFloat(Json, "NormalTextureScale", Ext.NormalTextureScale);
        Ext.HasNormalTextureScale = true;
    }
```

- [ ] **Step 3: Verify it compiles**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds with no errors. Parsed descriptors are populated but not yet consumed.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSceneGraph.cpp
git commit -m "feat(rtxpt): parse material JSON texture objects and NormalTextureScale"
```

---

## Task 3: Load, Dedup, And Apply External Textures In `RTXPTMaterials::Upload`

This is the core task. It plumbs the assets root through `Upload`, adds the resolve/load/dedup helper, applies descriptors to `MaterialPTData` (with the glTF fail-safe), moves `m_Stats.TextureCount`, and updates the bindless-table doc comment.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp` (`Upload` signature ~line 193; doc comment ~lines 198-202)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` (includes ~lines 32-34; anonymous namespace ~before line 177; scene-graph `Upload` ~lines 334-448)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` (call site line 689)

- [ ] **Step 1: Add includes to `RTXPTMaterials.cpp`**

Replace the existing standard-library include block:

```cpp
#include <algorithm>
#include <utility>
#include <vector>
```

with:

```cpp
#include "FileSystem.hpp"
#include "TextureUtilities.h"

#include <algorithm>
#include <cctype>
#include <filesystem>
#include <unordered_map>
#include <utility>
#include <vector>
```

(`LOG_WARNING_MESSAGE` is already available via the existing `#include "DebugUtilities.hpp"`.)

- [ ] **Step 2: Update the scene-graph `Upload` declaration in `RTXPTMaterials.hpp`**

Change:

```cpp
    bool Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData);
```

to:

```cpp
    bool Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData, const std::string& AssetsRoot);
```

Add `#include <string>` to the `RTXPTMaterials.hpp` include block (alongside `#include <vector>`) so the `std::string&` parameter type is available.

- [ ] **Step 3: Update the bindless-table doc comment in `RTXPTMaterials.hpp`**

Replace the comment above `GetTextureCount()`:

```cpp
    // Bindless material-texture table. Indices match GLTF::Model texture indices and are referenced by
    // MaterialPTData::baseColorTextureIndex / emissiveTextureIndex. The SRV views are owned here and keep
    // the underlying texture resources alive.
```

with:

```cpp
    // Bindless material-texture table holding one SRV per glTF texture and per external .material.json
    // texture. Indices are referenced by MaterialPTData texture-index fields. The SRV views are owned here
    // and (being non-default views) keep the underlying texture resources alive on every backend.
```

- [ ] **Step 4: Add resolve/load/dedup helpers to the anonymous namespace in `RTXPTMaterials.cpp`**

Inside the existing `namespace { ... }` block (the one that already contains `CreateMaterialTextureView`, `InvalidTextureIndex`, and `RemapMaterialTextureIndices`), add the following just before the block's closing `} // namespace`:

```cpp
// Resolves an authored material-texture path against the assets root, normalizing slashes and preferring a
// neighboring .dds when a .png is authored (matching RTXPT-fork).
std::string ResolveExternalTexturePath(const std::string& AssetsRoot, const std::string& LocalPath)
{
    std::string Resolved = (std::filesystem::path{AssetsRoot} / LocalPath).string();
    FileSystem::CorrectSlashes(Resolved);
    Resolved = FileSystem::SimplifyPath(Resolved.c_str());

    std::filesystem::path PathObj{Resolved};
    std::string           Ext = PathObj.extension().string();
    std::transform(Ext.begin(), Ext.end(), Ext.begin(),
                   [](unsigned char c) { return static_cast<char>(std::tolower(c)); });
    if (Ext == ".png")
    {
        std::string DdsCandidate = PathObj.replace_extension(".dds").string();
        FileSystem::CorrectSlashes(DdsCandidate);
        DdsCandidate = FileSystem::SimplifyPath(DdsCandidate.c_str());
        if (FileSystem::FileExists(DdsCandidate.c_str()))
            return DdsCandidate;
    }
    return Resolved;
}

struct ExternalTextureBinding
{
    Uint32 Index;
    bool   SRGB;
    bool   NormalMap;
};
using ExternalTextureCache = std::unordered_map<std::string, ExternalTextureBinding>;

// Loads (or reuses) an external material texture and appends its SRV to the bindless table. Returns the
// bindless index, or InvalidTextureIndex on failure. Deduplicates by resolved path; conflicting sRGB/NormalMap
// metadata for the same path warns and keeps the first binding.
Uint32 AppendExternalTexture(IRenderDevice*                            pDevice,
                             const RTXPTMaterialTextureDesc&           Desc,
                             const std::string&                        AssetsRoot,
                             std::vector<RefCntAutoPtr<ITextureView>>& TextureViews,
                             std::vector<IDeviceObject*>&              TextureBindings,
                             ExternalTextureCache&                     Cache)
{
    const std::string ResolvedPath = ResolveExternalTexturePath(AssetsRoot, Desc.LocalPath);

    const auto CacheIt = Cache.find(ResolvedPath);
    if (CacheIt != Cache.end())
    {
        if (CacheIt->second.SRGB != Desc.SRGB || CacheIt->second.NormalMap != Desc.NormalMap)
            LOG_WARNING_MESSAGE("RTXPT material texture '", ResolvedPath,
                                "' requested with conflicting sRGB/NormalMap metadata; keeping the first binding");
        return CacheIt->second.Index;
    }

    TextureLoadInfo LoadInfo{"RTXPT material texture"};
    LoadInfo.IsSRGB = Desc.SRGB;

    RefCntAutoPtr<ITexture> pTexture;
    CreateTextureFromFile(ResolvedPath.c_str(), LoadInfo, pDevice, &pTexture);
    if (!pTexture)
    {
        LOG_WARNING_MESSAGE("Failed to load RTXPT material texture: ", ResolvedPath);
        return InvalidTextureIndex;
    }

    RefCntAutoPtr<ITextureView> pSRV = CreateMaterialTextureView(pTexture);
    if (!pSRV)
    {
        LOG_WARNING_MESSAGE("Failed to create SRV for RTXPT material texture: ", ResolvedPath);
        return InvalidTextureIndex;
    }

    const Uint32 BindingIndex = static_cast<Uint32>(TextureBindings.size());
    TextureViews.emplace_back(std::move(pSRV));
    TextureBindings.push_back(TextureViews.back().RawPtr<IDeviceObject>());
    Cache.emplace(ResolvedPath, ExternalTextureBinding{BindingIndex, Desc.SRGB, Desc.NormalMap});
    return BindingIndex;
}
```

- [ ] **Step 5: Update the scene-graph `Upload` definition signature in `RTXPTMaterials.cpp`**

Change:

```cpp
bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData)
{
```

to:

```cpp
bool RTXPTMaterials::Upload(IRenderDevice* pDevice, const RTXPTSceneGraphData& SceneData, const std::string& AssetsRoot)
{
```

- [ ] **Step 6: Move `m_Stats.TextureCount` and declare the dedup cache**

In this `Upload`, the glTF append loop is followed by:

```cpp
    m_Stats.TextureCount = static_cast<Uint32>(m_TextureBindings.size());

    std::vector<MaterialPTData> MaterialData;
```

Replace those two lines with (drop the premature stat assignment, add the cache):

```cpp
    ExternalTextureCache ExternalCache;

    std::vector<MaterialPTData> MaterialData;
```

- [ ] **Step 7: Apply external texture descriptors inside the extension block**

Inside the same `Upload`, find the end of the `if (pExtension != nullptr && pExtension->Loaded)` block — the last statements are the enable-switch clears:

```cpp
                if (!Ext.EnableBaseTexture)
                    Data.flags &= ~kMaterialFlag_HasBaseColorTexture;
                if (!Ext.EnableEmissiveTexture)
                    Data.flags &= ~kMaterialFlag_HasEmissiveTexture;
                if (!Ext.EnableNormalTexture)
                    Data.flags &= ~kMaterialFlag_HasNormalTexture;
                if (!Ext.EnableOcclusionRoughnessMetallicTexture)
                    Data.flags &= ~kMaterialFlag_HasMetallicRoughnessTexture;
            }
```

Insert the following block immediately before that block's closing `}` (i.e. right after the `EnableOcclusionRoughnessMetallicTexture` clear and before the `}` that ends the `if (pExtension ... Loaded)` block):

```cpp
                // Apply external .material.json textures after scalar overrides and enable-switch clears, so
                // MaterialPTData reflects the final texture state before flag recomputation. An external texture
                // overrides the glTF binding for its slot; if the external load fails but a glTF binding exists,
                // the glTF binding is kept (fail-safe for shipped scenes), otherwise the slot falls back to factors.
                const auto ApplyExternalTexture = [&](const RTXPTMaterialTextureDesc& Desc, bool Enable,
                                                      Uint32 Flag, Uint32& TexIndex, float& TexSlice) {
                    if (!Enable || !Desc.HasPath)
                        return;

                    const Uint32 ExtIndex = AppendExternalTexture(pDevice, Desc, AssetsRoot,
                                                                  m_TextureViews, m_TextureBindings, ExternalCache);
                    if (ExtIndex != InvalidTextureIndex)
                    {
                        Data.flags |= Flag;
                        TexIndex = ExtIndex;
                        TexSlice = 0.0f;
                    }
                    else if ((Data.flags & Flag) == 0u)
                    {
                        Data.flags &= ~Flag;
                        TexIndex = 0;
                    }
                };

                ApplyExternalTexture(Ext.BaseTexture, Ext.EnableBaseTexture,
                                     kMaterialFlag_HasBaseColorTexture,
                                     Data.baseColorTextureIndex, Data.baseColorTextureSlice);
                ApplyExternalTexture(Ext.OcclusionRoughnessMetallicTexture, Ext.EnableOcclusionRoughnessMetallicTexture,
                                     kMaterialFlag_HasMetallicRoughnessTexture,
                                     Data.metallicRoughnessTextureIndex, Data.metallicRoughnessTextureSlice);
                ApplyExternalTexture(Ext.NormalTexture, Ext.EnableNormalTexture,
                                     kMaterialFlag_HasNormalTexture,
                                     Data.normalTextureIndex, Data.normalTextureSlice);
                ApplyExternalTexture(Ext.EmissiveTexture, Ext.EnableEmissiveTexture,
                                     kMaterialFlag_HasEmissiveTexture,
                                     Data.emissiveTextureIndex, Data.emissiveTextureSlice);

                const bool TransmissionEnabled = (Data.flags & kMaterialFlag_HasTransmission) != 0u;
                ApplyExternalTexture(Ext.TransmissionTexture, Ext.EnableTransmissionTexture && TransmissionEnabled,
                                     kMaterialFlag_HasTransmissionTexture,
                                     Data.transmissionTextureIndex, Data.transmissionTextureSlice);

                if (Ext.HasNormalTextureScale)
                    Data.normalScale = Ext.NormalTextureScale;
```

- [ ] **Step 8: Set `m_Stats.TextureCount` after the material loop**

At the end of `Upload`, change:

```cpp
    m_Stats.MaterialCount = static_cast<Uint32>(MaterialData.size());
    return CreateMaterialBuffer(pDevice, MaterialData);
```

to:

```cpp
    m_Stats.TextureCount  = static_cast<Uint32>(m_TextureBindings.size());
    m_Stats.MaterialCount = static_cast<Uint32>(MaterialData.size());
    return CreateMaterialBuffer(pDevice, MaterialData);
```

- [ ] **Step 9: Update the call site in `RTXPTSample.cpp`**

At `RTXPTSample.cpp:689`, change:

```cpp
    ResourcesReady &= m_Materials.Upload(m_pDevice, SceneData);
```

to:

```cpp
    ResourcesReady &= m_Materials.Upload(m_pDevice, SceneData, m_Scene.GetAssetsRoot());
```

- [ ] **Step 10: Verify it compiles**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds with no errors. If the linker reports an unresolved `CreateTextureFromFile`, do Task 6 Step 1 (add the explicit `Diligent-TextureLoader` link) and rebuild.

- [ ] **Step 11: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp \
        DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp \
        DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): load and apply external material JSON textures in Upload"
```

---

## Task 4: Make Classification Helpers Account For External Textures (D8)

The acceleration-structure build (`RTXPTAccelerationStructures.cpp:202-209`) and the lights build (`RTXPTLights.cpp:319`) classify materials through these helpers. Updating the helpers keeps any-hit / alpha-test / emissive-area-light decisions consistent with the final material texture state, without editing those two consumer files.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` (`RTXPTMaterialHasBaseColorTexture` ~lines 204-221; `RTXPTMaterialIsEmissiveAreaLight(Material, pExtension)` ~lines 251-263)

- [ ] **Step 1: Treat an authored, enabled external base texture as a base-color texture**

In `RTXPTMaterialHasBaseColorTexture`, after the existing enable-switch early-out, add a second early-out for the external path. Change:

```cpp
    if (pExtension != nullptr && pExtension->Loaded && !pExtension->EnableBaseTexture)
        return false;

    const int BaseColorTextureId = Material.GetTextureId(GLTF::DefaultBaseColorTextureAttribId);
```

to:

```cpp
    if (pExtension != nullptr && pExtension->Loaded && !pExtension->EnableBaseTexture)
        return false;

    // An authored, enabled external base texture provides base color even when the glTF carries none.
    if (pExtension != nullptr && pExtension->Loaded && pExtension->BaseTexture.HasPath)
        return true;

    const int BaseColorTextureId = Material.GetTextureId(GLTF::DefaultBaseColorTextureAttribId);
```

- [ ] **Step 2: Treat an authored, enabled external emissive texture as emissive-texture use**

Replace the body of `RTXPTMaterialIsEmissiveAreaLight(const GLTF::Material& Material, const RTXPTMaterialExtension* pExtension)`:

```cpp
    const bool ExtensionLoaded = pExtension != nullptr && pExtension->Loaded;
    const bool UsesEmissiveTexture =
        Material.GetTextureId(GLTF::DefaultEmissiveTextureAttribId) >= 0 &&
        (!ExtensionLoaded || pExtension->EnableEmissiveTexture);
    if (UsesEmissiveTexture)
        return false;

    const float3& Emission = ExtensionLoaded ? pExtension->EmissiveFactor : Material.Attribs.EmissiveFactor;
    return HasNonZeroEmission(Emission);
```

with:

```cpp
    const bool ExtensionLoaded = pExtension != nullptr && pExtension->Loaded;
    const bool UsesGLTFEmissiveTexture =
        Material.GetTextureId(GLTF::DefaultEmissiveTextureAttribId) >= 0 &&
        (!ExtensionLoaded || pExtension->EnableEmissiveTexture);
    const bool UsesExternalEmissiveTexture =
        ExtensionLoaded && pExtension->EmissiveTexture.HasPath && pExtension->EnableEmissiveTexture;
    if (UsesGLTFEmissiveTexture || UsesExternalEmissiveTexture)
        return false;

    const float3& Emission = ExtensionLoaded ? pExtension->EmissiveFactor : Material.Attribs.EmissiveFactor;
    return HasNonZeroEmission(Emission);
```

- [ ] **Step 3: Verify it compiles**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds with no errors.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp
git commit -m "fix(rtxpt): account for external material textures in material classification"
```

---

## Task 5: Wire The `RTXPT_ENABLE_MATERIAL_EXTENSION` Master Switch

Apply the master switch (defined in Task 1) at two points: the `.material.json` parse/load loop ("loaded") and the extension accessor ("used"). When the macro is `0`, no material JSON is read and every consumer falls back to pure glTF behavior.

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp` (parse loop in `RTXPTScene::LoadScene` ~lines 911-922)
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp` (`RTXPTGetMaterialExtension` ~lines 190-202)

- [ ] **Step 1: Gate the `.material.json` parse loop in `RTXPTScene::LoadScene`**

`RTXPTScene.cpp` already sees the macro via `RTXPTScene.hpp` → `RTXPTSceneGraph.hpp`. Wrap the candidate-loading loop in a `#if`. Change:

```cpp
            for (const std::string& Candidate : GetRTXPTMaterialCandidates(m_AssetsRoot, SceneName, Asset.ModelName, MaterialName))
            {
                if (!FileSystem::FileExists(Candidate.c_str()))
                    continue;

                RTXPTJsonLoadResult MaterialJson;
                if (LoadRTXPTRelaxedJsonFile(Candidate, MaterialJson) && MaterialJson.Json.is_object())
                {
                    Ext = ParseRTXPTMaterialExtension(Candidate, Asset.ModelName, MaterialName, MaterialJson.Json);
                    break;
                }
            }
```

to:

```cpp
#if RTXPT_ENABLE_MATERIAL_EXTENSION
            for (const std::string& Candidate : GetRTXPTMaterialCandidates(m_AssetsRoot, SceneName, Asset.ModelName, MaterialName))
            {
                if (!FileSystem::FileExists(Candidate.c_str()))
                    continue;

                RTXPTJsonLoadResult MaterialJson;
                if (LoadRTXPTRelaxedJsonFile(Candidate, MaterialJson) && MaterialJson.Json.is_object())
                {
                    Ext = ParseRTXPTMaterialExtension(Candidate, Asset.ModelName, MaterialName, MaterialJson.Json);
                    break;
                }
            }
#endif
```

When the macro is `0`, `Ext` stays default (`Loaded == false`) and no JSON is read; the surrounding `MaterialRemap`/`MaterialExtensions` bookkeeping is unchanged so consumers stay structurally valid.

- [ ] **Step 2: Gate the extension accessor in `RTXPTGetMaterialExtension`**

In `RTXPTMaterials.cpp`, replace the body of `RTXPTGetMaterialExtension`:

```cpp
    if (MaterialId >= Asset.MaterialRemap.size())
        return nullptr;

    const Uint32 ExtensionIdx = Asset.MaterialRemap[MaterialId];
    if (ExtensionIdx >= SceneData.MaterialExtensions.size())
        return nullptr;

    return &SceneData.MaterialExtensions[ExtensionIdx];
```

with:

```cpp
#if RTXPT_ENABLE_MATERIAL_EXTENSION
    if (MaterialId >= Asset.MaterialRemap.size())
        return nullptr;

    const Uint32 ExtensionIdx = Asset.MaterialRemap[MaterialId];
    if (ExtensionIdx >= SceneData.MaterialExtensions.size())
        return nullptr;

    return &SceneData.MaterialExtensions[ExtensionIdx];
#else
    (void)SceneData;
    (void)Asset;
    (void)MaterialId;
    return nullptr;
#endif
```

(The `(void)` casts avoid unused-parameter warnings when the extension is disabled.)

- [ ] **Step 3: Verify it compiles (default on)**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds with no errors. Default behavior is unchanged because the macro defaults to `1`.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp \
        DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp
git commit -m "feat(rtxpt): gate material extension behind RTXPT_ENABLE_MATERIAL_EXTENSION"
```

---

## Task 6: Confirm Link Configuration

`RTXPTEnvMapBaker.cpp` already calls `CreateTextureFromFile`, so the `Diligent-TextureLoader` symbols are expected to resolve transitively through `Diligent-AssetLoader` / `DiligentFX`. This task only adds an explicit link if Task 3's build failed to resolve it.

**Files:**
- Modify (conditional): `DiligentSamples/Samples/RTXPT/CMakeLists.txt` (`target_link_libraries(RTXPT PRIVATE ...)` ~line 378)

- [ ] **Step 1: Add explicit link only if needed**

If — and only if — Task 3 Step 10 produced an unresolved-external linker error for `CreateTextureFromFile`, add `Diligent-TextureLoader` to the `PRIVATE` link list. Change:

```cmake
target_link_libraries(RTXPT
PRIVATE
    Diligent-AssetLoader
    Diligent-JSON
    Diligent-SuperResolution-static
    DiligentFX
)
```

to:

```cmake
target_link_libraries(RTXPT
PRIVATE
    Diligent-AssetLoader
    Diligent-JSON
    Diligent-SuperResolution-static
    Diligent-TextureLoader
    DiligentFX
)
```

If Task 3 already linked cleanly, make no change and skip to Step 2.

- [ ] **Step 2: Verify (only if CMakeLists changed)**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds and links with no errors.

- [ ] **Step 3: Commit (only if CMakeLists changed)**

```bash
git add DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "build(rtxpt): link Diligent-TextureLoader explicitly for material textures"
```

---

## Task 7: Runtime, Negative, And Regression Validation

End-to-end validation (no unit-test harness exists). Run the built `RTXPT` sample and check behavior. Executable: `build/x64/Debug/DiligentSamples/Samples/RTXPT/Debug/RTXPT.exe`. Scenes are selected from the sample's scene UI.

- [ ] **Step 1: Build the full sample**

Run: `cmake --build build/x64/Debug --config Debug --target RTXPT`
Expected: builds and links cleanly.

- [ ] **Step 2: Focused validation — convergence-test floor texture**

- Launch `RTXPT.exe` and load `convergence-test.scene.json`.
- Confirm the floor renders the wood texture from `Models/living_room/textures/wood4.dds` (previously untextured).
- Confirm the material-texture path is active: with bindless support, `RTXPTMaterials::GetTextureCount()` is now > 0 for this scene, so `ENABLE_MATERIAL_TEXTURES` is defined and `MATERIAL_TEXTURE_COUNT` equals the live binding count.
- Run reference mode and visually compare the floor against RTXPT-fork / a known-good screenshot.

Expected: wood floor visible; no validation-layer errors about texture-view dimensions.

- [ ] **Step 3: Regression validation — shipped scenes with glTF + external textures**

- Load `bistro-programmer-art.scene.json` (Bistro authors `BaseTexture`/`NormalTexture`/`OcclusionRoughnessMetallicTexture` paths on top of glTF textures; the DDS files resolve under `Models/Bistro/objects/...`).
- Confirm Bistro renders correctly and no material that was textured before becomes factor-only.
- Load an ABeautifulGame scene and confirm the chess pieces/board still render their textures.

Expected: shipped scenes render through the external texture path with no visible texture loss.

- [ ] **Step 4: Negative validation — failed external load**

- In a local copy, edit one `convergence-test` material JSON (slot with no glTF texture) to point a texture `path` at a missing file. Load the scene.
- Expected: a warning is logged ("Failed to load RTXPT material texture: ..."), no crash, the slot's flag is cleared, and factor-only fallback renders.
- Separately, edit a Bistro material (slot that has a glTF texture) to point the external path at a missing file.
- Expected: a warning is logged and the glTF texture is retained (not dropped to factor-only).
- Restore both material JSONs.

- [ ] **Step 5: Regression validation — enable switches and absent objects**

- Confirm a material with an absent external texture object but `Enable*Texture = true` still shows its glTF texture.
- Confirm a material with `Enable*Texture = false` disables both glTF and external textures for that slot.
- Confirm any-hit/alpha-test behavior is correct for an alpha-masked material whose base color comes only from an external texture (no glTF base texture): the alpha test must apply (the AS gives that geometry an any-hit shader via the updated `RTXPTMaterialHasBaseColorTexture`).

- [ ] **Step 6: Master-switch validation — `RTXPT_ENABLE_MATERIAL_EXTENSION=0`**

- Reconfigure with the switch off, e.g. add `target_compile_definitions(RTXPT PRIVATE RTXPT_ENABLE_MATERIAL_EXTENSION=0)` (or temporarily change the header default to `0`), and rebuild.
- Load `convergence-test.scene.json` and `bistro-programmer-art.scene.json`.
- Expected: no `.material.json` is read; both scenes render with pure glTF material behavior (convergence-test floor is untextured again; Bistro renders via its glTF textures). No crashes and no extension-related warnings.
- Restore the switch to its default (`1`) and rebuild before finishing.

- [ ] **Step 7: Commit any validation-driven fixes**

If any validation step surfaces a defect, fix it, re-run the affected validation step, and commit with a `fix(rtxpt): ...` message describing the corrected behavior.

---

## Self-Review

**1. Spec coverage**

| Spec section | Covered by |
|---|---|
| D0 master switch `RTXPT_ENABLE_MATERIAL_EXTENSION` | Task 1 (define) + Task 5 (gate load & accessor) |
| D1 descriptor type + extension fields | Task 1 |
| D2 parse texture objects + `EnableTransmissionTexture` | Task 2 |
| D3 assets-root resolution + slash normalization + PNG→DDS | Task 2 (normalize), Task 3 Step 4 (`ResolveExternalTexturePath`), Step 5/9 (assets root plumbing) |
| D4 load through TextureLoader, preferred `Texture2DArray` SRV, no fallback | Task 3 Step 4 (`AppendExternalTexture` + `CreateMaterialTextureView`) |
| D5 dedup map (Upload-local), lifetime via `m_TextureViews` | Task 3 Step 4 (`ExternalTextureCache`), Step 6 (declared in `Upload`) |
| D6 preserve glTF order; move `m_Stats.TextureCount` | Task 3 Steps 6-8 |
| D7 apply descriptors per slot; fail-safe glTF retention; `NormalTextureScale` only if present | Task 3 Step 7; Task 1/2 for `HasNormalTextureScale` |
| D8 classification consistency in AS/Lights | Task 4 (helpers) |
| External-override-of-glTF decision (fail-safe) | Task 3 Step 7 (`else if ((Data.flags & Flag) == 0u)`) |
| Master-switch wiring (load gate + accessor gate) | Task 5 |
| Link configuration | Task 6 |
| Build / focused / negative / regression / master-switch validation | Task 7 |
| Acceptance criteria | Tasks 1-7 collectively; verified in Task 7 |

No spec requirement is left without a task.

**2. Placeholder scan:** No TBD/TODO/"handle edge cases"/"similar to" placeholders. Every code step shows complete code; every verify step shows the command and expected result.

**3. Type consistency:**
- `RTXPTMaterialTextureDesc{ LocalPath, HasPath, SRGB, NormalMap }` — defined Task 1, used identically in `ReadMaterialTexture` (Task 2), `AppendExternalTexture`/`ResolveExternalTexturePath`/`ApplyExternalTexture` (Task 3), and the D8 helpers (Task 4).
- `Upload(IRenderDevice*, const RTXPTSceneGraphData&, const std::string&)` — declared Task 3 Step 2, defined Step 5, called Step 9.
- `AppendExternalTexture(pDevice, Desc, AssetsRoot, m_TextureViews, m_TextureBindings, ExternalCache)` — signature (Step 4) matches call (Step 7).
- `ExternalTextureCache` / `ExternalTextureBinding` — defined and used only within `RTXPTMaterials.cpp` (Steps 4, 6, 7).
- Flag constants (`kMaterialFlag_Has*Texture`) and `MaterialPTData` field names match `RTXPTMaterials.hpp`.
- `EnableTransmissionTexture`, `HasNormalTextureScale`, `NormalTextureScale` — defined Task 1, set Task 2, consumed Task 3 Step 7.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-09-rtxpt-material-json-texture-loading.md`. Two execution options:

1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
