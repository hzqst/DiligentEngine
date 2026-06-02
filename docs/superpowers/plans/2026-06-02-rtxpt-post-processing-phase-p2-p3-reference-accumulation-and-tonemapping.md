# RTXPT Post-Processing Phase P2-P3 Reference Accumulation and Tone Mapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move reference accumulation and tone mapping out of raygen so the Diligent RTXPT sample follows RTXPT-fork's raw HDR `OutputColor -> AccumulatedRadiance -> ProcessedOutputColor -> LdrColor` display chain.

**Architecture:** Phase P2 introduces a Diligent-native `RTXPTAccumulationPass`, changes raygen to write one raw HDR sample per pixel, and removes the raygen-side accumulation UAV binding. Phase P3 introduces a Diligent-native `RTXPTToneMappingPass`, ports RTXPT-fork's tone-mapping parameter model and shader operators, makes exposure/UI state live, and switches final presentation to `LdrColor`.

**Tech Stack:** C++17, HLSL/DXC, Diligent Engine PSO/SRB APIs, Diligent texture SRV/UAV/RTV views, `IDeviceContext::GenerateMips`, staging readback buffers for auto exposure, ImGui, CMake sample registration, PowerShell + `rg` verification, reference source under `D:/RTXPT-fork/Rtxpt`.

---

## Current Baseline

- Driving spec: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`, Phase P2 and Phase P3.
- P0 mapping exists in `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`.
- P1 resource skeleton exists in the current tree: `OutputColor`, `AccumulatedRadiance`, `ProcessedOutputColor`, `LdrColor`, `LdrColorScratch`, and `RTXPTPostProcessPipeline` are already present.
- `PathTracerSample.rgen` still declares `u_AccumulationBuffer`, blends `pathRadiance`, calls `ToneMapACES`, multiplies by `g_Const.ptConsts.exposureScale`, and writes display-ready color to `u_Output`.
- `RTXPTRayTracingPass::Trace` still takes `pAccumulationUAV` and binds `u_AccumulationBuffer`.
- `RTXPTSample::Render` still runs `Trace -> optional RTXPTComputePass -> RTXPTBlitPass` and normal display still comes from `OutputColor`.
- `UpdateFrameConstants()` increments `m_AccumulationFrame`, writes `m_LastFrameConstants.ptConsts.sampleIndex/resetAccumulation`, then clears `m_ResetAccumulationPending` before `Render()`. P2 accumulation weight must therefore use `m_LastFrameConstants`, not `m_ResetAccumulationPending`.
- `RTXPTReferenceUIState` carries a small Phase 6 exposure subset; RTXPT-fork's full `ToneMappingParameters` is not represented yet.

## RTXPT-Fork Anchors

Use these anchors as the source-of-truth before editing:

- `D:/RTXPT-fork/Rtxpt/ProcessingPasses/AccumulationPass.{h,cpp,hlsl}` - reference accumulation pass, `blendFactor`, input resampling, history write suppression when weight is zero.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:1292-1297` - render-pass creation for accumulation and tone mapping.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2125` - tone-mapper `PreRender` before frame rendering.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2186-2203` - frame-tail order: accumulation, HDR post-process, tone mapping, LDR post-process.
- `D:/RTXPT-fork/Rtxpt/Sample.cpp:2771-2778` - reference accumulation weight computation.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMappingPasses.{h,cpp}` - parameter model, precomputed exposure/color transform, auto-exposure readback, pass-through semantics.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMapping_cb.h` - CPU/GPU tone-mapping constants layout and enum order.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ToneMapping.hlsl`, `ToneMapping.ps.hlsli`, `luminance_ps.hlsl` - shader entry points, luminance capture, and tone-map operators.
- `D:/RTXPT-fork/Rtxpt/ToneMapper/ColorUtils.h` - Rec.709/XYZ/CAT02 white-balance math.
- `D:/RTXPT-fork/Rtxpt/SampleUI.h:205-206` and `SampleUI.cpp:1546-1611` - live tone-mapping UI controls.
- `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli:165-168` - reference `CommitPixel` writes raw path radiance to `OutputColor`.

## Scope Boundaries

- P2 must remove raygen accumulation and raygen tone mapping together from the shader output path, but P2 may keep `exposureScale` fields until P3 removes the CPU/GPU layout entry.
- P2 must be testable on its own by confirming `ProcessedOutputColor` contains accumulated HDR. The final visually correct display is completed by P3.
- P3 must include all RTXPT-fork operators: `Linear`, `Reinhard`, `ReinhardModified`, `HejiHableAlu`, `HableUc2`, and `Aces`.
- P3 must make disabled tone mapping a pass-through from `ProcessedOutputColor` to `LdrColor`, while still allowing auto exposure state to update when enabled, matching RTXPT-fork behavior.
- P3 must default the operator to `HableUc2` to match RTXPT-fork `SampleUI.cpp` initialization; `Aces` remains selectable for comparison with the current Diligent raygen look.
- P3 must keep P4 bloom/LDR effects absent. No bloom, edge detection, shader debug, zoom, TAA, NRD, DLSS, or Streamline implementation lands in this plan.
- Do not copy Donut/NVRHI APIs or NVIDIA proprietary file headers into Diligent-owned files. Preserve names, constants, behavior contracts, and shader formulas in Diligent-native code.

## File Structure

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccumulationPass.hpp` - P2 compute pass interface, stats, and dispatch contract.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccumulationPass.cpp` - Diligent compute PSO/SRB, sampler, constants buffer upload, 8x8 dispatch.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTAccumulation.csh` - P2 accumulation shader.
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` - raw HDR raygen output only.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.{hpp,cpp}` - remove accumulation UAV binding and trace parameter.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}` - own/dispatch P2 accumulation and P3 tone mapping.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp` - P3 parameter/state model, constants, stats, and render contract.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp` - graphics PSOs, luminance texture/readback, exposure/color transform, `LdrColor` rendering.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMappingShared.h` - shared enum/constants layout.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.hlsl` - P3 pixel and capture entry points.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.ps.hlsli` - P3 operator functions and `applyToneMapping`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/Luminance.psh` - P3 luminance prepass.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}` - orchestration, UI, scene camera exposure import, diagnostics, final blit source.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp` and `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h` - remove raygen exposure scale while preserving layout stability.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}` - expose final-display helper or make sample explicitly blit `LdrColor`.
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt` - register new C++ and shader files.
- Modify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md` - link this plan from P2/P3.

---

### Task 0: Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`
- Verify: `D:/RTXPT-fork/Rtxpt`

- [ ] **Step 1: Confirm working-tree state**

Run:

```powershell
git status --short --branch
git -C DiligentSamples status --short --branch
```

Expected: branch lines are present. If any file is dirty, inspect it before editing and preserve user changes.

- [ ] **Step 2: Confirm P0/P1 prerequisites**

Run:

```powershell
rg -n "Phase 6 Post-Processing Pipeline Mapping|RTXPTAccumulationPass|RTXPTToneMappingPass|OutputColor.*raw HDR|ProcessedOutputColor|LdrColor" DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
rg -n "class RTXPTPostProcessPipeline|AccumulatedRadiance|ProcessedOutputColor|LdrColorScratch" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: mapping and P1 skeleton/resource names are present. If `RTXPTPostProcessPipeline` or the post-process targets are missing, finish P1 first.

- [ ] **Step 3: Confirm upstream anchors exist**

Run:

```powershell
Test-Path D:\RTXPT-fork\Rtxpt\ProcessingPasses\AccumulationPass.hlsl
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ToneMappingPasses.cpp
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ToneMapping_cb.h
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ToneMapping.ps.hlsli
Test-Path D:\RTXPT-fork\Rtxpt\ToneMapper\ColorUtils.h
```

Expected: every command prints `True`.

### Task 1: Port the Accumulation Shader

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTAccumulation.csh`
- Read: `D:/RTXPT-fork/Rtxpt/ProcessingPasses/AccumulationPass.hlsl`

