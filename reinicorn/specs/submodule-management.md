---
type: spec
title: Submodule Management Redesign
slug: submodule-management
lifecycle: done
status: implemented
created: 2026-02-17
author: Michael + Claude
---

# Submodule Management Redesign

## Problem

The harness submodule is docs and plans, not code. It should always track latest `main` — reproducibility doesn't matter. But the current setup fights this:

1. **Detached HEAD.** `git submodule update` (called by post-checkout and post-merge hooks) resets the submodule to a pinned commit on detached HEAD. This clobbers in-progress work and makes local commits awkward.
2. **No auto-commit.** Commands like `plan create`, `idea`, and `plan complete` write files but don't consistently commit inside the submodule. Some commit in the parent repo (wrong place), some don't commit at all.
3. **No ownership boundary.** Nothing stops agents from running raw `git` commands against the submodule, bypassing reins and causing state confusion.
4. **Submodule pointer noise.** `git status` shows the harness as modified whenever the submodule HEAD differs from the recorded pointer, creating irrelevant noise.

## Design

### Git Configuration

Add `branch` and `ignore` to `.gitmodules`:

```ini
[submodule "harness"]
    path = harness
    url = git@github.com:your-org/your-kb.git
    branch = main
    ignore = all
```

- `branch = main` tells `git submodule update --remote` to track the `main` branch.
- `ignore = all` suppresses all submodule diff noise in `git status`, `git diff`, and `git commit`. The parent repo stops caring about the pointer.

The parent repo's submodule pointer becomes cosmetic — updated occasionally so GitHub's web UI links to a recent harness commit, but never critical.

### Hook Changes

**post-checkout:** Remove the `git submodule update --init --recursive -- harness` block (both the bash hook and the Python `cmd_post_checkout`). Replace with a lightweight check: if `harness/` is empty (fresh clone or new worktree), run `git submodule update --init harness` followed by `cd harness && git checkout main` to land on a branch, not detached HEAD.

**post-merge:** Remove the `git submodule update` call. The harness is already on `main` locally; a merge in the parent repo doesn't need to touch it. Keep the stale plan archival logic.

**pre-push:** No changes — it only reads harness files.

### Auto-commit Helper

A shared utility `reins.harness.commit_harness(root, message)` that:

1. Checks if harness submodule is configured via `get_harness_dir()` (no-op if None).
2. Ensures the submodule is on `main` branch — if detached HEAD, checks out `main`.
3. Stages all changes inside `harness/` with `git add -A`.
4. Commits with the provided message.
5. Returns whether a commit was actually made.

Every harness-modifying command calls this after writing files. No push — just local commit.

### Command Changes

| Command | Current behavior | New behavior |
|---------|-----------------|-------------|
| `idea` | `git add` + `git commit` in parent repo | Write file, then `commit_harness(root, "idea: <slug>")` |
| `plan create` | No commit | Write files, then `commit_harness(root, "plan: create <branch>")` |
| `plan complete` | No commit | Move files, then `commit_harness(root, "plan: complete <branch>")` |
| `agent run` | No commit | Write job metadata, then `commit_harness(root, "agent: dispatch <task>")` |
| `sync` | `git submodule update --remote` (detaches HEAD) | `git fetch origin main` + `git merge origin/main` inside submodule (stays on branch) |
| `publish` | Commits in parent + rebases/pushes submodule | Push submodule `main` to remote. Optionally update parent pointer. |

### Publish Flow

`reins publish` becomes:

1. Ensure submodule is on `main` branch.
2. `git push origin main` inside the submodule.
3. If push fails (diverged), `git pull --rebase origin main` then retry once.
4. Optionally: `git add harness && git commit -m "chore(harness): update pointer"` in the parent — for GitHub UI only, not critical.

### Agent Guardrails

**Principle:** Agents must never run git commands directly against the harness submodule. All harness operations go through `reins` commands.

**Enforcement:**

- Agent hooks (e.g. Claude Code pre-tool hooks on Bash) should reject any `git` command that targets the harness directory — `git -C harness`, `cd harness && git`, `git submodule update`, etc.
- Skills and agent-task definitions should document this constraint so agents use `reins sync`, `reins publish`, etc.
- AGENTS.md should list the allowed reins commands for harness management.

**Escape hatch:**

- `reins harness-git <args>` (or similar) — a passthrough that forwards git args to the submodule. For human troubleshooting only. Prints a warning that it's a low-level escape hatch.

### CI Considerations

CI workflows use `token: ${{ secrets.SUBMODULE_PAT }}` with `submodules: recursive`. `actions/checkout` clones the submodule at whatever pointer the parent has. With `ignore = all`, the pointer might be stale, but CI only reads harness files for linting — staleness doesn't matter for structural checks. If CI needs latest harness, add `git submodule update --remote harness` after checkout.

## Alternatives Considered

### Git subtree

Merge harness history directly into the parent repo. Simpler mental model (no submodule init, no detached HEAD), but messier git history, harder to contribute changes back upstream, and breaks the cross-repo design goal — the harness is meant to be included in many related repositories as a shared submodule.

### Simple clone + .gitignore

Remove the submodule. `reins init` clones the harness into `harness/`, `.gitignore` ignores it. Simplest possible model, but `harness/` disappears from the parent repo (invisible on GitHub), CI needs its own clone step, and cross-repo sharing loses the git-native coordination that submodules provide.

## Future Considerations

- **Configurable commit/push behavior.** Per-command auto-commit and auto-push settings in `.reins-config`. For now: auto-commit always, push only on `reins publish`.
- **Cross-repo coordination.** When multiple repos include the same harness, `reins sync` and `reins publish` are the coordination points. The `branch = main` + `ignore = all` config works identically in every repo.
