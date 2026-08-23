---
type: idea
title: All harness docs must be created via reins CLI templates, never from scratch
slug: 2026-02-22-all-harness-docs-must-be-created-via-reins-cli-templates
lifecycle: done
status: resolved
created: 2026-02-22
author: Michael Biehl
---

# All harness docs must be created via reins CLI templates, never from scratch

**Resolved:** 2026-05-05

## Description

All harness docs must be created via reins CLI templates, never from scratch by an agent. The CLI should provide commands (or extend existing ones) to create docs from templates with provenance fields pre-filled (author, origin, human-validated: false, date). Agents should ONLY use these commands to create docs — never write raw markdown with hand-filled frontmatter. This prevents provenance fields from being guessed or copied incorrectly (as just happened with a git-version-compatibility doc marked human-validated: true by the agent). Templates enforce correct defaults; agents fill in the content.

## Resolution

Implemented end-to-end:

- `reins doc create` (design/plan/spec/debt/retro/idea/principle) scaffolds
  templates with `Author`, `Origin: ai-assisted`, `Human-validated: false`
  prefilled.
- `editor-hooks/enforce-doc-templates.sh` PreToolUse hook blocks raw
  Write/Edit on protected kb paths.
- AGENTS.md and `using-reins` skill instruct agents to always use the CLI.

See `src/reins/commands/doc_create.py` and
`src/reins/doc_types.py` (protected-path registry).