- [ ] **Step 1: Create the post-processing shader directory**

Run:

```powershell
New-Item -ItemType Directory -Force DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing
```

Expected: directory exists. Existing files are not modified.

- [ ] **Step 2: Add `RTXPTAccumulation.csh`**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTAccumulation.csh` with this content:

```hlsl
#include "RTXPTCommon.fxh"

struct RTXPTAccumulationConstants
{
    float2 OutputSize;
    float2 InputSize;
    float2 InputTextureSizeInv;
    float2 PixelOffset;
    float  BlendFactor;
    float3 _Padding0;
};

cbuffer g_AccumulationConstants
{
    RTXPTAccumulationConstants g_Const;
};

RWTexture2D<float4> u_AccumulatedColor;
RWTexture2D<float4> u_OutputColor;
Texture2D<float4>   t_InputColor;
SamplerState        s_LinearSampler;

[numthreads(8, 8, 1)]
void main(uint2 GlobalIdx : SV_DispatchThreadID)
{
    if (any(GlobalIdx >= uint2(g_Const.OutputSize)))
        return;

    float4 CompositedColor;
    if (all(g_Const.InputSize == g_Const.OutputSize))
    {
        CompositedColor = t_InputColor[GlobalIdx];
    }
    else
    {
        const float2 InputPos = (float2(GlobalIdx) + 0.5.xx) * (g_Const.InputSize / g_Const.OutputSize) + g_Const.PixelOffset;
        const float2 InputUV  = InputPos * g_Const.InputTextureSizeInv;
        CompositedColor       = t_InputColor.SampleLevel(s_LinearSampler, InputUV, 0.0);
    }

    const float4 PreviousColor = u_AccumulatedColor[GlobalIdx];
    const float4 OutputColor   = g_Const.BlendFactor < 1.0 ?
        lerp(PreviousColor, CompositedColor, g_Const.BlendFactor) :
        CompositedColor;

    if (g_Const.BlendFactor > 0.0)
        u_AccumulatedColor[GlobalIdx] = OutputColor;

    u_OutputColor[GlobalIdx] = OutputColor;
}
```

Expected: shader preserves RTXPT-fork blend/resample semantics. The bound check uses `>=` to avoid dispatch fringe out-of-bounds while preserving intended behavior.

- [ ] **Step 3: Register shader in CMake**

In `DiligentSamples/Samples/RTXPT/CMakeLists.txt`, add the shader path to `set(SHADERS ...)`:

```cmake
    assets/shaders/PostProcessing/RTXPTAccumulation.csh
```

Expected: `rg -n "RTXPTAccumulation.csh" DiligentSamples/Samples/RTXPT/CMakeLists.txt` prints one match.

### Task 2: Add `RTXPTAccumulationPass`

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccumulationPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTAccumulationPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Add the header**

Create `RTXPTAccumulationPass.hpp` with this public shape:

```cpp
#pragma once

#include <string>

#include "BasicMath.hpp"
#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "Sampler.h"
#include "ShaderResourceBinding.h"
#include "TextureView.h"

namespace Diligent
{

struct RTXPTAccumulationPassStats
{
    bool        Ready                = false;
    bool        LastDispatchExecuted = false;
    Uint32      DispatchCount        = 0;
    std::string DisabledReason;
};

struct RTXPTAccumulationDispatch
{
    ITextureView* pInputColorSRV          = nullptr;
    ITextureView* pAccumulatedRadianceUAV = nullptr;
    ITextureView* pProcessedOutputUAV     = nullptr;
    Uint32        InputWidth              = 0;
    Uint32        InputHeight             = 0;
    Uint32        OutputWidth             = 0;
    Uint32        OutputHeight            = 0;
    float2        PixelOffset             = float2{0.0f, 0.0f};
    float         BlendFactor             = 1.0f;
};

class RTXPTAccumulationPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, bool ComputeSupported);
    bool Render(IDeviceContext* pContext, const RTXPTAccumulationDispatch& Dispatch);

    bool                               IsReady() const { return m_Stats.Ready; }
    const RTXPTAccumulationPassStats& GetStats() const { return m_Stats; }

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
    RefCntAutoPtr<IBuffer>                m_Constants;
    RefCntAutoPtr<ISampler>               m_LinearSampler;
    RTXPTAccumulationPassStats            m_Stats;
};

} // namespace Diligent
```

Expected: the pass has no dependency on `RTXPTSample`; all per-frame state is supplied through `RTXPTAccumulationDispatch`.

- [ ] **Step 2: Implement initialization**

In `RTXPTAccumulationPass.cpp`, implement:

```cpp
#include "RTXPTAccumulationPass.hpp"
#include "DebugUtilities.hpp"
#include "GraphicsTypesX.hpp"
#include "MapHelper.hpp"

