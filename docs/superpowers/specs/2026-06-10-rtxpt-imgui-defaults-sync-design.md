# RTXPT ImGui Defaults Sync Design

Date: 2026-06-10
Status: Approved (pending written-spec review)

## Purpose

Synchronize the default values of the Diligent RTXPT ImGui state that has already
been ported from upstream RTXPT.

The source of truth is the upstream `SampleUI` state after construction and after
the constructor applies the Balanced performance preset:

- `D:/RTXPT-fork/Rtxpt/SampleUI.h`
- `D:/RTXPT-fork/Rtxpt/SampleUI.cpp`
- `D:/RTXPT-fork/Rtxpt/SampleCommon/CommandLine.h`
- `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.cpp`
- `D:/RTXPT-fork/External/Nrd/Include/NRDSettings.h` — required because
  `getDefaultREBLURSettings()` / `getDefaultRELAXSettings()` in `NrdConfig.cpp`
  only override a subset of fields; the remaining fields fall through to the
  `nrd::ReblurSettings` / `nrd::RelaxSettings` library defaults declared here.

## Scope

Only sync fields that already exist in Diligent:

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp::RTXPTReferenceUIState`
- `DiligentSamples/Samples/RTXPT/src/RTXPTRealtimeSettings.hpp::RTXPTRealtimeSettings`
- existing nested UI settings structs in `RTXPTRealtimeSettings.hpp`

Do not add upstream UI fields that have not been ported yet. Do not wire new
runtime behavior. Do not change shader constants, render target layouts, or
post-process contracts.

## Locked Decisions

1. Use upstream `SampleUI` constructor defaults plus `ApplyPreset(... Balanced ...)`.
2. Keep Diligent's current DLSS-RR protection. Upstream command-line default
   `RealtimeAA == 3` is not copied because local DLSS-RR remains a guarded,
   not-yet-implemented path.
3. Ignore `IntroPathTracer` local overrides, because they are a sample-specific
   post-constructor specialization rather than the requested `SampleUI`
   constructor default.
4. Preserve local-only fields where there is no upstream ported counterpart.
5. Default `NEEType` to upstream's `2` (NEE-AT). Unlike the DLSS-RR path guarded
   in decision 2, NEE-AT is already implemented in the Diligent port, so
   defaulting to it activates a working path rather than a stub. It is consumed
   by `PtConsts.NEEType` (`RTXPTSample.cpp`) and the lights baker
   (`RTXPTLightsBaker.cpp`, `NEEATEnabled = ImportanceSamplingType == 2u`),
   which is why the DLSS-RR guard does not apply here.

## Expected Default Alignment

Reference UI fields:

- `NEEType`: upstream constructor uses command-line default `2`; local should
  default to `2`.
- `NEEMISType`: upstream `SampleUIData` and Balanced preset use `1`; local should
  default to `1`.
- `NEECandidateSamples`, `NEEFullSamples`, `DiffuseBounceCount`, `EnableBloom`,
  `BloomRadius`, `BloomIntensity`, `NestedDielectricsQuality`,
  `EnableLDSamplerForBSDF`, firefly, tone mapping, camera aperture, and focal
  distance should remain aligned with upstream Balanced defaults.
- `EnvironmentMapEnabled` is local Diligent state and is not changed by this sync.

Realtime UI fields:

- `NRDMethod`: upstream `SampleUIData` defaults to `REBLUR`; local should default
  to `RTXPTNrdMethod::REBLUR`.
- `ReblurSettings.MaxFastAccumulatedFrameNum`: upstream effective default is `6`
  (`nrd::ReblurSettings` library default; `getDefaultREBLURSettings()` does not
  override it). Local currently defaults to `0` and should change to `6`.
- `ReblurSettings.HistoryFixFrameNum`: upstream effective default is `3`
  (`nrd::ReblurSettings` library default; not overridden by
  `getDefaultREBLURSettings()`). Local currently defaults to `0` and should
  change to `3`.
- `RelaxSettings.HistoryFixFrameNum`: upstream effective default is `3`
  (`nrd::RelaxSettings` library default; not overridden by
  `getDefaultRELAXSettings()`). Local currently defaults to `0` and should
  change to `3`.
- `RealtimeMode`, `RealtimeSamplesPerPixel`, `StandaloneDenoiser`,
  `RealtimeFireflyFilterEnabled`, `RealtimeFireflyFilterThreshold`, `TexLODBias`,
  stable-plane defaults, denoiser radiance clamp, NRD disocclusion thresholds,
  RELAX defaults, REBLUR defaults (the fields explicitly set by
  `getDefaultRELAXSettings()` / `getDefaultREBLURSettings()`), and DLSS-RR numeric
  clamps should remain aligned with upstream. The three NRD fall-through fields
  listed above are the only nested denoiser defaults that require a change.
- `RealtimeAA` remains `Disabled` locally to preserve the existing Diligent
  guard against incomplete DLSS-RR support.
- `DenoisingGuideDebugView` is local Diligent debug UI and is not changed.

## Verification

1. Re-run targeted `rg` checks against the local files and upstream files listed
   above.
2. Inspect `git diff` for the two target headers and confirm only default values
   changed.
3. Run the most relevant lightweight compile or static validation available for
   the touched RTXPT headers. If a full build is too costly or unavailable, report
   exactly what was not run.

