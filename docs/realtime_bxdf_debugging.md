# Realtime BxDF Symptoms and Conclusions

Last updated: 2026-06-09

This note is intentionally compressed. It keeps only the observed symptoms, confirmed
conclusions, and current next action.

## RESOLVED (2026-06-09) — Root Cause: DXC `-Od` Miscompiles `HandleHit`

Root cause confirmed. The DXC flag `-Od` (disable optimizations) makes DXC emit
incorrect DXIL for `PathTracer::HandleHit`, causing it to malfunction. The defect
is consistent with DXC's optimization-disabled DXIL mishandling nested `inout`
parameters — `HandleHit` and its callees thread deeply nested `inout` state, which
is exactly the construct that triggers the bad codegen.

Fix: compile with optimizations enabled (drop `-Od`). With optimization on, DXC
emits correct DXIL, every symptom below is gone, and reference mode now renders
identically to upstream RTXPT.

This confirms the "DXC / shader-codegen-class failure" hypothesis recorded
throughout this note. All source-level audits stand: the ported shader logic was
faithful; the failure was purely `-Od` DXIL codegen. The Serena `suggested_commands`
smoke command has been updated to drop `-Od`.

## Scope

- Scene: `convergence-test.scene.json`.
- Mode: RTXPT realtime / stable planes.
- Reference mode: visually correct after the BxDF fixes.
- Realtime mode: correct after dropping `-Od` (see Resolved section). Previously,
  under `-Od`, smooth-glass transmission rendered black.

## Symptoms

- Earlier realtime symptom: most non-emissive / non-reflective objects rendered black.
- That opaque/diffuse black-out is fixed.
- Current remaining symptom: the smooth / clear glass sphere, especially the top-right
  sphere in the reference comparison, is fully black in realtime mode.
- Rough / frosted transmission renders, though noisy.
- Metal reflection remains correct through PSR on plane 0.
- With the denoiser off, the final realtime output comes from
  `StablePlanes::GetAllRadiance`:
  `StableRadiance + sum(GetNoisyRadiance().xyz)`.
- The remaining failure is specific to the delta-transmission secondary-stable-plane
  path. Rough transmission is non-delta and stays on plane 0, so it does not hit the
  same broken path.

## Historical Conclusions

- The earlier static-resource assertion failures are not the current cause:
  - missing `t_Lights` for `Reference`
  - missing `g_MiniConst` for `FillStablePlanes`
  - missing `g_MiniConst` for `RelaxDenoiserFinalMerge`
- Direct-light bridge resources and stable-plane UAV bindings were confirmed not to be
  the explanation for the realtime black output.
- Build-to-Fill visibility is not globally broken: Build writes are visible to Fill.
- Same-state UAV transitions in DiligentCore D3D12 do produce UAV barriers, so the
  evidence does not point to missing UAV barriers.
- The original opaque/diffuse black-out was a DXC codegen-class issue in realtime
  `PathTracer::HandleHit`, triggered by the presence of nested-dielectric code in the
  compiled function body.
- Splitting `PathState::flagsAndVertexIndex` into independent `flags` and
  `vertexIndex` members, while preserving the packed wire layout at
  `PathPayload::pack` / `unpack`, fixed the opaque/diffuse black-out.

## Current Code Conclusions

- Gate 1, `PathState` packing split:
  - kept
  - fixes the opaque/diffuse black-out
  - preserves the 32-bit wire layout by recombining at pack/unpack boundaries
- Gate 2, realtime nested-dielectric handling restored to `PathTracer::HandleHit`:
  - opaque/diffuse remains correct with Gate 1 in place
  - smooth-glass realtime transmission remains black
- `getDeltaLobeIndex` fix in `MakeBSDFSample`:
  - kept
  - non-delta lobes now use invalid sentinel `0xFFFFFFFF`, matching upstream
  - correct but insufficient to fix smooth-glass transmission
- Gate 3, `stablePlaneIndex` de-alias:
  - kept
  - splits the remaining masked subfield out of the shared `flags` word
  - preserves the packed wire layout at `PathPayload::pack` / `unpack`
  - does not fix the smooth-glass black output
  - rules out `stablePlaneIndex` aliasing as the current cause

## Current Failure Localization

- The ported realtime shader logic has been checked against upstream RTXPT for:
  - StablePlanes infrastructure and branchID round-trip
  - BxDF delta-lobe transmission
  - Fill transition through `FirstHitFromVBuffer` and `StablePlanesOnScatter`
  - Build laydown through `StablePlanesHandleHit`, `SplitDeltaPath`, and final merge
- These clusters are considered source-faithful to `D:/RTXPT-fork`.
- Build lays the secondary transmission stable plane behind the glass.
- Fill does not route onto / fill that secondary plane.
- Fill does not appear to reach the surface behind the smooth glass through the
  expected secondary-plane path.
- Since source-level parity is exhausted and the remaining symptoms match the earlier
  opaque DXC issue pattern, the current diagnosis is still codegen-class behavior in
  realtime Fill path-state / branch traversal.
- Candidate state affected by codegen remains the Fill path state involved in routing,
  especially `stableBranchID`, `interiorList`, or related packed / conditional fields.

## Ruled Out For The Current Symptom

- direct-light static resources
- dynamic stable-plane UAV binding
- Build-to-Fill resource visibility
- same-state UAV barrier handling
- NEE spec/diffuse demodulation in the denoiser-off path
- volume absorption
- `ValidateNaNs`
- Build stable-plane laydown
- `_activeStablePlaneCount`
- `PSDDominantDeltaLobe`
- `cStablePlaneMaxVertexIndex`
- initial `stableBranchID`
- branchID / lobe-index source values after the `getDeltaLobeIndex` fix
- `stablePlaneIndex` aliasing after Gate 3
- the audited shader clusters listed above

## Current Conclusion

RESOLVED — see the Resolved (2026-06-09) section at the top. The DXC /
shader-codegen-class hypothesis was correct: the realtime symptoms (opaque/diffuse
black-out and smooth-glass transmission black) were caused by `-Od` DXIL
miscompilation of `PathTracer::HandleHit`, not by a source-logic mismatch with
upstream RTXPT. Source-level diffing was exhausted and found the port faithful,
which is consistent with the codegen root cause. Enabling optimizations fixes all
of it.

## Next Action

None required for the rendering bug — resolved by dropping `-Od`. Optional
follow-up: extract a minimal DXC `-Od` nested-`inout` repro and report it upstream
so the optimization-disabled debug configuration can be used safely again.