namespace Diligent
{

namespace
{
struct RTXPTAccumulationConstants
{
    float2 OutputSize         = float2{0.0f, 0.0f};
    float2 InputSize          = float2{0.0f, 0.0f};
    float2 InputTextureSizeInv = float2{0.0f, 0.0f};
    float2 PixelOffset        = float2{0.0f, 0.0f};
    float  BlendFactor        = 1.0f;
    float3 _Padding0          = float3{0.0f, 0.0f, 0.0f};
};
static_assert(sizeof(RTXPTAccumulationConstants) == 48, "RTXPTAccumulationConstants must match RTXPTAccumulation.csh");
} // namespace

void RTXPTAccumulationPass::Reset()
{
    m_PSO.Release();
    m_SRB.Release();
    m_Constants.Release();
    m_LinearSampler.Release();
    m_Stats = {};
}

bool RTXPTAccumulationPass::Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, bool ComputeSupported)
{
    Reset();
    if (!ComputeSupported)
    {
        m_Stats.DisabledReason = "Compute shaders are not supported by this device";
        return false;
    }
    if (pDevice == nullptr || pEngineFactory == nullptr)
    {
        m_Stats.DisabledReason = "Accumulation pass requires a render device and engine factory";
        return false;
    }

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PostProcessing", &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.ShaderType            = SHADER_TYPE_COMPUTE;
    ShaderCI.Desc.Name                  = "RTXPT accumulation CS";
    ShaderCI.SourceLanguage             = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler             = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags               = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.FilePath                   = "PostProcessing/RTXPTAccumulation.csh";
    ShaderCI.EntryPoint                 = "main";
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pCS;
    pDevice->CreateShader(ShaderCI, &pCS);
    VERIFY(pCS, "Failed to create RTXPT accumulation shader");
    if (!pCS)
        return false;

    ComputePipelineStateCreateInfo PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT accumulation PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_COMPUTE;
    PSOCreateInfo.pCS                  = pCS;

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC;
    ResourceLayout
        .AddVariable(SHADER_TYPE_COMPUTE, "g_AccumulationConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "t_InputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_AccumulatedColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "u_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "s_LinearSampler", SHADER_RESOURCE_VARIABLE_TYPE_STATIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateComputePipelineState(PSOCreateInfo, &m_PSO);
    VERIFY(m_PSO, "Failed to create RTXPT accumulation PSO");
    if (!m_PSO)
        return false;

    BufferDesc ConstDesc;
    ConstDesc.Name           = "RTXPT accumulation constants";
    ConstDesc.Size           = sizeof(RTXPTAccumulationConstants);
    ConstDesc.Usage          = USAGE_DYNAMIC;
    ConstDesc.BindFlags      = BIND_UNIFORM_BUFFER;
    ConstDesc.CPUAccessFlags = CPU_ACCESS_WRITE;
    pDevice->CreateBuffer(ConstDesc, nullptr, &m_Constants);

    SamplerDesc LinearSampler;
    LinearSampler.Name      = "RTXPT accumulation linear sampler";
    LinearSampler.MinFilter = FILTER_TYPE_LINEAR;
    LinearSampler.MagFilter = FILTER_TYPE_LINEAR;
    LinearSampler.MipFilter = FILTER_TYPE_LINEAR;
    LinearSampler.AddressU  = TEXTURE_ADDRESS_CLAMP;
    LinearSampler.AddressV  = TEXTURE_ADDRESS_CLAMP;
    LinearSampler.AddressW  = TEXTURE_ADDRESS_CLAMP;
    pDevice->CreateSampler(LinearSampler, &m_LinearSampler);

    VERIFY(m_Constants && m_LinearSampler, "Failed to create RTXPT accumulation resources");
    if (!m_Constants || !m_LinearSampler)
        return false;

    m_PSO->GetStaticVariableByName(SHADER_TYPE_COMPUTE, "g_AccumulationConstants")->Set(m_Constants);
    m_PSO->GetStaticVariableByName(SHADER_TYPE_COMPUTE, "s_LinearSampler")->Set(m_LinearSampler);
    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    VERIFY(m_SRB, "Failed to create RTXPT accumulation SRB");
    if (!m_SRB)
        return false;

    m_Stats.Ready = true;
    return true;
}
```

Expected: initialization mirrors local `RTXPTComputePass` style while preserving RTXPT-fork resource names.

- [ ] **Step 3: Implement dispatch**

Add:

```cpp
bool RTXPTAccumulationPass::Render(IDeviceContext* pContext, const RTXPTAccumulationDispatch& Dispatch)
{
    m_Stats.LastDispatchExecuted = false;

    if (!IsReady() || pContext == nullptr ||
        Dispatch.pInputColorSRV == nullptr ||
        Dispatch.pAccumulatedRadianceUAV == nullptr ||
        Dispatch.pProcessedOutputUAV == nullptr ||
        Dispatch.InputWidth == 0 || Dispatch.InputHeight == 0 ||
        Dispatch.OutputWidth == 0 || Dispatch.OutputHeight == 0)
    {
        return false;
    }

    {
        MapHelper<RTXPTAccumulationConstants> Constants{pContext, m_Constants, MAP_WRITE, MAP_FLAG_DISCARD};
        Constants->OutputSize          = float2{static_cast<float>(Dispatch.OutputWidth), static_cast<float>(Dispatch.OutputHeight)};
        Constants->InputSize           = float2{static_cast<float>(Dispatch.InputWidth), static_cast<float>(Dispatch.InputHeight)};
        Constants->InputTextureSizeInv = float2{1.0f / static_cast<float>(Dispatch.InputWidth), 1.0f / static_cast<float>(Dispatch.InputHeight)};
        Constants->PixelOffset         = Dispatch.PixelOffset;
        Constants->BlendFactor         = Dispatch.BlendFactor;
    }

    m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "t_InputColor")->Set(Dispatch.pInputColorSRV);
    m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "u_AccumulatedColor")->Set(Dispatch.pAccumulatedRadianceUAV);
    m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "u_OutputColor")->Set(Dispatch.pProcessedOutputUAV);

    pContext->SetPipelineState(m_PSO);
    pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

    DispatchComputeAttribs DispatchAttribs;
    DispatchAttribs.ThreadGroupCountX = (Dispatch.OutputWidth + 7) / 8;
    DispatchAttribs.ThreadGroupCountY = (Dispatch.OutputHeight + 7) / 8;
    DispatchAttribs.ThreadGroupCountZ = 1;
    pContext->DispatchCompute(DispatchAttribs);

    m_Stats.LastDispatchExecuted = true;
    ++m_Stats.DispatchCount;
    return true;
}
```

Expected: P2 dispatch writes `AccumulatedRadiance` and `ProcessedOutputColor` exactly once per frame.

- [ ] **Step 4: Register files in CMake**

Add to `SOURCE`:

```cmake
    src/RTXPTAccumulationPass.cpp
```

Add to `INCLUDE`:

```cmake
    src/RTXPTAccumulationPass.hpp
```

Expected: CMake knows the new pass.

### Task 3: Change Raygen to Raw HDR Output

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`

- [ ] **Step 1: Remove accumulation UAV declarations**

Remove every declaration of:

```hlsl
VK_IMAGE_FORMAT("rgba32f") RWTexture2D<float4> u_AccumulationBuffer;
```

Expected: `rg -n "u_AccumulationBuffer" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen` prints no matches after this task.

- [ ] **Step 2: Update diagnostic branches**

In the screen-pattern and minimal-trace diagnostic branches, replace dual writes with a single raw output write:

```hlsl
u_Output[pixel] = float4(debugColor, 1.0);
```

and:

```hlsl
u_Output[pixel] = float4(payload.color, 1.0);
```

Expected: diagnostics no longer require an accumulation target.

- [ ] **Step 3: Remove `ToneMapACES` from raygen**

Delete the `ToneMapACES` helper and replace the final accumulation block with:

```hlsl
    u_Output[pixel] = float4(pathRadiance, 1.0);
```

Expected: raygen writes exactly one raw HDR sample. It does not read previous history and does not apply exposure or tone mapping.

### Task 4: Remove Ray-Tracing Accumulation Binding

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Change the `Trace` signature**

In `RTXPTRayTracingPass.hpp`, change:

```cpp
bool Trace(IDeviceContext* pContext,
           ITextureView*   pOutputUAV,
           ITextureView*   pAccumulationUAV,
           Uint32          Width,
           Uint32          Height);
```

to:

```cpp
bool Trace(IDeviceContext* pContext,
           ITextureView*   pOutputUAV,
           Uint32          Width,
           Uint32          Height);
```

Expected: ray tracing no longer advertises accumulation ownership.

- [ ] **Step 2: Remove the dynamic accumulation variable**

In `RTXPTRayTracingPass.cpp`, remove the resource-layout variable:

```cpp
.AddVariable(SHADER_TYPE_RAY_GEN, "u_AccumulationBuffer", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
```

Keep the `u_Output` variable.

Expected: raygen resource layout matches the updated shader.

- [ ] **Step 3: Update `Trace` binding logic**

In `RTXPTRayTracingPass.cpp`, remove `pAccumulationUAV`, `pAccumColorVar`, and `m_Stats.AccumulationBound`. The binding block becomes:

```cpp
if (pOutputUAV == nullptr || Width == 0 || Height == 0)
    return false;

IShaderResourceVariable* pOutputColorVar = m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "u_Output");
if (pOutputColorVar == nullptr)
{
    UNEXPECTED("Failed to find RTXPT output binding");
    return false;
}

pOutputColorVar->Set(pOutputUAV);
```

Expected: trace dispatch only binds `OutputColor`.

- [ ] **Step 4: Update the sample call site**

In `RTXPTSample::Render`, change:

```cpp
m_RayTracingPass.Trace(m_pImmediateContext,
                       m_RenderTargets.GetOutputColorUAV(),
                       m_RenderTargets.GetAccumulatedRadianceUAV(),
                       m_RenderTargets.GetWidth(),
                       m_RenderTargets.GetHeight());
```

to:

```cpp
m_RayTracingPass.Trace(m_pImmediateContext,
                       m_RenderTargets.GetOutputColorUAV(),
                       m_RenderTargets.GetWidth(),
                       m_RenderTargets.GetHeight());
```

