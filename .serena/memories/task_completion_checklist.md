# Task Completion Checklist

Before claiming code work is complete:

- Confirm the requested behavior or analysis goal was addressed.
- Review changed files for scope control and accidental unrelated edits.
- Check formatting expectations for any touched C++/header/source files.
- For DiligentSamples formatting validation, run `BuildTools/FormatValidation/validate_format_win.bat` from `DiligentSamples/BuildTools/FormatValidation` so the script's relative paths resolve correctly. Use this after touching sample C++/header/source files when formatting validation is relevant or requested.
- Update copyright dates if source/header changes require it.
- Run only the verification that is appropriate and allowed for the task. If build/test execution was not requested or cannot be run, state that clearly.
- Report exact commands that were run and summarize relevant results; never invent command output.
- If verification is blocked, explain the blocker and reduce certainty in the completion statement.

Before commit/push/PR when requested:

- Use commit format `<type>(scope): <summary>`.
- Keep summary imperative, no trailing period, and <= 100 characters.
- Common types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`.
- Append `Co-Authored-By: GPT 5.5`.

For Serena-aware future sessions:

- Use `list_memories` first and read only memory files relevant to the task.
- Use targeted symbol/file lookup when memories are stale or insufficient.
- Update or add memory files when project knowledge changes in a way that is useful for future tasks.
