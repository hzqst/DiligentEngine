# RTXPT Realtime Fill Stable Planes Source Map

## Purpose

This map records the source-to-port contract for restoring RTXPT-fork `PATH_TRACER_MODE_FILL_STABLE_PLANES` behavior in Diligent RTXPT.

## Core Flow

| RTXPT-fork source | Diligent target | Notes |
|---|---|---|
| `Rtxpt/Shaders/PathTracerSample.hlsl::RAYGEN_ENTRY` | `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen::main` | Keep `FirstHitFromVBuffer -> while(path.isActive()) -> CommitPixel` for realtime variants. |
| `PathTracer.hlsli::HandleHit` | `PathTracer.hlsli::HandleHit` | Restore emission, stable-plane handling, scatter, NEE, termination ordering. |
| `PathTracer.hlsli::GenerateScatterRay` | `PathTracer.hlsli::GenerateScatterRay` | Use Diligent `SampleBSDF` and write `PathState` fields. |
| `PathTracerNEE.hlsli::HandleNEE` | `PathTracer.hlsli::HandleNEE` | Use existing Diligent `SampleDirectLightNEE` and `SampleEnvironmentNEE` helpers. |
| `PathTracerStablePlanes.hlsli::StablePlanesOnScatter` | existing `PathTracerStablePlanes.hlsli::StablePlanesOnScatter` | Call after fill-mode scatter. |
| `StablePlanes.hlsli::CommitDenoiserRadiance` | existing `StablePlanes.hlsli::CommitDenoiserRadiance` | Preserve storage contract. |

## Required Divergences

- Diligent keeps `RTXPTRayTracingPass` and Diligent PSO/SBT ownership instead of Donut/NVRHI pipeline objects.
- Diligent uses `PathTracerSample.rgen`, `.rchit`, `.rmiss`, and `.rahit` standalone shader files instead of a single Donut pipeline package.
- Visibility rays use a Diligent SBT ray type with `RTXPT_VISIBILITY_RAY_INDEX` and a `visibilityMain` miss entry to avoid mutating primary `PathState`.
- DLSS-RR, SER, OMM, and RTXDI final shading are outside this fill-path repair.

## Acceptance Links

- Spec: `docs/superpowers/specs/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1-spec.md`
- Implementation plan: `docs/superpowers/plans/2026-06-04-rtxpt-realtime-fill-stable-planes-1to1.md`
