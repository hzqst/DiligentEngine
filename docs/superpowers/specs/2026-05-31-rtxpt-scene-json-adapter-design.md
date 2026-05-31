# RTXPT Scene JSON Adapter Design

## Summary

This design replaces the current lightweight `.scene.json` path in the Diligent RTXPT sample with a full RTXPT scene adapter. The current loader only reads `models[0]` from the selected scene file and then treats the scene as one `GLTF::Model`. The new adapter loads the full scene graph, all model assets, all graph model instances, per-node transforms, RTXPT material extension JSON files, light metadata, cameras, sample settings, and game settings.

The selected approach is upstream metadata parity with CPU-first consumption. The adapter preserves RTXPT-fork scene semantics in CPU data and diagnostics, while keeping the existing GPU-facing shader contracts as stable as possible. This means the first implementation should not expand `MaterialPTData`, `PolymorphicLightInfo`, `SubInstanceData`, or HLSL resource layouts unless a small local change is strictly required to make full graph rendering correct.

Multi-model graph rendering is in scope. Per-model and per-instance skinning with independent per-instance animation state is also in scope. Advanced material and environment rendering features such as transmission, nested dielectrics, HDR environment-map importance sampling, and specialized material permutations are parsed and preserved as metadata, but their full shader consumption remains assigned to later RTXPT phases.

## Confirmed Requirements

- Load every path in the `.scene.json` `models` array.
- Load the recursive `graph` array instead of only selecting `models[0]`.
- Support graph model instances, nested graph nodes, per-node transforms, and parent transform inheritance.
- Support `translation`, `rotation`, `euler`, and `scaling` fields with RTXPT-fork-compatible defaults.
- Support relaxed scene JSON input, including trailing commas in existing assets such as `living-room.scene.json`.
- Preserve upstream material metadata from `*.material.json` files on the CPU.
- Preserve upstream light metadata, including `EnvironmentLight`, `DirectionalLight`, `PointLight`, `SpotLight`, and light extension fields such as proxy mesh nodes where present.
- Preserve camera, `SampleSettings`, and `GameSettings` metadata.
- Keep GPU-facing material, light, and sub-instance buffer layouts stable in this phase.
- Make multi-model graph instances render through the current reference path tracer.
- Support per-model and per-instance skinning with independent per-instance animation state.
- Keep `RTXPTSample` as the owner of UI, scene switching, and GPU resource rebuild orchestration.
- Keep `RTXPTScene` as the owner of loaded CPU scene data and scene adapter diagnostics.

## Non-Goals

- Do not implement full transmission, nested dielectric, or alpha-blend path tracing behavior in this phase.
- Do not implement HDR environment-map importance sampling in this phase.
- Do not add RTXPT-fork material shader permutations in this phase.
- Do not import Donut or NVRHI scene graph classes into the Diligent sample.
- Do not rewrite the reference path tracer algorithm.
- Do not make scene loading asynchronous.
- Do not add incremental hot reload of individual scene files, material files, or model assets.
- Do not keep rendering bind-pose fallback geometry when required skinned scene resources fail to initialize.

## Architecture

`RTXPTSample` remains the lifecycle owner. It still enumerates scene files, responds to UI scene selection, resets scene-dependent state, rebuilds resources, and creates or resets the ray tracing pass.

`RTXPTScene` becomes the adapter owner. It reads the selected `.scene.json`, loads all referenced glTF files, builds an RTXPT-owned scene graph representation, caches metadata, updates per-instance animation state, and exposes read-only traversal APIs to the GPU resource managers.

The adapter should add an RTXPT scene data layer, tentatively named `RTXPTSceneData` or `RTXPTSceneGraph`. It must not try to synthesize one fake `GLTF::Model` from all model assets. Each glTF remains an independent `GLTF::Model`, and the RTXPT layer composes the complete scene above those models.

The scene-dependent managers should consume this scene view instead of a single `GLTF::Model`:

- `RTXPTMaterials` uploads a global material table and global material texture bindings.
- `RTXPTLights` uploads the subset of parsed lights that the current GPU light contract can express and preserves the rest in diagnostics.
- `RTXPTAccelerationStructures` builds BLAS and TLAS records by traversing all model instances.
- `RTXPTSkinnedGeometry` is upgraded or wrapped as `RTXPTSkinnedSceneGeometry`, a scene-level manager for per-instance animation and skinned vertex output.
- `RTXPTRayTracingPass` continues to bind the same logical GPU resources. Any vertex/index scene bridge change must stay narrowly scoped and must not broaden the material or light shader contracts in this phase.

## Data Model

The adapter should keep explicit IDs for every scene-level entity to avoid pointer-only coupling:

- `ModelAssetId`: index into loaded model assets.
- `GraphNodeId`: index into parsed graph nodes.
- `ModelInstanceId`: index into graph nodes that instantiate a model.
- `MaterialGlobalId`: index into the global material table.
- `TextureGlobalId`: index into the global material texture binding list.
- `LightId`: index into parsed scene lights.
- `CameraId`: index into parsed scene cameras.

### Model Assets

Each `ModelAsset` represents one entry in `models[]`:

- original relative path
- resolved absolute path
- model name derived from the model file name, following RTXPT-fork material lookup behavior
- loaded `GLTF::Model`
- selected glTF scene index
- model-local default transforms
- material id remap table
- texture id remap table
- geometry and animation statistics

The same `ModelAsset` may be referenced by multiple graph model instances.

### Graph Nodes

Each `GraphNode` represents one recursive `.scene.json` graph node:

- name
- optional type string
- parent id
- children ids
- local transform
- global transform
- optional model asset id
- optional typed metadata id
- raw JSON metadata for unknown or deferred fields

The transform parser should use these defaults:

- `translation`: `{0, 0, 0}`
- `rotation`: XYZW quaternion, default identity
- `euler`: fallback when `rotation` is absent, using RTXPT-fork-compatible Euler conversion
- `scaling`: scalar or vec3, default `{1, 1, 1}`

Global graph transforms are computed recursively from parent to child.

### Model Instances

Each graph node with a valid `model` field creates one `ModelInstance`:

- graph node id
- model asset id
- instance name
- graph global transform
- per-instance animation state
- model-local transform cache
- optional skinned instance records

For example, `MechDrone` and `MechDroneInMicrowave` in `kitchen.scene.json` both reference `model: 4`. They share the same `ModelAsset`, but they are separate `ModelInstance` records with independent graph transforms and independent animation state.

### Material Extensions

Each parsed RTXPT material extension stores the upstream PTMaterial fields that exist in `*.material.json`, including at least:

- base/diffuse color
- specular color
- emissive color and intensity
- metalness
- roughness
- opacity
- alpha test toggle and cutoff
- texture toggles and texture paths
- normal texture scale
- transmission and diffuse transmission factors
- IoR
- thin surface
- nested priority
- volume attenuation fields
- NEE exclusion flags
- path-space decomposition flags
- analytic light proxy flag
- skip render flag
- raw JSON for unknown fields

The current GPU material upload only consumes fields already expressible by `MaterialPTData`. Deferred fields remain available through CPU metadata and diagnostics.

### Light Metadata

The adapter should parse and preserve:

- `DirectionalLight`
- `PointLight`
- `SpotLight`
- `EnvironmentLight`
- point and spot light extension metadata such as `proxyMeshNodes`
- raw JSON for unknown light fields

The GPU light buffer only uploads lights that can be represented by the current `PolymorphicLightInfo` layout. `EnvironmentLight` metadata is preserved and shown in diagnostics, but HDR environment-map rendering remains a later phase.

### Settings Metadata

`SampleSettings` should be parsed into typed optional fields such as:

- `realtimeMode`
- `enableAnimations`
- `startingCamera`
- `realtimeFireflyFilter`
- `maxBounces`
- `maxDiffuseBounces`
- `textureMIPBias`

`GameSettings` should preserve its JSON payload for later game prop loading and diagnostics. The first implementation should avoid wiring settings into unrelated renderer behavior unless the mapping is already obvious and safe.

## Loading Flow

`RTXPTScene::LoadScene()` should follow one fail-closed transaction:

1. Reset temporary scene data.
2. Resolve and validate the scene file path.
3. Load the file through a shared relaxed JSON parser.
4. Validate the root object, `models` array, and `graph` array.
5. Load all model assets referenced by `models[]`.
6. Recursively build graph nodes and model instances from `graph[]`.
7. Parse cameras, lights, sample settings, game settings, and unknown typed metadata.
8. Parse RTXPT material extension files for every loaded glTF material.
9. Build global material and texture remap tables.
10. Initialize per-instance animation state.
11. Cache scene statistics and diagnostics.
12. Commit the temporary data into `RTXPTScene` only after all structural steps succeed.

If a structural step fails, no partial scene data should remain visible to `RTXPTSample`.

## Relaxed JSON Parser

The adapter needs one JSON entry point for scene and material files. It should read the whole file into text, preprocess it conservatively, and then call `nlohmann::json`.

The parser must support trailing commas in arrays and objects because existing RTXPT assets include them. The parser should also use comment-tolerant parsing when available, or preprocess comments only if the implementation can do so without corrupting string literals.

The relaxed parser should report:

- file path
- whether strict parse failed
- whether relaxed parse succeeded
- parse error detail when available

This diagnostic is useful for asset cleanup without blocking current asset loading.

## Material Lookup

Material extension lookup follows RTXPT-fork `MaterialsBaker::Load()` ordering:

1. `Materials/<scene-file-stem>/<modelName>.<materialName>.material.json`
2. `Materials/<scene-file-stem>/<materialName>.material.json`
3. `Materials/<modelName>.<materialName>.material.json`
4. `Materials/<materialName>.material.json`

The scene file stem should match upstream behavior. For `bistro-programmer-art.scene.json`, the scene-specialized directory is `Materials/bistro-programmer-art.scene`.

If a material extension file is missing or invalid, the adapter uses the glTF material data and records a fallback diagnostic. Missing material extension files do not fail the scene.

## GPU Resource Integration

### Materials

`RTXPTMaterials::Upload()` should accept the RTXPT scene view. It builds a global material table by traversing every `ModelAsset` and every glTF material in that asset.

For each global material record:

- Start from the glTF material attributes.
- Apply only CPU-parsed RTXPT extension fields that map cleanly to current `MaterialPTData`.
- Remap glTF material texture ids into global texture ids.
- Preserve unsupported extension fields in CPU metadata.

The material texture binding list becomes a global list across all model assets. Shader-side material texture indices remain simple global indices and do not need to know which glTF model supplied the texture.

### Acceleration Structures

`RTXPTAccelerationStructures::BuildScene()` should accept the RTXPT scene view. It traverses every `ModelInstance`, then every mesh node and primitive in that instance's `ModelAsset`.

For static geometry, TLAS transforms use:

`graph instance global transform * glTF node global transform`

For each primitive, `SubInstanceData::MaterialID` stores the global material id from the model asset material remap table. Vertex and index offsets remain relative to the corresponding `GLTF::Model` buffers or the skinned scene vertex arena.

The TLAS remains a single scene-level TLAS. BLAS records should retain enough metadata to identify:

- model asset id
- model instance id
- glTF node pointer or node index
- primitive range
- skinned output range when applicable

### Lights

`RTXPTLights::Upload()` should accept the RTXPT scene view. It uploads the subset of parsed graph lights and glTF lights that fit the current GPU light layout.

