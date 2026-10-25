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

## Core PR follow-up — 2026-10-25

### Trigger signals

Official master integration does not include open PRs #812, #814 and #817. The RTXPT superproject uses Core branch `RTXPT-ray-reconstruction`, not the fork's older branch named `RTXPT`.

### Root cause / constraints

- #812: native D3D12 pipeline libraries ignored initial serialized data, used unstable/non-effective pipeline names, omitted mesh cache reuse, requested the wrong COM interface on cache hits, and lost the native cache when unpacking RenderStateCache PSOs.
- #814: device shaders reserialized by RenderStateCache lost the selected WGSL entry point. Master commit `f0bd4e5b6` fixes related pipeline/archive entry-point handling but does not cover this path.
- #817: reusing a signaled manual-reset fence event allows later waits to return prematurely; DXGI frame-latency waitable handles also require explicit closure on recreation/destruction.

### Correct practice and resulting commit

Core `36a1ee7d9921676df5aabef8f09139f7db6c498e` integrates the six fix commits: #812 `2a7f66ee5`, `55f85040d`, `fbb9a2783`; #814 `f3a3a04fb`; #817 `acac7294e`, `720904a21`. These were applied as patches in one integration commit; the original PR commits are not ancestors. Do not import #814's `6a9c8063c` development merge. Keep the prior coordinated merge baseline and all other submodule pins, including NRD/NRI.

Retain the PR fence and four WebGPU regressions. Added `RenderStateCacheTest.NativePipelineCacheRoundTrip`: three native-cache/archive save/load passes, released input blob, restored cache-content checks and actual compute execution. Diagnostic evidence is one native cache miss followed by two hits.

### Validation and limits

- VS 2022 x64 Release: built Core API/unit tests and RTXPT in `build/x64/Release`; independent WebGPU Core API build in `build/core-pr-webgpu` (local CMake cache only; tracked configuration unchanged).
- 556 Core unit tests, 59 D3D12 GPU tests, 54 Vulkan GPU tests and 6 WebGPU GPU tests passed. Three Metal-only mesh tests skipped. Vulkan RR uses `DILIGENT_TEST_RR_PRESET=5`.
- Run unit tests from `DiligentCore/Tests/DiligentCoreTest/assets` and GPU tests from `DiligentCore/Tests/DiligentCoreAPITest/assets`. The initial unit invocation from the executable directory failed due to missing shader assets; the full suite passed after correcting the working directory.
- Avoid rebuilding a test executable while it is running: Windows linker LNK1104 results from the locked executable. The Core unit/RTXPT rebuild passed after the first test process exited.
- WebGPU command-line mode is `--mode=wgpu`. Its four new RenderStateCache entry-point tests and two existing WGSL archive tests passed with no skips.
- All 15 modified files passed clang-format 10 and `git diff --check`. Updated modified legacy copyright years to 2026.
- Two hidden D3D12 RTXPT captures (reference and RR preset F) exited 0; screenshots inspected. RR totals: created=2, executed=2, resets=1, retaining native FP16 and HitObject + reorder. These are startup/render/shutdown smoke checks, not a long-running frame-latency handle leak or temporal image-quality benchmark.
- Evidence: `build/core-pr-unit-assets-tests.{log,xml}`, `build/core-pr-{d3d12,vk,webgpu}-tests.{log,xml}`, `build/core-pr-d3d12-regression-tests.{log,xml}`, `build/core-pr-captures/`; successful build logs: `core-pr812-814-817-verified-build.log` (API target), `core-pr-unit-rtxpt-build.log`, `core-pr-webgpu-build.log`.

### Applicable scope

Checking open upstream fixes after a master sync; persistent D3D12 PSO reuse, WebGPU RenderStateCache serialization and D3D12 CPU fence/DXGI resource lifetime.

## Coordinated RTXPT merge — 2026-10-25

This section records the current merge baseline. Earlier sections retain historical SHA and build-tool evidence.

### Trigger signals

