# Shader Optimization Level Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-shader `SHADER_OPTIMIZATION_LEVEL` control to `ShaderCreateInfo` so an application can override the optimization level of an individual shader across the DXC, FXC, and glslang/SPIRV-Tools compile paths, while the default value reproduces today's behavior exactly.

**Architecture:** A new `SHADER_OPTIMIZATION_LEVEL` enum (`DEFAULT`, `DISABLED`, `_0`..`_3`) and a `ShaderOptimizationLevel` field are added to `ShaderCreateInfo`. `DEFAULT` has no uniform meaning (each backend's current behavior already differs), so each backend keeps its existing `DEFAULT` code path and adds an explicit branch for `DISABLED`/`_0`..`_3`. The field is threaded into the render-state-cache hash and the device-object archive serializer (with an archive-version bump) so optimization level participates in cache identity.

**Tech Stack:** C++ (Diligent Engine / DiligentCore), DXC (`dxcapi.h`), FXC (`D3DCompile`), glslang + SPIRV-Tools, GoogleTest, CMake + Visual Studio.

**Reference spec:** `docs/superpowers/specs/2026-06-09-shader-optimization-level-design.md`

---

## Build/Test Policy (read before executing)

Per the project's `CLAUDE.md` rule ("Unless explicitly requested by the USER, DO NOT run test or run build commands on your own"), **do not build or run tests after each edit.** Implementation Tasks 1–8 are edit-and-commit only. All verification (full build, targeted tests, and format validation) is consolidated into **Task 9**, which the USER explicitly requested be run when everything is done. This is a deliberate deviation from per-step TDD because (a) the user instruction overrides the skill's TDD default and (b) this codebase requires a heavy full CMake/Visual Studio build that is impractical per-step.

The three `ASSERT_SIZEOF64` static asserts and the compiler-checked `operator==` / switch statements act as the in-source guardrails that a field was handled everywhere; the Task 9 build surfaces any miss.

## File Structure / Touch-Point Map

| File | Responsibility / Change |
|---|---|
| `DiligentCore/Graphics/GraphicsEngine/interface/Shader.h` | Public API: new enum, new `ShaderCreateInfo` field, `operator==` |
| `DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp` | DXC: map level → `-Od`/`-O0`..`-O3`, separate from debug-info args |
| `DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp` | FXC: map level → `D3DCOMPILE_*` optimization flags |
| `DiligentCore/Graphics/ShaderTools/include/GLSLangUtils.hpp` | Add `OptimizationLevel` to `GLSLtoSPIRVAttribs` |
| `DiligentCore/Graphics/ShaderTools/src/GLSLangUtils.cpp` | glslang: helper + HLSL & GLSL SPIR-V paths |
| `DiligentCore/Graphics/GraphicsEngineVulkan/src/ShaderVkImpl.cpp` | Plumb `ShaderCI.ShaderOptimizationLevel` into the Vulkan GLSL path |
| `DiligentCore/Graphics/GraphicsTools/src/XXH128Hasher.cpp` | Hash the new field |
| `DiligentCore/Tests/DiligentCoreTest/src/Common/HashUtilsTest.cpp` | Cover the new field in the hash test |
| `DiligentCore/Graphics/GraphicsEngine/src/PSOSerializer.cpp` | Serialize the new field |
| `DiligentCore/Tests/DiligentCoreTest/src/GraphicsEngine/PSOSerializerTest.cpp` | Cover the new field in the serializer round-trip test |
| `DiligentCore/Graphics/GraphicsEngine/include/DeviceObjectArchive.hpp` | Bump `ArchiveVersion` 9 → 10 |

**There are three `ASSERT_SIZEOF64(ShaderCreateInfo, 152, …)` guards** — in `XXH128Hasher.cpp` (line 66), `PSOSerializer.cpp` (line 530), and `HashUtilsTest.cpp` (line 1145). The new `Uint8` is expected to land in existing tail padding, so `sizeof` should stay `152` and none should fire. If the Task 9 build reports any of them firing, update **all three** to the size in the error message.

**Copyright rule:** Every file modified below must have its top copyright line's end year changed to `2026` (e.g. `Copyright 2019-2025` → `Copyright 2019-2026`; keep the original start year). This is a required sub-step in each task and is not repeated as a separate code block.

---

## Task 1: Public API — enum, field, and equality (`Shader.h`)

**Files:**
- Modify: `DiligentCore/Graphics/GraphicsEngine/interface/Shader.h`

- [ ] **Step 1: Add the `SHADER_OPTIMIZATION_LEVEL` enum**

In `Shader.h`, the block ends like this (around lines 418–422):

```c
    SHADER_COMPILE_FLAG_LAST = SHADER_COMPILE_FLAG_HLSL_TO_SPIRV_VIA_GLSL
};
DEFINE_FLAG_ENUM_OPERATORS(SHADER_COMPILE_FLAGS);

// clang-format on
```

Insert the new enum between `DEFINE_FLAG_ENUM_OPERATORS(SHADER_COMPILE_FLAGS);` and `// clang-format on` so it reads:

```c
    SHADER_COMPILE_FLAG_LAST = SHADER_COMPILE_FLAG_HLSL_TO_SPIRV_VIA_GLSL
};
DEFINE_FLAG_ENUM_OPERATORS(SHADER_COMPILE_FLAGS);


/// Describes the shader optimization level.
DILIGENT_TYPED_ENUM(SHADER_OPTIMIZATION_LEVEL, Uint8)
{
    /// Default optimization level.

    /// In `DILIGENT_DEBUG` builds, optimization is disabled; otherwise maximum optimization is used.
    /// This value reproduces the engine's historical per-backend behavior.
    SHADER_OPTIMIZATION_LEVEL_DEFAULT = 0,

    /// Optimization is explicitly disabled regardless of build configuration
    /// (DXC `-Od`, FXC `D3DCOMPILE_SKIP_OPTIMIZATION`, no SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_DISABLED,

    /// Optimization level 0 (DXC `-O0`, FXC optimization level 0, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_0,

    /// Optimization level 1 (DXC `-O1`, FXC optimization level 1, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_1,

    /// Optimization level 2 (DXC `-O2`, FXC optimization level 2, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_2,

    /// Optimization level 3 (DXC `-O3`, FXC optimization level 3, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_3,

    SHADER_OPTIMIZATION_LEVEL_COUNT
};

// clang-format on
```

- [ ] **Step 2: Add the `ShaderOptimizationLevel` field to `ShaderCreateInfo`**

Find this line (around line 526):

```c
    /// Shader compile flags (see Diligent::SHADER_COMPILE_FLAGS).
    SHADER_COMPILE_FLAGS CompileFlags DEFAULT_INITIALIZER(SHADER_COMPILE_FLAG_NONE);
```

Insert the new field immediately after it:

```c
    /// Shader compile flags (see Diligent::SHADER_COMPILE_FLAGS).
    SHADER_COMPILE_FLAGS CompileFlags DEFAULT_INITIALIZER(SHADER_COMPILE_FLAG_NONE);

    /// Shader optimization level. See Diligent::SHADER_OPTIMIZATION_LEVEL.
    SHADER_OPTIMIZATION_LEVEL ShaderOptimizationLevel DEFAULT_INITIALIZER(SHADER_OPTIMIZATION_LEVEL_DEFAULT);
```

(Placing a `Uint8` here lands it in the existing tail padding before the two trailing `const char*` members, so `sizeof(ShaderCreateInfo)` is expected to stay `152`. If the three `ASSERT_SIZEOF64` static asserts say otherwise at build time, update them to the reported value — see Tasks 5–6 and the verification gate.)

- [ ] **Step 3: Add the field to `ShaderCreateInfo::operator==`**

Find this comparison (around lines 675–676):

```c
        if (CI1.CompileFlags != CI2.CompileFlags)
            return false;
```

Insert immediately after it:

```c
        if (CI1.CompileFlags != CI2.CompileFlags)
            return false;

        if (CI1.ShaderOptimizationLevel != CI2.ShaderOptimizationLevel)
            return false;
```

- [ ] **Step 4: Update the copyright end year to 2026** (top of file: `Copyright 2019-2025` → `Copyright 2019-2026`).

- [ ] **Step 5: Commit**

```bash
git add DiligentCore/Graphics/GraphicsEngine/interface/Shader.h
git commit -m "feat(shader): add SHADER_OPTIMIZATION_LEVEL to ShaderCreateInfo"
```

---

## Task 2: DXC backend mapping (`DXCompiler.cpp`)

**Files:**
- Modify: `DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp`

- [ ] **Step 1: Add a file-local helper that maps the level to a DXC optimization argument**

Locate `void DXCompilerImpl::Compile(` (around line 694). Immediately **above** it (after the preceding function's closing `}` near line 691), insert this static helper:

```cpp
static const wchar_t* GetDxcOptimizationArg(SHADER_OPTIMIZATION_LEVEL OptimizationLevel,
                                            DXCompilerTarget          Target,
                                            const Version&            DxcVersion)
{
    switch (OptimizationLevel)
    {
        case SHADER_OPTIMIZATION_LEVEL_DISABLED:
            return DXC_ARG_SKIP_OPTIMIZATIONS;
        case SHADER_OPTIMIZATION_LEVEL_0:
            return DXC_ARG_OPTIMIZATION_LEVEL0;
        case SHADER_OPTIMIZATION_LEVEL_1:
            return DXC_ARG_OPTIMIZATION_LEVEL1;
        case SHADER_OPTIMIZATION_LEVEL_2:
            return DXC_ARG_OPTIMIZATION_LEVEL2;
        case SHADER_OPTIMIZATION_LEVEL_3:
            return DXC_ARG_OPTIMIZATION_LEVEL3;
        case SHADER_OPTIMIZATION_LEVEL_DEFAULT:
        default:
            // DEFAULT reproduces the historical behavior.
#ifdef DILIGENT_DEBUG
            return DXC_ARG_SKIP_OPTIMIZATIONS;
#else
            // For the Direct3D12 target, optimization was historically only enabled for DXC 1.5+.
            if (Target == DXCompilerTarget::Direct3D12 && DxcVersion < Version{1, 5})
                return DXC_ARG_SKIP_OPTIMIZATIONS;
            return DXC_ARG_OPTIMIZATION_LEVEL3;
#endif
    }
}
```

- [ ] **Step 2: Replace the Direct3D12 argument block to separate debug-info from the optimization arg**

Find this block (around lines 732–750):

```cpp
    std::vector<const wchar_t*> DxilArgs;
    if (m_Library.GetTarget() == DXCompilerTarget::Direct3D12)
    {
        //DxilArgs.push_back(L"-WX");  // Warnings as errors
#ifdef DILIGENT_DEBUG
        DxilArgs.push_back(DXC_ARG_DEBUG);              // Debug info
        DxilArgs.push_back(DXC_ARG_SKIP_OPTIMIZATIONS); // Disable optimization
        if (m_Library.GetVersion() >= Version{1, 5})
        {
            // Silence the following warning:
            // no output provided for debug - embedding PDB in shader container.  Use -Qembed_debug to silence this warning.
            DxilArgs.push_back(L"-Qembed_debug");
        }
#else
        if (m_Library.GetVersion() >= Version{1, 5})
            DxilArgs.push_back(DXC_ARG_OPTIMIZATION_LEVEL3); // Optimization level 3
        else
            DxilArgs.push_back(DXC_ARG_SKIP_OPTIMIZATIONS); // TODO: something goes wrong if optimization is enabled
#endif
    }
```

Replace it with:

```cpp
    std::vector<const wchar_t*> DxilArgs;
    if (m_Library.GetTarget() == DXCompilerTarget::Direct3D12)
    {
        //DxilArgs.push_back(L"-WX");  // Warnings as errors
#ifdef DILIGENT_DEBUG
        DxilArgs.push_back(DXC_ARG_DEBUG); // Debug info
        if (m_Library.GetVersion() >= Version{1, 5})
        {
            // Silence the following warning:
            // no output provided for debug - embedding PDB in shader container.  Use -Qembed_debug to silence this warning.
            DxilArgs.push_back(L"-Qembed_debug");
        }
#endif
        DxilArgs.push_back(GetDxcOptimizationArg(ShaderCI.ShaderOptimizationLevel, m_Library.GetTarget(), m_Library.GetVersion()));
    }
```

- [ ] **Step 3: Replace the Vulkan argument block**

Find this block (around lines 751–762):

```cpp
    else if (m_Library.GetTarget() == DXCompilerTarget::Vulkan)
    {
        DxilArgs.assign(
            {
                L"-spirv",
                L"-fspv-reflect",
#ifdef DILIGENT_DEBUG
                DXC_ARG_SKIP_OPTIMIZATIONS,
#else
                DXC_ARG_OPTIMIZATION_LEVEL3
#endif
            });
```

Replace it with:

```cpp
    else if (m_Library.GetTarget() == DXCompilerTarget::Vulkan)
    {
        DxilArgs.assign(
            {
                L"-spirv",
                L"-fspv-reflect",
            });
        DxilArgs.push_back(GetDxcOptimizationArg(ShaderCI.ShaderOptimizationLevel, m_Library.GetTarget(), m_Library.GetVersion()));
```

(Leave the rest of the Vulkan block — the `-fspv-target-env` logic that follows — unchanged.)

- [ ] **Step 4: Update the copyright end year to 2026.**

- [ ] **Step 5: Commit**

```bash
git add DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp
git commit -m "feat(shader): honor ShaderOptimizationLevel in DXC compiler"
```

---

## Task 3: FXC backend mapping (`ShaderD3DBase.cpp`)

**Files:**
- Modify: `DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp`

- [ ] **Step 1: Add the optimization-level mapping to `CompileShader`**

Find this block (around lines 122–125):

```cpp
    if (ShaderCI.CompileFlags & SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR)
        dwShaderFlags |= D3DCOMPILE_PACK_MATRIX_ROW_MAJOR;

    D3D_SHADER_MACRO Macros[] = {{"D3DCOMPILER", ""}, {}};
```

Insert the optimization mapping between them so it reads:

```cpp
    if (ShaderCI.CompileFlags & SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR)
        dwShaderFlags |= D3DCOMPILE_PACK_MATRIX_ROW_MAJOR;

    switch (ShaderCI.ShaderOptimizationLevel)
    {
        case SHADER_OPTIMIZATION_LEVEL_DISABLED: dwShaderFlags |= D3DCOMPILE_SKIP_OPTIMIZATION;   break;
        case SHADER_OPTIMIZATION_LEVEL_0:        dwShaderFlags |= D3DCOMPILE_OPTIMIZATION_LEVEL0; break;
        case SHADER_OPTIMIZATION_LEVEL_1:        dwShaderFlags |= D3DCOMPILE_OPTIMIZATION_LEVEL1; break;
        case SHADER_OPTIMIZATION_LEVEL_2:        dwShaderFlags |= D3DCOMPILE_OPTIMIZATION_LEVEL2; break;
        case SHADER_OPTIMIZATION_LEVEL_3:        dwShaderFlags |= D3DCOMPILE_OPTIMIZATION_LEVEL3; break;
        case SHADER_OPTIMIZATION_LEVEL_DEFAULT:
        default:
            // DEFAULT: no explicit optimization flag (FXC uses its own default level), preserving historical behavior.
            break;
    }

    D3D_SHADER_MACRO Macros[] = {{"D3DCOMPILER", ""}, {}};
```

- [ ] **Step 2: Update the copyright end year to 2026.**

- [ ] **Step 3: Commit**

```bash
git add DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp
git commit -m "feat(shader): honor ShaderOptimizationLevel in FXC compiler"
```

---

## Task 4: glslang / SPIRV-Tools backend mapping

**Files:**
- Modify: `DiligentCore/Graphics/ShaderTools/include/GLSLangUtils.hpp`
- Modify: `DiligentCore/Graphics/ShaderTools/src/GLSLangUtils.cpp`
- Modify: `DiligentCore/Graphics/GraphicsEngineVulkan/src/ShaderVkImpl.cpp`

- [ ] **Step 1: Add the `OptimizationLevel` field to `GLSLtoSPIRVAttribs`**

In `GLSLangUtils.hpp`, find the struct (around lines 56–67):

```cpp
struct GLSLtoSPIRVAttribs
{
    SHADER_TYPE                      ShaderType    = SHADER_TYPE_UNKNOWN;
    const char*                      ShaderSource  = nullptr;
    int                              SourceCodeLen = 0;
    ShaderMacroArray                 Macros;
    IShaderSourceInputStreamFactory* pShaderSourceStreamFactory = nullptr;
    SpirvVersion                     Version                    = SpirvVersion::Vk100;
    IDataBlob**                      ppCompilerOutput           = nullptr;
    bool                             AssignBindings             = true;
    bool                             UseRowMajorMatrices        = false;
};
```

Add the new field before the closing brace:

```cpp
struct GLSLtoSPIRVAttribs
{
    SHADER_TYPE                      ShaderType    = SHADER_TYPE_UNKNOWN;
    const char*                      ShaderSource  = nullptr;
    int                              SourceCodeLen = 0;
    ShaderMacroArray                 Macros;
    IShaderSourceInputStreamFactory* pShaderSourceStreamFactory = nullptr;
    SpirvVersion                     Version                    = SpirvVersion::Vk100;
    IDataBlob**                      ppCompilerOutput           = nullptr;
    bool                             AssignBindings             = true;
    bool                             UseRowMajorMatrices        = false;
    SHADER_OPTIMIZATION_LEVEL        OptimizationLevel          = SHADER_OPTIMIZATION_LEVEL_DEFAULT;
};
```

- [ ] **Step 2: Add a file-local helper in `GLSLangUtils.cpp`**

Find the `HLSLtoSPIRV` definition (around line 430):

```cpp
std::vector<unsigned int> HLSLtoSPIRV(const ShaderCreateInfo& ShaderCI,
```

Insert this static helper immediately **above** it:

```cpp
static SPIRV_OPTIMIZATION_FLAGS GetSpirvPerformanceFlag(SHADER_OPTIMIZATION_LEVEL OptimizationLevel)
{
    // SPIRV-Tools has no O0-O3 granularity. DEFAULT and explicit levels 0..3 enable the
    // performance pass; only DISABLED skips it. (Legalization, when required, is applied separately.)
    return OptimizationLevel == SHADER_OPTIMIZATION_LEVEL_DISABLED ?
        SPIRV_OPTIMIZATION_FLAG_NONE :
        SPIRV_OPTIMIZATION_FLAG_PERFORMANCE;
}

std::vector<unsigned int> HLSLtoSPIRV(const ShaderCreateInfo& ShaderCI,
```

- [ ] **Step 3: Apply the level on the HLSL→SPIRV path (legalization always on)**

In `HLSLtoSPIRV`, find this block (around lines 488–491):

```cpp
#ifdef USE_SPIRV_TOOLS
    // SPIR-V bytecode generated from HLSL must be legalized to
    // turn it into a valid vulkan SPIR-V shader.
    std::vector<uint32_t> LegalizedSPIRV = OptimizeSPIRV(SPIRV, SpirvVersionToSpvTargetEnv(Version), SPIRV_OPTIMIZATION_FLAG_LEGALIZATION | SPIRV_OPTIMIZATION_FLAG_PERFORMANCE);
```

Replace the two lines with:

```cpp
#ifdef USE_SPIRV_TOOLS
    // SPIR-V bytecode generated from HLSL must be legalized to
    // turn it into a valid vulkan SPIR-V shader. Legalization is always applied for correctness;
    // the performance pass is gated by the shader optimization level.
    const SPIRV_OPTIMIZATION_FLAGS OptimizationFlags = SPIRV_OPTIMIZATION_FLAG_LEGALIZATION | GetSpirvPerformanceFlag(ShaderCI.ShaderOptimizationLevel);
    std::vector<uint32_t> LegalizedSPIRV = OptimizeSPIRV(SPIRV, SpirvVersionToSpvTargetEnv(Version), OptimizationFlags);
```

- [ ] **Step 4: Apply the level on the GLSL→SPIRV path (skip optimization when disabled)**

In `GLSLtoSPIRV`, find this block (around lines 537–548):

```cpp
#ifdef USE_SPIRV_TOOLS
    std::vector<uint32_t> OptimizedSPIRV = OptimizeSPIRV(SPIRV, SpirvVersionToSpvTargetEnv(Attribs.Version), SPIRV_OPTIMIZATION_FLAG_PERFORMANCE);
    if (!OptimizedSPIRV.empty())
    {
        return OptimizedSPIRV;
    }
    else
    {
        LOG_ERROR("Failed to optimize SPIR-V.");
    }
#endif
    return SPIRV;
```

Replace it with:

```cpp
#ifdef USE_SPIRV_TOOLS
    const SPIRV_OPTIMIZATION_FLAGS OptimizationFlags = GetSpirvPerformanceFlag(Attribs.OptimizationLevel);
    if (OptimizationFlags != SPIRV_OPTIMIZATION_FLAG_NONE)
    {
        std::vector<uint32_t> OptimizedSPIRV = OptimizeSPIRV(SPIRV, SpirvVersionToSpvTargetEnv(Attribs.Version), OptimizationFlags);
        if (!OptimizedSPIRV.empty())
        {
            return OptimizedSPIRV;
        }
        else
        {
            LOG_ERROR("Failed to optimize SPIR-V.");
        }
    }
#endif
    return SPIRV;
```

- [ ] **Step 5: Plumb the level into the Vulkan GLSL caller**

In `ShaderVkImpl.cpp`, find this block (around lines 146–148):

```cpp
        Attribs.UseRowMajorMatrices        = (ShaderCI.CompileFlags & SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR) != 0;
        Attribs.pShaderSourceStreamFactory = ShaderCI.pShaderSourceStreamFactory;
        Attribs.ppCompilerOutput           = VkShaderCI.ppCompilerOutput;
```

Replace it with:

```cpp
        Attribs.UseRowMajorMatrices        = (ShaderCI.CompileFlags & SHADER_COMPILE_FLAG_PACK_MATRIX_ROW_MAJOR) != 0;
        Attribs.pShaderSourceStreamFactory = ShaderCI.pShaderSourceStreamFactory;
        Attribs.ppCompilerOutput           = VkShaderCI.ppCompilerOutput;
        Attribs.OptimizationLevel          = ShaderCI.ShaderOptimizationLevel;
```

(The WebGPU and GL-archiver callers of `GLSLtoSPIRV` are intentionally left unmodified — they keep the `SHADER_OPTIMIZATION_LEVEL_DEFAULT` default, which preserves their current behavior, since those backends are out of scope.)

- [ ] **Step 6: Update the copyright end year to 2026 on all three files.**

- [ ] **Step 7: Commit**

```bash
git add DiligentCore/Graphics/ShaderTools/include/GLSLangUtils.hpp \
        DiligentCore/Graphics/ShaderTools/src/GLSLangUtils.cpp \
        DiligentCore/Graphics/GraphicsEngineVulkan/src/ShaderVkImpl.cpp
git commit -m "feat(shader): honor ShaderOptimizationLevel in glslang/SPIRV-Tools path"
```

---

## Task 5: Hash the new field + cover it in the hash test

**Files:**
- Modify: `DiligentCore/Graphics/GraphicsTools/src/XXH128Hasher.cpp`
- Test: `DiligentCore/Tests/DiligentCoreTest/src/Common/HashUtilsTest.cpp`

- [ ] **Step 1: Add the field to the `ShaderCreateInfo` hash update**

Find this block (around lines 66–78):

```cpp
    ASSERT_SIZEOF64(ShaderCI, 152, "Did you add new members to ShaderCreateInfo? Please handle them here.");

    Update(static_cast<Uint32>(ShaderCI.SourceLength), // Aka ByteCodeSize
           ShaderCI.EntryPoint,
           ShaderCI.Desc,
           ShaderCI.SourceLanguage,
           ShaderCI.ShaderCompiler,
           ShaderCI.HLSLVersion,
           ShaderCI.GLSLVersion,
           ShaderCI.GLESSLVersion,
           ShaderCI.MSLVersion,
           ShaderCI.CompileFlags,
           ShaderCI.LoadConstantBufferReflection);
```

Replace it with (adds `ShaderCI.ShaderOptimizationLevel` to the hashed fields):

```cpp
    ASSERT_SIZEOF64(ShaderCI, 152, "Did you add new members to ShaderCreateInfo? Please handle them here.");

    Update(static_cast<Uint32>(ShaderCI.SourceLength), // Aka ByteCodeSize
           ShaderCI.EntryPoint,
           ShaderCI.Desc,
           ShaderCI.SourceLanguage,
           ShaderCI.ShaderCompiler,
           ShaderCI.HLSLVersion,
           ShaderCI.GLSLVersion,
           ShaderCI.GLESSLVersion,
           ShaderCI.MSLVersion,
           ShaderCI.CompileFlags,
           ShaderCI.ShaderOptimizationLevel,
           ShaderCI.LoadConstantBufferReflection);
```

> **Note on `ASSERT_SIZEOF64`:** the new `Uint8` is expected to occupy existing padding, leaving `sizeof(ShaderCreateInfo) == 152`, so the `152` literal should remain correct and the assert should not fire. **If the Task 9 build reports this static assert firing**, change `152` here to the exact size shown in the error message.

- [ ] **Step 2: Cover the new field in `TEST(XXH128HasherTest, ShaderCreateInfo)`**

In `HashUtilsTest.cpp`, find this pair of lines (around lines 1187–1188):

```cpp
    TEST_FLAGS(CompileFlags, static_cast<SHADER_COMPILE_FLAGS>(1), SHADER_COMPILE_FLAG_LAST);
    TEST_BOOL(LoadConstantBufferReflection);
```

Insert the optimization-level range between them (mirroring the struct field order):

```cpp
    TEST_FLAGS(CompileFlags, static_cast<SHADER_COMPILE_FLAGS>(1), SHADER_COMPILE_FLAG_LAST);
    TEST_RANGE(ShaderOptimizationLevel, static_cast<SHADER_OPTIMIZATION_LEVEL>(1), SHADER_OPTIMIZATION_LEVEL_COUNT);
    TEST_BOOL(LoadConstantBufferReflection);
```

This iterates `DISABLED`, `_0`, `_1`, `_2`, `_3` (the enum `AddRange` overload uses a half-open `[1, COUNT)` range), asserting each produces a new unique hash — which only holds if Step 1 added the field to `XXH128State::Update`. Leave the `ASSERT_SIZEOF64(ShaderCreateInfo, 152, ...)` at line 1145 unchanged (same caveat as Step 1).

- [ ] **Step 3: Update the copyright end year to 2026 on both files** (`HashUtilsTest.cpp` currently reads `Copyright 2019-2024` → `2019-2026`; keep its second `Copyright 2015-2019 Egor Yusov` line as-is).

- [ ] **Step 4: Commit**

```bash
git add DiligentCore/Graphics/GraphicsTools/src/XXH128Hasher.cpp \
        DiligentCore/Tests/DiligentCoreTest/src/Common/HashUtilsTest.cpp
git commit -m "feat(shader): include ShaderOptimizationLevel in ShaderCreateInfo hash"
```

---

## Task 6: Serialize the new field + cover it in the serializer test

**Files:**
- Modify: `DiligentCore/Graphics/GraphicsEngine/src/PSOSerializer.cpp`
- Test: `DiligentCore/Tests/DiligentCoreTest/src/GraphicsEngine/PSOSerializerTest.cpp`

- [ ] **Step 1: Add the field to `ShaderSerializer<Mode>::SerializeCI`**

Find this block (around lines 511–526):

```cpp
    if (!Ser(CI.Desc.Name,
             CI.Desc.ShaderType,
             CI.Desc.UseCombinedTextureSamplers,
             CI.Desc.CombinedSamplerSuffix,
             CI.EntryPoint,
             CI.SourceLanguage,
             CI.ShaderCompiler,
             CI.HLSLVersion,
             CI.GLSLVersion,
             CI.GLESSLVersion,
             CI.MSLVersion,
             CI.CompileFlags,
             CI.LoadConstantBufferReflection,
             CI.GLSLExtensions,
             CI.WebGPUEmulatedArrayIndexSuffix))
        return false;
```

Replace it with (adds `CI.ShaderOptimizationLevel`):

```cpp
    if (!Ser(CI.Desc.Name,
             CI.Desc.ShaderType,
             CI.Desc.UseCombinedTextureSamplers,
             CI.Desc.CombinedSamplerSuffix,
             CI.EntryPoint,
             CI.SourceLanguage,
             CI.ShaderCompiler,
             CI.HLSLVersion,
             CI.GLSLVersion,
             CI.GLESSLVersion,
             CI.MSLVersion,
             CI.CompileFlags,
             CI.ShaderOptimizationLevel,
             CI.LoadConstantBufferReflection,
             CI.GLSLExtensions,
             CI.WebGPUEmulatedArrayIndexSuffix))
        return false;
```

> **Note on `ASSERT_SIZEOF64`:** the same caveat as Task 5 applies to the `ASSERT_SIZEOF64(ShaderCreateInfo, 152, ...)` at the end of this function (around line 530). Leave it at `152`; if the Task 9 build reports it firing, update it to the size in the error message.

- [ ] **Step 2: Set a non-default value in the serializer round-trip test**

In `PSOSerializerTest.cpp`, find this pair of lines in `SerializeShaderCreateInfo` (around lines 791–792):

```cpp
    RefCI.CompileFlags                   = SHADER_COMPILE_FLAG_SKIP_REFLECTION;
    RefCI.LoadConstantBufferReflection   = true;
```

Insert the new field assignment between them:

```cpp
    RefCI.CompileFlags                   = SHADER_COMPILE_FLAG_SKIP_REFLECTION;
    RefCI.ShaderOptimizationLevel        = SHADER_OPTIMIZATION_LEVEL_3;
    RefCI.LoadConstantBufferReflection   = true;
```

- [ ] **Step 3: Assert the field round-trips**

In the same function, find this comparison block (around lines 841–842):

```cpp
    EXPECT_EQ   (CI.CompileFlags,   RefCI.CompileFlags);
    EXPECT_EQ   (CI.LoadConstantBufferReflection, RefCI.LoadConstantBufferReflection);
```

Insert the new comparison between them:

```cpp
    EXPECT_EQ   (CI.CompileFlags,   RefCI.CompileFlags);
    EXPECT_EQ   (CI.ShaderOptimizationLevel, RefCI.ShaderOptimizationLevel);
    EXPECT_EQ   (CI.LoadConstantBufferReflection, RefCI.LoadConstantBufferReflection);
```

(`PSOSerializerTest.cpp` already carries the `2019-2026` copyright, so no copyright edit is needed there.)

- [ ] **Step 4: Update the copyright end year to 2026 on `PSOSerializer.cpp`.**

- [ ] **Step 5: Commit**

```bash
git add DiligentCore/Graphics/GraphicsEngine/src/PSOSerializer.cpp \
        DiligentCore/Tests/DiligentCoreTest/src/GraphicsEngine/PSOSerializerTest.cpp
git commit -m "feat(shader): serialize ShaderOptimizationLevel in shader CI"
```

---

## Task 7: Bump the archive format version (`DeviceObjectArchive.hpp`)

**Files:**
- Modify: `DiligentCore/Graphics/GraphicsEngine/include/DeviceObjectArchive.hpp`

Rationale: Task 6 changes the serialized byte layout of shader records. Bumping the archive version makes the loader cleanly reject older archives (with an "unsupported version" message) instead of silently misreading them. Existing archives must be regenerated — consistent with the render-state-cache invalidation introduced by Task 5.

- [ ] **Step 1: Increment `ArchiveVersion`**

Find this line (around line 131):

```cpp
    static constexpr Uint32 ArchiveVersion    = 9;
```

Change it to:

```cpp
    static constexpr Uint32 ArchiveVersion    = 10;
```

- [ ] **Step 2: Update the copyright end year to 2026.**

- [ ] **Step 3: Commit**

```bash
git add DiligentCore/Graphics/GraphicsEngine/include/DeviceObjectArchive.hpp
git commit -m "chore(archive): bump ArchiveVersion for ShaderOptimizationLevel serialization"
```

---

## Task 8: Spot-check the public C-interface / interop surface

This is a verification-only task (no expected code change). `DILIGENT_TYPED_ENUM` and `DEFAULT_INITIALIZER` already emit both the C and C++ forms, and the `.NET`/SharpGen generator discovers struct fields automatically, so no manual interop edits are typically required.

- [ ] **Step 1:** Confirm no generated/checked-in interop file hardcodes the `ShaderCreateInfo` field list or `sizeof`. Search:

```bash
grep -rn "WebGPUEmulatedArrayIndexSuffix" DiligentCore --include=*.cs --include=*.h --include=*.hpp
```

Expected: matches only in the interface header and any auto-generated binding that already lists every field. If a hand-maintained binding lists fields explicitly, add `ShaderOptimizationLevel` there following the neighboring field's pattern and commit it; otherwise no change.

- [ ] **Step 2 (only if a file was changed): Commit**

```bash
git add <changed interop file>
git commit -m "feat(shader): expose ShaderOptimizationLevel in language bindings"
```

---

## Task 9: Verification gate (USER-run — build, tests, format validation)

This is the consolidated verification step the user asked to run "when everything is done." Run these from the repository root (`d:/DiligentEngine-hzqst`).

- [ ] **Step 1: Configure & build (Debug)**

```bash
cmake --build build/x64/Debug --config Debug
```

(If the build tree does not yet exist, configure first per `CLAUDE.md`:
`cmake -S . -B build/x64/Debug -G "Visual Studio 17 2022" -A x64 -DDILIGENT_BUILD_TESTS=ON -DDILIGENT_DEVELOPMENT=ON`.)

Expected: clean build. If any of the three `ASSERT_SIZEOF64(ShaderCreateInfo, 152, ...)` static asserts fires (in `XXH128Hasher.cpp:66`, `PSOSerializer.cpp:530`, or `HashUtilsTest.cpp:1145`), update **all three** `152` literals to the size reported in the error and rebuild, then amend/commit:
`git commit -am "fix(shader): update ShaderCreateInfo size asserts"`.

- [ ] **Step 2: Run the serializer & hash unit tests**

```bash
cd build/x64/Debug/Tests/DiligentCoreTest/Debug
DiligentCoreTest.exe --gtest_filter="*PSOSerializer*:*HashUtils*:*XXH128Hasher*"
```

Expected: PASS. These now set/vary `ShaderOptimizationLevel` (Tasks 5–6), so they actively verify the field is hashed and serialized — `XXH128HasherTest.ShaderCreateInfo` fails if the field was omitted from `XXH128State::Update`, and `PSOSerializerTest.SerializeShaderCI_*` fails if it was omitted from `SerializeCI`.

- [ ] **Step 3: Format validation** (the explicitly requested check)

```bash
cd BuildTools/FormatValidation
./validate_format_win.bat
```

Expected: no formatting violations reported. If any of the modified files fail, fix per the tool's output (or run the `DiligentCore-ValidateFormatting` CMake target) and commit:
`git commit -am "style: clang-format shader optimization level changes"`.

- [ ] **Step 4 (optional, GPU required): Smoke-test an explicit level**

In a `DILIGENT_DEBUG` build, create a DXC shader with `ShaderCI.ShaderOptimizationLevel = SHADER_OPTIMIZATION_LEVEL_3` and confirm it compiles and runs (this is the RTXPT `HandleHit` `-Od` miscompile workaround). With `SHADER_OPTIMIZATION_LEVEL_DEFAULT`, behavior must match the pre-change build.

---

## Self-Review

**Spec coverage** (against `2026-06-09-shader-optimization-level-design.md`):
- Enum `SHADER_OPTIMIZATION_LEVEL` + `ShaderOptimizationLevel` field + `operator==` → Task 1. ✓
- DXC mapping (DEFAULT keeps current incl. <1.5 guard; DISABLED/`_0`..`_3` explicit) → Task 2. ✓
- FXC mapping (DEFAULT no flag; DISABLED/`_0`..`_3`) → Task 3. ✓
- glslang: legalization always on HLSL path; PERFORMANCE on DEFAULT/`_0`..`_3`; DISABLED skips perf; `GLSLtoSPIRVAttribs` field + Vulkan caller → Task 4. ✓
- Hash + hash test → Task 5. ✓ Serializer + serializer test → Task 6. ✓ Archive version bump → Task 7. ✓
- Test coverage: `XXH128HasherTest.ShaderCreateInfo` and `PSOSerializerTest.SerializeShaderCI_*` updated to exercise the new field (Tasks 5–6); all three `ASSERT_SIZEOF64` guards accounted for. ✓
- Scope: WebGPU/GL/Metal left at default (no-op) → noted in Task 4 Step 5. ✓
- Copyright-year update → sub-step in every task. ✓
- Format validation at the end → Task 9 Step 3. ✓

**Placeholder scan:** No TBD/TODO/"handle edge cases"; every code step shows complete code. The only conditional ("if the static assert fires…") includes the exact action to take. ✓

**Type/name consistency:** `ShaderOptimizationLevel` (field), `SHADER_OPTIMIZATION_LEVEL` + `_DEFAULT/_DISABLED/_0../_3/_COUNT` (enum), `GetDxcOptimizationArg`, `GetSpirvPerformanceFlag`, `GLSLtoSPIRVAttribs::OptimizationLevel` are used identically across Tasks 1–6. ✓
