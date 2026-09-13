---
title: RTXPT Vulkan Structured Vertex Layout
type: note
permalink: diligentengine-hzqst/rtxpt-vulkan-structured-vertex-layout
---

## Trigger
D3D12 RTXPT surfaces render correctly, but Vulkan produces geometry-fixed black triangles and corrupted normals/UVs before RR evaluation.

## Root cause and constraints
The CPU GLTF vertex stream stores position/normal/UV at offsets 0/12/24 with stride 32. Default DXC SPIR-V storage-buffer layout of float3/float3/float2 emits offsets 0/16/32 and ArrayStride 48. BLAS reads the correct CPU stride while hit shaders read different vertices. D3D12 SRV descriptors carry their structured stride; Vulkan descriptors do not override SPIR-V ArrayStride.

## Correct approach
RTXPT uses PackedGeometryVertexData with two float4 members, offsets 0/16 and stride 32. Shared PackGeometryVertex/UnpackGeometryVertex helpers preserve the CPU layout. All hit, emissive-triangle and skinning reads/writes use this storage type; GeometryVertexData remains an unpacked shading value. Do not change global DXC layout flags or require scalar-block-layout to solve this sample-local ABI problem.

## Verification
Offline closest-hit SPIR-V changed from offsets 0/16/32, stride 48 to offsets 0/16, stride 32. GeometryVertexLayoutTest.hpp runs the production skinning shader on two distinct interleaved CPU vertices and checks GPU-readback positions, normals and UVs after a known translation. Old shader failed on Vulkan (expected normal component 1, got 0.970142); fixed shader and all 14 RR input cases passed on Vulkan and D3D12. Vulkan RR Native E real-scene black triangles disappeared on RTX 5060, 2026-09-14.

## Scope and remaining limits
Applies to structured vertex buffers shared across CPU/D3D/Vulkan, especially adjacent float3 members. This fix does not solve the separately reproduced Vulkan reference-mode VK_ERROR_DEVICE_LOST, nor the known NGX Vulkan preset-F validation failure.
