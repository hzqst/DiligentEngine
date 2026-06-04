# RTXPT Realtime G8-G9 NRD Integration and Denoise Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional Diligent-native NRD integration for the RTXPT realtime path and wire the realtime standalone `Denoise` orchestration around the existing G7 prepare/final-merge passes.

**Architecture:** Detect the RTXPT-local NRD/NRI submodules by default and keep explicit root overrides for external SDK testing. When NRD is unavailable or explicitly disabled, the sample still builds and runs through `NoDenoiserFinalMerge`; when NRD is available, `RTXPTSample` owns one `RTXPTNrdIntegration` per stable plane, resets/recreates them on render-target or method changes, and runs prepare -> NRD -> final merge from the highest active plane down to plane 0.

**Tech Stack:** C++17, Diligent Engine compute PSOs/SRBs, HLSL/DXC, optional NVIDIA NRD SDK (`NRD`, `NRDSettings`, `NRDDescs`), RTXPT-fork reference files `Rtxpt/NRD/NrdIntegration.*`, `Rtxpt/NRD/NrdConfig.*`, and `Rtxpt/Sample.cpp::Denoise`.

---

## Scope

Implements:

- CMake gate `RTXPT_NRD_ROOT` and `RTXPT_NRI_ROOT`; by default they point to `DiligentSamples/Samples/RTXPT/thirdparty/NRD` and `DiligentSamples/Samples/RTXPT/thirdparty/NRI`.
- `RTXPTNrdConfig` UI-to-NRD setting conversion for REBLUR and RELAX.
- `RTXPTNrdIntegration` instance creation, NRD compute PSO creation, samplers, permanent/transient pools, constant buffer, resource mapping, common settings, and dispatch.
- `RTXPTSample::Denoise` equivalent orchestration using existing G7 `RunDenoiserPrepare` and `RunDenoiserFinalMerge`.
- UI/status behavior that enables standalone NRD only when the build gate is active and `RealtimeAA != DLSSRR`.
- Reset/recreate behavior for render-target resize, NRD method switch, and realtime cache reset.
- Mapping document updates for the new G8/G9 symbols.

Does not implement:

- DLSS-RR or Streamline resource tagging.
- TAA/SR handoff beyond the already existing `PresentRealtimeFinalOutput()` path.
- New denoiser guide algorithms; G6/G7 own guide and prepare/merge shaders.
- Fetching NRD or NRI from the network. The plan uses the RTXPT sample `thirdparty/NRD` and `thirdparty/NRI` submodules by default, while still allowing explicit root overrides.

---

## File Structure

Create:

- `DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.hpp`
  Declares NRD availability helpers and UI setting conversion interfaces.
- `DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.cpp`
  Converts `RTXPTRealtimeSettings` REBLUR/RELAX UI fields into NRD method settings when NRD is enabled.
- `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.hpp`
  Declares per-plane Diligent-native NRD integration, dispatch attribs, and stats.
- `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp`
  Owns optional NRD instance, Diligent compute PSOs/SRBs, samplers, pool textures, common settings, and dispatch resource mapping.

Modify:

- `DiligentSamples/Samples/RTXPT/CMakeLists.txt`
  Register new files, add optional NRD subdirectory/link, compile definitions, and NRD shader include paths.
- `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp`
  Add NRD shader include path and `RTXPT_HAS_NRD_HEADERS` shader macro so `RTXPTDenoiserNRD.hlsli` can use real NRD headers when present.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
  Add per-plane `RTXPTNrdIntegration`, denoise helpers, and lifecycle reset helper declarations.
- `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`
  Replace the G8 stub branch with `Denoise`, update UI availability text, and reset/recreate NRD instances.
- `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`
  Add G8/G9 mapping rows.

Reference but do not modify:

- `docs/superpowers/specs/2026-06-03-rtxpt-realtime-pathtrace-denoise-port-spec.md`
- `D:/RTXPT-fork/Rtxpt/NRD/NrdIntegration.{h,cpp}`
- `D:/RTXPT-fork/Rtxpt/NRD/NrdConfig.{h,cpp}`
- `D:/RTXPT-fork/Rtxpt/Sample.cpp::Denoise`

---

### Task 1: Add Optional NRD Build Gate and Register Files

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/CMakeLists.txt`

- [ ] **Step 1: Register new RTXPT NRD source/header files**

Add the new source files to `SOURCE` after `src/RTXPTPostProcessPass.cpp`:

```cmake
    src/RTXPTNrdConfig.cpp
    src/RTXPTNrdIntegration.cpp
```

Add the new headers to `INCLUDE` after `src/RTXPTPostProcessPass.hpp`:

```cmake
    src/RTXPTNrdConfig.hpp
    src/RTXPTNrdIntegration.hpp
