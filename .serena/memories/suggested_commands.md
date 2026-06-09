# Suggested Commands

Run commands from the repository root unless noted otherwise.

Source setup:

```powershell
git submodule update --init --recursive
```

One-command Windows Debug build/install:

```powershell
.\build-x64-Debug.bat
```

Manual Windows configure:

```powershell
cmake -S . -B build\x64\Debug -G "Visual Studio 17 2022" -A x64 -DCMAKE_INSTALL_PREFIX=install\x64\Debug -DDILIGENT_BUILD_FX=TRUE -DDILIGENT_BUILD_SAMPLES=TRUE -DDILIGENT_BUILD_TOOLS=TRUE -DDILIGENT_NO_WEBGPU=TRUE -DDILIGENT_NO_ARCHIVER=FALSE -DDILIGENT_BUILD_TESTS=TRUE -DDILIGENT_DEVELOPMENT=TRUE -DDILIGENT_NO_FORMAT_VALIDATION=FALSE -DDILIGENT_USE_SPIRV_TOOLCHAIN=TRUE
```

Manual Windows build/install:

```powershell
cmake --build build\x64\Debug --config Debug --target install
```

Format validation:

```powershell
cd DiligentCore\BuildTools\FormatValidation
.\validate_format_win.bat
```

Alternative format validation target:

```powershell
cmake --build build\x64\Debug --target DiligentCore-ValidateFormatting
```

Targeted GoogleTest examples:

```powershell
cd build\x64\Debug\Tests\DiligentCoreAPITest\Debug
.\DiligentCoreAPITest.exe --mode=gl --gtest_filter="DrawCommandTest*:InlineConstants*"
.\DiligentCoreAPITest.exe --mode=gl --gtest_filter="DrawCommandTest*:InlineConstants*" --non_separable_progs
.\DiligentCoreAPITest.exe --mode=vk --gtest_filter="DrawCommandTest*:InlineConstants*"
```

.NET/NuGet packaging from `DiligentCore`:

```powershell
python -m pip install -r .\BuildTools\.NET\requirements.txt
python .\BuildTools\.NET\dotnet-build-package.py -c Debug -d .\
```

Serena inspection:

```powershell
serena tools list
serena project index
serena project health-check
```

RTXPT closest-hit shader smoke:

Use this when debugging RTXPT ray tracing shaders. Compile through the repo DXC 1.10.2605.24 path with `-Zi -Qembed_debug -Zpr`.

DO NOT add `-Od`. Compiling with `-Od` (disable optimizations) makes DXC emit incorrect DXIL for `PathTracer::HandleHit` (consistent with `-Od` DXIL mishandling nested `inout` parameters). This was the confirmed root cause of the realtime/reference BxDF rendering regression — with `-Od` the glass/transmission and opaque paths render black; with optimizations enabled, reference mode matches upstream RTXPT exactly. Build and smoke-test the RTXPT shaders with optimizations enabled. See `docs/realtime_bxdf_debugging.md` and `docs/realtime_bxdf_diff.md` (2026-06-09 root-cause entries).

```powershell
cd DiligentSamples\Samples\RTXPT\assets
& "D:\DiligentEngine-hzqst\build\tools\dxc\v1.10.2605.24-preview\bin\x64\dxc.exe" -T lib_6_5 -E main -Zi -Qembed_debug -Zpr -D PATH_TRACER_MODE=<0|1|2> -D ENABLE_MATERIAL_TEXTURES=1 -D MATERIAL_TEXTURE_COUNT=1024 -D RTXPT_ENABLE_LOW_DISCREPANCY_SAMPLER_FOR_BSDF=1 -D DXCOMPILER=1 -D "VK_IMAGE_FORMAT(x)=" -I shaders -I shaders\PathTracer shaders\PathTracer\PathTracerClosestHit.rchit -Fo NUL
```

Run all three `PATH_TRACER_MODE` values when checking `PathTracerClosestHit.rchit`: `0` Reference, `1` BuildStablePlanes, `2` FillStablePlanes.

Do not run test or runtime smoke commands unless the user explicitly asks for them.
