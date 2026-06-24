# `#include` resolution inconsistency: compiler include handlers vs. shader include preprocessing

## TL;DR

DiligentCore resolves shader `#include "X"` directives **two different ways**:

- **At compile time**, DXC/FXC maintain include-stack context. Once an include file has been loaded
  as e.g. `Lighting/LightingTypes.hlsli`, a nested `#include "LightingConfig.h"` inside that
  file is resolved relative to the loaded include path, so the compiler can request
  `Lighting/LightingConfig.h`.
- **At scan/unroll time**, DiligentCore's own hand-rolled include paths
  (`ProcessShaderIncludesImpl` for hashing/hot-reload dependency tracking and
  `UnrollShaderIncludesImpl` for GL archiver source packaging) used to pass the
  **literal include string** to the stream factory with **no include-stack /
  parent-directory resolution**, so the same nested include was looked up as bare
  `LightingConfig.h`.

For shaders with nested directory-relative includes (RTXPT's `PathTracer/Lighting/…` tree), DXC/FXC
compiles cleanly while the scanner throws `Failed to load shader source file 'LightingConfig.h'`.
This is a **latent DiligentCore bug**, not an RTXPT bug. RTXPT works around it by hashing the
cache `BY_NAME` (which never invokes the scanner). The proper fix is to make both
`ProcessShaderIncludesImpl` and `UnrollShaderIncludesImpl` resolve nested includes against
their parent include file's path, matching DXC/FXC include-stack behavior.

---

## Symptom

With the render-state cache in `RENDER_STATE_CACHE_FILE_HASH_MODE_BY_CONTENT` (the default):

```
ERROR: Failed to create input stream for source file LightingConfig.h
ERROR: Failed to load shader source file 'LightingConfig.h'
[std::runtime_error / std::pair exceptions]
ERROR: Failed to process includes in file 'PathTracer/Lighting/LightsBaker.hlsl': Unknown error
```

The shader **compiles fine** (DXC/FXC resolve the nested include) — only the *content-hash* step,
which re-scans the source to enumerate includes, fails. In the current RTXPT chain,
`LightsBaker.hlsl` includes `Lighting/LightingTypes.hlsli`; then `LightingTypes.hlsli` includes
the sibling `LightingConfig.h`. DXC/FXC resolve the second include as `Lighting/LightingConfig.h`,
but the scanner tries bare `LightingConfig.h` against the flat search dirs
`shaders;shaders\PathTracer`, neither of which contains the file at that bare name.

---

## The two resolution paths

### DXC/FXC keep include-stack context for nested includes

Diligent builds a source string and passes it to DXC/FXC with an empty source name; the emitted
`#line 1 "PathTracer/Lighting/LightsBaker.hlsl"` marker is useful for diagnostics, but it is not
a reliable mechanism for resolving a top-level sibling include relative to `ShaderCI.FilePath`.

The RTXPT case that compiles is nested: `LightsBaker.hlsl` includes
`Lighting/LightingTypes.hlsli`, which is reachable from the `shaders\PathTracer` search dir. Once
the compiler has loaded that include file, a sibling include inside it,
`#include "LightingConfig.h"`, is resolved relative to the loaded include path, so DXC/FXC ask
their include handlers for a path equivalent to `Lighting/LightingConfig.h`.

[`DxcIncludeHandlerImpl::LoadSource`](../../../../DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp)
(DXCompiler.cpp:185) and
[`D3DIncludeImpl::Open`](../../../../DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp)
(ShaderD3DBase.cpp:65) do not compute that parent directory themselves; they open the path
requested by the compiler. The DXC handler additionally strips a leading `.\` if DXC provides one:

```cpp
// DXCompiler.cpp:210-214
if (fileName.size() > 2 && fileName[0] == '.' && (fileName[1] == '\\' || fileName[1] == '/'))
    fileName.erase(0, 2);
m_pStreamFactory->CreateInputStream(fileName.c_str(), &pSourceStream);
```

`shaders\PathTracer` (search dir) + `Lighting/LightingConfig.h` → **found**. ✔

### The scanner/unroller passed the literal include text verbatim

[`ProcessShaderIncludesImpl`](../../../../DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp)
(ShaderToolsCommon.cpp:428) is a text scanner. It first opens `Lighting/LightingTypes.hlsli`;
inside that file, `FindIncludes` extracts the raw string between the quotes
(`LightingConfig.h`) and, when it recurses, sets the child's path to that **bare name**,
discarding the parent include path `Lighting/`:

```cpp
// ShaderToolsCommon.cpp:439-448
[&](const std::string& FilePath, size_t Start, size_t End)
{
    if (!Includes.insert(FilePath).second)
        return;

    ShaderCreateInfo IncludeCI{ShaderCI};
    IncludeCI.FilePath     = FilePath.c_str();   // "LightingConfig.h" — parent include path thrown away
    IncludeCI.Source       = nullptr;
    IncludeCI.SourceLength = 0;
    ProcessShaderIncludesImpl(IncludeCI, Includes, IncludeHandler);
}
```

`ReadShaderSourceFile` then calls `CreateInputStream("LightingConfig.h")`, which only sees the
factory's flat search dirs (`shaders;shaders\PathTracer`), none of which contain `Lighting/`, and
throws (ShaderToolsCommon.cpp:243-245). ✘

`UnrollShaderIncludesImpl` had the sibling form of the same bug: the recursive `IncludeCI.FilePath`
was set from the bare include string before inlining the include source. That path is used by
`Archiver_GL.cpp` when packaging GLSL source for the GL archiver.

---

## The crux

The parent include path **is** available — during the failing nested recursion,
`ShaderCI.FilePath` holds `Lighting/LightingTypes.hlsli` — but the scanner never combines it with
the nested include name. So:

| | Include resolution rule |
|---|---|
| **DXC/FXC** | include-stack-aware: a nested include is resolved relative to the loaded include file path, then search paths |
| **Diligent scanner** | literal include string only; no include-stack / parent-path context |

They agree **only** when every nested include is reachable from a search dir by the exact spelling
used in the source (e.g. Tutorial26's flat assets). RTXPT uses nested directory-relative includes,
so DXC/FXC compile cleanly while the scanner — invoked only to compute the `BY_CONTENT` cache hash —
fails.

It is a genuine latent bug, not just a quirk: **any** project with directory-relative shader
includes hits it under content hashing, hot-reload dependency tracking
(`ShaderReloadFactory`/`RenderStateCache::Reload`), or GL archiver source unrolling.

---

## Why RTXPT currently sidesteps it (`BY_NAME`)

RTXPT creates the cache with
`CacheCI.FileHashMode = RENDER_STATE_CACHE_FILE_HASH_MODE_BY_NAME`. The `BY_NAME` hash path
(`HashShaderCIByFileName` in RenderStateCacheImpl.cpp) keys on file path + entry point + macros +
compile flags and **never reads source or calls `ProcessShaderIncludesImpl`**, so the
inconsistency is never exercised. Trade-off: editing a shader's *source* no longer auto-invalidates
the cache (changing macros/entry point/flags still does); delete `RTXPT.cache` after a `.hlsl`
edit. This is a workaround in the sample, not a fix for the engine bug.

---

## Scope / impact

Affected whenever Diligent's hand-rolled include preprocessing runs over shaders with
directory-relative includes:

- `RENDER_STATE_CACHE_FILE_HASH_MODE_BY_CONTENT` — content hashing (the failure above).
- Hot reload — `ProcessShaderIncludes` builds the include→dependency set for `Reload()`.
- GL archiver — `UnrollShaderIncludes` inlines include source when packaging shader source.
- glslang compilation (Vulkan/WebGPU SPIR-V) — `IncluderImpl` in `GLSLangUtils.cpp`,
  Diligent's own include handler for glslang, served `"…"` includes with no
  include-stack context, so nested directory-relative includes failed to **compile**,
  not just to hash. See the note below.

Not affected: actual DXC/FXC compilation (resolves correctly via the vendor include
handlers), and `BY_NAME` hashing.

### Note: the glslang compile-time handler had the same bug (now fixed)

DXC/FXC are not the only compilers Diligent feeds. The Vulkan and WebGPU backends
compile through glslang, whose `#include` directives are served by Diligent's own
hand-rolled `IncluderImpl` (`GLSLangUtils.cpp`), **not** by a vendor include handler.
That handler had the **same class of bug** as the scanner: `includeLocal` returned
`nullptr`, so every `"…"` include fell through to a literal-name lookup with no
include-stack / parent-path context — exactly like `ProcessShaderIncludesImpl`. Nested
parent-relative includes therefore failed to *compile* on Vulkan/WebGPU, not merely to
hash under `BY_CONTENT`.

This was fixed in commits `b5c04a18` and `da8a8de0`: the top-level source name
(`ShaderCI.FilePath`) is now passed to glslang, and `IncluderImpl::includeLocal`
resolves a nested include against the including file's directory first, falling back to
the search dirs — matching DXC/FXC include-stack behavior. The scanner/unroller fixes
(`ProcessShaderIncludesImpl` / `UnrollShaderIncludesImpl`) do **not** cover glslang: it
is a separate code path with its own include handler, so don't assume the scanner
commits closed this one.

---

## Proposed fix

Make `ProcessShaderIncludesImpl` and `UnrollShaderIncludesImpl` resolve each nested include
against the **path of the currently scanned/unrolled include file** before recursing — matching
DXC/FXC include-stack behavior. DiligentCore
already provides the needed helpers in `BasicFileSystem.hpp` (already included by
`ShaderToolsCommon.cpp`): `FileSystem::GetPathComponents`, `FileSystem::IsPathAbsolute`,
`FileSystem::SimplifyPath`, `FileSystem::SlashSymbol`.

### Minimal version (only safe for parent-relative include trees)

In the `FindIncludes` recursion lambda (ShaderToolsCommon.cpp:439-448), a simple parent-directory
join fixes trees where includes are consistently parent-relative. It is **not** safe for mixed
search-dir-relative includes such as RTXPT's first `Lighting/LightingTypes.hlsli` include; use the
robust version below for the engine fix.

```cpp
[&](const std::string& IncludeName, size_t /*Start*/, size_t /*End*/)
{
    // Resolve the include relative to the path of the currently scanned file.
    // Without this, a nested include like "LightingConfig.h" inside
    // Lighting/LightingTypes.hlsli loses the "Lighting/" parent path and
    // cannot be found via the flat search dirs.
    std::string ResolvedPath = IncludeName;
    if (ShaderCI.FilePath != nullptr && !FileSystem::IsPathAbsolute(IncludeName.c_str()))
    {
        std::string ParentDir;
        FileSystem::GetPathComponents(ShaderCI.FilePath, &ParentDir, nullptr);
        if (!ParentDir.empty())
        {
            ResolvedPath = FileSystem::SimplifyPath(
                (ParentDir + FileSystem::SlashSymbol + IncludeName).c_str(),
                FileSystem::SlashSymbol);
        }
    }

    if (!Includes.insert(ResolvedPath).second)
        return;

    ShaderCreateInfo IncludeCI{ShaderCI};
    IncludeCI.FilePath     = ResolvedPath.c_str();   // resolved, not bare -> next level's parent dir is correct
    IncludeCI.Source       = nullptr;
    IncludeCI.SourceLength = 0;
    ProcessShaderIncludesImpl(IncludeCI, Includes, IncludeHandler);
}
```

Because the recursion now stores the **resolved** path in `IncludeCI.FilePath`, deeper
parent-relative includes compose correctly: `A/B/file.hlsl` → `"C/inc.h"` → `A/B/C/inc.h` →
`"deep.h"` → `A/B/C/deep.h`.

### Robust version (matches DXC/FXC include-stack lookup)

DXC/FXC's effective behavior is **current-include-relative first, then include search paths**. A
search-dir-relative include (e.g. RTXPT's first `Lighting/LightingTypes.hlsli`) should still
resolve under its original spelling, not under a blindly prepended parent directory. To match
this, probe the stream factory and fall back to the original include spelling when the
current-include-relative path can't be opened:

```cpp
static std::string ResolveIncludePath(const ShaderCreateInfo& ShaderCI, const std::string& IncludeName)
{
    if (ShaderCI.FilePath == nullptr || FileSystem::IsPathAbsolute(IncludeName.c_str()))
        return IncludeName;

    std::string ParentDir;
    FileSystem::GetPathComponents(ShaderCI.FilePath, &ParentDir, nullptr);
    if (ParentDir.empty())
        return IncludeName;

    const std::string Relative = FileSystem::SimplifyPath(
        (ParentDir + FileSystem::SlashSymbol + IncludeName).c_str(), FileSystem::SlashSymbol);

    // Phase 1: current-include-relative. Phase 2 (fallback): search-dir/original spelling.
    if (ShaderCI.pShaderSourceStreamFactory != nullptr)
    {
        RefCntAutoPtr<IFileStream> pStream;
        ShaderCI.pShaderSourceStreamFactory->CreateInputStream(Relative.c_str(), &pStream);
        if (pStream)
            return Relative;
        return IncludeName; // let the existing error path report the original name if this also fails
    }
    return Relative;
}
```

### Notes / risks

- **Hash stability:** the values inserted into `Includes` change from bare names to resolved
  paths, so `BY_CONTENT` hashes shift once. This is a one-time cache invalidation, not a
  correctness problem.
- **Backward compatibility:** flat-asset projects (e.g. Tutorial26) still resolve, because the
  robust version falls back to the original include spelling. The minimal version can break mixed
  trees where a nested file is first included by a search-dir-relative path.
- **Angle vs quote includes:** the scanner does not distinguish `<...>` from `"..."`. The robust
  (try-relative-then-bare) approach is a superset of correct behavior for both and is the safer
  choice.

---

## Testing the fix

1. With the fix applied, switch RTXPT back to
   `RENDER_STATE_CACHE_FILE_HASH_MODE_BY_CONTENT` and confirm cache population no longer logs
   `Failed to process includes in file 'PathTracer/Lighting/LightsBaker.hlsl'`.
2. Verify Tutorial26_StateCache (flat assets) still populates and reuses its cache.
3. Confirm the enumerated include set for a nested shader now contains resolved paths
   (`Lighting/LightingConfig.h`) rather than bare names.
4. Confirm `UnrollShaderIncludes` can inline a nested parent-relative include tree.

If the fix is upstreamed, RTXPT can drop the `BY_NAME` workaround and regain source-edit cache
invalidation.

---

## Key references

| What | Location |
|---|---|
| Scanner/unroller that discarded the parent dir (root cause) | `DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp` (`ProcessShaderIncludesImpl`, `UnrollShaderIncludesImpl`) |
| Scanner throws on unresolved include | `DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp:224-245` |
| GL archiver source inlining path | `DiligentCore/Graphics/Archiver/src/Archiver_GL.cpp` (`UnrollSource`) |
| DXC handler — opens the include path requested by DXC and strips leading `.\` | `DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp:185-237` |
| FXC handler — opens the include path requested by D3DCompile | `DiligentCore/Graphics/GraphicsEngineD3DBase/src/ShaderD3DBase.cpp:57-96` |
| glslang handler — same class of bug, fixed in `b5c04a18`/`da8a8de0` (see note) | `DiligentCore/Graphics/ShaderTools/src/GLSLangUtils.cpp` (`IncluderImpl::includeLocal`) |
| `BY_NAME` hashing (never calls the scanner) | `DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp` (`HashShaderCIByFileName`) |
| Path helpers for the fix | `DiligentCore/Platforms/Basic/interface/BasicFileSystem.hpp:171-197` |
| RTXPT workaround (`BY_NAME`) | `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` (cache create block) |
