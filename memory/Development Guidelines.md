---
title: Development Guidelines
type: note
permalink: diligentengine-hzqst/development-guidelines
---

# Development Guidelines

General workflow:

- Start from Basic Memory notes when available, then use targeted repository reads or symbol lookup only for the relevant area.
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

Memory notes:

- Project knowledge lives in the git-tracked `memory/` directory, served by the `basic-memory` MCP server configured in `.mcp.json` (project `diligentengine-hzqst`).
- Discover notes with `search_notes` and read only what the task requires with `read_note`; persist changes with `write_note`/`edit_note`/`delete_note`.
- When the basic-memory MCP server is not attached, the same operations are available via `uvx basic-memory tool ...`.
- Do not hand-edit note files; frontmatter (`title`/`type`/`permalink`) is tool-managed.
