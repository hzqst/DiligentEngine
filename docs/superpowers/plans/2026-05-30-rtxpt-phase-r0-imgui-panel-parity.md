# RTXPT Phase R0 — ImGui Panel Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the RTXPT sample's single flat debug panel into RTXPT-fork-style `CollapsingHeader` sections, relabel the reference-mode controls to match RTXPT-fork, add present-but-disabled placeholder controls for not-yet-ported features (with phase tooltips), relocate all status/debug readouts to their own section, and add a `RESET_ON_CHANGE`-equivalent reset-on-change helper — **without changing any rendering behavior.**

**Architecture:** This is a **UI-presentation-only** refactor of `RTXPTSample::UpdateUI()`. It touches exactly two C++ files (`RTXPTSample.hpp`, `RTXPTSample.cpp`). It introduces one C++ UI-state struct (`RTXPTReferenceUIState`) that mirrors the reference-mode subset of RTXPT-fork's `SampleUIData` (`D:/RTXPT-fork/Rtxpt/SampleUI.h`). The struct backs the **disabled placeholder** controls only; its fields are **not** wired into `RTXPTPathTracerSettings` / the GPU frame constants, so the GPU contract and the `static_assert(sizeof(RTXPTPathTracerSettings) == 48)` are untouched. The eight already-functional controls (max/min/NEE bounces, NEE toggle, env-NEE toggle, light/env intensity, reset) keep their existing wiring into `m_LastFrameConstants.PathTracer` (see `UpdateFrameConstants`, `RTXPTSample.cpp:274-283`) and continue to restart accumulation on change. Because the render output is unchanged, the primary verification is **compile + visual inspection**, not pixel diffs.

**Tech Stack:** C++17, Diligent Engine sample framework (`SampleBase`), Dear ImGui 1.92.1 (vendored in `DiligentTools/ThirdParty/imgui`, exposes `BeginDisabled`/`EndDisabled`, `ImGuiHoveredFlags_AllowWhenDisabled`, `SetTooltip`), clang-format 10.0.0 (CI-enforced).

---

## Why no automated tests

ImGui immediate-mode panel code in this sample has **no unit-test harness** — the panel is only observable by running the sample and looking at it. Strict TDD ("write a failing test, watch it fail") does not apply here. The substitute verification, applied per task, is:

1. **Compile** — the project builds (user-initiated per workspace rule; commands provided).
2. **Layout guard** — no GPU struct grows, so the existing `static_assert`s must still compile unchanged (they are not edited).
3. **Visual checklist** — a concrete, itemized manual inspection (Task 3) confirming sections, labels, disabled/greyed state, tooltips, and that functional controls still reset accumulation.

Per the workspace rule and the user's global `CLAUDE.md`, **do not auto-run build/run/format commands.** All such commands below are marked *(user-initiated)* — run them only when the user explicitly asks.

---

## File Structure

| File | Responsibility | Change |
|------|----------------|--------|
| `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` | Sample class declaration + GPU-mirrored structs | **Modify:** add `RTXPTReferenceUIState` struct + `RTXPTReferenceUIState m_ReferenceUI;` member |
| `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` | Sample implementation incl. `UpdateUI()` | **Modify:** replace the body of `UpdateUI()` (`RTXPTSample.cpp:462-598`) |

No new files. No `CMakeLists.txt` change (no new shaders/sources). No copyright-year change is required: both files already carry the 2026 Diligent/Apache header.

---

## Control mapping (the contract this plan implements)

Each row is a control in the new panel. **Functional** = already wired to the GPU; keep working, only regroup/relabel. **Placeholder** = present but `BeginDisabled(true)`, backed by an `m_ReferenceUI` field, with a tooltip naming the phase that will enable it.

