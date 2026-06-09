# Shader Optimization Level — Design Spec

**Date:** 2026-06-09
**Status:** Approved (pending written-spec review)
**Branch:** RTXPT

## Motivation

Shader optimization level is currently hardcoded by build configuration (`DILIGENT_DEBUG`)
inside each compiler backend. In `DILIGENT_DEBUG` builds, DXC compiles with `-Od`
(`DXC_ARG_SKIP_OPTIMIZATIONS`). On the RTXPT branch this triggers a DXC `-Od` miscompilation
of the ray-tracing `HandleHit`/BxDF code (recorded in commit `c91a47a`), and there is no
per-shader way to force optimization on to work around it.

This change adds a per-shader `SHADER_OPTIMIZATION_LEVEL` control to `ShaderCreateInfo`, letting
an application override the optimization level for an individual shader while leaving the global
build-config-driven default untouched for every other shader.

## Goals

- Add a `SHADER_OPTIMIZATION_LEVEL` enum and a `ShaderOptimizationLevel` field to `ShaderCreateInfo`.
- Allow per-shader control across the **DXC**, **FXC**, and **glslang/SPIRV-Tools** compile paths.
- **Zero regression:** the default value reproduces today's behavior byte-for-byte on every backend.

## Non-Goals / Out of Scope

- **OpenGL-native GLSL** — the native driver compiles GLSL; there is no portable optimization-level
  knob. The field is a no-op here.
- **WebGPU / WGSL** and **Metal / MSL** — not requested; the field is ignored on these backends.
- No central policy abstraction — see "Architecture decision" below.

## Decisions (locked)

| Topic | Decision |
|---|---|
| Enum name | `SHADER_OPTIMIZATION_LEVEL` (full word, matching `SHADER_COMPILER`, `SHADER_SOURCE_LANGUAGE`, etc.) |
| Field name | `ShaderOptimizationLevel` (as requested) |
| Backends controlled | DXC, FXC, glslang/SPIRV-Tools |
| `DEFAULT` semantics | **Keep each backend's current behavior exactly** (zero regression) |
| glslang granularity | SPIRV-Tools has no `O0`–`O3`; levels collapse to on/off of `SPIRV_OPTIMIZATION_FLAG_PERFORMANCE`. Legalization on the HLSL→SPIRV path is **always** applied (correctness, not perf). |

## Architecture decision

`DEFAULT` is defined as "keep the current behavior," and the current behavior is already
**different per backend** (DXC: debug→`-Od`/release→`-O3`; FXC: no explicit opt flag; glslang:
always `PERFORMANCE`). Because `DEFAULT` has no single uniform meaning, there is nothing to
centralize. Each backend therefore:

1. On `DEFAULT` → executes its **existing code path unchanged**.
2. On `DISABLED` / `_0`..`_3` → takes a new explicit `else`/`switch` branch mapping the level to
   that compiler's native flags.

No shared `Resolve...()` helper is introduced (an earlier draft proposed one; it is unnecessary
under the zero-regression `DEFAULT` semantics).

## Public API (`DiligentCore/Graphics/GraphicsEngine/interface/Shader.h`)

New enum, placed after `SHADER_COMPILE_FLAGS` inside the `clang-format off` block:

```c
/// Describes the shader optimization level.
DILIGENT_TYPED_ENUM(SHADER_OPTIMIZATION_LEVEL, Uint8)
{
    /// Default optimization level.
    /// In DILIGENT_DEBUG builds, optimization is disabled; otherwise maximum optimization is used.
    /// This value reproduces the engine's historical per-backend behavior.
    SHADER_OPTIMIZATION_LEVEL_DEFAULT = 0,

    /// Optimization is explicitly disabled regardless of build configuration
    /// (DXC `-Od`, FXC `D3DCOMPILE_SKIP_OPTIMIZATION`, no SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_DISABLED,

    /// Optimization level 0 (DXC `-O0`, FXC `D3DCOMPILE_OPTIMIZATION_LEVEL0`, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_0,

    /// Optimization level 1 (DXC `-O1`, FXC `D3DCOMPILE_OPTIMIZATION_LEVEL1`, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_1,

    /// Optimization level 2 (DXC `-O2`, FXC `D3DCOMPILE_OPTIMIZATION_LEVEL2`, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_2,

    /// Optimization level 3 (DXC `-O3`, FXC `D3DCOMPILE_OPTIMIZATION_LEVEL3`, SPIR-V performance pass).
    SHADER_OPTIMIZATION_LEVEL_3,

    SHADER_OPTIMIZATION_LEVEL_COUNT
};
```

