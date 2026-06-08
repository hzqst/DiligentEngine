# Realtime BxDF Symptoms and Conclusions

Last updated: 2026-06-08

This note is intentionally compressed. It keeps only the observed symptoms, confirmed
conclusions, and current next action.

## Scope

- Scene: `convergence-test.scene.json`.
- Mode: RTXPT realtime / stable planes.
- Reference mode: visually correct after the BxDF fixes.
- Realtime mode: opaque and metal paths are currently correct; smooth-glass
  transmission is still broken.

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

Source-level diffing is exhausted. Opaque, diffuse, and metal realtime rendering are
correct after Gate 1 / Gate 2 / Gate 3, but smooth-glass realtime transmission remains
black because the Fill path does not route through the glass onto the secondary stable
plane that Build lays down.

The strongest current conclusion is a DXC / shader-codegen-class failure in the
realtime Fill delta-transmission path, not a known source-logic mismatch with upstream
RTXPT.

## Next Action

Use GPU capture, preferably RenderDoc or PIX, on a smooth-glass pixel. Read the
stable-plane header and the Fill path state to answer:

- Does the Fill path physically refract through the glass?
- What `stableBranchID` does the Fill path carry?
- How does that value compare with the stored secondary-plane branchID?
- Which packed or conditional Fill state field diverges at runtime?

If the capture confirms codegen corruption, apply a Gate-1-style mitigation to the
specific implicated field, or extract a minimal DXC repro and report it upstream.
