# Coordinate System Convention

## RTXPT Raygen Screen/NDC Convention

- `DiligentSamples/Samples/RTXPT` path tracing camera-ray helpers now follow RTXPT-fork screen-to-NDC convention.
- Screen pixel coordinates use a top-left origin: `pixel == uint2(0, 0)` is the top-left pixel, and sampling starts at `pixel + 0.5` plus jitter.
- Camera-ray NDC is D3D-style with `x` in `[-1, 1]` and top-to-bottom `y` in `[1, -1]`:

```hlsl
const float2 p   = (float2(pixel) + float2(0.5, 0.5) + jitter) / float2(data.ViewportSize);
const float2 ndc = float2(2.0, -2.0) * p + float2(-1.0, 1.0);
```

- `ComputeNonNormalizedRayDirPinhole` uses the jitter directly in screen pixel space, matching RTXPT-fork pinhole behavior.
- `ComputeRayThinlens` uses RTXPT-fork thin-lens jitter signs before NDC conversion:

```hlsl
const float2 p   = (float2(pixel) + float2(0.5, 0.5) + float2(-jitter.x, jitter.y)) / float2(data.ViewportSize);
const float2 ndc = float2(2.0, -2.0) * p + float2(-1.0, 1.0);
```

- CPU camera jitter should be stored in `PathTracerCameraData::Jitter` with RTXPT-fork `BridgeCamera` signs: `jitter * float2(1, -1)`. The value is in pixel space; realtime Super Resolution passes the upscaler-provided pixel-space jitter directly into the camera path so ray generation and DLSS execute use the same offset.
- Per-pixel shader jitter is added to `camera.Jitter` before ray generation; the ray helper then applies the pinhole/thin-lens signs described above.

## Related Files

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerBridge.hlsli`
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
- Reference: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerHelpers.hlsli`
- Reference: `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerShared.h`

## Verification Signal

- Targeted static check: `PathTracerHelpers.hlsli::ComputeNonNormalizedRayDirPinhole` should use `float2(2.0, -2.0) * p + float2(-1.0, 1.0)`.
- Targeted static check: `PathTracerHelpers.hlsli::ComputeRayThinlens` should compute `p` with `float2(-jitter.x, jitter.y)` and the same Y-flipped NDC expression.
- Targeted static check: `RTXPTSample.cpp::MakePathTracerCameraData` should store `Data.Jitter = Jitter * float2{1.0f, -1.0f}`.
- Build verification, when practical: `cmake --build build\x64\Debug --config Debug --target RTXPT` from the superproject root.

## RTXPT Post-Process Fullscreen Y Convention

- Final presentation blit and intermediate offscreen post-process passes should both preserve top-to-top texture orientation after the RTXPT-fork raygen screen convention is applied.
- `assets/shaders/RTXPTBlit.vsh` is presentation-only, but it must align with RTXPT-fork `External/Donut/shaders/rect_vs.hlsl` default blit behavior:

```hlsl
Output.UV  = float2(VertexId >> 1, VertexId & 1) * 2.0;
Output.Pos = float4(Output.UV.x * 2.0 - 1.0, 1.0 - Output.UV.y * 2.0, 0.0, 1.0);
```

- With this mapping, the top of the screen samples `UV.y == 0`, matching the top-left-origin UAV/image convention used by raygen and post-processing.
- Offscreen-to-offscreen passes, including tone mapping and luminance prepass, should continue using `assets/shaders/PostProcessing/RTXPTFullscreen.vsh`, which uses the same top-to-top fullscreen mapping.
- Do not reintroduce the old presentation flip form `float4(Output.UV * 2.0 - 1.0, ...)` in `RTXPTBlit.vsh`; after raygen NDC was aligned to RTXPT-fork, that form flips the final image vertically.
- Targeted static checks:
  - `RTXPTToneMappingPass.cpp` must reference `PostProcessing/RTXPTFullscreen.vsh`.
  - `RTXPTBloomPass.cpp` must reference `PostProcessing/RTXPTFullscreen.vsh`.
  - `RTXPTBlitPass.cpp` should reference `RTXPTBlit.vsh` for final swapchain presentation.
  - `RTXPTBlit.vsh` should use `1.0 - Output.UV.y * 2.0` for `SV_POSITION.y`.
  - Build verification: `cmake --build build\x64\Debug --config Debug --target RTXPT`.
