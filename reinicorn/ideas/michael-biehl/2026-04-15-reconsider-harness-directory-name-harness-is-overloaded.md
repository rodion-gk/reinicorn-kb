---
type: idea
title: 'Reconsider ''harness/'' directory name. ''Harness'' is overloaded: OpenAI''s ''Harness'
slug: 2026-04-15-reconsider-harness-directory-name-harness-is-overloaded
lifecycle: done
status: resolved
created: 2026-04-15
author: Michael Biehl
---

# Reconsider 'harness/' directory name. 'Harness' is overloaded: OpenAI's 'Harness

**Resolved:** 2026-04-16

## Description

Reconsider 'harness/' directory name. 'Harness' is overloaded: OpenAI's 'Harness Engineering' uses the term for the runtime wrapping the agent (Claude Code, OpenCode), while reins uses it for the shared knowledge-base directory. Two distinct concepts, same word. Renaming is a big refactor (CLI, hooks, linters, skills, schemas), so treat as a design-doc-worthy discussion. Candidates: 'context/', 'knowledge/', 'playbook/', 'atlas/'.

## Resolution

Promoted to a design doc and implemented. Settled on **`kb/`** (knowledgebase)
with the GitHub submodule repo renamed `reins-kb`.

- Design doc: `reins/design-docs/harness-to-kb-rename.md`
- Execution plan: `reins/exec-plans/active/harness-to-kb-rename/plan.md`
- Implementation: two-phase merge (internal flip, then user-facing flip)
  landed on main on 2026-04-16.

## Notes

_No additional notes yet._
