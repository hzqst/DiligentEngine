# RTXPT Post-Processing Phase P5 Presentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `LdrColor` the single normal presentation source for RTXPT and keep `RTXPTBlitPass` as the final swapchain copy after P4 post-processing.

**Architecture:** Keep the existing Diligent-native blit pass, but tighten the presentation contract so the sample renders only `RTXPTRenderTargets::GetPresentationSRV()`, which returns `LdrColor`. Detach the legacy debug compute display path from the render tail and UI so optional diagnostic output can no longer override the normal swapchain source.

**Tech Stack:** C++17, Diligent Engine texture view and swapchain APIs, ImGui diagnostics, CMake RTXPT sample target, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P5.
- P4 render order exists in `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`: trace, accumulation, pre-tone post-process, tone mapping, post-tone post-process, optional debug compute, final blit.
- `RTXPTSample::Render()` currently lets `m_EnableDebugComputePass` replace the final display source with `ComputeColor`.
- `RTXPTRenderTargets` exposes `GetDisplaySRV(bool UseComputeOutput)`, which encodes the same old branch and should not remain the presentation contract.
- `RTXPTBlitPass` is already a focused fullscreen swapchain copy and can stay unchanged unless verification finds it cannot present `LdrColor`.
- `RTXPT_FORK_MAPPING.md` already maps upstream final `Blit` to `RTXPTBlitPass`, but its resource table still describes overlays and `ComputeColor` as display-facing diagnostics.

## RTXPT-Fork Anchors

Read these before editing:

- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2194-2214` - upstream frame tail runs HDR post-process, tone mapping, LDR post-process, then blits `LdrColor`.
- `D:/RTXPT-fork/Rtxpt/SampleCommon/RenderTargets.h:35-36` - upstream `LdrColor` is final post-tonemapped color and `LdrColorScratch` is only ping-pong scratch.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp:923-965` - current Diligent P4/P5 tail and debug compute display override.
- `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp:75-78` and `.cpp:263-279` - current compute/display accessors.
- `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp:98-128` - current final swapchain copy.

## Scope Boundaries

- Do not port shader debug buffers, zoom overlay, or debug-line overlay in P5.
- Do not replace `RTXPTBlitPass` unless a local Diligent fullscreen helper is already preferred by maintainers during execution. This plan keeps `RTXPTBlitPass`.
- Do not delete `RTXPTComputePass.{hpp,cpp}` files in this phase. Detach their sample entry points and leave any file deletion to a separate cleanup decision.
- Do not change raygen output, accumulation math, tone-mapping operators, bloom behavior, LDR edge detection, or render target formats.
- Do not let `OutputColor`, `ComputeColor`, `ProcessedOutputColor`, or scratch targets become the normal swapchain source.
- Backend expectation stays D3D12 and Vulkan for P1-P5.

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp` - replace the boolean display helper with explicit `GetPresentationSRV()`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp` - implement `GetPresentationSRV()` as `LdrColor` only.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp` - remove legacy debug compute ownership from the sample presentation path.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` - stop creating `ComputeColor`, stop dispatching debug compute at frame tail, and blit the presentation SRV.
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md` - document the P5 presentation contract and remove overlay/display wording from the resource contract.
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md` - add the P5 follow-up plan link under Phase P5.
- No CMake changes are required because P5 does not add files and does not delete the legacy compute pass files.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. Existing dirty files must be inspected and preserved before editing.

- [ ] **Step 2: Confirm P5 spec no longer asks for overlays**

Run:

```powershell
rg -n -i "shader debug|debug-line|debug line|zoom|diagnostic overlays|overlays|overlay|ShaderDebug|DebugLines" docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md
```

Expected: no matches.

- [ ] **Step 3: Confirm the current legacy presentation override**

Run:

```powershell
rg -n "m_EnableDebugComputePass|ComputeExecuted|GetDisplaySRV|GetComputeColorSRV|Debug compute pass" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected before P5 implementation: matches show the debug compute toggle, dispatch branch, `GetDisplaySRV(bool)`, and compute display accessors.

- [ ] **Step 4: Confirm the current P4 render order**

Run:

```powershell
rg -n "RunAccumulation|RunPreToneMapping|RunToneMapping|RunPostToneMapping|m_BlitPass.Render" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected order: `RunAccumulation`, `RunPreToneMapping`, `RunToneMapping`, `RunPostToneMapping`, then `m_BlitPass.Render`.

### Task 1: Make the Presentation Source Explicit

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`

- [ ] **Step 1: Replace the boolean display accessor declaration**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`, replace:

```cpp
    ITextureView* GetComputeColorUAV() const;
    ITextureView* GetComputeColorSRV() const;
    ITextureView* GetDisplaySRV(bool UseComputeOutput) const;
```

with:

```cpp
    ITextureView* GetComputeColorUAV() const;
    ITextureView* GetComputeColorSRV() const;
    ITextureView* GetPresentationSRV() const;
```

Expected: the public presentation contract has no boolean that can select non-`LdrColor` sources.

- [ ] **Step 2: Replace the boolean display accessor implementation**

In `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`, replace:

```cpp
ITextureView* RTXPTRenderTargets::GetDisplaySRV(bool UseComputeOutput) const
{
    if (UseComputeOutput && m_ComputeColor)
        return GetComputeColorSRV();

    return GetLdrColorSRV();
}
```

with:

```cpp
ITextureView* RTXPTRenderTargets::GetPresentationSRV() const
{
    return GetLdrColorSRV();
}
```

Expected: `LdrColor` is the only source returned by the presentation accessor.

- [ ] **Step 3: Verify accessor contract**

Run:

```powershell
rg -n "GetPresentationSRV|GetDisplaySRV" DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: `GetPresentationSRV` declaration and definition are present; `GetDisplaySRV` has no matches.

- [ ] **Step 4: Commit the render-target contract**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp
git -C DiligentSamples commit -m "fix(rtxpt): make presentation source explicit" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only the render-target accessor rename and implementation change.

### Task 2: Present Only `LdrColor` From the Frame Tail

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Stop requesting `ComputeColor` in `EnsureRenderTargets()`**

In `RTXPTSample::EnsureRenderTargets()`, replace the `Resize` call:

```cpp
    const bool                     Ok = m_RenderTargets.Resize(m_pDevice,
                                                              SCDesc.Width,
                                                              SCDesc.Height,
                                                              Formats,
                                                              m_FeatureCaps.ComputeShaders,
                                                              m_FeatureCaps.RayTracing);
```

with:

```cpp
    constexpr bool CreateComputeOutput = false;
    const bool     Ok                  = m_RenderTargets.Resize(m_pDevice,
                                                               SCDesc.Width,
                                                               SCDesc.Height,
                                                               Formats,
                                                               CreateComputeOutput,
                                                               m_FeatureCaps.RayTracing);
```

Expected: normal render target creation no longer allocates `ComputeColor` for presentation.

- [ ] **Step 2: Stop requesting `ComputeColor` in `WindowResize()`**

In `RTXPTSample::WindowResize()`, replace the `Resize` call:

```cpp
    const bool                     Ok = m_RenderTargets.Resize(m_pDevice,
                                                              Width,
                                                              Height,
                                                              Formats,
                                                              m_FeatureCaps.ComputeShaders,
                                                              m_FeatureCaps.RayTracing);
```

with:

```cpp
    constexpr bool CreateComputeOutput = false;
    const bool     Ok                  = m_RenderTargets.Resize(m_pDevice,
                                                               Width,
                                                               Height,
                                                               Formats,
                                                               CreateComputeOutput,
                                                               m_FeatureCaps.RayTracing);