| New panel section | Control label | Kind | Backed by | Notes / tooltip phase |
|-------------------|---------------|------|-----------|------------------------|
| Path Tracer | `Mode` combo `Reference\0Realtime` | Placeholder | local (fixed 0) | Realtime out of scope (umbrella Phase 5.5+) |
| Path Tracer → Setup | `Reset##REFMACC` button | Functional | `RequestAccumulationReset` | RTXPT `Reset##REFMACC` |
| Path Tracer → Setup | `Accumulated samples:` text | Functional (readout) | `m_AccumulationFrame` | — |
| Path Tracer → Setup | `Jitter anti-aliasing` | Placeholder (checked) | `m_ReferenceUI.AccumulationAA` | Always on in our port |
| Path Tracer → Setup | `Max bounces` slider | Functional | `m_MaxBounces` | RTXPT `Max bounces` |
| Path Tracer → Setup | `Max diffuse bounces` slider | Placeholder | `m_ReferenceUI.DiffuseBounceCount` | Phase R5 (G9) |
| Path Tracer → Setup | `Min bounces (RR start)` slider | Functional | `m_MinBounces` | divergence; our RR start |
| Path Tracer → Setup | `Use Russian Roulette early out` | Placeholder (checked) | `m_ReferenceUI.EnableRussianRoulette` | Always on; start = Min bounces |
| Path Tracer → Setup | `FireflyFilter (reference *)` + `FF Threshold` | Placeholder | `m_ReferenceUI.ReferenceFireflyFilterEnabled` / `...Threshold` | Phase R1 (G1) |
| Path Tracer → Post processing | `Enable tone mapping` | Placeholder (checked) | `m_ReferenceUI.EnableToneMapping` | ACES always applied; pass = Phase 6 |
| Path Tracer → Light sampling | `Use Next Event Estimation` | Functional | `m_EnableNEE` | RTXPT `Use Next Event Estimation` |
| Path Tracer → Light sampling | `Environment NEE + MIS` | Functional | `m_EnableEnvNEE` | divergence; keep functional |
| Path Tracer → Light sampling | `NEE bounces` slider | Functional | `m_MaxNEEBounces` | optional clamp; stays functional |
| Path Tracer → Light sampling | `Light intensity scale` slider | Functional | `m_LightIntensityScale` | divergence; keep functional |
| Path Tracer → Light sampling → NEE settings | `Sampling technique` combo | Placeholder | `m_ReferenceUI.NEEType` | Phase R3 (G5) |
| Path Tracer → Light sampling → NEE settings | `Candidate samples` | Placeholder | `m_ReferenceUI.NEECandidateSamples` | Phase R3 (G5) |
| Path Tracer → Light sampling → NEE settings | `Full samples` | Placeholder | `m_ReferenceUI.NEEFullSamples` | Phase R3 (G5) |
| Path Tracer → Light sampling → NEE settings | `MIS Type` combo | Placeholder | `m_ReferenceUI.NEEMISType` | Phase R3 (G5) |
| PT: Advanced Settings | `Nested Dielectrics` combo | Placeholder | `m_ReferenceUI.NestedDielectricsQuality` | Phase R6 (G10) |
| PT: Advanced Settings | `Enable LD sampler for BSDF` | Placeholder (checked) | `m_ReferenceUI.EnableLDSamplerForBSDF` | Phase R5 (G9) |
| Environment Map | `Enabled` | Placeholder | `m_ReferenceUI.EnvironmentMapEnabled` | Phase R4 (G7) HDR env map |
| Environment Map | `Intensity` slider | Functional | `m_EnvIntensity` | maps RTXPT Environment Map → Intensity |
| Scene | scene-camera combo + scene counts/errors | Functional + readout | `m_Scene` / `m_Materials` / `m_Lights` | moved out of the flat dump |
| Status / Debug | all backend/caps/AS/bridge/trace/compute/blit readouts + debug-compute toggle + legacy TODO lines | Functional + readout | various | relocated here |

---

## Task 1: Add the reference-mode UI-state struct

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` (insert struct after `RTXPTPathTracerSettings` block ending at line 75; add member in the `private:` member list near line 143)

- [ ] **Step 1: Add the `RTXPTReferenceUIState` struct**

Insert the following **immediately after** the `static_assert(sizeof(RTXPTPathTracerSettings) == 48, ...)` line (`RTXPTSample.hpp:75`) and **before** `struct RTXPTFrameConstants` (line 77):

```cpp
// Reference-mode UI state, mirroring the reference subset of RTXPT-fork's SampleUIData
// (D:/RTXPT-fork/Rtxpt/SampleUI.h). These fields back the present-but-disabled placeholder
// controls in UpdateUI(): each one is implemented in a later phase (R1/R3/R4/R5/R6 or the
// separate tone-mapping Phase 6) and is intentionally NOT yet wired into
// RTXPTPathTracerSettings / the GPU frame constants. Wiring a field in is part of the phase
// that enables its control, at which point the matching BeginDisabled() guard is removed.
struct RTXPTReferenceUIState
{
    bool  AccumulationAA                  = true;  // Jitter AA: always on in our port (no toggle yet).
    bool  EnableRussianRoulette           = true;  // RR: always on; start bounce == Min bounces (RR start).
    bool  ReferenceFireflyFilterEnabled   = true;  // Phase R1 (G1): adaptive firefly filter.
    float ReferenceFireflyFilterThreshold = 5.0f;  // Phase R1 (G1).
    int   DiffuseBounceCount              = 2;     // Phase R5 (G9): separate diffuse-bounce limit.
    bool  EnableToneMapping               = true;  // Phase 6: configurable tone-map pass (ACES is always applied now).
    int   NEEType                         = 1;     // Phase R3 (G5): 0=Uniform, 1=Power+, 2=NEE-AT.
    int   NEECandidateSamples             = 5;     // Phase R3 (G5): RIS candidate count.
    int   NEEFullSamples                  = 1;     // Phase R3 (G5): visibility-tested full samples.
    int   NEEMISType                      = 0;     // Phase R3 (G5): 0=Full, 1=ApproxInRealtime, 2=Approximate.
    int   NestedDielectricsQuality        = 1;     // Phase R6 (G10): 0=Off, 1=Fast, 2=Quality.
    bool  EnableLDSamplerForBSDF          = true;  // Phase R5 (G9): low-discrepancy (Sobol/Owen) sampler.
    bool  EnvironmentMapEnabled           = false; // Phase R4 (G7): HDR env-map loading (procedural sky is always active).
};
```

- [ ] **Step 2: Add the member to `RTXPTSample`**

In the `private:` section of `class RTXPTSample`, add the member immediately after the `RTXPTFrameConstants m_LastFrameConstants;` line (`RTXPTSample.hpp:123`):

```cpp
    RTXPTFrameConstants         m_LastFrameConstants;
    RTXPTReferenceUIState       m_ReferenceUI;