```

- [ ] **Step 2: Add default NRD/NRI root detection before `add_sample_app`**

Insert this block after the `set(SHADERS ...)` block and before `add_sample_app(RTXPT ...)`:

```cmake
set(RTXPT_NRD_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/NRD" CACHE PATH "NVIDIA NRD SDK root for the RTXPT standalone denoiser")
set(RTXPT_NRI_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/NRI" CACHE PATH "NVIDIA NRI SDK root used by the RTXPT NRD submodule when NRD_NRI is enabled")
set(RTXPT_HAS_NRD OFF)
set(RTXPT_NRD_SHADER_INCLUDE_DIR "")
set(RTXPT_NRD_SHADER_SOURCE_DIR "")
set(RTXPT_NRI_ROOT_NORMALIZED "")
set(RTXPT_NRI_AVAILABLE OFF)
set(NRD_NRI OFF CACHE BOOL "Build NRD NRI integration")

if(RTXPT_NRD_ROOT)
    file(TO_CMAKE_PATH "${RTXPT_NRD_ROOT}" RTXPT_NRD_ROOT_NORMALIZED)
    if(RTXPT_NRI_ROOT)
        file(TO_CMAKE_PATH "${RTXPT_NRI_ROOT}" RTXPT_NRI_ROOT_NORMALIZED)
        if(EXISTS "${RTXPT_NRI_ROOT_NORMALIZED}/CMakeLists.txt")
            set(FETCHCONTENT_SOURCE_DIR_NRI "${RTXPT_NRI_ROOT_NORMALIZED}" CACHE PATH "Use RTXPT's local NRI submodule when NRD_NRI is enabled" FORCE)
            set(RTXPT_NRI_AVAILABLE ON)
        elseif(NRD_NRI)
            message(WARNING "NRD_NRI is ON, but RTXPT_NRI_ROOT does not contain a valid NRI checkout.")
        endif()
    endif()

    set(RTXPT_NRD_INCLUDE_DIR "${RTXPT_NRD_ROOT_NORMALIZED}/Include")
    set(RTXPT_NRD_SHADER_INCLUDE_DIR "${RTXPT_NRD_ROOT_NORMALIZED}/Shaders/Include")
    set(RTXPT_NRD_SHADER_SOURCE_DIR "${RTXPT_NRD_ROOT_NORMALIZED}/Shaders/Source")

    if(EXISTS "${RTXPT_NRD_ROOT_NORMALIZED}/CMakeLists.txt" AND
       EXISTS "${RTXPT_NRD_INCLUDE_DIR}/NRD.h" AND
       EXISTS "${RTXPT_NRD_INCLUDE_DIR}/NRDSettings.h" AND
       EXISTS "${RTXPT_NRD_INCLUDE_DIR}/NRDDescs.h" AND
       EXISTS "${RTXPT_NRD_SHADER_INCLUDE_DIR}/NRD.hlsli" AND
       EXISTS "${RTXPT_NRD_SHADER_SOURCE_DIR}/REBLUR_DiffuseSpecular_TemporalAccumulation.cs.hlsl" AND
       (NOT NRD_NRI OR RTXPT_NRI_AVAILABLE))
        set(NRD_STATIC_LIBRARY ON CACHE BOOL "Build NRD as a static library" FORCE)
        set(NRD_EMBEDS_DXIL_SHADERS OFF CACHE BOOL "RTXPT compiles NRD HLSL through Diligent DXC" FORCE)
        set(NRD_EMBEDS_DXBC_SHADERS OFF CACHE BOOL "RTXPT compiles NRD HLSL through Diligent DXC" FORCE)
        set(NRD_EMBEDS_SPIRV_SHADERS OFF CACHE BOOL "RTXPT compiles NRD HLSL through Diligent DXC" FORCE)
        add_subdirectory("${RTXPT_NRD_ROOT_NORMALIZED}" "${CMAKE_CURRENT_BINARY_DIR}/NRD")
        set(RTXPT_HAS_NRD ON)
    else()
        message(WARNING "RTXPT_NRD_ROOT is set, but required NRD files were not found, or NRD_NRI is ON without a valid RTXPT_NRI_ROOT. RTXPT standalone NRD will be disabled.")
    endif()
endif()
```

- [ ] **Step 3: Link and expose compile definitions after `target_link_libraries(RTXPT ...)`**

Add this block after the existing `target_link_libraries(RTXPT PRIVATE ...)` call:

```cmake
target_compile_definitions(RTXPT
PRIVATE
    RTXPT_HAS_NRD=$<BOOL:${RTXPT_HAS_NRD}>
    RTXPT_NRD_SHADER_INCLUDE_DIR="${RTXPT_NRD_SHADER_INCLUDE_DIR}"
    RTXPT_NRD_SHADER_SOURCE_DIR="${RTXPT_NRD_SHADER_SOURCE_DIR}"
)

if(RTXPT_HAS_NRD)
    target_link_libraries(RTXPT PRIVATE NRD)
    target_include_directories(RTXPT PRIVATE
        "${RTXPT_NRD_INCLUDE_DIR}"
        "${RTXPT_NRD_SHADER_INCLUDE_DIR}"
    )
endif()
```

- [ ] **Step 4: Configure without NRD and verify the gate stays off**

Run:

```powershell
cmake -S . -B build\rtxpt-g8-no-nrd -G "Visual Studio 17 2022" -A x64 -DRTXPT_NRD_ROOT= -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
```

Expected:

```text
Configuring done
Generating done
```

No fatal error should mention `NRD.h`, `NRDSettings.h`, or `NRI`.

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/CMakeLists.txt
git commit -m "chore(rtxpt): add optional NRD build gate"
```

---

### Task 2: Add NRD Config Conversion

**Files:**

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.cpp`

- [ ] **Step 1: Create `RTXPTNrdConfig.hpp`**

```cpp
#pragma once

#include "RTXPTRealtimeSettings.hpp"

namespace Diligent
{

const char* RTXPTGetNrdUnavailableReason();
bool        RTXPTIsNrdAvailable();

#if RTXPT_HAS_NRD

#    include "NRD.h"
#    include "NRDSettings.h"

nrd::Denoiser     RTXPTToNrdDenoiser(RTXPTNrdMethod Method);
nrd::RelaxSettings RTXPTMakeRelaxSettings(const RTXPTNrdRelaxUiSettings& Ui);
nrd::ReblurSettings RTXPTMakeReblurSettings(const RTXPTNrdReblurUiSettings& Ui);

#endif

} // namespace Diligent
```

- [ ] **Step 2: Create `RTXPTNrdConfig.cpp`**

```cpp
#include "RTXPTNrdConfig.hpp"

namespace Diligent
{

const char* RTXPTGetNrdUnavailableReason()
{
#if RTXPT_HAS_NRD
    return "Standalone NRD is available.";
#else
    return "Standalone denoiser disabled: NRD/NRI submodules are missing or RTXPT_NRD_ROOT was cleared.";
#endif
}

bool RTXPTIsNrdAvailable()
{
#if RTXPT_HAS_NRD
    return true;
#else
    return false;
#endif
}

#if RTXPT_HAS_NRD

namespace
{

nrd::HitDistanceReconstructionMode ToNrdHitDistanceMode(RTXPTNrdHitDistanceReconstructionMode Mode)
{
    switch (Mode)
    {
        case RTXPTNrdHitDistanceReconstructionMode::Off:
            return nrd::HitDistanceReconstructionMode::OFF;
        case RTXPTNrdHitDistanceReconstructionMode::Area3x3:
            return nrd::HitDistanceReconstructionMode::AREA_3X3;
        case RTXPTNrdHitDistanceReconstructionMode::Area5x5:
            return nrd::HitDistanceReconstructionMode::AREA_5X5;
        default:
            return nrd::HitDistanceReconstructionMode::OFF;
    }
}

} // namespace

nrd::Denoiser RTXPTToNrdDenoiser(RTXPTNrdMethod Method)
{
    return Method == RTXPTNrdMethod::RELAX ?
        nrd::Denoiser::RELAX_DIFFUSE_SPECULAR :
        nrd::Denoiser::REBLUR_DIFFUSE_SPECULAR;
}

nrd::RelaxSettings RTXPTMakeRelaxSettings(const RTXPTNrdRelaxUiSettings& Ui)
{
    nrd::RelaxSettings Settings;
    Settings.enableAntiFirefly                  = Ui.EnableAntiFirefly;
    Settings.hitDistanceReconstructionMode      = ToNrdHitDistanceMode(Ui.HitDistanceReconstructionMode);
    Settings.diffusePrepassBlurRadius           = Ui.DiffusePrepassBlurRadius;
    Settings.specularPrepassBlurRadius          = Ui.SpecularPrepassBlurRadius;
    Settings.diffuseMaxAccumulatedFrameNum      = Ui.DiffuseMaxAccumulatedFrameNum;
    Settings.specularMaxAccumulatedFrameNum     = Ui.SpecularMaxAccumulatedFrameNum;
    Settings.diffuseMaxFastAccumulatedFrameNum  = Ui.DiffuseMaxFastAccumulatedFrameNum;
    Settings.specularMaxFastAccumulatedFrameNum = Ui.SpecularMaxFastAccumulatedFrameNum;
    Settings.historyFixFrameNum                 = Ui.HistoryFixFrameNum;
    Settings.atrousIterationNum                 = Ui.AtrousIterationNum;
    Settings.lobeAngleFraction                  = Ui.LobeAngleFraction;
    Settings.specularLobeAngleSlack             = Ui.SpecularLobeAngleSlack;
    Settings.depthThreshold                     = Ui.DepthThreshold;
    Settings.antilagSettings.accelerationAmount = Ui.AntilagAccelerationAmount;
    Settings.antilagSettings.spatialSigmaScale  = Ui.AntilagSpatialSigmaScale;
    Settings.antilagSettings.temporalSigmaScale = Ui.AntilagTemporalSigmaScale;
    Settings.antilagSettings.resetAmount        = Ui.AntilagResetAmount;
    return Settings;
}

nrd::ReblurSettings RTXPTMakeReblurSettings(const RTXPTNrdReblurUiSettings& Ui)
{
    nrd::ReblurSettings Settings;
    Settings.enableAntiFirefly             = Ui.EnableAntiFirefly;
    Settings.hitDistanceReconstructionMode = ToNrdHitDistanceMode(Ui.HitDistanceReconstructionMode);
    Settings.maxAccumulatedFrameNum        = Ui.MaxAccumulatedFrameNum;
    Settings.maxFastAccumulatedFrameNum    = Ui.MaxFastAccumulatedFrameNum;
    Settings.historyFixFrameNum            = Ui.HistoryFixFrameNum;
    Settings.diffusePrepassBlurRadius      = Ui.DiffusePrepassBlurRadius;
    Settings.specularPrepassBlurRadius     = Ui.SpecularPrepassBlurRadius;
    return Settings;
}

#endif

} // namespace Diligent
```

- [ ] **Step 3: Build without NRD to prove the file compiles with the gate off**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.hpp DiligentSamples/Samples/RTXPT/src/RTXPTNrdConfig.cpp
git commit -m "feat(rtxpt): add NRD setting conversion"
```

---

### Task 3: Add `RTXPTNrdIntegration` Public API and No-NRD Stub

**Files:**

- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.hpp`
- Create: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp`

- [ ] **Step 1: Create `RTXPTNrdIntegration.hpp`**

```cpp
#pragma once

#include "RTXPTFrameConstants.hpp"
#include "RTXPTNrdConfig.hpp"
#include "RTXPTRenderTargets.hpp"

#include <string>
#include <vector>

namespace Diligent
{

struct RTXPTNrdFrameAttribs
{
    const RTXPTRenderTargets*    pRenderTargets  = nullptr;
    const SampleConstants*       pFrameConstants = nullptr;
    const RTXPTRealtimeSettings* pRealtime       = nullptr;
    Uint32                       PlaneIndex      = 0;
    Uint32                       FrameIndex      = 0;
    float                        TimeDeltaSeconds = -1.0f;
    bool                         ResetHistory    = false;
    bool                         EnableValidation = false;
};

struct RTXPTNrdIntegrationStats
{
    bool        Ready                = false;
    bool        LastDispatchExecuted = false;
    Uint32      DispatchCount        = 0;
    Uint32      LastPlaneIndex       = 0;
    Uint32      LastDispatches       = 0;
    Uint32      Width                = 0;
    Uint32      Height               = 0;
    RTXPTNrdMethod Method            = RTXPTNrdMethod::REBLUR;
    std::string LastFailureReason;
};

class RTXPTNrdIntegration
{
public:
    ~RTXPTNrdIntegration();

    void Reset();

    bool Initialize(IRenderDevice*  pDevice,
                    IEngineFactory* pEngineFactory,
                    RTXPTNrdMethod  Method,
                    Uint32          Width,
                    Uint32          Height,
                    bool            ComputeSupported);

    bool Dispatch(IDeviceContext* pContext, const RTXPTNrdFrameAttribs& Attribs);

    bool                               IsReady() const { return m_Stats.Ready; }
    RTXPTNrdMethod                     GetMethod() const { return m_Stats.Method; }
    Uint32                             GetWidth() const { return m_Stats.Width; }
    Uint32                             GetHeight() const { return m_Stats.Height; }
    const char*                        GetLastFailureReason() const { return m_Stats.LastFailureReason.c_str(); }
    const RTXPTNrdIntegrationStats&    GetStats() const { return m_Stats; }

private:
    bool Fail(const char* Reason);

#if RTXPT_HAS_NRD
    struct PipelineState
    {
        RefCntAutoPtr<IPipelineState>         PSO;
        RefCntAutoPtr<IShaderResourceBinding> SRB;
    };

    bool CreateInstance(RTXPTNrdMethod Method);
    bool CreateConstantBuffer(IRenderDevice* pDevice);
    bool CreateSamplers(IRenderDevice* pDevice);
    bool CreatePipelines(IRenderDevice* pDevice, IEngineFactory* pEngineFactory);
    bool CreatePoolTextures(IRenderDevice* pDevice, Uint32 Width, Uint32 Height);
    bool BindDispatchResources(IDeviceContext* pContext,
                               const nrd::DispatchDesc& DispatchDesc,
                               const nrd::PipelineDesc& PipelineDesc,
                               PipelineState&           Pipeline,
                               const RTXPTNrdFrameAttribs& Attribs);
    void PopulateCommonSettings(nrd::CommonSettings& Settings, const RTXPTNrdFrameAttribs& Attribs) const;

    nrd::Instance*              m_Instance = nullptr;
    nrd::Identifier             m_Identifier = 0;
    RefCntAutoPtr<IRenderDevice> m_Device;
    RefCntAutoPtr<IBuffer>      m_ConstantBuffer;
    std::vector<PipelineState>  m_Pipelines;
    std::vector<RefCntAutoPtr<ISampler>> m_Samplers;
    std::vector<RefCntAutoPtr<ITexture>> m_PermanentTextures;
    std::vector<RefCntAutoPtr<ITexture>> m_TransientTextures;
#endif

    RTXPTNrdIntegrationStats m_Stats;
};

} // namespace Diligent
```

- [ ] **Step 2: Create the no-NRD and shared parts of `RTXPTNrdIntegration.cpp`**

```cpp
#include "RTXPTNrdIntegration.hpp"

#include "DebugUtilities.hpp"

namespace Diligent
{

RTXPTNrdIntegration::~RTXPTNrdIntegration()
{
    Reset();
}

bool RTXPTNrdIntegration::Fail(const char* Reason)
{
    m_Stats.LastFailureReason = Reason != nullptr ? Reason : "NRD integration failed";
    DEV_ERROR(m_Stats.LastFailureReason.c_str());
    return false;
}

#if !RTXPT_HAS_NRD

void RTXPTNrdIntegration::Reset()
{
    m_Stats = {};
}

bool RTXPTNrdIntegration::Initialize(IRenderDevice*,
                                     IEngineFactory*,
                                     RTXPTNrdMethod Method,
                                     Uint32 Width,
                                     Uint32 Height,
                                     bool)
{
    m_Stats.Method = Method;
    m_Stats.Width  = Width;
    m_Stats.Height = Height;
    return Fail(RTXPTGetNrdUnavailableReason());
}

bool RTXPTNrdIntegration::Dispatch(IDeviceContext*, const RTXPTNrdFrameAttribs&)
{
    m_Stats.LastDispatchExecuted = false;
    return Fail(RTXPTGetNrdUnavailableReason());
}

#endif

} // namespace Diligent
```

- [ ] **Step 3: Build without NRD**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.hpp DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp
git commit -m "feat(rtxpt): add NRD integration interface"
```

---

### Task 4: Implement NRD Instance, Shaders, Samplers, and Pools

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp`

- [ ] **Step 1: Add NRD-enabled includes and helpers**

Append this below the shared no-NRD block in `RTXPTNrdIntegration.cpp`:

```cpp
#if RTXPT_HAS_NRD

#include "MapHelper.hpp"
#include "ShaderMacroHelper.hpp"

#include <algorithm>
#include <array>
#include <cstdlib>
#include <sstream>

namespace Diligent
{

namespace
{

void* NrdAllocate(void*, size_t Size, size_t)
{
    return std::malloc(Size);
}

void* NrdReallocate(void*, void* Memory, size_t Size, size_t)
{
    return std::realloc(Memory, Size);
}

void NrdFree(void*, void* Memory)
{
    std::free(Memory);
}

TEXTURE_FORMAT ToDiligentFormat(nrd::Format Format)
{
    switch (Format)
    {
        case nrd::Format::R8_UNORM:             return TEX_FORMAT_R8_UNORM;
        case nrd::Format::R8_SNORM:             return TEX_FORMAT_R8_SNORM;
        case nrd::Format::R8_UINT:              return TEX_FORMAT_R8_UINT;
        case nrd::Format::R8_SINT:              return TEX_FORMAT_R8_SINT;
        case nrd::Format::RG8_UNORM:            return TEX_FORMAT_RG8_UNORM;
        case nrd::Format::RG8_SNORM:            return TEX_FORMAT_RG8_SNORM;
        case nrd::Format::RG8_UINT:             return TEX_FORMAT_RG8_UINT;
        case nrd::Format::RG8_SINT:             return TEX_FORMAT_RG8_SINT;
        case nrd::Format::RGBA8_UNORM:          return TEX_FORMAT_RGBA8_UNORM;
        case nrd::Format::RGBA8_SNORM:          return TEX_FORMAT_RGBA8_SNORM;
        case nrd::Format::RGBA8_UINT:           return TEX_FORMAT_RGBA8_UINT;
        case nrd::Format::RGBA8_SINT:           return TEX_FORMAT_RGBA8_SINT;
        case nrd::Format::RGBA8_SRGB:           return TEX_FORMAT_RGBA8_UNORM_SRGB;
        case nrd::Format::R16_UNORM:            return TEX_FORMAT_R16_UNORM;
        case nrd::Format::R16_SNORM:            return TEX_FORMAT_R16_SNORM;
        case nrd::Format::R16_UINT:             return TEX_FORMAT_R16_UINT;
        case nrd::Format::R16_SINT:             return TEX_FORMAT_R16_SINT;
        case nrd::Format::R16_SFLOAT:           return TEX_FORMAT_R16_FLOAT;
        case nrd::Format::RG16_UNORM:           return TEX_FORMAT_RG16_UNORM;
        case nrd::Format::RG16_SNORM:           return TEX_FORMAT_RG16_SNORM;
        case nrd::Format::RG16_UINT:            return TEX_FORMAT_RG16_UINT;
        case nrd::Format::RG16_SINT:            return TEX_FORMAT_RG16_SINT;
        case nrd::Format::RG16_SFLOAT:          return TEX_FORMAT_RG16_FLOAT;
        case nrd::Format::RGBA16_UNORM:         return TEX_FORMAT_RGBA16_UNORM;
        case nrd::Format::RGBA16_SNORM:         return TEX_FORMAT_RGBA16_SNORM;
        case nrd::Format::RGBA16_UINT:          return TEX_FORMAT_RGBA16_UINT;
        case nrd::Format::RGBA16_SINT:          return TEX_FORMAT_RGBA16_SINT;
        case nrd::Format::RGBA16_SFLOAT:        return TEX_FORMAT_RGBA16_FLOAT;
        case nrd::Format::R32_UINT:             return TEX_FORMAT_R32_UINT;
        case nrd::Format::R32_SINT:             return TEX_FORMAT_R32_SINT;
        case nrd::Format::R32_SFLOAT:           return TEX_FORMAT_R32_FLOAT;
        case nrd::Format::RG32_UINT:            return TEX_FORMAT_RG32_UINT;
        case nrd::Format::RG32_SINT:            return TEX_FORMAT_RG32_SINT;
        case nrd::Format::RG32_SFLOAT:          return TEX_FORMAT_RG32_FLOAT;
        case nrd::Format::RGB32_UINT:           return TEX_FORMAT_RGB32_UINT;
        case nrd::Format::RGB32_SINT:           return TEX_FORMAT_RGB32_SINT;
        case nrd::Format::RGB32_SFLOAT:         return TEX_FORMAT_RGB32_FLOAT;
        case nrd::Format::RGBA32_UINT:          return TEX_FORMAT_RGBA32_UINT;
        case nrd::Format::RGBA32_SINT:          return TEX_FORMAT_RGBA32_SINT;
        case nrd::Format::RGBA32_SFLOAT:        return TEX_FORMAT_RGBA32_FLOAT;
        case nrd::Format::R10_G10_B10_A2_UNORM: return TEX_FORMAT_RGB10A2_UNORM;
        case nrd::Format::R11_G11_B10_UFLOAT:   return TEX_FORMAT_R11G11B10_FLOAT;
        default:                                return TEX_FORMAT_UNKNOWN;
    }
}

} // namespace
```

- [ ] **Step 2: Implement reset and initialization flow**

Add this after the helper namespace:

```cpp
void RTXPTNrdIntegration::Reset()
{
    if (m_Instance != nullptr)
    {
        nrd::DestroyInstance(*m_Instance);
        m_Instance = nullptr;
    }

    m_Device.Release();
    m_ConstantBuffer.Release();
    m_Pipelines.clear();
    m_Samplers.clear();
    m_PermanentTextures.clear();
    m_TransientTextures.clear();
    m_Stats = {};
}

bool RTXPTNrdIntegration::Initialize(IRenderDevice*  pDevice,
                                     IEngineFactory* pEngineFactory,
                                     RTXPTNrdMethod  Method,
                                     Uint32          Width,
                                     Uint32          Height,
                                     bool            ComputeSupported)
{
    Reset();
    m_Stats.Method = Method;
    m_Stats.Width  = Width;
    m_Stats.Height = Height;

    if (!ComputeSupported)
        return Fail("RTXPT NRD requires compute shader support");
    if (pDevice == nullptr || pEngineFactory == nullptr)
        return Fail("RTXPT NRD requires a device and engine factory");
    if (Width == 0 || Height == 0)
        return Fail("RTXPT NRD requires a non-zero render size");

    m_Device = pDevice;
    if (!CreateInstance(Method) ||
        !CreateConstantBuffer(pDevice) ||
        !CreateSamplers(pDevice) ||
        !CreatePipelines(pDevice, pEngineFactory) ||
        !CreatePoolTextures(pDevice, Width, Height))
        return false;

    m_Stats.Ready = true;
    return true;
}
```

- [ ] **Step 3: Implement instance, constant buffer, and samplers**

```cpp
bool RTXPTNrdIntegration::CreateInstance(RTXPTNrdMethod Method)
{
    const nrd::DenoiserDesc DenoiserDescs[] =
    {
        {m_Identifier, RTXPTToNrdDenoiser(Method)}
    };

    nrd::InstanceCreationDesc Desc = {};
    Desc.allocationCallbacks.Allocate   = NrdAllocate;
    Desc.allocationCallbacks.Reallocate = NrdReallocate;
    Desc.allocationCallbacks.Free       = NrdFree;
    Desc.denoisers                      = DenoiserDescs;
    Desc.denoisersNum                   = 1;

    const nrd::Result Result = nrd::CreateInstance(Desc, m_Instance);
    return Result == nrd::Result::SUCCESS && m_Instance != nullptr ?
        true :
        Fail("Failed to create NRD instance");
}

bool RTXPTNrdIntegration::CreateConstantBuffer(IRenderDevice* pDevice)
{
    const nrd::InstanceDesc& InstanceDesc = nrd::GetInstanceDesc(*m_Instance);

    BufferDesc Desc;
    Desc.Name           = "RTXPT NRD constants";
    Desc.Size           = InstanceDesc.constantBufferMaxDataSize;
    Desc.BindFlags      = BIND_UNIFORM_BUFFER;
    Desc.Usage          = USAGE_DYNAMIC;
    Desc.CPUAccessFlags = CPU_ACCESS_WRITE;

    pDevice->CreateBuffer(Desc, nullptr, &m_ConstantBuffer);
    return m_ConstantBuffer ? true : Fail("Failed to create NRD constant buffer");
}

bool RTXPTNrdIntegration::CreateSamplers(IRenderDevice* pDevice)
{
    const nrd::InstanceDesc& InstanceDesc = nrd::GetInstanceDesc(*m_Instance);
    m_Samplers.resize(InstanceDesc.samplersNum);

    for (Uint32 SamplerIndex = 0; SamplerIndex < InstanceDesc.samplersNum; ++SamplerIndex)
    {
        SamplerDesc Desc;
        Desc.Name = InstanceDesc.samplers[SamplerIndex] == nrd::Sampler::LINEAR_CLAMP ?
            "RTXPT NRD linear clamp sampler" :
            "RTXPT NRD nearest clamp sampler";
        Desc.MinFilter = InstanceDesc.samplers[SamplerIndex] == nrd::Sampler::LINEAR_CLAMP ? FILTER_TYPE_LINEAR : FILTER_TYPE_POINT;
        Desc.MagFilter = Desc.MinFilter;
        Desc.MipFilter = Desc.MinFilter;
        Desc.AddressU  = TEXTURE_ADDRESS_CLAMP;
        Desc.AddressV  = TEXTURE_ADDRESS_CLAMP;
        Desc.AddressW  = TEXTURE_ADDRESS_CLAMP;

        pDevice->CreateSampler(Desc, &m_Samplers[SamplerIndex]);
        if (!m_Samplers[SamplerIndex])
            return Fail("Failed to create NRD sampler");
    }

    return true;
}
```

- [ ] **Step 4: Implement pipeline and pool texture creation**

```cpp
bool RTXPTNrdIntegration::CreatePipelines(IRenderDevice* pDevice, IEngineFactory* pEngineFactory)
{
    const nrd::InstanceDesc& InstanceDesc = nrd::GetInstanceDesc(*m_Instance);
    m_Pipelines.resize(InstanceDesc.pipelinesNum);

    std::string SearchDirs = std::string{RTXPT_NRD_SHADER_SOURCE_DIR} + ";" +
        std::string{RTXPT_NRD_SHADER_INCLUDE_DIR};

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory(SearchDirs.c_str(), &pShaderSourceFactory);

    for (Uint32 PipelineIndex = 0; PipelineIndex < InstanceDesc.pipelinesNum; ++PipelineIndex)
    {
        const nrd::PipelineDesc& NrdPipeline = InstanceDesc.pipelines[PipelineIndex];

        ShaderMacroHelper Macros;
        Macros.Add("NRD_COMPILER_DXC", 1);
        Macros.Add("NRD_NORMAL_ENCODING", 2);
        Macros.Add("NRD_ROUGHNESS_ENCODING", 1);
        Macros.Add("NRD_CONSTANT_BUFFER_REGISTER_INDEX", InstanceDesc.constantBufferRegisterIndex);
        Macros.Add("NRD_SAMPLERS_BASE_REGISTER_INDEX", InstanceDesc.samplersBaseRegisterIndex);
        Macros.Add("NRD_RESOURCES_BASE_REGISTER_INDEX", InstanceDesc.resourcesBaseRegisterIndex);
        Macros.Add("NRD_ROOT_SPACE_INDEX", InstanceDesc.rootSpaceIndex);
        Macros.Add("NRD_RESOURCES_SPACE_INDEX", InstanceDesc.resourcesSpaceIndex);

        ShaderCreateInfo ShaderCI;
        ShaderCI.Desc.ShaderType            = SHADER_TYPE_COMPUTE;
        ShaderCI.Desc.Name                  = NrdPipeline.shaderFileName;
        ShaderCI.SourceLanguage             = SHADER_SOURCE_LANGUAGE_HLSL;
        ShaderCI.ShaderCompiler             = SHADER_COMPILER_DXC;
        ShaderCI.CompileFlags               = SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR;
        ShaderCI.FilePath                   = NrdPipeline.shaderFileName;
        ShaderCI.EntryPoint                 = NrdPipeline.shaderEntryPointName != nullptr ? NrdPipeline.shaderEntryPointName : "main";
        ShaderCI.Macros                     = Macros;
        ShaderCI.pShaderSourceStreamFactory = pShaderSourceFactory;

        RefCntAutoPtr<IShader> pCS;
        pDevice->CreateShader(ShaderCI, &pCS);
        if (!pCS)
            return Fail("Failed to create NRD compute shader");

        ComputePipelineStateCreateInfo PSOCreateInfo;
        PSOCreateInfo.PSODesc.Name         = NrdPipeline.shaderFileName;
        PSOCreateInfo.PSODesc.PipelineType = PIPELINE_TYPE_COMPUTE;
        PSOCreateInfo.pCS                  = pCS;
        PSOCreateInfo.PSODesc.ResourceLayout.DefaultVariableType = SHADER_RESOURCE_VARIABLE_TYPE_DYNAMIC;

        pDevice->CreateComputePipelineState(PSOCreateInfo, &m_Pipelines[PipelineIndex].PSO);
        if (!m_Pipelines[PipelineIndex].PSO)
            return Fail("Failed to create NRD compute PSO");

        m_Pipelines[PipelineIndex].PSO->CreateShaderResourceBinding(&m_Pipelines[PipelineIndex].SRB, true);
        if (!m_Pipelines[PipelineIndex].SRB)
            return Fail("Failed to create NRD shader resource binding");
    }

    return true;
}

bool RTXPTNrdIntegration::CreatePoolTextures(IRenderDevice* pDevice, Uint32 Width, Uint32 Height)
{
    const nrd::InstanceDesc& InstanceDesc = nrd::GetInstanceDesc(*m_Instance);
    m_PermanentTextures.resize(InstanceDesc.permanentPoolSize);
    m_TransientTextures.resize(InstanceDesc.transientPoolSize);

    auto CreatePoolTexture = [&](const nrd::TextureDesc& NrdDesc, const char* PoolName, Uint32 Index, RefCntAutoPtr<ITexture>& Texture) {
        const TEXTURE_FORMAT Format = ToDiligentFormat(NrdDesc.format);
        if (Format == TEX_FORMAT_UNKNOWN)
            return Fail("Unsupported NRD pool texture format");

        std::ostringstream Name;
        Name << "RTXPT NRD " << PoolName << " texture " << Index;

        TextureDesc Desc;
        Desc.Name      = Name.str().c_str();
        Desc.Type      = RESOURCE_DIM_TEX_2D;
        Desc.Width     = std::max(Width / static_cast<Uint32>(NrdDesc.downsampleFactor), 1u);
        Desc.Height    = std::max(Height / static_cast<Uint32>(NrdDesc.downsampleFactor), 1u);
        Desc.Format    = Format;
        Desc.BindFlags = BIND_SHADER_RESOURCE | BIND_UNORDERED_ACCESS;

        pDevice->CreateTexture(Desc, nullptr, &Texture);
        return Texture ? true : Fail("Failed to create NRD pool texture");
    };

    for (Uint32 Index = 0; Index < InstanceDesc.permanentPoolSize; ++Index)
    {
        if (!CreatePoolTexture(InstanceDesc.permanentPool[Index], "permanent", Index, m_PermanentTextures[Index]))
            return false;
    }

    for (Uint32 Index = 0; Index < InstanceDesc.transientPoolSize; ++Index)
    {
        if (!CreatePoolTexture(InstanceDesc.transientPool[Index], "transient", Index, m_TransientTextures[Index]))
            return false;
    }

    return true;
}
```

- [ ] **Step 5: Configure and build with the default NRD/NRI submodules**

Use the RTXPT sample submodules without passing explicit roots:

```powershell
cmake -S . -B build\rtxpt-g8-with-nrd -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON
cmake --build build\rtxpt-g8-with-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 6: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp
git commit -m "feat(rtxpt): create NRD instance resources"
```

---

### Task 5: Implement NRD Common Settings and Dispatch Resource Mapping

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp`

- [ ] **Step 1: Add common settings population**

```cpp
void RTXPTNrdIntegration::PopulateCommonSettings(nrd::CommonSettings& Settings, const RTXPTNrdFrameAttribs& Attribs) const
{
    const SampleConstants& Constants = *Attribs.pFrameConstants;
    const PathTracerViewData& View   = Constants.view;
    const PathTracerViewData& Prev   = Constants.previousView;

    auto CopyMatrix = [](float* Dst, const float4x4& Src) {
        for (Uint32 Row = 0; Row < 4; ++Row)
        {
            for (Uint32 Col = 0; Col < 4; ++Col)
                Dst[Row * 4 + Col] = Src.m[Row][Col];
        }
    };

    CopyMatrix(Settings.worldToViewMatrix, View.MatWorldToView);
    CopyMatrix(Settings.worldToViewMatrixPrev, Prev.MatWorldToView);
    CopyMatrix(Settings.viewToClipMatrix, View.MatViewToClip);
    CopyMatrix(Settings.viewToClipMatrixPrev, Prev.MatViewToClip);

    Settings.isMotionVectorInWorldSpace = false;
    Settings.motionVectorScale[0]       = Attribs.pRenderTargets->GetRenderWidth() != 0 ? 1.0f / static_cast<float>(Attribs.pRenderTargets->GetRenderWidth()) : 0.0f;
    Settings.motionVectorScale[1]       = Attribs.pRenderTargets->GetRenderHeight() != 0 ? 1.0f / static_cast<float>(Attribs.pRenderTargets->GetRenderHeight()) : 0.0f;
    Settings.motionVectorScale[2]       = 1.0f;
    Settings.cameraJitter[0]            = View.PixelOffset.x;
    Settings.cameraJitter[1]            = View.PixelOffset.y;
    Settings.cameraJitterPrev[0]        = Prev.PixelOffset.x;
    Settings.cameraJitterPrev[1]        = Prev.PixelOffset.y;
    Settings.frameIndex                 = Attribs.FrameIndex;
    Settings.denoisingRange             = 20000.0f;
    Settings.enableValidation           = Attribs.EnableValidation && Attribs.pRenderTargets->GetDenoiserOutValidationUAV() != nullptr;
    Settings.disocclusionThreshold      = Attribs.pRealtime->NRDDisocclusionThreshold;
    Settings.disocclusionThresholdAlternate = Attribs.pRealtime->NRDDisocclusionThresholdAlternate;
    Settings.isDisocclusionThresholdMixAvailable = Attribs.pRealtime->NRDUseAlternateDisocclusionThresholdMix;
    Settings.timeDeltaBetweenFrames     = Attribs.TimeDeltaSeconds;
    Settings.accumulationMode           = Attribs.ResetHistory ? nrd::AccumulationMode::CLEAR_AND_RESTART : nrd::AccumulationMode::CONTINUE;
    Settings.resourceSize[0]            = Attribs.pRenderTargets->GetRenderWidth();
    Settings.resourceSize[1]            = Attribs.pRenderTargets->GetRenderHeight();
    Settings.resourceSizePrev[0]        = Settings.resourceSize[0];
    Settings.resourceSizePrev[1]        = Settings.resourceSize[1];
    Settings.rectSize[0]                = Settings.resourceSize[0];
    Settings.rectSize[1]                = Settings.resourceSize[1];
    Settings.rectSizePrev[0]            = Settings.resourceSize[0];
    Settings.rectSizePrev[1]            = Settings.resourceSize[1];
}
```

- [ ] **Step 2: Add resource mapping helper inside `BindDispatchResources`**

```cpp
bool RTXPTNrdIntegration::BindDispatchResources(IDeviceContext*              pContext,
                                                const nrd::DispatchDesc&     DispatchDesc,
                                                const nrd::PipelineDesc&     PipelineDesc,
                                                PipelineState&               Pipeline,
                                                const RTXPTNrdFrameAttribs&  Attribs)
{
    const RTXPTRenderTargets& RenderTargets = *Attribs.pRenderTargets;

    auto GetView = [&](const nrd::ResourceDesc& Resource, TEXTURE_VIEW_TYPE ViewType) -> IDeviceObject* {
        switch (Resource.type)
        {
            case nrd::ResourceType::IN_MV:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserMotionVectorsSRV() : RenderTargets.GetDenoiserMotionVectorsUAV();
            case nrd::ResourceType::IN_NORMAL_ROUGHNESS:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserNormalRoughnessSRV() : RenderTargets.GetDenoiserNormalRoughnessUAV();
            case nrd::ResourceType::IN_VIEWZ:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserViewspaceZSRV() : RenderTargets.GetDenoiserViewspaceZUAV();
            case nrd::ResourceType::IN_SPEC_RADIANCE_HITDIST:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserSpecRadianceHitDistSRV() : RenderTargets.GetDenoiserSpecRadianceHitDistUAV();
            case nrd::ResourceType::IN_DIFF_RADIANCE_HITDIST:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserDiffRadianceHitDistSRV() : RenderTargets.GetDenoiserDiffRadianceHitDistUAV();
            case nrd::ResourceType::IN_DISOCCLUSION_THRESHOLD_MIX:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserDisocclusionThresholdMixSRV() : RenderTargets.GetDenoiserDisocclusionThresholdMixUAV();
            case nrd::ResourceType::OUT_SPEC_RADIANCE_HITDIST:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserOutSpecRadianceHitDistSRV(Attribs.PlaneIndex) : RenderTargets.GetDenoiserOutSpecRadianceHitDistUAV(Attribs.PlaneIndex);
            case nrd::ResourceType::OUT_DIFF_RADIANCE_HITDIST:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserOutDiffRadianceHitDistSRV(Attribs.PlaneIndex) : RenderTargets.GetDenoiserOutDiffRadianceHitDistUAV(Attribs.PlaneIndex);
            case nrd::ResourceType::OUT_VALIDATION:
                return ViewType == TEXTURE_VIEW_SHADER_RESOURCE ? RenderTargets.GetDenoiserOutValidationSRV() : RenderTargets.GetDenoiserOutValidationUAV();
            case nrd::ResourceType::TRANSIENT_POOL:
                return Resource.indexInPool < m_TransientTextures.size() && m_TransientTextures[Resource.indexInPool] ?
                    m_TransientTextures[Resource.indexInPool]->GetDefaultView(ViewType) : nullptr;
            case nrd::ResourceType::PERMANENT_POOL:
                return Resource.indexInPool < m_PermanentTextures.size() && m_PermanentTextures[Resource.indexInPool] ?
                    m_PermanentTextures[Resource.indexInPool]->GetDefaultView(ViewType) : nullptr;
            default:
                return nullptr;
        }
    };

    Uint32 ResourceIndex = 0;
    for (Uint32 RangeIndex = 0; RangeIndex < PipelineDesc.resourceRangesNum; ++RangeIndex)
    {
        const nrd::ResourceRangeDesc& Range = PipelineDesc.resourceRanges[RangeIndex];
        const TEXTURE_VIEW_TYPE ViewType = Range.descriptorType == nrd::DescriptorType::TEXTURE ?
            TEXTURE_VIEW_SHADER_RESOURCE :
            TEXTURE_VIEW_UNORDERED_ACCESS;

        for (Uint32 Offset = 0; Offset < Range.descriptorsNum; ++Offset, ++ResourceIndex)
        {
            if (ResourceIndex >= DispatchDesc.resourcesNum)
                return Fail("NRD dispatch resource range is inconsistent");

            IDeviceObject* pView = GetView(DispatchDesc.resources[ResourceIndex], ViewType);
            if (pView == nullptr)
                return Fail("NRD dispatch requested an unavailable RTXPT texture view");

            IShaderResourceVariable* pVar = Pipeline.SRB->GetVariableByIndex(SHADER_TYPE_COMPUTE, ResourceIndex);
            if (pVar == nullptr)
                return Fail("NRD shader resource variable is missing");
            pVar->Set(pView, SET_SHADER_RESOURCE_FLAG_ALLOW_OVERWRITE);
        }
    }

    if (ResourceIndex != DispatchDesc.resourcesNum)
        return Fail("NRD dispatch resource count mismatch");

    pContext->CommitShaderResources(Pipeline.SRB, RESOURCE_STATE_TRANSITION_MODE_TRANSITION);
    return true;
}
```

If `GetVariableByIndex` order does not match Diligent shader reflection order, replace this exact block with named binding using reflected names from the NRD resource HLSLI files, and keep the same `nrd::ResourceType` mapping table above.

- [ ] **Step 3: Add the dispatch method**

```cpp
bool RTXPTNrdIntegration::Dispatch(IDeviceContext* pContext, const RTXPTNrdFrameAttribs& Attribs)
{
    m_Stats.LastDispatchExecuted = false;
    m_Stats.LastDispatches       = 0;
    m_Stats.LastPlaneIndex       = Attribs.PlaneIndex;

    if (!IsReady() || pContext == nullptr || Attribs.pRenderTargets == nullptr || Attribs.pFrameConstants == nullptr || Attribs.pRealtime == nullptr)
        return Fail("RTXPT NRD dispatch is missing required attributes");
    if (Attribs.PlaneIndex >= kRTXPTStablePlaneCount)
        return Fail("RTXPT NRD plane index is out of range");

    const void* MethodSettings = nullptr;
    nrd::RelaxSettings RelaxSettings;
    nrd::ReblurSettings ReblurSettings;
    if (Attribs.pRealtime->NRDMethod == RTXPTNrdMethod::RELAX)
    {
        RelaxSettings  = RTXPTMakeRelaxSettings(Attribs.pRealtime->RelaxSettings);
        MethodSettings = &RelaxSettings;
    }
    else
    {
        ReblurSettings = RTXPTMakeReblurSettings(Attribs.pRealtime->ReblurSettings);
        MethodSettings = &ReblurSettings;
    }

    nrd::SetDenoiserSettings(*m_Instance, m_Identifier, MethodSettings);

    nrd::CommonSettings CommonSettings = {};
    PopulateCommonSettings(CommonSettings, Attribs);
    nrd::SetCommonSettings(*m_Instance, CommonSettings);

    const nrd::DispatchDesc* DispatchDescs = nullptr;
    Uint32 DispatchCount = 0;
    nrd::GetComputeDispatches(*m_Instance, &m_Identifier, 1, DispatchDescs, DispatchCount);

    const nrd::InstanceDesc& InstanceDesc = nrd::GetInstanceDesc(*m_Instance);
    for (Uint32 DispatchIndex = 0; DispatchIndex < DispatchCount; ++DispatchIndex)
    {
        const nrd::DispatchDesc& DispatchDesc = DispatchDescs[DispatchIndex];
        if (DispatchDesc.pipelineIndex >= m_Pipelines.size())
            return Fail("NRD dispatch pipeline index is out of range");

        PipelineState& Pipeline = m_Pipelines[DispatchDesc.pipelineIndex];
        if (!Pipeline.PSO || !Pipeline.SRB)
            return Fail("NRD dispatch pipeline is not initialized");

        if (DispatchDesc.constantBufferData != nullptr && DispatchDesc.constantBufferDataSize != 0)
        {
            MapHelper<Uint8> Constants{pContext, m_ConstantBuffer, MAP_WRITE, MAP_FLAG_DISCARD};
            std::memcpy(Constants, DispatchDesc.constantBufferData, DispatchDesc.constantBufferDataSize);
        }

        IShaderResourceVariable* pConstants = Pipeline.SRB->GetVariableByName(SHADER_TYPE_COMPUTE, "REBLUR_TemporalAccumulationConstants");
        if (pConstants != nullptr)
            pConstants->Set(m_ConstantBuffer, SET_SHADER_RESOURCE_FLAG_ALLOW_OVERWRITE);

        for (Uint32 SamplerIndex = 0; SamplerIndex < m_Samplers.size(); ++SamplerIndex)
        {
            IShaderResourceVariable* pSamplerVar = Pipeline.SRB->GetVariableByIndex(SHADER_TYPE_COMPUTE, DispatchDesc.resourcesNum + SamplerIndex);
            if (pSamplerVar != nullptr)
                pSamplerVar->Set(m_Samplers[SamplerIndex], SET_SHADER_RESOURCE_FLAG_ALLOW_OVERWRITE);
        }

        if (!BindDispatchResources(pContext, DispatchDesc, InstanceDesc.pipelines[DispatchDesc.pipelineIndex], Pipeline, Attribs))
            return false;

        pContext->SetPipelineState(Pipeline.PSO);
        DispatchComputeAttribs DispatchAttribs;
        DispatchAttribs.ThreadGroupCountX = DispatchDesc.gridWidth;
        DispatchAttribs.ThreadGroupCountY = DispatchDesc.gridHeight;
        DispatchAttribs.ThreadGroupCountZ = 1;
        pContext->DispatchCompute(DispatchAttribs);
        ++m_Stats.LastDispatches;
    }

    m_Stats.LastDispatchExecuted = DispatchCount != 0;
    m_Stats.DispatchCount += DispatchCount;
    return true;
}

} // namespace Diligent

#endif
```

After implementing, inspect shader reflection failures. NRD shaders use different constant buffer names per pass, so if the named constant buffer binding above misses most passes, replace it with an SRB variable-index pass that skips texture/sampler variables only after confirming Diligent reflection order. Keep this as a focused adjustment inside `RTXPTNrdIntegration.cpp`.

- [ ] **Step 4: Build with NRD and fix reflection binding issues in this file only**

Run:

```powershell
cmake --build build\rtxpt-g8-with-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTNrdIntegration.cpp
git commit -m "feat(rtxpt): dispatch NRD compute passes"
```

---

### Task 6: Enable NRD Headers for G7 Post-Process Shaders

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp`

- [ ] **Step 1: Add NRD shader include path to the post-process shader factory**

Find the existing `CreateDefaultShaderSourceStreamFactory` call in `RTXPTPostProcessPass::Initialize`. Replace its search string construction with:

```cpp
    std::string ShaderSearchPaths = "shaders;shaders\\PostProcessing";
#if RTXPT_HAS_NRD
    if (RTXPT_NRD_SHADER_INCLUDE_DIR[0] != '\0')
    {
        ShaderSearchPaths += ";";
        ShaderSearchPaths += RTXPT_NRD_SHADER_INCLUDE_DIR;
    }
#endif

    RefCntAutoPtr<IShaderSourceInputStreamFactory> pShaderSourceFactory;
    pEngineFactory->CreateDefaultShaderSourceStreamFactory(ShaderSearchPaths.c_str(), &pShaderSourceFactory);
```

- [ ] **Step 2: Add shader macro to `CreatePostProcessPSO`**

After:

```cpp
    Macros.Add("RTXPT_POST_PROCESS_MODE", GetModeMacro(Pass));
```

Add:

```cpp
    Macros.Add("RTXPT_HAS_NRD_HEADERS", RTXPT_HAS_NRD ? 1 : 0);
```

- [ ] **Step 3: Build without and with NRD**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
cmake --build build\rtxpt-g8-with-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
Build succeeded.
```

- [ ] **Step 4: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTPostProcessPass.cpp
git commit -m "feat(rtxpt): enable NRD post-process shader helpers"
```

---

### Task 7: Add Sample Ownership, UI Availability, and Lifecycle Resets

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp`
- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Include the NRD integration header**

In `RTXPTSample.hpp`, add:

```cpp
#include "RTXPTNrdIntegration.hpp"
```

- [ ] **Step 2: Add private helper declarations**

In the private method list near realtime post-process helpers, add:

```cpp
    bool Denoise();
    bool EnsureNrdIntegrations();
    void ResetNrdIntegrations();
```

- [ ] **Step 3: Add per-plane integration storage**

Near `m_DenoisingGuidesBaker`, add:

```cpp
    std::array<RTXPTNrdIntegration, kRTXPTStablePlaneCount> m_NrdIntegrations;
```

- [ ] **Step 4: Include `<array>` if needed**

If `RTXPTSample.hpp` does not already include `<array>`, add:

```cpp
#include <array>
```

- [ ] **Step 5: Replace local availability constants in `RTXPTSample.cpp`**

Find:

```cpp
constexpr bool        kRTXPTStandaloneNrdAvailable = false;
constexpr const char* kRTXPTNrdDisabledReason      = "Standalone denoiser disabled: NRD integration starts in G8.";
```

Replace availability checks with functions:

```cpp
bool IsStandaloneNrdAvailable()
{
    return RTXPTIsNrdAvailable();
}

const char* GetStandaloneNrdDisabledReason()
{
    return RTXPTGetNrdUnavailableReason();
}
```

Then replace each `kRTXPTStandaloneNrdAvailable` use with `IsStandaloneNrdAvailable()` and each `kRTXPTNrdDisabledReason` use with `GetStandaloneNrdDisabledReason()`.

- [ ] **Step 6: Reset NRD instances on render-target recreate**

In `RTXPTSample::EnsureRenderTargets`, after `RequestRealtimeReset(...)` inside the `if (Ok && ResourcesValid && ...)` block, add:

```cpp
        ResetNrdIntegrations();
```

- [ ] **Step 7: Reset NRD instances when method changes**

In the UI block where `m_RealtimeUI.NRDMethod` changes, after:

```cpp
            RequestRealtimeReset(RTXPT_REALTIME_RESET_NRD_HISTORY, "NRD mode changed");
```

Add:

```cpp
            ResetNrdIntegrations();
```

- [ ] **Step 8: Implement `ResetNrdIntegrations` and `EnsureNrdIntegrations`**

Add these functions before `RunRealtimePostProcess()`:

```cpp
void RTXPTSample::ResetNrdIntegrations()
{
    for (RTXPTNrdIntegration& Integration : m_NrdIntegrations)
        Integration.Reset();
}

bool RTXPTSample::EnsureNrdIntegrations()
{
    if (!IsStandaloneNrdAvailable())
    {
        RecordRealtimePathTraceStatus(GetStandaloneNrdDisabledReason());
        return false;
    }

    const Uint32 Width  = m_RenderTargets.GetRenderWidth();
    const Uint32 Height = m_RenderTargets.GetRenderHeight();
    for (RTXPTNrdIntegration& Integration : m_NrdIntegrations)
    {
        const bool NeedsRecreate =
            !Integration.IsReady() ||
            Integration.GetMethod() != m_RealtimeUI.NRDMethod ||
            Integration.GetWidth() != Width ||
            Integration.GetHeight() != Height;

        if (NeedsRecreate &&
            !Integration.Initialize(m_pDevice, m_pEngineFactory, m_RealtimeUI.NRDMethod, Width, Height, m_FeatureCaps.ComputeShaders))
        {
            RecordRealtimePathTraceStatus(Integration.GetLastFailureReason());
            ResetNrdIntegrations();
            return false;
        }
    }

    return true;
}
```

- [ ] **Step 9: Build without NRD**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 10: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.hpp DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): own per-plane NRD integrations"
```

---

### Task 8: Wire Realtime `Denoise` Orchestration

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp`

- [ ] **Step 1: Add the `Denoise` method**

Add this before `RunRealtimePostProcess()`:

```cpp
bool RTXPTSample::Denoise()
{
    if (!m_RealtimeUI.ActualUseStandaloneDenoiser())
        return true;

    if (m_RealtimeUI.RealtimeAA == RTXPTRealtimeAAMode::DLSSRR)
    {
        RecordRealtimePathTraceStatus("Standalone NRD is disabled for DLSS-RR");
        return false;
    }

    if (!EnsureNrdIntegrations())
        return false;

    const bool ResetHistory =
        HasRealtimeResetFlag(m_CurrentFrameRealtimeReset, RTXPT_REALTIME_RESET_NRD_HISTORY) ||
        HasRealtimeResetFlag(m_CurrentFrameRealtimeReset, RTXPT_REALTIME_RESET_REALTIME_CACHES);
    const bool EnableValidation = m_RenderTargets.GetDenoiserOutValidationUAV() != nullptr;

    Int32 MaxPassCount = std::min(m_RealtimeUI.StablePlanesActiveCount, static_cast<Int32>(kRTXPTStablePlaneCount));
    MaxPassCount       = std::max(MaxPassCount, Int32{1});

    bool InitWithStableRadiance = true;
    for (Int32 Pass = MaxPassCount - 1; Pass >= 0; --Pass)
    {
        const Uint32 PlaneIndex = static_cast<Uint32>(Pass);
        if (!m_PostProcessPipeline.RunDenoiserPrepare(m_pImmediateContext,
                                                      m_RenderTargets,
                                                      m_RealtimeUI.NRDMethod,
                                                      PlaneIndex,
                                                      InitWithStableRadiance))
        {
            RecordRealtimePathTraceStatus("Denoiser prepare dispatch failed");
            return false;
        }

        InitWithStableRadiance = false;
        InsertDenoiserPrepareOutputBarriers(m_pImmediateContext, m_RenderTargets);

        RTXPTNrdFrameAttribs Attribs;
        Attribs.pRenderTargets   = &m_RenderTargets;
        Attribs.pFrameConstants  = &m_LastFrameConstants;
        Attribs.pRealtime        = &m_RealtimeUI;
        Attribs.PlaneIndex       = PlaneIndex;
        Attribs.FrameIndex       = m_FrameIndex;
        Attribs.TimeDeltaSeconds = m_LastElapsedTimeSeconds > 0.0f ? m_LastElapsedTimeSeconds : -1.0f;
        Attribs.ResetHistory     = ResetHistory;
        Attribs.EnableValidation = EnableValidation;

        if (!m_NrdIntegrations[PlaneIndex].Dispatch(m_pImmediateContext, Attribs))
        {
            RecordRealtimePathTraceStatus(m_NrdIntegrations[PlaneIndex].GetLastFailureReason());
            return false;
        }

        const bool HasValidation = m_RenderTargets.GetDenoiserOutValidationSRV() != nullptr;
        if (!m_PostProcessPipeline.RunDenoiserFinalMerge(m_pImmediateContext,
                                                         m_RenderTargets,
                                                         m_RealtimeUI.NRDMethod,
                                                         PlaneIndex,
                                                         HasValidation))
        {
            RecordRealtimePathTraceStatus("Denoiser final merge dispatch failed");
            return false;
        }

        InsertDenoiserFinalMergeOutputBarrier(m_pImmediateContext, m_RenderTargets);
    }

    m_LastRealtimeFinalMergeReady = true;
    RecordRealtimePathTraceStatus("Realtime PathTrace, NRD denoise, and final merge dispatched");
    return true;
}
```

- [ ] **Step 2: Replace the G8 stub in `RunRealtimePostProcess`**

Replace the current `UseStandaloneDenoiser = false` branch with:

```cpp
bool RTXPTSample::RunRealtimePostProcess()
{
    if (m_RealtimeUI.ActualUseStandaloneDenoiser())
    {
        if (!Denoise())
            return false;
        return PresentRealtimeFinalOutput();
    }

    if (!RunRealtimeNoDenoiserFinalMerge())
        return false;

    return PresentRealtimeFinalOutput();
}
```

- [ ] **Step 3: Add an explicit no-NRD fallback when UI cannot enable standalone denoiser**

In the UI logic that disables the standalone denoiser checkbox, keep `m_RealtimeUI.StandaloneDenoiser` untouched if the user selected it while NRD was unavailable, but `ActualUseStandaloneDenoiser()` must stay false because the checkbox is disabled. Runtime still follows:

```cpp
if (!m_RealtimeUI.ActualUseStandaloneDenoiser())
    RunRealtimeNoDenoiserFinalMerge();
```

This preserves the spec requirement that no-NRD builds fall back to `NoDenoiserFinalMerge`.

- [ ] **Step 4: Build without and with NRD**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
cmake --build build\rtxpt-g8-with-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
Build succeeded.
```

- [ ] **Step 5: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp
git commit -m "feat(rtxpt): orchestrate realtime NRD denoise"
```

---

### Task 9: Update Mapping Docs and Run Final Verification

**Files:**

- Modify: `DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md`

- [ ] **Step 1: Add G8/G9 mapping rows**

Append these rows near the existing G7 denoiser mapping:

```markdown
| `NRD/NrdConfig.{h,cpp}` | `src/RTXPTNrdConfig.{hpp,cpp}` | Realtime G8 | Converts Diligent realtime UI fields into NRD REBLUR/RELAX settings behind the `thirdparty/NRD` + `thirdparty/NRI` default roots, with `RTXPT_NRD_ROOT`/`RTXPT_NRI_ROOT` override support. |
| `NRD/NrdIntegration.{h,cpp}` | `src/RTXPTNrdIntegration.{hpp,cpp}` | Realtime G8 | Diligent-native NRD instance, compute PSO/SRB, sampler, pool texture, common settings, and dispatch resource mapping for one stable plane. |
| `Sample.cpp::Denoise` | `src/RTXPTSample.cpp::Denoise` | Realtime G9 | Runs prepare -> NRD -> final merge from highest active stable plane down to plane 0, initializing stable radiance only on the first processed plane. |
| `Sample.h::m_nrd` | `src/RTXPTSample.hpp::m_NrdIntegrations` | Realtime G9 | One `RTXPTNrdIntegration` per stable plane, reset on render-target recreate or NRD method switch. |
```

- [ ] **Step 2: Source-scan for forbidden DLSS-RR execution**

Run:

```powershell
rg -n "DLSSRR|RealtimeAA == RTXPTRealtimeAAMode::DLSSRR|ActualUseStandaloneDenoiser|Denoise\\(" DiligentSamples\Samples\RTXPT\src DiligentSamples\Samples\RTXPT\assets\shaders
```

Expected:

```text
Matches show DLSSRR guarded out of standalone NRD, and Denoise is called only from realtime post-process orchestration.
```

- [ ] **Step 3: Build without NRD**

Run:

```powershell
cmake --build build\rtxpt-g8-no-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 4: Build with default NRD/NRI submodules**

Run:

```powershell
cmake --build build\rtxpt-g8-with-nrd --config Debug --target RTXPT
```

Expected:

```text
Build succeeded.
```

- [ ] **Step 5: Run targeted format validation**

Run from the DiligentSamples format-validation folder:

```powershell
Push-Location DiligentSamples\BuildTools\FormatValidation
.\validate_format_win.bat
Pop-Location
```

Expected:

```text
No formatting violations for the touched RTXPT files.
```

- [ ] **Step 6: Manual smoke checks in the RTXPT sample**

Run the sample from the Debug output created by the local build. In the UI:

```text
1. Realtime mode on, standalone denoiser off: image reaches final presentation through NoDenoiserFinalMerge.
2. Realtime mode on, standalone denoiser on, NRD method REBLUR: prepare, NRD dispatch, and final merge run without missing texture-view errors.
3. Switch NRD method to RELAX: old instances are destroyed, new RELAX instances initialize, output presents.
4. Resize the window: NRD instances are recreated for the new render size.
5. Press reset realtime caches: NRD uses CLEAR_AND_RESTART for one frame, then CONTINUE.
6. Set RealtimeAA to DLSSRR: standalone NRD stays disabled and no NRD dispatch occurs.
```

- [ ] **Step 7: Commit**

```bash
git add DiligentSamples/Samples/RTXPT/RTXPT_FORK_MAPPING.md
git commit -m "docs(rtxpt): map NRD denoise integration"
```

---

## Self-Review

- [ ] **Spec coverage:** G8 is covered by Tasks 1-6: optional dependency gate, NRD instance creation, shaders/PSOs, samplers, pool textures, texture format conversion, resource mapping, common settings, and unavailable fallback. G9 is covered by Tasks 7-8: sample ownership, reset/recreate lifecycle, per-plane prepare -> NRD -> merge, highest-to-lowest order, and DLSS-RR exclusion.
- [ ] **Red-flag scan:** Search this plan for vague filler phrases listed in the writing-plans skill and replace any match with concrete instructions.
- [ ] **Type consistency:** Confirm `RTXPTNrdMethod`, `RTXPTRealtimeSettings`, `RTXPTRenderTargets`, `SampleConstants`, `PathTracerViewData::MatWorldToView`, `PathTracerViewData::MatViewToClip`, `PathTracerViewData::PixelOffset`, and all `GetDenoiser*SRV/UAV` names match the current source before implementation.
- [ ] **Risk check:** The highest-risk area is Diligent binding of NRD shader resources because NRD exposes register/resource descriptors while Diligent binding usually uses reflected names. Keep all binding fixes inside `RTXPTNrdIntegration.cpp` and do not change G7 shader resource names unless reflection proves it is necessary.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-06-04-rtxpt-realtime-g8-g9-nrd-integration-denoise-orchestration.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - Execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.
