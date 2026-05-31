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