---
type: idea
title: 'Extend next: beyond the CLI: git hooks and dynamic task handoff. (1) Git hooks ('
slug: 2026-07-04-extend-next-beyond-the-cli-git-hooks-and-dynamic-task-handof
lifecycle: active
status: new
created: 2026-07-04
author: Michael Biehl
---

# Extend next: beyond the CLI: git hooks and dynamic task handoff. (1) Git hooks (

## Description

Extend next: beyond the CLI: git hooks and dynamic task handoff. (1) Git hooks (post-merge, pre-push, session-start) emit output too — they should follow the same next: footer convention so hook output guides the agent like command output does. (2) Dynamic next: content driven by work state: the active plan's checkboxes already live in the kb, so surfaces like reins home, kb status --compact, and plan show could emit the first unchecked task as next: — making the plan a live handoff document. Any agent picking up the branch gets 'you are here, do this next' injected automatically at session start, instead of re-deriving state from progress.md.

## Notes

Footer-kind split (decided 2026-07-04): prose task content must
NOT reuse `next:`. Keep `next:` meaning exactly "runnable command, copy-paste
safe" (as shipped: enforcement tests, golden principle #12, skills), and add
`next-task:` for prose work items (e.g. the first unchecked plan checkbox).

Why not rename to `next-cmd:`: `next:` is already unambiguous today — the
ambiguity only arrives with the new prose kind, so only the new kind needs a
new label. Renaming would churn every command, test, and doc for zero
information gain. The shared `next` prefix keeps both greppable via `^next`.

Rationale for the split: for agents the cost difference between copy-pasting
and following an instruction is negligible — the difference is error rate.
`next:` is a zero-interpretation channel; prose inserts an intent→action
translation step where context-poor agents introduce variance, and agents
trained by our own convention to execute whatever follows `next:` could try
to shell-execute a sentence. Plan checkboxes written under the writing-plans
discipline (complete code, exact commands) make `next-task:` lines unusually
high-fidelity instructions.