- Updating Core changes `LoadAndCreateSuperResolutionFactory` from a device pointer to `SuperResolutionFactoryCreateInfo` (upstream `be74a747d`).
- Updating Tools changes `Material::GetTextureAttrib()` to return `Material::TextureAttribs`, whose GPU fields are nested under `.ShaderAttribs`.
- FX can fast-forward but depends on newer Tools GLTF types and, when Radient is enabled, Core's `GPU_UPLOAD_MAX_BUFFER_SIZE`.
- Samples can merge without text conflicts but Tutorial27 requires the newer Core SR factory API.

### Root cause / constraints

Git merge compatibility does not establish cross-module API compatibility. The local Core RR implementation uses shared NGX lifetime management and per-test SR factory ownership. Tools must preserve authored transmission alpha modes and the three local transmission/IOR regression tests. In this checkout, `origin` is DiligentGraphics official upstream; the remote named `upstream` points to hzqst forks.

### Correct practice and resulting commits

1. Merge Core official master `cd2329b91`; adapt the RTXPT SR pass and RR test through `SuperResolutionFactoryCreateInfo`. Resolve the SR test helper conflict by preserving per-test `RefCntAutoPtr` ownership and removing the upstream cached-factory release in ArchivedFSR. Core result: `b5a694ad4`.
2. Merge Tools official master `dd499e0`; retain both local GLTF string-loading helpers and upstream UV transform helpers. Preserve the authored alpha-mode implementation and all three transmission/IOR tests. Tools result: `ce6a7d0`.
3. Fast-forward FX to official master `9b65ac23`; change the six RTXPT texture-slice reads to `GetTextureAttrib(...).ShaderAttribs.TextureSlice`. The two RTXPT API adaptations are Samples commit `e0f7b827`.
4. Merge Samples official master `41fdf49f` after the adaptations. Samples result: `50774cdc`.
5. Superproject integration commit: `e2361e5a2f49785c4566adf507fa9bc1b373ad82`. Original local tips and official master tips are ancestors of each resulting submodule HEAD. No history was rewritten.

Update Core's pinned ThirdParty gitlinks and Tools' pinned imgui gitlink. Keep Samples NRD `366a90d25ce3772de9360a1e23117d5a455a2771`, NRI `f5fc90a9fcaab97617b4e7a715534247b273a345`, MathLib `974e1387ba936740c7cdc494792d2641bc127e86`, and ShaderMake `5daebdbef45088fc2369d441391ecab0eba25e54` unchanged.

### Validation and limits

- Windows VS 2022 / CMake 3.31.12, `build/x64/Release`, `--config Release`, existing `DILIGENT_NO_RADIENT=ON` configuration.
- Built `DiligentCoreAPITest`, `DiligentToolsTest`, `DiligentToolsGPUTest`, `RTXPT`, `GLTFViewer`, and `Tutorial27_PostProcessing`.
- Core D3D12: 12 SR/RR tests passed. Vulkan: 14 SR/RR and promoted dynamic UBO tests passed, using `DILIGENT_TEST_RR_PRESET=5` (E).
- Tools: 191 unit tests passed. Run from `DiligentTools/Tests/DiligentToolsTest/assets`; running from the executable directory causes missing RenderStates JSON failures. GLTFResourceManager: 9 tests passed on each of D3D12 and Vulkan. Total: 235 tests, zero failures or skips.
- clang-format 10 checks passed for the modified RTXPT files and Tools test file; changed Core test ranges were checked.
- Four two-render golden-image captures of `convergence-test.scene.json` succeeded: D3D12/Vulkan reference and D3D12 F/Vulkan E RR. RR logs show two evaluations each; D3D12 retains `HitObject + reorder`. Artifacts: `build/upstream-merge-captures/`; test XML/logs: `build/merge-core-*` and `build/merge-tools-*`.
- D3D12 logs warn that the DirectSR device factory cannot be created on this host; DLSS RR is available and executes. DirectSR availability, full temporal image quality, performance benchmarking, Debug builds, and Radient were not validated by this sync.

### Applicable scope

Coordinated official-master merges of Core/Tools/FX/Samples for this RTXPT fork, especially when preserving RR, native FP16, SM 6.9/SER, and transmission alpha semantics.

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
