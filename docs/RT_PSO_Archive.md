# Ray-Tracing PSO archiving assertion in `RenderStateCache` (D3D12 + Vulkan multi-backend build)

## TL;DR

When RTXPT routes its ray-tracing PSO through an `IRenderStateCache` and runs on the
**D3D12** backend, the cache's PSO-archiving step trips a **Vulkan** debug assertion:

```
Debug assertion failed: pShader != nullptr
  PipelineStateVkImpl::ShaderStageInfo::Append(const ShaderVkImpl*)   PipelineStateVkImpl.cpp:715
  ShaderStageInfoVk::Append(const SerializedShaderImpl*)              Archiver_Vk.cpp:98
  PipelineStateUtils::ExtractShaders<SerializedShaderImpl>(...)       PipelineStateBase.hpp:275
  SerializedPipelineStateImpl::ExtractShadersVk(...)                 Archiver_Vk.cpp:268
  SerializePSOCreateInfo<Measure/Write>(... RayTracingPSOCreateInfo) SerializedPipelineStateImpl.cpp:117
  SerializedPipelineStateImpl::Initialize<RayTracingPSOCreateInfo>   SerializedPipelineStateImpl.cpp:286
  RenderStateCacheImpl::CreatePipelineStateInternal<RayTracingPSO>   RenderStateCacheImpl.cpp:918
  RTXPTRayTracingPass::Initialize (CreateVariant lambda)             RTXPTRayTracingPass.cpp:381
```

This is a **DiligentCore bug**, not an RTXPT bug. The ray-tracing PSO serializer extracts a
shader→index map for **every backend compiled into DiligentCore** (gated on the *compile-time*
macros `VULKAN_SUPPORTED` / `D3D12_SUPPORTED`), instead of only for the backend the PSO is
actually being archived for (the *runtime* `ArchiveInfo.DeviceFlags`). On a D3D12 run in a
build that also has the Vulkan backend enabled, the Vulkan extractor runs over shaders that were
only ever compiled/archived for D3D12, dereferences a non-existent Vulkan shader, and asserts.

It affects **only ray-tracing PSOs** (compute/graphics PSOs are unaffected) and only when the
DiligentCore build contains a backend other than the one being run.

---

## Symptom

- Backend at runtime: **D3D12**.
- Yet the failing frames are all in the **Vulkan** archiver (`PipelineStateVkImpl`,
  `ShaderStageInfoVk`, `ExtractShadersVk`).
- The assertion is `pShader != nullptr` — a *Vulkan* shader pointer is null.
- It fires while the `RenderStateCache` is **archiving the ray-tracing PSO** (the path-tracer
  PSO created in `RTXPTRayTracingPass::Initialize`), not while creating it on the device.

Earlier compute/graphics pipelines in the same run archived fine (`Added pipeline ...`); only the
ray-tracing PSO trips the assert.

---

## How the `RenderStateCache` archives a PSO

`RenderStateCache::CreateXxxPipelineState` does two things
([RenderStateCacheImpl.cpp:888-930](../../../../DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp)):

1. Creates the **real** PSO on the running device (line 888) — this always succeeds; the app
   gets a valid PSO regardless of what happens next.
2. **Re-serializes** the PSO through the archiver so it can be written to disk
   (lines 909-930), wrapped in `try { ... } catch (...) {}`.

Step 2 builds a `SerializedPipelineStateImpl`, scoping it to the running backend only:

```cpp
// RenderStateCacheImpl.cpp:915-918
PipelineStateArchiveInfo ArchiveInfo;
ArchiveInfo.DeviceFlags = RenderDeviceTypeToArchiveDataFlag(m_DeviceType);   // D3D12 only
m_pSerializationDevice->CreatePipelineState(SerializedPsoCI, ArchiveInfo, &pSerializedPSO);
```

Shaders are likewise archived for the running backend only
([RenderStateCacheImpl.cpp:454-456](../../../../DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp)):

```cpp
ShaderArchiveInfo ArchiveInfo;
ArchiveInfo.DeviceFlags = RenderDeviceTypeToArchiveDataFlag(m_DeviceType);   // D3D12 only
m_pSerializationDevice->CreateShader(ArchiveShaderCI, ArchiveInfo, &pArchivedShader);
```

So every serialized shader carries **D3D12 compiled data only**. Inside the serialized shader the
per-backend compiled blobs live in a fixed-size array indexed by device type
([SerializedShaderImpl.hpp:121](../../../../DiligentCore/Graphics/Archiver/include/SerializedShaderImpl.hpp)):

```cpp
std::array<std::unique_ptr<CompiledShader>, DeviceType::Count> m_Shaders;   // m_Shaders[Vulkan] == nullptr
```

`m_Shaders[D3D12]` is populated; `m_Shaders[Vulkan]` is **null**.

---

## The bug: compile-time backend gating in the RT serializer

