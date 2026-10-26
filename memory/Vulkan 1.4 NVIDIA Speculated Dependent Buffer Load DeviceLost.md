---
title: Vulkan 1.4 NVIDIA Speculated Dependent Buffer Load DeviceLost
type: note
permalink: diligentengine-hzqst/vulkan-1.4-nvidia-speculated-dependent-buffer-load-device-lost
tags:
- vulkan
- nvidia
- device-lost
- hlsl
- rtxpt
- robustness
---

# Vulkan 1.4 NVIDIA Speculated Dependent Buffer Load DeviceLost

## Trigger signal
- Vulkan backend only: `Failed to submit command buffer`, `VK_ERROR_DEVICE_LOST`, `All pending fences must now be complete!`; D3D12 passes the same shader.
- Compute shader with a guard `if (tid >= Count) return;` followed by a load whose result is used as an index into another buffer, e.g. `t_PTMaterialData[t_SubInstanceData[tid].MaterialID]`.
- Fails only when memory past the end of the first buffer holds non-zero data (for example adjacent Diligent sub-allocations); fresh, zeroed raw allocations hide it. `numthreads(1,1,1)` passes.

## Root cause / constraint
- NVIDIA driver (RTX 5060, 621.01, Oct 2026) with instance `apiVersion` 1.4 and `robustBufferAccess` disabled executes the dependent load chain for lanes that the guard rejects. A garbage index read past the buffer then faults (32-bit offset `index * stride` lands in unmapped memory).
- Guarded-out lanes perform no stores (verified), so the guard itself is honoured; only the loads are speculated.
- API 1.2 / 1.3 pass, 1.4 fails, 1.4 + `robustBufferAccess=TRUE` passes. Reproduced in a pure native Vulkan program with no Diligent runtime, so this is not a Diligent backend bug.
- Diligent contributes only indirectly: `GraphicsAPIVersion={0,0}` picks the highest instance version (`EngineFactoryVk.cpp` ~L710), it never enables `robustBufferAccess`, and its sub-allocator places buffers next to each other.

## Correct approach
- In the shader, clamp the index even after the guard, so a speculated load stays in bounds. Use the project idiom:
  `const uint safeIndex = min(index, max(1u, count) - 1u);`
  This idiom also appears in `LightFeedback.hlsl` RemapPastToCurrent and `LightProxyBuild.hlsl` ImportanceBooster.
- Fallback if needed: pin `EngineVkCreateInfo::GraphicsAPIVersion = {1,3}`, or enable robust buffer access (needs a Diligent API change).
- Do not treat validation/GPU-AV silence as proof: an old `build/rtxpt-v190/vk_layer_settings.txt` turns on GPU-AV safe mode for `.*emissive.*` shaders, so run repros from a clean directory such as `build/rtxpt-v190/run-clean/`.

## Verification method
- `build/rtxpt-v190/EmissiveRawVk.cpp` (native): `EMISSIVE_POISON=2` fills bytes past the 32-byte SubInstance range. `EMISSIVE_POISON_VALUE=records-0` poisons only MaterialID, and `EMISSIVE_BAD_VALUE` selects the poison value; avoid 0x10000000, because x144 wraps to offset 0 in 32 bits. `EMISSIVE_API_MINOR=N` and `EMISSIVE_ROBUST=1` switch the API version and enable robust buffer access. `EMISSIVE_BIG_OUTPUT=1` counts stores from guarded-out lanes.
- `build/rtxpt-v190/EmissiveRebind.cpp` (full Diligent, 64 resource recreations): `EMISSIVE_API_MINOR=N`. Use the DLLs from `build/x64/Release/DiligentSamples/Samples/RTXPT/Release/`; the core DLL directory lacks `DiligentNGX_64r.dll` (LoadLibrary error 126).
- After the clamp in `EmissiveTriangleBuild.hlsl`, `EmissiveRebind` passes on Vk default (1.4), Vk 1.4, Vk 1.3 and D3D12.

## Scope of applicability
- Applies to any RTXPT / Diligent compute shader on Vulkan where a value loaded by a guarded lane feeds another buffer address.
- Fixed with the clamp idiom on 2026-10-26 (DiligentSamples RTXPT `c3960911`, `7275924c`):
  - `EmissiveTriangleBuild.hlsl` main
  - `SkinnedVertexBuild.csh` (skin joints -> t_JointMatrices)
  - `LightProxyBuild.hlsl` ProcessClustersCS, ComputeProxyCountsCS and ScatterProxiesCS (trailing groups of the two-dimensional indirect dispatch)
- Verification: LightFeedbackTest (`tests/build_lighting_tests.bat`) passes on D3D12 and Vulkan after the LightProxyBuild change; SkinnedVertexBuild is compile-checked only.
- Lower risk (left unchanged): texture fetches, modulo-bounded indices (LightsBaker, SHaRC), pixel-derived addresses (post-process, denoising guides).

## Relations
- relates_to [[RTXPT Error Handling]]
- relates_to [[HLSL GetDimensions Cross-Device Caveat]]
