---
type: idea
title: Add reins support for git worktrees, including proper kb submodule lifecycle man
slug: 2026-04-24-add-reins-support-for-git-worktrees-including-proper-kb-subm
lifecycle: active
status: new
created: 2026-04-24
author: Claude
---

# Add reins support for git worktrees, including proper kb submodule lifecycle man

## Description

Add reins support for git worktrees, including proper kb submodule lifecycle management per-worktree.

Context:
- post-checkout hook already fires on 'git worktree add' (git invokes it with checkout_type=1), and the existing hook does a minimal 'git submodule update --init kb' + 'checkout main' when kb/ is empty.
- No reins CLI command currently manages worktrees; users create them with raw 'git worktree add'.
- The using-git-worktrees skill recommends worktrees as a workflow, but the CLI provides no helpers.

Open questions:
1. Should reins expose a first-class 'reins worktree add/remove/list' command that wraps git and guarantees consistent kb setup (submodule init, branch, pointer staging, ticket/branch conventions)?
2. Submodules share git dirs across worktrees — is the current background 'submodule update --init' in post-checkout safe when the main worktree already has kb checked out? Need to verify no races or index corruption across linked worktrees.
3. kb commits from a secondary worktree: do commit_kb / stage_kb_pointer behave correctly when run from a linked worktree path? Does 'repo_root()' resolve to the worktree or the main checkout?
4. Cleanup: when a worktree is removed via 'git worktree remove', nothing currently tells reins. Do we need a pre-remove hook or a 'reins worktree remove' wrapper to clean up kb state / draft ideas / in-progress exec plans tied to that branch?
5. Should the post-checkout worktree-creation path suggest /create-exec-plan the same way new-branch detection does, given worktrees are typically created for a specific piece of work?

Next step: write a design doc exploring the CLI surface and the submodule-in-worktree edge cases before implementing.

## Notes

_No additional notes yet._
