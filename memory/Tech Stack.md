---
title: Tech Stack
type: note
permalink: diligentengine-hzqst/tech-stack
---

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

- Project knowledge notes are stored under the git-tracked `memory/` directory, served by the `basic-memory` MCP server configured in `.mcp.json` (project `diligentengine-hzqst`).
- `basic-memory` is available through `uvx basic-memory`; when the MCP server is not attached, notes are maintained with `tool write-note` / `tool edit-note` / `tool search-notes`.
