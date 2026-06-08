# Realtime BxDF Debugging Notes

Date: 2026-06-06

This note records the RTXPT realtime-mode BxDF/transmission debugging state so a new
conversation can continue without repeating already completed checks.

## Scope

- Scene under test: `convergence-test.scene.json`.
- Mode under test: realtime / stable planes.
- Reference mode has been reported as visually correct after the BxDF fixes.
- Realtime mode remains incorrect: the scene has repeatedly shown black output or
  diagnostic colors. The latest reported output before this note contained red/green.

## Main Symptom History

- Initial realtime symptom: most non-emissive/non-reflective objects were black.
- After several diagnostic passes, the output evolved through blue, green, yellow/green,
  red/green, and remains red/green at the latest report.
- Transmission is still missing in realtime for the right-side balls in
  `convergence-test.scene.json`.

## Earlier Assertion Failures

These were investigated separately from the current red/green issue.

- `RTXPT static shader variable is missing: t_Lights for variant: Reference`
- `RTXPT static shader variable is missing: g_MiniConst for variant: FillStablePlanes`
- `RTXPT static shader variable is missing: mini constants (g_MiniConst) for pass:
  RelaxDenoiserFinalMerge`

The current investigation has moved past these assertion failures. The latest issue is
not believed to be a missing direct-light static binding.

## Direct-Light Bridge Findings

Per-variant static reflection logs were added for:

- `t_Lights`
- `t_LightingControl`
- `t_LightProxyCounters`
- `t_LightSamplingProxies`
- `t_LocalSamplingBuffer`
- `u_FeedbackTotalWeight`
- `u_FeedbackCandidates`
- `t_EmissiveTriangles`

Confirmed observations:

- Reference and BuildStablePlanes intentionally do not reflect every direct-light
  bridge resource because unused resources are optimized out.
- FillStablePlanes reflects and binds all direct-light bridge resources in the latest
  diagnostic logs:
  - `reflected=yes`
  - `object=yes`
  - `RGEN/RMISS/RCHIT/RAHIT=yes`
  - `set=yes`
- Therefore the latest red/green output is not explained by direct-light static
  resources being absent from the FillStablePlanes shader table.

## Dynamic Stable-Plane Resource Binding

Dynamic reflection logs were added for:

- `u_StableRadiance`
- `u_StablePlanesHeader`
- `u_StablePlanesBuffer`

Latest logs confirmed for both BuildStablePlanes and FillStablePlanes:

- `reflected=yes`
- `object=yes`
- `RGEN/RMISS/RCHIT/RAHIT=yes`
- `set=yes`
- dispatch dimensions: `width=1280`, `height=941`
- `stableHeader=yes`
- `stableBuffer=yes`

This rules out the simple explanation that the realtime stable-plane UAVs are missing
from dynamic binding.

## Build-to-Fill Visibility Findings

A cross-pass marker was added:

- Build `StablePlanes.StartPixel()` writes `u_StableRadiance[pixel].w = 0.5`.
- Fill `FirstHitFromVBuffer()` checks `u_StableRadiance[pixel].w` before reading
  `u_StablePlanesBuffer`.

Latest user result:

- Output still contained red/green.
- It did not contain yellow.

Interpretation:

- Build-to-Fill visibility is not globally broken.
- Build's `StartPixel()` write to `u_StableRadiance` is visible in Fill.
- The remaining issue is more specific than "Build pass did not run" or "all UAV writes
  are invisible to Fill".

## UAV Barrier Check

The current RTXPT barrier helper uses same-state UAV transitions:

```cpp
StateTransitionDesc{resource,
                    RESOURCE_STATE_UNORDERED_ACCESS,
                    RESOURCE_STATE_UNORDERED_ACCESS,
                    STATE_TRANSITION_FLAG_UPDATE_STATE}
```

DiligentCore D3D12 implementation was checked:

- Same-state `RESOURCE_STATE_UNORDERED_ACCESS -> RESOURCE_STATE_UNORDERED_ACCESS`
  transitions are explicitly treated as UAV barriers.
- They generate `D3D12_RESOURCE_BARRIER_TYPE_UAV`.

So the current evidence does not point to "same-state UAV barrier is ignored" as the
primary cause.

## Current Diagnostic Colors

The current diagnostic code is in:

- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`
- `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
- `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`

Current Fill color meanings:

- Yellow: Fill cannot see Build's `u_StableRadiance.w` StartPixel marker.
- Cyan: Fill sees StartPixel marker, but Build did not record a final plane0
  `StoreStablePlane()` write, and Fill still reads plane0 as empty.
- Red: Build recorded `StoreStablePlane(plane0, vertexIndex = 0)`.
- Orange: Build called `UpdatePathTravelled()`, but after
  `path.incrementVertexIndex()` the vertex index was still `0`.
- Pink: Build `StablePlanesHandleHit()` entered with plane0 `vertexIndex == 0`.
- Light blue: Build `StablePlanesHandleMiss()` entered with plane0 `vertexIndex == 0`.
- White: Build `HandleNestedDielectrics()` returned as an accepted hit, but plane0
  `vertexIndex` was already back to `0` before calling `StablePlanesHandleHit()`.
- Blue-green: Build was about to call `HandleNestedDielectrics()`, and plane0
  `vertexIndex` was already `0`.
- Purple: Build `HandleNestedDielectrics()` saw plane0 `vertexIndex == 0` after
  evaluating `isThinSurface()`.
- Pale yellow: Build `HandleNestedDielectrics()` saw plane0 `vertexIndex == 0` after
  evaluating `getNestedPriority()`.
- Dark blue: Build `HandleNestedDielectrics()` saw plane0 `vertexIndex == 0` after
  evaluating `isTrueIntersection()`.
- Hot pink/red: Build `HandleNestedDielectrics()` saw plane0 `vertexIndex == 0`
  after `ComputeOutsideIoR()`.
- Gray: Build `HandleNestedDielectrics()` saw plane0 `vertexIndex == 0` after
  `UpdateSurfaceOutsideIoR()` and immediately before returning an accepted hit.
- Amber/orange: Build caller observed `flagsAndVertexIndex == 0` immediately after
  accepted `HandleNestedDielectrics()` returned.
- Bright green-cyan: Build caller observed the vertex bits clear to `0` immediately
  after accepted `HandleNestedDielectrics()` returned, while the pre-call vertex was
  nonzero and the full flags word was not entirely zero.
- Marker `20.0` is reserved for the follow-up experiment where the caller observes that
  same vertex-bit clear, restores the low 10 vertex bits from the pre-call
  `flagsAndVertexIndex`, and then lets the pixel continue through normal Fill shading.
- Yellow from marker `21.0`: Build saw plane0 `vertexIndex == 0` immediately before
  calling `StablePlanesHandleHit()`.
- Blue from marker `22.0`: Build saw plane0 `vertexIndex != 0` immediately before
  calling `StablePlanesHandleHit()`, but Fill later still read plane0 as
  `vertexIndex == 0`. This points at the `StablePlanesHandleHit()` call boundary or
  code inside that function.
- Magenta: Build recorded `StoreStablePlane(plane0, vertexIndex >= 1)`, but Fill reads
  `StablePlanesBuffer` plane0 as `vertexIndex == 0`.
- Blue: Fill reads the explicit stable-plane buffer sentinel written by Build
  `StartPixel()`.
- Green: Fill reached the post-hit diagnostic branch where `pathStopping == false` but
  `path.isActive() == false` after `StablePlanesHandleHit()`.
- Yellow: Fill observed an accepted `HandleNestedDielectrics()` return where
  `path.isActive() == false`.
- Orange: Fill observed `path.isActive() == false` after surface-emission accumulation,
  before the Fill no-op `StablePlanesHandleHit()` call.

Latest reported color after adding the final vertex-index marker split:

- The image still contained lots of red/green.
- A small amount of magenta/purple appeared in sphere reflections.

Interpretation:

- Most affected pixels are Build `StoreStablePlane(plane0)` recording
  `vertexIndex == 0`.
- Some reflected pixels have a stronger structured-buffer mismatch signature:
  Build recorded a nonzero plane0 vertex marker in `u_StableRadiance.w`, but Fill read
  `StablePlanesBuffer` plane0 as `vertexIndex == 0`.

After that report, an additional diagnostic was added:

- Build `UpdatePathTravelled()` writes marker `8.0` if `path.incrementVertexIndex()`
  has just executed and `path.getVertexIndex()` is still `0`.
- `StoreStablePlane()` preserves this marker instead of overwriting it with
  `1.0 + vertexIndex`.
- Fill maps that marker to orange.

The next run did not report orange; it still contained lots of red/green plus a small
amount of magenta/purple in sphere reflections. This means `UpdatePathTravelled()` did
not immediately leave the vertex index at 0, but `StoreStablePlane()` later still
recorded 0 for many pixels.

After that, source-specific zero-vertex markers were added:

- Build `StablePlanesHandleHit()` writes marker `9.0` if plane0 enters with
  `vertexIndex == 0`.
- Build `StablePlanesHandleMiss()` writes marker `10.0` if plane0 enters with
  `vertexIndex == 0`.
- `StoreStablePlane()` preserves markers `>= 8.0`.
- Fill maps `9.0` to pink and `10.0` to light blue.

The next run should use orange/pink/light-blue/red/cyan/magenta/green to determine
whether plane0 zero comes from hit handling, miss handling, immediate increment failure,
or a later structured-buffer mismatch.

Latest report after adding hit/miss zero-vertex source markers:

- The output contained pink and green.

Interpretation:

- The dominant zero-vertex source is Build surface-hit handling, not Build miss handling.
- Because earlier orange was not reported, one remaining blind spot was marker
  overwriting: the hit marker could hide an earlier increment failure.
- The diagnostic marker write was changed to preserve the earliest marker `>= 8.0`.
- A new white marker was added after `HandleNestedDielectrics()` returns accepted but
  before `StablePlanesHandleHit()` is called.

The next run should distinguish:

- Orange: increment failure immediately after `UpdatePathTravelled()`.
- Blue-green: vertex is already zero before `HandleNestedDielectrics()`.
- White: vertex becomes zero by the time nested-dielectric handling finishes.
- Pink: vertex is nonzero through the added checks, but `StablePlanesHandleHit()` still
  sees zero at entry.

Latest report after adding the white marker:

- The previous pink regions became white.
- The original green regions remained green.

Interpretation:

- The zero vertex occurs before `StablePlanesHandleHit()`; the issue is in the code
  between `UpdatePathTravelled()` and `StablePlanesHandleHit()`.
- Because orange was not reported earlier, the immediate post-increment check did not
  catch the zero.
- A new blue-green marker was added immediately before calling
  `HandleNestedDielectrics()`.

The next run should distinguish:

- Blue-green: vertex became zero before nested handling is called, likely in the
  volume/interior-list block or an adjacent state update.
- White: vertex is nonzero before nested handling but becomes zero after
  `HandleNestedDielectrics()` returns accepted.

Latest report after adding the blue-green pre-nested marker:

- White did not become blue-green.

Interpretation:

- `UpdatePathTravelled()` and the pre-nested volume/interior-list block are not the
  source of the plane0 `vertexIndex == 0` for the white pixels.
- The current evidence points inside `HandleNestedDielectrics()` accepted handling, or
  to the function call / `inout` copy-back boundary immediately after it.

The next diagnostic split added markers inside `HandleNestedDielectrics()`:

- Purple: zero after `isThinSurface()`.
- Pale yellow: zero after `getNestedPriority()`.
- Dark blue: zero after `isTrueIntersection()`.
- Hot pink/red: zero after `ComputeOutsideIoR()`.
- Gray: zero after `UpdateSurfaceOutsideIoR()` just before accepted return.
- White still appearing without any of those new colors would point at the function
  return / `inout` copy-back boundary rather than code inside the accepted branch.

Latest report after adding the internal `HandleNestedDielectrics()` split:

- The affected pixels were still white.

Interpretation:

- The callee did not observe `path.getVertexIndex() == 0` at the sampled internal
  points.
- The caller observes `path.getVertexIndex() == 0` immediately after an accepted return.
- This strongly points at the `HandleNestedDielectrics()` call boundary / `inout`
  copy-back, not at `isThinSurface()`, `getNestedPriority()`, `isTrueIntersection()`,
  `ComputeOutsideIoR()`, or `UpdateSurfaceOutsideIoR()` themselves.

The next diagnostic split added caller-side markers around the return boundary:

- Amber/orange: the full `flagsAndVertexIndex` word is `0` after the accepted return.
- Bright green-cyan: only the vertex bits are observed as cleared after the accepted
  return, while the pre-call vertex was nonzero.
- White still appearing after this split means the caller-side zero does not match
  either of the above categories and needs a wider state snapshot.

Latest report after adding the caller-side return-boundary split:

- The affected pixels were bright green-cyan.

Interpretation:

- `flagsAndVertexIndex` is not entirely cleared.
- The low vertex-index bits are cleared across the accepted
  `HandleNestedDielectrics()` return boundary.
- Other flags in `flagsAndVertexIndex` remain set, so this is more precise than a full
  path termination or full `PathState` zeroing event.

Current follow-up experiment:

- If accepted `HandleNestedDielectrics()` returns with vertex bits cleared but the
  pre-call vertex was nonzero, restore only the low 10 vertex bits from the pre-call
  `flagsAndVertexIndex`.
- Store marker `20.0` for record keeping, but do not map it to a Fill diagnostic color
  when the restored stable plane has a nonzero vertex. This intentionally lets the
  realtime path continue into normal stable-plane shading.
- If the bright green-cyan/white diagnostic disappears or the black realtime image
  materially improves, this confirms the lost vertex bits are on the causal path.

Latest report after restoring the Build-side low 10 vertex bits:

- The cyan / bright green-cyan diagnostic disappeared.
- The remaining dominant diagnostic color is bright green.

Interpretation:

- Restoring the Build-side vertex bits is causally relevant and prevents Fill from
  reading plane0 as zero-vertex for those pixels.
- The remaining bright green is now a separate Fill-side path-active issue.
- In the current port, `StablePlanesHandleHit()` is effectively empty in
  FillStablePlanes, so a green diagnostic observed immediately after that call does not
  prove the call itself disabled the path. The inactive state may already exist after
  accepted `HandleNestedDielectrics()` or after emission accumulation.

Current Fill-side split:

- Yellow: accepted Fill `HandleNestedDielectrics()` returned with `path.isActive() ==
  false`.
- Orange: path became inactive after surface-emission accumulation, before
  `StablePlanesHandleHit()`.
- Green remains: path is still active at those two new checkpoints but becomes inactive
  or is observed inactive at/after the no-op stable-plane hit hook, requiring a wider
  snapshot around the call boundary.

Latest report after replacing early-return diagnostics with non-control-flow-changing
checkpoint recording:

- The remaining color is still bright green.

Interpretation:

- Fill path is still active after accepted nested-dielectric handling.
- Fill path is still active after surface-emission accumulation.
- The next suspicious boundary is the `StablePlanesHandleHit()` call. In the current
  code this function is effectively empty for FillStablePlanes because its body is
  guarded by `PATH_TRACER_MODE == PATH_TRACER_MODE_BUILD_STABLE_PLANES`, but the call
  still passes `PathState` as `inout`.

Current experiment:

- Compile the `StablePlanesHandleHit()` call only for BuildStablePlanes.
- FillStablePlanes skips this no-op `inout PathState` call.
- If bright green disappears, the Fill inactive state is caused by the no-op call
  boundary, consistent with the earlier Build-side evidence that `inout PathState`
  boundaries can corrupt low bits of `flagsAndVertexIndex`.

Latest report after skipping the Fill no-op `StablePlanesHandleHit()` call:

- Bright green did not disappear.

Interpretation:

- The no-op Fill `StablePlanesHandleHit()` call boundary is not the cause of the
  remaining inactive state.
- In FillStablePlanes, the code between the surface-emission checkpoint and the
  post-hit diagnostic has now narrowed to the `pathStopping` query / flag-reading
  region.

Current raw-flag split:

- Yellow/orange checkpoints now use raw `flagsAndVertexIndex` active-bit checks instead
  of `path.isActive()`.
- Fill computes `pathStopping` from raw
  `PathFlags::terminateAtNextBounce` bits instead of calling
  `path.isTerminatingAtNextBounce()`.
- Purple: raw active bit was present after surface emission but is lost by the
  path-stopping query region.
- Red: raw active bit is absent at the post-hit diagnostic fallback.
- Blue: raw active bit is still present, but `path.isActive()` reports inactive.

Latest report after the raw-flag split:

- The user reported green.

Important correction:

- This was ambiguous because Fill's zero-vertex `FirstHitFromVBuffer()` branch mapped
  marker `>= 19.0` to green-cyan, and marker `20.0` also satisfied that condition.
- Marker `20.0` means Build attempted to restore the low 10 vertex bits after the
  accepted nested-dielectric call boundary.
- The mapping now handles marker `20.0` first and displays it as yellow if Fill still
  reads `vertexIndex == 0`.
- The post-hit raw-active fallback color was changed from green to red, so future
  reports can distinguish these sources.

Latest report after this split:

- The user reported red.

Second ambiguity found:

- `FirstHitFromVBuffer()` also used pure red for the older case where Build recorded
  `StoreStablePlane(plane0, vertexIndex = 0)` via marker `1.0`.
- That conflicted with the newer post-hit raw-active fallback red.
- The old zero-vertex marker color is now changed to rose/magenta (`float4(4, 0, 2, 4)`).
- Pure red now uniquely means the post-hit raw-active fallback.

Latest report after splitting rose/magenta from pure red:

- The user reported pure red.

Interpretation:

- The pure-red fallback should have been covered by the previous orange or purple
  checks if all local bool diagnostics were trustworthy.
- This suggests the diagnostic bool chain itself may be unreliable under the current
  HLSL optimization / inlining pattern, or the active bit changes at a point not
  represented by the bools.

Current correction:

- Replace the Fill-side bool chain with raw `flagsAndVertexIndex` snapshots:
  after accepted nested handling, after surface-emission accumulation, and after the
  raw `pathStopping` calculation.
- Yellow/orange/purple now come directly from those snapshots.
- The pure-red fallback has been removed from this diagnostic path; if a red color is
  seen again, it should come from another source and must be re-identified by grep.

Latest report after raw snapshots:

- The user reported yellow.

Third ambiguity found:

- Yellow was still shared by three sources: missing Build StartPixel marker,
  marker `20.0` zero-vertex readback, and Fill accepted-nested raw active-bit loss.
- The missing-StartPixel case is now gray.
- Marker `20.0` zero-vertex readback is now cyan-blue.
- Yellow is now reserved for Fill accepted `HandleNestedDielectrics()` returning with
  the raw active bit already clear.

Latest report after yellow was made unique:

- The user reported yellow.

Interpretation:

- Fill accepted `HandleNestedDielectrics()` returns with the raw active bit cleared.
- This is the Fill-side analogue of the earlier Build-side accepted
  `HandleNestedDielectrics()` boundary issue, where low vertex bits were cleared.

Current follow-up experiment:

- Save Fill `flagsAndVertexIndex` before `HandleNestedDielectrics()`.
- If the accepted return clears the active bit while the pre-call active bit was set,
  restore only the active bit.
- This is guarded by `RTXPT_FILL_DIAGNOSTIC_FLOW_TRACE` and is a causality test, not a
  final cleanup.
- If yellow disappears, the active-bit loss is on the causal path for the remaining
  realtime failure.

Latest report after restoring the Fill active bit:

- Yellow still exists.

Interpretation:

- The active-bit restore did not remove the yellow classification.
- The next split distinguishes whether the raw active bit was already gone before
  `HandleNestedDielectrics()` was called, or whether it was present before the call but
  still absent after the restore attempt.

Current split:

- Gray: Fill raw active bit is already clear before `HandleNestedDielectrics()`.
- Yellow: Fill raw active bit was present before `HandleNestedDielectrics()`, the
  accepted return cleared it, and writing the active bit back to
  `path.flagsAndVertexIndex` still reads back as inactive.
- Cyan-green: Fill active bit restore reads back successfully and the path continues to
  the post-hit diagnostic point.
- White: unexpected fallback where the accepted-return snapshot is inactive even though
  it did not match the explicit restore-failed state.

Latest report after adding write-after-read diagnostics:

- The user reported white.

Correction:

- The cyan-green "restore succeeded" branch was ordered after the white fallback and
  could be masked by it.
- The diagnostic ordering now checks restore state `3` before the white fallback.
- Purple-blue now means restore state stayed `0`, but the accepted-return snapshot is
  inactive.
- White is now reserved for a narrower unknown fallback after states `1`, `2`, `3`,
  and `0+inactive` are handled.

Latest report after the priority fix:

- The user reported cyan-green.

Interpretation:

- The Fill active-bit restore does read back successfully.
- This confirms the accepted `HandleNestedDielectrics()` return boundary clears the
  active bit, and restoring that bit is possible.

Current follow-up experiment:

- Do not display cyan-green when the restore succeeds.
- Let the restored path continue into the normal Fill shading path.
- If the image improves or a later diagnostic color appears, the active-bit restore is
  part of the causal chain and the next remaining failure is downstream.

Latest report after letting active-bit restore continue:

- The image became all white.

Interpretation:

- The white was likely not a new failure state. The current `RTXPT_FILL_DIAGNOSTIC_FLOW_TRACE`
  path painted successful `HandleNEE()` output as `float4(4, 4, 4, 4)`.
- This means the path progressed past the previous active-bit failure and reached the
  later direct-light / BSDF path.

Current correction:

- Remove the flow-trace success-color writes after `GenerateScatterRay()` and
  `HandleNEE()` so normal radiance can show through.
- Change remaining white diagnostic colors to non-white colors:
  missing StartPixel marker is now blue-gray, the old post-nested fallback is
  purple, and NaN diagnostics are orange.
- If the image is still white after this, it is more likely actual radiance or another
  source outside these diagnostics.

Latest report after removing the success-path white override:

- The image became pink.

Ambiguity:

- Pink was shared by multiple enabled diagnostics:
  Fill `HandleHit()` entry found the path already inactive, Build stored plane0
  `vertexIndex == 0`, and Build `StablePlanesHandleHit()` saw zero vertex.

Current split:

- Cyan-green: Fill `HandleHit()` entry sees `path.isActive() == false`.
- Orange: Fill reads plane0 `vertexIndex == 0` with the old stored-vertex marker,
  meaning Build wrote `StoreStablePlane(plane0, vertexIndex = 0)`.
- Rose/magenta: Build `StablePlanesHandleHit()` entered with plane0
  `vertexIndex == 0`.
- Dark red: NaN/Inf validation failed in Fill.

Latest report after that split:

- The image is rose/magenta.

Interpretation:

- The zero is visible at the Build `StablePlanesHandleHit()` entry, not merely in Fill.
- If the post-`HandleNestedDielectrics()` restore path had written marker `20.0`, the
  Fill-side color would have been the marker-20 color instead of rose/magenta.
- Therefore the next split records the `PathState` immediately before the
  `StablePlanesHandleHit()` call:
  - Yellow: vertex was already zero before the call.
  - Blue: vertex was nonzero before the call and was lost at the call boundary or inside
    `StablePlanesHandleHit()`.

Important safety note from the first Fill-side split attempt:

- A version of this diagnostic used early `return` statements immediately after setting
  yellow/orange diagnostic radiance.
- That build triggered a D3D12 device removal in realtime testing.
- The likely cause is that these early returns bypassed the original terminal cleanup /
  stable-plane exploration control flow in `HandleHit()`.
- The current version records boolean checkpoints and waits until the original
  post-`StablePlanesHandleHit()` diagnostic point to set the color, without introducing
  those early returns.

## Current Narrowed Hypotheses

Still plausible:

- Build is actually calling `StoreStablePlane()` for plane0 with `vertexIndex == 0`.
- Fill is reading stale/zero data from `StablePlanesBuffer` even though
  `u_StableRadiance` visibility works.
- Some Build path writes the sentinel in `StartPixel()` but then does not execute final
  `StoreStablePlane()` for certain pixels.
- The current port's hit-stage structure differs from upstream: upstream
  `HandleHit()` increments `path.getVertexIndex()` before `Bridge::loadSurface()`,
  while the Diligent port constructs `SurfaceData` in closest-hit before calling
  `PathTracer::HandleHit()`. This difference may affect material/surface logic that
  depends on vertex index, though it does not by itself prove the plane0 red cause.

Currently less likely or already checked:

- Direct-light bridge variables missing in FillStablePlanes.
- Dynamic stable-plane UAVs missing or not set.
- Entire Build pass not running.
- Entire Build-to-Fill UAV visibility failing.
- Same-state UAV barrier being ignored by Diligent D3D12.
- Stable-plane address/stride mismatch; previous checks did not find a mismatch between
  the render-target allocation stride and shader addressing.

## Upstream RTXPT Checks Already Performed

- `deltaLobeCount = max(cMaxDeltaLobes - 1, deltaLobeCount)` exists in upstream
  RTXPT-fork too; do not "fix" this unless new evidence says it is wrong in this port.
- Upstream also uses mode-dependent payload/path-state layout patterns. Any `#if` inside
  shared structs is dangerous for cross-stage layout, but it is not automatically wrong;
  the real requirement is that every stage compiled into the same variant agrees on the
  exact layout and packing.

## Current Diagnostic/Experimental Code To Remember