```

Expected: resize behavior matches first render-target creation.

- [ ] **Step 3: Replace debug compute presentation branch**

In `RTXPTSample::Render()`, replace:

```cpp
    const bool ComputeExecuted =
        m_EnableDebugComputePass &&
        m_DebugComputePass.Dispatch(m_pImmediateContext,
                                    m_RenderTargets.GetOutputColorSRV(),
                                    m_RenderTargets.GetComputeColorUAV(),
                                    m_RenderTargets.GetWidth(),
                                    m_RenderTargets.GetHeight());

    ITextureView* pDisplaySRV = ComputeExecuted ? m_RenderTargets.GetComputeColorSRV() :
                                                  m_RenderTargets.GetLdrColorSRV();
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pDisplaySRV))
```

with:

```cpp
    ITextureView* pPresentationSRV = m_RenderTargets.GetPresentationSRV();
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pPresentationSRV))
```

Expected: final blit always receives the P5 presentation source, which is `LdrColor`.

- [ ] **Step 4: Verify frame-tail contract**

Run:

```powershell
rg -n "ComputeExecuted|m_EnableDebugComputePass|GetComputeColorSRV|GetPresentationSRV|m_BlitPass.Render" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: `GetPresentationSRV` appears immediately before `m_BlitPass.Render`; `ComputeExecuted`, `m_EnableDebugComputePass`, and `GetComputeColorSRV` have no matches in `RTXPTSample.cpp`.

- [ ] **Step 5: Commit the frame-tail presentation change**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "fix(rtxpt): present final LdrColor" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains the render-target resize and final blit source changes.

### Task 3: Detach Legacy Debug Compute From Sample UI and Ownership

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Remove the debug compute include**

In `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`, remove:

```cpp
#include "RTXPTComputePass.hpp"
```

Expected: `RTXPTSample.hpp` no longer depends on the legacy compute pass type.

- [ ] **Step 2: Remove sample members for debug compute presentation**

In `RTXPTSample` private members, remove:

```cpp
    RTXPTComputePass               m_DebugComputePass;
```

and remove:

```cpp
    bool                           m_EnableDebugComputePass      = false;
```

Expected: sample state no longer has a toggle or pass instance that can replace final presentation output.

- [ ] **Step 3: Remove debug compute initialization**

In `RTXPTSample::CreateRTPipelines()`, remove:

```cpp
    m_DebugComputePass.Initialize(m_pDevice,
                                  m_pEngineFactory,
                                  "RTXPT debug compute pass",
                                  "RTXPTDebugCompute.csh",
                                  m_FrameConstantsCB,
                                  m_FeatureCaps.ComputeShaders);
```

Expected: P5 setup no longer creates the legacy diagnostic display pass.

- [ ] **Step 4: Remove debug compute UI diagnostics**

In `RTXPTSample::UpdateUI()`, remove:

```cpp
        const RTXPTComputePassStats&           ComputeStats = m_DebugComputePass.GetStats();
```

and replace this block:

```cpp
        ImGui::Separator();
        ImGui::Checkbox("Debug compute pass", &m_EnableDebugComputePass);
        ImGui::Text("Compute dispatch: %s", m_DebugComputePass.IsReady() ? "ready" : "not ready");
        ImGui::Text("Compute executed: %s", ComputeStats.LastDispatchExecuted ? "yes" : "no");
        ImGui::Text("Compute dispatch count: %u", ComputeStats.DispatchCount);
        ImGui::Text("Blit draw count: %u", m_BlitPass.GetDrawCount());
```

with:

```cpp
        ImGui::Separator();
        ImGui::Text("Presentation source: LdrColor");
        ImGui::Text("Blit draw count: %u", m_BlitPass.GetDrawCount());
```

Expected: UI still reports final presentation status but exposes no alternate display source.

- [ ] **Step 5: Verify sample no longer references debug compute**

Run:

```powershell
rg -n "RTXPTComputePass|m_DebugComputePass|m_EnableDebugComputePass|Debug compute pass|ComputeStats" DiligentSamples/Samples/RTXPT/src/RTXPTSample.*
```

Expected: no matches.

