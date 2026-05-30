# RTXPT Runtime Scene Switching Design

## Summary

This design adds runtime scene switching to the Diligent RTXPT sample, matching the user-facing workflow used by `D:/RTXPT-fork`: the sample builds an available scene list at startup by enumerating `*.scene.json` files and exposes that list in the ImGui panel. Selecting another scene reloads the current scene resources and resets the camera to the selected scene's default camera.

The first implementation is intentionally a lightweight fork-parity step. It makes runtime switching usable without pulling the full RTXPT scene graph adapter into this change. The selected `.scene.json` file supplies the scene cameras and the first model path from its `models` array; full multi-model graph composition remains out of scope.

## Confirmed Requirements

- Build the available scene list at startup by enumerating `*.scene.json` files under the resolved RTXPT assets root.
- Expose a `Scene` combo in the RTXPT ImGui UI, similar to RTXPT-fork.
- Switch scenes at runtime from the UI.
- After switching scenes, reset to the selected scene's default camera.
- Reset the default camera projection state, including FOV and zNear/zFar, from the selected scene camera when available.
- Rebuild scene-dependent GPU resources after a scene switch.
- Reset accumulation after a scene switch.
- Keep the first implementation focused on the selected scene file and `models[0]`; do not implement full `.scene.json` graph composition in this step.

## Non-Goals

- Full RTXPT `.scene.json` graph adaptation with multiple model nodes, per-node transforms, custom material JSON overlays, and game props.
- Incremental hot-reload of individual resources.
- Asynchronous scene loading.
- Rollback to the previous scene when a new scene fails to load.
- New renderer features, shader behavior changes, or path tracer algorithm changes.

## Architecture

`RTXPTSample` remains the lifecycle, UI, and GPU-resource orchestration owner. It gains the available-scene list and the current-scene selection API:

- `m_AvailableScenes`: sorted file names such as `bistro-programmer-art.scene.json`.
- `m_CurrentSceneName`: the currently selected scene file.
- `EnumerateAvailableScenes()`: scans the resolved asset root for direct child files ending in `.scene.json`.
- `SetCurrentScene(const std::string& SceneName, bool ForceReload = false)`: reloads the selected scene and rebuilds scene-dependent GPU resources.

`RTXPTScene` remains the owner of the currently loaded CPU scene data:

- It gains a general `LoadScene(...)` entry point that accepts a scene file name.
- `LoadDefaultScene(...)` becomes a small preferred-scene wrapper or compatibility path around `LoadScene(...)`.
- It reads the selected `.scene.json`, validates `models[0]`, loads that glTF model through the existing Diligent GLTF loader, loads scene cameras, and caches scene statistics.

Scene enumeration intentionally belongs to `RTXPTSample`, not `RTXPTScene`. `RTXPTSample` needs the list for UI and initial selection. `RTXPTScene` only represents the currently loaded scene.

GPU resource rebuilds also stay in `RTXPTSample`. The scene object does not own materials, lights, acceleration structures, ray tracing passes, render targets, or UI state.

## Startup Flow

1. `RTXPTSample::Initialize()` resolves `m_AssetsRoot` with the existing asset-root helper.
2. `EnumerateAvailableScenes()` scans `m_AssetsRoot` for direct child `*.scene.json` files.
3. The sample chooses `bistro-programmer-art.scene.json` if it is present.
4. If the preferred scene is absent, the sample chooses the first sorted scene file.
5. If no scene files are found, the sample records a clear error and keeps the fallback render path active.
6. The selected initial scene is loaded through `SetCurrentScene(SelectedScene, true)`.

## Runtime Switch Flow

When the user selects a different scene in the ImGui combo:

1. `SetCurrentScene()` returns early if the selected scene is already current and `ForceReload` is false.
2. Current scene-dependent state is reset:
   - `RTXPTMaterials`
   - `RTXPTLights`
   - `RTXPTAccelerationStructures`
   - `RTXPTRayTracingPass`
   - selected scene camera index
   - accumulated sample state and camera history
3. `RTXPTScene::LoadScene()` loads the selected scene file.
4. If loading succeeds, `RTXPTSample` rebuilds:
   - material buffer and texture bindings
   - light buffer
   - static BLAS / TLAS
   - ray tracing pass bindings and SBT
5. If the loaded scene has at least one scene camera, `ApplySceneCamera(0)` becomes the post-load default.
6. If there is no scene camera, `InitializeCamera()` restores the free camera defaults.
7. Accumulation is reset with reason `Scene changed`.

This is a full reload, not an incremental swap. The full reload keeps the CPU scene, material buffer, light buffer, TLAS, and RT pass bindings in sync.

