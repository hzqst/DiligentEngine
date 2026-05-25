# Code Style Conventions

- Follow the nearest `.clang-format`; formatting is validated by CI.
- The configured style is based on Microsoft style and uses clang-format 10.0.0 expectations.
- Temporarily disable formatting only when necessary with `// clang-format off` and re-enable with `// clang-format on`.
- Update copyright dates when changing source/header files if the project convention requires it.
- Prefer existing module patterns, helper APIs, ownership boundaries, and naming conventions.
- Keep changes local and minimal unless the task explicitly requires broader refactoring.

Header include order:

1. System or standard library headers.
2. Diligent Engine interface headers.
3. Base class implementation headers.
4. Object implementation headers.
5. Other dependency headers.

Source include order:

1. Precompiled header, usually `pch.h`.
2. Corresponding header for the source file.
3. System or standard library headers.
4. Interface headers.
5. Object implementation headers.
6. Other dependency headers.

Testing conventions:

- Tests are primarily GoogleTest based.
- Prefer focused tests covering key path, boundary cases, and error paths.
- Use expected value before actual value in assertions where the local style allows it.