`SerializedPipelineStateImpl::Initialize` patches shaders strictly according to the **runtime**
`ArchiveInfo.DeviceFlags` (a `while (DeviceBits) { switch (ExtractLSB(DeviceBits)) ... }` loop,
[SerializedPipelineStateImpl.cpp:194-242](../../../../DiligentCore/Graphics/Archiver/src/SerializedPipelineStateImpl.cpp)).
For a D3D12 archive it calls only `PatchShadersD3D12`. Correct.

But the **ray-tracing-specific** `SerializePSOCreateInfo` overload ignores the device flags and
probes every *compiled-in* backend
([SerializedPipelineStateImpl.cpp:108-145](../../../../DiligentCore/Graphics/Archiver/src/SerializedPipelineStateImpl.cpp)):

```cpp
template <SerializerMode Mode>
void SerializePSOCreateInfo(Serializer<Mode>& Ser,
                            const RayTracingPipelineStateCreateInfo& PSOCreateInfo,
                            std::array<const char*, MAX_RESOURCE_SIGNATURES>& PRSNames)
{
    RayTracingShaderMapType ShaderMapVk;
    RayTracingShaderMapType ShaderMapD3D12;
#if VULKAN_SUPPORTED
    SerializedPipelineStateImpl::ExtractShadersVk(PSOCreateInfo, ShaderMapVk);     // <-- runs on a D3D12 app
    VERIFY_EXPR(!ShaderMapVk.empty());
#endif
#if D3D12_SUPPORTED
    SerializedPipelineStateImpl::ExtractShadersD3D12(PSOCreateInfo, ShaderMapD3D12);
    VERIFY_EXPR(!ShaderMapD3D12.empty());
#endif

    VERIFY(ShaderMapVk.empty() || ShaderMapD3D12.empty() || ShaderMapVk == ShaderMapD3D12,
           "Ray tracing shader map must be same for Vulkan and Direct3D12 backends");

    RayTracingShaderMapType ShaderMap;
    if      (!ShaderMapVk.empty())    std::swap(ShaderMap, ShaderMapVk);
    else if (!ShaderMapD3D12.empty()) std::swap(ShaderMap, ShaderMapD3D12);
    else                              return;
    /* ... use ShaderMap to remap group shader pointers to indices ... */
}
```

Why this exists: a ray-tracing PSO records its shaders inside *shader groups* (raygen / miss /
hit groups). To serialize those groups portably, the archiver needs a stable `IShader* → index`
map. When an archive targets **both** Vulkan and D3D12 (the offline Render State Packager use
case), both maps are populated and the `VERIFY` asserts they are identical so a single group table
serves both backends.

The outer logic is even written to *tolerate* one backend being absent — note the
`ShaderMapVk.empty() || ShaderMapD3D12.empty() || ...` guard and the "use whichever is non-empty"
selection. The intent was clearly "if only one backend has data, just use it."

**The defect:** `ExtractShadersVk` cannot *produce* an empty map gracefully when there is no
Vulkan data — it crashes first. It calls the generic extractor, which for each shader does
([Archiver_Vk.cpp:82-101](../../../../DiligentCore/Graphics/Archiver/src/Archiver_Vk.cpp)):

```cpp
inline const ShaderVkImpl* GetShaderVk(const SerializedShaderImpl* pShader) {
    const CompiledShaderVk* p = pShader->GetShader<const CompiledShaderVk>(DeviceType::Vulkan);
    return p != nullptr ? &p->ShaderVk : nullptr;            // <-- nullptr: m_Shaders[Vulkan] is empty
}
struct ShaderStageInfoVk : PipelineStateVkImpl::ShaderStageInfo {
    void Append(const SerializedShaderImpl* pShader) {
        ShaderStageInfo::Append(GetShaderVk(pShader));        // <-- Append(nullptr)
        Serialized.push_back(pShader);
    }
};
```

and `PipelineStateVkImpl::ShaderStageInfo::Append`
([PipelineStateVkImpl.cpp:715](../../../../DiligentCore/Graphics/GraphicsEngineVulkan/src/PipelineStateVkImpl.cpp)):

```cpp
void PipelineStateVkImpl::ShaderStageInfo::Append(const ShaderVkImpl* pShader) {
    VERIFY_EXPR(pShader != nullptr);                          // <-- ASSERTION FIRES HERE
    VERIFY(/* dedup check */, "Shader '", pShader->GetDesc().Name, "' ...");
    const SHADER_TYPE NewShaderType = pShader->GetDesc().ShaderType;   // <-- would deref null if continued
    ...
}
```

`GetShader` is a blind `static_cast` of the array slot
([SerializedShaderImpl.hpp:93-96](../../../../DiligentCore/Graphics/Archiver/include/SerializedShaderImpl.hpp)),
so it returns null **iff** `m_Shaders[Vulkan]` is null — which, for a D3D12-only archive, it
always is.

---

## Why only ray-tracing PSOs

