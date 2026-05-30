# RTXPT Skinned glTF / Skinned Vertex Buffer / Dynamic BLAS Design

## Summary

This design defines the missing geometry pipeline needed for animated / skinned glTF to be safe for RTXPT-style ray tracing. The goal is not to rework the whole sample architecture; it is to make one contract true and keep it true:

**the skinned mesh vertices seen by the ray tracer, the skinned mesh vertices used to build / update BLAS, and the skinned mesh vertices used by any later emissive-triangle builder must all come from the same current GPU buffer for the same frame.**

That contract is the prerequisite for `docs/superpowers/plans/2026-05-30-rtxpt-phase-r2-emissive-area-lights.md`, which assumes the emissive-triangle builder can consume current-frame GPU geometry and must not fall back to bind-pose data.

The current repository already has partial pieces:

- `DiligentFX/PBR/src/GLTF_PBR_Renderer.cpp` already writes skinning joint data from `GLTF::ModelTransforms::Skins[...]` into the raster path's joints buffer.
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp` already builds static BLAS from the loaded GLTF vertex / index buffers.
- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp` explicitly marks dynamic / skinned BLAS update as future work.
- `DiligentCore/Graphics/GraphicsEngine/interface/DeviceContext.h` already supports `BuildBLAS(Update=true)` for updateable BLAS.

The missing part is a current-frame skinned vertex buffer path that can feed both ray tracing and any later geometry-derived light extraction without CPU readback.

## Relationship To The RTXPT Roadmap

This spec is a prerequisite for RTXPT R2 emissive area lights, but it does not implement emissive NEE itself.

It sits between the existing static RTXPT geometry path and the later lighting work:

- Static glTF remains on the existing BLAS / TLAS path.
- Skinned glTF gets a current-frame GPU vertex buffer.
- Dynamic BLAS updates or refits use that buffer.
- RTXPT consumers can then read the same current-frame geometry.

## Confirmed Requirements And Decisions

- **One current geometry source.** Do not read skinned geometry back to the CPU. The current-frame GPU buffer is the source of truth.
- **GPU skinning, not bind-pose fallback.** Skinned geometry must be produced on GPU before BLAS update and before ray dispatch. If the skinned path is unavailable, the sample must fail closed or disable that branch with a clear reason.
- **Stable topology.** This spec covers skinned meshes whose topology does not change across frames. Vertex positions and normals may change; vertex count, index count, and primitive ordering stay stable.
- **Shared vertex contract.** The skinned output buffer must match the RTXPT path-tracer vertex contract used by `GeometryVertexData` so the existing RT hit shaders can read it without a second format translation.
- **Rigid transforms stay in TLAS.** Skinning changes vertices. Node-level rigid transforms remain a TLAS concern.
- **Dynamic BLAS is updateable.** Skinned BLAS should be created with `RAYTRACING_BUILD_AS_ALLOW_UPDATE` and updated in place from the current skinned buffer when possible. Full rebuild is a fallback only if update cannot be used on a backend or configuration.
- **Mixed scenes are allowed.** Static primitives keep the existing static BLAS path. Skinned primitives use the dynamic path. A scene can contain both.
- **Backend scope.** D3D12 and Vulkan are first-class. If a backend cannot support the needed compute / ray tracing path, the feature is disabled explicitly rather than approximated with stale geometry.

## Scope Boundary

### In scope

- glTF skin detection and classification.
- Current-frame skinned vertex buffer generation.
- Updateable / rebuildable BLAS for skinned primitives.
- A scene-level contract that exposes current-frame geometry to RTXPT consumers.
- Update scheduling tied to animation / skinning dirtiness.

### Out of scope

- Emissive triangle extraction and emissive NEE / MIS.
- Light importance sampling / RIS.
- CPU skinning fallback.
- Morph targets.
- OMM, compaction, RTXDI, NRD, DLSS, SER, or other realtime-track features.
- Rewriting the GLTF viewer or PBR renderer.

## Design

### Geometry Ownership

The skinned path owns a dedicated current-frame vertex buffer. That buffer is not a transient scratch copy; it is the buffer that downstream systems read after skinning completes.

The buffer should carry the same fields RTXPT already consumes in `GeometryVertexData`:

- position
- normal
- texcoord0

That keeps the path tracer hit shaders unchanged in shape and makes the skinned path a format-preserving extension instead of a second geometry language.

