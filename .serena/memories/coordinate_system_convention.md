# Coordinate System Convention

## RTXPT Raygen Screen Y Convention

- In `DiligentSamples/Samples/RTXPT`, the current Diligent RTXPT output/blit chain expects the production raygen screen mapping to keep the pre-R7 inverse-view-projection convention: `ndc = uv * 2.0 - 1.0`.
- This means `pixel.y == 0` (top row in `DispatchRaysIndex().xy`) maps to `ndc.y == -1`, not `+1`.
- Do not directly port RTXPT-fork/Donut `ComputeRayThinlens` screen mapping `float2(2, -2) * p + float2(-1, 1)` into this Diligent shader path unless the presentation/blit/output convention is changed at the same time.
- If raygen maps top pixels to `ndc.y == +1` while the output chain remains Diligent-style, the final image appears vertically mirrored, and camera mouse pitch feels vertically inverted because the rendered view is flipped relative to the unchanged `FirstPersonCamera` input logic.
- For `PathTracerHelpers.hlsli::ComputeNonNormalizedRayDirPinhole`, keep the Diligent convention:

```hlsl
const float2 p   = (float2(pixel) + float2(0.5, 0.5) + jitter) / float2(data.ViewportSize);
const float2 ndc = p * 2.0 - 1.0;
```

## Related Files

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerHelpers.hlsli`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.psh`
- `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.vsh`
- `DiligentSamples/SampleBase/src/FirstPersonCamera.cpp`

## Verification Signal

- A regression symptom is: final RTXPT render is vertically mirrored and mouse pitch feels inverted after camera-ray changes.
- The targeted static check is to confirm `ComputeNonNormalizedRayDirPinhole` uses `p * 2.0 - 1.0` rather than the Donut-style Y-flipped NDC expression.
- Build verification should include `cmake --build build\x64\Debug --config Debug --target RTXPT` from the superproject root.

## RTXPT Post-Process Fullscreen Y Convention

- Final presentation blit and intermediate offscreen post-process passes must use different fullscreen VS conventions.
- `assets/shaders/RTXPTBlit.vsh` is presentation-only. Its `UV`/`SV_POSITION` pairing intentionally performs the final vertical orientation conversion when copying to the swapchain.
- Offscreen-to-offscreen passes, including tone mapping and luminance prepass, must not use `RTXPTBlit.vsh`; doing so flips the image before the final blit and changes the total flip count.
- Use `assets/shaders/PostProcessing/RTXPTFullscreen.vsh` for offscreen fullscreen graphics passes. It keeps render-target top mapped to `UV.y == 0`:

```hlsl
Output.UV  = float2(VertexId >> 1, VertexId & 1) * 2.0;
Output.Pos = float4(Output.UV.x * 2.0 - 1.0, 1.0 - Output.UV.y * 2.0, 0.0, 1.0);
```

- Regression symptom: after inserting tone mapping or another graphics post-process pass, final RTXPT image appears vertically mirrored and camera pitch feels inverted even though `FirstPersonCamera` and raygen NDC mapping are unchanged.
- Targeted static checks:
  - `RTXPTToneMappingPass.cpp` must reference `PostProcessing/RTXPTFullscreen.vsh`, not `RTXPTBlit.vsh`.
  - `RTXPTBlitPass.cpp` should still reference `RTXPTBlit.vsh` for final swapchain presentation.
  - Build verification: `cmake --build build\x64\Debug --config Debug --target RTXPT`.