Known temporary diagnostics currently present in the tree include:

- `RTXPT_FILL_DIAGNOSTIC_FLOW_TRACE`
- `RTXPT_STABLE_PLANE_SENTINEL_DIAGNOSTIC`
- direct-light static reflection logs
- dynamic stable-plane UAV reflection logs
- stable-plane sentinel write/read checks
- stable-radiance marker write/read checks
- plane0 stored-vertex marker in `u_StableRadiance.w`
- Build vertex-increment failure marker in `u_StableRadiance.w`
- Build hit/miss zero-vertex source markers in `u_StableRadiance.w`
- Build post-nested accepted-hit zero-vertex marker in `u_StableRadiance.w`
- Build pre-nested zero-vertex marker in `u_StableRadiance.w`

These should be removed or converted into proper guarded debug tooling after the root
cause is confirmed.

## Verification Already Run

Recent local checks passed:

```powershell
C:\VulkanSDK\1.4.350.0\Bin\dxc.exe -T lib_6_6 -E main -I DiligentSamples\Samples\RTXPT\assets\shaders -I DiligentSamples\Samples\RTXPT\assets\shaders\PathTracer -D "VK_IMAGE_FORMAT(x)=" -D PATH_TRACER_MODE=2 -D RTXPT_STABLE_PLANE_SENTINEL_DIAGNOSTIC=1 -D RTXPT_FILL_DIAGNOSTIC_FLOW_TRACE=1 DiligentSamples\Samples\RTXPT\assets\shaders\PathTracer\PathTracerSample.rgen -Fo NUL

C:\VulkanSDK\1.4.350.0\Bin\dxc.exe -T lib_6_6 -E main -I DiligentSamples\Samples\RTXPT\assets\shaders -I DiligentSamples\Samples\RTXPT\assets\shaders\PathTracer -D "VK_IMAGE_FORMAT(x)=" -D PATH_TRACER_MODE=1 -D RTXPT_STABLE_PLANE_SENTINEL_DIAGNOSTIC=1 DiligentSamples\Samples\RTXPT\assets\shaders\PathTracer\PathTracerSample.rgen -Fo NUL

cmake --build build\x64\Debug --config Debug --target RTXPT -- /m /v:minimal /nologo
```

`git diff --check` was also run for the touched shader diagnostics and reported no new
whitespace errors.

## Suggested Next Step

Run realtime `convergence-test.scene.json` again with the current diagnostic build and
record whether the output is yellow, blue, rose/magenta, or a mixture.

Most important for the current build:

- Yellow means Build already has plane0 `vertexIndex == 0` immediately before calling
  `StablePlanesHandleHit()`.
- Blue means Build still has a nonzero plane0 vertex immediately before the call, and
  the loss happens at the `StablePlanesHandleHit()` call boundary or inside that
  function.
- Rose/magenta after this new split would mean the pre-call marker did not survive to
  Fill; inspect the marker write/read path before interpreting the color as an older
  `StablePlanesHandleHit()` entry marker.
- White means the old `StablePlanesHandleHit()` entry marker is still the marker that
  Fill observes. This confirms the entry check is compiled, but the pre-call marker did
  not run or did not survive.
- If the image is still rose/magenta after marker 9 has been remapped to white, suspect
  a stale runtime shader/cache or another remaining rose/magenta source before drawing
  conclusions from the diagnostic table.

Latest report after remapping marker 9 to white:

- The image is still rose/magenta.

Follow-up isolation:

- The copied runtime shader under
  `build/x64/Debug/DiligentSamples/Samples/RTXPT/Debug/shaders/PathTracer/PathTracerSample.rgen`
  matches the source shader hash, so the normal CMake asset copy path is not stale.
- All known PathTracer diagnostic outputs that used rose/magenta/purple-like colors have
  now been remapped away from that color family.
- A source scan for remaining PathTracer `float3/float4` pink/magenta diagnostic colors
  returns no matches.
- If the next run still shows rose/magenta, interpret it as either a shader/runtime cache
  outside the copied text assets, a different non-PathTracer shader/output path, or actual
  scene/postprocess color rather than one of the current PathTracer debug colors.

Latest report after removing rose/magenta diagnostic colors:

- The image is green.

Follow-up isolation:

- Green still had multiple possible sources:
  - Build marker/plane-read diagnostics in `FirstHitFromVBuffer()`.
  - Fill `HandleHit()` active/pathStopping flow diagnostics.
  - A few disabled-by-default diagnostic branches that still used green/cyan.
- The current shader remaps those green/cyan diagnostic outputs away from the green
  family.
- A source scan now finds no enabled PathTracer `SetL()` green diagnostic colors.
- If the next run is still green, treat it as likely actual scene/postprocess output or a
  non-PathTracer path rather than one of the current green debug colors.

Most important for the current build:

- Blue: likely Fill accepted-return active-bit loss, or Build pre-`StablePlanesHandleHit`
  nonzero marker.
- Red: likely one of the Build marker/plane-read paths that used to look green/cyan.
- White: likely entry/missing-marker/build-hit-style diagnostic that used to look green
  or white.
- Orange/yellow: likely pathStopping/pre-call/older marker classes.

Latest report after removing green/cyan diagnostic colors:

- The image is blue.

Follow-up isolation:

- Blue still had multiple possible sources:
  - `FirstHitFromVBuffer()` read the explicit stable-plane sentinel.
  - Build wrote marker `22.0`: vertex was nonzero immediately before
    `StablePlanesHandleHit()`, but Fill later read plane0 `vertexIndex == 0`.
  - Build wrote marker `15.0`: zero vertex observed after `isTrueIntersection()`.
  - Fill `HandleHit()` saw inactive state immediately after `UpdatePathTravelled()`.
  - Fill accepted `HandleNestedDielectrics()` returned with active bit lost.
  - Fill post-check fallback saw `path.isActive() == false`.
- The current shader remaps those blue sources away from blue:
  - Red: Build pre-`StablePlanesHandleHit()` nonzero marker or Fill post-check
    inactive fallback.
  - Orange: Build nested true-intersection marker or Fill inactive immediately after
    `UpdatePathTravelled()`.
  - Yellow: Fill accepted-return active-bit loss.
  - White: explicit stable-plane sentinel / missing-marker style source.
- If the next run is still blue, treat it as likely non-current PathTracer debug output,
  postprocess/scene color, or a stale runtime shader cache outside the copied text assets.

Latest report after removing blue diagnostic colors:

- The image is yellow.

Follow-up isolation:

- Yellow was still shared by Build/plane-read diagnostics, Fill accepted-return
  active-bit diagnostics, and a payload round-trip diagnostic.
- The current shader remaps these yellow sources by category:
  - Red: Build/plane-read yellow sources, including marker `21.0`
    pre-`StablePlanesHandleHit()` zero vertex and marker `11.0` after-nested zero vertex.
  - White: Fill accepted-return active-bit loss sources.
  - Orange: payload round-trip / disabled terminal-gate yellow sources.
- If the next run is still yellow, treat it as non-current PathTracer debug output,
  postprocess/scene color, or stale runtime shader cache outside the copied text assets.

Latest report after removing yellow diagnostic colors:

- The image is white.

Follow-up isolation:

- White was still shared by Build/plane-read diagnostics, Fill accepted-return
  active-bit diagnostics, and several older missing-marker/minimal diagnostics.
- The current shader remaps pure white diagnostic colors away from white.
- The Fill accepted-return active-bit states are now split explicitly:
  - Gray: Fill entered accepted nested handling with the raw active bit already absent
    before the call.
  - Red: active-bit restore was attempted but did not read back.
  - Cyan: active bit was absent after accepted `HandleNestedDielectrics()`, then the
    diagnostic restore wrote it back successfully (`fillAcceptedActiveRestoreState == 3`).
  - Orange: unexpected fallback where accepted-return active bit is still absent after
    the classification path.
- If the next run is cyan, the current strongest evidence is that the accepted
  `HandleNestedDielectrics()` call boundary clears the Fill path active bit, and the
  local restore masks that state loss.
- If the next run is still white, treat it as non-current debug output, actual
  scene/postprocess output, or another non-PathTracer path.

Interpret the next result as follows:

- Red dominates: inspect why Build has `path.getVertexIndex() == 0` at
  `StoreStablePlane(plane0)`.
- Orange appears: inspect `PathState::incrementVertexIndex()` or the packed
  `flagsAndVertexIndex` state immediately before/after `UpdatePathTravelled()`.
- Blue-green appears: inspect the code between `UpdatePathTravelled()` and
  `HandleNestedDielectrics()`, especially the volume/interior-list block.
- White appears: inspect `HandleNestedDielectrics()` and the code between
  `UpdatePathTravelled()` and `StablePlanesHandleHit()`.
- Purple/pale-yellow/dark-blue/hot-pink/gray appears: inspect the corresponding internal
  step of `HandleNestedDielectrics()` listed above.
- White remains without those internal colors: suspect the `HandleNestedDielectrics()`
  call boundary or HLSL `inout` copy-back behavior, because the caller observes
  `vertexIndex == 0` after return while the callee did not observe it internally.
- Amber/orange appears: inspect why the accepted call boundary clears
  `flagsAndVertexIndex` entirely; this also explains green post-hit diagnostics because
  the active/stable-plane flags live in the same word.
- Bright green-cyan appears: inspect why only the low vertex bits are cleared across the
  accepted call boundary.
- After the marker-20 restore experiment, a more normal image means the immediate
  mitigation is valid and the remaining root-cause work should focus on why the
  `inout` call boundary drops the vertex bits.
- Yellow appears after the Fill split: investigate the accepted Fill
  `HandleNestedDielectrics()` call boundary for active-flag loss.
- Orange appears after the Fill split: inspect `AccumulatePathRadiance()` or related
  Fill emission accumulation side effects.
- Bright green disappears after skipping the Fill no-op `StablePlanesHandleHit()` call:
  keep the call Build-only or refactor the hook so Fill does not pass `PathState`
  through an empty `inout` function.
- Purple appears after the raw-flag split: inspect the `pathStopping` query region and
  related flag access.
- Blue appears after the raw-flag split: inspect `PathState::isActive()` /
  `PathState::hasFlag()` behavior versus direct bit reads.
- Yellow appears from the marker-20 split: Build attempted the vertex-bit restore, but
  Fill still read plane0 as `vertexIndex == 0`; inspect whether the restored
  `flagsAndVertexIndex` is the value passed to `StoreStablePlane()`.
- Gray appears: Fill did not see the Build StartPixel marker in `u_StableRadiance.w`.
- Cyan-blue appears: Build attempted the vertex-bit restore, but Fill still read plane0
  as `vertexIndex == 0`; inspect whether the restored `flagsAndVertexIndex` is the
  value passed to `StoreStablePlane()`.
- Yellow appears after this split: Fill accepted `HandleNestedDielectrics()` returned
  with the raw active bit already clear.
- After the active-bit restore experiment, a new color or more normal shading means
  the Fill accepted-return active-bit loss is causal.
- Rose/magenta appears after this split: Build still stored plane0 with
  `vertexIndex == 0`.
- Pure red appears after this split: inspect why raw active bit is absent at the post-hit
  fallback despite earlier raw-active checkpoints.
- Pink appears: inspect why Build surface-hit stable-plane handling enters with
  plane0 `vertexIndex == 0` after the post-nested diagnostic check.
- Light blue appears: inspect miss/false-hit/nested-dielectric paths that can decrement
  the vertex index back to 0 before `StablePlanesHandleMiss()`.
- Cyan appears: inspect Build paths where `StartPixel()` runs but no final plane0
  `StoreStablePlane()` happens.
- Magenta appears: investigate structured-buffer write/read visibility or layout,
  because the texture marker says Build stored a nonzero vertex but Fill reads zero.
- Green remains: continue tracing Fill `StablePlanesHandleHit()` and the conditions that
  terminate the path while `pathStopping == false`.

## 2026-06-06: Cyan-Green Accepted-Return Result

Latest user report:

- The image is cyan-green after the Fill accepted-active restore split.

Interpretation:

- The raw active bit was present before Fill called `HandleNestedDielectrics()`.
- The hit was accepted (`rejectedFalseHit == false`).
- Immediately after the accepted return, the raw active bit in
  `PathState::flagsAndVertexIndex` was absent.
- The local restore wrote the active bit back successfully.

Decision:

- Treat this as causal evidence that the accepted `HandleNestedDielectrics()` call
  boundary can lose packed `flagsAndVertexIndex` state in stable-plane modes.
- Apply a non-diagnostic guard for accepted returns: snapshot `flagsAndVertexIndex`
  before the call and restore it only when the hit is accepted.
- Do not restore this word for rejected false hits, because that path intentionally
  updates rejected-hit counters, interior-list state, origin and vertex index.
