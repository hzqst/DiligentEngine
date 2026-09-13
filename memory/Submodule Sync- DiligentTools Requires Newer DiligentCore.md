---
title: 'Submodule Sync: DiligentTools Requires Newer DiligentCore'
type: note
permalink: diligentengine-hzqst/submodule-sync-diligent-tools-requires-newer-diligent-core
tags:
- submodule
- sync
- diligentcore
- diligenttools
- gltf
---

# Submodule Sync: DiligentTools Requires Newer DiligentCore

## Trigger Signal

Syncing `DiligentTools` to upstream `DiligentGraphics/DiligentTools` master fails to build against the Core pinned in this repo, with errors like:

```text
error C1083: cannot open include file: "../../../DiligentCore/Common/interface/SharedMutex.hpp"
error C2039: "GetSerializationDeviceInfo" is not a member of "Diligent::ISerializationDevice"
```

## Root Cause / Constraint

Upstream DiligentTools master depends on DiligentCore APIs newer than the previously pinned Core (fork `RTXPT-4` @ `b244e65`, API256015, 148 commits behind):

- `f004cbc AssetLoader: use SharedMutex for GLTF caches` requires `Common/interface/SharedMutex.hpp` (Core `635d5005f`).
- `11d69cc Update RenderStatePackager for serialization device info API` requires `ISerializationDevice::GetSerializationDeviceInfo` (Core `fce80de8c`, which is an API change touching the whole serialization subsystem, not purely additive).

The newest DiligentTools upstream commit that still builds against the old Core is `5134b6b`; every commit at/after `f004cbc` needs the newer Core. So "sync Tools to latest" and "keep old Core" are mutually exclusive.

## Correct Practice

- Sync order matters: bring `DiligentCore` up first (or together with Tools), then `DiligentTools`.
- Update the Core ThirdParty submodules as well: `git -C DiligentCore submodule update --init --recursive` (glslang, SPIRV-Tools, SPIRV-Cross, SPIRV-Headers, Vulkan-Headers, volk all move).
- To sync Tools to latest master: `git fetch origin && git rebase --onto origin/master <last-shared-commit> RTXPT`. Expect conflicts in `AssetLoader/src/GLTFLoader.cpp` because upstream extracted material loading into `LoadMaterial()`; resolve by taking the upstream refactored file and re-applying the local semantic patch into the new location.
- Switching Core to upstream master drops the fork's 9 local shader-include commits (glslang nested include / DXC dir). Upstream now carries equivalents (`671bce3fc GLSLangUtils: support parent-relative nested includes`, `dbf2f7b8c DXC: support parent-relative nested includes`, `92e7c8b92 ShaderTools: centralize shader include path resolution`). They are NOT patch-identical (`git cherry origin/master <fork>` reports `+`), so runtime shader include behavior should be re-checked after such a sync.
- Always create a backup ref before rewriting: `git branch backup/<name> <old-tip>` inside each submodule.

## Build Tooling Gotchas

- The existing `build/x64/Debug` was configured with Strawberry CMake 3.29 (`CMAKE_ROOT=C:/Strawberry/c/share/cmake-3.29`). Building it with the PATH CMake 3.31 breaks reconfigure with `include could not find requested file .../CMakeDetermineCompilerSupport.cmake`. Use `/c/Strawberry/c/bin/cmake.exe --build <dir> --config Debug --target <t>`.
- Under Git Bash, MSBuild args get path-mangled (`/m` becomes `M:/`), causing `MSBUILD : error MSB1008: Only one project can be specified`. Prefix with `MSYS2_ARG_CONV_EXCL='*'`.

## Validation

```bash
MSYS2_ARG_CONV_EXCL='*' /c/Strawberry/c/bin/cmake.exe --build build/x64/Debug --config Debug --target DiligentToolsTest -- /m
# Run the test exe from the assets dir, otherwise RenderState*/RenderStatePackager tests fail on missing RenderStates/*.json:
cd DiligentTools/Tests/DiligentToolsTest/assets && <build>/DiligentToolsTest.exe   # expect 136/136 pass
MSYS2_ARG_CONV_EXCL='*' /c/Strawberry/c/bin/cmake.exe --build build/x64/Debug --config Debug --target RTXPT -- /m   # expect 0 errors, RTXPT.exe produced
```

## Preserved Local DiligentTools Patch

`fix(gltf): preserve alpha mode for transmission` removes `Mat.Attribs.AlphaMode = Material::ALPHA_MODE_BLEND;` from the `KHR_materials_transmission` branch in `AssetLoader/src/GLTFLoader.cpp`, so the `alphaMode` authored in the glTF material is kept instead of being forced to BLEND.

Upstream deliberately keeps forcing BLEND: the fork's `GLTF-Transmission` branch restored it in `65d3d59`, and only the IoR half landed as PR #266 (`a57b5a3 GLTFLoader: Enable IoR`). Re-apply this patch on every DiligentTools sync; the three tests in `Tests/DiligentToolsTest/src/GLTFLoaderTest.cpp` (`TransmissionKeepsDefaultOpaqueAlphaMode`, `TransmissionPreservesAuthoredAlphaMode`, `KHRMaterialsIORAppliesToTransmission`) guard it.

## Applicable Scope

Any periodic sync of `DiligentCore` / `DiligentTools` / `DiligentFX` / `DiligentSamples` submodules in this repository, and builds of `build/x64/Debug` on this machine.
