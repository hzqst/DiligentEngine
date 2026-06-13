# Transmissive Materials Render Incorrectly (Glass / Transparency)

Status: **root cause identified, fix not yet applied**
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
   `RTXPTMaterialIsAlphaBlended()` returns `true` for *any* transmission
   (`EnableTransmission || TransmissionFactor > 0 || DiffuseTransmissionFactor > 0`):
   `src/RTXPTMaterials.cpp:327-334`.
   It is additionally forced because the **DiligentFX glTF loader sets `ALPHA_MODE_BLEND` on every
   material carrying `KHR_materials_transmission`** (`DiligentTools/AssetLoader/src/GLTFLoader.cpp`,
   ~line 1870). Either source sets `kMaterialFlag_AlphaBlend` during upload:
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

## Out of scope (not caused by this bug)

- **Opacity-1.0 transmissive materials are unaffected** by the alpha-blend defect, because the
  any-hit always accepts (`rand > 1.0` is never true). This covers `kitchen.Glass` (the wine
  glasses, meshes `Mesh_14` / `Mesh_185` / `Mesh_14.001`), the `transparent-machines` "HOME"
  sculpture, and `GlassMicrowave`. Their closest-hit transmission path matches upstream.
- **The kitchen's big window has no glass pane.** Its meshes (`Window_Frame`, `Window_Blinds`,
  `Window_Sill`, `Window_Panels`, …) use opaque wood/plastic/metal materials; the opening shows
  the environment map directly. A blown-out/"opaque" window and darker-looking wine glasses are
  most consistent with **auto-exposure / tone-mapping** (the port screenshot uses auto-exposure
  with EV −4.0, which clips the bright exterior and crushes interior midtones), not transmission.
  Track this separately from the transparency fix.

## Fix direction

Decouple *transmission* from *alpha-blend* so transmissive dielectrics stay opaque geometry and
refract via the closest-hit BSDF, exactly as upstream:

1. **`RTXPTMaterialIsAlphaBlended()` (`src/RTXPTMaterials.cpp:327-334`)** must not return `true`
   purely because a material has transmission. Reserve `kMaterialFlag_AlphaBlend` for genuine
   alpha-blended, *non-transmissive* surfaces (e.g. decals / foliage cards with `<1` opacity and no
   transmission).
2. **Do not let the loader-forced `ALPHA_MODE_BLEND` (for `KHR_materials_transmission`) feed the
   alpha-blend flag.** When `kMaterialFlag_HasTransmission` is set, the material should build as
   opaque geometry (`RAYTRACING_GEOMETRY_FLAG_OPAQUE`) and not request the stochastic any-hit.
3. Confirm `RTXPTMaterialNeedsAnyHit()` (`src/RTXPTMaterials.cpp:336-342`) then yields any-hit
   only for true alpha-*test* (and legitimate non-transmissive alpha-blend) materials.
4. Re-test `kitchen.scene.json`: the `glass-liquid-ice` glass should refract correctly; opacity-1.0
   glass should be unchanged.

### Related defects to address opportunistically (masked in this scene)

- The plain-glTF material path (no `.material.json` sidecar) never sets `kMaterialFlag_ThinSurface`
  and never parses `KHR_materials_ior` — IoR is hardcoded to `1.5`
  (`src/RTXPTMaterials.cpp:111`; loader has no `KHR_materials_ior` parsing). Harmless for scenes
  with sidecar material files, but wrong for scenes that rely on glTF-only transmission.

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
