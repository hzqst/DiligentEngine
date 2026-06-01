# RTXPT Post-Processing Phase P0 Mapping and Contract Prep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lock the RTXPT-fork post-processing source map, Diligent destination ownership, and raw-HDR output contract before Phase P1-P3 code work begins.

**Architecture:** This phase is documentation and contract prep only: no render behavior changes, no shader binding changes, and no pass implementation. It updates the RTXPT fork mapping, links the design spec to this follow-up plan, and replaces broad Phase 6 notes with structured markers that point future work at the right Diligent-native owner files.

**Tech Stack:** Markdown under `docs/superpowers`, DiligentSamples RTXPT C++17/HLSL source inventory, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`. `DiligentSamples` is a git submodule; mapping and marker edits inside it require a submodule commit during execution.

---

## Current Baseline

- The driving spec is `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P0.
- Current Diligent reference raygen still declares `u_Output` as `VK_IMAGE_FORMAT("rgba8")` and `u_AccumulationBuffer` as `VK_IMAGE_FORMAT("rgba32f")` in `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`.
- Current Diligent raygen blends `pathRadiance` into `u_AccumulationBuffer`, applies `ToneMapACES(accumulated * g_Const.ptConsts.exposureScale)`, and writes the result to `u_Output`.
- `RTXPTRenderTargets::Resize` is still called with `TEX_FORMAT_RGBA8_UNORM` from `RTXPTSample::EnsureRenderTargets`.
- `RTXPTSample::Render` still dispatches `RTXPTRayTracingPass::Trace`, optional `RTXPTComputePass`, then `RTXPTBlitPass` from `OutputColor` or `ComputeColor`.
- `RTXPTReferenceUIState` already carries Phase 6 tone-mapping fields, but `Enable tone mapping` is disabled and documented as future work.
- Top-level `git status --short` currently shows the Phase 6 spec as untracked. Treat it as user-authored context and do not overwrite it.

## RTXPT-Fork Anchors

Use these source anchors as authoritative references for P0 mapping:

- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1279-1330` - `CreateRenderPasses`, including accumulation, tone mapping, bloom/post process, TAA, and baker pass creation.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1827-1885` - `PostProcessPreToneMapping` and `PostProcessPostToneMapping`.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2186-2213` - frame-tail post-process order: AA/accumulation, HDR post-process, tone mapping, LDR post-process, overlays, final blit.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2621-2763` - `PostProcessAA`, including reference accumulation and advanced realtime output selection.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2330-2348` - post-process binding set resources for `LdrColorScratch`, `OutputColor`, `ProcessedOutputColor`, and `LdrColor`.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h:34-43` - render target ownership comments and output contract.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.cpp:154-247` - `AccumulatedRadiance`, HDR `OutputColor`, `ProcessedOutputColor`, `LdrColor`, and `LdrColorScratch` formats.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/AccumulationPass.{h,cpp,hlsl}` - reference accumulation pass, `blendFactor`, input resampling, accumulated HDR output.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMappingPasses.{h,cpp}` - `ToneMappingPass`, parameters, auto exposure, white balance, pass-through when disabled.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMapping_cb.h`, `ToneMapping.hlsl`, `ToneMapping.ps.hlsli`, `luminance_ps.hlsl` - tone-mapping constants and shader operators.
- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.{h,cpp,hlsl}` - HDR/LDR compute variants, stable-plane merge, NRD prepare/final-merge targets.
- `D:/RTXPT-fork/Rtxpt/Shaders/Bindings/ShaderResourceBindings.hlsli:21-26` - shared post-process resource names and slots.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli:165-168` - reference-mode `CommitPixel` writes raw `path.GetL()` to `OutputColor`.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:205,301-307,315` and `SampleUI.cpp:942,1534-1619` - tone mapping, bloom, HDR/LDR test pass UI controls.

## Diligent Destination Ownership

P0 must record this ownership so later phases do not re-decide it:

- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` owns the RTXPT-fork-to-Diligent source map and divergence notes.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}` owns all RTXPT render target creation and accessor names.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}` owns per-frame orchestration, reset reasons, UI state, and fallback clear colors.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` owns ray tracing bindings only. It must stop owning accumulation UAV binding in Phase P2.
- `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.{hpp,cpp}` remains the final swapchain copy wrapper unless a later phase explicitly replaces it with an existing Diligent full-screen helper.
- Future P1-P5 post-processing pass ownership is Diligent-native:
  - Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}` - orchestration owner for accumulation, HDR post-process, tone mapping, LDR post-process, display-source selection, status, and fallback readiness.
  - Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccumulationPass.{hpp,cpp}` - Diligent compute wrapper for reference accumulation.
  - Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.{hpp,cpp}` - Diligent graphics or compute wrapper for tone mapping.
  - Create: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.{hpp,cpp}` - shared wrapper for bloom/test HDR/LDR post-process effects once needed.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTAccumulation.csh`.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.hlsl`.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.ps.hlsli`.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMappingShared.h`.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/Luminance.psh`.
  - Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`.
- `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.{hpp,cpp}` stays diagnostic-only. Do not grow it into the long-term post-processing framework.
- `DiligentSamples/Samples/RTXPT/CMakeLists.txt` registers future C++ and HLSL files after the corresponding implementation phase creates them.

## Cross-Cutting Contracts

- **Raw HDR ownership:** `OutputColor` means raw path tracing or merged realtime radiance, not display-ready color.
- **Accumulation ownership:** `AccumulatedRadiance` stores reference accumulation. `ProcessedOutputColor` stores the HDR result consumed by tone mapping.
- **Tone mapping ownership:** tone mapping is a post-process pass. Raygen must not apply `ToneMapACES` or `exposureScale` after Phase P3.
- **Display ownership:** `LdrColor` is the normal final blit source. `OutputColor` is never the normal swapchain source after P3.
- **Reset semantics:** camera, scene, AS, materials, lights, env map, accumulation AA, tone mapping, and post-process controls reset only the histories they invalidate.
- **Capability gating:** unsupported formats, compute shaders, ray tracing, future denoisers, and optional NVIDIA integrations fail closed with visible reasons.
- **RTXPT-fork alignment:** copy concepts, resource names, and behavior contracts. Do not copy Donut/NVRHI APIs, NVIDIA file headers, or large verbatim source blocks into Diligent-owned files.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `D:/RTXPT-fork/Rtxpt`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Top-level may list `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md` as untracked. `DiligentSamples` should be clean before P0 edits; if dirty, read those files before editing.

- [ ] **Step 2: Confirm RTXPT-fork anchor files exist**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\Sample.cpp
Test-Path D:\RTXPT-fork\Rtxpt\SampleCommon\RenderTargets.h
Test-Path D:\RTXPT-fork\Rtxpt\ProcessingPasses\AccumulationPass.hlsl
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ToneMappingPasses.h
Test-Path D:\RTXPT-fork\Rtxpt\ProcessingPasses\PostProcess.hlsl
```

Expected: every command prints `True`.

- [ ] **Step 3: Confirm current Diligent post-process debt markers**

Run:

```powershell
rg -n "ToneMapACES|u_AccumulationBuffer|exposureScale|OutputColor is the rgba8|Enable tone mapping|Phase 6" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: matches in `PathTracerSample.rgen`, `RTXPTFrameConstants.hpp`, `PathTracerShared.h`, `RTXPTSample.hpp`, `RTXPTSample.cpp`, and `RTXPTRayTracingPass.cpp`.

- [ ] **Step 4: Do not commit**

Expected: Task 0 only establishes the baseline.

### Task 1: Audit Source and Destination Anchors

**Files:**
- Read: `D:/RTXPT-fork/Rtxpt/Sample.cpp`
- Read: `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.*`
- Read: `D:/RTXPT-fork/Rtxpt/ProcessingPasses/*`
- Read: `D:/RTXPT-fork/Rtxpt/ToneMapper/*`
- Read: `DiligentSamples/Samples/RTXPT/src/*`
- Read: `DiligentSamples/Samples/RTXPT/assets/shaders/*`

- [ ] **Step 1: Capture RTXPT-fork source anchors**

Run:

```powershell
rg -n "CreateRenderPasses|PostProcessAA|PostProcessPreToneMapping|ToneMappingPass|PostProcessPostToneMapping|BlitTexture|AccumulatedRadiance|ProcessedOutputColor|LdrColor|LdrColorScratch" D:/RTXPT-fork/Rtxpt/Sample.cpp D:/RTXPT-fork/Rtxpt/SampleCommon D:/RTXPT-fork/Rtxpt/ProcessingPasses D:/RTXPT-fork/Rtxpt/ToneMapper D:/RTXPT-fork/Rtxpt/Shaders/Bindings
```

Expected: matches include the RTXPT-fork anchors listed above. Keep this command output available while editing the mapping table.

- [ ] **Step 2: Capture current Diligent destination anchors**

Run:

```powershell
rg -n "RTXPTRenderTargets|RTXPTRayTracingPass|RTXPTBlitPass|RTXPTComputePass|EnsureRenderTargets|Render\\(|GetDisplaySRV|u_Output|u_AccumulationBuffer|ToneMapACES|exposureScale" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected: matches identify all current owners and the debt points that P1-P3 will change.

- [ ] **Step 3: Confirm future destination names are unused**

Run:

```powershell
rg -n "RTXPTPostProcessPipeline|RTXPTAccumulationPass|RTXPTToneMappingPass|RTXPTPostProcessPass|RTXPTAccumulation\\.csh|PostProcessing/ToneMapper" DiligentSamples/Samples/RTXPT
```

Expected: no matches. If matches exist, review them and adjust the mapping table to the actual local names.

### Task 2: Update RTXPT Fork Mapping

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Insert the Phase 6 mapping section**

Insert the following section before `## Skinned glTF Current Geometry` in `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`:

```markdown
## Phase 6 Post-Processing Pipeline Mapping

Phase 6 ports the RTXPT-fork post-processing display contract. This section is the ownership contract for P1-P8 implementation tasks; it does not imply that the listed Diligent files already exist.

### Phase 6 Source-to-Destination Map

| RTXPT-fork source | Diligent RTXPT destination | Phase | Contract |
|---|---|---:|---|
| `SampleCommon/RenderTargets.h` | `src/RTXPTRenderTargets.hpp` | P1 | Resource names, accessors, and ownership comments for `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, `LdrColorScratch`, and later temporal/denoiser targets. |
| `SampleCommon/RenderTargets.cpp` | `src/RTXPTRenderTargets.cpp` | P1 | Diligent texture creation and fallback logic. `OutputColor` is HDR radiance, `AccumulatedRadiance` is `RGBA32F`, `ProcessedOutputColor` is HDR post-AA/post-accumulation, and `LdrColor`/`LdrColorScratch` are display-format ping-pong targets. |
| `Sample.cpp::CreateRenderPasses` | `src/RTXPTPostProcessPipeline.{hpp,cpp}` plus `src/RTXPTSample.cpp` | P1 | Diligent-native pass construction and readiness. `RTXPTSample` owns lifetime and frame orchestration; the pipeline owns post-process pass objects. |
| `Sample.cpp::PostProcessAA` | `src/RTXPTAccumulationPass.{hpp,cpp}` and later TAA/DLSS-specific classes | P2, P6, P8 | Reference mode runs accumulation from `OutputColor` into `AccumulatedRadiance` and `ProcessedOutputColor`. TAA/DLSS/DLSS-RR are later gated modes. |
| `ProcessingPasses/AccumulationPass.h` | `src/RTXPTAccumulationPass.hpp` | P2 | Public Diligent accumulation pass interface, status, and resource binding contract. |
| `ProcessingPasses/AccumulationPass.cpp` | `src/RTXPTAccumulationPass.cpp` | P2 | Diligent compute PSO/SRB creation and dispatch. No Donut/NVRHI API copy. |
| `ProcessingPasses/AccumulationPass.hlsl` | `assets/shaders/PostProcessing/RTXPTAccumulation.csh` | P2 | Blend raw `OutputColor` into `AccumulatedRadiance`; write HDR `ProcessedOutputColor`. Preserve blend-factor semantics and render/display-size resampling hook. |
| `ToneMapper/ToneMappingPasses.h` | `src/RTXPTToneMappingPass.hpp` and `src/RTXPTSample.hpp` | P3 | Tone-mapping parameter model: operator, enable flag, exposure mode, exposure compensation/value/range, auto exposure, white balance, white luminance/scale, clamp. |
| `ToneMapper/ToneMappingPasses.cpp` | `src/RTXPTToneMappingPass.cpp` | P3 | Diligent tone-map pass setup, luminance resources, constants upload, pass-through when disabled, and frame advance. |
| `ToneMapper/ToneMapping_cb.h` | `assets/shaders/PostProcessing/ToneMapper/ToneMappingShared.h` and matching C++ struct in `src/RTXPTToneMappingPass.hpp` | P3 | CPU/GPU constants layout. Add `static_assert` checks when the C++ side is introduced. |
| `ToneMapper/ToneMapping.hlsl` | `assets/shaders/PostProcessing/ToneMapper/ToneMapping.hlsl` | P3 | Diligent shader entry points for tone mapping and optional luminance capture. |
| `ToneMapper/ToneMapping.ps.hlsli` | `assets/shaders/PostProcessing/ToneMapper/ToneMapping.ps.hlsli` | P3 | Tone-map operators: `Linear`, `Reinhard`, `ReinhardModified`, `HejiHableAlu`, `HableUc2`, `Aces`. |
| `ToneMapper/luminance_ps.hlsl` | `assets/shaders/PostProcessing/ToneMapper/Luminance.psh` | P3 | Auto-exposure luminance prepass. CPU readback is optional and must be gated if implemented. |
| `ProcessingPasses/PostProcess.h` | `src/RTXPTPostProcessPass.hpp` | P4, P7, P8 | Shared post-process effect wrapper. Reference bloom/LDR tests land before stable-plane/NRD variants. |
| `ProcessingPasses/PostProcess.cpp` | `src/RTXPTPostProcessPass.cpp` | P4, P7, P8 | Diligent compute/graphics dispatch for HDR/LDR post-process and future stable-plane merge/NRD paths. |
| `ProcessingPasses/PostProcess.hlsl` | `assets/shaders/PostProcessing/RTXPTPostProcess.csh` | P4, P7, P8 | HDR bloom/test hooks, LDR edge-detection/test hook, `NoDenoiserFinalMerge`, NRD prepare/final-merge variants. Advanced variants stay disabled until prerequisites exist. |
| `Shaders/Bindings/ShaderResourceBindings.hlsli` | `assets/shaders/PostProcessing/RTXPTPostProcessBindings.hlsli` or local declarations in each post-process shader | P4-P8 | Resource naming reference: `t_LdrColorScratch`, `u_OutputColor`, `u_ProcessedOutputColor`, `u_PostTonemapOutputColor`. Diligent binding slots may differ, but names and ownership should remain recognizable. |
| `Sample.cpp::PostProcessPreToneMapping` | `src/RTXPTPostProcessPipeline.cpp` and `src/RTXPTPostProcessPass.cpp` | P4 | HDR post-process scheduling. Bloom is first; `TestRaygenPP_HDR` is optional. |
| `Sample.cpp::PostProcessPostToneMapping` | `src/RTXPTPostProcessPipeline.cpp` and `src/RTXPTPostProcessPass.cpp` | P4 | LDR post-process scheduling after `LdrColorScratch` exists. |
| `Sample.cpp` final `m_CommonPasses->BlitTexture` | `src/RTXPTBlitPass.{hpp,cpp}` or an existing Diligent full-screen helper | P5 | Final swapchain copy. Normal source is `LdrColor`, not `OutputColor`. |
| `SampleUI.h` / `SampleUI.cpp` tone-mapping and post-process controls | `src/RTXPTSample.{hpp,cpp}` | P3-P5 | Existing disabled controls become live as each pass lands. UI changes must request only the histories they invalidate. |
| `Shaders/PathTracer/PathTracer.hlsli::CommitPixel` | `assets/shaders/PathTracer/PathTracerSample.rgen` | P2 | Diligent raygen writes raw `pathRadiance` to `u_Output` in reference mode. Accumulation and tone mapping are not raygen responsibilities. |

### Phase 6 Resource Contract

| Resource | Diligent owner | Format target | Size | Producer | Consumer | Notes |
|---|---|---|---|---|---|---|
| `OutputColor` | `RTXPTRenderTargets` | Prefer `TEX_FORMAT_RGBA16_FLOAT`; allow a documented closest supported HDR UAV fallback | render size | Raygen reference path tracer in P2; stable-plane merge in P7 | Accumulation, TAA/DLSS/DLSS-RR, HDR post-process | Raw HDR radiance only. It is not display-ready and must not contain ACES output. |
| `AccumulatedRadiance` | `RTXPTRenderTargets` | `TEX_FORMAT_RGBA32_FLOAT` | render size | `RTXPTAccumulationPass` | `RTXPTAccumulationPass` | Reference-mode accumulation history. Unsupported UAV format disables accumulation with a visible reason. |
| `ProcessedOutputColor` | `RTXPTRenderTargets` | Same HDR format family as `OutputColor` | display size after P6, render size until render/display split exists | Accumulation, TAA, DLSS, DLSS-RR, or final merge | HDR post-process and tone mapping | This is the HDR image that tone mapping reads. |
| `LdrColor` | `RTXPTRenderTargets` | `TEX_FORMAT_RGBA8_UNORM` or Diligent-supported sRGB equivalent when needed | display size | `RTXPTToneMappingPass` | LDR post-process, overlays, final blit | Normal final display source. |
| `LdrColorScratch` | `RTXPTRenderTargets` | Match `LdrColor` | display size | Copy/ping-pong before LDR effects | LDR post-process | Exists before LDR edge/test pass is enabled. |
| `ComputeColor` | `RTXPTRenderTargets` | Existing debug format | swapchain size | `RTXPTComputePass` | Debug display only | Diagnostic path only; not part of the long-term post-processing framework. |

### Phase 6 Behavioral Contracts

- `PathTracerSample.rgen` writes one raw HDR sample or debug radiance value to `u_Output`.
- `PathTracerSample.rgen` must not write `u_AccumulationBuffer` after P2.
- `PathTracerSample.rgen` must not call `ToneMapACES` after P3.
- `PathTracerConstants::exposureScale` remains a temporary bridge until P3, then tone-mapping exposure data moves into tone-map pass state.
- Disabling tone mapping is a post-process pass-through from `ProcessedOutputColor` to `LdrColor`; it is not a raygen macro or branch.
- Tone mapping and auto exposure must not feed back into `AccumulatedRadiance`.
- Bloom, LDR edge/test effects, overlays, TAA, NRD, DLSS, and DLSS-RR are optional consumers/producers around the base `OutputColor -> ProcessedOutputColor -> LdrColor` chain.
- P1-P5 must stay backend-neutral for D3D12 and Vulkan. P8 integrations may be D3D12/NVIDIA-only, but must compile out cleanly elsewhere.
```

- [ ] **Step 2: Verify the mapping section was inserted once**

Run:

```powershell
rg -n "Phase 6 Post-Processing Pipeline Mapping|Phase 6 Source-to-Destination Map|Phase 6 Resource Contract|Phase 6 Behavioral Contracts" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: one match for each heading.

- [ ] **Step 3: Verify all required RTXPT-fork filenames are named**

Run:

```powershell
rg -n "RenderTargets|AccumulationPass|ToneMappingPasses|ToneMapping_cb|ToneMapping\\.hlsl|ToneMapping\\.ps\\.hlsli|luminance_ps|PostProcess|ShaderResourceBindings|SampleUI|PathTracer\\.hlsli" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: matches in the new Phase 6 section.

### Task 3: Link the Spec to This Plan

**Files:**
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Add the P0 plan link**

In the `### Phase P0: Mapping and Contract Prep` section, after the milestone line, add:

```markdown
- Follow-up plan: `docs/superpowers/plans/2026-06-01-rtxpt-post-processing-phase-p0-mapping-contract-prep.md`.
- P0 acceptance gate: `RTXPT_FORK_MAPPING.md` must name every RTXPT-fork post-processing source anchor, the Diligent owner file or planned pass file, the raw HDR `OutputColor` contract, and the structured Phase 6 marker locations before P1 implementation starts.
```

- [ ] **Step 2: Verify the spec links this plan once**

Run:

```powershell
rg -n "rtxpt-post-processing-phase-p0-mapping-contract-prep|P0 acceptance gate" docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md
```

Expected: both phrases appear once in the Phase P0 section.

### Task 4: Replace Broad Phase 6 Notes with Structured Markers

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Add render-target contract marker**

In `RTXPTRenderTargets.hpp`, above `class RTXPTRenderTargets`, add:

```cpp
// TODO(RTXPT-Port Phase 6/P1): Split the current display-ready OutputColor
// contract into RTXPT-fork-style raw HDR OutputColor, AccumulatedRadiance,
// ProcessedOutputColor, LdrColor, and LdrColorScratch targets.
```

Expected: one structured P1 marker in the header.

- [ ] **Step 2: Add render-target format marker**

In `RTXPTRenderTargets.cpp`, immediately before the `CreateTarget(pDevice, "RTXPT OutputColor", m_OutputColor)` call, add:

```cpp
    // TODO(RTXPT-Port Phase 6/P1): OutputColor becomes a raw HDR UAV
    // (prefer RGBA16F) instead of the current RGBA8 display target.
```

Expected: the marker is next to the current `m_Format`-driven `OutputColor` creation.

- [ ] **Step 3: Add ray tracing binding marker**

In `RTXPTRayTracingPass.cpp`, next to the `.AddVariable(SHADER_TYPE_RAY_GEN, "u_AccumulationBuffer", ...)` line, add:

```cpp
        // TODO(RTXPT-Port Phase 6/P2): remove this raygen accumulation UAV
        // binding after RTXPTAccumulationPass owns accumulation.
```

Expected: the marker makes `u_AccumulationBuffer` ownership temporary.

- [ ] **Step 4: Add UI state marker**

In `RTXPTSample.hpp`, replace the broad Phase 6 tone-mapping comments above `RTXPTReferenceUIState` with:

```cpp
// Reference-mode UI state, mirroring the reference subset of RTXPT-fork's SampleUIData
// (D:/RTXPT-fork/Rtxpt/SampleUI.h). Phase 6/P3 moves tone-mapping exposure
// from raygen-side exposureScale into the dedicated tone-mapping pass state.
```

Expected: the comment names P3 and does not imply raygen exposure ownership is permanent.

- [ ] **Step 5: Add frame-constant marker**

In `RTXPTFrameConstants.hpp`, replace the `exposureScale` comment with:

```cpp
    float  exposureScale            = 1.0f; // TODO(RTXPT-Port Phase 6/P3): remove after tone-mapper owns exposure.
```

Expected: the marker is attached to the temporary CPU-side field.

- [ ] **Step 6: Add shader shared-constant marker**

In `PathTracerShared.h`, replace the `exposureScale` comment with:

```hlsl
    float exposureScale;            // TODO(RTXPT-Port Phase 6/P3): remove after tone-mapper owns exposure.
```

Expected: the CPU/GPU comments match.

- [ ] **Step 7: Replace raygen end marker**

In `PathTracerSample.rgen`, replace:

```hlsl
// TODO(RTXPT-Port Phase 6): Move tone mapping from raygen into the dedicated post-process chain.
```

with:

```hlsl
// TODO(RTXPT-Port Phase 6/P2): write raw HDR pathRadiance to u_Output and move
// reference accumulation from raygen into RTXPTAccumulationPass.
// TODO(RTXPT-Port Phase 6/P3): remove raygen ToneMapACES/exposureScale use after
// RTXPTToneMappingPass owns ProcessedOutputColor -> LdrColor.
```

Expected: no unscoped `TODO(RTXPT-Port Phase 6)` remains in raygen.

- [ ] **Step 8: Replace disabled tone-mapping tooltip text**

In `RTXPTSample.cpp`, keep the control disabled, but change the comment and tooltip to:

```cpp
            // TODO(RTXPT-Port Phase 6/P3): make this live when RTXPTToneMappingPass
            // owns ProcessedOutputColor -> LdrColor.
            ImGui::BeginDisabled(true);
            ImGui::Checkbox("Enable tone mapping", &m_ReferenceUI.EnableToneMapping);
            ImGui::EndDisabled();
            PlaceholderTooltip("Tone mapping is still raygen-side ACES; Phase 6/P3 moves it into the post-process chain.");
```

Expected: UI still behaves the same, but points to P3.

- [ ] **Step 9: Add render orchestration marker**

In `RTXPTSample.cpp`, immediately before `const bool TraceExecuted =` in `RTXPTSample::Render`, add:

```cpp
    // TODO(RTXPT-Port Phase 6/P1-P5): replace the direct Trace -> optional debug
    // compute -> blit path with OutputColor -> ProcessedOutputColor -> LdrColor
    // post-processing, keeping RTXPTBlitPass as the final swapchain copy.
```

Expected: the current frame-tail chain has one structured orchestration marker.

- [ ] **Step 10: Verify marker structure**

Run:

```powershell
rg -n "TODO\\(RTXPT-Port Phase 6" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected: every match includes `/P1`, `/P2`, `/P3`, or `/P1-P5`. There should be no source match with exactly `TODO(RTXPT-Port Phase 6):`.

### Task 5: Verify Mapping Completeness

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`
- Verify: structured source markers

- [ ] **Step 1: Check required resource names**

Run:

```powershell
rg -n "OutputColor|AccumulatedRadiance|ProcessedOutputColor|LdrColor|LdrColorScratch|ComputeColor" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: each resource name appears in the Phase 6 section with owner and contract context.

- [ ] **Step 2: Check required future Diligent pass names**

Run:

```powershell
rg -n "RTXPTPostProcessPipeline|RTXPTAccumulationPass|RTXPTToneMappingPass|RTXPTPostProcessPass|RTXPTBlitPass|RTXPTComputePass" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md
```

Expected: matches explain ownership. `RTXPTComputePass` must be documented as diagnostic-only.

- [ ] **Step 3: Check required behavior phrases**

Run:

```powershell
rg -n "raw HDR|display-ready|ToneMapACES|exposureScale|ProcessedOutputColor -> LdrColor|not raygen responsibilities|diagnostic-only|fail closed" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected: matches show that P0 records the contract and marks temporary debt without changing behavior.

- [ ] **Step 4: Confirm no implementation files were created**

Run:

```powershell
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTPostProcessPipeline.hpp
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTAccumulationPass.hpp
Test-Path DiligentSamples\Samples\RTXPT\src\RTXPTToneMappingPass.hpp
Test-Path DiligentSamples\Samples\RTXPT\assets\shaders\PostProcessing
```

Expected: every command prints `False`. P0 names future files but does not implement them.

### Task 6: Review and Commit P0 Contract Prep

**Files:**
- Review: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Review: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Review: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Review: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- Review: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Review submodule diff**

Run:

```powershell
git -C DiligentSamples diff -- Samples/RTXPT/RTXPT_FORK_MAPPING.md Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
```

Expected: documentation, comments, and structured markers only. No C++ or HLSL behavior changes.

- [ ] **Step 2: Review top-level diff**

Run:

```powershell
git diff -- docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md docs/superpowers/plans/2026-06-01-rtxpt-post-processing-phase-p0-mapping-contract-prep.md
```

Expected: spec link plus this plan only.

- [ ] **Step 3: Commit the DiligentSamples submodule changes**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen
git -C DiligentSamples commit -m "docs(rtxpt): map post-processing phase 6 contracts" -m "Co-Authored-By: GPT 5.5"
```

Expected: a DiligentSamples commit is created.

- [ ] **Step 4: Commit the top-level plan/spec changes**

Run:

```powershell
git add DiligentSamples docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md docs/superpowers/plans/2026-06-01-rtxpt-post-processing-phase-p0-mapping-contract-prep.md
git commit -m "docs(rtxpt): plan post-processing phase p0" -m "Co-Authored-By: GPT 5.5"
```

Expected: a top-level commit is created, including the submodule pointer update and plan/spec changes.

## Completion Verification

Run:

```powershell
rg -n "Phase 6 Post-Processing Pipeline Mapping|OutputColor.*raw HDR|AccumulatedRadiance|ProcessedOutputColor|LdrColorScratch|RTXPTPostProcessPipeline|RTXPTAccumulationPass|RTXPTToneMappingPass" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "TODO\\(RTXPT-Port Phase 6" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
rg -n "rtxpt-post-processing-phase-p0-mapping-contract-prep|P0 acceptance gate" docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md
git -C DiligentSamples status --short
git status --short
```

Expected:

- The mapping command prints the Phase 6 section, all core resource names, and all future Diligent pass owners.
- The marker command prints only structured Phase 6 markers with `/P1`, `/P2`, `/P3`, or `/P1-P5`.
- The spec command prints the plan link and acceptance gate.
- `git -C DiligentSamples status --short` is clean after the submodule commit.
- `git status --short` is clean after the top-level commit, unless the executor intentionally skipped commits.

## P0 Handoff Criteria

P0 is complete when:

- `RTXPT_FORK_MAPPING.md` names every RTXPT-fork post-processing source file required by the Phase 6 spec.
- Every listed RTXPT-fork source has a Diligent owner file or a planned Diligent-native pass file.
- The mapping states that `OutputColor` is raw HDR radiance and `LdrColor` is the normal final blit source.
- Broad source comments that previously said only `Phase 6` now point to specific P1/P2/P3/P1-P5 scopes.
- No behavior, shader binding, render target format, or UI enablement changed in P0.
- Verification commands above were run and their results were recorded in the final execution summary.

## Self-Review Notes

- Spec coverage: P0 goal G0 is covered by Tasks 1-2; the follow-up plan link is covered by Task 3; structured markers are covered by Task 4; verification is covered by Task 5 and Completion Verification.
- Placeholder scan: the only `TODO` strings in this plan are explicit `TODO(RTXPT-Port Phase 6/...)` marker text required by the spec. There are no unresolved placeholders.
- Type/name consistency: the plan consistently uses `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, `LdrColorScratch`, `RTXPTPostProcessPipeline`, `RTXPTAccumulationPass`, `RTXPTToneMappingPass`, and `RTXPTPostProcessPass`.
