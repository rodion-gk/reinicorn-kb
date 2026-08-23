---
type: idea
title: 'Post-submodule-removal follow-ups from the branch review: (1) update --diff trig'
slug: post-submodule-removal-follow-ups-from-the-branch-review-1-u
lifecycle: active
status: new
created: 2026-08-14
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Post-submodule-removal follow-ups from the branch review: (1) update --diff trig

## Description

Post-submodule-removal follow-ups from the branch review: (1) update --diff triggers migration (move detection below diff handling); (2) double fetch in publish/sync paths (ensure_kb_on_main + push retry both fetch); (3) untested failure paths: pre-push fail-open without origin/main ref, spec-gate empty-clone guard, migration clone-failure recovery message; (4) migration rmtree of a .git-less non-empty kb/ is unguarded; (5) spec_gate redundant .git check; post_checkout private _main_checkout_root import + redundant resolve; test_git local _git_init duplicate + dead superproject mock branch; test_init_slug_override could pin REINICORN_KB_REMOTE; test_assets stale submodule docstring; post_checkout test module docstring says 'submodule init'; C1 refusal message could name .reinicorn-config as the where

## Notes

_No additional notes yet._
