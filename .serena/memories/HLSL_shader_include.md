# HLSL Shader Include Path Pitfall

## Trigger Signal

Runtime shader compilation fails in Diligent with errors like:

```text
Failed to create input stream for source file <IncludeName>
Failed to open shader include file <IncludeName>. Check that the file exists
fatal error: '<IncludeName>' file not found
```

A common RTXPT example is a shader under `Samples/RTXPT/assets/shaders/PathTracer/Lighting/` including a sibling file as:

```hlsl
#include "LightingTypes.hlsli"
```

while the shader is compiled at runtime with `ShaderCI.FilePath = "PathTracer/Lighting/LightsBaker.hlsl"` and source factory roots such as `"shaders;shaders\\PathTracer"`.

## Root Cause / Constraint

Diligent's DXC include handler (`DxcIncludeHandlerImpl::LoadSource`) passes the include string directly to `IShaderSourceInputStreamFactory::CreateInputStream`. `DefaultShaderSourceStreamFactory` searches only its configured roots plus the include string; it does not resolve includes relative to the current shader file's directory.

So `#include "LightingTypes.hlsli"` is searched as:

- `shaders/LightingTypes.hlsli`
- `shaders/PathTracer/LightingTypes.hlsli`

It is not automatically searched as:

- `shaders/PathTracer/Lighting/LightingTypes.hlsli`

## Correct Practice

Write runtime-compiled HLSL includes relative to one of the configured shader source roots, not relative to the including file, unless that directory is explicitly added to the source factory roots.

For RTXPT passes using `CreateDefaultShaderSourceStreamFactory("shaders;shaders\\PathTracer", ...)`, include files under `PathTracer/Lighting/` as:

```hlsl
#include "Lighting/LightingTypes.hlsli"
```

Existing sample pattern: `PathTracer/EmissiveTriangleBuild.hlsl` can include `PathTracerShared.h` because `shaders/PathTracer` is already one of the configured roots.

Prefer fixing the include path over broadening the source factory search roots when only one shader include is wrong; this keeps the runtime search behavior predictable.

Nested include caveat from RTXPT R5: when `PathTracerSample.rgen` includes `"Utils/StatelessSampleGenerators.hlsli"`, a sibling include inside that file must be written as `#include "SampleGenerators.hlsli"`, not `#include "Utils/SampleGenerators.hlsli"`. The latter can be normalized by DXC/Diligent as `Utils/Utils/SampleGenerators.hlsli` and fail at runtime.

RTXPT ToneMapper caveat from P3: when a shader is compiled with `ShaderCI.FilePath = "PostProcessing/ToneMapper/ToneMapping.hlsl"` and source roots such as `"shaders"`, sibling includes inside `PostProcessing/ToneMapper/` must not repeat the current directory prefix. This broken pattern:

```hlsl
#include "PostProcessing/ToneMapper/ToneMappingShared.h"
#include "PostProcessing/ToneMapper/ToneMapping.ps.hlsli"
```

can fail at runtime as a doubled path:

```text
PostProcessing\ToneMapper\PostProcessing\ToneMapper\ToneMappingShared.h
```

Use sibling includes instead:

```hlsl
#include "ToneMappingShared.h"
#include "ToneMapping.ps.hlsli"
```

Validation used for the RTXPT ToneMapper fix:

```powershell
rg -n "PostProcessing/ToneMapper/ToneMappingShared|PostProcessing/ToneMapper/ToneMapping\\.ps\\.hlsli" DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/ToneMapper
& 'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.26100.0\\x86\\dxc.exe' -T ps_6_0 -E main_ps -I DiligentSamples\\Samples\\RTXPT\\assets\\shaders DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PostProcessing\\ToneMapper\\ToneMapping.hlsl
& 'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.26100.0\\x86\\dxc.exe' -T cs_6_0 -E capture_cs -I DiligentSamples\\Samples\\RTXPT\\assets\\shaders DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PostProcessing\\ToneMapper\\ToneMapping.hlsl
& 'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.26100.0\\x86\\dxc.exe' -T ps_6_0 -E main -I DiligentSamples\\Samples\\RTXPT\\assets\\shaders DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PostProcessing\\ToneMapper\\Luminance.psh
cmake --build build\\x64\\Debug --config Debug --target RTXPT
```

Expected: the `rg` command has no matches; all three `dxc` commands and the RTXPT target build exit 0.

## Validation

Before claiming the fix, verify both path resolution and shader compilation:

```powershell
$roots = @('DiligentSamples\\Samples\\RTXPT\\assets\\shaders', 'DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PathTracer')
foreach ($name in @('LightingTypes.hlsli','Lighting\\LightingTypes.hlsli')) {
  $matches = @($roots | ForEach-Object { Join-Path $_ $name } | Where-Object { Test-Path $_ })
  '{0}: {1}' -f $name, ($(if ($matches.Count -gt 0) { 'found -> ' + ($matches -join '; ') } else { 'not found' }))
}
```

Expected for the RTXPT LightsBaker case:

```text
LightingTypes.hlsli: not found
Lighting\LightingTypes.hlsli: found -> ...\PathTracer\Lighting\LightingTypes.hlsli
```

Then compile the affected entries with equivalent include roots:

```powershell
dxc -T cs_6_0 -E ClearFeedbackCS -I DiligentSamples\\Samples\\RTXPT\\assets\\shaders -I DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PathTracer DiligentSamples\\Samples\\RTXPT\\assets\\shaders\\PathTracer\\Lighting\\LightsBaker.hlsl
```

Also run the target build when possible:

```powershell
cmake --build build\\x64\\Debug --config Debug --target RTXPT
```

## Applicable Scope

Applies to Diligent runtime shader compilation paths that use `CreateDefaultShaderSourceStreamFactory`, especially RTXPT compute/raytracing shaders under `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/`.