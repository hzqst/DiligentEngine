---
title: Vulkan Native Helpers Across Module Boundaries
type: note
permalink: diligentengine-hzqst/vulkan-native-helpers-across-module-boundaries
---

# Vulkan native helpers across module boundaries

## Trigger
An RR provider compiled into an executable or separate DLL receives a Vulkan device/context created by the graphics-engine DLL. Execution crashes at address zero in CommandBuffer::FlushBarriers when inputs have pending transitions.

## Root cause / constraint
Vulkan dispatch pointers belong to each linked module. Calling DeviceContextVkImpl::GetCommandBuffer inline from the provider executes native helpers against that module's uninitialized dispatch table. Tests with no pending barriers may miss this boundary.

## Correct approach
Use the owning context's public virtual API for command submission and native command-buffer access. The RR Vulkan provider calls IDeviceContext::Flush before transitions, then IDeviceContextVk::GetVkCommandBuffer for NGX. This also ends implicit rendering. Keep explicit render passes and active queries outside RR execution.

## Verification
RayReconstructionTest.EvaluateAndReadback exercises pending transitions and both static/shared RR objects. Vulkan E reproduces the access violation before the fix and passes afterwards. Real RTXPT BUILD/FILL -> prepare -> RR must also be tested; resource-only tests do not cover evaluation.

## Scope
Applies to native graphics integrations that cross an executable/DLL boundary and use module-local Vulkan dispatch pointers. Do not replace a virtual engine call with an internal inline helper merely because the C++ object layout is accessible.
