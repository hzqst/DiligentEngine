# `#include` resolution inconsistency: DXC's include handler vs. `ProcessShaderIncludesImpl`

## TL;DR

DiligentCore resolves shader `#include "X"` directives **two different ways**:

- **At compile time**, DXC (a real preprocessor) resolves each include **relative to the
  directory of the file doing the including**.
- **At scan time**, DiligentCore's own `ProcessShaderIncludesImpl` — a hand-rolled text scanner
  used to compute the `BY_CONTENT` render-state-cache hash (and for hot-reload dependency
  tracking and archiver include enumeration) — passes the **literal include string** to the
  stream factory with **no parent-directory resolution**, so it only finds includes reachable by
  their bare name from the factory's flat search-dir list.

For shaders that write includes relative to their own location (RTXPT's nested
`PathTracer/Lighting/…` tree), DXC compiles cleanly while the scanner throws
`Failed to load shader source file 'LightingConfig.h'`. This is a **latent DiligentCore bug**,
not an RTXPT bug. RTXPT works around it by hashing the cache `BY_NAME` (which never invokes the
scanner). The proper fix is to make `ProcessShaderIncludesImpl` resolve nested includes against
their parent file's directory, the way DXC does.

---

## Symptom

With the render-state cache in `RENDER_STATE_CACHE_FILE_HASH_MODE_BY_CONTENT` (the default):

```
ERROR: Failed to create input stream for source file LightingConfig.h
ERROR: Failed to load shader source file 'LightingConfig.h'
[std::runtime_error / std::pair exceptions]
ERROR: Failed to process includes in file 'PathTracer/Lighting/LightsBaker.hlsl': Unknown error
```

The shader **compiles fine** (DXC resolves the include) — only the *content-hash* step, which
re-scans the source to enumerate includes, fails. `LightsBaker.hlsl` lives in
`shaders/PathTracer/Lighting/` and does `#include "LightingConfig.h"` (a sibling), but the stream
factory's search dirs are `shaders;shaders\PathTracer` — neither contains `Lighting/`.

---

## The two resolution paths

### DXC's handler resolves relative to the including file

DXC tracks the *current file* being compiled and **joins the include string with that file's
directory before** calling the include handler. By the time
[`DxcIncludeHandlerImpl::LoadSource`](../../../../DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp)
runs (DXCompiler.cpp:185), `pFilename` is already the resolved relative path — e.g. compiling
`PathTracer/Lighting/LightsBaker.hlsl` with `#include "LightingConfig.h"` hands the handler
`.\PathTracer\Lighting\LightingConfig.h`. The handler just strips a leading `.\` and opens it:

```cpp
// DXCompiler.cpp:210-214
if (fileName.size() > 2 && fileName[0] == '.' && (fileName[1] == '\\' || fileName[1] == '/'))
    fileName.erase(0, 2);                         // <-- proof DXC prepended the parent directory
