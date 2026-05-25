# Project Overview

DiligentEngine-hzqst is a local checkout of the Diligent Engine super-repository.
Diligent Engine is a modern cross-platform low-level 3D graphics library and rendering framework.
It provides a consistent front-end API over Direct3D11, Direct3D12, OpenGL/OpenGLES, Vulkan, WebGPU, and commercial Metal support.

The repository is organized as a CMake super-project with four main submodules:

- `DiligentCore`: foundation module and low-level graphics API abstraction. It includes rendering backend implementations, common interfaces, platform utilities, shader tools, and core tests.
- `DiligentTools`: libraries and tools built on top of Core, including texture loading, asset loading, Dear ImGui integration, native app support, HLSL-to-GLSL conversion, Render State Notation, and render state packaging.
- `DiligentFX`: high-level rendering framework and components such as PBR, Hydrogent, shadows, and post-processing effects.
- `DiligentSamples`: tutorials, sample applications, and demos that exercise Core, Tools, and FX.

The root `CMakeLists.txt` always adds `DiligentCore`, then conditionally adds `DiligentTools`, `DiligentFX`, `DiligentSamples`, `Doc`, optional `DiligentCorePro`, and optional `DiligentCommunity`.

The local root contains a `.serena/project.yml` configured for `hlsl`, `cpp`, `bash`, and `python`.