Expected: ray tracing can run even before the accumulation pass checks its own resources.

### Task 5: Wire P2 Through `RTXPTPostProcessPipeline`

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add accumulation ownership**

In `RTXPTPostProcessPipeline.hpp`, include the pass and add methods:

```cpp
#include "RTXPTAccumulationPass.hpp"

bool RunAccumulation(IDeviceContext*              pContext,
                     const RTXPTRenderTargets&   RenderTargets,
                     Uint32                      SampleIndex,
                     bool                        ResetAccumulation);
```

Add a private member:

```cpp
RTXPTAccumulationPass m_AccumulationPass;
```

Expected: pipeline skeleton becomes the owner of P2 scheduling.

- [ ] **Step 2: Initialize and reset the pass**

In `Reset()`, call:

```cpp
m_AccumulationPass.Reset();
```

In `Initialize(...)`, after base checks:

```cpp
if (ComputeSupported)
    m_Stats.AccumulationStageReady = m_AccumulationPass.Initialize(pDevice, pEngineFactory, ComputeSupported);
```

Expected: unsupported compute keeps accumulation disabled with a visible reason.

- [ ] **Step 3: Implement `RunAccumulation`**

Use cached frame constants from `RTXPTSample` to compute the weight:

```cpp
bool RTXPTPostProcessPipeline::RunAccumulation(IDeviceContext*            pContext,
                                               const RTXPTRenderTargets& RenderTargets,
                                               Uint32                    SampleIndex,
                                               bool                      ResetAccumulation)
{
    if (!m_AccumulationPass.IsReady())
        return false;

    const Uint32 ClampedSampleIndex = std::max(SampleIndex, 1u);
    const float  BlendFactor        = ResetAccumulation ? 1.0f : 1.0f / static_cast<float>(ClampedSampleIndex);

    RTXPTAccumulationDispatch Dispatch;
    Dispatch.pInputColorSRV          = RenderTargets.GetOutputColorSRV();
    Dispatch.pAccumulatedRadianceUAV = RenderTargets.GetAccumulatedRadianceUAV();
    Dispatch.pProcessedOutputUAV     = RenderTargets.GetProcessedOutputColorUAV();
    Dispatch.InputWidth              = RenderTargets.GetWidth();
    Dispatch.InputHeight             = RenderTargets.GetHeight();
    Dispatch.OutputWidth             = RenderTargets.GetWidth();
    Dispatch.OutputHeight            = RenderTargets.GetHeight();
    Dispatch.PixelOffset             = float2{0.0f, 0.0f};
    Dispatch.BlendFactor             = BlendFactor;

    const bool Executed = m_AccumulationPass.Render(pContext, Dispatch);
    m_Stats.AccumulationStageReady = m_AccumulationPass.IsReady();
    return Executed;
}
```

Expected: first frame after reset uses weight `1.0`; subsequent frames use `1/N`, where `N == m_LastFrameConstants.ptConsts.sampleIndex`.

- [ ] **Step 4: Call accumulation after trace**

In `RTXPTSample::Render`, immediately after successful trace, call:

```cpp
const bool AccumulationExecuted =
    m_PostProcessPipeline.RunAccumulation(m_pImmediateContext,
                                          m_RenderTargets,
                                          m_LastFrameConstants.ptConsts.sampleIndex,
                                          m_LastFrameConstants.ptConsts.resetAccumulation != 0);
if (!AccumulationExecuted)
{
    ClearFallback(float4{0.0f, 0.2f, 1.0f, 1.0f});
    return;
}
```

Expected: P2 produces `ProcessedOutputColor` every rendered frame.

- [ ] **Step 5: Use `ProcessedOutputColor` for temporary P2 display**

Until P3 lands, set the normal blit source to `ProcessedOutputColor`:

```cpp
ITextureView* pDisplaySRV = ComputeExecuted ? m_RenderTargets.GetComputeColorSRV() :
                                             m_RenderTargets.GetProcessedOutputColorSRV();
```

Expected: P2 standalone display is HDR/untonemapped but proves accumulation runs. P3 replaces this source with `LdrColor`.

### Task 6: Verify P2

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`

- [ ] **Step 1: Scan for removed raygen ownership**

Run:

```powershell
rg -n "u_AccumulationBuffer|ToneMapACES|exposureScale" DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.*
```

Expected: no matches in raygen or `RTXPTRayTracingPass.*`. `exposureScale` may still appear in `RTXPTFrameConstants.*`, `PathTracerShared.h`, and `RTXPTSample.cpp` until P3.

- [ ] **Step 2: Build the sample**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If this build tree does not exist, configure according to the project guide before running the build.

- [ ] **Step 3: Commit P2**

Run inside `DiligentSamples`:

```powershell
git add Samples/RTXPT/src/RTXPTAccumulationPass.* Samples/RTXPT/src/RTXPTPostProcessPipeline.* Samples/RTXPT/src/RTXPTRayTracingPass.* Samples/RTXPT/src/RTXPTSample.cpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen Samples/RTXPT/assets/shaders/PostProcessing/RTXPTAccumulation.csh Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add reference accumulation pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: submodule commit records P2. If the top-level repository tracks the submodule pointer, update that pointer in the final integration commit.

### Task 7: Add Tone-Mapping Shared Types

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMappingShared.h`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Create tone-mapper shader directory**

Run:

```powershell
New-Item -ItemType Directory -Force DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper
```

Expected: directory exists.

- [ ] **Step 2: Add shared HLSL constants**

Create `ToneMappingShared.h`:

```hlsl
#ifndef __RTXPT_TONE_MAPPING_SHARED_H__
#define __RTXPT_TONE_MAPPING_SHARED_H__

#define RTXPT_TONEMAPPING_AUTOEXPOSURE_CPU 1
#define RTXPT_TONEMAPPING_EXPOSURE_KEY 0.042

enum RTXPTToneMapperOperator
{
    RTXPTToneMapperOperator_Linear = 0,
    RTXPTToneMapperOperator_Reinhard = 1,
    RTXPTToneMapperOperator_ReinhardModified = 2,
    RTXPTToneMapperOperator_HejiHableAlu = 3,
    RTXPTToneMapperOperator_HableUc2 = 4,
    RTXPTToneMapperOperator_Aces = 5
};

struct RTXPTToneMappingConstants
{
    float  WhiteScale;
    float  WhiteMaxLuminance;
    uint   ToneMapOperator;
    uint   Clamped;
    uint   AutoExposure;
    float  AvgLuminance;
    float  AutoExposureLumValueMin;
    float  AutoExposureLumValueMax;
    float3x4 ColorTransform;
    uint   Enabled;
    uint   _Padding0;
    uint   _Padding1;
    uint   _Padding2;
};

#endif
```

Expected: enum order and constants match RTXPT-fork `ToneMapping_cb.h`.

- [ ] **Step 3: Add C++ parameter model**

In `RTXPTToneMappingPass.hpp`, define:

```cpp
enum class RTXPTExposureMode : Uint32
{
    AperturePriority = 0,
    ShutterPriority  = 1,
};

enum class RTXPTToneMapperOperator : Uint32
{
    Linear             = 0,
    Reinhard           = 1,
    ReinhardModified   = 2,
    HejiHableAlu       = 3,
    HableUc2           = 4,
    Aces               = 5,
};

