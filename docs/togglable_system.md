# Togglable Node System (window stays opaque — `_togglable_off` occluder not hidden)

Status: **designed, not yet implemented** — this document captures the root cause and the planned diffs for a
faithful port of RTXPT-fork's *Togglable node* system. Awaiting sign-off before editing code.
Scope: scene-graph node show/hide by glTF name suffix (`_togglable` / `_togglable_off`).
Repro scene: `assets/kitchen.scene.json` (kitchen window).

## Symptom

After the high-resolution env-cube fix ([window_environment_map_issue.md](window_environment_map_issue.md)),
the environment map is sharp, but the **kitchen window is still opaque** — a flat surface covers the opening
instead of showing the ocean that upstream RTXPT-fork displays.

## Root cause (this corrects the earlier env-cube-only diagnosis)

There **is** an occluder over the window: glTF node **`Mesh_295_togglable_off`** (node index 384, references
mesh 309, opaque `Material_295`), a rotated/scaled plane sitting in the window opening.

- **Upstream hides it.** RTXPT-fork has a *Togglable node* system: any scene-graph node whose name ends in
  `_togglable` (visible by default) or `_togglable_off` (**hidden** by default) is registered and, for the
  `_off` variant, immediately teleported out of the scene at load.
- **The port has no togglable handling at all** (grep `togglable` across `Samples/RTXPT/src` → 0 matches). So
  `Mesh_295_togglable_off` is built into the TLAS at its authored position and renders as an opaque plane over
  the window.

The env-cube fix was real but secondary (it sharpened the whole environment); the **primary** reason the window
is opaque is this un-hidden `_togglable_off` plane.

Confirmed in the port's byte-identical `assets/Models/Kitchen/kitchen.gltf`:

| Node | Index | Mesh | Children | Default |
|------|-------|------|----------|---------|
| `Ceiling_togglable`      | 346 | 277 | none (leaf) | **on**  |
| `Mesh_295_togglable_off` | 384 | 309 | none (leaf) | **off** |

Both are leaf mesh nodes (no children) — single-node teleport suffices for this scene; the port will still
handle node subtrees generically for faithfulness to other scenes.

## Upstream mechanism (RTXPT-fork)

| Concern | Location |
|--------|----------|
| `struct TogglableNode { SceneGraphNode*; double3 OriginalTranslation; string UIName; IsSelected(); SetSelected(); }` | `Rtxpt/SampleUI.h:51-58` |
| `UpdateTogglableNodes()` walks the scene graph; registers `_togglable` (default on) / `_togglable_off` (then `SetSelected(false)`) | `Rtxpt/SampleUI.cpp:2299-2326` |
| `SetSelected(false)` ⇒ `SceneNode->SetTranslation({-10000,-10000,-10000})`; `SetSelected(true)` ⇒ restore `OriginalTranslation`; `IsSelected()` ⇒ translation == original | `Rtxpt/SampleUI.cpp:2286-2297` |
| `TrimTogglable()` strips the `_togglable…` suffix for the display name | `Rtxpt/SampleUI.cpp:195-201` |
| Build list once after scene load: `UpdateTogglableNodes(*m_ui.TogglableNodes, GetScene()->GetSceneGraph()->GetRootNode())` | `Rtxpt/Sample.cpp:557-561` |
| Clear list on scene unload: `m_ui.TogglableNodes = nullptr` | `Rtxpt/Sample.cpp:421` |
| **"Togglables"** UI panel: one checkbox per node; toggling calls `SetSelected` + `ResetAccumulation = true` | `Rtxpt/SampleUI.cpp:555-567` |
| (Top-screen "big button" widgets reuse the same nodes) | `Rtxpt/SampleUI.cpp:1878-1879` |
| `LocalConfig` test-only hook toggles nodes by name | `Rtxpt/SampleCommon/LocalConfig.cpp:93-98` |

Upstream operates on donut's live `SceneGraphNode`; the renderer re-reads node transforms each frame, so a
toggle is reflected without an explicit rebuild.

## Port architecture differences

The port does **not** use donut's live scene graph. The kitchen is one monolithic glTF (`GLTF::Model`), and:

- Each glTF node with a mesh becomes its own **TLAS instance** during `RTXPTAccelerationStructures::BuildScene`
  (`src/RTXPTAccelerationStructures.cpp:255-426`).
- That instance's world transform is read from
  **`RTXPTModelInstance::Transforms.NodeGlobalMatrices[pNode->Index]`** (`:251-253`, `:340`).
- `NodeGlobalMatrices` is computed once at instance creation via `Model->ComputeTransforms(...)`
  (`src/RTXPTScene.cpp:672`) and is **not** recomputed per frame for static (non-animated) instances
  (`RTXPTScene::Update`, `src/RTXPTScene.cpp:1002-1038`). Togglable props are static, so a patched matrix
  persists.

**Consequence:** the faithful analog of upstream's `SetTranslation({-10000,…})` is to teleport the node's
(and its descendants') entries in `Transforms.NodeGlobalMatrices`. Because the TLAS is baked, a runtime toggle
must trigger an acceleration-structure rebuild (upstream gets this “for free” from its live scene graph).

## Planned implementation (diffs)

### 1. `src/RTXPTSceneGraph.hpp` — new struct

Add next to `RTXPTModelInstance` (faithful analog of `TogglableNode`):

```cpp
// Faithful port of RTXPT-fork's TogglableNode (D:/RTXPT-fork/Rtxpt/SampleUI.h / SampleUI.cpp).
// A scene-graph node whose glTF name ends in "_togglable" (visible by default) or "_togglable_off"
// (hidden by default) can be shown/hidden at runtime. Hiding teleports the node's subtree far out of the
// scene (matching upstream's SetTranslation({-10000,...})), so the geometry leaves the visible TLAS without
// rebuilding the BLAS. Toggling requires an acceleration-structure rebuild to take effect.
struct RTXPTTogglableNode
{
    RTXPTSceneId          ModelInstanceId = InvalidRTXPTSceneId; // Owning model instance.
    std::string           UIName;                                // Display name (suffix trimmed).
    bool                  Selected        = true;                // true = visible, false = hidden.
    std::vector<int>      NodeIndices;                           // Node + descendants (indices into NodeGlobalMatrices).
    std::vector<float4x4> OriginalMatrices;                      // Original global matrices, for restore.
};
```

### 2. `src/RTXPTScene.hpp` — owner of the list (mirrors `m_ui.TogglableNodes`)

- New member: `std::vector<RTXPTTogglableNode> m_TogglableNodes;`
- New public API:
  ```cpp
  void                                    BuildTogglableNodes();              // after scene load
  const std::vector<RTXPTTogglableNode>&  GetTogglableNodes() const;
  bool                                    SetTogglableSelected(size_t Index, bool Selected); // patches transforms
  ```
- Clear `m_TogglableNodes` in `ResetLoadedData()`.

### 3. `src/RTXPTScene.cpp` — logic (mirrors `UpdateTogglableNodes` / `SetSelected` / `TrimTogglable`)

- Anonymous-namespace helpers: `TrimTogglable()` (`rfind("_togglable")`) and a suffix test, replicating
  `SampleUI.cpp:195-201` / `:2305`.
- `BuildTogglableNodes()`:
  - For each `RTXPTModelInstance`, walk `Asset.Model->Scenes[SceneIndex].LinearNodes` (same iteration as the AS
    build).
  - Match name suffix `_togglable` (default on) else `_togglable_off` (default off), exactly like upstream's
    `addIfTogglable` ordering.
  - Record the matched node + descendant node indices and their original `Transforms.NodeGlobalMatrices`.
  - For `_togglable_off`, immediately call the hide path (teleport).
- `SetTogglableSelected(Index, Selected)`:
  - Hidden ⇒ for each stored node index, write `OriginalMatrix` with its translation row offset by
    `(-10000,-10000,-10000)` (Diligent row-major `_41/_42/_43`); Visible ⇒ restore `OriginalMatrix`.
  - Update `Selected`; return whether it changed.

### 4. `src/RTXPTSample.hpp` — deferred rebuild flag

- New member: `bool m_TogglableNodesDirty = false;`

### 5. `src/RTXPTSample.cpp` — build, UI, rebuild (mirrors `Sample.cpp` + `SampleUI.cpp`)

- **Build after load** in `SetCurrentScene`: after `LoadScene` succeeds and `ApplySceneEnvironmentSettings()`,
  **before** `RebuildSceneDependentResources()` (so default-off nodes are hidden in the very first
  `BuildScene`): `m_Scene.BuildTogglableNodes();`  (analog of `Sample.cpp:557-561`).
- **UI panel** in `UpdateUI()` — add `if (ImGui::CollapsingHeader("Togglables")) { … }` (placed near the
  existing "Scene" / "Environment Map" headers), one `ImGui::Checkbox(UIName)` per node (analog of
  `SampleUI.cpp:555-567`). On change: `m_Scene.SetTogglableSelected(i, sel)`, set
  `m_TogglableNodesDirty = true`, and `RequestAccumulationReset("Togglable node changed")`.
- **Deferred rebuild** in `Update()` (before path tracing): if `m_TogglableNodesDirty`, call
  `RebuildSceneDependentResources()` and clear the flag. (Reuses the proven full-rebuild path; see *Decisions*.)

### 6. `docs/window_environment_map_issue.md` — correction

Note that the **primary** cause of the opaque window is this un-hidden `_togglable_off` occluder; the env-cube
resolution was a secondary improvement. Cross-link this document.

## Decisions / trade-offs

- **Teleport vs. skip-from-TLAS.** Teleport (faithful to upstream) keeps the instance in the BVH and moves it
  out of view. Chosen over dropping the instance, which would diverge from upstream semantics.
- **Translation delta vs. absolute.** Apply a `(-10000,-10000,-10000)` *delta* per subtree node (preserves
  relative layout while hidden); restore from stored originals. Upstream sets an absolute translation on a
  single node; the delta form generalizes correctly to subtrees.
- **Full rebuild on toggle.** Toggling sets `m_TogglableNodesDirty` and re-runs `RebuildSceneDependentResources()`
  (rebuilds BLAS+TLAS, re-uploads materials/lights, re-bakes env/emissive). Heavier than a TLAS-only refit, but
  it is the existing, tested entry point and toggling is a rare manual action; it is also correct if an
  emissive/area-light prop is ever toggled. A lighter TLAS-only refit is a possible follow-up.
- **Skip `LocalConfig`.** Upstream's `LocalConfig.cpp` togglable hook is a hardcoded test-config subsystem the
  port doesn't have; out of scope for this feature.
- **Subtrees.** Both kitchen togglables are leaves, but `BuildTogglableNodes` collects descendants so group
  togglables in other scenes also hide correctly.

## Fork mapping

Names/structure kept aligned with `RTXPT-fork` per `RTXPT_FORK_MAPPING.md`:
`RTXPTTogglableNode` ↔ `TogglableNode`, `BuildTogglableNodes` ↔ `UpdateTogglableNodes`,
`SetTogglableSelected` ↔ `TogglableNode::SetSelected`, `TrimTogglable` ↔ `TrimTogglable`, the **"Togglables"**
panel ↔ the upstream "Togglables" collapsing header.

## Verification (pending)

1. Build; open `kitchen.scene.json` — the window should show the ocean (the `_togglable_off` plane hidden by
   default), matching upstream.
2. UI ⇒ **Togglables**: toggling `Mesh_295` on shows the plane (window opaque again); off hides it. Toggling
   `Ceiling` (default on) off opens the ceiling. Each toggle resets accumulation and rebuilds the AS.

## Key references

- Upstream: `D:/RTXPT-fork/Rtxpt/SampleUI.{h,cpp}`, `Sample.cpp:421,557-561`, `SampleCommon/LocalConfig.cpp:93-98`
- Port scene graph / instances: `src/RTXPTScene.cpp:667-674,1002-1043`, `src/RTXPTSceneGraph.hpp`
- Port AS node→instance transform: `src/RTXPTAccelerationStructures.cpp:251-253,340,406-426`
- Port UI / scene load: `src/RTXPTSample.cpp:711-797` (`RebuildSceneDependentResources`/`SetCurrentScene`),
  `:2030` (`UpdateUI`), `:1910` (`Update`)
- Related: [window_environment_map_issue.md](window_environment_map_issue.md),
  [transmissive_materials_issue.md](transmissive_materials_issue.md)