Directional, point, and spot lights should be transformed by their graph/global transforms. Environment lights are tracked in CPU metadata and diagnostics in this phase. The renderer continues to use the current procedural environment path until the HDR environment-map phase consumes the environment metadata.

### Ray Tracing Pass

`RTXPTRayTracingPass` should continue binding the same logical resources:

- TLAS
- material buffer
- material texture array
- light buffer
- sub-instance buffer
- source vertex buffer or global static vertex access path
- skinned vertex buffer
- index buffer or global index access path

If multi-model source vertex and index buffers cannot be represented by the current pass bindings, the implementation should add a minimal scene-level bridge that preserves the shader contract as much as possible. The design goal is still no broad HLSL material/light layout expansion in this phase.

## Skinned Scene Geometry

The current `RTXPTSkinnedGeometry` is a single-model helper. It accepts one `GLTF::Model`, one scene index, one source vertex buffer, one skinning buffer, and one `GLTF::ModelTransforms` object.

Full scene graph support requires a scene-level manager, tentatively named `RTXPTSkinnedSceneGeometry`. The name matters because the ownership and index space change from one model to the whole composed scene.

`RTXPTSkinnedSceneGeometry` should manage:

- all skinned model instances in the composed scene
- per-instance animation state
- per-instance `GLTF::ModelTransforms`
- a global skinned vertex arena
- global joint matrix ranges
- skinning jobs
- mapping from `ModelAssetId + ModelInstanceId + NodeIndex` to skinned output ranges
- dynamic BLAS update metadata

Each skinned `ModelInstance` gets independent animation state:

- animation enabled flag
- animation index
- animation time
- play speed
- optional per-instance time offset

`MechDrone` and `MechDroneInMicrowave` should therefore share their source `ModelAsset`, but have independent instance state and independent skinned output ranges when the model contains skinned geometry. Correctness should not rely on shared-pose optimization. Future optimization may share output between instances only when their full pose state is provably identical.

The skinning compute path can continue using the existing shader shape:

- source vertex buffer
- source skinning buffer
- joint matrix buffer
- output skinned vertex arena
- per-job constants

However, the job list must become scene-level. Each dispatch selects the proper source buffers, joint range, destination range, and vertex count for one skinned node instance.

Dynamic BLAS updates must use the same skinned output ranges that closest-hit shading reads. Bind-pose fallback is not allowed for skinned geometry in this phase.

## Animation Flow

On scene load:

1. Each `ModelInstance` initializes its animation state from its model asset and scene settings.
2. Instances with animated or skinned models compute initial transforms.
3. Skinned scene geometry allocates all output ranges and joint ranges.
4. The first skinning update runs before BLAS/TLAS construction so acceleration structures and closest-hit fetches reference the same current-frame geometry.

Each frame:

1. `RTXPTScene::Update()` advances per-instance animation state.
2. Animated instances recompute their model-local transforms with the instance root transform.
3. `RTXPTSkinnedSceneGeometry::Update()` uploads global joint matrices and dispatches skinning jobs.
4. `RTXPTAccelerationStructures::UpdateDynamicBLAS()` updates dynamic BLAS records and scene TLAS transforms for affected instances.
5. Accumulation resets when animated geometry changed.

## Failure Handling

Structural scene failures are fail-closed:

- scene file missing
- relaxed JSON parse failure
- missing or invalid root object
- missing or invalid `models` or `graph`
- model path missing
- model index out of range when required for a model instance
- glTF model load failure
- scene-level resource rebuild failure
- skinned scene geometry initialization failure when skinned graph instances are present
- dynamic BLAS update failure for skinned geometry

Soft metadata failures are fail-soft:

- missing material extension file
- invalid optional material extension field
- invalid optional light extension field
- unknown `type`
- invalid optional settings field

Soft failures are logged and counted in diagnostics. Structural failures reset scene-dependent GPU resources and keep the fallback render path stable.

## UI And Diagnostics

