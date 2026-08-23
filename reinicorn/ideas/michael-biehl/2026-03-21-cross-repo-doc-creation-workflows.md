---
type: idea
title: cross-repo doc creation workflows
slug: 2026-03-21-cross-repo-doc-creation-workflows
lifecycle: active
status: new
created: 2026-03-21
author: Michael Biehl
---

# cross-repo doc creation workflows

## Description

When working in tightly coupled multi-repo setups (e.g., an app repo + a sibling library repo), docs created via `reins <type> create` land in the current repo's harness by default. This is wrong when the doc belongs to a sibling project — e.g., designing a library feature while working in the app repo.

## Proposed Changes

1. **CLI flag**: `reins <type> create --repo <target>` to place a doc in a different reins-managed repo's harness
2. **Skill guidance**: A few lines in relevant skills reminding agents to confirm which repo a doc belongs to when projects are coupled
3. **Sibling repo discovery**: Reins could maintain a lightweight registry of related repos (config or auto-discovered via shared parent dir) to offer as targets

## Origin

Real workflow pain — wrote a library design doc while working in the app repo; the agent correctly used `reins <type> create` but it landed in the app repo's harness instead of the library's.