| Pipeline type | Shader serialization path | Probes all compiled backends? |
|---|---|---|
| Graphics / Compute / Tile | `SerializePSOCreateInfo` serializes shaders inline, driven by `ArchiveInfo.DeviceFlags` via `PatchShadersXxx` | **No** — runtime device flag only |
| **Ray tracing** | `SerializePSOCreateInfo` builds a cross-backend shader→index map via `ExtractShadersVk` + `ExtractShadersD3D12` | **Yes** — compile-time `#if XXX_SUPPORTED` |

Only the ray-tracing overload performs the all-backends extraction, so only ray-tracing PSOs hit
the absent-backend shader.

The bug is symmetric: running on **Vulkan** in a build that also enabled D3D12 would fail the same
way inside `ExtractShadersD3D12` (`m_Shaders[D3D12]` null). The failing extractor is always the
*non-running* compiled-in backend.

---

## Severity — not harmless

- **Debug build:** `VERIFY_EXPR` breaks into the debugger at the assertion. The immediately
  following `pShader->GetDesc()` dereferences null, so *continuing* past the break leads to an
  access violation rather than a clean recovery.
- **Release build:** `VERIFY_EXPR` compiles out, so `Append(nullptr)` proceeds directly to
  `pShader->GetDesc()` → access violation.
- The archiving call is wrapped in `try { ... } catch (...) {}`
  ([RenderStateCacheImpl.cpp:909](../../../../DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp)),
  but under MSVC's default `/EHsc` a hardware access violation is an SEH exception that
  `catch (...)` does **not** catch, so it is not silently swallowed.

The real device PSO (RenderStateCacheImpl.cpp:888) was already created successfully, so *rendering*
is unaffected — the failure is confined to the disk-archiving step. But the step cannot complete:
the ray-tracing PSO is never written to `RTXPT.cache`, and the process asserts/crashes at the
attempt.

> Note: An earlier hypothesis blamed a stale/partial `RTXPT.cache` and a re-serialize-from-bytecode
> path. The D3D12 detail disproves it — the trigger is the compile-time backend gating above and
> reproduces on a **fresh** cache, on the very first ray-tracing PSO.

---

## Reproduction conditions

All of the following must hold:

1. DiligentCore built with **more than one** PSO-capable backend (e.g. D3D12 **and** Vulkan — the
   default Windows configuration).
2. Application runs on one of them (D3D12 here).
3. A **ray-tracing** PSO is created through an `IRenderStateCache` (which auto-archives it).

Compute/graphics-only usage, or a single-backend build, does not reproduce it.

---

## Options

### A. RTXPT-side: keep RT shaders cached, create the RT PSO on the device (recommended, low-risk)

Route the five RT shaders through the cache as today (`pStateCache->CreateShader`) — the expensive
HLSL→DXIL compile of the path tracer stays cached and archived (shader archiving does **not** hit
this bug) — but create the ray-tracing **PSO** directly on the device:

```cpp
// RTXPTRayTracingPass.cpp, CreateVariant lambda (~line 381)
pDevice->CreateRayTracingPipelineState(PSOCreateInfo, &State.PSO);   // was: pStateCache->CreateRayTracingPipelineState(...)
```

Effect: avoids the broken RT-PSO archiving path entirely; only the RT PSO itself is not archived
(cheap pipeline relink — the shaders it references are still served from the cache). Slightly
narrows the "route every PSO through the cache" goal for this one PSO.

### B. Upstream DiligentCore fix (correct, broader)

Make the ray-tracing `SerializePSOCreateInfo` respect the archive's actual device flags instead of
the compile-time macros. Either:

- Pass `ArchiveInfo.DeviceFlags` into `SerializePSOCreateInfo` and only call `ExtractShadersVk` /
  `ExtractShadersD3D12` for flags that are set; or
- Make `GetShaderVk` / `ShaderStageInfoVk::Append` (and the D3D12 equivalents) tolerate a missing
  per-backend blob by skipping the shader and returning an **empty** map — the existing
  `ShaderMapVk.empty() || ShaderMapD3D12.empty() || ...` guard and "use whichever is non-empty"
  selection already handle a single populated map.

Either change keeps multi-backend offline packaging working while letting the single-backend
runtime `RenderStateCache` archive ray-tracing PSOs.

### C. Do nothing in code

Acceptable only if the RT PSO is never routed through the cache. With it routed, the run
asserts/crashes in the archiving step, so this is not viable as-is.

---

## Key references

| What | Location |
|---|---|
| Compile-time backend gating (root cause) | `DiligentCore/Graphics/Archiver/src/SerializedPipelineStateImpl.cpp:108-145` |
| Vk extractor that dereferences absent shader | `DiligentCore/Graphics/Archiver/src/Archiver_Vk.cpp:82-105`, `:268-276` |
| Assertion site | `DiligentCore/Graphics/GraphicsEngineVulkan/src/PipelineStateVkImpl.cpp:715` |
| Per-backend compiled-blob array | `DiligentCore/Graphics/Archiver/include/SerializedShaderImpl.hpp:93-121` |
| Cache scopes archive to running device only | `DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp:454-456`, `:915-918` |
| Archiving wrapped in `catch(...)` | `DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp:909-930` |
| RTXPT RT PSO creation call site | `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp:381` |
