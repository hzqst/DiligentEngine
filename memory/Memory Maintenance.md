---
title: Memory Maintenance
type: note
permalink: diligentengine-hzqst/memory-maintenance
---

# Memory Maintenance

## Discovery Model

- Core principle: progressive discovery through references, building a graph of notes.
- Notes live in the git-tracked `memory/` directory and are indexed by the `basic-memory` MCP server.
- Start from a high-level note and follow `[[wikilink]]` references into more specific notes; the graph depth depends on project complexity.
- Use folders to group related notes when the content structure warrants it (folders can mirror project structure or topics such as debugging and architecture).
- A reference's surrounding text should state when to read the linked note and which aspects it covers, e.g. prefer "RTXPT jitter conventions across raygen, DLSS, and TAA: [[SuperResolutionJitter]]" over a bare link.
- Notes themselves should not contain information about when to read them; this is the responsibility of the referring note.

## Style

Dense agent notes, not prose docs. Prefer invariants, terse bullets.
Avoid obvious context, rationale, and examples unless they prevent likely mistakes.
Keep guidance durable and generalizable, not task-local.

## Add/update threshold

Add or update notes only with stable, non-obvious project conventions that avoid complex rediscovery in the future.
Do not add: quick-read facts; generic language/framework knowledge; one-off task notes; volatile line-level details; behavior likely to change soon.

## Maintenance Actions

- Renaming/moving notes: use the basic-memory `move_note` tool so references are updated automatically; do not move files by hand.
- Checking for orphaned notes: `uvx basic-memory orphans --project diligentengine-hzqst`.
