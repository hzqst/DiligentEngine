# SuperResolutionJitter

## Issue

When RTXPT realtime AA is set to `SuperResolution` with DLSS, visible stair-step aliasing can remain if the camera ray generation path does not use the same pixel-space jitter passed to DLSS.

## Root Cause

`ISuperResolution::GetJitterOffset()` returns jitter in pixel space, typically around `[-0.5, 0.5]`. RTXPT path tracer ray helpers also expect camera jitter in pixel space because shaders apply it directly as:

```hlsl
(float2(pixel) + float2(0.5, 0.5) + jitter) / float2(data.ViewportSize)
```

A previous local RTXPT path normalized SR jitter before assigning it to `CameraJitter`, effectively shrinking the ray-generation jitter to near zero while DLSS still received the correct pixel-space jitter. This mismatch prevents DLSS from accumulating true sub-pixel samples and shows up as obvious jagged edges.

## Correct Behavior

In `RTXPTSample::UpdateFrameConstants`, the SuperResolution branch must assign:

```cpp
CameraJitter = m_CurrentSuperResolutionFrame.Jitter;
```

Do not divide by render width/height and do not flip Y here. `MakePathTracerCameraData()` already stores the value using RTXPT-fork `BridgeCamera` convention:

```cpp
Data.Jitter = Jitter * float2{1.0f, -1.0f};
```

`RTXPTSuperResolutionPass` should keep passing the same pixel-space jitter to DLSS execute:

```cpp
Attribs.JitterX = FrameDesc.Jitter.x;
Attribs.JitterY = FrameDesc.Jitter.y;
```

## Relevant Files

- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSuperResolutionPass.cpp`
- `DiligentSamples/Samples/RTXPT/shaders/PathTracer/PathTracerHelpers.hlsli`
- `DiligentSamples/Samples/RTXPT/shaders/PathTracer/PathTracerBridge.hlsli`
- Regression check: `DiligentSamples/Samples/RTXPT/test_super_resolution_jitter.py`

## Reference Alignment

Original RTXPT-fork uses `m_view->GetPixelOffset()` directly as pixel-space camera jitter, passes it to `BridgeCamera`, and passes the same `jitterOffset` to Streamline/DLSS constants.

## TAA Follow-up

RTXPT realtime TAA had the same class of bug at a different boundary. DiligentFX `TemporalAntiAliasing` consumes normalized projection/NDC jitter (`2*x/width`, `-2*y/height`), but RTXPT/Donut `view.GetPixelOffset()`, NRD `cameraJitter`, and RTXPT path tracer camera rays all use pixel-space jitter in `[-0.5, 0.5]`.

Correct TAA behavior in `DiligentSamples/Samples/RTXPT`:

- `RTXPTTemporalAAPass::ComputePixelJitter()` returns pixel-space Halton jitter: `Halton(base, sample) - 0.5f`.
- `RTXPTSample::UpdateFrameConstants()` passes that pixel-space value into `MakePathTracerCameraData()` and `MakePathTracerViewData()`.
- `RTXPTTemporalAAPass::MakeCameraAttribs()` is the DiligentFX adapter boundary; it converts `PathTracerViewData::PixelOffset` to normalized TAA jitter before calling `TemporalAntiAliasing::GetJitteredProjMatrix()` and before assigning `HLSL::CameraAttribs::f2Jitter`.

Do not pass pixel-space `View.PixelOffset` directly into DiligentFX TAA camera attribs, and do not pass normalized DiligentFX jitter into RTXPT ray generation or NRD.

## Applicability

Applies to RTXPT realtime SuperResolution/DLSS integration and realtime TAA jitter integration. The safe rule is: camera rays, NRD, Donut-style `PixelOffset`, and DLSS jitter are pixel-space; DiligentFX TAA projection/camera attrib jitter is normalized and should be converted only at the TAA adapter boundary.