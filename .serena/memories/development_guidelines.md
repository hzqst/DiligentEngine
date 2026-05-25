# Development Guidelines

General workflow:

- Start from Serena memories when available, then use targeted repository reads or symbol lookup only for the relevant area.
- Avoid full repository scans when a module-specific or symbol-specific lookup is enough.
- Treat `ThirdParty`, generated build outputs, media assets, and large shader test trees as expensive context.
- For architecture, call-chain, data-flow, entry-point, and dependency questions, prefer semantic/symbol search when available; use `rg` for exact string enumeration.

Local safety and delivery rules:

- Do not run build or test commands unless explicitly requested by the user, or unless the current task explicitly requires verification and running them is allowed.
- Do not use destructive commands such as `git reset --hard` or unsafe deletion.
- Do not modify `.git` with non-Git tools.
- Do not hard-code secrets or credentials.
- Use parameterized database queries and avoid shell/SQL construction from untrusted input.
- Do not terminate processes not started for the current task unless explicitly requested.

Task handling:

- For small scoped edits, use the shortest path that preserves quality.
- For unclear or higher-risk behavior changes, clarify goals, boundaries, risk, and verification first.
- Escalate planning/review rigor when changes touch shared APIs, schemas, contracts, public types, persistence, concurrency, or cross-module behavior.
- Keep source edits focused; avoid unrelated refactors and metadata churn.

Serena notes:

- The project expects `activate_project` at startup when Serena MCP tools are available.
- In this Codex session, Serena CLI exists but Serena MCP tools are not directly exposed, so onboarding memories may be maintained as files under `.serena/memories`.
- `serena project health-check` timed out in this checkout during onboarding; logs showed HLSL language server/request issues and an encoding issue under `DiligentCore/ThirdParty/glslang/Test`.
