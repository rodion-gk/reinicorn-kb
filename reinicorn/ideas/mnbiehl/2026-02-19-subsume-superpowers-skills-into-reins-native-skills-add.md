---
type: idea
title: Subsume superpowers skills into reins native skills. Add guardrail (agent ho
slug: 2026-02-19-subsume-superpowers-skills-into-reins-native-skills-add
lifecycle: done
status: resolved
created: 2026-02-19
author: mnbiehl
---

# Subsume superpowers skills into reins native skills. Add guardrail (agent ho

**Resolved:** 2026-05-05

## Description

Subsume superpowers skills into reins native skills. Add guardrail (agent hook or lint rule) that prevents docs from being placed in docs/plans/ instead of harness/. Resolve conflict between superpowers brainstorming/writing-plans hardcoded paths and harness convention.

## Resolution

Landed via the `skills-fork-template-docs` exec plan and the M3+M4 skills
unification pass. Superpowers skills were forked into `.claude/skills/`,
patched to use kb paths, and the `update-superpowers` skill keeps the
forks in sync. `editor-hooks/enforce-doc-templates.sh` blocks raw doc
writes in protected kb paths.

- Plan: `kb/reins/exec-plans/completed/skills-fork-template-docs/`
- Design: `kb/reins/design-docs/skills-fork-and-template-docs.md`
