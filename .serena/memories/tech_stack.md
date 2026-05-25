# Tech Stack

- Primary language: C++.
- Shader language: HLSL is the universal shading language; shader toolchains also handle GLSL, MSL, DirectX bytecode, SPIR-V, and related formats.
- Build system: CMake.
- Common Windows generator: Visual Studio 17 2022 with x64 architecture.
- Tests: GoogleTest under module-specific `Tests` directories.
- Formatting: clang-format 10.0.0 style, configured by `.clang-format` files.
- Packaging: .NET/NuGet support exists in DiligentCore build tools; .NET SDK 6.0+ may be needed for .NET packaging paths.
- Platforms/backends: Windows, UWP, Linux, Android, Apple platforms, Emscripten/Web; graphics APIs include D3D11, D3D12, OpenGL/GLES/WebGL, Vulkan, WebGPU, and commercial Metal.

Important local tooling:

- `serena.exe` is available at `C:\Users\HZDEV\.local\bin\serena.exe`.
- `.serena/cache` already contains symbol caches for bash, cpp, hlsl, and python.
- A full `serena project health-check` may be expensive in this checkout because ThirdParty and shader test files are large and some HLSL/encoding paths have caused timeouts or language server termination.