struct RTXPTToneMappingParameters
{
    RTXPTExposureMode       ExposureMode         = RTXPTExposureMode::AperturePriority;
    RTXPTToneMapperOperator ToneMapOperator      = RTXPTToneMapperOperator::HableUc2;
    bool                    AutoExposure         = false;
    float                   ExposureCompensation = 0.0f;
    float                   ExposureValue        = 0.0f;
    float                   FilmSpeed            = 100.0f;
    float                   FNumber              = 1.0f;
    float                   Shutter              = 1.0f;
    bool                    WhiteBalance         = false;
    float                   WhitePoint           = 6500.0f;
    float                   WhiteMaxLuminance    = 1.0f;
    float                   WhiteScale           = 5.1f;
    bool                    Clamped              = true;
    float                   ExposureValueMin     = -16.0f;
    float                   ExposureValueMax     = 16.0f;
};
```

Expected: names are Diligent-local but preserve RTXPT-fork parameter meaning and defaults, with operator default aligned to RTXPT-fork UI initialization.

- [ ] **Step 4: Register new shader files in CMake**

Add to `SHADERS`:

```cmake
    assets/shaders/PostProcessing/ToneMapper/ToneMappingShared.h
    assets/shaders/PostProcessing/ToneMapper/ToneMapping.hlsl
    assets/shaders/PostProcessing/ToneMapper/ToneMapping.ps.hlsli
    assets/shaders/PostProcessing/ToneMapper/Luminance.psh
```

Expected: all P3 shader files are listed.

### Task 8: Add Tone-Mapping Shaders

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.ps.hlsli`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/ToneMapping.hlsl`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper/Luminance.psh`

- [ ] **Step 1: Add `ToneMapping.ps.hlsli`**

Implement these functions with the same constants and operator order as RTXPT-fork:

```hlsl
#ifndef __RTXPT_TONE_MAPPING_PS_HLSLI__
#define __RTXPT_TONE_MAPPING_PS_HLSLI__

#include "ToneMappingShared.h"

SamplerState      s_LuminanceSampler;
SamplerState      s_ColorSampler;
Texture2D<float4> t_Color;
Texture2D<float>  t_Luminance;

cbuffer g_ToneMappingConstants
{
    RTXPTToneMappingConstants g_Params;
};

float CalcLuminance(float3 Color)
{
    return dot(Color, float3(0.299, 0.587, 0.114));
}

float3 ToneMapLinear(float3 Color) { return Color; }

float3 ToneMapReinhard(float3 Color)
{
    const float Luminance = max(CalcLuminance(Color), 1.0e-6);
    return Color * ((Luminance / (Luminance + 1.0)) / Luminance);
}

float3 ToneMapReinhardModified(float3 Color)
{
    const float Luminance = max(CalcLuminance(Color), 1.0e-6);
    const float White2    = max(g_Params.WhiteMaxLuminance * g_Params.WhiteMaxLuminance, 1.0e-6);
    const float Mapped    = (Luminance * (1.0 + Luminance / White2)) / (1.0 + Luminance);
    return Color * (Mapped / Luminance);
}

float3 ToneMapHejiHableAlu(float3 Color)
{
    Color = max(0.0.xxx, Color - 0.004.xxx);
    Color = (Color * (6.2 * Color + 0.5)) / (Color * (6.2 * Color + 1.7) + 0.06);
    return pow(Color, 2.2.xxx);
}

float3 ApplyUc2Curve(float3 Color)
{
    const float A = 0.22;
    const float B = 0.30;
    const float C = 0.10;
    const float D = 0.20;
    const float E = 0.01;
    const float F = 0.30;
    return ((Color * (A * Color + C * B) + D * E) / (Color * (A * Color + B) + D * F)) - (E / F);
}

float3 ToneMapHableUc2(float3 Color)
{
    Color = ApplyUc2Curve(2.0 * Color);
    const float WhiteScale = 1.0 / ApplyUc2Curve(g_Params.WhiteScale.xxx).x;
    return Color * WhiteScale;
}

float3 ToneMapAces(float3 Color)
{
    Color *= 0.6;
    const float A = 2.51;
    const float B = 0.03;
    const float C = 2.43;
    const float D = 0.59;
    const float E = 0.14;
    return saturate((Color * (A * Color + B)) / (Color * (C * Color + D) + E));
}

float3 ApplyOperator(float3 Color)
{
    switch (g_Params.ToneMapOperator)
    {
        case RTXPTToneMapperOperator_Linear:           return ToneMapLinear(Color);
        case RTXPTToneMapperOperator_Reinhard:         return ToneMapReinhard(Color);
        case RTXPTToneMapperOperator_ReinhardModified: return ToneMapReinhardModified(Color);
        case RTXPTToneMapperOperator_HejiHableAlu:     return ToneMapHejiHableAlu(Color);
        case RTXPTToneMapperOperator_HableUc2:         return ToneMapHableUc2(Color);
        case RTXPTToneMapperOperator_Aces:             return ToneMapAces(Color);
        default:                                       return Color;
    }
}

float4 ApplyToneMapping(float2 UV)
{
    const float4 SourceColor = t_Color.Sample(s_ColorSampler, UV);
    float3 FinalColor = SourceColor.rgb;

    if (g_Params.AutoExposure != 0)
    {
        const float AvgLuminance = max(g_Params.AvgLuminance, 1.0e-6);
        FinalColor *= clamp(RTXPT_TONEMAPPING_EXPOSURE_KEY / AvgLuminance,
                            g_Params.AutoExposureLumValueMin,
                            g_Params.AutoExposureLumValueMax);
    }

    if (g_Params.Enabled != 0)
    {
        FinalColor = mul(FinalColor, (float3x3)g_Params.ColorTransform);
        FinalColor = ApplyOperator(FinalColor);
        if (g_Params.Clamped != 0)
            FinalColor = saturate(FinalColor);
    }

    return float4(FinalColor, SourceColor.a);
}

#endif
```

Expected: shader operators are complete and ordered for CPU enum compatibility.

- [ ] **Step 2: Add `ToneMapping.hlsl`**

Create:

```hlsl
#pragma pack_matrix(row_major)

#include "ToneMappingShared.h"
#include "ToneMapping.ps.hlsli"

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

struct PSOutput
{
    float4 Color : SV_TARGET;
};

void main_ps(in PSInput Input,
             out PSOutput Output)
{
    Output.Color = ApplyToneMapping(Input.UV);
}

RWBuffer<float>   u_CaptureTarget;
Texture2D<float>  t_CaptureSource;

[numthreads(1, 1, 1)]
void capture_cs(uint3 DispatchThreadID : SV_DispatchThreadID)
{
    uint Width;
    uint Height;
    uint MipLevels;
    t_CaptureSource.GetDimensions(0, Width, Height, MipLevels);
    u_CaptureTarget[0] = t_CaptureSource.Load(int3(0, 0, MipLevels - 1));
}
```

Expected: P3 exposes both pixel tone mapping and compute luminance capture.

- [ ] **Step 3: Add `Luminance.psh`**

Create:

```hlsl
Texture2D<float4> t_Color;
SamplerState      s_ColorSampler;

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

struct PSOutput
{
    float4 Color : SV_TARGET;
};

float CalcLuminance(float3 Color)
{
    return dot(Color, float3(0.299, 0.587, 0.114));
}

void main(in PSInput Input,
          out PSOutput Output)
{
    const float4 Color = t_Color.Sample(s_ColorSampler, Input.UV);
    const float  LogLuminance = log2(max(0.0001, CalcLuminance(Color.rgb)));
    Output.Color = float4(LogLuminance, 0.0, 0.0, 1.0);
}
```

Expected: luminance prepass matches RTXPT-fork's log-luminance contract.

### Task 9: Implement `RTXPTToneMappingPass`

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTToneMappingPass.cpp`
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Add pass stats and render request**

In `RTXPTToneMappingPass.hpp`, add:

```cpp
struct RTXPTToneMappingPassStats
{
    bool        Ready              = false;
    bool        AutoExposureReady  = false;
    bool        LastRenderExecuted = false;
    Uint32      RenderCount        = 0;
    float       LastAvgLuminance   = 0.0f;
    std::string DisabledReason;
};

