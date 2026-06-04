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

Do not run test or runtime smoke commands unless the user explicitly asks for them.
