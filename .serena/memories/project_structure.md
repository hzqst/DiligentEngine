# Project Structure

Repository root:

- `CMakeLists.txt`: top-level super-project entry point.
- `README.md`: main Diligent Engine overview and build/run instructions.
- `build-x64-Debug.bat`: local one-command Windows configure/build/install helper.
- `.clang-format`: root formatting configuration.
- `.gitmodules`: declares the four main submodules.
- `Doc`: documentation project.
- `Tests`: root-level tests or integration assets.
- `Media`: shared media assets.
- `build`, `x64`, `install`: generated build/install outputs when present.

Submodules:

- `DiligentCore`
  - `Graphics`: graphics backend implementations and graphics/shader tools.
  - `Platforms`: platform-specific layers for Win32, UWP, Linux, Android, Apple, Emscripten, and basic/common platform support.
  - `Primitives`: lower-level infrastructure and common interfaces.
  - `Common`: shared implementations.
  - `BuildTools`: CMake utilities, formatting validation, packaging support, and helper tools.
  - `Tests`: unit, API, GPU, and test framework code.
  - `doc`: core documentation.
- `DiligentTools`
  - `TextureLoader`, `AssetLoader`, `Imgui`, `NativeApp`, `HLSL2GLSLConverter`, `RenderStateNotation`, `RenderStatePackager`, and tests.
- `DiligentFX`
  - `PBR`, `Hydrogent`, `Components`, `PostProcess`, shared shaders/utilities, and tests.
- `DiligentSamples`
  - `Tutorials`, `Samples`, `SampleBase`, Android and Unity plugin examples.

Submodule URLs:

- `DiligentCore`: `https://github.com/DiligentGraphics/DiligentCore.git`
- `DiligentTools`: `https://github.com/DiligentGraphics/DiligentTools.git`
- `DiligentSamples`: `https://github.com/DiligentGraphics/DiligentSamples.git`
- `DiligentFX`: `https://github.com/DiligentGraphics/DiligentFX.git`