- Disable the temporary Fill/Build color diagnostics by default so the next test shows
  the real realtime image instead of the diagnostic color overlay.

## 2026-06-06: Blue After Accepted-Return Guard

Latest user report:

- The image became blue after disabling the Fill/Build path-tracer color diagnostics.

Interpretation:

- This blue no longer maps to the previous path-tracer flow-trace colors.
- The remaining default blue source was the post-process raw stable-plane final-merge
  diagnostic: when `DebugColor` was nearly zero but `DebugValidPlanes != 0`, it wrote
  `float3(0.0, 0.0, 4.0)`.
- `RTXPTPostProcessPass.cpp` still enabled
  `RTXPT_POST_PROCESS_DIAGNOSTIC_STABLE_PLANE_RAW` for final merge passes by default.

Decision:

- Remove the default C++ macro injection for
  `RTXPT_POST_PROCESS_DIAGNOSTIC_STABLE_PLANE_RAW`.
- Keep the shader-side macro and branch as an opt-in diagnostic, defaulting to `0`.

## 2026-06-06: Real Black After Removing Color Overlays

Latest user report:

- With Fill/Build path-tracer diagnostics and post-process raw diagnostics disabled,
  realtime shows sky, reflections and emissive surfaces, but most ordinary surfaces are
  black.

Interpretation:

- This is no longer a diagnostic overlay color.
- Miss/environment, specular/reflection and emissive accumulation paths are alive.
- The next likely boundary is Fill direct-light/NEE accumulation versus committing
  `path.GetL()` into stable-plane noisy radiance and final merge.

Current diagnostic:

- `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE` is enabled for FillStablePlanes.
- Red: NEE is disabled or `NEEFullSamples == 0`.
- Yellow: the vertex is beyond `maxNEEBounceCount`.
- Blue: NEE was evaluated for the surface but returned zero radiance.
- Green: NEE returned nonzero radiance.
- White: the Fill ray reached final commit with no radiance/marker in `path.GetL()`.
- If the image remains black, inspect `CommitDenoiserRadiance()` / final merge reads,
  because the diagnostic marker was not visible after commit.

Latest report:

- The image is white.

Interpretation:

- Final commit is visible, but `AccumulateNEERadiance()` did not run for the affected
  ordinary surfaces.

Follow-up stage split:

- Pale blue: Fill initialized from the V-buffer stable plane, but no later hit/miss
  stage marked the path.
- Cyan: Fill entered `HandleMiss()`.
- Purple: Fill entered `HandleHit()`, but did not reach NEE accumulation.
- Orange: Fill returned through rejected nested-dielectric false-hit handling.
- Orange-red: Fill terminated/inactivated before the terminal NEE branch could run.
- Magenta: Fill reached the normal non-terminal scatter region before NEE accumulation.
- White: no stage marker was written before final commit.

Latest report:

- The image is yellow.

Interpretation:

- `AccumulateNEERadiance()` was reached, but the diagnostic branch saw
  `preScatterPath.getVertexIndex() > min(maxNEEBounceCount, bounceCount)`.
- In this port, `HandleNEE()` had an extra vertex/max-bounce gate:
  `preScatterPath.getVertexIndex() <= maxNEEBounces`.
- Upstream RTXPT `PathTracerNEE.hlsli::HandleNEE()` does not gate NEE by path vertex
  index; it gates on valid non-delta BSDF/light/full-sample conditions instead.

Fix decision:

- Remove the extra vertex/max-bounce gate from Fill `HandleNEE()`.
- Disable the temporary `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE` macro injection so
  the next run shows real realtime lighting again.

## 2026-06-06: Still Black After Removing NEE Vertex Gate

Latest user report:

- After the Fill `HandleNEE()` vertex/max-bounce gate was removed, ordinary
  realtime surfaces in `convergence-test.scene.json` are still fully black.
- Sky, reflected environment/emissive content and emissive surfaces remain visible.

Interpretation:

- The removed vertex gate was a real port mismatch versus upstream RTXPT, but it was
  not the only cause of the realtime black ordinary surfaces.
- The current evidence still points at Fill direct lighting/NEE rather than the
  miss, specular/reflection or emissive paths.
- The next boundary to isolate is inside `HandleNEE()`:
  direct-light source availability, candidate generation, BSDF/pdf evaluation,
  visibility, pre-throughput radiance, throughput multiplication, and the
  `AccumulateNEERadiance()`/stable-plane noisy-radiance commit boundary.

Current diagnostic:

- `RTXPT_FILL_DIAGNOSTIC_NEE_INTERNAL_TRACE` is now injected only for the
  `FillStablePlanes` variant from `RTXPTRayTracingPass.cpp`.
- The diagnostic overrides the Fill NEE result with a strong color so the user can
  report where the NEE chain stops.
- Red: `NEEEnabled == 0` or `NEEFullSamples == 0`.
- Yellow: there is no direct-light source and env NEE is disabled.
- Orange: direct/env NEE were entered but neither produced an effective sample source
  (`DirectLightSample` candidate missing and env sample/pdf path not reached).
- Blue: a direct candidate or env BSDF/pdf existed, but all visibility checks failed.
- Magenta: raw NEE radiance existed before throughput, but
  `preScatterPath.GetThp()` was effectively zero.
- Green: NEE produced a nonzero post-throughput result; if the scene still looks black
  after disabling diagnostics, inspect accumulation/final merge rather than sampling.
- Purple: NEE reached later stages but contribution remained zero for a reason not yet
  separated by this probe, likely BSDF/radiance/fadeout/firefly-filter contribution.
- White: fallback/unknown diagnostic state.

Next action:

- Ask the user to run realtime `convergence-test.scene.json` once and report the
  dominant color on the ordinary black surfaces.

Latest report:

- The ordinary surfaces are still black after enabling
  `RTXPT_FILL_DIAGNOSTIC_NEE_INTERNAL_TRACE`.

Interpretation:

- The NEE-internal color did not become visible. This means either Fill did not reach
  `HandleNEE()` / `AccumulateNEERadiance()` for those pixels, or the noisy-radiance
  commit/final-merge path is not showing `path.GetL()` for the affected stable planes.

Follow-up diagnostic:

- Also inject `RTXPT_FILL_DIAGNOSTIC_INJECT_COMMIT_RADIANCE` for `FillStablePlanes`.
- This bypasses `HandleHit()` and NEE by adding strong red radiance to `path.GetL()`
  immediately before `CommitPixel()`.
- If the next run is still black on ordinary surfaces, investigate Fill
  `CommitDenoiserRadiance()` / `StablePlanesUAV.PackedNoisyRadianceAndSpecAvg` /
  final merge reads first.
- If the next run becomes red, the Fill noisy-radiance display path is alive and the
  root cause is earlier: `FirstHitFromVBuffer()` / `HandleHit()` / NEE accumulation
  does not populate `path.GetL()` for the ordinary surfaces.

Latest report:

- The image became red with `RTXPT_FILL_DIAGNOSTIC_INJECT_COMMIT_RADIANCE`.

Interpretation:

- Fill dispatch, `path.GetL()`, `CommitPixel()`, `CommitDenoiserRadiance()`,
  `StablePlanesUAV.PackedNoisyRadianceAndSpecAvg`, and the final merge path are all
  capable of displaying Fill noisy radiance.
- The ordinary-surface black result is therefore earlier than the final commit:
  normal Fill execution is not producing radiance in `path.GetL()` before commit.

Follow-up diagnostic:

- Disable the forced commit-red injection.
- Enable only `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE` for `FillStablePlanes`.
- This separates Fill control-flow stages without overriding the real NEE result.
- Pale blue: `FirstHitFromVBuffer()` ran and no later hit/miss/NEE marker wrote
  radiance.
- Cyan: Fill entered `HandleMiss()`.
- Purple: Fill entered `HandleHit()` but did not reach the later NEE/scatter marker.
- Orange: rejected nested-dielectric false-hit or terminal/inactive path before NEE.
- Magenta: Fill reached the normal non-terminal scatter region before NEE
  accumulation.
- Red: NEE disabled or no full samples.
- Blue: `AccumulateNEERadiance()` ran but the real NEE result was zero.
- Green: `AccumulateNEERadiance()` ran and the real NEE result was nonzero.

## 2026-06-06: Shared Layout Parity Audit

This pass audited GPU-facing shared layouts against `D:/RTXPT-fork` and the local
Diligent CPU mirrors.

Findings:

- `PathTracerCameraData`, `PathState`, `PathPayload`, `StablePlane`,
  `LightsBakerConstants`, and `LightingControlData` keep the upstream wire order.
- `PathPayload` is a Diligent port translation of upstream `uint4 packed[5]`: local
  code uses named `packed0..packed4` fields, but the cross-stage wire payload is still
  five `uint4` lanes. `StablePlane::PackCustomPayload()` now uses the same shared
  payload-count macro.
- `PathTracerConstants`, `PathTracerViewData`, `SampleConstants`,
  `RTXPTEnvMapConstants`, `MaterialPTData`, `SubInstanceData`,
  `PolymorphicLightInfo`, `EmissiveTriangle`, and `GeometryVertexData` are
  Diligent-port backing layouts rather than upstream CPU structs. They now have
  additional `sizeof`/`offsetof` guards on the CPU side for cbuffer row boundaries,
  `float3`/`uint3` follower fields, and StructuredBuffer stride-critical fields.
- `LightingControlData` is intentionally translated on CPU as a 112-byte control
  header followed by a 464-byte `LightsBakerConstants` payload/padding block. The
  env-map baker sub-layout inside that payload is guarded by shared word offsets.
- No `bool` or enum was found in CPU-shared wire records. HLSL-only helper structs
  such as `NEEBSDFMISInfo` still use `bool`, but they are manually packed before
  crossing payload/storage boundaries.

Remaining caution:

- `SkinVertexData` is consumed from the GLTF skin stream (`JOINTS_0`/`WEIGHTS_0`) and
  has no dedicated CPU mirror struct in this port; the current audit only guards the
  skinned output `GeometryVertexData` stride/offsets.

## 2026-06-06: Yellow Outcome Trace After Shared Layout Audit

Latest report:

- The realtime `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE` result is yellow.

Interpretation:

- In the current shader mapping, yellow comes from outcome marker `4.0`, written when
  Fill `HandleHit()` returns through the `rejectedFalseHit` branch after
  `HandleNestedDielectrics()`.
- Comparing against `D:/RTXPT-fork` found a behavior mismatch in the ported full
  `SurfaceData` nested-dielectric path: for `RTXPT_NESTED_DIELECTRICS_QUALITY == 1`,
  upstream stops rejecting false hits after `kMaxRejectedDielectricHits` and accepts the
  approximate intersection. The local code still returned `false` after the rejection
  counter reached the limit, without advancing origin or terminating.

Fix applied:

- Move the `return false` into the branch that actually increments `RejectedHits`, and
  keep the `RTXPT_NESTED_DIELECTRICS_QUALITY == 2` termination branch returning false.
  Quality 1 now falls through to the accepted-hit path once the reject limit is reached,
  matching upstream behavior.

Latest report:

- The output is still yellow.

Interpretation:

- The previous fix did not eliminate the false-hit path before final commit.
- Yellow still only identifies the broad `rejectedFalseHit` boundary, so it is no longer
  specific enough to locate the bad input.

Follow-up diagnostic:

- The rejected nested-dielectric branch now writes unique RGB colors before returning:
  - Red: `isTrueIntersection()` was false even though `interiorList` was empty.
  - Magenta: the false-hit condition was internally inconsistent
    (`nestedPriority == 0` or `topNestedPriority <= nestedPriority`).
  - Green: first false-hit reject with a non-empty interior list and
    `topNestedPriority > nestedPriority`.
  - Blue: repeated false-hit reject with a non-empty interior list and
    `topNestedPriority > nestedPriority`.
  - Cyan: quality-2 reject-limit termination path.

Latest report:

- The floor is still yellow; the spheres became red.

Interpretation:

- Red narrows one class of affected pixels to an impossible state:
  `interiorList.isEmpty()` is true after `isTrueIntersection()` returned false.
  With an empty stack the top nested priority is 0, so any `uint nestedPriority`
  should make `nestedPriority >= topNestedPriority` true.
- Yellow on the floor still represents an uncolored marker-4 false-hit path, so the
  rejected-hit boundary remains involved there as well.

Follow-up fix/probe:

- Make `InteriorList::isTrueIntersection()` explicitly return true for an empty list
  before comparing priorities. This is semantically equivalent to upstream's packed
  priority comparison for an empty stack, but removes the impossible empty-list false
  classification and tests whether the HLSL expression/optimization is on the causal
  path.

Latest report after the empty-list guard:

- The floor is still yellow; the spheres are still red.

Interpretation:

- Red is now ambiguous because other active diagnostics can also write red, and the
  nested-reject colors were written inside `HandleNestedDielectrics()` through an
  `inout PathState`.
