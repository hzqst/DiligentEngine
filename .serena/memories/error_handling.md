# RTXPT Error Handling

## Scope

Applies to `DiligentSamples/Samples/RTXPT` code paths, especially RTXPT pass initialization, required GPU feature checks, required buffer/resource wiring, shader/PSO/SRB/pass resource creation, and runtime pass execution guards.

## Rules

- Do not store or propagate failure text through `Stats::DisabledReason` in RTXPT. Stats should expose readiness/state counters only; failure details belong in the logging/assertion path.
- Treat missing required GPU capabilities as fatal for RTXPT passes. `ComputeShaders == false`, `RayTracing == false`, and missing standalone ray tracing shader support should use `DEV_ERROR` or `UNEXPECTED` and return failure instead of silently disabling a pass.
- Treat missing required inputs as fatal/invariant failures. Examples include missing frame constants, TLAS, required vertex/skinned-vertex/index buffers, scene buffers, render targets, or required device/factory/context objects. Use `DEV_ERROR` or `VERIFY`/`VERIFY_EXPR` depending on the local pattern, then return failure if the API reports `bool`.
- Treat shader, PSO, SRB, sampler, constant buffer, luminance resource, acceleration structure, and other required pass-resource creation failures as fatal/unexpected. Prefer `DEV_ERROR` where a development-build diagnostic should be emitted; use `UNEXPECTED` for impossible internal states or missing reflected shader variables.
- Use `LOG_ERROR_MESSAGE` only for genuinely non-fatal, recoverable user/data/environment issues where RTXPT can continue with an intentional fallback or where invalid external content is being reported.

## UI/Stats

RTXPT debug UI should report readiness/counters from stats, but must not display stored disabled-reason strings. If a pass is not ready, the reason should already have been emitted by the failing code path.

## Verification

After touching RTXPT error handling, run a targeted search such as `rg "DisabledReason|FeatureDisabled" DiligentSamples/Samples/RTXPT` and build or compile-check the affected RTXPT target when available.