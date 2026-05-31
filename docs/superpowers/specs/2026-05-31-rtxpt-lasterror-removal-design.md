# RTXPT LastError Removal Design

## Summary

This design removes `LastError` and `m_LastError` as status or communication channels from the RTXPT sample. Error reporting moves to the Diligent logging and diagnostic macros at the failure site:

- Use `LOG_ERROR_MESSAGE` for recoverable or non-fatal errors where the sample should keep running, such as missing or invalid `.scene.json` files and optional resource fallback paths.
- Use `VERIFY`, `VERIFY_EXPR`, `DEV_ERROR`, or `UNEXPECTED` for fatal, unexpected, or invariant-breaking failures, such as PSO/SRB/SBT creation failure or missing shader variables after a matching resource layout was declared.

The existing `bool` return values remain the control-flow signal for callers. `DisabledReason` remains the user-visible status channel for expected feature/capability disablement.

## Confirmed Requirements

- Remove `LastError` fields from RTXPT stats structs.
- Remove `m_LastError` members and `GetLastError()` accessors from helper classes.
- Stop displaying `LastError` strings in `RTXPTSample::UpdateUI`.
- Log recoverable failures immediately with `LOG_ERROR_MESSAGE`.
- Assert or report development errors for unexpected failures with Diligent diagnostic macros.
- Keep existing successful behavior and fallback behavior unchanged.
- Keep `DisabledReason` for expected feature disablement, such as unsupported ray tracing, standalone RT shaders, compute shaders, or unavailable frame constants.

## Non-Goals

- Do not change renderer algorithms, shader code, ray tracing dispatch behavior, scene selection behavior, or accumulation behavior.
- Do not introduce exceptions or process termination as a new control-flow mechanism.
- Do not add a replacement UI error-state field.
- Do not refactor unrelated RTXPT lifecycle or resource ownership.
- Do not edit historical plan documents that mention `LastError`; those are archival records.

## Affected Area

The change is limited to `DiligentSamples/Samples/RTXPT/src`.

The primary data structures to simplify are:

- `RTXPTMaterialStats`
- `RTXPTLightStats`
- `RTXPTAccelerationStructureStats`
- `RTXPTRayTracingPassStats`
- `RTXPTComputePassStats`
- `RTXPTSkinnedGeometryStats`
- `RTXPTScene`
- `RTXPTRenderTargets`
- `RTXPTBlitPass`

`RTXPTSample::UpdateUI` must stop reading these removed fields and accessors.

## Error Handling Design

### Recoverable Errors

Recoverable errors are logged with `LOG_ERROR_MESSAGE` at the point where the failure is detected. The function then follows its existing control flow, usually returning `false` or continuing with an existing fallback.

Examples:

- Empty or missing RTXPT scene file name.
- Missing `.scene.json` file.
- Invalid scene JSON or missing `models[0]`.
- Missing glTF file.
- `GLTF::Model` construction throwing while loading user-provided scene data.
- Material texture missing or material texture view creation failure; texture binding should remain disabled as it does today.
- RGBA32F accumulation UAV unsupported; accumulation remains unavailable and the sample keeps rendering through the fallback output.
- Render target creation failure where the caller can keep the sample alive through the fallback path.

The scene camera parser remains a soft optional path. If camera extraction fails, scene loading can still succeed and the sample continues to use the default camera behavior.

### Unexpected Or Fatal Failures

Unexpected failures use Diligent diagnostics. In debug/development builds these should surface loudly; in release builds the existing `bool` return path still prevents dereferencing null resources.

Examples:

- Shader creation failure for sample-owned shaders.
- Graphics, compute, or ray tracing PSO creation failure.
- SRB or SBT creation failure after PSO creation.
- Static shader variable missing after the resource layout explicitly declared it.
- Required shader-resource binding missing during trace, blit, or skinning dispatch.
- Internal resource invariants failing during BLAS/TLAS build or update.

Where a function needs to preserve release-mode control flow, use the pattern:

```cpp
VERIFY(Resource, "Failed to create RTXPT resource");
if (!Resource)
    return false;
```

Use `UNEXPECTED` or `DEV_ERROR` when the failure is not attached to a single boolean resource expression or when an impossible state is detected.

## UI Design

The RTXPT ImGui panel keeps capability and runtime status rows, but no longer renders stored `LastError` text.

Keep:

- Scene loaded/missing status.
- Material, light, acceleration structure, skinning, RT pass, compute pass, render target, and blit status counters.
- `DisabledReason` rows for expected disabled features.

Remove:

- Asset load error row from `RTXPTScene::GetLastError()`.
- Material/light/AS/skinning/RT/compute error rows from `Stats.LastError`.
- Render target and blit error rows from `GetLastError()`.

## Validation

Targeted validation should include:

- `rg "LastError|GetLastError|m_LastError|m_Stats\\.LastError" DiligentSamples/Samples/RTXPT/src` returns no active source matches.
- `rg "LOG_ERROR_MESSAGE|VERIFY\\(|VERIFY_EXPR|DEV_ERROR|UNEXPECTED" DiligentSamples/Samples/RTXPT/src` shows the new error-reporting sites.
- CMake/build verification for the RTXPT sample or the smallest available DiligentSamples target in the local checkout.
- If a full build is unavailable, at minimum run static source searches and report that build verification could not be executed.