- [ ] **Step 6: Commit sample ownership cleanup**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "refactor(rtxpt): detach legacy debug compute display" -m "Co-Authored-By: GPT 5.5"
```

Expected: commit contains only sample include/member/init/UI cleanup.

### Task 4: Update Mapping and Spec Links

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Update the P5 mapping row**

In `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`, replace the P5 final blit row:

```markdown
| `Sample.cpp` final `m_CommonPasses->BlitTexture` | `src/RTXPTBlitPass.{hpp,cpp}` or an existing Diligent full-screen helper | P5 | Final swapchain copy. Normal source is `LdrColor`, not `OutputColor`. |
```

with:

```markdown
| `Sample.cpp` final `m_CommonPasses->BlitTexture` | `src/RTXPTBlitPass.{hpp,cpp}` plus `RTXPTRenderTargets::GetPresentationSRV()` | P5 | Final swapchain copy. `GetPresentationSRV()` returns `LdrColor`, and no debug/compute output may replace the normal swapchain source. |
```

Expected: mapping names both the blit pass and the explicit source contract.

- [ ] **Step 2: Update resource contract rows**

In the resource table, replace:

```markdown
| `LdrColor` | `RTXPTRenderTargets` | `TEX_FORMAT_RGBA8_UNORM` or Diligent-supported sRGB equivalent when needed | display size | `RTXPTToneMappingPass` | LDR post-process, overlays, final blit | Normal final display source. |
| `ComputeColor` | `RTXPTRenderTargets` | Existing debug format | swapchain size | `RTXPTComputePass` | Debug display only | Diagnostic path only; not part of the long-term post-processing framework. |
```

with:

```markdown
| `LdrColor` | `RTXPTRenderTargets` | `TEX_FORMAT_RGBA8_UNORM` or Diligent-supported sRGB equivalent when needed | display size | `RTXPTToneMappingPass` | LDR post-process, final blit | Normal final display source. |
| `ComputeColor` | `RTXPTRenderTargets` | Existing debug format | swapchain size | Legacy `RTXPTComputePass` only when explicitly reintroduced outside P5 | No normal presentation consumer | Diagnostic scratch only; P5 does not request or present it. |
```

Expected: mapping no longer describes overlays or compute output as display consumers.

- [ ] **Step 3: Update behavior contract wording**

Replace:

```markdown
- Bloom, LDR edge/test effects, overlays, TAA, NRD, DLSS, and DLSS-RR are optional consumers/producers around the base `OutputColor -> ProcessedOutputColor -> LdrColor` chain.
```

with:

```markdown
- Bloom, LDR edge/test effects, TAA, NRD, DLSS, and DLSS-RR are optional consumers/producers around the base `OutputColor -> ProcessedOutputColor -> LdrColor` chain.
- Final presentation reads only `LdrColor` through `RTXPTRenderTargets::GetPresentationSRV()`.
```

Expected: P5 display ownership is explicit in the behavior contracts.

- [ ] **Step 4: Add the P5 follow-up plan link to the spec**

In `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, update Phase P5:

```markdown
### Phase P5: Presentation
- Goal: G5.
- Touches: final blit.
- Milestone: `LdrColor` is the only normal swapchain source.
- Follow-up plan: `docs/superpowers/plans/2026-06-03-rtxpt-post-processing-phase-p5-presentation.md`.
```

Expected: the spec points to this implementation plan.

- [ ] **Step 5: Verify docs no longer mention removed overlays**

Run:

```powershell
rg -n -i "shader debug|debug-line|debug line|zoom|diagnostic overlays|overlays|overlay|ShaderDebug|DebugLines" docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
```

Expected: no matches.

- [ ] **Step 6: Commit documentation updates**

Run:

```powershell
git -C DiligentSamples add Samples/RTXPT/RTXPT_FORK_MAPPING.md
git -C DiligentSamples commit -m "docs(rtxpt): document P5 presentation contract" -m "Co-Authored-By: GPT 5.5"
git add docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md docs/superpowers/plans/2026-06-03-rtxpt-post-processing-phase-p5-presentation.md
git commit -m "docs(rtxpt): add P5 presentation plan" -m "Co-Authored-By: GPT 5.5"
```

Expected: submodule commit contains mapping changes; top-level commit contains spec link and this plan.

### Task 5: Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.*`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Verify: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.*`
- Verify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
- Verify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Run source-level presentation checks**

Run:

```powershell
rg -n "GetPresentationSRV|m_BlitPass.Render|RunPostToneMapping" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
rg -n "ComputeExecuted|m_EnableDebugComputePass|m_DebugComputePass|Debug compute pass|GetDisplaySRV" DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: first command shows `GetPresentationSRV` and final blit after `RunPostToneMapping`; second command has no matches.

- [ ] **Step 2: Run resource allocation checks**

Run:

```powershell
rg -n "CreateComputeOutput = false|GetComputeColorSRV|GetComputeColorUAV|RTXPT ComputeColor" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp
```

Expected: `CreateComputeOutput = false` appears in `EnsureRenderTargets()` and `WindowResize()`. `GetComputeColorSRV`, `GetComputeColorUAV`, and `RTXPT ComputeColor` may remain only inside `RTXPTRenderTargets.cpp`; they must not be referenced by `RTXPTSample.cpp`.

- [ ] **Step 3: Verify render order**

Run:

```powershell
rg -n "RunAccumulation|RunPreToneMapping|RunToneMapping|RunPostToneMapping|GetPresentationSRV|m_BlitPass.Render" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected order: `RunAccumulation`, `RunPreToneMapping`, `RunToneMapping`, `RunPostToneMapping`, `GetPresentationSRV`, `m_BlitPass.Render`.

- [ ] **Step 4: Build the RTXPT target**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: target builds without C++ or HLSL compile errors. If the build directory does not exist, run:

```powershell
.\build-x64-Debug.bat
```

Expected: configure/build completes and produces the RTXPT sample target.

- [ ] **Step 5: Runtime smoke on D3D12**

Launch the RTXPT sample on D3D12 from the Debug build output.

Expected:

1. The sample launches and presents a tone-mapped image.
2. Bloom on/off still affects the image before tone mapping.
3. LDR edge detection on/off still affects `LdrColor` before blit.
4. The Status / Debug panel reports `Presentation source: LdrColor`.
5. There is no `Debug compute pass` checkbox.

- [ ] **Step 6: Runtime smoke on Vulkan**

Launch the RTXPT sample on Vulkan from the Debug build output.

Expected:

1. The sample launches and presents a tone-mapped image.
2. The final blit succeeds after `RunPostToneMapping`.
3. No debug compute output can be selected as the displayed image.

- [ ] **Step 7: Final status check**

Run:

```powershell
git status --short
git -C DiligentSamples status --short
```

Expected: only deliberate top-level docs changes and deliberate `DiligentSamples/Samples/RTXPT` changes are present if commits were not created during execution.

---

## Acceptance Gate

P5 is complete only when all of these are true:

- `RTXPTSample::Render()` order is `RunAccumulation -> RunPreToneMapping -> RunToneMapping -> RunPostToneMapping -> GetPresentationSRV -> RTXPTBlitPass::Render`.
- `RTXPTRenderTargets::GetPresentationSRV()` exists and returns only `GetLdrColorSRV()`.
- `RTXPTSample.*` has no `RTXPTComputePass`, `m_DebugComputePass`, `m_EnableDebugComputePass`, `ComputeExecuted`, `Debug compute pass`, or `GetDisplaySRV` references.
- `EnsureRenderTargets()` and `WindowResize()` pass `CreateComputeOutput=false` to `RTXPTRenderTargets::Resize()`.
- `RTXPTBlitPass` remains the final swapchain copy and receives `LdrColor` through the presentation accessor.
- `RTXPT_FORK_MAPPING.md` documents P5 as `LdrColor`-only presentation and contains no overlay wording.
- The P5 spec links to this plan.
- The RTXPT target builds, and D3D12/Vulkan smoke runs present the P4-processed `LdrColor`.

## Self-Review Checklist

- Spec coverage: G5 is covered by explicit `LdrColor` presentation, final blit preservation, and documentation updates.
- Scope coverage: shader debug buffers, zoom overlay, and debug-line overlay are excluded from all tasks.
- Source consistency: `GetPresentationSRV()` is the only new accessor name and is used by `RTXPTSample::Render()`.
- No deletion dependency: `RTXPTComputePass.{hpp,cpp}` files remain in the repository and are not deleted by this plan.
