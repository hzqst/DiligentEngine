# RTXPT Phase 4 Draw And Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the first runnable RTXPT draw/dispatch layer: an output UAV, a minimal ray tracing PSO/SBT with `IDeviceContext::TraceRays`, a fullscreen blit path, and reusable compute dispatch infrastructure.

**Architecture:** Keep `RTXPTSample` as the lifecycle owner and add focused pass classes for render targets, ray tracing, compute, and blitting. Phase 4 writes a visible `OutputColor` texture through a minimal primary-ray pass, optionally runs one debug compute post-process, then blits to the swapchain; stable planes, RTXDI, light feedback, and denoising guide chains are represented as structured TODOs and disabled pass slots until the Phase 5 shader layers exist.

**Tech Stack:** C++17, DiligentSamples `SampleBase`, DiligentCore ray tracing PSO/SBT APIs, DiligentCore compute PSO APIs, HLSL 6.3 ray tracing shaders compiled by DXC, Diligent render target and shader resource binding APIs, Dear ImGui.

---

## Baseline

Current Phase 3 state in `DiligentSamples/Samples/RTXPT`:

- `RTXPTSample` loads the default scene, frame constants, material buffer, light buffer, BLAS, and TLAS.
- `RTXPTAccelerationStructures::GetTLAS()` exposes the built TLAS.
- `RTXPTSample::Render()` still clears the swapchain and contains `TODO(RTXPT-Port Phase 4): add TraceRays path and RT PSO/SBT.`
- There is no `shaders` folder under `DiligentSamples/Samples/RTXPT` yet.
- `SampleBase` requests device features as optional by default, so RTXPT can keep launching on devices without standalone ray tracing shaders.

This plan assumes the Phase 3 submodule changes are already committed and the top-level repository starts clean.

---

## Scope

This plan implements the Phase 4 runnable milestone:

- Add RTXPT shader assets for minimal ray tracing, debug compute, and blitting.
- Add `OutputColor` and `ComputeColor` textures with UAV/SRV views.
- Add a minimal RT PSO with one raygen, one miss shader, one closest-hit shader, and one SBT hit group bound for the whole TLAS.
- Call `IDeviceContext::TraceRays` when ray tracing, standalone RT shaders, TLAS, and output UAV are available.
- Add a reusable compute pass wrapper and execute a minimal fullscreen texture-processing compute shader.
- Blit the selected SRV to the swapchain.
- Keep clear fallback behavior when Phase 4 resources cannot be created or device support is missing.
- Add structured Phase 4 TODOs for stable-plane, RTXDI, light feedback, and denoising guide chains.

This plan intentionally does not:

- Port the full RTXPT reference path tracer shader core.
- Add material-specialized hit groups, alpha-test any-hit, or full SBT permutation generation.
- Implement stable plane data structures or RTXDI reservoir algorithms.
- Integrate NRD, DLSS, Streamline, NVAPI, SER, OMM, or post-processing beyond the minimal debug compute pass.
- Add automated build or runtime execution to the plan creation task; build/runtime commands are listed for execution only when explicitly requested.

---

## File Structure

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  - Add Phase 4 source/header files and shader assets to `add_sample_app`.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  - Add pass objects, render target ownership, and helper declarations.
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  - Initialize Phase 4 passes, resize output textures, replace the RT clear path with TraceRays/compute/blit, and show pass status in UI.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
  - Own `OutputColor` and `ComputeColor` textures and expose UAV/SRV views.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
  - Create resize-safe `TEX_FORMAT_RGBA8_UNORM` UAV/SRV render targets.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
  - Own minimal RT PSO, SRB, SBT, and pass stats.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
  - Compile RT shaders, bind frame constants/TLAS/output UAV, update SBT, and call `TraceRays`.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.hpp`
  - Own a reusable compute PSO/SRB wrapper and dispatch stats.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`
  - Compile a compute shader, bind input/output textures, and dispatch 8x8 thread groups.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.hpp`
  - Own fullscreen graphics PSO/SRB for swapchain presentation.
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp`
  - Draw a fullscreen triangle from the selected SRV into the swapchain RTV.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh`
  - Share frame constants and ray payload structs between HLSL shaders.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
  - Generate primary camera rays and write `OutputColor`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`
  - Write a sky gradient on miss.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`
  - Write barycentric/depth debug color on hit.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTDebugCompute.csh`
  - Copy and lightly mark `OutputColor` into `ComputeColor`.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.vsh`
  - Fullscreen triangle vertex shader.
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.psh`
  - Load source texture texels into the swapchain render target.

---

### Task 0: Phase 3 Baseline Preflight

**Files:**
- Verify: top-level repository
- Verify: `DiligentSamples`

- [ ] **Step 1: Confirm top-level state**

Run from `d:\DiligentEngine-hzqst`:

```powershell
git status --short --branch
```

Expected:

```text
## RTXPT...origin/RTXPT
```

If other unrelated files appear, leave them unstaged and continue only if they do not overlap `docs/superpowers/plans` or `DiligentSamples/Samples/RTXPT`.

- [ ] **Step 2: Confirm DiligentSamples Phase 3 state**

Run:

```powershell
git -C DiligentSamples status --short --branch
rg -n "TODO\(RTXPT-Port Phase 4\): add TraceRays path and RT PSO/SBT" DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
```

Expected:

```text
DiligentSamples branch is clean
DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp contains the Phase 4 TraceRays TODO
```

- [ ] **Step 3: Confirm Phase 4 has no existing shader folder**

Run:

```powershell
Test-Path DiligentSamples/Samples/RTXPT/assets/shaders
```

Expected before Task 1:

```text
False
```

If it returns `True`, inspect that folder before creating files and preserve unrelated user work.

---

### Task 1: Add Phase 4 Shader Assets And Shader CMake Entries

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTDebugCompute.csh`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.vsh`
- Create: `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.psh`

- [ ] **Step 1: Create shared shader declarations**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTCommon.fxh`:

```hlsl
#ifndef RTXPT_COMMON_FXH
#define RTXPT_COMMON_FXH

struct RTXPTFrameConstants
{
    float4x4 ViewProj;
    float4x4 ViewProjInv;
    float4   CameraPosition_Time;
    float4   ViewportSize_FrameIdx;
};

