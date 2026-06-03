# RTXPT Post-Processing Pipeline Port Design

## Summary

This design defines the Phase 6 work needed to port the RTXPT-fork post-processing pipeline into
`DiligentSamples/Samples/RTXPT` after the reference path tracer completion work
(`docs/superpowers/specs/2026-05-30-rtxpt-reference-pathtracer-completion-design.md`).

The reference path-tracing algorithm goals R0-R7 are now present in the Diligent sample: the raygen-driven
N-bounce loop, firefly filtering, all-bounce NEE, emissive-triangle MIS, `LightsBaker`, `EnvMapBaker`,
BSDF/sampler fidelity, transmission/nested dielectrics, shadow-origin polish, and thin-lens camera controls
all have direct source-level counterparts. However, strict RTXPT-fork `PATH_TRACER_MODE_REFERENCE`
`PathTrace` parity is not complete until the output contract is also moved to the same post-processing flow:
RTXPT-fork writes raw HDR radiance to `OutputColor`, then runs accumulation, HDR post-process, tone mapping,
LDR post-process, and final blit. The Diligent port still accumulates in `PathTracerSample.rgen`,
applies `ToneMapACES` in raygen, writes an `RGBA8` `OutputColor`, and presents through a small blit pass.

Phase 6 therefore starts with the reference-mode display chain. Advanced realtime post-processing
(stable-plane final merge, NRD, TAA, DLSS/DLSS-RR, Streamline) is specified here as later gated work, but it
depends on the realtime/stable-plane track from the umbrella spec and must not block the reference-mode
post-processing milestone.

Reference sources are under `D:/RTXPT-fork/Rtxpt/`:

- `Sample.cpp`: `CreateRenderPasses`, `PostProcessAA`, `PostProcessPreToneMapping`,
  `ToneMappingPass::Render`, `PostProcessPostToneMapping`, final `Blit`.
- `SampleCommon/RenderTargets.*`: `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`,
  `LdrColor`, `LdrColorScratch`, temporal/denoiser/DLSS-RR resources.
- `ToneMapper/ToneMappingPasses.*`, `ToneMapper/ToneMapping.hlsl`,
  `ToneMapper/ToneMapping.ps.hlsli`, `ToneMapper/luminance_ps.hlsl`.
- `ProcessingPasses/PostProcess.*`, `ProcessingPasses/PostProcess.hlsl`.

## Current Diligent State

- `PathTracer/PathTracerSample.rgen` owns reference accumulation in `u_AccumulationBuffer`, applies
  `ToneMapACES(accumulated * exposureScale)`, and writes `u_Output` as the display-ready image.
- `RTXPTRenderTargets` owns `OutputColor` (`TEX_FORMAT_RGBA8_UNORM`), optional `ComputeColor`, and optional
  `AccumColor` (`TEX_FORMAT_RGBA32_FLOAT`).
- `RTXPTSample::Render` runs `RTXPTRayTracingPass::Trace`, optional diagnostic `RTXPTComputePass`, then
  `RTXPTBlitPass`.
- The UI already exposes the RTXPT-fork `Enable tone mapping` control as disabled, with text saying the
  configurable tone-map pass is Phase 6.
- `RTXPTComputePass` still contains a broad structured TODO for RTXDI DI/GI and denoising-guide compute
  chains; it should not become the long-term post-processing framework.

## Target RTXPT Flow

Reference-mode target flow:

```text
PathTrace writes raw HDR OutputColor
  -> AccumulationPass writes AccumulatedRadiance + ProcessedOutputColor
  -> PostProcessPreToneMapping applies optional HDR effects such as bloom
  -> ToneMappingPass writes LdrColor
  -> PostProcessPostToneMapping applies optional LDR effects
  -> final blit to swapchain
```

Realtime target flow, after the realtime/stable-plane track exists:

```text
PathTrace / stable-plane final shading writes OutputColor or stable-plane radiance
  -> NoDenoiserFinalMerge, standalone NRD, TAA, DLSS, or DLSS-RR path
  -> ProcessedOutputColor
  -> HDR post-process
  -> ToneMappingPass
  -> LDR post-process
  -> swapchain
```

## Goals

**G0 - Source-level parity audit and mapping.**
- Record every RTXPT-fork post-processing source file and its Diligent destination in
  `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`.
- Keep the existing style rule: port behavior and names where useful, but use Diligent-native pass/resource
  ownership instead of copying Donut/NVRHI APIs.

**G1 - HDR render-target contract.**
- Replace the display-ready `RGBA8` `OutputColor` contract with an HDR path-tracing output texture
  (`RGBA16F` or the closest Diligent-supported equivalent), plus `AccumulatedRadiance`,
  `ProcessedOutputColor`, `LdrColor`, and `LdrColorScratch`.
- Keep a fallback path that can still present a clear/error color if any post-process pass fails to initialize.

**G2 - Move accumulation out of raygen.**
- `PathTracerSample.rgen` writes the current raw `pathRadiance` sample to `OutputColor`.
- A Diligent-native accumulation pass reads `OutputColor`, updates `AccumulatedRadiance`, and writes
  `ProcessedOutputColor`.