- This area has already shown HLSL inout copyback hazards, so colors written in the
  nested helper may not survive back to `HandleHit()`.

Follow-up diagnostic:

- `HandleHit()` now snapshots the nested-dielectric inputs before calling
  `HandleNestedDielectrics()` and writes the reject classification in the caller after
  a `rejectedFalseHit` return:
  - Teal/green-cyan: rejected while the pre-call interior list was empty.
  - Magenta: rejected even though the pre-call priority comparison looked acceptable.
  - Green: first reject with a non-empty list and `topNestedPriority > nestedPriority`.
  - Blue: repeated reject with a non-empty list and `topNestedPriority > nestedPriority`.
- Plain red now points away from the nested-reject classifier, most likely to the
  active NEE-disabled/full-samples diagnostic in `AccumulateNEERadiance()`.

Latest report:

- The spheres are still pure red.

Interpretation:

- With only `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE` injected for
  `FillStablePlanes`, pure red is still ambiguous between:
  - the helper-local nested empty-list false-hit diagnostic written inside
    `HandleNestedDielectrics()`;
  - the `AccumulateNEERadiance()` NEE gate diagnostic for
    `NEEEnabled == false || NEEFullSamples == 0`.
- The caller-side nested classifier should overwrite the helper-local color if the
  `rejectedFalseHit` return reaches `HandleHit()`, so persistent pure red leans
  toward the NEE gate, but the colors need to be made unique before treating that as
  root cause.

Follow-up diagnostic:

- Change the helper-local nested empty-list diagnostic from pure red to light
  green/cyan (`float4(1, 4, 2, 4)`).
- Split the `AccumulateNEERadiance()` NEE gate into two unique colors:
  - Orange (`float4(4, 1, 0, 4)`): `NEEEnabled == false`.
  - Pink (`float4(4, 0, 2, 4)`): `NEEFullSamples == 0`.
- If the spheres remain pure red after this build, the active red source is outside
  these two outcome-trace writers and the next step is to locate an unexpectedly
  enabled diagnostic macro or another commit-time red write.

Latest report:

- The spheres changed to light green/cyan.

Interpretation:

- This confirms the realtime sphere pixels are reaching the nested-dielectric
  empty-list false-hit path, not the NEE-disabled/full-samples gate.
- Upstream `RTXPT-fork` also does not restore `StablePlane::FlagsAndVertexIndex` or
  `StablePlane::PackedCounters` in `FirstHitFromVBuffer()`, and `StoreStablePlane()`
  intentionally passes `0, 0` for those values. So missing stable-plane
  flags/counters restore is not the root difference.
- For an empty `InteriorList`, upstream `isTrueIntersection()` is semantically always
  true because `getTopNestedPriority()` is 0 and `nestedPriority == 0 ||
  nestedPriority >= 0` must hold for `uint`.
- The local shader already added an explicit empty-list guard inside
  `InteriorList::isTrueIntersection()`, but Fill still rejects. That points to the
  full `inout PathState` nested helper seeing or copying a state different from the
  caller, consistent with earlier `flagsAndVertexIndex` copyback issues in this area.

Follow-up fix/probe:

- In `FillStablePlanes` only, if the caller-side `path.interiorList.isEmpty()` is true
  before calling `HandleNestedDielectrics()`, accept the hit immediately and update
  the surface outside IoR to 1.0 for non-thin surfaces.
- This is intended to be semantically equivalent to upstream's empty-stack true
  intersection result, but avoids routing the empty-stack Fill re-hit through the
  problematic full-`PathState` `inout` helper.
- Expected next diagnostic result: the spheres should no longer be light green/cyan.
  If they become green, NEE is producing non-zero radiance; if blue, the path reaches
  NEE but the NEE result is zero; if orange or pink, the relevant NEE CPU constants
  are disabled/zero.

Latest report:

- The spheres changed to orange.

Interpretation:

- This confirms the previous empty-list nested false-hit was bypassed and the path now
  reaches a later outcome.
- Orange is still ambiguous in the current diagnostic build:
  - `AccumulateNEERadiance()` writes orange when the shader sees
    `PtConsts.NEEEnabled == 0`.
  - The final raygen outcome mapper also used orange for marker 5 fallback, meaning
    the path terminated without RGB radiance before final commit.

Follow-up diagnostic:

- Keep `AccumulateNEERadiance()` `NEEEnabled == 0` as orange.
- Change the final marker 5 fallback color to muted pink/gray
  (`float4(2, 1, 1, 4)`) so it no longer collides with NEE-disabled orange.
- Add a one-time CPU-side frame-constant log in `RTXPTSample::UpdateFrameConstants()`
  showing `m_EnableNEE`, uploaded `NEEEnabled`, `NEEFullSamples`,
  `NEECandidateSamples`, `maxNEEBounceCount`, packed environment NEE state, and
  analytic light count.
- If the spheres remain orange, shader-side `PtConsts.NEEEnabled` is 0 and the CPU
  log will distinguish UI/runtime state from constant-buffer/layout readback issues.
  If they become muted pink/gray, the orange came from the marker 5 fallback path.

Latest report:

- The spheres changed to muted pink/gray.

Interpretation:

- The previous orange was the final marker 5 fallback, not the
  `AccumulateNEERadiance()` `NEEEnabled == 0` path.
- Marker 5 is written when `HandleHit()` reaches the `pathStopping ||
  !path.isActive()` branch without already having RGB radiance, but not through the
  `pathStopping && path.isActive()` terminal-NEE path.
- Therefore the next target is the active/termination state transition before this
  branch.

Follow-up diagnostic:

- Under `RTXPT_FILL_DIAGNOSTIC_NEE_OUTCOME_TRACE`, snapshot
  `flagsAndVertexIndex` at these points:
  - entry to `HandleHit()`;
  - after `UpdatePathTravelled()`;
  - after accepted nested-dielectric handling and flag restore;
  - after surface-emission accumulation.
- At the marker 5 branch, write unique RGB colors before the fallback marker:
  - Red: path was already inactive on entry to `HandleHit()`.
  - Yellow: active was lost during `UpdatePathTravelled()`.
  - Magenta: active was lost during accepted nested-dielectric handling/restore.
  - Cyan: active was lost during surface-emission accumulation.
  - Blue: `pathStopping` and inactive are both set at the branch.
- If none of these colors appear and muted pink/gray remains, the branch is being
  reached through another state not captured by the snapshots.

Latest report:

- The spheres changed to magenta.

Interpretation:

- The previous diagnostic grouped all nested-stage state together. Magenta means the
  `HandleHit()` marker-5 branch saw active as lost after accepted nested handling and
  flag restore, but it does not yet distinguish:
  - active was already lost between travel and the pre-nested snapshot;
  - active was lost by the raw nested helper/empty-stack shortcut before restore;
  - active was lost specifically by restoring `preNestedFlagsAndVertexIndex`.

Follow-up diagnostic:

- Split the nested-stage snapshot into:
  - `fillDiagFlagsBeforeNested`;
  - `fillDiagFlagsRawPostNested`;
  - `fillDiagFlagsAfterRestore`.
- Refined marker-5 colors:
  - White: active was lost before the pre-nested snapshot.
  - Purple: raw post-nested state is inactive.
  - Magenta: raw post-nested state is active, but restore result is inactive.
  - Cyan still means active was lost after surface-emission accumulation.

Latest report:

- The reported color is exact `FF00FF` magenta.

Interpretation:

- The current diagnostic build still had several pure magenta writers, so exact
  `FF00FF` is not uniquely attributable to the refined restore classifier:
  - helper-local nested false-hit inconsistency;
  - caller-side nested rejected-hit acceptable-priority classifier;
  - restore-after-nested active-loss classifier;
  - final raygen marker-6 fallback.
- Source scan confirmed these were all still present as `float4(4, 0, 4, 4)` or
  equivalent `float3(4, 0, 4)`.

Follow-up diagnostic:

- Remove all remaining pure `FF00FF` diagnostic writes from the PathTracer shader
  path and remap them to unique colors:
  - Helper-local nested inconsistency: rose (`float4(4, 0, 1, 4)`).
  - Caller-side nested acceptable-priority reject: violet (`float4(1, 0, 4, 4)`).
  - Restore-after-nested active-loss: lime (`float4(1, 4, 0, 4)`).
  - Raygen marker-6 fallback: lavender (`float4(2, 1, 4, 4)`).
  - Internal NEE diagnostic magenta was also changed to purple-like
    `float3(2, 0, 4)`.
- A follow-up `rg` scan found no remaining exact `float3/float4(4, 0, 4, ...)`
  diagnostic color in `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer`.
- If the next run still shows exact `FF00FF`, suspect stale shader/runtime output or
  an output path outside the searched PathTracer shader files.

## 2026-06-07: Upstream Parity Candidates After `cf829f`

Context for this pass:

- Known-good DiligentSamples commit: `cf829f3294c517c7046a0e0b200c66f9f5d6c57c`.
- Current DiligentSamples HEAD during the audit: `913e36e6`.
- The regression range after `cf829f` contains only a few RTXPT shader/state commits
  relevant to realtime diffuse:
  - `a5777c5f`: restore BxDF delta lobe export;
  - `529f312d`: align shader payload and layout contracts;
  - `970a7378`: align realtime BxDF and material state;
  - `1ff5a7b8`: remove realtime diagnostic shader paths.
- Upstream comparison source: `D:/RTXPT-fork`.
- This section records candidate differences only. It does not claim the final root
  cause until a minimal shader probe or revert confirms the causal path.

Files compared in this pass:

- Local:
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathState.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathPayload.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracer.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerSample.rgen`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerStablePlanes.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/StablePlanes.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/PathTracerClosestHit.rchit`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/Rendering/Materials/BxDF.hlsli`
  - `DiligentSamples/Samples/RTXPT/assets/shaders/PostProcessing/RTXPTPostProcess.csh`
  - `DiligentSamples/Samples/RTXPT/src/RTXPTRayTracingPass.cpp`
- Upstream:
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathState.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathPayload.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracer.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracerSample.hlsl`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerStablePlanes.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/StablePlanes.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/PathTracerNEE.hlsli`
  - `D:/RTXPT-fork/Rtxpt/Shaders/PathTracer/Rendering/Materials/StandardBSDF.hlsli`
  - `D:/RTXPT-fork/Rtxpt/ProcessingPasses/PostProcess.hlsl`
  - `D:/RTXPT-fork/Rtxpt/NRD/DenoiserNRD.hlsli`

Strong candidate: masked vertex-index accessors introduced by `529f312d`.

- `cf829f` and upstream both used the unsafe RTXPT-style vertex arithmetic:
  - `setVertexIndex()` clears the low bits and ORs the raw index.
  - `incrementVertexIndex()` does `flagsAndVertexIndex += 1`.
  - `decrementVertexIndex()` does `flagsAndVertexIndex -= 1`.
- Current HEAD uses masked helpers:
  - `setVertexIndex(index)` writes `(index & kVertexIndexBitMask)`.
  - `incrementVertexIndex()` calls `setVertexIndex(getVertexIndex() + 1u)`.
  - `decrementVertexIndex()` calls `setVertexIndex(getVertexIndex() - 1u)`.
- This was introduced in `529f312d`, after the known-good commit.
- Why it matters:
  - Prior diagnostics repeatedly found Build storing plane0 with
    `vertexIndex == 0`.
  - Fill `FirstHitFromVBuffer()` does `path.setVertexIndex(vertexIndex - 1)`.
  - With the current masked helper, `0 - 1` becomes `1023` while preserving the
    path flags.
  - `FirstHitFromVBuffer()` then checks
    `HasFinishedSurfaceBounces(path.getVertexIndex() + 1, ...)`, so a wrapped
    `1023` becomes `1024` and will exceed normal bounce limits immediately.
  - That can force the path into `terminateAtNextBounce` / terminal Fill logic
    before the first normal diffuse scatter has a chance to mark
    `stablePlaneBaseScatterDiff`.
- Difference versus upstream:
  - Upstream's unsafe arithmetic can borrow/carry between the low vertex bits and
    high flag bits. That is ugly, but it is the behavior RTXPT was written around.
  - The current masked arithmetic prevents flag borrow/carry, but it also makes
    low-bit underflow/overflow wrap silently inside the 10-bit vertex field.
- Minimal causality tests:
  - Temporarily restore the upstream unsafe `PathState` vertex helpers only.
  - Or guard `FirstHitFromVBuffer()` against `vertexIndex == 0` and make this state
    impossible before `HasFinishedSurfaceBounces()`.
  - Instrument `flagsAndVertexIndex` before and after `setVertexIndex(vertexIndex - 1)`
    in Fill to confirm whether affected pixels become vertex `1023`.

Strong candidate: local NEE stores total radiance as specular average.

- Upstream `PathTracerNEE.hlsli` accumulates direct-light specular average as:
  - `float4 bsdfThp = bsdf.eval(...)`;
  - `radiance = bsdfThp.rgb * lightSample.Li`;
  - `specAvg = bsdfThp.w * Average(lightSample.Li)`;
  - then both `radiance` and `specAvg` are multiplied by path throughput.
- Local `PathTracer.hlsli::HandleNEE()` currently combines direct and environment
  NEE into `directRadiance`, then writes:
  - `specAvg = preScatterPath.hasFlag(stablePlaneBaseScatterDiff) ? 0.0 :
    Average(directRadiance)`;
  - `result.AccumulateRadiance(directRadiance, specAvg)`.
- This local simplification existed at `cf829f`, so it is not the only regression
  by itself.
- Why it became a strong candidate now:
  - `StablePlane::GetNoisyDiffRadiance()` computes diffuse radiance as:
    `l.rgb * saturate(1.0 - l.a / Average(l.rgb))`.
  - If `l.a == Average(l.rgb)`, the diffuse component becomes exactly zero.
  - If the wrapped/terminal Fill path reaches NEE before any normal diffuse scatter
    sets `stablePlaneBaseScatterDiff`, ordinary diffuse direct lighting can be stored
    as fully specular, making the realtime diffuse component black even though noisy
    RGB radiance exists.
- This matches the reported symptom more directly than generic "NEE is zero":
  diffuse can be black because the denoiser split classifies the available radiance
  as specular.
- Minimal causality tests:
  - In local `HandleNEE()`, carry spec average from BSDF evaluation like upstream
    instead of using `Average(directRadiance)`.
  - As a diagnostic only, force `specAvg = 0.0` for ordinary non-delta diffuse test
    pixels and check whether `StablePlane_DiffRadiance` becomes visible.
  - Visualize `SP.GetNoisyRadianceAndSpecRA().rgb` and `.a` before demodulation:
    if RGB is nonzero and `.a` tracks `Average(rgb)`, this split is confirmed.

Strong candidate: `970a7378` added full nested-dielectric handling to local realtime
`HandleHit()`.

- `cf829f` local realtime `HandleHit()` did not yet run the full
  `HandleNestedDielectrics(surfaceData, path, workingContext)` path.
- `970a7378` added:
  - `HandleNestedDielectrics()` for full `SurfaceData`;
  - volume absorption before nested handling;
  - `UpdateSurfaceOutsideIoR()`;
  - transmission-time `UpdateNestedDielectricsOnScatterTransmission()`.
- Upstream has equivalent nested-dielectric logic, but the local port differs in
  structure because `SurfaceData` is preloaded in closest-hit and then passed into
  `PathTracer::HandleHit()`.
- Why it matters:
  - `HandleNestedDielectrics()` calls `path.decrementVertexIndex()` on rejected false
    hits.
  - The helper takes full `PathState` as `inout`.
  - Earlier diagnostics in this file already showed suspicious `inout PathState`
    copy-back behavior around this helper: low vertex bits and later the active bit
    appeared to change across the accepted-return boundary even when internal probes
    did not see the same state.
  - The current masked `decrementVertexIndex()` from `529f312d` makes any reject at
    vertex zero wrap to `1023` without clearing active flags.
- Already investigated but still relevant:
  - The quality-1 reject-limit behavior was brought closer to upstream by falling
    through to accepted-hit handling when the reject limit is reached.
  - An explicit empty-interior-list guard was added for
    `InteriorList::isTrueIntersection()`, but diagnostics still indicated that Fill
    could reject empty-stack hits unless the caller avoided the problematic helper
    path.
- Minimal causality tests:
  - Keep the helper, but snapshot and restore only the vertex bits around accepted
    nested handling in both Build and Fill to see whether diffuse recovers.
  - In Fill only, keep the previous empty-stack shortcut and compare final
    `PackedNoisyRadianceAndSpecAvg`.
  - Test `970a7378^` plus only the BxDF changes, if possible, to separate BxDF parity
    from nested-state parity.

Candidate: current ray-cone propagation is incomplete versus upstream.

- Upstream `UpdatePathTravelledLengthOnly()` does:
  - `path.rayCone = path.rayCone.propagateDistance(rayTCurrent)`;
  - then advances scene length.
- Local `PathTracerStablePlanes.hlsli::UpdatePathTravelledLengthOnly()` currently
  only advances scene length.
- Upstream `GenerateScatterRay()` also expands ray cone spread on non-delta scatter:
  - `path.rayCone = RayCone::make(path.rayCone.getWidth(),
    min(path.rayCone.getSpreadAngle() +
    ComputeRayConeSpreadAngleExpansionByScatterPDF(bs.pdf), 2.0 * K_PI))`.
- Local `GenerateScatterRay()` does not perform this expansion.
- Why it matters:
  - Ray cones feed material texture LOD, light-sampler coherence, and some upstream
    bridge decisions.
  - This can change BxDF inputs and NEE sampling after the first hit.
- Why it is not the top suspect:
  - It does not directly explain an exactly black diffuse split when RGB radiance is
    otherwise visible.
  - It is still worth fixing or probing after the vertex/specAvg chain is tested.

Candidate: local `HandleHit()` preloads `SurfaceData` in closest-hit before
`UpdatePathTravelled()`.

- Upstream loads `SurfaceData` inside `PathTracer::HandleHit()` after
  `UpdatePathTravelled()`, using the updated `path.rayCone` and vertex index.
- Local closest-hit calls `LoadCurrentSurfaceData()` before entering
  `PathTracer::HandleHit()`, and `PathTracer::HandleHit()` then updates path travel.
- This is an upstream difference, but it was already present in the known-good
  `cf829f` commit.
- Conclusion for this pass:
  - Do not treat this as the primary regression source.
  - Keep it in mind because it amplifies the ray-cone propagation difference: local
    realtime material sampling currently cannot use the same post-travel ray cone and
    vertex index that upstream uses for `Bridge::loadSurface()`.

Candidate already checked: payload size and stable-plane wire layout.

- Current `PathPayload` is still five `uint4` lanes, i.e. 80 bytes.
- `RTXPTRayTracingPass.cpp` currently sets `MaxPayloadSize` to
  `sizeof(float) * 40`, i.e. 160 bytes, so the RT payload is not obviously truncated.
- Local `PathPayload::{pack,unpack}` and `StablePlane::{PackCustomPayload,
  UnpackCustomPayload}` use the same 5-lane order as upstream, though local code now
  routes through `RTXPT_PATH_PAYLOAD_UINT4_COUNT` and array conversion helpers.
- CPU/HLSL `StablePlane` layout was previously audited and remains aligned.
- Conclusion for this pass:
  - Keep the layout guards and `MaxPayloadSize` in sync, but current evidence does
    not point to payload truncation as the diffuse-black cause.

Candidate already checked: stable-plane `FlagsAndVertexIndex` and `PackedCounters`
are intentionally not restored for base planes.

- Upstream `StoreStablePlane()` calls for the base plane also pass `0, 0` for
  `flagsAndVertexIndex` and `packedCounters`.
- Local `PathTracerStablePlanes.hlsli` currently does the same.
- `FirstHitFromVBuffer()` in both upstream and local does not restore those fields
  from the stable plane before setting the Fill path state.
- Conclusion for this pass:
  - Missing stable-plane flags/counters restore is not a local divergence by itself.
  - The more suspicious part is how the local masked vertex helpers behave when the
    stored stable-plane vertex index is zero.

Candidate mostly aligned: BxDF diffuse/specular estimate and denoiser demodulation.

- Current `ActiveBSDF::estimateSpecDiffBSDF()` is now much closer to upstream
  `StandardBSDF::estimateSpecDiffBSDF()` than the earlier simple
  `max(diffuse/specular, 0.04)` estimate.
- Current post-process prepare/final merge behavior matches upstream at the important
  level:
  - prepare divides noisy diffuse by `DiffBSDFEstimate`;
  - final merge multiplies denoised diffuse by the same estimate.
- Therefore the demodulate/remodulate path is not a strong standalone explanation for
  black diffuse.
- However, if `PackedNoisyRadianceAndSpecAvg.a` equals the total noisy radiance
  average, `GetNoisyDiffRadiance()` will feed zero into this otherwise-correct
  demodulation path.

Suggested next probes, in priority order:

1. Probe the local Fill stable-plane split directly:
   - output `SP.GetNoisyRadianceAndSpecRA().rgb`;
   - output `SP.GetNoisyRadianceAndSpecRA().a`;
   - output `SP.GetNoisyDiffRadiance()`;
   - output unpacked `DiffBSDFEstimate`.
   If RGB is nonzero, alpha is approximately `Average(rgb)`, and diffuse is zero,
   the `specAvg` classification path is confirmed.
2. Probe `FirstHitFromVBuffer()` immediately after
   `path.setVertexIndex(vertexIndex - 1)`:
   - stable-plane `vertexIndex`;
   - resulting `path.getVertexIndex()`;
   - raw `flagsAndVertexIndex`;
   - result of the subsequent `HasFinishedSurfaceBounces()` check.
3. Temporarily restore upstream unsafe vertex helpers from `cf829f` and rerun realtime
   diffuse debug view. This is a causality test, not necessarily the final preferred
   fix.
4. Temporarily make local NEE carry upstream-style spec average instead of
   `Average(directRadiance)`. If diffuse returns while combined radiance is unchanged,
   the denoiser split was the immediate cause.
5. After the vertex/specAvg path is proven or ruled out, fix ray cone propagation to
   match upstream and re-check material/NEE stability.

## 2026-06-07: Possible DXC / Full `inout PathState` Copy-Back Repro Candidate

This note captures the current strongest compiler-sensitive suspicion. It is not yet
proof of a DXC compiler bug, but it records the local HLSL shape that appears capable
of reproducing the realtime active-bit loss.

Problem summary:

- RTXPT logic does not show a valid accepted-hit path that should clear
  `PathFlags::active` after `HandleNestedDielectrics()` accepts a hit.
- The failure is concentrated at the `HandleNestedDielectrics(surfaceData, path,
  workingContext)` boundary where a full `PathState` is passed as `inout`.
- Earlier diagnostics in the same area showed low vertex bits being cleared across the
  accepted nested-dielectric return boundary.
- The latest diagnostics showed the raw active bit present before the call and absent
  immediately after the accepted return.
- A post-return active-bit restore marker was observed as `#E0E0E0`, proving the
  restore branch executed, but the final classifier still reported the later
  after-nested inactive state. This suggests field write/read or later full-struct
  copy-back behavior remains unstable in this pattern.

Relevant packed flag layout:

```hlsl
static const uint kVertexIndexBitCount = 10u;

enum class PathFlags
{
    active = (1 << 0),
    // ...
};

struct PathState
{
    // Large mixed payload: packed floats, nested structs, counters, ray cone, etc.
    uint4        PackOriginId;
    uint4        PackDirSceneLength;
    uint2        pack23;
#if PATH_TRACER_MODE != PATH_TRACER_MODE_BUILD_STABLE_PLANES
    uint2        pack45;
#endif
    InteriorList interiorList;
    uint         packedCounters;
    uint         stableBranchID;
    RayCone      rayCone;
    uint         pack0;
    uint         pack1;
    uint         flagsAndVertexIndex;

    bool isActive()
    {
        return (flagsAndVertexIndex & (((uint)PathFlags::active) << kVertexIndexBitCount)) != 0u;
    }
};
```

Problematic caller shape:

```hlsl
inline void HandleHit(inout PathState path,
                      SurfaceData surfaceData,
                      const float3 surfaceEmission,
                      const float3 rayOrigin,
                      const float3 rayDir,
                      const float rayTCurrent,
                      const WorkingContext workingContext)
{
    const uint activeBit   = ((uint)PathFlags::active) << kVertexIndexBitCount;
    const uint flagsBefore = path.flagsAndVertexIndex;

    const bool rejectedFalseHit =
        !HandleNestedDielectrics(surfaceData, path, workingContext);

    const uint flagsAfter = path.flagsAndVertexIndex;

    // Observed in FillStablePlanes:
    // (flagsBefore & activeBit) != 0
    // (flagsAfter  & activeBit) == 0
    // rejectedFalseHit == false
    //
    // In this state, NEE is skipped because path.isActive() is false.
    if (!rejectedFalseHit && !path.isActive())
    {
        // Diagnostic marker: accepted nested return lost active bit.
    }
}
```

Callee shape that should not clear active on accepted return:

```hlsl
inline bool HandleNestedDielectrics(inout SurfaceData surfaceData,
                                    inout PathState path,
                                    const WorkingContext workingContext)
{
#if RTXPT_NESTED_DIELECTRICS_QUALITY > 0
    if (surfaceData.shadingData.mtl.isThinSurface())
        return true;

    const uint nestedPriority =
        surfaceData.shadingData.mtl.getNestedPriority();

    const bool trueIntersection =
        path.interiorList.isTrueIntersection(nestedPriority);

    if (!trueIntersection)
    {
        const uint maxRejectedHits =
            GetMaxRejectedDielectricHits(RTXPT_NESTED_DIELECTRICS_QUALITY);

        if (path.getCounter(PackedCounters::RejectedHits) < maxRejectedHits)
        {
            path.incrementCounter(PackedCounters::RejectedHits);
            path.interiorList.handleIntersection(
                surfaceData.shadingData.materialID,
                nestedPriority,
                surfaceData.shadingData.frontFacing);
            path.SetOrigin(ComputeRayOrigin(
                surfaceData.shadingData.posW,
                -surfaceData.shadingData.faceNCorrected));
            path.decrementVertexIndex();
            return false;
        }

#if RTXPT_NESTED_DIELECTRICS_QUALITY == 2
        path.terminate();
        return false;
#endif
    }

    const float outsideIoR =
        ComputeOutsideIoR(path.interiorList,
                          surfaceData.shadingData.materialID,
                          surfaceData.shadingData.frontFacing);
    UpdateSurfaceOutsideIoR(surfaceData, outsideIoR);
#endif

    // Accepted hit path reaches this return. There is no explicit path.terminate()
    // or active-bit clear here.
    return true;
}
```

Why this is suspicious:

- `PathState` is a large struct with conditional fields and nested structs.
- The callee mutates only selected `PathState` members on rejected-hit paths, while the
  accepted path mostly reads `path.interiorList` and updates `surfaceData`.
- The caller observes `flagsAndVertexIndex` changing across an accepted `inout`
  function return even though the accepted path has no matching active-bit clear.
- Bit-level diagnostics around the same boundary previously found vertex-index bits
  being cleared, which points at the same packed field rather than at normal RTXPT
  path termination.

Current engineering conclusion:

- Treat this as a high-probability compiler-sensitive HLSL pattern:
  full `inout PathState` copy-back around nested dielectric handling.
- Do not yet call it a proven DXC bug without a smaller standalone shader reduction
  and optimization-level comparison.
- Preferred local fix direction: stop passing the full `PathState` into
  `HandleNestedDielectrics()`. Pass only the required fields (`InteriorList`,
  rejected-hit counter, origin update, vertex-index update, outside IoR / accepted
  result) and apply changes explicitly in the caller.

## 2026-06-07: Workaround Trial - Avoid Full `PathState` `inout`

Follow-up after DXC upgrade:

- Stable DXC `v1.9.2602.24` did not change the realtime probe result.
- Preview DXC `v1.10.2605.24` did not change the realtime probe result.
- Therefore the next local experiment is an engineering workaround rather than
  another compiler-version probe.

Workaround shape:

- `HandleNestedDielectrics()` no longer receives the full `PathState` as `inout`.
- It receives only the small state it needs to inspect: `InteriorList` and the
  rejected-hit counter.
- It returns a `NestedDielectricsResult` containing explicit copy-back fields:
  updated `InteriorList`, rejected-hit counter, optional origin update, optional
  vertex-index decrement, optional path termination, accepted/rejected result, and
  outside IoR.
- The caller applies those fields to `path` directly after the call.

Validation completed so far:

- Direct DXC compile of `PathTracerSample.rgen` with
  `PATH_TRACER_MODE=PATH_TRACER_MODE_FILL_STABLE_PLANES` succeeds with only the
  known payload-qualifier warning.
- Direct DXC compile of `PathTracerSample.rgen` with
  `PATH_TRACER_MODE=PATH_TRACER_MODE_BUILD_STABLE_PLANES` succeeds with only the
  known payload-qualifier warning.
- `RTXPT` target builds successfully in `build/x64/Debug`.

Runtime interpretation for the active diagnostic state:

- If output changes from `#E0E0E0` to cyan/green/red/blue-class markers, then the
  full-`PathState` `inout` boundary was at least part of the trigger.
- If output remains `#E0E0E0`, then active-bit loss is not caused solely by
  `HandleNestedDielectrics()` taking full `PathState` as `inout`; continue tracing
  the next write/read boundary after the restore marker.

Follow-up result:

- After the small-field `NestedDielectricsResult` workaround, the realtime output
  still reported the old `#E0E0E0` diagnostic.
- This means the restore branch still executes, but the final Fill path still reaches
  the inactive handling path.
- The next probe replaces the single restore marker with an active-state stage code
  in `StableRadiance.w`, so post-process debug mode 0 can identify the first point
  where the restored active bit disappears.

Current stage-code color map:

- Green: code `2`; direct active bit is visible immediately after restore.
- Cyan: code `3`; direct active bit survives through the emission block.
- Blue: code `4`; direct active bit survives through `isTerminatingAtNextBounce()`.
- Blue-cyan: code `5`; the final inactive branch was entered while the direct active
  bit was still set, pointing at `PathState::isActive()` / `hasFlag()` or compiler
  read behavior rather than an actual bit clear.
- Lime: code `6`; execution reached immediately before evaluating
  `path.isTerminatingAtNextBounce()`, but the following after-call stage did not
  overwrite the code. This points at the `isTerminatingAtNextBounce()` expression or
  the following control-flow / UAV-write visibility.
- Red: code `12`; direct active bit is missing immediately after the restore write.
- Magenta: code `13`; direct active bit is lost before/through the emission block.
- Yellow: code `14`; direct active bit is lost during/after
  `isTerminatingAtNextBounce()`.
- Orange: code `15`; direct active bit is lost only when entering the final
  inactive branch.
- Pink: code `16`; same-block overwrite immediately after code `3` was observed.
  If code `3` remains instead of code `16`, the issue is no longer
  `isTerminatingAtNextBounce()`; it points at same-address UAV write ordering,
  write competition, or compiler handling of consecutive debug writes.

Observed follow-up:

- Runtime color `#00BFE0` is closest to the cyan/code-`3` class.
- Interpretation: the restored active bit survives through the emission block.
- Since the prior probe did not advance to code `4/14/15`, the next split writes
  stage codes using a saved `pixelPos` instead of `path.GetPixelPos()` and inserts
  code `6` immediately before `path.isTerminatingAtNextBounce()`.
- Runtime still reported `#00BFE0` after the saved-`pixelPos` / code-`6` probe.
  The next split writes code `16` immediately after code `3` in the same block to
  determine whether a second same-address debug write can become visible at all.
- Runtime still reported `#00BFE0` after the same-block code-`16` write. This makes
  the pixel-level `StableRadiance.w` trace suspect: other Fill paths for the same
  pixel can race and leave code `3` as the final visible value.
- The next split writes the same stage code into the current stable plane's
  `PackedNoisyRadianceAndSpecAvg` as a plane-local marker (`.r = code`, `.a = 16`).
  Post-process debug mode 0 now gives this plane-local marker priority over
  `StableRadiance.w`.
- Runtime still reported `#00BFE0`, which means the previous frame could still have
  been reading the `StableRadiance.w` fallback or missing the plane-local marker on
  plane0.
- The current split writes the plane-local marker both to `path.getStablePlaneIndex()`
  and to plane0. Post-process debug mode 0 no longer falls back to `StableRadiance.w`:
  absence of a plane-local marker is shown as dark gray.
- Runtime still reported `#00BFE0` with fallback disabled. This could mean true
  plane-local code `3`, but it can also be a false-positive because the first
  plane-local marker signature only checked `.a > 8`.
- The current split strengthens the marker signature to
  `.r = code, .g = 13, .b = 7, .a = 16`. Post-process debug mode 0 only treats a
  plane as a trace marker when all three signature channels match.
- Runtime still reported `#00BFE0`, so the marker is real, but the "last write wins"
  float marker remains vulnerable to write competition: a path that only reaches the
  emission-stage write can race with a path that reaches later stages.
- The current split stores the trace in `StablePlane.PackedCounters` using
  `InterlockedMax()`. The high byte carries signature `0xD1`, and the low byte is a
  monotonic stage code. Post-process debug mode 0 reads this integer trace first.

Current atomic stage-code color map:

- Green: code `20`; active bit is visible immediately after restore.
- Cyan: code `30`; active bit survives through the emission block.
- Pink: code `40`; same-block write immediately after code `30` is visible.
- Lime: code `50`; execution reaches immediately before
  `path.isTerminatingAtNextBounce()`.
- Blue: code `60`; active bit survives through `isTerminatingAtNextBounce()`.
- Blue-cyan: code `70`; the final inactive branch was entered while the direct
  active bit was still set.
- Red: code `120`; active bit is missing immediately after restore.
- Magenta: code `130`; active bit is lost before/through the emission block.
- Yellow: code `140`; active bit is lost during/after
  `isTerminatingAtNextBounce()`.
- Orange: code `150`; direct active bit is lost only when entering the final
  inactive branch.

Follow-up split:

- Plane-local atomic trace still showed cyan/code `30`.
- The current shader no longer writes plane-local code `30` or code `40` from
  `HandleHit()`. It writes code `50` directly when the emission-after-active test
  succeeds.
- Therefore any remaining cyan/code `30` in the next run indicates stale/foreign
  trace data, a shader-reload problem, or a different writer still producing that
  signature outside the currently inspected `HandleHit()` path.
- Follow-up still showed cyan. Current source and runtime shader copies no longer
  contain a plane-local code-`30` writer, so the next check changes post-process'
  code-`30` color from cyan to red. If cyan remains after this, the post-process
  shader itself is stale or not reloaded.
- User confirmed the Debug executable path is
  `D:/DiligentEngine-hzqst/build/x64/Debug/DiligentSamples/Samples/RTXPT/Debug/RTXPT.exe`.
- Runtime still showed cyan after code-`30` was changed to red. The next probe adds
  `RTXPT_DEBUG_FINAL_MERGE_SENTINEL=1` in `RTXPTPostProcess.csh`, which forces
  plane0 final-merge debug pixels with surfaces to pure red and returns immediately.
  If cyan remains, the observed frame is not produced by this final-merge debug
  branch.
- Verification: `cmake --build build/x64/Debug --config Debug --target RTXPT`
  succeeded, and the source/runtime `RTXPTPostProcess.csh` SHA256 hashes match:
  `46E8129E93E4056FFB4E989524C40D72D9F2583BFA783F200DDC6F1758A35AFA`.
  The runtime copy under the confirmed Debug executable directory contains the
  sentinel.

## 2026-06-07: Root Cause Confirmed — DXC Miscompilation of Realtime `HandleHit` (Resolution)

This section supersedes the long "plane0 `vertexIndex == 0` / active-bit lost" symptom
chase above. Using the known-good rollback `cf829f` as an anchor, the realtime
diffuse/non-reflective black-out was bisected and root-caused, and the tree was returned to
a working-opaque configuration.

### Bisection result

- Known-good DiligentSamples commit: `cf829f3294c517c7046a0e0b200c66f9f5d6c57c` — realtime
  opaque/diffuse correct; realtime nested dielectrics / transmission were not yet ported.
- Regression range `cf829f..970a7378`. Reference mode is correct throughout, so the BxDF
  componentization commits (eval/sample math) are sound; the black-out is realtime-only.
- The realtime diffuse black-out is introduced by `970a7378` ("align realtime BxDF and
  material state"), which added full nested-dielectric handling to realtime
  `PathTracer::HandleHit`: volume absorption + `HandleNestedDielectrics(inout SurfaceData,
  inout PathState)` + the false-hit early-return.
- `529f312d`'s masked vertex accessors (`setVertexIndex`/`incrementVertexIndex`/
  `decrementVertexIndex`) were tested and are NOT the trigger. They were still reverted to
  the upstream `+= 1 / -= 1 / raw-OR` form because that is the correct port shape
  (matches `D:/RTXPT-fork`).

### Root cause = DXC miscompilation, not a fixable logic bug

Decisive observation: an opaque primary hit is classified thin
(`Bridge::isThinSurface` = thin-flag OR no-transmission) and has an empty interior list, so
it executes NONE of the nested-dielectric code (the `if (!mtl.isThinSurface())` guard and the
`if (!interiorList.isEmpty())` volume guard are both false). Yet merely having that code
present in `HandleHit`'s body blacks out opaque. DXC compiles all branches of `HandleHit`;
the presence of the nested-dielectric branch corrupts codegen of the path's packed
`flagsAndVertexIndex` (active bit / vertex index), so the primary diffuse stable plane is
never recorded → black. This is exactly the "active bit lost across the boundary, but no
internal probe sees the clear" signature chased above — it was never a runtime data-flow
bug, it was codegen.

Experiment table (only the **presence of nested-dielectric code in `HandleHit`'s body**
toggles opaque; inline-vs-helper and masked-vs-upstream accessors do not):