ConstantBuffer<RTXPTFrameConstants> g_FrameConstants;

struct RTXPTPrimaryPayload
{
    float4 ColorDepth;
};

#endif
```

- [ ] **Step 2: Create minimal ray tracing shaders**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rgen`:

```hlsl
#include "RTXPTCommon.fxh"

RaytracingAccelerationStructure g_TLAS;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4> g_OutputColor;

[shader("raygeneration")]
void main()
{
    const uint2 Pixel      = DispatchRaysIndex().xy;
    const uint2 Dimensions = DispatchRaysDimensions().xy;
    const float2 UV        = (float2(Pixel) + 0.5) / float2(Dimensions);
    const float2 NDC       = UV * 2.0 - 1.0;

    const float4 WorldPos = mul(float4(NDC, 1.0, 1.0), g_FrameConstants.ViewProjInv);
    const float3 Origin   = g_FrameConstants.CameraPosition_Time.xyz;
    const float3 RayDir   = normalize(WorldPos.xyz / WorldPos.w - Origin);

    RayDesc Ray;
    Ray.Origin    = Origin;
    Ray.Direction = RayDir;
    Ray.TMin      = 0.001;
    Ray.TMax      = 10000.0;

    RTXPTPrimaryPayload Payload;
    Payload.ColorDepth = float4(0.02, 0.03, 0.04, 1.0);

    TraceRay(g_TLAS,
             RAY_FLAG_FORCE_OPAQUE,
             0xFF,
             0,
             1,
             0,
             Ray,
             Payload);

    g_OutputColor[Pixel] = float4(saturate(Payload.ColorDepth.rgb), 1.0);
}
```

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rmiss`:

```hlsl
#include "RTXPTCommon.fxh"

[shader("miss")]
void main(inout RTXPTPrimaryPayload Payload)
{
    const float T = saturate(WorldRayDirection().y * 0.5 + 0.5);
    const float3 Horizon = float3(0.48, 0.58, 0.68);
    const float3 Zenith  = float3(0.05, 0.08, 0.14);
    Payload.ColorDepth   = float4(lerp(Horizon, Zenith, T), 1.0);
}
```

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTMinimal.rchit`:

```hlsl
#include "RTXPTCommon.fxh"

[shader("closesthit")]
void main(inout RTXPTPrimaryPayload Payload,
          in BuiltInTriangleIntersectionAttributes Attributes)
{
    const float3 Barycentrics = float3(1.0 - Attributes.barycentrics.x - Attributes.barycentrics.y,
                                       Attributes.barycentrics.x,
                                       Attributes.barycentrics.y);
    const float InstanceTint = frac(float(InstanceID() * 17 + PrimitiveIndex() * 3) * 0.037);
    const float Depth       = saturate(RayTCurrent() / 150.0);
    const float3 Color      = lerp(Barycentrics, float3(Depth, 1.0 - Depth, InstanceTint), 0.35);
    Payload.ColorDepth      = float4(Color, Depth);
}
```

- [ ] **Step 3: Create debug compute and blit shaders**

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTDebugCompute.csh`:

```hlsl
#include "RTXPTCommon.fxh"

Texture2D<float4> g_InputColor;
VK_IMAGE_FORMAT("rgba8") RWTexture2D<float4> g_OutputColor;

[numthreads(8, 8, 1)]
void main(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    const uint2 Pixel = DispatchThreadId.xy;
    const uint Width  = (uint)g_FrameConstants.ViewportSize_FrameIdx.x;
    const uint Height = (uint)g_FrameConstants.ViewportSize_FrameIdx.y;

    if (Pixel.x >= Width || Pixel.y >= Height)
        return;

    const float2 UV       = (float2(Pixel) + 0.5) / float2(max(Width, 1u), max(Height, 1u));
    const float4 Input    = g_InputColor.Load(int3(Pixel, 0));
    const float Vignette  = smoothstep(0.95, 0.20, length(UV - 0.5));
    const float PhaseMark = frac(g_FrameConstants.ViewportSize_FrameIdx.w * 0.0078125);
    const float3 Marked   = saturate(Input.rgb * (0.92 + 0.08 * Vignette) + float3(0.02, 0.015, 0.01) * PhaseMark);

    g_OutputColor[Pixel] = float4(Marked, 1.0);
}
```

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.vsh`:

```hlsl
struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

void main(in uint VertexId : SV_VertexID,
          out PSInput Output)
{
    Output.UV  = float2(VertexId >> 1, VertexId & 1) * 2.0;
    Output.Pos = float4(Output.UV * 2.0 - 1.0, 0.0, 1.0);
}
```

Create `DiligentSamples/Samples/RTXPT/assets/shaders/RTXPTBlit.psh`:

```hlsl
Texture2D<float4> g_Texture;

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 UV  : TEX_COORD;
};

struct PSOutput
{
    float4 Color : SV_TARGET;
};

void main(in PSInput Input,
          out PSOutput Output)
{
    float2 Dimensions;
    g_Texture.GetDimensions(Dimensions.x, Dimensions.y);
    const int3 Texel = int3(Input.UV * Dimensions, 0);
    Output.Color = g_Texture.Load(Texel);
}
```

- [ ] **Step 4: Register Phase 4 shader files in CMake**

Modify `DiligentSamples/Samples/RTXPT/CMakeLists.txt` by adding this shader list after the `INCLUDE` list:

```cmake
set(SHADERS
    assets/shaders/RTXPTCommon.fxh
    assets/shaders/RTXPTMinimal.rgen
    assets/shaders/RTXPTMinimal.rmiss
    assets/shaders/RTXPTMinimal.rchit
    assets/shaders/RTXPTDebugCompute.csh
    assets/shaders/RTXPTBlit.vsh
    assets/shaders/RTXPTBlit.psh
)
```

Then update `add_sample_app` to include:

```cmake
    SHADERS
        ${SHADERS}
```

Expected `add_sample_app` shape:

```cmake
add_sample_app(RTXPT
    IDE_FOLDER
        DiligentSamples/Samples
    SOURCES
        ${SOURCE}
    INCLUDES
        ${INCLUDE}
    SHADERS
        ${SHADERS}
    DXC_REQUIRED
        YES
)
```

