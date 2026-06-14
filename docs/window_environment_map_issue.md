# Window Renders Opaque/White Instead of Showing the Environment Map

Status: **fix implemented (needs build + visual verification) — high-resolution environment cube bake
added to `RTXPTEnvMapBaker`; awaiting confirmation that the kitchen window shows the ocean envmap**
Scope: distant environment map sampled by the path tracer (background + environment NEE). Independent of
the transmissive-material work in [transmissive_materials_issue.md](transmissive_materials_issue.md).
Repro scene: `assets/kitchen.scene.json` (env map `EnvironmentMaps/simons_town_rocks_4k_cube_bc6u.dds`).

## Symptom

Looking through the kitchen window from inside, the port renders a flat, washed-out light/white area
instead of the ocean visible through the same window in upstream RTXPT-fork. Crucially:

- From **outside the room**, the ocean environment map renders correctly in the port — so the env map
  loads and is sampled.
- The window is **geometrically open** to the environment, so this is purely how the env map is sampled,
  not a glass/occlusion problem.

## What it is *not* (ruled out by geometry inspection)

The kitchen window has **no glass pane**. Verified from `Models/Kitchen/kitchen.gltf` (scene + glTF are
byte-identical between the port and `D:/RTXPT-fork`):

- `Window_Panels` (node 374) is the **muntin grid**, not a sheet: 488 triangles, ~12 % areal coverage,
  vertical bars with gaps. Material is opaque `Trim_Plastic_MDL` — identical in both renderers.
- The back wall (`Wall_Back_Right`) has a **clean window opening** (0 % wall coverage inside the window
  rectangle at the back plane).
- No transmissive material exists anywhere in the window frame depth.

So rays through the muntin gaps + wall opening miss all geometry and sample the environment map directly.

## Root cause

**The port feeds the path tracer DiligentFX's GGX-prefiltered IBL cube as the environment map, instead of
a high-resolution environment cube.** The prefiltered cube is only `PBR_Renderer::PrefilteredEnvMapDim =
256`² per face, whereas upstream bakes a **2048²** cube.

- Port: `RTXPTEnvMapBaker::PrecomputeCubemap` set `m_EnvironmentMapSRV =
  m_IBLPrecompute->GetPrefilteredEnvMapSRV()` (256²), and the importance/NEE map was built from it too
  (`src/RTXPTEnvMapBaker.cpp`). The path tracer binds this SRV to `t_EnvironmentMap`
  (`src/RTXPTSample.cpp:1284` → `RTXPTRayTracingPass`), sampled at LOD 0 for the background
  (`shaders/PathTracer/PathTracer.hlsli` `HandleMiss`).
- Upstream: `Rtxpt/Lighting/Distant/EnvMapBaker.cpp:374-375` bakes a `2048` (real env) / `1024`
  (procedural sky) cube and samples *that* for the background.

### Why the window blows out but the full-screen view does not

Downsampling the 4096² source to 256² **averages the very bright sky into the horizon/ocean texels**. The
window faces the bright sky-meets-sea horizon (the scene's main backlight), so those few low-resolution
texels wash out to flat light. Filling the *whole* screen (the outside view) the 256² cube still reads as
recognizable ocean because most of what you see is away from that bright averaged band. At 2048² the sky
and ocean stay separated at the horizon, matching upstream.

The GGX-prefiltered cube is also semantically wrong here: it is an IBL convolution product meant for rough
specular reflections, not the literal distant environment. Its irradiance and BRDF-LUT siblings are not
used by the RTXPT path tracer at all (only `GetEnvironmentMapSRV()` is bound).

## Fix

Bake a high-resolution environment cube from the loaded source and bind it as `t_EnvironmentMap` (and as
the importance-map source), mirroring upstream's dedicated `EnvMapBaker`.

- New shader stages in `shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl`:
  - `EnvCubeBakeVS` — replicates DiligentFX `CubemapFace.vsh` (full-screen quad → per-face world
    direction via a per-face rotation in `cbEnvCubeBake`).
  - `SampleEnvToCubePS` — samples the source at that direction. `ENV_BAKE_SOURCE_CUBE` selects a cube
    source vs a 2D equirectangular source (the latter uses the same `TransformDirectionToSphereMapUV`
    mapping as DiligentFX, so orientation is preserved).
- `RTXPTEnvMapBaker::BakeHighResEnvironmentCube` (`src/RTXPTEnvMapBaker.cpp`): renders the 6 faces of a
  **2048² RGBA16F** cube with a full mip chain (`MISC_TEXTURE_FLAG_GENERATE_MIPS` + `GenerateMips`) using
  the **same six face rotations** as `PBR_Renderer::PrecomputeCubemaps`, so the baked cube keeps the exact
  orientation the path tracer already expects. `m_EnvironmentMapSRV` then points at this cube.
- The DiligentFX `PrecomputeCubemaps` call is kept only for the (unused-by-RT but readiness-checked)
  irradiance / BRDF-LUT products; if the high-res bake fails the renderer falls back to the prefiltered
  cube so it still draws.
- Resolution is fixed at 2048² for every source (rather than upstream's 2048/1024 split) so the cube
  texture — and therefore the `STATIC` `t_EnvironmentMap` binding — is created once and reused across
  re-bakes (runtime environment swaps re-render its contents in place rather than allocating a new SRV).

### Files

- `DiligentSamples/Samples/RTXPT/shaders/PathTracer/Lighting/EnvMapImportanceBaker.hlsl`
- `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.hpp`
- `DiligentSamples/Samples/RTXPT/src/RTXPTEnvMapBaker.cpp`

### Verification (pending)

Build and open `kitchen.scene.json`; the window should show the ocean environment (matching upstream)
rather than a flat white/light area, and the env-map baker info should report a 2048² cube.

## Follow-ups / notes

- Upstream additionally extracts directional lights, applies optional BC6H compression, and uses a
  low-resolution pre-pass; none of those are reproduced here — only the resolution defect that caused the
  visible artifact is addressed.
- The earlier hypothesis that the bright window was purely an auto-exposure / tone-mapping artifact (noted
  in [transmissive_materials_issue.md](transmissive_materials_issue.md) "Out of scope") was incorrect for
  the through-window content; the dominant cause is the environment-cube resolution above. Auto-exposure
  may still affect the very brightest exterior highlights.