### Skinning Data Flow

The scene already knows when a node references a skin and when the model has joint transforms for the current frame.

The new data flow is:

1. Load glTF scene and classify primitives.
2. Gather joint matrices and any per-node skin metadata for the current frame.
3. Run GPU skinning into the current-frame vertex buffer.
4. Build or update the skinned BLAS from that buffer.
5. Make the same buffer visible to RT consumers.

The key rule is that steps 3, 4, and 5 all use the same frame-local data.

### BLAS Strategy

For skinned primitives, the BLAS is created as updateable and refreshed from the current skinned buffer every frame the skin is dirty.

This keeps topology stable and avoids reauthoring the scene as a different mesh every frame. It also matches the natural semantics of skinning: the mesh deforms, but its topology does not.

If a backend or configuration cannot support the updateable path, the skinned branch is disabled explicitly. The implementation must not quietly point RT consumers back at bind-pose geometry.

### Scene Classification

The scene should distinguish at least these cases:

- static mesh, no vertex deformation
- rigid animated node, transform changes only
- skinned mesh, vertex deformation changes every frame

Rigid animation can remain a TLAS concern. Skinned meshes require the current-frame vertex buffer and dynamic BLAS.

## Phases

### Phase S0: Scene Classification And Metadata

Goal: identify which primitives are static, which are rigid-animated, and which are skinned.

Runnable milestone:

- static scenes continue to load and render unchanged
- skinned meshes are detected and reported
- the sample can say which branch is active without guessing

### Phase S1: Current-Frame Skinned Vertex Buffer

Goal: generate the skinned path-tracer vertex buffer on GPU from the current joint data.

Runnable milestone:

- a skinned glTF asset produces a current-frame vertex buffer
- the buffer matches the RT vertex contract
- no CPU geometry readback is needed

### Phase S2: Dynamic BLAS Update

Goal: create updateable BLAS for skinned primitives and refresh them from the current skinned buffer.

Runnable milestone:

- skinned geometry updates in place frame to frame
- static primitives remain on the original static BLAS path
- D3D12 and Vulkan both keep a valid runnable path

### Phase S3: RTXPT Geometry Handoff

Goal: expose the same current-frame buffer to RTXPT consumers so later emissive-triangle extraction and closest-hit fetches see identical geometry.

Runnable milestone:

- RTXPT can bind and consume skinned geometry without special-casing bind-pose data
- a future emissive-builder pass can use the same buffer without changing the geometry contract

## Cross-Cutting Contracts

- **No mixed-frame geometry.** Skinning output, BLAS update, and RT fetch must be from the same frame.
- **No bind-pose fallback.** If the skinned buffer is not current, do not trace against it.
- **No hidden topology drift.** If a primitive changes topology, this spec does not cover it.
- **One format for consumers.** RT consumers should not need to know whether a primitive was static or skinned in order to read its current vertex data.
- **Rigid transform separation.** Instance transforms and skinning transforms are distinct responsibilities.
- **Backend capability gate.** Unsupported backends or feature combinations should disable the skinned branch explicitly and preserve sample launchability.

## Reference Implementation Anchors

These files are the main context for this spec:

- `DiligentSamples/Samples/GLTFViewer/src/GLTFViewer.cpp`
- `DiligentFX/PBR/src/GLTF_PBR_Renderer.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
- `DiligentCore/Graphics/GraphicsEngine/interface/DeviceContext.h`

The first two show how existing Diligent code already handles skinned GLTF data for raster rendering. The RTXPT files show the current static geometry path that needs a skinned current-frame extension.

## Verification Strategy

1. Static scenes still load and render as before.
2. Skinned scenes produce a current-frame vertex buffer on GPU.
3. The BLAS built from that buffer matches the same frame's geometry.
4. RT consumers read the same geometry the BLAS used.
5. If the skinned branch is disabled, the sample still launches and explains why.

The key correctness check is frame coherence, not visual style: the skinning result, BLAS, and RT hit data must agree on the same geometry for the same frame.

## Open-Work / TODO Marker Policy

Until this spec is implemented, RTXPT code that depends on skinned current-frame geometry should keep structured TODO markers instead of pretending bind-pose data is good enough.

Recommended marker shape:

`// TODO(RTXPT-Port Phase R2): requires current-frame skinned vertex buffer + dynamic BLAS update.`