- Accumulation reset semantics must continue to follow camera, scene, light, env-map, material, and UI changes.

**G3 - Port configurable tone mapping.**
- Implement `ToneMappingPass` parity: operators `Linear`, `Reinhard`, `Reinhard Modified`,
  `Heji Hable ALU`, `Hable UC2`, `Aces`; `Enable`; exposure mode; exposure compensation/value;
  auto exposure; white balance; white luminance/scale; clamp.
- Scene camera exposure metadata should feed the tone-mapper state, not raygen-side `exposureScale`.
- The existing disabled `Enable tone mapping` UI becomes live and matches RTXPT-fork's Tone Mapping section.

**G4 - Restore HDR and LDR post-process hooks.**
- Port `PostProcessPreToneMapping` for bloom first; keep the RTXPT debug `TestRaygenPP_HDR` path optional.
- Port `PostProcessPostToneMapping` for the LDR edge-detection/test path only after `LdrColorScratch`
  exists.
- Feature toggles must be independent: disabling bloom or LDR effects must leave accumulation, tone mapping,
  and blit intact.

**G5 - Presentation.**
- The normal display source becomes `LdrColor`.
- Keep the current `RTXPTBlitPass` only as the final swapchain copy or replace it with the existing Diligent
  full-screen helper if that better matches local patterns.

**G6 - TAA and render/display-size split.**
- Add `TemporalFeedback1`, `TemporalFeedback2`, `CombinedHistoryClampRelax`, motion-vector inputs, and the
  reference TAA scheduling contract only after the base HDR->LDR chain is stable.
- Support a future render size distinct from display size so DLSS/TAA/DLSS-RR can share the same target model.

**G7 - Denoising guides and no-denoiser final merge.**
- Port `DenoisingGuidesBaker` and `PostProcess::NoDenoiserFinalMerge` as the bridge from future stable-plane
  outputs to `OutputColor`.
- This goal depends on the realtime stable-plane output contract; until that track exists, keep the passes
  compiled out or disabled with visible reasons.

**G8 - Standalone NRD denoising.**
- Port the `PostProcess` prepare/final-merge compute variants for RELAX and REBLUR, plus the Diligent-native
  equivalent of `NrdIntegration`.
- This is advanced realtime scope and depends on G7 plus stable-plane radiance, normal/roughness, hit-distance,
  motion-vector, and disocclusion resources.

**G9 - DLSS / DLSS-RR / Streamline.**
- Treat DLSS and DLSS-RR as optional D3D12/NVIDIA integrations. The base post-processing chain must not depend
  on them.
- The target DLSS-RR resource contract includes `RRDiffuseAlbedo`, `RRSpecAlbedo`,
  `RRNormalsAndRoughness`, `RRSpecMotionVectors`, and display-size `ProcessedOutputColor`.

## Non-Goals

- Re-porting the realtime stable-plane path tracer itself. Phase 6 can create the merge/denoising consumers,
  but stable-plane generation belongs to the realtime/advanced track.
- Re-porting RTXDI/ReSTIR DI/GI algorithms. Their final shading hooks may later feed the post-process chain,
  but the algorithms are not part of this spec.
- A verbatim Donut/NVRHI framework clone. Passes should be Diligent-native and follow the current sample's
  ownership style.
- Blocking the reference-mode post-processing milestone on NRD, DLSS, or Streamline.

## Phase Design

### Phase P0: Mapping and Contract Prep
- Goal: G0.
- Touches: `RTXPT_FORK_MAPPING.md`, this spec's follow-up plan, and structured `TODO(RTXPT-Port Phase 6)`
  markers.
- Milestone: all source anchors and destination files are named before code work begins.
- Follow-up plan: `docs/superpowers/plans/2026-06-01-rtxpt-post-processing-phase-p0-mapping-contract-prep.md`.
- P0 acceptance gate: `RTXPT_FORK_MAPPING.md` must name every RTXPT-fork post-processing source anchor, the Diligent owner file or planned pass file, the raw HDR `OutputColor` contract, and the structured Phase 6 marker locations before P1 implementation starts.

- 明确 RTXPT-fork 的哪些后处理源文件对应 Diligent 端哪些文件或新 pass
例如 ToneMappingPasses.*、PostProcess.*、RenderTargets.* 分别落到哪里。

- 明确哪些是必须 1:1 对齐的行为契约
特别是 OutputColor 必须从当前“raygen 内累积并 tone map 后的 RGBA8”改成 RTXPT-fork 风格的“raw HDR path tracing output”。

- 清理/收窄 Phase 6 的 TODO 和 mapping 文档
避免后面 P1-P3 实现时还在争论“这个 pass 应该归谁管”“这个资源是不是 raw HDR”“tone mapping 到底在 raygen 还是后处理”。

