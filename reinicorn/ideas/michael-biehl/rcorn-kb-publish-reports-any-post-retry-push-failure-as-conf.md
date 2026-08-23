---
type: idea
title: rcorn kb publish reports any post-retry push failure as conflicting changes, mas
slug: rcorn-kb-publish-reports-any-post-retry-push-failure-as-conf
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
---

# rcorn kb publish reports any post-retry push failure as conflicting changes, mas

## Description

rcorn kb publish reports any post-retry push failure as conflicting changes, masking auth errors and other real causes

## Notes

Surfaced alongside
[[worktree-kb-init-uses-the-gitmodules-https-url-and-drops-the]]
(2026-07-27).

### What happened

`rcorn kb publish` reported:

```
Publishing kb changes...
Push failed, pulling latest and retrying...
error: Publish failed — kb has conflicting changes. Resolve any conflicts in kb/, then retry.
next: rcorn kb publish
```

There were no conflicting changes. `kb git status` said *"working tree clean,
ahead of 'origin/main' by 2 commits"* and
`rev-list --left-right --count origin/main...main` returned `0  2` — a clean
fast-forward. The actual cause was authentication:

```
fatal: could not read Username for 'https://github.com': No such device or address
```

The message sent me to resolve conflicts that did not exist, and the suggested
next step (`rcorn kb publish`) was the command that had just failed — so
following the guidance verbatim loops forever.

### Why it matters

The misdiagnosis costs more than the bug. An agent following the error text
will start "resolving conflicts" in a clean tree, and the retry hint guarantees
a loop. This is the failure mode `block-raw-kb-git.sh` exists to prevent —
surfacing CLI failures rather than letting agents work around them — undone by
an error message that describes the wrong failure.

### Fix direction

`push_main_with_retry` (`kb.py:131`) collapses every non-zero exit after the
pull-and-retry into one "conflicting changes" message. It should branch on
git's stderr:

- `could not read Username` / `Permission denied (publickey)` / `Authentication
  failed` → "kb remote is unauthenticated" + the remote URL and protocol.
- `non-fast-forward` / `fetch first` → the current conflict message.
- `protected branch` / `GH006` → "kb main is protected; the review lane owns
  this path".
- anything else → surface git's stderr verbatim rather than guessing.

Cheap general rule: when the CLI cannot classify a git failure, print git's
stderr instead of substituting a guess.
