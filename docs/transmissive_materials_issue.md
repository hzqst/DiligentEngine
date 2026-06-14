# Transmissive Materials Render Incorrectly (Glass / Transparency)

Status: **resolved — classification + loader alpha-mode fixes applied, and the default `Max bounces`
aligned to upstream (4 → 20) to fix the remaining black thick glass; glTF-only IoR follow-up still open**
Scope: reference *and* realtime path-tracer modes (the defect is in material classification + BLAS setup, not mode-specific)
Repro scene: `assets/kitchen.scene.json`

## Symptoms

Compared to upstream RTXPT-fork rendering the same scene:

- Transparent glass (most visibly the `glass-liquid-ice` green-drink model) looks wrong — it
  appears mostly see-through / washed-out instead of a properly refracting dielectric.
- The kitchen "window" looks opaque/blown-out. (See the note in
  [Out of scope](#out-of-scope-not-caused-by-this-bug) — the big window has no glass pane; this
  is an exposure artifact, not transmission.)

The inconsistency reproduces even in **Reference** mode, ruling out the realtime/denoiser path.

## Root cause

**The port mis-classifies every transmissive material as *alpha-blended*, which makes its
geometry non-opaque and routes it through a stochastic alpha-blend any-hit shader. Upstream
RTXPT-fork keeps transmissive glass as opaque geometry and performs refraction solely in the
closest-hit BSDF; its any-hit does alpha *testing* only, never stochastic alpha blending of
transmissive surfaces.**

For glass authored with low `Opacity`, the stochastic any-hit discards the vast majority of ray
hits, so the refraction BSDF almost never executes.

### Verified causal chain (port)

1. **CPU material classification**
   Before the fix, `RTXPTMaterialIsAlphaBlended()` returned `true` for *any* transmission
   (`EnableTransmission || TransmissionFactor > 0 || DiffuseTransmissionFactor > 0`):
   `src/RTXPTMaterials.cpp:327-334`.
   It was additionally forced because the **DiligentTools glTF loader set `ALPHA_MODE_BLEND` on every
   material carrying `KHR_materials_transmission`** (`DiligentTools/AssetLoader/src/GLTFLoader.cpp`,
   ~line 1870, fixed together with the RTXPT classification). Either source set
   `kMaterialFlag_AlphaBlend` during upload:
   `src/RTXPTMaterials.cpp:144-145` and `src/RTXPTMaterials.cpp:539-541`.

2. **BLAS geometry flag**
   `RTXPTMaterialNeedsAnyHit()` = alpha-tested **or** alpha-blended (`src/RTXPTMaterials.cpp:336-342`).
   When true, the geometry is built as `RAYTRACING_GEOMETRY_FLAG_NONE` (non-opaque) instead of
   `RAYTRACING_GEOMETRY_FLAG_OPAQUE`: `src/RTXPTAccelerationStructures.cpp:379-381`.

3. **Stochastic alpha-blend any-hit**
   For an alpha-blended hit the any-hit reads the base-color alpha (= authored `Opacity`) and
   stochastically passes through:
   `shaders/PathTracer/PathTracerAnyHit.rahit:35-47`
   ```hlsl
   const float alpha = Bridge::getBaseColor(material, texCoord).a;   // = Opacity
   if (Hash32ToFloat(seed) > saturate(alpha))
       IgnoreHit();                                                  // ray passes straight through
   ```

### Why this corrupts the glass

`Opacity` is authored as a low value for these dielectrics, and the any-hit treats it as a
passthrough probability:

| Material (`.material.json`)            | Opacity | Any-hit effect           |
|----------------------------------------|---------|--------------------------|
| `glass-liquid-ice.TransparentGlass`    | 0.061   | ~94% of hits `IgnoreHit` |
| `glass-liquid-ice.Liquid`              | 0.086   | ~91% ignored             |
| `glass-liquid-ice.Ice`                 | 0.104   | ~90% ignored             |

So ~90% of camera rays pass straight through the glass with no refraction; only the remaining
fraction reach the closest-hit and refract. The result is a faint ghost of refraction over an
otherwise transparent background instead of a real refracting dielectric.

### Upstream reference behavior (RTXPT-fork)

- A smooth transmissive glass (`EnableAlphaTesting = false`) stays **opaque** at the BLAS level;
  transmission is purely a BSDF property
  (`Rtxpt/Shaders/PathTracerBridgeDonut.hlsli` `loadSurface`,
  `Rtxpt/Materials/MaterialsBaker.cpp` domain flagging).
- `ANYHIT_ENTRY` only performs alpha *testing* (`Bridge::AlphaTest`), never stochastic alpha
  blending of transmissive surfaces (`Rtxpt/Shaders/PathTracerMaterialSpecializations.hlsl`).
- For glass with `transmissionFactor = 1`, `metalness = 0`, the BSDF yields
  `pSpecularReflectionTransmission ≈ 1.0`, i.e. full Fresnel reflection + refraction.

## What was ruled out

The transmission code path itself is faithful to upstream — the bug is *only* in the
classification / BLAS routing above. Verified equivalent to RTXPT-fork:

- `MakeStandardBSDFData`: eta and `specularTransmission` — `shaders/.../BxDF.hlsli:106-112`.
- `transmissionAlbedo = thinSurface ? transmission : sqrt(transmission)` and the
  "rough transmission with same IoR → delta" guard (`eta == 1 ? alpha = 0`) —
  `shaders/.../BxDF.hlsli:730-754`.
- Thin-surface refraction override (`eta = 1` for the transmission lobe) —
  `shaders/.../BxDF.hlsli:516`, `:578-580`, `:649`.
- `FalcorBSDF::__init` / `getLobes` / `evalDeltaLobes` lobe selection and weights —
  `shaders/.../BxDF.hlsli:726-873`.
- Nested-dielectric false-hit rejection and caps (`kMaxRejectedDielectricHits = 4` at
  `RTXPT_NESTED_DIELECTRICS_QUALITY = 1`) — `shaders/PathTracer/PathTracerNestedDielectrics.hlsli`,
  `shaders/PathTracer/PathTracer.hlsli:317-392`.
- Material-extension (`.material.json`) loading: matches by ModelName/MaterialName and the files
  exist for the scene (`kitchen.Glass`, `glass-liquid-ice.*`, `transparent-machines-*`), so
  `HasTransmission`, IoR, and `ThinSurface` are populated correctly.

## Second root cause: opacity-1.0 glass renders black (default `Max bounces` 4 vs upstream 20)

The alpha-blend fix above makes transmissive glass build as opaque geometry and refract via the
closest-hit BSDF, but the **opacity-1.0 wine glasses (`kitchen.Glass`) still rendered solid black**.
This is an *independent* defect — a port default that diverged from upstream, not a transmission-code
bug:

- Upstream RTXPT-fork defaults `BounceCount = 20` (`Rtxpt/SampleUI.h`).
- The port defaulted `m_MaxBounces = 4` (`src/RTXPTSample.hpp`), and `kitchen.scene.json` does not
  override `maxBounces`, so the scene ran at 4 (see the screenshot's "Max bounce" slider).

A delta refraction increments the path vertex index but does **not** count as a diffuse bounce; the
path still terminates once `bounceCount < vertexIndex` (`HasFinishedSurfaceBounces`,
`shaders/PathTracer/PathTracer.hlsli`, identical to upstream). A thick wine-glass wall burns vertices
fast — outer-enter → inner-exit → far-inner-enter → far-outer-exit reaches ~vertex 4, and the
refracted ray is terminated *inside* the glass before it can reach the environment map, contributing
zero radiance → converges to black even at 149 samples. Thin / single-interface glass survives at 4
bounces, which is exactly why only the thick glasses were affected. At upstream's default of 20 there
is ample budget and the glass refracts correctly.

The transmission code itself is faithful to upstream — verified: CPU classification, `InteriorList` /
nested-dielectric handling, volume absorption (`loadHomogeneousVolumeData`), `FalcorBSDF` /
`MakeStandardBSDFData` lobe selection and eta, closest-hit surface setup, and the bounce-termination
logic. The only divergence was the default value.

### Fix

Align the port default with upstream (4 → 20):

- `src/RTXPTSample.hpp` — `m_MaxBounces = 20`
- `src/RTXPTFrameConstants.hpp` — `bounceCount = 20` (frame-constants struct default, for consistency)

## Out of scope (not caused by *this* alpha-blend bug)

- **Opacity-1.0 transmissive materials are unaffected by the alpha-blend defect specifically**,
  because the any-hit always accepts (`rand > 1.0` is never true). This covers `kitchen.Glass` (the
  wine glasses, meshes `Mesh_14` / `Mesh_185` / `Mesh_14.001`), the `transparent-machines` "HOME"
  sculpture, and `GlassMicrowave`. Note, however, that the wine glasses **did** still render black —
  for the unrelated bounce-budget reason documented in *Second root cause* above, not because of
  alpha blending.
- **The kitchen's big window has no glass pane.** Its meshes (`Window_Frame`, `Window_Blinds`,
  `Window_Sill`, `Window_Panels`, …) use opaque wood/plastic/metal materials; the opening shows
  the environment map directly. The blown-out/"opaque" window is an **auto-exposure / tone-mapping**
  artifact (the port screenshot uses auto-exposure with EV −4.0, which clips the bright exterior),
  not a transmission bug. (An earlier revision of this doc also blamed the dark wine glasses on
  exposure — that was incorrect; they were the bounce-count defect above.)

## Fix direction

Decouple *transmission* from *alpha-blend* so transmissive dielectrics stay opaque geometry and
refract via the closest-hit BSDF, exactly as upstream:

1. **`RTXPTMaterialIsAlphaBlended()` (`src/RTXPTMaterials.cpp:327-334`)** must not return `true`
   purely because a material has transmission. Reserve `kMaterialFlag_AlphaBlend` for genuine
   alpha-blended, *non-transmissive* surfaces (e.g. decals / foliage cards with `<1` opacity and no
   transmission).
2. **Do not force `ALPHA_MODE_BLEND` in the loader for `KHR_materials_transmission`, and do not let
   any legacy loader-forced value feed the alpha-blend flag.** When `kMaterialFlag_HasTransmission`
   is set, the material should build as opaque geometry (`RAYTRACING_GEOMETRY_FLAG_OPAQUE`) and not
   request the stochastic any-hit. GLTFViewer and `GLTF_PBR_Renderer` transmission display behavior
   is intentionally left as a separate follow-up.
3. Confirm `RTXPTMaterialNeedsAnyHit()` (`src/RTXPTMaterials.cpp:336-342`) then yields any-hit
   only for true alpha-*test* (and legitimate non-transmissive alpha-blend) materials.
4. Re-test `kitchen.scene.json`: the `glass-liquid-ice` glass should refract correctly; opacity-1.0
   glass should be unchanged.

### Related defects / follow-up (masked in this scene)

- The plain-glTF material path (no `.material.json` sidecar) now sets `kMaterialFlag_ThinSurface`
  for transmission without `KHR_materials_volume`, but still does not parse `KHR_materials_ior`.
  IoR is hardcoded to `1.5` (`src/RTXPTMaterials.cpp`; loader has no `KHR_materials_ior`
  parsing). Harmless for scenes with sidecar material files, but wrong for scenes that rely on
  glTF-only custom IoR.

## Key references

- Port material classification/upload: `src/RTXPTMaterials.cpp`
- BLAS opaque-flag selection: `src/RTXPTAccelerationStructures.cpp:379-381`
- Any-hit: `shaders/PathTracer/PathTracerAnyHit.rahit`
- Closest-hit / transmission BSDF: `shaders/PathTracer/PathTracerClosestHit.rchit`,
  `shaders/PathTracer/Rendering/Materials/BxDF.hlsli`,
  `shaders/PathTracer/Rendering/Materials/MaterialBridge.hlsli`
- Nested dielectrics: `shaders/PathTracer/PathTracerNestedDielectrics.hlsli`
- Upstream baseline: `D:/RTXPT-fork` (`PathTracerBridgeDonut.hlsli`,
  `PathTracerMaterialSpecializations.hlsl`, `Materials/MaterialsBaker.cpp`, `BxDF.hlsli`)
- Port↔fork mapping: `RTXPT_FORK_MAPPING.md`
