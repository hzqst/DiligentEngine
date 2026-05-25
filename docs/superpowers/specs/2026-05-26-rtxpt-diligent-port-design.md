# RTXPT DiligentEngine Port Design

## Summary

This design defines how to port NVIDIA RTXPT's advanced real-time path tracing sample into DiligentEngine as a new sample named `RTXPT`, located at `DiligentSamples/Samples/RTXPT`.

The long-term target is to reproduce the current RTXPT advanced sample as completely as practical. The implementation must still move in runnable increments: after every migration task, the `RTXPT` sample must start and render a valid fallback or feature-specific result that can be reviewed. Any capability that is not complete yet must be isolated behind a feature flag or fallback path and marked in code with a structured `// TODO(RTXPT-Port Phase N): ...` comment.

The port must support both Diligent D3D12 and Vulkan backends from the start. D3D12-only or Vulkan-only advanced paths may exist only behind explicit capability checks.

## Confirmed Requirements

- Add a new Diligent sample at `DiligentSamples/Samples/RTXPT`.
- Use the shortest path that preserves quality, but keep every task deliverable runnable.
- Target complete RTXPT advanced sample behavior over time.
- Start from a minimal runnable sample, then progressively migrate features.
- Keep D3D12 and Vulkan as first-class supported backends.
- Use DiligentCore's existing DXC/ShaderMake tooling; do not introduce a separate DXC/ShaderMake dependency.
- Treat RTXDI, NRD, OMM, Streamline/DLSS and NVAPI as sample-level optional dependencies.
- Copy runtime assets into `DiligentSamples/Samples/RTXPT/assets`; the sample must not depend on `D:/RTXPT-fork` at runtime.
- Use `DiligentTools/AssetLoader` for scene, model, texture and base material loading.
- Preserve incomplete work using structured code comments and a central open-work registry.

## Recommended Migration Route

Use a vertical, runnable-increment route:

1. Build the Diligent sample shell and minimal capability/debug view.
2. Add resource creation and a minimal scene/asset path.
3. Add resource updates, including BLAS/TLAS and frame constants.
4. Add minimal TraceRays and compute dispatches.
5. Replace minimal shaders with RTXPT shader layers.
6. Restore post-processing and advanced RTXPT features.

This route is preferred because every step can be launched and reviewed. It also prevents a long infrastructure-only phase where much code is present but no sample can run.

Rejected alternatives:

- Build all infrastructure first, then render: closer to RTXPT's original structure, but it delays runnable validation too long.
- Migrate module by module horizontally: clear ownership, but each completed module may not produce a runnable sample and integration risk accumulates.

## Target Directory Layout

```text
DiligentSamples/Samples/RTXPT/
  CMakeLists.txt
  src/
    RTXPTSample.hpp
    RTXPTSample.cpp
    RTXPTRenderPipeline.hpp
    RTXPTRenderPipeline.cpp
    RTXPTFeatureCaps.hpp
    RTXPTSettings.hpp
    RTXPTScene.hpp
    RTXPTScene.cpp
    RTXPTRenderTargets.hpp
    RTXPTRenderTargets.cpp
    RTXPTAccelerationStructures.hpp
    RTXPTAccelerationStructures.cpp
    RTXPTMaterials.hpp
    RTXPTMaterials.cpp
    RTXPTLights.hpp
    RTXPTLights.cpp
    RTXPTBindingModel.hpp
    RTXPTBindingModel.cpp
    RTXPTPathTracer.hpp
    RTXPTPathTracer.cpp
    RTXPTPostProcess.hpp
    RTXPTPostProcess.cpp
  assets/
    scenes/
    materials/
    models/
    textures/
    shaders/
```

The exact file list may be adjusted during implementation if local Diligent sample conventions suggest a smaller or better boundary. The stable architectural boundary is:

- `RTXPTSample`: Diligent sample lifecycle, UI, resize, update and render entry points.
- `RTXPTRenderPipeline`: per-frame orchestration.
- `RTXPTFeatureCaps`: backend and optional-feature capability detection.
- `RTXPTScene`: AssetLoader-based scene/model/texture/base-material loading plus RTXPT-specific scene/material/light extensions.
- `RTXPTRenderTargets`: render textures, history resources and resize handling.
- `RTXPTAccelerationStructures`: BLAS/TLAS creation, build/update, instance contribution and SBT hit group contract.
- `RTXPTMaterials`: CPU and GPU material data, texture indices and material specialization metadata.
- `RTXPTLights`: environment, analytic lights, emissive triangles and later RTXPT light baker/RTXDI data.
- `RTXPTBindingModel`: SRB, shader variables, bindless/fallback bindings and resource slot ownership.
- `RTXPTPathTracer`: RT PSO/SBT setup and TraceRays dispatches.
- `RTXPTPostProcess`: accumulation, tone mapping, denoising, DLSS/Streamline and debug overlays.

## Phase Design

### Phase 1: Investigation

Purpose:

- Create an executable migration map, not only notes.
- Map RTXPT modules to Diligent sample modules.
- Map NVRHI/Donut concepts to Diligent APIs.
- Identify D3D12/Vulkan differences and advanced dependency constraints.

Runnable milestone:

- A new `RTXPT` sample exists and launches.
- It displays a clear/debug screen with backend and capability status.
- It detects RayTracing, standalone RT shader support, RayQuery, bindless/resource array support, shader compiler availability and compute support.

### Phase 2: Resource Creation And Initialization

Purpose:

- Create CMake integration, source skeleton and assets directory.
- Copy the first default scene asset closure into `DiligentSamples/Samples/RTXPT/assets`.
- Create basic render targets, constant buffers, samplers and initial binding structures.
- Use `DiligentTools/AssetLoader` for model, texture and base material input.

Runnable milestone:

- The sample starts without requiring `D:/RTXPT-fork`.
- It loads default scene metadata or a simplified default scene.
- It renders a clear/debug/tone-mapped fallback view.

### Phase 3: Resource Update

Purpose:

- Update camera, frame constants, material buffers, light buffers and per-frame settings.
- Build and update static mesh BLAS and TLAS.
- Maintain sub-instance data and hit group contribution indices.
- Start with static meshes and full TLAS rebuild; add dynamic/skinned update paths later.

Runnable milestone:

- D3D12 and Vulkan can create the scene acceleration structures when supported by the device.
- A simple shader/debug path can prove frame constants and scene data are valid.
- If ray tracing is unavailable, the sample still launches and explains the disabled path.

### Phase 4: Draw And Dispatch Calls

Purpose:

- Add a minimal RT PSO/SBT and `IDeviceContext::TraceRays`.
- Add a reusable compute pass helper for later RTXPT passes.
- Move from minimal ray tracing to reference path tracing, stable planes and RTXDI dispatches.

Runnable milestone:

- A raygen/miss/closest-hit loop writes basic color, normal or depth into `OutputColor`.
- The sample can present this result on both D3D12 and Vulkan where RT support exists.
- Compute dispatch infrastructure can run a minimal fullscreen or texture-processing pass.

### Phase 5: Shader Porting

Purpose:

- Port shaders by dependency layer, not as one large drop.
- Keep C++ bindings and HLSL register declarations synchronized.
- Preserve D3D12/Vulkan-compatible base variants before adding advanced backend-specific variants.

Shader layers:

1. Diligent minimal RT shader.
2. Shared structs, constants and binding declarations.
3. Scene bridge and material bridge.
4. Reference path tracer core.
5. Material specialization, alpha test and any-hit.
6. Stable planes and realtime mode.
7. RTXDI shader bridge and passes.
8. NRD, denoising guides and post-process.
9. Optional NVAPI, SER, OMM and DLSS-related variants.

Runnable milestone:

- Each shader layer lands with a runnable fallback.
- Unsupported variants compile out through macros and capability checks.

### Phase 6: Post Processing

Purpose:

- Restore the display chain from `OutputColor` to swapchain.
- Start with simple accumulation/tone mapping/blit.
- Add bloom, TAA, NRD standalone denoise, DLSS/DLSS-RR/Streamline, debug overlays, shader debug and zoom progressively.

Runnable milestone:

- Basic tone mapping and present work independently of optional post-process features.
- Each advanced pass can be individually disabled without breaking the sample.