```

- [ ] **Step 3: Compile-check the header change** *(user-initiated)*

The struct is currently unused (its member is read/written in Task 2). An unused struct/member produces no warning. Build:

Run *(user-initiated)*: `cmake --build build/x64/Debug --config Debug`
Expected: builds successfully; no new warnings about `RTXPTReferenceUIState`.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp
git commit -m "feat(samples): RTXPT R0 add reference-mode UI-state struct

Mirrors the reference subset of RTXPT-fork SampleUIData to back the
present-but-disabled placeholder controls added in the R0 panel refactor.
Fields are not yet wired into RTXPTPathTracerSettings.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Rewrite `UpdateUI()` into RTXPT-fork-style sections

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp:462-598` (replace the entire body of `RTXPTSample::UpdateUI()`; leave `RequestAccumulationReset` at lines 600-604 unchanged)

- [ ] **Step 1: Replace the whole `UpdateUI()` function body**

Replace everything from `void RTXPTSample::UpdateUI()` through its closing `}` (currently `RTXPTSample.cpp:462-598`) with the following. It preserves every functional control's wiring (same `m_*` members, same reset reasons) and every readout (relocated into `Scene` / `Status / Debug`), and adds the disabled placeholders per the mapping table.

```cpp
void RTXPTSample::UpdateUI()
{
    // RESET_ON_CHANGE equivalent (cf. D:/RTXPT-fork/Rtxpt/SampleUI.cpp:49): when a control
    // reports a change, restart progressive accumulation. Returns the change flag so callers
    // can also write the edited value back.
    auto ResetOnChange = [this](bool Changed, const char* Reason) -> bool {
        if (Changed)
            RequestAccumulationReset(Reason);
        return Changed;
    };
    // Tooltip for a present-but-disabled placeholder control. AllowWhenDisabled lets the
    // tooltip appear even though the preceding item was inside BeginDisabled()/EndDisabled().
    auto PlaceholderTooltip = [](const char* Text) {
        if (ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenDisabled))
            ImGui::SetTooltip("%s", Text);
    };

    const ImVec4 CategoryColor{0.60f, 0.85f, 1.00f, 1.00f};
    const float  Indent = 16.0f;

    ImGui::Begin("RTXPT");

    // ------------------------------------------------------------------ Path Tracer
    if (ImGui::CollapsingHeader("Path Tracer", ImGuiTreeNodeFlags_DefaultOpen))
    {
        ImGui::Indent(Indent);

        // Mode (Reference only; Realtime track is out of scope).
        {
            int ModeIndex = 0;
            ImGui::BeginDisabled(true);
            ImGui::Combo("Mode", &ModeIndex, "Reference\0Realtime\0\0");
            ImGui::EndDisabled();
            PlaceholderTooltip("Realtime mode is out of scope for the reference path tracer (umbrella Phase 5.5+).");
        }

        ImGui::TextColored(CategoryColor, "Setup:");
        ImGui::Indent(Indent);
        {
            if (ImGui::Button("Reset##REFMACC"))
                RequestAccumulationReset("User reset");
            ImGui::SameLine();
            ImGui::Text("Accumulated samples: %u", m_AccumulationFrame);

            // Jitter AA: always on in our port; shown checked + disabled.
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("Jitter anti-aliasing", &m_ReferenceUI.AccumulationAA);
            ImGui::EndDisabled();
            PlaceholderTooltip("Per-sample pixel jitter is always enabled in the reference path tracer.");

            int MaxBouncesUI = static_cast<int>(m_MaxBounces);
            if (ResetOnChange(ImGui::SliderInt("Max bounces", &MaxBouncesUI, 1, 16), "Max bounces changed"))
                m_MaxBounces = static_cast<Uint32>(MaxBouncesUI);

            // Max diffuse bounces: placeholder until the BSDF/sampler work (Phase R5).
            ImGui::BeginDisabled(true);
            ImGui::SliderInt("Max diffuse bounces", &m_ReferenceUI.DiffuseBounceCount, 0, 16);
            ImGui::EndDisabled();
            PlaceholderTooltip("Separate diffuse-bounce limit lands with the BSDF/sampler work (Phase R5).");

            int MinBouncesUI = static_cast<int>(m_MinBounces);
            if (ResetOnChange(ImGui::SliderInt("Min bounces (RR start)", &MinBouncesUI, 0, 16), "Min bounces changed"))
                m_MinBounces = static_cast<Uint32>(MinBouncesUI);

            // Russian roulette: always on; start bounce is "Min bounces (RR start)".
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("Use Russian Roulette early out", &m_ReferenceUI.EnableRussianRoulette);
            ImGui::EndDisabled();
            PlaceholderTooltip("Russian roulette is always enabled; its start bounce is 'Min bounces (RR start)'.");

            // Adaptive firefly filter: Phase R1 (G1). Each control gets its own
            // BeginDisabled/EndDisabled scope so IsItemHovered() in PlaceholderTooltip()
            // attaches a tooltip to that specific item rather than only the last one.
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("FireflyFilter (reference *)", &m_ReferenceUI.ReferenceFireflyFilterEnabled);
            ImGui::EndDisabled();
            PlaceholderTooltip("Adaptive firefly filter lands in Phase R1.");
            if (m_ReferenceUI.ReferenceFireflyFilterEnabled)
            {
                ImGui::Indent(Indent);
                ImGui::BeginDisabled(true);
                ImGui::InputFloat("FF Threshold", &m_ReferenceUI.ReferenceFireflyFilterThreshold, 0.1f, 0.2f, "%.5f");
                ImGui::EndDisabled();
                PlaceholderTooltip("Adaptive firefly filter threshold lands in Phase R1.");
                ImGui::Unindent(Indent);
            }
        }
        ImGui::Unindent(Indent); // end Setup

        ImGui::TextColored(CategoryColor, "Post processing:");
        ImGui::Indent(Indent);
        {
            // Tone mapping: ACES is always applied in raygen today; a configurable
            // tone-map pass is tracked separately as Phase 6.
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("Enable tone mapping", &m_ReferenceUI.EnableToneMapping);
            ImGui::EndDisabled();
            PlaceholderTooltip("ACES tone mapping is always applied in raygen; a configurable tone-map pass is tracked as Phase 6.");
        }
        ImGui::Unindent(Indent); // end Post processing

        ImGui::TextColored(CategoryColor, "Light sampling:");
        ImGui::Indent(Indent);
        {
            ResetOnChange(ImGui::Checkbox("Use Next Event Estimation", &m_EnableNEE), "NEE toggled");
            ResetOnChange(ImGui::Checkbox("Environment NEE + MIS", &m_EnableEnvNEE), "Environment NEE toggled");

            int MaxNEEBouncesUI = static_cast<int>(m_MaxNEEBounces);
            if (ResetOnChange(ImGui::SliderInt("NEE bounces", &MaxNEEBouncesUI, 0, 16), "NEE bounce budget changed"))
                m_MaxNEEBounces = static_cast<Uint32>(MaxNEEBouncesUI);

            ResetOnChange(ImGui::SliderFloat("Light intensity scale", &m_LightIntensityScale, 0.0f, 10.0f), "Light intensity changed");

            if (m_EnableNEE)
            {
                ImGui::TextColored(CategoryColor, "NEE settings:");
                ImGui::Indent(Indent);
                {
                    // Light importance sampling (RIS/WRS) + MIS type: Phase R3 (G5).
                    ImGui::BeginDisabled(true);
                    ImGui::Combo("Sampling technique", &m_ReferenceUI.NEEType, "Uniform\0Power+\0NEE-AT\0\0");
                    ImGui::EndDisabled();
                    PlaceholderTooltip("Light importance sampling (RIS/WRS) lands in Phase R3.");

                    ImGui::BeginDisabled(true);
                    ImGui::InputInt("Candidate samples", &m_ReferenceUI.NEECandidateSamples, 1);
                    ImGui::EndDisabled();
                    PlaceholderTooltip("RIS candidate count lands in Phase R3.");

                    ImGui::BeginDisabled(true);
                    ImGui::InputInt("Full samples", &m_ReferenceUI.NEEFullSamples, 1);
                    ImGui::EndDisabled();
                    PlaceholderTooltip("Visibility-tested full-sample count lands in Phase R3.");

                    ImGui::BeginDisabled(true);
                    ImGui::Combo("MIS Type", &m_ReferenceUI.NEEMISType, "Full\0ApproxInRealtime\0Approximate\0\0");
                    ImGui::EndDisabled();
                    PlaceholderTooltip("Selectable MIS type lands in Phase R3.");
                }
                ImGui::Unindent(Indent);
            }
        }
        ImGui::Unindent(Indent); // end Light sampling

        ImGui::Unindent(Indent); // end Path Tracer
    }

    // ------------------------------------------------------------ PT: Advanced Settings
    if (ImGui::CollapsingHeader("PT: Advanced Settings"))
    {
        ImGui::Indent(Indent);

        // Nested dielectrics: Phase R6 (G10).
        ImGui::BeginDisabled(true);
        ImGui::Combo("Nested Dielectrics", &m_ReferenceUI.NestedDielectricsQuality, "Off\0Fast\0Quality\0\0");
        ImGui::EndDisabled();
        PlaceholderTooltip("Nested dielectrics land in Phase R6.");

        // Low-discrepancy sampler for BSDF: Phase R5 (G9).
        ImGui::BeginDisabled(true);
        ImGui::Checkbox("Enable LD sampler for BSDF", &m_ReferenceUI.EnableLDSamplerForBSDF);
        ImGui::EndDisabled();
        PlaceholderTooltip("Low-discrepancy (Sobol/Owen) sampler lands in Phase R5.");

        ImGui::Unindent(Indent);
    }

    // ------------------------------------------------------------------ Environment Map
    if (ImGui::CollapsingHeader("Environment Map"))
    {
        ImGui::Indent(Indent);

        // HDR env-map loading: Phase R4 (G7). A procedural sky is always active.
        ImGui::BeginDisabled(true);
        ImGui::Checkbox("Enabled", &m_ReferenceUI.EnvironmentMapEnabled);
        ImGui::EndDisabled();
        PlaceholderTooltip("HDR environment-map loading lands in Phase R4; a procedural sky is always active.");

        ResetOnChange(ImGui::SliderFloat("Intensity", &m_EnvIntensity, 0.0f, 5.0f), "Environment intensity changed");

        ImGui::Unindent(Indent);
    }

    // ------------------------------------------------------------------------- Scene
    if (ImGui::CollapsingHeader("Scene"))
    {
        ImGui::Indent(Indent);

        ImGui::Text("Scene: %s", m_Scene.HasValidContent() ? "loaded" : "missing");
        ImGui::Text("Scene file: %s", m_Scene.GetLoadedSceneName().empty() ? "none" : m_Scene.GetLoadedSceneName().c_str());
        ImGui::Text("Model path: %s", m_Scene.GetModelPath().empty() ? "none" : m_Scene.GetModelPath().c_str());
        ImGui::Text("Scene cameras: %u", m_Scene.GetCameraCount());
        if (m_Scene.GetCameraCount() > 0)
        {
            const char* PreviewName = "none";
            if (m_SelectedSceneCamera >= 0)
            {
                if (const RTXPTSceneCamera* pSelectedCamera = m_Scene.GetCamera(static_cast<Uint32>(m_SelectedSceneCamera)))
                    PreviewName = pSelectedCamera->Name.c_str();
            }

            if (ImGui::BeginCombo("Scene camera", PreviewName))
            {
                for (Uint32 CameraIdx = 0; CameraIdx < m_Scene.GetCameraCount(); ++CameraIdx)
                {
                    const RTXPTSceneCamera* pCamera = m_Scene.GetCamera(CameraIdx);
                    if (pCamera == nullptr)
                        continue;

                    const bool IsSelected = static_cast<int>(CameraIdx) == m_SelectedSceneCamera;
                    ImGui::PushID(static_cast<int>(CameraIdx));
                    if (ImGui::Selectable(pCamera->Name.c_str(), IsSelected))
                        ApplySceneCamera(CameraIdx);
                    if (IsSelected)
                        ImGui::SetItemDefaultFocus();
                    ImGui::PopID();
                }
                ImGui::EndCombo();
            }
        }
        if (!m_Scene.GetLastError().empty())
            ImGui::TextWrapped("Asset load error: %s", m_Scene.GetLastError().c_str());
        ImGui::Text("Mesh nodes: %u", m_Scene.GetMeshNodeCount());
        ImGui::Text("Primitives: %u", m_Scene.GetPrimitiveCount());
        ImGui::Text("Materials: %u", m_Materials.GetStats().MaterialCount);
        ImGui::Text("Lights: %u", m_Lights.GetStats().LightCount);
        if (!m_Materials.GetStats().LastError.empty())
            ImGui::TextWrapped("Material buffer error: %s", m_Materials.GetStats().LastError.c_str());
        if (!m_Lights.GetStats().LastError.empty())
            ImGui::TextWrapped("Light buffer error: %s", m_Lights.GetStats().LastError.c_str());

        ImGui::Unindent(Indent);
    }

    // ------------------------------------------------------------------ Status / Debug
    if (ImGui::CollapsingHeader("Status / Debug"))
    {
        ImGui::Indent(Indent);

        const RTXPTAccelerationStructureStats& ASStats      = m_AccelerationStructures.GetStats();
        const RTXPTRayTracingPassStats&        RTPassStats  = m_RayTracingPass.GetStats();
        const RTXPTComputePassStats&           ComputeStats = m_DebugComputePass.GetStats();

        ImGui::Text("Backend: %s", GetRenderDeviceTypeString(m_pDevice->GetDeviceInfo().Type));
        ImGui::Text("RayTracing: %s", m_FeatureCaps.RayTracing ? "yes" : "no");
        ImGui::Text("Standalone RT shaders: %s", m_FeatureCaps.StandaloneRayTracingShaders ? "yes" : "no");
        ImGui::Text("RayQuery: %s", m_FeatureCaps.RayQuery ? "yes" : "no");
        ImGui::Text("Bindless: %s", m_FeatureCaps.BindlessResources ? "yes" : "no");
        ImGui::Text("Compute: %s", m_FeatureCaps.ComputeShaders ? "yes" : "no");
        ImGui::Text("Assets root: %s", m_AssetsRoot.c_str());
        ImGui::Separator();
        ImGui::Text("Acceleration structures: %s", m_AccelerationStructures.IsBuilt() ? "built" : "not built");
        ImGui::Text("BLAS: %u", ASStats.BLASCount);
        ImGui::Text("TLAS instances: %u", ASStats.InstanceCount);
        ImGui::Text("RT geometries: %u", ASStats.GeometryCount);
        ImGui::Text("Sub-instances: %u", ASStats.SubInstanceCount);
        ImGui::Text("Alpha-tested geometries: %u", ASStats.AlphaTestedGeometryCount);
        if (!ASStats.DisabledReason.empty())
            ImGui::TextWrapped("AS disabled: %s", ASStats.DisabledReason.c_str());
        if (!ASStats.LastError.empty())
            ImGui::TextWrapped("AS error: %s", ASStats.LastError.c_str());
        ImGui::Separator();
        ImGui::Text("Frame constants: %s", m_FrameConstantsCB ? "created" : "missing");
        ImGui::Text("Frame index: %u", m_FrameIndex);
        ImGui::Text("Viewport: %.0f x %.0f", m_LastFrameConstants.ViewportSize_FrameIdx.x, m_LastFrameConstants.ViewportSize_FrameIdx.y);
        ImGui::Separator();
        ImGui::Text("OutputColor: %s", m_RenderTargets.IsValid() ? "created" : "missing");
        ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
        ImGui::Text("Material bridge: %s", RTPassStats.MaterialBridgeBound ? "bound" : "fallback");
        ImGui::Text("Sub-instance bridge: %s", RTPassStats.SubInstanceBound ? "bound" : "fallback");
        ImGui::Text("Light bridge: %s", RTPassStats.LightBridgeBound ? "bound" : "fallback");
        ImGui::Text("Vertex buffer: %s", RTPassStats.VertexBufferBound ? "bound" : "fallback");
        ImGui::Text("Index buffer: %s", RTPassStats.IndexBufferBound ? "bound" : "fallback");
        ImGui::Text("Material textures loaded: %u", m_Materials.GetStats().TextureCount);
        ImGui::Text("Material textures bound: %s (%u)", RTPassStats.MaterialTexturesBound ? "yes" : "no", RTPassStats.MaterialTextureCount);
        ImGui::Text("Alpha-test any-hit: %s", RTPassStats.AnyHitEnabled ? "enabled" : "disabled");
        ImGui::Text("Accumulation target: %s", m_AccumulationActive ? "active (RGBA32F)" : "inactive (RGBA8 fallback)");
        ImGui::Text("Accumulation frame: %u", m_AccumulationFrame);
        ImGui::Text("TraceRays executed: %s", RTPassStats.LastTraceExecuted ? "yes" : "no");
        ImGui::Text("TraceRays count: %u", RTPassStats.TraceCount);
        if (!RTPassStats.DisabledReason.empty())
            ImGui::TextWrapped("TraceRays disabled: %s", RTPassStats.DisabledReason.c_str());
        if (!RTPassStats.LastError.empty())
            ImGui::TextWrapped("TraceRays error: %s", RTPassStats.LastError.c_str());
        ImGui::Separator();
        ImGui::Checkbox("Debug compute pass", &m_EnableDebugComputePass);
        ImGui::Text("Compute dispatch: %s", m_DebugComputePass.IsReady() ? "ready" : "not ready");
        ImGui::Text("Compute executed: %s", ComputeStats.LastDispatchExecuted ? "yes" : "no");
        ImGui::Text("Compute dispatch count: %u", ComputeStats.DispatchCount);
        if (!ComputeStats.DisabledReason.empty())
            ImGui::TextWrapped("Compute disabled: %s", ComputeStats.DisabledReason.c_str());
        if (!ComputeStats.LastError.empty())
            ImGui::TextWrapped("Compute error: %s", ComputeStats.LastError.c_str());
        if (!m_RenderTargets.GetLastError().empty())
            ImGui::TextWrapped("Render target error: %s", m_RenderTargets.GetLastError().c_str());
        if (!m_BlitPass.GetLastError().empty())
            ImGui::TextWrapped("Blit error: %s", m_BlitPass.GetLastError().c_str());
        ImGui::Text("Blit draw count: %u", m_BlitPass.GetDrawCount());
        ImGui::Separator();
        ImGui::TextColored(CategoryColor, "Roadmap (open work):");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R1): adaptive firefly filter, NEE at all bounces, decorrelated seeding.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R3): light importance sampling (RIS/WRS) + photometric units.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R4): HDR environment map with importance sampling + MIS.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R5): VNDF/Frostbite/multi-scatter BSDF + low-discrepancy sampler.");
        ImGui::TextWrapped("TODO(RTXPT-Port Phase R6): transmission / nested dielectrics / ALPHA_MODE_BLEND.");

        ImGui::Unindent(Indent);
    }

    ImGui::End();
}
```

