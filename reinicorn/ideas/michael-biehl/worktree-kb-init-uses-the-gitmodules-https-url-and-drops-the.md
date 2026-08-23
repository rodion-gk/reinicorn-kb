---
type: idea
title: Worktree kb init uses the .gitmodules HTTPS URL and drops the main checkout SSH
slug: worktree-kb-init-uses-the-gitmodules-https-url-and-drops-the
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
---

# Worktree kb init uses the .gitmodules HTTPS URL and drops the main checkout SSH 

## Description

Worktree kb init uses the .gitmodules HTTPS URL and drops the main checkout SSH remote, so rcorn kb publish fails in every worktree

## Notes

Hit while setting up a worktree for `feat/unified-kb-doc-frontmatter`
(2026-07-27). Every `rcorn kb publish` from the worktree failed.

### Root cause

`.gitmodules` declares the kb remote as **HTTPS**:

```
url = https://github.com/crystldm/reinicorn-kb.git
```

But the main checkout's kb has a local override to **SSH** — from
`.git/modules/kb/config`:

```
[remote "origin"]
	url = git@github.com:crystldm/reinicorn-kb.git
```

`gh auth status` reports "Git operations protocol: ssh", and there is **no**
credential helper configured at global or system scope. So HTTPS pushes have no
credentials at all.

When the post-checkout hook initializes `kb/` in a new worktree, it clones from
the `.gitmodules` URL and the local SSH override is not carried over. Result:

```
fatal: could not read Username for 'https://github.com': No such device or address
```

Reads work (the hook borrows objects via `--reference`, no network), so the
breakage only appears at publish time — after work has been done.

### Workaround

```
rcorn kb git remote set-url origin git@github.com:crystldm/reinicorn-kb.git
```

### Fix directions

- Have worktree kb init copy `remote.origin.url` from the main checkout's kb
  config rather than re-reading `.gitmodules`, so any local override (SSH,
  mirror, internal host) follows into worktrees.
- Or resolve the remote through `REINICORN_KB_REMOTE` in `.reinicorn-config`,
  which already records the URL and is the direction
  [[remove-the-kb-submodule]] is heading anyway.
- Either way, worth a check at publish time that fails with "kb remote is
  unauthenticated" instead of the current misdiagnosis — see
  [[rcorn-kb-publish-reports-any-post-retry-push-failure-as-conf]].

Note this becomes moot in its current form if the kb stops being a submodule,
but the underlying issue — worktree kb init not inheriting local remote config —
survives that change.
