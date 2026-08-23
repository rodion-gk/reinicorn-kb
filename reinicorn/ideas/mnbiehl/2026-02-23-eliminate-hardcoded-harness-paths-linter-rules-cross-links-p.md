---
type: idea
title: 'eliminate-hardcoded-harness-paths: linter rules (cross_links, plan_structure, do'
slug: 2026-02-23-eliminate-hardcoded-harness-paths-linter-rules-cross-links-p
lifecycle: done
status: resolved
created: 2026-02-23
author: mnbiehl
---

# eliminate-hardcoded-harness-paths: linter rules (cross_links, plan_structure, do

**Resolved:** 2026-05-05

## Description

eliminate-hardcoded-harness-paths: linter rules (cross_links, plan_structure, docs_freshness), bash hooks (post-checkout, atlassian variant), and CI workflow (doc-gardening.yml) all hardcode harness/ instead of reading from .gitmodules via get_harness_dir(). Refactor to use a shared helper for custom submodule path support.

## Resolution

Solved by the harness→kb rename (Phase 1 introduced the `KB_DIR_NAME`
indirection). All three linter rules now import `KB_DIR_NAME` from
`reins.config`; hooks/scripts/extensions no longer reference a literal
`harness` path.

- Design: `kb/reins/design-docs/harness-to-kb-rename.md`
- Plan: `kb/reins/exec-plans/active/harness-to-kb-rename/`
- Code: `src/reins/linter/rules/{cross_links,docs_freshness,plan_structure}.py`