## Backend Capability Model

`RTXPTSample` initializes `RTXPTFeatureCaps` from Diligent device and adapter information. This object controls feature creation, UI enablement and shader macro selection.

Important capabilities:

- RayTracing.
- Standalone ray tracing shaders.
- RayQuery.
- Bindless resources or descriptor indexing/runtime arrays.
- Compute shader support.
- Shader compiler target support for DXIL and SPIR-V.
- BLAS/TLAS build and update support.
- Acceleration structure compaction support.
- Texture and UAV format support required by RTXPT render targets.

Backend-specific behavior:

- D3D12 and Vulkan share the same high-level render pipeline.
- The upper-level sample flow must not hard-code a D3D12-only path.
- Advanced features that are unavailable on one backend are disabled with an explicit reason.
- D3D12-only shader extensions such as NVAPI/SER paths require both C++ feature caps and shader macro guards.
- Core validation targets are D3D12 and Vulkan support for BLAS/TLAS, TraceRays, basic compute and tone mapping.

## Resource And Asset Strategy

Runtime assets are copied into the sample:

```text
DiligentSamples/Samples/RTXPT/assets/
  scenes/
  materials/
  models/
  textures/
  shaders/
```

Asset loading uses `DiligentTools/AssetLoader` for models, textures and base material data. RTXPT-specific `.scene.json`, material JSON extensions, light metadata and path tracing settings are parsed by `RTXPTScene` as an adaptation layer above AssetLoader.

Initial material support:

- Base color.
- Roughness.
- Metallic.
- Alpha mode.
- Texture bindings.

Later material support:

- Nested dielectrics.
- Transmission.
- Advanced BSDF parameters.
- Material shader permutation.
- Alpha-test any-hit specialization.

Initial light support:

- Environment light.
- Directional light.
- Simple analytic lights.
- Basic emissive mesh extraction where data is available.

Later light support:

- RTXPT `LightsBaker`.
- Environment map baker.
- NEE feedback.
- Light proxy generation.
- RTXDI light buffer and ReGIR data.

Initial acceleration structure support:

- Static triangle mesh BLAS.
- TLAS built from static scene instances.
- Full TLAS rebuild on scene or transform changes.

Later acceleration structure support:

- Dynamic/skinned BLAS update.
- Per-material opaque/alpha flags.
- OMM integration.
- Compaction/update optimization.
- Exact RTXPT sub-instance and hit group contribution behavior.

## Per-Frame Flow

The target per-frame order is:

```text
UpdateSettingsAndCamera
UpdateSceneAndAnimation
UpdateMaterials
UpdateLights
UpdateAccelerationStructures
UpdateFrameConstants
RunPathTracingAndComputePasses
RunPostProcess
Present
```

`RTXPTAccelerationStructures` owns the contract between:

- `SubInstanceData`.
- `InstanceContributionToHitGroupIndex`.
- `MaterialHitGroupIndex`.
- SBT hit group binding.

This contract is high risk and must be kept explicit. RTXPT's original sample depends on stable ordering between scene geometry, sub-instances, material permutations, TLAS instance contribution and shader table hit groups.

## Draw And Dispatch Strategy

The draw/dispatch path evolves in this order:

1. Clear/debug path.
2. Minimal TraceRays path.
3. Reference path tracing path.
4. Realtime stable-plane path.
5. Compute pass framework.
6. RTXDI, denoising guides and other compute/RT chains.

Every pass is created through capability-gated construction. If a pass is unavailable, it must not leave the sample in a partially constructed state. The render path either falls back to a previous runnable path or disables that feature with a visible/logged reason.

## Shader Strategy

Shader sources live under:

```text
DiligentSamples/Samples/RTXPT/assets/shaders
```

The port should preserve RTXPT's shader directory structure where practical. Diligent shader creation controls:

- Include paths.
- Entry points.
- Shader type.
- Macros.
- DXIL/SPIR-V target selection.
- Backend-specific feature macros.

Binding requirements:

- C++ resource slots and HLSL register declarations must be updated together.
- Shader resource changes require updates to `RTXPTBindingModel`, SRB creation, render target ownership and all affected HLSL bindings.
- Base shader variants must compile for both D3D12 and Vulkan.
- Advanced D3D12-only shader paths must compile out cleanly on Vulkan.

## Post-Processing Strategy

The final display chain grows toward:

```text
OutputColor
  -> accumulation or no-denoiser final merge
  -> basic tone mapping
  -> optional bloom
  -> optional TAA
  -> optional NRD standalone denoise
  -> optional DLSS/DLSS-RR/Streamline
  -> debug overlays / shader debug / zoom
  -> swapchain
```

The first runnable version only needs a basic path from `OutputColor` to the swapchain. Each later post-process feature must be independently gated.

## Open Work Registry And Code TODO Policy

All incomplete migration work must be marked in code with this format:

```cpp
// TODO(RTXPT-Port Phase 4): Restore stable-plane fill pass; current fallback uses reference path tracing only.
```

Shader files use the same searchable form:

```hlsl
// TODO(RTXPT-Port Phase 5): Restore material-specialized alpha-test any-hit shader variant.
```

Each structured TODO must include:

- Phase number.
- Missing capability.
- Current fallback behavior.
- Direction for completion.

During phase review, run a targeted search such as:

```powershell
rg "TODO\(RTXPT-Port" DiligentSamples/Samples/RTXPT
```

Initial open-work registry:

- Phase 2: Copy the complete RTXPT asset closure after the first default scene works.
- Phase 2: Add full RTXPT `.scene.json` adaptation above AssetLoader.
- Phase 3: Add dynamic and skinned BLAS update.
- Phase 3: Restore OMM flags and alpha/opaque mapping.
- Phase 3: Restore acceleration structure compaction and incremental update paths.
- Phase 4: Add stable-plane pre-pass and fill-stable-planes dispatch.
- Phase 4: Add RTXDI DI/GI dispatch chain.
- Phase 4: Add light feedback and denoising guide compute chains.
- Phase 5: Restore material permutation and full hit group table generation.
- Phase 5: Restore alpha-test any-hit specialization.
- Phase 5: Restore NVAPI/SER shader variants where supported.
- Phase 5: Restore OMM shader integration.
- Phase 6: Restore NRD standalone denoise.
- Phase 6: Restore DLSS/DLSS-RR/Streamline integration.
- Phase 6: Restore bloom, TAA, shader debug, zoom and debug overlays.

When a task is completed, the corresponding structured TODO should be removed or narrowed. New gaps discovered during implementation must be added as structured TODOs and, when broad enough, reflected in the open-work registry.

## Error Handling And Fallbacks

- If the device lacks required ray tracing support, the sample launches in capability/debug mode.
- If a pass cannot be created, that feature is disabled and the previous runnable render path is used.
- If an asset is missing, the sample reports the path and falls back to a minimal procedural/default scene when possible.
- If a shader variant fails due to an unsupported advanced path, the feature must compile out behind a macro; base variants should remain available.
- Resize and render-target recreation must leave the sample in a valid state.

## Verification Approach

Do not claim a phase is complete without evidence from the corresponding targeted verification. Build or run commands are not run automatically unless explicitly requested by the user or required by the active task rules.

Suggested phase checks:

- Phase 1: Configure/build target and launch `RTXPT` capability view.
- Phase 2: Launch `RTXPT` with copied assets and render fallback output.
- Phase 3: Launch on D3D12 and Vulkan capable devices and verify AS creation or graceful disable.
- Phase 4: Verify minimal TraceRays output and compute pass execution.
- Phase 5: Verify each shader layer compiles and the sample still renders a fallback or migrated result.
- Phase 6: Verify each post-process feature can be toggled without breaking the base display chain.

## Implementation Plan Handoff

After this design is approved, the next step is to create an implementation plan that breaks the work into small runnable tasks. The first plan should focus on Phase 1 and the minimal Phase 2 skeleton:

- Add sample directory and CMake integration.
- Add `RTXPTSample` lifecycle.
- Add `RTXPTFeatureCaps`.
- Add clear/debug render path.
- Add initial asset directory and default-scene selection policy.
- Add the first structured TODO registry entries in code where fallback behavior is used.
