---
type: plan
title: 'Execution Plan: docs/scope-kb-git-guideline'
slug: docs-scope-kb-git-guideline
lifecycle: active
status: in-progress
created: 2026-08-04
author: Michael Biehl
branch: docs/scope-kb-git-guideline
ticket: N/A
spec: N/A
---

# Execution Plan: docs/scope-kb-git-guideline

## Goal
Scope the AGENTS.md "use rcorn, never raw Git on the KB" guideline so it
clearly applies to agents and contributors operating *on* the KB — not to
`src/reinicorn/`, which implements the rcorn interface and necessarily uses
git internally. CodeRabbit (PR #36) read the current wording literally and
flagged `commit_kb`'s `run_git` calls as a guideline violation; any other
agent could make the same misreading.

## Acceptance Criteria
- [ ] AGENTS.md guideline states the implementation exemption explicitly
- [ ] No behavior/code changes; docs-only diff

## Approach
Add one clarifying sentence to the Knowledge base section of AGENTS.md.
Chosen over a `.coderabbit.yaml` path instruction or a CodeRabbit learning
because the fix in the shared guideline works for every reader, not just
CodeRabbit.

## Tasks
- [ ] Amend the guideline sentence in AGENTS.md
- [ ] PR

## Dependencies
Motivated by the PR #36 review; no code dependency on it.