### Phase P1: HDR Render Targets and Pass Skeleton
- Goals: G1, partial G5.
- Touches: `RTXPTRenderTargets.*`, `RTXPTSample.*`, new `RTXPTPostProcess*` or focused pass classes.
- Milestone: sample still presents through the existing blit path, but the resource graph contains
  HDR `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, and `LdrColor` placeholders.

### Phase P2: Reference Accumulation Pass
- Goal: G2.
- Touches: `PathTracerSample.rgen`, `RTXPTRayTracingPass.*`, accumulation shader/pass, frame constants.
- Milestone: raygen no longer writes `u_AccumulationBuffer` or calls `ToneMapACES`; accumulation is a separate
  pass and converges to the same image as the current path when ACES is applied afterward.
- Follow-up plan: `docs/superpowers/plans/2026-06-02-rtxpt-post-processing-phase-p2-p3-reference-accumulation-and-tonemapping.md`.

### Phase P3: ToneMappingPass
- Goal: G3.
- Touches: `ToneMapper/` shaders and C++ pass equivalents, `RTXPTSample` UI/state, scene camera exposure import.
- Milestone: `ProcessedOutputColor -> LdrColor` matches RTXPT-fork controls; disabling tone mapping is a
  pass-through, not a raygen change.
- Follow-up plan: `docs/superpowers/plans/2026-06-02-rtxpt-post-processing-phase-p2-p3-reference-accumulation-and-tonemapping.md`.

### Phase P4: Bloom and LDR Post-Process
- Goal: G4.
- Touches: bloom pass integration, `PostProcess` compute/RT hook equivalents, `LdrColorScratch`.
- Milestone: bloom and LDR edge/test effects are independently toggleable and do not disturb the base display chain.
- Follow-up plan: `docs/superpowers/plans/2026-06-02-rtxpt-post-processing-phase-p4-bloom-and-ldr-post-process.md`.

### Phase P5: Presentation
- Goal: G5.
- Touches: final blit.
- Milestone: `LdrColor` is the only normal swapchain source.
- Follow-up plan: `docs/superpowers/plans/2026-06-03-rtxpt-post-processing-phase-p5-presentation.md`.

### Phase P6: TAA and Render/Display Size Split
- Goal: G6.
- Touches: temporal targets, motion-vector contracts, jitter/update flow, render-target resize logic.
- Milestone: reference/realtime modes can choose between direct accumulation/present and TAA without breaking the
  base tone-mapped output.

### Phase P7: Denoising Guides and Stable-Plane Merge
- Goal: G7.
- Touches: DenoisingGuidesBaker equivalent, `PostProcess.hlsl` compute variants, stable-plane resource bindings.
- Milestone: once stable-plane outputs exist, `NoDenoiserFinalMerge` can produce `OutputColor` for the same
  downstream Phase P3-P5 chain.

### Phase P8: NRD and DLSS/DLSS-RR
- Goals: G8, G9.
- Touches: NRD integration layer, Streamline/DLSS optional layer, RR guide render targets and resource tagging.
- Milestone: advanced denoisers are gated, optional, and can fail closed while reference accumulation and tone
  mapping still work.

## Cross-Cutting Contracts

- **Output ownership:** `OutputColor` is raw HDR path-tracing or merged realtime radiance. `ProcessedOutputColor`
  is the HDR post-AA/accumulation result. `LdrColor` is the tone-mapped/presentable source.
- **Accumulation determinism:** tone mapping and auto exposure must not feed back into the accumulated HDR buffer.
- **Reset semantics:** camera, scene, AS, material, light, env-map, tone-mapping mode, AA, and post-process changes
  must reset only the histories they invalidate.
- **Capability gating:** compute, ray tracing, typed UAV formats, optional denoiser libraries, and optional Streamline
  support must each have explicit disabled reasons.
- **D3D12 and Vulkan:** P1-P5 are core and must work on both backends. P8 DLSS/DLSS-RR may be D3D12/NVIDIA-only but
  must compile out cleanly elsewhere.
- **Binding hygiene:** resource additions require synchronized C++ pass layout, HLSL declarations, fallback resources,
  and status/debug UI.

## Verification Strategy

Per phase:

1. Source scan for stale `ToneMapACES`, `u_AccumulationBuffer`, `exposureScale`, and broad Phase 6 TODO markers.
2. C++/HLSL layout checks for new constants and render-target bindings.
3. Build the `RTXPT` target on the configured Windows build when explicitly requested or during phase acceptance.
4. Runtime smoke on D3D12 and Vulkan for P1-P5: sample launches, accumulates, tone maps, and presents.
5. Visual parity checks against RTXPT-fork for a fixed reference scene: raw HDR path output, accumulated HDR,
   tone-mapped LDR, bloom on/off, tone mapping on/off.
6. Advanced P7-P8 checks only after their prerequisites exist: stable-plane merge correctness, NRD resource
   validity, Streamline resource tagging, and graceful fallback when unsupported.

## Initial Implementation Recommendation

Start with P1-P3 as a single reference-mode implementation plan. It is the shortest route to strict
`PATH_TRACER_MODE_REFERENCE` output-contract parity and removes the current raygen-side tone-mapping TODO. P4-P5 can
follow as visible feature work. P6-P8 should wait until the realtime/stable-plane prerequisites are scheduled.