- [ ] **Step 2: Compile-check the rewrite** *(user-initiated)*

Run *(user-initiated)*: `cmake --build build/x64/Debug --config Debug`
Expected: builds successfully. Watch for: unbalanced `Indent`/`Unindent`, a stray reference to the now-removed local `int MaxBouncesUI`/`MinBouncesUI`/`MaxNEEBouncesUI` outside their scopes, or a missing `EndDisabled()`. There should be none.

- [ ] **Step 3: Run clang-format on the changed file** *(user-initiated)*

clang-format 10.0.0 is CI-enforced; the file must validate. Run *(user-initiated)* the project target:

Run: `cmake --build build/x64/Debug --target DiligentCore-ValidateFormatting`
Expected: PASS (no formatting diffs reported for `RTXPTSample.cpp`).

If it reports diffs, apply them: `clang-format -i DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` (using clang-format 10.0.0), then re-run the validation target.

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(samples): RTXPT R0 ImGui panel parity with RTXPT-fork

Restructure UpdateUI() into RTXPT-fork-style CollapsingHeader sections
(Path Tracer / PT: Advanced Settings / Environment Map / Scene /
Status & Debug). Relabel reference-mode controls to match RTXPT-fork,
add present-but-disabled placeholder controls for not-yet-ported
features with phase tooltips, relocate all status/debug readouts to
their own section, and add a RESET_ON_CHANGE-equivalent helper.

