---
type: idea
title: Centralize gh CLI usage into a shared reins.github module with standard invo
slug: 2026-02-20-centralize-gh-cli-usage-into-a-shared-reins-github-modul
lifecycle: done
status: resolved
created: 2026-02-20
author: mnbiehl
---

# Centralize gh CLI usage into a shared reins.github module with standard invo

**Resolved:** 2026-05-05

## Description

Centralize gh CLI usage into a shared reins.github module with standard invocation, retry, auth checking, and consistent error handling. Currently feedback.py calls gh directly via subprocess — this should be extracted before adding more gh-dependent commands.

## Resolution

`src/reins/github.py` now wraps the `gh` CLI with `gh_available()` and
`run_gh()` (centralized auth/error handling). Added during the M2
init-rework + repo creation milestone.