m_pStreamFactory->CreateInputStream(fileName.c_str(), &pSourceStream);
```

`shaders/` (search dir) + `PathTracer/Lighting/LightingConfig.h` → **found**. ✔

### The scanner passes the literal include text verbatim

[`ProcessShaderIncludesImpl`](../../../../DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp)
(ShaderToolsCommon.cpp:428) is a text scanner. `FindIncludes` extracts the raw string *between
the quotes* (`LightingConfig.h`) and, when it recurses, sets the child's path to that **bare
name**, discarding the parent's directory:

```cpp
// ShaderToolsCommon.cpp:439-448
[&](const std::string& FilePath, size_t Start, size_t End)
{
    if (!Includes.insert(FilePath).second)
        return;

    ShaderCreateInfo IncludeCI{ShaderCI};
    IncludeCI.FilePath     = FilePath.c_str();   // "LightingConfig.h" — parent dir thrown away
    IncludeCI.Source       = nullptr;
    IncludeCI.SourceLength = 0;
    ProcessShaderIncludesImpl(IncludeCI, Includes, IncludeHandler);
}
```

`ReadShaderSourceFile` then calls `CreateInputStream("LightingConfig.h")`, which only sees the
factory's flat search dirs (`shaders;shaders\PathTracer`), none of which contain `Lighting/`, and
throws (ShaderToolsCommon.cpp:243-245). ✘

---

## The crux

The parent path **is** available — `ShaderCI.FilePath` holds
`PathTracer/Lighting/LightsBaker.hlsl` at the moment of recursion — but the scanner never combines
it with the include name. So:

| | Include resolution rule |
|---|---|
| **DXC** | parent-file-relative, then include search paths (real C/preprocessor semantics) |
| **Diligent scanner** | flat stream-factory search-dir list only |

They agree **only** when every include is reachable from a search dir by its bare name (e.g.
Tutorial26's flat assets). RTXPT writes includes relative to each file's own location, so DXC
compiles cleanly while the scanner — invoked only to compute the `BY_CONTENT` cache hash — fails.

It is a genuine latent bug, not just a quirk: **any** project with directory-relative shader
includes hits it under content hashing, hot-reload dependency tracking
(`ShaderReloadFactory`/`RenderStateCache::Reload`), or the archiver's include enumeration.

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

Affected whenever `ProcessShaderIncludesImpl` runs over shaders with directory-relative includes:

- `RENDER_STATE_CACHE_FILE_HASH_MODE_BY_CONTENT` — content hashing (the failure above).
- Hot reload — `ProcessShaderIncludes` builds the include→dependency set for `Reload()`.
- Archiver — include enumeration when packaging shader source.

Not affected: actual DXC/FXC compilation (resolves correctly), and `BY_NAME` hashing.

---

## Proposed fix

Make `ProcessShaderIncludesImpl` resolve each nested include against the **directory of the
including file** before recursing — mirroring `DxcIncludeHandlerImpl`. DiligentCore already
provides the needed helpers in `BasicFileSystem.hpp` (already included by `ShaderToolsCommon.cpp`):
`FileSystem::GetPathComponents`, `FileSystem::IsPathAbsolute`, `FileSystem::SimplifyPath`,
`FileSystem::SlashSymbol`.

### Minimal version (fixes the common case)

In the `FindIncludes` recursion lambda (ShaderToolsCommon.cpp:439-448), join the include with the
parent directory and normalize:

```cpp
[&](const std::string& IncludeName, size_t /*Start*/, size_t /*End*/)
{
    // Resolve the include relative to the directory of the including file,
    // the way DXC/FXC (and DxcIncludeHandlerImpl) do. Without this, a nested
    // include like "LightingConfig.h" inside PathTracer/Lighting/LightsBaker.hlsl
    // loses its parent directory and cannot be found via the flat search dirs.
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

Because the recursion now stores the **resolved** path in `IncludeCI.FilePath`, deeper includes
compose correctly: `A/B/file.hlsl` → `"C/inc.h"` → `A/B/C/inc.h` → `"deep.h"` → `A/B/C/deep.h`,
matching DXC.

### Robust version (matches DXC's two-phase lookup)

DXC tries **parent-relative first, then the include search paths**. A search-dir-relative include
(e.g. a shared `Common.hlsli` that lives directly in a search dir but is included from a nested
file) resolves under the bare name, not the parent-relative one. To match this, probe the stream
factory and fall back to the bare name when the parent-relative path can't be opened:

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

    // Phase 1: parent-file-relative. Phase 2 (fallback): search-dir/bare name.
    if (ShaderCI.pShaderSourceStreamFactory != nullptr)
    {
        RefCntAutoPtr<IFileStream> pStream;
        ShaderCI.pShaderSourceStreamFactory->CreateInputStream(Relative.c_str(), &pStream);
        if (pStream)
            return Relative;
        return IncludeName;   // let the existing error path report the original name if this also fails
    }
    return Relative;
}
```

### Notes / risks

- **Hash stability:** the values inserted into `Includes` change from bare names to resolved
  paths, so `BY_CONTENT` hashes shift once. This is a one-time cache invalidation, not a
  correctness problem.
- **Backward compatibility:** flat-asset projects (e.g. Tutorial26) still resolve, because the
  robust version falls back to the bare name; the minimal version keeps working as long as the
  parent-relative path is reachable from a search dir (it is, since the parent file itself was
  found that way).
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
   (`PathTracer/Lighting/LightingConfig.h`) rather than bare names.

If the fix is upstreamed, RTXPT can drop the `BY_NAME` workaround and regain source-edit cache
invalidation.

---

## Key references

| What | Location |
|---|---|
| Scanner that discards the parent dir (root cause) | `DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp:428-454` |
| Scanner throws on unresolved include | `DiligentCore/Graphics/ShaderTools/src/ShaderToolsCommon.cpp:224-245` |
| DXC handler — resolves parent-relative, strips `.\` | `DiligentCore/Graphics/ShaderTools/src/DXCompiler.cpp:185-237` |
| `BY_NAME` hashing (never calls the scanner) | `DiligentCore/Graphics/GraphicsTools/src/RenderStateCacheImpl.cpp` (`HashShaderCIByFileName`) |
| Path helpers for the fix | `DiligentCore/Platforms/Basic/interface/BasicFileSystem.hpp:171-197` |
| RTXPT workaround (`BY_NAME`) | `DiligentSamples/Samples/RTXPT/src/RTXPTSample.cpp` (cache create block) |