## Scene File Parsing

The first runtime switching step only needs enough scene parsing to choose the active glTF model and camera defaults.

`RTXPTScene::LoadScene()` reads the selected `.scene.json` with the existing `nlohmann::json` path used by `LoadSceneCameras()`:

- The root must be an object.
- `models` must be a non-empty array.
- `models[0]` must be a non-empty string.
- The selected model path is resolved relative to `m_AssetsRoot`.
- The model file must exist before constructing `GLTF::Model`.

The existing camera parser continues to read cameras from `graph` and animated cameras from `animations`. Camera loading should not be required for a scene to load. A scene with no cameras falls back to the default free camera after load.

The selected scene name remains the file name, not an absolute path. The UI and logs should show the short file name, while `m_ModelPath` stores the resolved model path for diagnostics.

## Camera And Projection Reset

Scene switching resets camera state by design.

If `RTXPTScene::GetCameraCount() > 0`, `ApplySceneCamera(0)` is called after scene resource rebuild. That function already updates:

- camera position and orientation
- `m_CameraVerticalFov`
- `m_CameraNearPlane`
- `m_CameraFarPlane`
- projection matrix
- selected scene camera index
- accumulation reset request

If no scene camera exists, `InitializeCamera()` restores the default free camera. The selected camera index is set to `-1`.

`m_HasLastCameraMatrices` is cleared after a scene switch so the next update frame does not compare the new camera against stale matrices from the previous scene.

## UI Design

The Scene section gains a `Scene` combo above the existing scene camera combo.

The combo preview is:

- the current scene file when loaded or selected,
- `none` when no scene is selected,
- `no scenes found` when enumeration found no available files.

The selectable entries are the sorted `m_AvailableScenes` file names.

The existing diagnostics stay visible:

- scene loaded / missing state
- scene file
- model path
- scene cameras
- load error
- mesh nodes
- primitives
- materials
- lights

This keeps the feature visible without adding a new window or changing the broader RTXPT UI layout.

## Failure Handling

Scene loading is fail-closed. If a selected scene fails to load, the sample does not keep partial resources from the previous scene.

Failure cases:

- No `.scene.json` files were found under the assets root.
- The selected scene file does not exist.
- The scene JSON is invalid or not an object.
- The `models` array is absent or empty.
- `models[0]` is not a usable string.
- The resolved model file is missing.
- `GLTF::Model` construction throws.
- Scene-dependent GPU resource creation fails after the scene model loads.

Expected behavior:

- `RTXPTScene::GetLastError()` reports the specific reason and path when relevant.
- Scene-dependent GPU managers are reset.
- `RTXPTRayTracingPass` is not ready.
- The existing fallback render path remains active.
- The user may select another scene from the combo.

The first implementation does not roll back to the previous scene. Avoiding rollback keeps state ownership simple and prevents mixed old/new GPU resources.

## Files

Expected implementation touch points:

- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

No shader file changes are expected.

No asset files are expected to be added. The feature consumes the existing `DiligentSamples/Samples/RTXPT/assets/*.scene.json` files.

## Validation

Targeted validation:

- `git diff --check -- DiligentSamples/Samples/RTXPT/src/RTXPTScene.hpp DiligentSamples/Samples/RTXPT/src/RTXPTScene.cpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Build the RTXPT sample target in the existing Debug x64 build tree when available.
- If no usable build tree exists, run the smallest available configure/build path and report any environment limitation.

Manual smoke criteria:

- Launch RTXPT.
- The Scene combo lists the `*.scene.json` files present under the resolved assets root.
- The initial scene is `bistro-programmer-art.scene.json` when present.
- Switching to `kitchen.scene.json` changes the shown scene file, model path, camera list, and scene statistics.
- Switching to another available scene, such as `living-room.scene.json` or `convergence-test.scene.json`, also rebuilds the scene statistics.
- The camera resets to the new scene's default camera after each switch.
- Accumulation restarts after each switch.
- If a scene cannot load, the UI shows a clear error and the fallback render path remains stable.

## Acceptance Criteria

- Runtime scene switching is available from the RTXPT ImGui UI.
- Available scenes are discovered from startup enumeration of local `.scene.json` files.
- Scene switching rebuilds scene-dependent CPU and GPU resources through one reload path.
- The post-switch camera uses the new scene's default camera when available.
- FOV and clip planes reset from that camera.
- No previous-scene resources remain bound after a failed scene load.
- The implementation remains scoped to the lightweight `models[0]` loading path and does not attempt full scene graph adaptation.