UI-only: no rendering behavior or GPU struct layout changes.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Verify panel parity and unchanged rendering

**Files:** none (manual verification).

This task has no code changes. It confirms the R0 success criteria from the spec (G0). Build/run is *(user-initiated)*.

- [ ] **Step 1: Launch the sample** *(user-initiated)*

Run *(user-initiated)*, e.g.: `build/x64/Debug/DiligentSamples/Samples/RTXPT/Debug/RTXPT.exe`
(Use the actual built sample path/launcher for your configuration; run on **both** D3D12 and Vulkan if your toggle/flags allow, per the spec's both-backends requirement.)

- [ ] **Step 2: Walk the visual checklist**

Confirm each item:

- [ ] The window now shows five collapsible sections: **Path Tracer** (open by default), **PT: Advanced Settings**, **Environment Map**, **Scene**, **Status / Debug** (all collapsed by default except Path Tracer).
- [ ] Under **Path Tracer**, the `Setup:` / `Post processing:` / `Light sampling:` category labels appear (tinted), with controls indented beneath them.
- [ ] **Disabled/greyed** controls render greyed and are not editable: `Mode`, `Jitter anti-aliasing`, `Max diffuse bounces`, `Use Russian Roulette early out`, `FireflyFilter (reference *)` (+ `FF Threshold`), `Enable tone mapping`, `Sampling technique`, `Candidate samples`, `Full samples`, `MIS Type`, `Nested Dielectrics`, `Enable LD sampler for BSDF`, `Environment Map → Enabled`.
- [ ] Hovering each disabled control shows a tooltip naming the phase (e.g. "…lands in Phase R3.", "…Phase R6.", "always on…", "…Phase 6.").
- [ ] **Functional** controls are editable and changing any of them visibly **restarts accumulation** (the image resets to noisy then re-converges): `Max bounces`, `Min bounces (RR start)`, `NEE bounces`, `Use Next Event Estimation`, `Environment NEE + MIS`, `Light intensity scale`, `Environment Map → Intensity`, and the `Reset##REFMACC` button.
- [ ] The `Scene` section's scene-camera combo still switches cameras.
- [ ] All former readouts (backend, caps, AS counts, bridges, trace/compute/blit stats) are present under **Status / Debug**; the `Debug compute pass` checkbox still works there; the roadmap TODO lines are at the bottom of that section.

- [ ] **Step 3: Confirm rendering is unchanged**

With all controls left at their defaults, the converged image must look **identical** to the pre-R0 build (R0 changes no shader, no frame constant, no GPU struct). If the image differs, a functional control was accidentally rewired or dropped — diff against the control mapping table above and fix before considering R0 done.

---

## Self-Review

**1. Spec coverage (G0 success criteria):**
- "UI mirrors RTXPT-fork's `CollapsingHeader` grouping and control labels for reference-mode-relevant controls" → Task 2 (Path Tracer / PT: Advanced Settings / Environment Map sections with RTXPT labels). ✓
- "status/debug readouts move to their own section" → Task 2, `Status / Debug` + `Scene` sections. ✓
- "a reset-on-change helper matches `RESET_ON_CHANGE`" → Task 2, `ResetOnChange` lambda. ✓
- "Controls for features not yet implemented are present but disabled/greyed with a tooltip naming the phase" → Task 2 placeholders via `BeginDisabled(true)` + `PlaceholderTooltip`; phases per the mapping table. ✓
- Cross-cutting "Settings & frame-constants layout" contract (`static_assert(sizeof == 48)`) → untouched; placeholders are backed by `RTXPTReferenceUIState`, not `RTXPTPathTracerSettings`. ✓
- Verification strategy "Runnable on both backends" + "unbiasedness/unchanged image" → Task 3 Steps 1 & 3. ✓

**2. Placeholder scan:** No `TBD`/`implement later`/"add appropriate…" in plan steps. The in-panel `TODO(RTXPT-Port Phase …)` strings are intentional roadmap text required by the Open-Work/TODO Marker Policy, with complete wording given. ✓

**3. Type consistency:** The struct type `RTXPTReferenceUIState` and member `m_ReferenceUI` (Task 1) are used with exactly those names in Task 2. Field names referenced in Task 2 (`AccumulationAA`, `EnableRussianRoulette`, `ReferenceFireflyFilterEnabled`, `ReferenceFireflyFilterThreshold`, `DiffuseBounceCount`, `EnableToneMapping`, `NEEType`, `NEECandidateSamples`, `NEEFullSamples`, `NEEMISType`, `NestedDielectricsQuality`, `EnableLDSamplerForBSDF`, `EnvironmentMapEnabled`) all match the struct definition in Task 1, with ImGui-compatible types (`int` for combos/`InputInt`, `bool` for checkboxes, `float` for `InputFloat`/`SliderFloat`). Functional members (`m_MaxBounces`, `m_MinBounces`, `m_MaxNEEBounces`, `m_EnableNEE`, `m_EnableEnvNEE`, `m_LightIntensityScale`, `m_EnvIntensity`, `m_AccumulationFrame`, `m_SelectedSceneCamera`, `m_EnableDebugComputePass`) and the readout accessors match the existing `RTXPTSample.hpp` declarations and the pre-R0 `UpdateUI`. The `ResetOnChange` reason strings reuse the exact strings the pre-R0 code passed to `RequestAccumulationReset`. ✓