struct RTXPTToneMappingRenderAttribs
{
    ITextureView*                     pSourceSRV = nullptr;
    ITextureView*                     pLdrRTV    = nullptr;
    Uint32                            Width      = 0;
    Uint32                            Height     = 0;
    bool                              Enabled    = true;
    const RTXPTToneMappingParameters* pParams    = nullptr;
};
```

Expected: render inputs are explicit and independent of `RTXPTSample`.

- [ ] **Step 2: Add private resources**

In the same header, declare:

```cpp
class RTXPTToneMappingPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice,
                    IEngineFactory* pEngineFactory,
                    TEXTURE_FORMAT LdrFormat,
                    bool ComputeSupported);
    bool ResizeResources(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT SourceFormat);
    bool Render(IDeviceContext* pContext, const RTXPTToneMappingRenderAttribs& Attribs);

    bool                             IsReady() const { return m_Stats.Ready; }
    const RTXPTToneMappingPassStats& GetStats() const { return m_Stats; }

private:
    bool CreateSamplers(IRenderDevice* pDevice);
    bool CreateLuminanceResources(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT SourceFormat);
    bool UpdateToneMappingConstants(IDeviceContext* pContext, const RTXPTToneMappingParameters& Params, bool Enabled);
    void PollReadback(IDeviceContext* pContext);

    RefCntAutoPtr<IPipelineState>         m_LuminancePSO;
    RefCntAutoPtr<IPipelineState>         m_ToneMapPSO;
    RefCntAutoPtr<IPipelineState>         m_CapturePSO;
    RefCntAutoPtr<IShaderResourceBinding> m_LuminanceSRB;
    RefCntAutoPtr<IShaderResourceBinding> m_ToneMapSRB;
    RefCntAutoPtr<IShaderResourceBinding> m_CaptureSRB;
    RefCntAutoPtr<IBuffer>                m_ToneMappingCB;
    RefCntAutoPtr<IBuffer>                m_AvgLuminanceGPU;
    RefCntAutoPtr<IBufferView>            m_AvgLuminanceUAV;
    RefCntAutoPtr<IBuffer>                m_AvgLuminanceReadback[3];
    RefCntAutoPtr<ITexture>               m_LuminanceTexture;
    RefCntAutoPtr<ITextureView>           m_LuminanceRTV;
    RefCntAutoPtr<ITextureView>           m_LuminanceSRV;
    RefCntAutoPtr<ISampler>               m_LinearSampler;
    RefCntAutoPtr<ISampler>               m_PointSampler;
    RTXPTToneMappingPassStats             m_Stats;
    Uint32                                m_Width = 0;
    Uint32                                m_Height = 0;
    int                                   m_LastReadbackWritten = -1;
};
```

Expected: pass owns the resources needed to match RTXPT-fork's auto-exposure path.

- [ ] **Step 3: Implement exposure/color transform helpers**

In `RTXPTToneMappingPass.cpp`, implement Diligent-local helpers using RTXPT-fork `ColorUtils.h` math:

```cpp
float ClampFloat(float Value, float MinValue, float MaxValue)
{
    return std::max(MinValue, std::min(MaxValue, Value));
}

void UpdateExposureValue(RTXPTToneMappingParameters& Params)
{
    const float ShutterMin = 0.001f;
    const float ShutterMax = 10000.0f;
    const float FNumberMin = 0.1f;
    const float FNumberMax = 100.0f;

    const float EVMin = std::log2(ShutterMin * FNumberMin * FNumberMin);
    const float EVMax = std::log2(ShutterMax * FNumberMax * FNumberMax);
    Params.ExposureValue = ClampFloat(Params.ExposureValue, EVMin, EVMax);

    if (Params.ExposureMode == RTXPTExposureMode::AperturePriority)
    {
        Params.Shutter = std::pow(2.0f, Params.ExposureValue) / (Params.FNumber * Params.FNumber);
        Params.Shutter = ClampFloat(Params.Shutter, ShutterMin, ShutterMax);
    }
    else
    {
        Params.FNumber = std::sqrt(std::pow(2.0f, Params.ExposureValue) / Params.Shutter);
        Params.FNumber = ClampFloat(Params.FNumber, FNumberMin, FNumberMax);
    }
}
```

Expected: exposure mode behavior matches RTXPT-fork.

- [ ] **Step 4: Implement PSO setup**

Follow `RTXPTBlitPass` style:

```cpp
ShaderCI.Desc.ShaderType = SHADER_TYPE_VERTEX;
ShaderCI.FilePath        = "RTXPTBlit.vsh";
ShaderCI.EntryPoint      = "main";
```

For tone mapping:

```cpp
ShaderCI.Desc.ShaderType = SHADER_TYPE_PIXEL;
ShaderCI.FilePath        = "PostProcessing/ToneMapper/ToneMapping.hlsl";
ShaderCI.EntryPoint      = "main_ps";
```

For luminance:

```cpp
ShaderCI.Desc.ShaderType = SHADER_TYPE_PIXEL;
ShaderCI.FilePath        = "PostProcessing/ToneMapper/Luminance.psh";
ShaderCI.EntryPoint      = "main";
```

For capture:

```cpp
ShaderCI.Desc.ShaderType = SHADER_TYPE_COMPUTE;
ShaderCI.FilePath        = "PostProcessing/ToneMapper/ToneMapping.hlsl";
ShaderCI.EntryPoint      = "capture_cs";
```

Expected: pixel passes use full-screen triangle draw; capture uses `DispatchCompute({1, 1, 1})`.

- [ ] **Step 5: Implement luminance resources**

Create a power-of-two luminance texture with render-target and shader-resource usage:

```cpp
TextureDesc Desc;
Desc.Name      = "RTXPT luminance texture";
Desc.Type      = RESOURCE_DIM_TEX_2D;
Desc.Width     = 1u << static_cast<Uint32>(std::floor(std::log2(static_cast<float>(Width))));
Desc.Height    = 1u << static_cast<Uint32>(std::floor(std::log2(static_cast<float>(Height))));
Desc.MipLevels = ComputeMipLevelsCount(Desc.Width, Desc.Height);
Desc.Format    = SourceFormat == TEX_FORMAT_RGBA32_FLOAT ? TEX_FORMAT_R32_FLOAT : TEX_FORMAT_R16_FLOAT;
Desc.BindFlags = BIND_SHADER_RESOURCE | BIND_RENDER_TARGET;
```

Expected: the texture can be rendered into, sampled, and mip-generated with `IDeviceContext::GenerateMips(m_LuminanceSRV)`.

- [ ] **Step 6: Implement readback buffers**

Create a GPU buffer and three staging buffers:

```cpp
BufferDesc GPUDesc;
GPUDesc.Name              = "RTXPT average luminance GPU";
GPUDesc.Size              = sizeof(float);
GPUDesc.BindFlags         = BIND_UNORDERED_ACCESS;
GPUDesc.Usage             = USAGE_DEFAULT;
GPUDesc.Mode              = BUFFER_MODE_FORMATTED;

BufferViewDesc UAVDesc;
UAVDesc.Name                 = "RTXPT average luminance UAV";
UAVDesc.ViewType             = BUFFER_VIEW_UNORDERED_ACCESS;
UAVDesc.Format.ValueType     = VT_FLOAT32;
UAVDesc.Format.NumComponents = 1;

