---
type: idea
title: 'Bug: ''reins plan create'' ignores the name argument and always writes to'
slug: 2026-03-13-bug-reins-doc-create-plan-ignores-the-name-argument-and
lifecycle: done
status: resolved
created: 2026-03-13
author: Michael Biehl
---

# Bug: 'reins plan create' ignores the name argument and always writes to 

**Resolved:** 2026-07-24

## Description

Bug: 'reins plan create' ignores the name argument and always writes to the current branch's plan dir. Should create exec-plans/active/{name}/plan.md instead of exec-plans/active/{branch}/plan.md

## Resolution

Obsolete — fixed by the CLI restructure. `rcorn plan create` no longer
accepts a name argument at all: plans are branch-derived by design, and
argparse rejects extra arguments loudly (`rcorn plan create foo` →
"error: unrecognized arguments: foo", exit 2). Nothing is silently
ignored anymore. Verified 2026-07-24; no code change needed.
