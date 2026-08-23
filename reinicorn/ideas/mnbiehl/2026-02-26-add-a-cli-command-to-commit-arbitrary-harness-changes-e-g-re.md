---
type: idea
title: Add a CLI command to commit arbitrary harness changes (e.g. 'reins harness c
slug: 2026-02-26-add-a-cli-command-to-commit-arbitrary-harness-changes-e-g-re
lifecycle: done
status: resolved
created: 2026-02-26
author: mnbiehl
---

# Add a CLI command to commit arbitrary harness changes (e.g. 'reins harness c

**Resolved:** 2026-05-05

## Description

Add a CLI command to commit arbitrary harness changes (e.g. 'reins harness commit "message"'). Currently commit_harness() is only available as a Python function called internally by doc create, plan create, etc. Editing harness files outside those commands (e.g. writing a plan manually) requires raw git submodule operations.

## Resolution

Addressed by `reins kb-git` (M5) — a passthrough that runs git commands
inside the kb submodule with reins's safeguards. Use
`reins kb-git commit -am "..."` to commit arbitrary kb edits. `reins kb publish`
then pushes the kb and parent in the right order.

- Command: `src/reins/commands/kb_git.py`