BufferDesc ReadbackDesc;
ReadbackDesc.Name           = "RTXPT average luminance readback";
ReadbackDesc.Size           = sizeof(float);
ReadbackDesc.Usage          = USAGE_STAGING;
ReadbackDesc.BindFlags      = BIND_NONE;
ReadbackDesc.CPUAccessFlags = CPU_ACCESS_READ;
```

After `CreateBuffer(GPUDesc, ...)`, create `m_AvgLuminanceUAV` with:

```cpp
m_AvgLuminanceGPU->CreateView(UAVDesc, &m_AvgLuminanceUAV);
```

Bind `m_AvgLuminanceUAV` to `u_CaptureTarget`.

Expected: readback is lagged by three frames and never blocks after initial warm-up.

- [ ] **Step 7: Implement render order**

`Render()` must:

1. Upload tone-mapping constants after `UpdateExposureValue`.
2. If auto exposure is enabled, render log luminance to mip 0, call `GenerateMips(m_LuminanceSRV)`, dispatch `capture_cs`, copy the GPU float into the next readback buffer, then poll the oldest buffer.
3. Set `pLdrRTV` as the render target.
4. Bind `t_Color`, `t_Luminance`, samplers, constants.
5. Draw `DrawAttribs{3, DRAW_FLAG_VERIFY_ALL}`.

Expected: `ProcessedOutputColor` is sampled and `LdrColor` is the only color attachment written by this pass.

- [ ] **Step 8: Register C++ files**

Add to `SOURCE`:

```cmake
    src/RTXPTToneMappingPass.cpp
```

Add to `INCLUDE`:

```cmake
    src/RTXPTToneMappingPass.hpp
```

Expected: CMake includes the tone-mapping pass.

### Task 10: Wire P3 Through Pipeline and Presentation

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPipeline.{hpp,cpp}`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.{hpp,cpp}`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.{hpp,cpp}`

- [ ] **Step 1: Add tone-mapper ownership**

In `RTXPTPostProcessPipeline.hpp`, include `RTXPTToneMappingPass.hpp`, add:

```cpp
bool RunToneMapping(IDeviceContext*                    pContext,
                    const RTXPTRenderTargets&         RenderTargets,
                    const RTXPTToneMappingParameters& Params,
                    bool                              Enabled);
```

and private member:

```cpp
RefCntAutoPtr<IRenderDevice> m_Device;
RTXPTToneMappingPass m_ToneMappingPass;
```

Expected: P3 scheduling is owned by the post-process pipeline, matching the P0 ownership map.

- [ ] **Step 2: Initialize tone mapping**

In `Initialize(...)`, call:

```cpp
m_Device = pDevice;
m_Stats.ToneMappingStageReady =
    m_ToneMappingPass.Initialize(pDevice, pEngineFactory, TEX_FORMAT_RGBA8_UNORM, ComputeSupported);
```

Expected: tone mapping can initialize even if compute is unavailable; only auto exposure/capture is compute-dependent.

- [ ] **Step 3: Implement `RunToneMapping`**

Use:

```cpp
if (!m_ToneMappingPass.ResizeResources(m_Device, RenderTargets.GetWidth(), RenderTargets.GetHeight(), RenderTargets.GetProcessedOutputColorFormat()))
    return false;

RTXPTToneMappingRenderAttribs Attribs;
Attribs.pSourceSRV = RenderTargets.GetProcessedOutputColorSRV();
Attribs.pLdrRTV    = RenderTargets.GetLdrColorRTV();
Attribs.Width      = RenderTargets.GetWidth();
Attribs.Height     = RenderTargets.GetHeight();
Attribs.Enabled    = Enabled;
Attribs.pParams    = &Params;
return m_ToneMappingPass.Render(pContext, Attribs);
```

Expected: `RTXPTRenderTargets` must expose `GetLdrColorRTV()` before this compiles.

- [ ] **Step 4: Add render-target RTV accessors**

In `RTXPTRenderTargets.hpp`, add:

```cpp
ITextureView* GetProcessedOutputColorRTV() const;
ITextureView* GetLdrColorRTV() const;
```

In `RTXPTRenderTargets.cpp`, implement:

```cpp
ITextureView* RTXPTRenderTargets::GetProcessedOutputColorRTV() const
{
    return m_ProcessedOutputColor ? m_ProcessedOutputColor->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetLdrColorRTV() const
{
    return m_LdrColor ? m_LdrColor->GetDefaultView(TEXTURE_VIEW_RENDER_TARGET) : nullptr;
}
```

Expected: tone mapping can render to `LdrColor`.

- [ ] **Step 5: Call tone mapping after accumulation**

In `RTXPTSample::Render`, after accumulation and before final blit:

```cpp
const bool ToneMappingExecuted =
    m_PostProcessPipeline.RunToneMapping(m_pImmediateContext,
                                         m_RenderTargets,
                                         m_ReferenceUI.ToneMapping,
                                         m_ReferenceUI.EnableToneMapping);
if (!ToneMappingExecuted)
{
    ClearFallback(float4{0.0f, 0.8f, 0.3f, 1.0f});
    return;
}
```

Expected: P3 writes `LdrColor` every frame.

- [ ] **Step 6: Blit `LdrColor`**

Replace normal display source selection with:

```cpp
ITextureView* pDisplaySRV = ComputeExecuted ? m_RenderTargets.GetComputeColorSRV() :
                                             m_RenderTargets.GetLdrColorSRV();
```

Expected: `LdrColor` is the normal swapchain source after P3.

### Task 11: Promote Tone-Mapping UI State

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Replace scalar exposure subset with parameter struct**

In `RTXPTReferenceUIState`, replace:

```cpp
bool  ToneMappingAutoExposure;
float ToneMappingExposureCompensation;
float ToneMappingExposureValue;
float ToneMappingExposureValueMin;
float ToneMappingExposureValueMax;
```

with:

```cpp
RTXPTToneMappingParameters ToneMapping;
```

Expected: UI state can represent all RTXPT-fork controls.

- [ ] **Step 2: Update defaults**

Replace `ResetToneMappingSettings` with:

```cpp
void ResetToneMappingSettings(RTXPTReferenceUIState& UI)
{
    UI.EnableToneMapping = true;
    UI.ToneMapping = {};
    UI.ToneMapping.ToneMapOperator = RTXPTToneMapperOperator::HableUc2;
}
```

Expected: default operator matches RTXPT-fork UI initialization.

- [ ] **Step 3: Import scene camera exposure**

In `ApplySceneCamera`, update:

```cpp
m_ReferenceUI.ToneMapping.AutoExposure         = pCamera->EnableAutoExposure.value_or(false);
m_ReferenceUI.ToneMapping.ExposureCompensation = pCamera->ExposureCompensation.value_or(0.0f);
m_ReferenceUI.ToneMapping.ExposureValue        = pCamera->ExposureValue.value_or(0.0f);
m_ReferenceUI.ToneMapping.ExposureValueMin     = pCamera->ExposureValueMin.value_or(-16.0f);
m_ReferenceUI.ToneMapping.ExposureValueMax     = pCamera->ExposureValueMax.value_or(16.0f);
```

Expected: scene camera metadata feeds post-process tone mapping rather than raygen constants.

- [ ] **Step 4: Make quick tone-map checkbox live**

Remove the disabled wrapper around `"Enable tone mapping"`:

```cpp
ImGui::Checkbox("Enable tone mapping", &m_ReferenceUI.EnableToneMapping);
```

Expected: reference panel quick toggle matches RTXPT-fork reference panel.

- [ ] **Step 5: Add global Tone Mapping section**

In the post-processing UI block, add controls with RTXPT-fork labels:

```cpp
static const char* Operators[] = {"Linear", "Reinhard", "Reinhard Modified", "Heji Hable ALU", "Hable UC2", "Aces"};
int Operator = static_cast<int>(m_ReferenceUI.ToneMapping.ToneMapOperator);
if (ImGui::Combo("Operator", &Operator, Operators, _countof(Operators)))
    m_ReferenceUI.ToneMapping.ToneMapOperator = static_cast<RTXPTToneMapperOperator>(std::clamp(Operator, 0, 5));

ImGui::Checkbox("Auto Exposure", &m_ReferenceUI.ToneMapping.AutoExposure);
if (m_ReferenceUI.ToneMapping.AutoExposure)
{
    ImGui::InputFloat("Auto Exposure Min", &m_ReferenceUI.ToneMapping.ExposureValueMin);
    ImGui::InputFloat("Auto Exposure Max", &m_ReferenceUI.ToneMapping.ExposureValueMax);
}

static const char* ExposureModes[] = {"Aperture Priority", "Shutter Priority"};
int ExposureMode = static_cast<int>(m_ReferenceUI.ToneMapping.ExposureMode);
if (ImGui::Combo("Exposure Mode", &ExposureMode, ExposureModes, _countof(ExposureModes)))
    m_ReferenceUI.ToneMapping.ExposureMode = static_cast<RTXPTExposureMode>(std::clamp(ExposureMode, 0, 1));

ImGui::InputFloat("Exposure Compensation", &m_ReferenceUI.ToneMapping.ExposureCompensation);
ImGui::InputFloat("Exposure Value", &m_ReferenceUI.ToneMapping.ExposureValue);
ImGui::InputFloat("Film Speed", &m_ReferenceUI.ToneMapping.FilmSpeed);
ImGui::InputFloat("fNumber", &m_ReferenceUI.ToneMapping.FNumber);
ImGui::InputFloat("Shutter", &m_ReferenceUI.ToneMapping.Shutter);
ImGui::Checkbox("Enable White Balance", &m_ReferenceUI.ToneMapping.WhiteBalance);
ImGui::InputFloat("White Point", &m_ReferenceUI.ToneMapping.WhitePoint);
ImGui::InputFloat("White Max Luminance", &m_ReferenceUI.ToneMapping.WhiteMaxLuminance);
ImGui::InputFloat("White Scale", &m_ReferenceUI.ToneMapping.WhiteScale);
ImGui::Checkbox("Enable Clamp", &m_ReferenceUI.ToneMapping.Clamped);
```

Expected: all RTXPT-fork P3 tone-mapping controls are present and live.

### Task 12: Remove Raygen Exposure Layout Debt

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTFrameConstants.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Remove CPU exposure scale calculation**

Delete `ComputeToneMappingExposureScale` and remove:

```cpp
m_LastFrameConstants.ptConsts.exposureScale = ComputeToneMappingExposureScale(m_ReferenceUI);
```

Expected: raygen constants no longer carry display exposure.

- [ ] **Step 2: Preserve `PathTracerConstants` size**

In `RTXPTFrameConstants.hpp`, replace:

```cpp
float  exposureScale;
```

with:

```cpp
float  _paddingP3_0 = 0.0f;
```

In `PathTracerShared.h`, replace the matching field with:

```hlsl
float _paddingP3_0;
```

Expected: `sizeof(PathTracerConstants) == 80` and `sizeof(SampleConstants) == 480` remain true.

- [ ] **Step 3: Verify stale exposure references**

Run:

```powershell
rg -n "ToneMapACES|exposureScale|ComputeToneMappingExposureScale" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer
```

Expected: no matches.

### Task 13: Final Verification

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: `docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md`

- [ ] **Step 1: Source-contract scan**

Run:

```powershell
rg -n "u_AccumulationBuffer|ToneMapACES|exposureScale|ComputeToneMappingExposureScale" DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders
rg -n "RTXPTAccumulationPass|RTXPTToneMappingPass|RTXPTAccumulation.csh|ToneMappingShared.h|Luminance.psh" DiligentSamples/Samples/RTXPT/CMakeLists.txt DiligentSamples/Samples/RTXPT/src DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing
```

Expected: first command prints no stale raygen accumulation/tone-map ownership. Second command prints CMake, C++, and shader registrations.

- [ ] **Step 2: Build**

Run:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: build succeeds. If the configured build tree is unavailable, report that and run at least source scans.

- [ ] **Step 3: Runtime smoke**

Run the RTXPT sample on D3D12 and Vulkan from the configured sample launcher.

Expected:

- Reference mode launches without fallback clear colors.
- `Accumulated samples` increases.
- Toggling `Enable tone mapping` changes between tone-mapped and pass-through output.
- Operator combo changes image response.
- Manual `Exposure Compensation` changes brightness.
- Auto Exposure updates without a persistent stall after warm-up.
- Final blit draw count increases and normal source is `LdrColor`.

- [ ] **Step 4: Visual parity capture**

Compare against RTXPT-fork on the same fixed scene:

```text
RTXPT-fork: OutputColor raw HDR -> AccumulatedRadiance -> ProcessedOutputColor -> LdrColor
Diligent:   OutputColor raw HDR -> AccumulatedRadiance -> ProcessedOutputColor -> LdrColor
```

Expected: with matching operator, exposure settings, white balance, and clamp, the Diligent LDR image tracks RTXPT-fork within expected backend/rendering differences.

- [ ] **Step 5: Commit P3**

Run inside `DiligentSamples`:

```powershell
git add Samples/RTXPT/src/RTXPTToneMappingPass.* Samples/RTXPT/src/RTXPTPostProcessPipeline.* Samples/RTXPT/src/RTXPTSample.* Samples/RTXPT/src/RTXPTRenderTargets.* Samples/RTXPT/src/RTXPTFrameConstants.hpp Samples/RTXPT/assets/shaders/PathTracer/PathTracerShared.h Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper Samples/RTXPT/CMakeLists.txt
git commit -m "feat(rtxpt): add tone mapping pass" -m "Co-Authored-By: GPT 5.5"
```

Then commit top-level spec/plan changes and submodule pointer if needed:

```powershell
git add docs/superpowers/specs/2026-06-01-rtxpt-post-processing-pipeline-port-design.md docs/superpowers/plans/2026-06-02-rtxpt-post-processing-phase-p2-p3-reference-accumulation-and-tonemapping.md DiligentSamples
git commit -m "docs(rtxpt): plan post-processing accumulation and tone mapping" -m "Co-Authored-By: GPT 5.5"
```

Expected: commits separate P2 implementation, P3 implementation, and top-level planning/submodule pointer updates.

## Self-Review Checklist

- P2 source parity: `RTXPTAccumulationPass` preserves blend factor, resampling hook, 8x8 dispatch, accumulated HDR write, and `ProcessedOutputColor` write.
- P2 ownership: raygen writes raw HDR only; `RTXPTRayTracingPass` binds only `u_Output`.
- P2 timing: accumulation weight uses `m_LastFrameConstants.ptConsts.sampleIndex/resetAccumulation`, not the already-cleared `m_ResetAccumulationPending`.
- P3 source parity: all RTXPT-fork operators, exposure mode, auto exposure bounds, white balance, white luminance/scale, clamp, and pass-through semantics are represented.
- P3 presentation: normal swapchain source is `LdrColor`, not `OutputColor` or `ProcessedOutputColor`.
- Layout hygiene: CPU/GPU shared structs keep static assertions passing after `exposureScale` is removed.
- Capability gating: compute-disabled devices report accumulation/auto-exposure disabled reasons; tone mapping can still render manual exposure when graphics is available.
- Mapping continuity: `RTXPT_FORK_MAPPING.md` remains consistent with the implemented owner files.