The existing RTXPT ImGui panel should remain the main UI surface. The Scene section should add or update diagnostics:

- scene graph nodes
- model assets
- model instances
- materials total
- material extensions loaded
- material fallbacks
- directional lights
- point lights
- spot lights
- environment lights
- unknown typed nodes
- static cameras
- animated cameras
- `SampleSettings` present
- `GameSettings` present
- skinned instances
- skinning jobs
- skinned vertices
- joint matrices
- dynamic BLAS records
- adapter warnings count

The UI should show recent adapter warnings when useful, but it should not become a general asset editor.

## Files

Likely implementation touch points:

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTMaterials.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTLights.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTLights.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTAccelerationStructures.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSkinnedGeometry.cpp`

The implementation may add focused helper files if keeping all adapter data structures inside `RTXPTScene.*` would make those files too large.

No asset files are expected to be changed as part of the adapter. Existing scene files, including non-strict JSON assets, should load through the relaxed parser.

## Validation

Targeted validation should include:

- Parse every `DiligentSamples/Samples/RTXPT/assets/*.scene.json` file through the relaxed adapter.
- Confirm `living-room.scene.json` succeeds despite its trailing comma in `models`.
- Confirm `kitchen.scene.json` creates separate `ModelInstance` records for `MechDrone` and `MechDroneInMicrowave`.
- Confirm multi-model scenes report model asset and instance counts greater than one.
- Confirm material extension lookup finds both scene-specialized and shared material files when present.
- Confirm missing material extension files fall back to glTF material data without failing the scene.
- Confirm `EnvironmentLight` nodes are counted and preserved as metadata.
- Confirm per-instance animation state exists for skinned model instances.
- Confirm skinned scene geometry produces independent skinned output ranges per skinned instance.
- Confirm `git diff --check` passes for touched files.
- Build the RTXPT sample target in the existing Debug x64 build tree when available.

Manual smoke criteria:

- Launch RTXPT.
- Scene combo still enumerates available `*.scene.json` files.
- Switching to `kitchen.scene.json` shows multiple model assets and model instances.
- `MechDrone` and `MechDroneInMicrowave` appear as separate graph model instances in diagnostics.
- Switching to `bistro-programmer-art.scene.json` reports many model assets and instances instead of only `models[0]`.
- Switching to `living-room.scene.json` succeeds through relaxed parsing.
- Fallback rendering remains stable when a deliberately invalid scene fails to load.

## Acceptance Criteria

- `.scene.json` loading uses the full `models` and recursive `graph` data.
- Multi-model graph instances render through the current RTXPT reference path tracer.
- Graph transforms affect rendered model instance placement.
- Material extension JSON files are discovered, parsed, preserved, and counted.
- Missing material extension files fall back cleanly to glTF material data.
- Light, camera, sample settings, and game settings metadata are parsed and preserved.
- GPU-facing material, light, and sub-instance struct layouts remain stable in this phase.
- Per-instance animation state is represented for model instances.
- Skinned graph instances use per-instance skinned output ranges and dynamic BLAS updates.
- No bind-pose fallback is used for skinned geometry when current-frame geometry is required.
- UI diagnostics make the adapter behavior visible enough to validate scene composition.

## Risks And Mitigations

- Multi-model source vertex and index buffers may not fit the current ray tracing pass binding shape. Mitigate by adding the smallest possible scene-level bridge while avoiding broad shader contract expansion.
- Per-instance skinned output can increase memory use. Mitigate with diagnostics and future shared-pose optimization only after correctness is established.
- Relaxed JSON preprocessing can corrupt strings if implemented carelessly. Mitigate by using library support where available or by writing a string-literal-aware preprocessor.
- Material names can collide across model assets. Mitigate with model-name-qualified lookup and global material ids.
- Unknown upstream metadata may appear in future assets. Mitigate by preserving raw JSON metadata and counting unknown typed nodes or fields.