New field in `ShaderCreateInfo` (default `0` so every existing zero-initialized call site keeps
today's behavior):

```c
/// Shader optimization level. See Diligent::SHADER_OPTIMIZATION_LEVEL.
SHADER_OPTIMIZATION_LEVEL ShaderOptimizationLevel DEFAULT_INITIALIZER(SHADER_OPTIMIZATION_LEVEL_DEFAULT);
```

Also add `ShaderOptimizationLevel` to `ShaderCreateInfo::operator==` (in the
`#if DILIGENT_CPP_INTERFACE` block).

The existing constructors rely on in-class initializers for unlisted members, so they need no
changes — the new field defaults correctly.

## Per-backend behavior

### DXC — `DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp` (~L735–762)

Refactor so debug-*info* args stay gated on `DILIGENT_DEBUG`, while the optimization arg is
selected by `ShaderCI.ShaderOptimizationLevel`:

- Debug info (`DXC_ARG_DEBUG`, and `-Qembed_debug` for the D3D12 target when DXC ≥ 1.5) remains
  emitted under `DILIGENT_DEBUG`, independent of the optimization level.
- Optimization arg:
  - `DEFAULT` → existing logic verbatim:
    - D3D12 target: debug → `DXC_ARG_SKIP_OPTIMIZATIONS`; release → `DXC_ARG_OPTIMIZATION_LEVEL3`
      if DXC ≥ 1.5 else `DXC_ARG_SKIP_OPTIMIZATIONS` (legacy guard preserved).
    - Vulkan target: debug → `DXC_ARG_SKIP_OPTIMIZATIONS`; release → `DXC_ARG_OPTIMIZATION_LEVEL3`.
  - `DISABLED` → `DXC_ARG_SKIP_OPTIMIZATIONS` (`-Od`).
  - `_0`..`_3` → `DXC_ARG_OPTIMIZATION_LEVEL0`..`DXC_ARG_OPTIMIZATION_LEVEL3` (`-O0`..`-O3`).

Explicit levels are honored as written; the "DXC < 1.5 → force `-Od`" legacy guard applies **only**
to `DEFAULT` (an explicit level is a deliberate opt-in by the application).

If `DXC_ARG_OPTIMIZATION_LEVEL0/1/2` are not defined by the bundled `dxcapi.h`, fall back to the
literal argument strings `L"-O0"`/`L"-O1"`/`L"-O2"` (`DXC_ARG_OPTIMIZATION_LEVEL3` is already used).

### FXC — `DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp` (~L105–128)

`D3DCOMPILE_DEBUG` stays under `DILIGENT_DEBUG`. Add an optimization mapping that ORs into
`dwShaderFlags`:

- `DEFAULT` → no explicit optimization flag (unchanged — FXC uses its own default level).
- `DISABLED` → `D3DCOMPILE_SKIP_OPTIMIZATION`.
- `_0`..`_3` → `D3DCOMPILE_OPTIMIZATION_LEVEL0`..`D3DCOMPILE_OPTIMIZATION_LEVEL3`.

Note: an old comment warns that a *D3D10* optimization macro caused FXC failures. The modern
`D3DCOMPILE_OPTIMIZATION_LEVEL3` flag is the correct one and is only emitted when the application
explicitly requests it, so the historical warning does not block this mapping.

### glslang / SPIRV-Tools — `DiligentCore/Graphics/ShaderTools/src/GLSLangUtils.cpp` (~L491, L538)

**`HLSLtoSPIRV`** (has `ShaderCI` directly): always start from
`SPIRV_OPTIMIZATION_FLAG_LEGALIZATION` (required to turn HLSL-generated SPIR-V into valid Vulkan
SPIR-V — never dropped), then:

- `DEFAULT` → add `SPIRV_OPTIMIZATION_FLAG_PERFORMANCE` (unchanged behavior).
- `DISABLED` → legalization only (no performance pass).
- `_0`..`_3` → add `SPIRV_OPTIMIZATION_FLAG_PERFORMANCE`.

**`GLSLtoSPIRV`** (takes `GLSLtoSPIRVAttribs`, no `ShaderCI`): add an `OptimizationLevel` field to
`GLSLtoSPIRVAttribs` (`DiligentCore/Graphics/ShaderTools/include/GLSLangUtils.hpp`), populated from
`ShaderCI.ShaderOptimizationLevel` at the call site(s). Then:

- `DEFAULT` / `_0`..`_3` → `SPIRV_OPTIMIZATION_FLAG_PERFORMANCE` (unchanged behavior).
- `DISABLED` → skip the `OptimizeSPIRV(...)` call entirely and return the raw SPIR-V.

Call sites that construct `GLSLtoSPIRVAttribs` must be updated to set the new field from `ShaderCI`.

## Cache correctness

The optimization level changes the compiled bytecode, so it **must** participate in the
render-state-cache key and the serialized archive, otherwise a shader compiled at one level could
be served from cache when another was requested.

### Hash — `DiligentCore/Graphics/GraphicsTools/src/XXH128Hasher.cpp` (~L64–78)

- Add `ShaderCI.ShaderOptimizationLevel` to the `Update(...)` argument list.
- Bump `ASSERT_SIZEOF64(ShaderCI, 152, ...)` to the new `sizeof(ShaderCreateInfo)` (a `Uint8`
  plus alignment padding; the exact value is confirmed at build time — expected 160).

Adding the field to the hash invalidates existing render-state caches once on upgrade. This is
normal and expected when the struct layout changes.

### Serializer — `DiligentCore/Graphics/GraphicsEngine/src/PSOSerializer.cpp` (~L507–526)

- Add `CI.ShaderOptimizationLevel` to `SerializeShaderCreateInfo`.
- Verify whether the archive/serializer version constant needs a bump for the new field; bump it if
  the project policy requires it for format changes.

## Housekeeping

- Update the copyright year on every modified source/header file (per `CLAUDE.md`).
- Follow clang-format / existing alignment style in `Shader.h` (the file uses `clang-format off`
  around the enum/struct region — keep new members aligned with their neighbors).

## Touch-point summary

| File | Change |
|---|---|
| `Graphics/GraphicsEngine/interface/Shader.h` | New `SHADER_OPTIMIZATION_LEVEL` enum; new `ShaderOptimizationLevel` field; `operator==` update |
| `Graphics/ShaderTools/src/DXCompiler.cpp` | Separate debug-info args from opt arg; map level → DXC `-O*`/`-Od` |
| `Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp` | Map level → `D3DCOMPILE_*` optimization flags |
| `Graphics/ShaderTools/src/GLSLangUtils.cpp` | Map level → SPIR-V `PERFORMANCE` pass (HLSL & GLSL paths) |
| `Graphics/ShaderTools/include/GLSLangUtils.hpp` | Add `OptimizationLevel` to `GLSLtoSPIRVAttribs` |
| `Graphics/GraphicsTools/src/XXH128Hasher.cpp` | Hash the new field; bump `ASSERT_SIZEOF64` |
| `Graphics/GraphicsEngine/src/PSOSerializer.cpp` | Serialize the new field; verify version bump |

## Testing notes

- Existing shader/PSO tests must keep passing with the default value (proves zero regression).
- Spot-check: a shader created with `SHADER_OPTIMIZATION_LEVEL_3` in a `DILIGENT_DEBUG` build
  produces optimized DXC bytecode (the RTXPT `HandleHit` workaround), and one created with
  `SHADER_OPTIMIZATION_LEVEL_DISABLED` produces unoptimized bytecode in a release build.
- `XXH128Hasher` / `PSOSerializer` round-trip tests (`HashUtilsTest`, `PSOSerializerTest`) should be
  reviewed for the added field.
