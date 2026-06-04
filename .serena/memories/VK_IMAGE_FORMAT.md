# VK_IMAGE_FORMAT HLSL 编译约定

触发信号：RTXPT / Diligent HLSL shader 中使用 `VK_IMAGE_FORMAT("...")` annotation，尤其是在手动用 `dxc.exe` 做裸编译验证时看到 `expected parameter declarator` 等宏未定义错误。

约定：不要在项目 shader 文件里手动添加 `#ifndef VK_IMAGE_FORMAT` / `#define VK_IMAGE_FORMAT(format)` fallback。Diligent 的 HLSL 编译流程会通过 `DiligentCore/Graphics/ShaderTools/include/HLSLDefinitions.fxh` 自动提供 `VK_IMAGE_FORMAT` 定义。

正确做法：
- 保持 shader 中的 `VK_IMAGE_FORMAT("...")` annotation 原样。
- 若需要用裸 `dxc.exe` 做临时验证，应在验证命令或临时编译环境中补充等价宏/包含 Diligent 编译定义，而不是修改 shader 源文件。
- 遇到相关编译错误时，先检查是否绕过了 Diligent shader compile pipeline。

适用范围：DiligentCore / DiligentSamples 中通过 Diligent shader tooling 编译的 HLSL、HLSLI、CSH、RGEN/RMISS/RCHIT/RAHIT 等 shader 文件。