Do not add the Phase 4 C++ source/header files to `SOURCE` or `INCLUDE` in this task. They are registered in Task 5 after all files exist.

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit shader and CMake setup inside DiligentSamples**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/assets/shaders
git -C DiligentSamples commit -m "feat(rtxpt): add phase 4 shader assets" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the CMake shader registration and the seven new shader files.

---

### Task 2: Add Output Render Targets And Swapchain Blit Pass

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp`

- [ ] **Step 1: Create render target owner header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.hpp`:

```cpp
#pragma once

#include <string>

#include "RenderDevice.h"
#include "RefCntAutoPtr.hpp"
#include "Texture.h"
#include "TextureView.h"

namespace Diligent
{

class RTXPTRenderTargets
{
public:
    void Reset();
    bool Resize(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT Format, bool CreateComputeOutput);

    bool IsValid() const { return m_OutputColor != nullptr; }

    ITextureView* GetOutputColorUAV() const;
    ITextureView* GetOutputColorSRV() const;
    ITextureView* GetComputeColorUAV() const;
    ITextureView* GetComputeColorSRV() const;
    ITextureView* GetDisplaySRV(bool UseComputeOutput) const;

    Uint32             GetWidth() const { return m_Width; }
    Uint32             GetHeight() const { return m_Height; }
    TEXTURE_FORMAT     GetFormat() const { return m_Format; }
    const std::string& GetLastError() const { return m_LastError; }

private:
    bool CreateTarget(IRenderDevice* pDevice, const char* Name, RefCntAutoPtr<ITexture>& Target);

    RefCntAutoPtr<ITexture> m_OutputColor;
    RefCntAutoPtr<ITexture> m_ComputeColor;
    Uint32                  m_Width  = 0;
    Uint32                  m_Height = 0;
    TEXTURE_FORMAT          m_Format = TEX_FORMAT_UNKNOWN;
    std::string             m_LastError;
};

} // namespace Diligent
```

- [ ] **Step 2: Create render target owner implementation**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTRenderTargets.cpp`:

```cpp
#include "RTXPTRenderTargets.hpp"

namespace Diligent
{

void RTXPTRenderTargets::Reset()
{
    m_OutputColor.Release();
    m_ComputeColor.Release();
    m_Width = 0;
    m_Height = 0;
    m_Format = TEX_FORMAT_UNKNOWN;
    m_LastError.clear();
}

bool RTXPTRenderTargets::CreateTarget(IRenderDevice* pDevice, const char* Name, RefCntAutoPtr<ITexture>& Target)
{
    TextureDesc Desc;
    Desc.Name      = Name;
    Desc.Type      = RESOURCE_DIM_TEX_2D;
    Desc.Width     = m_Width;
    Desc.Height    = m_Height;
    Desc.Format    = m_Format;
    Desc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;

    Target.Release();
    pDevice->CreateTexture(Desc, nullptr, &Target);
    if (!Target)
    {
        m_LastError = std::string{"Failed to create "} + Name;
        return false;
    }

    if (Target->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) == nullptr ||
        Target->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) == nullptr)
    {
        m_LastError = std::string{Name} + " is missing SRV or UAV";
        Target.Release();
        return false;
    }

    return true;
}

bool RTXPTRenderTargets::Resize(IRenderDevice* pDevice, Uint32 Width, Uint32 Height, TEXTURE_FORMAT Format, bool CreateComputeOutput)
{
    if (Width == 0 || Height == 0)
        return false;

    const bool HasRequestedTargets =
        m_OutputColor != nullptr &&
        (!CreateComputeOutput || m_ComputeColor != nullptr) &&
        (CreateComputeOutput || m_ComputeColor == nullptr);

    if (HasRequestedTargets && m_Width == Width && m_Height == Height && m_Format == Format)
        return true;

    Reset();
    m_Width  = Width;
    m_Height = Height;
    m_Format = Format;

    if (!CreateTarget(pDevice, "RTXPT OutputColor", m_OutputColor))
        return false;

    if (CreateComputeOutput && !CreateTarget(pDevice, "RTXPT ComputeColor", m_ComputeColor))
        return false;

    return true;
}

ITextureView* RTXPTRenderTargets::GetOutputColorUAV() const
{
    return m_OutputColor ? m_OutputColor->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetOutputColorSRV() const
{
    return m_OutputColor ? m_OutputColor->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetComputeColorUAV() const
{
    return m_ComputeColor ? m_ComputeColor->GetDefaultView(TEXTURE_VIEW_UNORDERED_ACCESS) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetComputeColorSRV() const
{
    return m_ComputeColor ? m_ComputeColor->GetDefaultView(TEXTURE_VIEW_SHADER_RESOURCE) : nullptr;
}

ITextureView* RTXPTRenderTargets::GetDisplaySRV(bool UseComputeOutput) const
{
    if (UseComputeOutput && m_ComputeColor)
        return GetComputeColorSRV();

    return GetOutputColorSRV();
}

} // namespace Diligent
```

- [ ] **Step 3: Create blit pass header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.hpp`:

```cpp
#pragma once

#include <string>

#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderResourceBinding.h"
#include "SwapChain.h"
#include "TextureView.h"

namespace Diligent
{

class RTXPTBlitPass
{
public:
    void Reset();
    bool Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, ISwapChain* pSwapChain);
    bool Render(IDeviceContext* pContext, ISwapChain* pSwapChain, ITextureView* pSourceSRV);

    bool               IsReady() const { return m_PSO && m_SRB; }
    Uint32             GetDrawCount() const { return m_DrawCount; }
    const std::string& GetLastError() const { return m_LastError; }

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
    Uint32                                m_DrawCount = 0;
    std::string                           m_LastError;
};

} // namespace Diligent
```

- [ ] **Step 4: Create blit pass implementation**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTBlitPass.cpp`:

```cpp
#include "RTXPTBlitPass.hpp"

#include "GraphicsTypesX.hpp"

namespace Diligent
{

void RTXPTBlitPass::Reset()
{
    m_PSO.Release();
    m_SRB.Release();
    m_DrawCount = 0;
    m_LastError.clear();
}

bool RTXPTBlitPass::Initialize(IRenderDevice* pDevice, IEngineFactory* pEngineFactory, ISwapChain* pSwapChain)
{
    Reset();

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory(nullptr, &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.SourceLanguage             = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler             = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags               = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pVS;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_VERTEX;
    ShaderCI.Desc.Name       = "RTXPT blit VS";
    ShaderCI.FilePath        = "RTXPTBlit.vsh";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pVS);

    RefCntAutoPtr<IShader> pPS;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_PIXEL;
    ShaderCI.Desc.Name       = "RTXPT blit PS";
    ShaderCI.FilePath        = "RTXPTBlit.psh";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pPS);

    if (!pVS || !pPS)
    {
        m_LastError = "Failed to create RTXPT blit shaders";
        return false;
    }

    GraphicsPipelineStateCreateInfo PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT blit PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_GRAPHICS;
    PSOCreateInfo.GraphicsPipeline.NumRenderTargets             = 1;
    PSOCreateInfo.GraphicsPipeline.RTVFormats[0]                = pSwapChain->GetDesc().ColorBufferFormat;
    PSOCreateInfo.GraphicsPipeline.PrimitiveTopology            = PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP;
    PSOCreateInfo.GraphicsPipeline.RasterizerDesc.CullMode      = CULL_MODE_NONE;
    PSOCreateInfo.GraphicsPipeline.DepthStencilDesc.DepthEnable = False;
    PSOCreateInfo.PSODesc.ResourceLayout.DefaultVariableType    = SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC;
    PSOCreateInfo.pVS = pVS;
    PSOCreateInfo.pPS = pPS;

    pDevice->CreateGraphicsPipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_LastError = "Failed to create RTXPT blit PSO";
        return false;
    }

    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_LastError = "Failed to create RTXPT blit SRB";
        return false;
    }

    return true;
}

