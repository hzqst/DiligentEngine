---
title: RTXPT GPU TLAS — Issue 28 Validation Limits
type: note
permalink: diligentengine-hzqst/rtxpt-gpu-tlas-issue-28-validation-limits
tags:
- rtxpt
- tlas
- validation
- issue-28
---

# RTXPT GPU TLAS — Issue 28 Validation Limits

## Overview
2026-10-26: Issue #28 的 GPU 实例准备/TLAS 更新已实现，但不能宣称全部验收通过。固定上游参考 RTXPT-fork `9503910839f013f7ee0a2590a1131d84160ca712`。本记录用于后续维护时避免丢失已知限制。

## Responsibilities
- 记录 GPU/CPU TLAS 路径的支持范围、验证证据及未通过项。
- 区分 AS 验证成功与 Vulkan NGX/DLSS-RR 生命周期失败，避免以退出码 0 判定成功。

## Involved Files & Symbols
- `DiligentCore/Graphics/GraphicsEngine/interface/DeviceContext.h`: `BuildTLASAttribs::InstanceBufferIsPrebuilt`，默认 false；API 版本 256022。
- `DiligentSamples/Samples/RTXPT/src/GPUTLAS.hpp`: GPU 实例准备及模板资源。
- `DiligentSamples/Samples/RTXPT/shaders/TLASCull.hlsl`: 原生 64-byte 实例描述符写入。
- `DiligentSamples/Samples/RTXPT/tests/VisibilityRayTest.cpp`: 差分测试、计时和 DXGI 进程显存测量。
- `DiligentSamples/Samples/RTXPT/tests/README.md`: GPU TLAS validation (issue #28)，完整测试结果与复现命令。
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`: GPU instance preparation and TLAS，保留差异及公共接口契约。

## Architecture
每次场景/拓扑重建先由 CPU BuildTLAS 初始化原生描述符，再复制 64 bytes/instance 的 GPU 模板。GPU 使用既有变换缓冲及 16 bytes/instance 参数表写入变换、custom ID 和 mask，保留模板中的 BLAS 地址、native flags 和两槽 SBT contribution。Core 跳过描述符上传，但保留 CPU 实例元数据、BLAS 生命周期/同步及 SBT 管理。模板必须在 BLAS 地址、flags、SBT 布局或拓扑改变时重建。`--gpu_tlas false` 选择 CPU 回退。

## Dependencies
- D3D12/Vulkan；公共 API 升为 256022，使用方必须重新编译，不能混用旧 DLL。
- 不新增第三方依赖；不移植 OMM、BLAS pooling、视锥剔除或实例压缩。

## Notes
### 验证结果
- Release RTXPT 和两后端构建通过；Core 改动及新 helper 通过 clang-format 10；两个子模块 git diff --check 通过。
- RTX 5060 / Windows：D3D12 uint32、D3D12 uint16 + HitObject/reorder、Vulkan uint16/uint32 的七场景逐射线差分通过，AS 验证无错误。
- 覆盖变换/旋转/非均匀缩放、mask 与实时 mask 更新、实例增删、空 TLAS、BLAS refit、材质/命中距离/阴影比较，以及原生描述符字节比较。
- D3D12 GPU 336 帧生命周期序列（serial、重载、奇数尺寸、Reference/RR 切换）为 failures=0 / engine errors=0。
- 两后端 CPU 回退动态场景短跑退出 0，无引擎错误；Vulkan GPU Reference 32 帧也无引擎错误。

### 未通过项与正确处理
- 触发信号：Vulkan RR 生命周期序列在帧 0 和 96 报 `VUID-vkCmdDraw-None-09600`，`nv.ngx.dlssd.resource` 图像预期 GENERAL、实际 UNDEFINED。
- 结果：failures=2 / engine errors=4。CPU TLAS 路径执行相同序列也复现相同四条错误，说明不是 GPU 实例准备特有问题；NGX 布局错误的更深根因及修复不在本次完成范围。
- 正确做法：保留此未通过项；不能只看进程退出码 0，也不能宣称 Vulkan RR 重载通过。后续排查需保持同一序列的 CPU/GPU 对照。

### 性能与支持范围限制
- 三实例非均匀变换 refit、validation 开启、单次测量：D3D12 CPU/GPU 路径提交 12.0/63.0 us，GPU 区间 10.912/18.688 us；Vulkan 提交 17.8/86.0 us，GPU 区间 10.240/13.184 us。小场景更慢，不宣称加速；大型场景扩展性、稳定吞吐、其他 GPU 和图像质量基准尚未测量。
- 额外显式 GPU 缓冲为 80 bytes/instance。代表样本 DXGI 进程 local usage：D3D12 55,558,144 -> 55,566,336 bytes；Vulkan 122,142,720 -> 122,142,720 bytes。分配器页、延迟释放和驱动粒度影响读数，差值不能当作精确缓冲成本。
- 一次性 CPU 模板初始化包含额外初始 TLAS build；稳态更新不回读 GPU 数据。动画沿用 serial 路径，此次不宣称 async overlap 验收。
- 引擎测试支持真实零实例 TLAS；样例 AS owner 对空图清理资源并保持未构建。未新增 UI 加载/渲染无模型 scene JSON 的支持。
- Vulkan validation layer 1.4.350 低于 headers 1.4.363 的警告，D3D12 可选 DirectSR/PIX 警告，均保留在原始日志。

## Callers
- `RTXPTSample::RebuildSceneDependentResources` → `RTXPTAccelerationStructures::BuildScene`。
- 动画更新 → BLAS refit（若存在）→ `RTXPTAccelerationStructures::UpdateTLAS` → GPU prepare → Core BuildTLAS。