| HandleHit nested-dielectric code | Vertex accessors | Opaque |
|---|---|---|
| absent (`cf829f`)                                   | upstream | OK    |
| present — `HandleNestedDielectrics(inout)` call (`970a7378`) | masked   | BLACK |
| absent (experimental revert)                        | masked   | OK    |
| present — inlined directly on `path`                | masked   | BLACK |
| present — inlined; scatter helper also inlined      | masked   | BLACK |
| present — inlined directly on `path`                | upstream | BLACK |
| absent (current resolution)                         | upstream | OK    |

Inlining (removing the nested `inout PathState` boundary so `path` only crosses
`HandleHit`'s own boundary) did NOT help — confirming the trigger is the code's presence in
the compiled function, not the `inout` copy-back per se. Consistent with everything above:
the symptom moves with code shape and is unaffected by DXC stable/preview version swaps.

### Current code state (the resolution)

- `PathTracer::HandleHit` contains NO realtime nested-dielectric handling — cf829f-equivalent.
  A `NOTE:` comment in the function body records why it must not be re-added.
- Removed the now-unreachable realtime helpers `HandleNestedDielectrics(inout SurfaceData,
  inout PathState)` and `UpdateSurfaceOutsideIoR` (uncalled → not reachable from any entry
  point → not compiled → zero effect on the shader output; pure cleanup). The reference-mode
  `HandleNestedDielectrics(RTXPTMaterialHitPayload, ...)` overload in
  `PathTracerNestedDielectrics.hlsli` is untouched.
- `UpdateNestedDielectricsOnScatterTransmission` (scatter path, `GenerateScatterRay`) is kept
  and still called; it only affects transmission scatter and does not break opaque.
- `PathState` vertex accessors reverted from `529f312d`'s masked read-modify-write back to the
  upstream `flagsAndVertexIndex += 1 / -= 1` form.
- Result: realtime opaque/diffuse correct (user-confirmed). Realtime transmission through
  nested dielectrics is DEFERRED — same capability level as `cf829f`.

### Do NOT

Do not re-add nested-dielectric handling to realtime `HandleHit` until the miscompilation is
resolved. It will black out opaque again regardless of how it is written (helper or inline,
masked or upstream accessors). The earlier "eliminate the inout copy-back" attempts are dead
ends for this symptom.

### Two root-fix paths for realtime transmission

1. Minimal DXC repro + report: build a standalone shader reproducing "a compiled-but-not-
   executed code block in a function corrupts an adjacent packed bitfield in the same
   struct", diff across `-O` levels and DXC versions, and file a DXC issue. This is the
   cleanest proof and unblocks an upstream fix.
2. Refactor `PathState` packing: change the representation of `flagsAndVertexIndex` and the
   `#if RTXPT_NESTED_DIELECTRICS_QUALITY`-conditional members (e.g., split the active/vertex
   bits out of the shared word, or avoid the masked read-modify-write pattern in the
   compiled-but-skipped branch) so the miscompiled pattern is avoided, then re-introduce
   nested-dielectric handling in realtime `HandleHit` and re-verify opaque + transmission.

## 2026-06-07 (cont.): Path #2 Implemented — Opaque Fixed; Realtime Transmission Still Black (codegen-class)

Root-fix path #2 above (refactor `PathState` packing) was implemented and **confirmed to resolve the
opaque/diffuse black-out**. Realtime transmission was then investigated extensively and remains black; it
is now isolated to the FILL pass and bears the same "logic faithful, behavior wrong" signature as the
opaque DXC miscompilation. This section supersedes the "Do NOT re-add" note above for the opaque case
(Gate 1 made re-adding safe). Details below for the next session.

### What was implemented and committed (preserves the opaque fix)

All in `DiligentSamples/Samples/RTXPT/assets/shaders/PathTracer/`:

1. **Gate 1 — `PathState` packing split** (`PathState.hlsli`, `PathPayload.hlsli`). The shared packed word
   `flagsAndVertexIndex` was split into two independent struct members `flags` (PathFlags region, bits
   [10..31]) and `vertexIndex` (low 10 bits). All accessors (`hasFlag`/`setFlag`/`clearScatterEventFlags`/
   `get|setStablePlaneIndex`/`get|set|increment|decrementVertexIndex`) operate on the de-aliased members.
   The single 32-bit wire layout is unchanged — the two members are recombined only at the
   `PathPayload::pack`/`unpack` boundary (`packed[4].w = flags | (vertexIndex & kVertexIndexBitMask)`), so
   cross-pass/wire compatibility is preserved. This de-aliasing removes the intra-word read-modify-write
   pattern DXC was miscompiling.
2. **Gate 2 — nested-dielectric handling restored to realtime `HandleHit`** (`PathTracer.hlsli`), verbatim
   from `970a7378` (volume absorption + `HandleNestedDielectrics(inout SurfaceData, inout PathState)` +
   false-hit early-return; `volumeAbsorption` fed into `StablePlanesHandleHit`), plus the two helpers
   `UpdateSurfaceOutsideIoR` and `HandleNestedDielectrics`. On top of Gate 1's packing, **opaque shading
   stays correct** (user-confirmed) → the DXC miscompilation is resolved.
3. **`getDeltaLobeIndex` fix** (`PathTracer.hlsli`, `MakeBSDFSample`). Non-delta lobes now map to the
   invalid sentinel `0xFFFFFFFF` (matching upstream `IBSDF::getDeltaLobeIndex`), not `0`. Consumed by
   `StablePlanesOnScatter` via `StablePlanesAdvanceBranchID`; with `0` a non-delta scatter produced a
   valid-looking branchID and corrupted FILL branch tracking. **Correct and kept**, but did NOT fix the
   transmission black on its own.

Result: realtime **opaque/diffuse correct**, realtime **metal reflection correct** (via PSR on plane 0).
Realtime **transmission (glass) still black**.

### Realtime transmission — investigation & current localization

This feature **never worked** in the port (cf829f lacked it; `970a7378` added it but the opaque black-out
hid that it was broken). So this is completing an unfinished port, not fixing a regression.

User tests with the **denoiser OFF** → final output = `StablePlanes::GetAllRadiance` =
`StableRadiance + Σ_plane GetNoisyRadiance().xyz` (raw radiance; it does NOT use `specAvg`). Decisive probe
results:

- **Per-pixel false-color of the stable-plane buffer** (R = plane0 has radiance; G = a secondary plane
  idx>0 is LAID i.e. valid branchID; B = a secondary plane is FILLED i.e. has radiance): glass spheres are
  **GREEN** → BUILD **lays** the transmission stable plane behind the glass, but FILL **never fills** any
  secondary plane (no B anywhere). Diffuse spheres are RED (plane0 filled), as expected.
- **Magenta marker on `StablePlanesOnScatter` transition onto a secondary plane: never fired on glass** →
  the FILL path's advanced `stableBranchID` never matches a secondary plane's stored branchID; the path
  never routes onto / deposits into the transmission plane.
- **Green marker on any FILL hit past the base plane (`BouncesFromStablePlane >= 1`): never fired on
  glass** → the FILL path does not reach a surface behind the glass.

Ruled OUT (each by a targeted probe): NEE spec/diffuse demodulation (the no-denoiser path ignores
`specAvg`, so the early `specAvg=0` probe was a no-op — the demodulation hypothesis from earlier in this
doc was never actually exercised in the denoiser-off config); volume absorption (shared with the working
reference path in `PathTracerSample.rgen`); `ValidateNaNs` (port runs it, upstream has it `#if 0` —
disabling did not help; an independent divergence, not this bug); BUILD plane laydown (correct — plane is
laid); `_activeStablePlaneCount` (defaults to `kRTXPTStablePlaneCount=3`), `PSDDominantDeltaLobe` (glass
material sets it = 0 → P1=1, dominant follows transmission), `cStablePlaneMaxVertexIndex` (15 both),
initial `stableBranchID` (1 both) — all faithful.

Two deep cross-repo agent traces (FILL transmission scatter, BSDF sample/eval, lobe componentization,
branchID lifecycle, `StablePlanesOnScatter`/`FirstHitFromVBuffer`/`CommitDenoiserRadiance`) both concluded
the FILL transmission **logic is faithful to upstream** `D:/RTXPT-fork`. The only concrete divergence found
was the `getDeltaLobeIndex` sentinel (fixed, insufficient).

### Conclusion / next session

Everything faithful + still broken = the **same signature as the opaque DXC miscompilation** (logic
correct, codegen wrong). The realtime FILL `HandleHit`/`GenerateScatterRay` now contains the
nested-dielectric + delta-transmission scatter code; the working hypothesis is that this corrupts the
codegen of some packed FILL path-state (candidates: `stableBranchID`, `interiorList`, or the flags word
again) so the FILL branch traversal / refraction misbehaves at runtime despite correct source — mirroring
how the opaque path was corrupted before Gate 1.

Recommended next steps (source-level diffing is exhausted):

1. **GPU capture (RenderDoc/PIX)** of a glass pixel: read the stable-plane header (per-plane branchIDs) and
   the FILL path state to get ground truth — does the FILL path physically refract through the glass, and
   what `stableBranchID` does it carry vs the stored plane branchIDs? This distinguishes codegen corruption
   from a remaining logic gap.
2. If codegen: apply the Gate-1-style mitigation to whichever FILL packed field the capture implicates
   (de-alias it / change its representation), or build a minimal DXC repro and report upstream.
3. Keep the `getDeltaLobeIndex` fix regardless.

Do NOT re-chase: demodulation, volume, `ValidateNaNs`, BUILD laydown, or the branchID/lobe-index *values*
(all verified faithful). The open question is purely **why FILL does not route the path through the glass
to fill the plane BUILD laid**, and the evidence points at codegen, not source logic.

## 2026-06-08: Gate 3 (de-alias `stablePlaneIndex`) — implemented, TESTED, RULED OUT

Sharper symptom (vs reference screenshot): the **smooth/clear** glass sphere (top-right) is fully black,
while the **rough/frosted** sphere renders (noisy). Rough transmission is a non-delta lobe handled in place
on plane 0; smooth transmission is a **delta** lobe needing a secondary stable plane → the break is
specifically the **delta-transmission secondary-plane path**, matching the earlier localization.

**Four fresh parallel cross-repo audits** (StablePlanes infra / branchID round-trip; BxDF delta-lobe
transmission; FILL transition `FirstHitFromVBuffer`+`StablePlanesOnScatter`; BUILD laydown
`StablePlanesHandleHit`/`SplitDeltaPath` + final merge `GetAllRadiance`) **all re-confirmed the realtime
shader logic is faithful to upstream** `D:/RTXPT-fork`. Notably the suspected
`deltaLobeCount = max(cMaxDeltaLobes - 1, deltaLobeCount)` is **faithful** — upstream uses `max` too (not a
min→max inversion).

**Gate 3 change** (the one aliased field Gate 1 left behind): `stablePlaneIndex` lived as a 2-bit subfield
(bits 24..25) inside the shared `flags` word, read/written by masked RMW. `StablePlanesOnScatter` is the
only site that writes it with a **non-zero** value, **interleaved with four `setFlag` RMWs on that same
word** — identical intra-word aliasing signature to the opaque `flagsAndVertexIndex` bug; and
`CommitDenoiserRadiance` uses `getStablePlaneIndex()` as the radiance write-address selector. Opaque/metal
only ever set the index once (to 0, at init), never during the interleaved transition → explains why they
were unaffected. Fix: split `stablePlaneIndex` into its own `PathState` member; recombine into
`packed[4].w` bits 24..25 only at the `PathPayload::pack/unpack` boundary. Wire format **byte-identical**
(verified: FILL ray round-trip and BUILD exploration `PackCustomPayload` lane-copy both preserve
`packed[4].w`). Files: `PathState.hlsli`, `PathPayload.hlsli`.

**RESULT: top-right glass STILL BLACK.** Opaque/metal remain correct (no regression). → `stablePlaneIndex`
aliasing is **NOT** the cause. Kept anyway as a harmless, Gate-1-aligned de-alias that removes a latent
aliasing pattern and rules out the last masked sub-field of `flags`.

### Conclusion (updated)

Source-level diffing is now **exhausted**: every stable-plane shader cluster audited faithful, AND both
remaining `flags`-aliased sub-fields (`vertexIndex` in Gate 1, `stablePlaneIndex` in Gate 3) de-aliased
with no effect on transmission. This **strongly confirms the codegen-class diagnosis**; the next ground
-truth step is **GPU capture (RenderDoc/PIX)** of a smooth-glass pixel (read per-plane branchIDs + FILL
path state: does FILL physically refract through the glass, and what `stableBranchID` does it carry vs the
stored plane branchIDs?). Add to **Do NOT re-chase**: `stablePlaneIndex` aliasing (Gate 3), and the four
audited clusters above.