bool RTXPTBlitPass::Render(IDeviceContext* pContext, ISwapChain* pSwapChain, ITextureView* pSourceSRV)
{
    if (!IsReady() || pSourceSRV == nullptr)
        return false;

    m_SRB->GetVariableByName(SHADER_TYPE_PIXEL, "g_Texture")->Set(pSourceSRV);

    ITextureView* pRTV = pSwapChain->GetCurrentBackBufferRTV();
    pContext->SetRenderTargets(1, &pRTV, nullptr, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
    pContext->SetPipelineState(m_PSO);
    pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
    pContext->Draw(DrawAttribs{3, DRAW_FLAG_VERIFY_ALL});

    ++m_DrawCount;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 5: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTBlitPass.hpp Samples/RTXPT/src/RTXPTBlitPass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 6: Commit render target and blit pass**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRenderTargets.hpp Samples/RTXPT/src/RTXPTRenderTargets.cpp Samples/RTXPT/src/RTXPTBlitPass.hpp Samples/RTXPT/src/RTXPTBlitPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add phase 4 render target blit path" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only render target and blit pass files.

---

### Task 3: Add Minimal Ray Tracing Pass

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

- [ ] **Step 1: Create ray tracing pass header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.hpp`:

```cpp
#pragma once

#include <string>

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderBindingTable.h"
#include "ShaderResourceBinding.h"
#include "TextureView.h"
#include "TopLevelAS.h"

namespace Diligent
{

struct RTXPTRayTracingPassStats
{
    bool        Ready             = false;
    bool        LastTraceExecuted = false;
    Uint32      TraceCount        = 0;
    std::string DisabledReason;
    std::string LastError;
};

class RTXPTRayTracingPass
{
public:
    void Reset();

    bool Initialize(IRenderDevice*  pDevice,
                    IDeviceContext* pContext,
                    IEngineFactory* pEngineFactory,
                    IBuffer*        pFrameConstants,
                    ITopLevelAS*    pTLAS,
                    bool            RayTracingSupported,
                    bool            StandaloneRTShadersSupported);

    bool Trace(IDeviceContext* pContext, ITextureView* pOutputUAV, Uint32 Width, Uint32 Height);

    bool                            IsReady() const { return m_Stats.Ready; }
    const RTXPTRayTracingPassStats& GetStats() const { return m_Stats; }

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
    RefCntAutoPtr<IShaderBindingTable>    m_SBT;
    RefCntAutoPtr<ITopLevelAS>            m_TLAS;
    RTXPTRayTracingPassStats              m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create ray tracing pass implementation**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`:

```cpp
#include "RTXPTRayTracingPass.hpp"

#include <algorithm>

#include "GraphicsTypesX.hpp"

namespace Diligent
{

void RTXPTRayTracingPass::Reset()
{
    m_PSO.Release();
    m_SRB.Release();
    m_SBT.Release();
    m_TLAS.Release();
    m_Stats = {};
}

bool RTXPTRayTracingPass::Initialize(IRenderDevice*  pDevice,
                                     IDeviceContext* pContext,
                                     IEngineFactory* pEngineFactory,
                                     IBuffer*        pFrameConstants,
                                     ITopLevelAS*    pTLAS,
                                     bool            RayTracingSupported,
                                     bool            StandaloneRTShadersSupported)
{
    Reset();

    if (!RayTracingSupported)
    {
        m_Stats.DisabledReason = "Ray tracing is not supported by this device";
        return false;
    }

    if (!StandaloneRTShadersSupported)
    {
        m_Stats.DisabledReason = "Standalone ray tracing shaders are not supported by this device";
        return false;
    }

    if (pFrameConstants == nullptr || pTLAS == nullptr)
    {
        m_Stats.DisabledReason = "Frame constants or TLAS are unavailable";
        return false;
    }

    m_TLAS = pTLAS;

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory(nullptr, &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.UseCombinedTextureSamplers = false;
    ShaderCI.SourceLanguage                  = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler                  = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags                    = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.HLSLVersion                     = {6, 3};
    ShaderCI.pShaderSourceStreamFactory      = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pRayGen;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_GEN;
    ShaderCI.Desc.Name       = "RTXPT minimal raygen";
    ShaderCI.FilePath        = "RTXPTMinimal.rgen";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pRayGen);

    RefCntAutoPtr<IShader> pMiss;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_MISS;
    ShaderCI.Desc.Name       = "RTXPT minimal miss";
    ShaderCI.FilePath        = "RTXPTMinimal.rmiss";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pMiss);

    RefCntAutoPtr<IShader> pClosestHit;
    ShaderCI.Desc.ShaderType = SHADER_TYPE_RAY_CLOSEST_HIT;
    ShaderCI.Desc.Name       = "RTXPT minimal closest hit";
    ShaderCI.FilePath        = "RTXPTMinimal.rchit";
    ShaderCI.EntryPoint      = "main";
    pDevice->CreateShader(ShaderCI, &pClosestHit);

    if (!pRayGen || !pMiss || !pClosestHit)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal ray tracing shaders";
        return false;
    }

    RayTracingPipelineStateCreateInfoX PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = "RTXPT minimal RT PSO";
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_RAY_TRACING;
    PSOCreateInfo.AddGeneralShader("Main", pRayGen);
    PSOCreateInfo.AddGeneralShader("PrimaryMiss", pMiss);
    PSOCreateInfo.AddTriangleHitShader("PrimaryHit", pClosestHit);
    PSOCreateInfo.RayTracingPipeline.MaxRecursionDepth = 1;
    PSOCreateInfo.RayTracingPipeline.ShaderRecordSize  = 0;
    PSOCreateInfo.MaxAttributeSize = static_cast<Uint32>(sizeof(float) * 2);
    PSOCreateInfo.MaxPayloadSize   = static_cast<Uint32>(sizeof(float) * 4);

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_RAY_GEN | SHADER_TYPE_RAY_MISS | SHADER_TYPE_RAY_CLOSEST_HIT,
                     "g_FrameConstants",
                     SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_TLAS", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_RAY_GEN, "g_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateRayTracingPipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal RT PSO";
        return false;
    }

    if (auto* pVar = m_PSO->GetStaticVariableByName(SHADER_TYPE_RAY_GEN, "g_FrameConstants"))
        pVar->Set(pFrameConstants);
    if (auto* pVar = m_PSO->GetStaticVariableByName(SHADER_TYPE_RAY_MISS, "g_FrameConstants"))
        pVar->Set(pFrameConstants);
    if (auto* pVar = m_PSO->GetStaticVariableByName(SHADER_TYPE_RAY_CLOSEST_HIT, "g_FrameConstants"))
        pVar->Set(pFrameConstants);
    if (auto* pVar = m_PSO->GetStaticVariableByName(SHADER_TYPE_RAY_GEN, "g_TLAS"))
        pVar->Set(m_TLAS);

    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal RT SRB";
        return false;
    }

    ShaderBindingTableDesc SBTDesc;
    SBTDesc.Name = "RTXPT minimal SBT";
    SBTDesc.pPSO = m_PSO;
    pDevice->CreateSBT(SBTDesc, &m_SBT);
    if (!m_SBT)
    {
        m_Stats.LastError = "Failed to create RTXPT minimal SBT";
        return false;
    }

    m_SBT->BindRayGenShader("Main");
    m_SBT->BindMissShader("PrimaryMiss", 0);
    m_SBT->BindHitGroupForTLAS(m_TLAS, 0, "PrimaryHit");
    pContext->UpdateSBT(m_SBT);

    // TODO(RTXPT-Port Phase 4): Restore stable-plane pre-pass and fill-stable-planes dispatch; current path traces one minimal primary-ray pass.
    m_Stats.Ready = true;
    return true;
}

bool RTXPTRayTracingPass::Trace(IDeviceContext* pContext, ITextureView* pOutputUAV, Uint32 Width, Uint32 Height)
{
    m_Stats.LastTraceExecuted = false;

    if (!IsReady() || pOutputUAV == nullptr || Width == 0 || Height == 0)
        return false;

    m_SRB->GetVariableByName(SHADER_TYPE_RAY_GEN, "g_OutputColor")->Set(pOutputUAV);

    pContext->SetPipelineState(m_PSO);
    pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

    TraceRaysAttribs Attribs;
    Attribs.DimensionX = Width;
    Attribs.DimensionY = Height;
    Attribs.pSBT       = m_SBT;
    pContext->TraceRays(Attribs);

    m_Stats.LastTraceExecuted = true;
    ++m_Stats.TraceCount;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit ray tracing pass**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTRayTracingPass.hpp Samples/RTXPT/src/RTXPTRayTracingPass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add minimal phase 4 trace rays pass" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only the ray tracing pass files.

---

### Task 4: Add Reusable Compute Pass Helper

**Files:**
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`

- [ ] **Step 1: Create compute pass header**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.hpp`:

```cpp
#pragma once

#include <string>

#include "Buffer.h"
#include "DeviceContext.h"
#include "EngineFactory.h"
#include "PipelineState.h"
#include "RefCntAutoPtr.hpp"
#include "RenderDevice.h"
#include "ShaderResourceBinding.h"
#include "TextureView.h"

namespace Diligent
{

struct RTXPTComputePassStats
{
    bool        Ready                = false;
    bool        LastDispatchExecuted = false;
    Uint32      DispatchCount        = 0;
    std::string DisabledReason;
    std::string LastError;
};

class RTXPTComputePass
{
public:
    void Reset();

    bool Initialize(IRenderDevice*  pDevice,
                    IEngineFactory* pEngineFactory,
                    const char*     PassName,
                    const char*     ShaderFilePath,
                    IBuffer*        pFrameConstants,
                    bool            ComputeSupported);

    bool Dispatch(IDeviceContext* pContext, ITextureView* pInputSRV, ITextureView* pOutputUAV, Uint32 Width, Uint32 Height);

    bool                           IsReady() const { return m_Stats.Ready; }
    const RTXPTComputePassStats&   GetStats() const { return m_Stats; }
    const std::string&             GetName() const { return m_Name; }

private:
    RefCntAutoPtr<IPipelineState>         m_PSO;
    RefCntAutoPtr<IShaderResourceBinding> m_SRB;
    RTXPTComputePassStats                 m_Stats;
    std::string                           m_Name;
};

} // namespace Diligent
```

- [ ] **Step 2: Create compute pass implementation**

Create `DiligentSamples/Samples/RTXPT/src/RTXPTComputePass.cpp`:

```cpp
#include "RTXPTComputePass.hpp"

#include "GraphicsTypesX.hpp"

namespace Diligent
{

void RTXPTComputePass::Reset()
{
    m_PSO.Release();
    m_SRB.Release();
    m_Stats = {};
    m_Name.clear();
}

bool RTXPTComputePass::Initialize(IRenderDevice*  pDevice,
                                  IEngineFactory* pEngineFactory,
                                  const char*     PassName,
                                  const char*     ShaderFilePath,
                                  IBuffer*        pFrameConstants,
                                  bool            ComputeSupported)
{
    Reset();
    m_Name = PassName != nullptr ? PassName : "RTXPT compute pass";

    if (!ComputeSupported)
    {
        m_Stats.DisabledReason = "Compute shaders are not supported by this device";
        return false;
    }

    if (pFrameConstants == nullptr)
    {
        m_Stats.DisabledReason = "Frame constants are unavailable";
        return false;
    }

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory(nullptr, &pShaderSourceFactory);

    ShaderCreateInfo ShaderCI;
    ShaderCI.Desc.ShaderType           = SHADER_TYPE_COMPUTE;
    ShaderCI.Desc.Name                 = m_Name.c_str();
    ShaderCI.SourceLanguage            = SHADER_SOURCE_LANGUAGE_HLSL;
    ShaderCI.ShaderCompiler            = SHADER_COMPILER_DXC;
    ShaderCI.CompileFlags              = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
    ShaderCI.FilePath                  = ShaderFilePath;
    ShaderCI.EntryPoint                = "main";
    ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

    RefCntAutoPtr<IShader> pCS;
    pDevice->CreateShader(ShaderCI, &pCS);
    if (!pCS)
    {
        m_Stats.LastError = "Failed to create " + m_Name + " shader";
        return false;
    }

    ComputePipelineStateCreateInfo PSOCreateInfo;
    PSOCreateInfo.PSODesc.Name         = m_Name.c_str();
    PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_COMPUTE;
    PSOCreateInfo.pCS                  = pCS;

    PipelineResourceLayoutDescX ResourceLayout;
    ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_MUTABLE;
    ResourceLayout
        .AddVariable(SHADER_TYPE_COMPUTE, "g_FrameConstants", SHADER_RESOURCE_VARIABLE_TYPE_STATIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "g_InputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC)
        .AddVariable(SHADER_TYPE_COMPUTE, "g_OutputColor", SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC);
    PSOCreateInfo.PSODesc.ResourceLayout = ResourceLayout;

    pDevice->CreateComputePipelineState(PSOCreateInfo, &m_PSO);
    if (!m_PSO)
    {
        m_Stats.LastError = "Failed to create " + m_Name + " PSO";
        return false;
    }

    m_PSO->GetStaticVariableByName(SHADER_TYPE_COMPUTE, "g_FrameConstants")->Set(pFrameConstants);
    m_PSO->CreateShaderResourceBinding(&m_SRB, true);
    if (!m_SRB)
    {
        m_Stats.LastError = "Failed to create " + m_Name + " SRB";
        return false;
    }

    // TODO(RTXPT-Port Phase 4): Restore RTXDI DI/GI, light feedback, and denoising-guide compute chains; current helper runs only the debug color pass.
    m_Stats.Ready = true;
    return true;
}

bool RTXPTComputePass::Dispatch(IDeviceContext* pContext, ITextureView* pInputSRV, ITextureView* pOutputUAV, Uint32 Width, Uint32 Height)
{
    m_Stats.LastDispatchExecuted = false;

    if (!IsReady() || pInputSRV == nullptr || pOutputUAV == nullptr || Width == 0 || Height == 0)
        return false;

    m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "g_InputColor")->Set(pInputSRV);
    m_SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "g_OutputColor")->Set(pOutputUAV);

    pContext->SetPipelineState(m_PSO);
    pContext->CommitShaderResources(m_SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);

    DispatchComputeAttribs DispatchAttribs;
    DispatchAttribs.ThreadGroupCountX = (Width + 7) / 8;
    DispatchAttribs.ThreadGroupCountY = (Height + 7) / 8;
    DispatchAttribs.ThreadGroupCountZ = 1;
    pContext->DispatchCompute(DispatchAttribs);

    m_Stats.LastDispatchExecuted = true;
    ++m_Stats.DispatchCount;
    return true;
}

} // namespace Diligent
```

- [ ] **Step 3: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/src/RTXPTComputePass.hpp Samples/RTXPT/src/RTXPTComputePass.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 4: Commit compute pass helper**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/src/RTXPTComputePass.hpp Samples/RTXPT/src/RTXPTComputePass.cpp
git -C DiligentSamples commit -m "feat(rtxpt): add phase 4 compute pass helper" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing only compute pass helper files.

---

### Task 5: Wire Phase 4 Passes Into RTXPTSample

**Files:**
- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Register Phase 4 C++ files in CMake**

Modify the existing `SOURCE` list in `DiligentSamples/Samples/RTXPT/CMakeLists.txt` so it contains:

```cmake
set(SOURCE
    src/RTXPTSample.cpp
    src/RTXPTScene.cpp
    src/RTXPTMaterials.cpp
    src/RTXPTLights.cpp
    src/RTXPTAccelerationStructures.cpp
    src/RTXPTRenderTargets.cpp
    src/RTXPTRayTracingPass.cpp
    src/RTXPTComputePass.cpp
    src/RTXPTBlitPass.cpp
)
```

Modify the existing `INCLUDE` list so it contains:

```cmake
set(INCLUDE
    src/RTXPTSample.hpp
    src/RTXPTScene.hpp
    src/RTXPTMaterials.hpp
    src/RTXPTLights.hpp
    src/RTXPTAccelerationStructures.hpp
    src/RTXPTRenderTargets.hpp
    src/RTXPTRayTracingPass.hpp
    src/RTXPTComputePass.hpp
    src/RTXPTBlitPass.hpp
)
```

- [ ] **Step 2: Add pass includes and members to `RTXPTSample.hpp`**

Add these includes with the existing RTXPT includes:

```cpp
#include "RTXPTBlitPass.hpp"
#include "RTXPTComputePass.hpp"
#include "RTXPTRayTracingPass.hpp"
#include "RTXPTRenderTargets.hpp"
```

Add these private methods:

```cpp
    void CreatePhase4Passes();
    bool EnsureRenderTargets();
    void ClearFallback(const float4& ClearColor);
```

Add these private members:

```cpp
    RTXPTRenderTargets m_RenderTargets;
    RTXPTRayTracingPass m_RayTracingPass;
    RTXPTComputePass    m_DebugComputePass;
    RTXPTBlitPass       m_BlitPass;
    bool                m_EnableDebugComputePass = true;
```

- [ ] **Step 3: Add helper implementations to `RTXPTSample.cpp`**

Add these methods after `UpdateFrameConstants`:

```cpp
void RTXPTSample::CreatePhase4Passes()
{
    m_BlitPass.Initialize(m_pDevice, m_pEngineFactory, m_pSwapChain);

    m_RayTracingPass.Initialize(m_pDevice,
                                m_pImmediateContext,
                                m_pEngineFactory,
                                m_FrameConstantsCB,
                                m_AccelerationStructures.GetTLAS(),
                                m_FeatureCaps.RayTracing,
                                m_FeatureCaps.StandaloneRayTracingShaders);

    m_DebugComputePass.Initialize(m_pDevice,
                                  m_pEngineFactory,
                                  "RTXPT debug compute pass",
                                  "RTXPTDebugCompute.csh",
                                  m_FrameConstantsCB,
                                  m_FeatureCaps.ComputeShaders);
}

bool RTXPTSample::EnsureRenderTargets()
{
    const SwapChainDesc& SCDesc = m_pSwapChain->GetDesc();
    return m_RenderTargets.Resize(m_pDevice,
                                  SCDesc.Width,
                                  SCDesc.Height,
                                  TEX_FORMAT_RGBA8_UNORM,
                                  m_FeatureCaps.ComputeShaders);
}

void RTXPTSample::ClearFallback(const float4& ClearColor)
{
    ITextureView* pRTV = m_pSwapChain->GetCurrentBackBufferRTV();
    m_pImmediateContext->SetRenderTargets(1, &pRTV, nullptr, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
    m_pImmediateContext->ClearRenderTarget(pRTV, ClearColor.Data(), RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
}
```

- [ ] **Step 4: Initialize Phase 4 passes after Phase 3 resources**

At the end of `RTXPTSample::Initialize`, after the Phase 3 scene/material/light/AS setup block, call:

```cpp
    CreatePhase4Passes();
    EnsureRenderTargets();
```

Expected local ordering:

```cpp
    if (const GLTF::Model* pModel = m_Scene.GetModel())
    {
        m_Materials.Upload(m_pDevice, *pModel);
        if (m_Scene.GetSceneIndex() < pModel->Scenes.size())
            m_Lights.Upload(m_pDevice, pModel->Scenes[m_Scene.GetSceneIndex()], m_Scene.GetTransforms());

        m_AccelerationStructures.BuildStaticScene(m_pDevice,
                                                  m_pImmediateContext,
                                                  *pModel,
                                                  m_Scene.GetSceneIndex(),
                                                  m_Scene.GetIndexType(),
                                                  m_Scene.GetTransforms(),
                                                  m_FeatureCaps.RayTracing);
    }
    else
    {
        m_AccelerationStructures.Reset();
    }

    CreatePhase4Passes();
    EnsureRenderTargets();
```

- [ ] **Step 5: Replace clear-only RT branch in `Render()`**

Replace the current `RTXPTSample::Render()` body with:

```cpp
void RTXPTSample::Render()
{
    const auto ClearColor = float4{0.05f, 0.05f, 0.07f, 1.0f};

    if (!EnsureRenderTargets())
    {
        ClearFallback(ClearColor);
        return;
    }

    const bool TraceExecuted =
        m_RayTracingPass.Trace(m_pImmediateContext,
                               m_RenderTargets.GetOutputColorUAV(),
                               m_RenderTargets.GetWidth(),
                               m_RenderTargets.GetHeight());

    if (!TraceExecuted)
    {
        ClearFallback(ClearColor);
        return;
    }

    const bool ComputeExecuted =
        m_EnableDebugComputePass &&
        m_DebugComputePass.Dispatch(m_pImmediateContext,
                                    m_RenderTargets.GetOutputColorSRV(),
                                    m_RenderTargets.GetComputeColorUAV(),
                                    m_RenderTargets.GetWidth(),
                                    m_RenderTargets.GetHeight());

    ITextureView* pDisplaySRV = m_RenderTargets.GetDisplaySRV(ComputeExecuted);
    if (!m_BlitPass.Render(m_pImmediateContext, m_pSwapChain, pDisplaySRV))
    {
        ClearFallback(ClearColor);
        return;
    }
}
```

- [ ] **Step 6: Update resize behavior**

Replace `RTXPTSample::WindowResize` with:

```cpp
void RTXPTSample::WindowResize(Uint32 Width, Uint32 Height)
{
    if (Width == 0 || Height == 0)
        return;

    m_RenderTargets.Resize(m_pDevice,
                           Width,
                           Height,
                           TEX_FORMAT_RGBA8_UNORM,
                           m_FeatureCaps.ComputeShaders);
}
```

- [ ] **Step 7: Add UI status for Phase 4 passes**

Add this block near the end of `RTXPTSample::UpdateUI()`, before the existing TODO text:

```cpp
    const RTXPTRayTracingPassStats& RTPassStats = m_RayTracingPass.GetStats();
    const RTXPTComputePassStats&    ComputeStats = m_DebugComputePass.GetStats();
    ImGui::Separator();
    ImGui::Text("OutputColor: %s", m_RenderTargets.IsValid() ? "created" : "missing");
    ImGui::Text("TraceRays pass: %s", m_RayTracingPass.IsReady() ? "ready" : "not ready");
    ImGui::Text("TraceRays executed: %s", RTPassStats.LastTraceExecuted ? "yes" : "no");
    ImGui::Text("TraceRays count: %u", RTPassStats.TraceCount);
    if (!RTPassStats.DisabledReason.empty())
        ImGui::TextWrapped("TraceRays disabled: %s", RTPassStats.DisabledReason.c_str());
    if (!RTPassStats.LastError.empty())
        ImGui::TextWrapped("TraceRays error: %s", RTPassStats.LastError.c_str());
    ImGui::Checkbox("Debug compute pass", &m_EnableDebugComputePass);
    ImGui::Text("Compute dispatch: %s", m_DebugComputePass.IsReady() ? "ready" : "not ready");
    ImGui::Text("Compute executed: %s", ComputeStats.LastDispatchExecuted ? "yes" : "no");
    ImGui::Text("Compute dispatch count: %u", ComputeStats.DispatchCount);
    if (!ComputeStats.DisabledReason.empty())
        ImGui::TextWrapped("Compute disabled: %s", ComputeStats.DisabledReason.c_str());
    if (!ComputeStats.LastError.empty())
        ImGui::TextWrapped("Compute error: %s", ComputeStats.LastError.c_str());
    if (!m_RenderTargets.GetLastError().empty())
        ImGui::TextWrapped("Render target error: %s", m_RenderTargets.GetLastError().c_str());
    if (!m_BlitPass.GetLastError().empty())
        ImGui::TextWrapped("Blit error: %s", m_BlitPass.GetLastError().c_str());
    ImGui::Text("Blit draw count: %u", m_BlitPass.GetDrawCount());
```

Keep the existing Phase 1 UI TODO and add one Phase 4 UI TODO line:

```cpp
    ImGui::Text("TODO(RTXPT-Port Phase 4): expose stable-plane, RTXDI, light feedback, and denoising-guide pass toggles after their shaders are ported.");
```

- [ ] **Step 8: Run a non-build formatting check**

Run:

```powershell
git -C DiligentSamples diff --check -- Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
```

Expected: no output and exit code 0.

- [ ] **Step 9: Commit RTXPTSample Phase 4 wiring**

Run:

```bash
git -C DiligentSamples add Samples/RTXPT/CMakeLists.txt Samples/RTXPT/src/RTXPTSample.hpp Samples/RTXPT/src/RTXPTSample.cpp
git -C DiligentSamples commit -m "feat(rtxpt): wire phase 4 draw and dispatch passes" -m "Co-Authored-By: GPT 5.5"
```

Expected: one `DiligentSamples` commit containing CMake C++ registration, `RTXPTSample.hpp`, and `RTXPTSample.cpp`.

---

### Task 6: Phase 4 Verification And Handoff

**Files:**
- Verify: `DiligentSamples/Samples/RTXPT`
- Verify: top-level repository

- [ ] **Step 1: Confirm structured TODOs**

Run:

```powershell
rg -n "TODO\(RTXPT-Port Phase 4\)" DiligentSamples/Samples/RTXPT
```

Expected TODOs:

```text
Stable-plane pre-pass and fill-stable-planes dispatch deferred behind minimal TraceRays path
RTXDI DI/GI, light feedback, and denoising-guide compute chains deferred behind debug compute helper
UI pass toggles deferred until the shaders are ported
```

- [ ] **Step 2: Confirm file registration**

Run:

```powershell
rg -n "RTXPTRenderTargets|RTXPTRayTracingPass|RTXPTComputePass|RTXPTBlitPass|RTXPTMinimal|RTXPTDebugCompute|RTXPTBlit" DiligentSamples/Samples/RTXPT/CMakeLists.txt
```

Expected: every new Phase 4 source/header/shader file is listed in `CMakeLists.txt`.

- [ ] **Step 3: Optional compile verification when the user explicitly requests it**

The workspace rule says not to run build commands unless explicitly requested. If the user asks for build verification, run the configured RTXPT target:

```powershell
cmake --build build\x64\Debug --config Debug --target RTXPT
```

Expected: command exits with code 0. If this build tree or target is unavailable, inspect the configured build directory first and report the exact alternative command used.

- [ ] **Step 4: Optional D3D12 runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with D3D12 on a standalone-RT-capable machine. Expected UI facts:

```text
Scene: loaded
Acceleration structures: built
OutputColor: created
TraceRays pass: ready
TraceRays executed: yes
TraceRays count: increases every frame
Compute dispatch: ready
Compute executed: yes when Debug compute pass is enabled
Blit draw count: increases every frame
```

Expected visual result: a visible sky/background and barycentric/depth debug coloring for hit geometry instead of the old clear-only fallback.

- [ ] **Step 5: Optional Vulkan runtime verification when the user explicitly requests it**

Launch `Samples/RTXPT` with Vulkan on a standalone-RT-capable machine. Expected UI facts match the D3D12 run:

```text
TraceRays pass: ready
TraceRays executed: yes
Compute dispatch: ready
Blit draw count: increases every frame
```

If standalone ray tracing shaders are unavailable on the Vulkan device, expected fallback UI facts are:

```text
TraceRays pass: not ready
TraceRays disabled: Standalone ray tracing shaders are not supported by this device
The sample still launches and clears the swapchain
```

- [ ] **Step 6: Commit top-level submodule pointer and plan**

After all `DiligentSamples` Phase 4 commits are complete, run from `d:\DiligentEngine-hzqst`:

```bash
git add DiligentSamples docs/superpowers/plans/2026-05-27-rtxpt-phase4-draw-dispatch.md
git commit -m "feat(samples): plan and add RTXPT phase 4 draw dispatch" -m "Co-Authored-By: GPT 5.5"
```

Expected: top-level commit records the updated `DiligentSamples` submodule pointer and this plan document.

---

## Self-Review Checklist

- [x] The plan directly implements the Phase 4 runnable milestone from `docs/superpowers/specs/2026-05-26-rtxpt-diligent-port-design.md`.
- [x] The plan adds a minimal RT PSO, SBT, and `IDeviceContext::TraceRays`.
- [x] The plan adds a reusable compute pass helper and a minimal compute dispatch.
- [x] The plan adds an `OutputColor` texture, optional `ComputeColor` texture, and swapchain blit path.
- [x] The no-ray-tracing and no-standalone-RT-shader fallbacks keep the sample runnable.
- [x] Stable planes, RTXDI, light feedback, and denoising guide work are represented by structured `TODO(RTXPT-Port Phase 4)` comments and disabled pass scope.
- [x] Full shader porting, material-specialized SBT generation, denoising, DLSS, OMM, SER, and NVAPI are kept out of this runnable increment.
- [x] Every new source/header/shader file is added to `DiligentSamples/Samples/RTXPT/CMakeLists.txt`.
- [x] Verification commands avoid build/runtime execution unless the user explicitly asks for it.
- [x] Commit commands use the required `Co-Authored-By: GPT 5.5` trailer.
