---
type: plan
title: 'QA Runbook: MVP Unified Attach'
slug: qa-runbook
lifecycle: done
status: done
created: 2026-02-20
author: Michael Biehl
branch: mvp-unified-attach
---

# QA Runbook: MVP Unified Attach

**PR:** #4 (`feature/mvp-unified-attach`)
**Tester:** _______________

## Prerequisites

- A **scratch test repo** (disposable — attach will modify it)
- Python >= 3.13
- `gh` CLI authenticated (`gh auth status`)
- `git` configured with user.name / user.email
- An editor set via `$EDITOR` or `$VISUAL`

## Setup

```bash
# Install reins from the feature branch
pip install "git+https://github.com/mnbiehl/reins.git@feature/mvp-unified-attach"

# Or if testing locally:
cd /path/to/reins && pip install -e .
```

Verify:

```bash
which reins
reins --version  # expect 0.1.0
```

---

## Test Cases

### T1: Install from git URL

| Step | Action | Expected |
|------|--------|----------|
| 1 | `pip install "git+https://github.com/mnbiehl/reins.git@feature/mvp-unified-attach"` | Installs without errors |
| 2 | `reins --version` | Prints `0.1.0` |
| 3 | `reins --help` | Lists `attach`, `feedback`, `hooks`, `sync`, `publish`, `status`, `plan`, `idea`, `lint` |

**Result:** PASS / FAIL _______________

---

### T2: Attach to fresh repo (local bare harness)

| Step | Action | Expected |
|------|--------|----------|
| 1 | `mkdir /tmp/qa-test && cd /tmp/qa-test && git init` | Fresh git repo |
| 2 | `reins attach` | Prints header, asks for harness repo mode |
| 3 | Choose `2` (create local bare repo) | Creates `../qa-test-harness.git`, prints success |
| 4 | Observe output | `harness/` submodule added, AGENTS.md copied, skills copied, hooks installed |
| 5 | `ls harness/` | Contains golden-principles.md, exec-plans/, etc. |
| 6 | `cat AGENTS.md` | Contains reins agent instructions |
| 7 | `ls .claude/skills/` | Contains skill files (count > 0) |
| 8 | `ls .git/hooks/` | Contains post-checkout, pre-push, post-merge |
| 9 | `cat .git/hooks/post-checkout` | Contains reins marker, calls `reins _hook-check` |

**Result:** PASS / FAIL _______________

---

### T3: Attach to repo (existing harness URL)

| Step | Action | Expected |
|------|--------|----------|
| 1 | Fresh git repo (as T2 step 1) | — |
| 2 | `reins attach`, choose `1` | Prompts for harness repo URL |
| 3 | Enter a valid git URL | Submodule added, rest of attach proceeds |

**Result:** PASS / FAIL _______________

---

### T4: Attach idempotence — re-running on already-attached repo

| Step | Action | Expected |
|------|--------|----------|
| 1 | Use repo from T2 (already attached) | — |
| 2 | `reins attach` | Detects existing `harness/`, prompts about removal |
| 3 | Choose `N` (don't remove) | Prints error and exits (non-zero) |
| 4 | Observe AGENTS.md handling | Prompts "Overwrite AGENTS.md? [y/N]" |

**Result:** PASS / FAIL _______________

---

### T5: Attach outside a git repo

| Step | Action | Expected |
|------|--------|----------|
| 1 | `cd /tmp && mkdir not-a-repo && cd not-a-repo` | Plain directory |
| 2 | `reins attach` | Prints "Not inside a git repository", exits non-zero |

**Result:** PASS / FAIL _______________

---

### T6: Git hooks fire

| Step | Action | Expected |
|------|--------|----------|
| 1 | Use attached repo from T2 | — |
| 2 | `git add -A && git commit -m "test: initial"` | Commit succeeds |
| 3 | `git checkout -b test-branch` | post-checkout hook fires (may suggest plan creation) |
| 4 | Make a change, commit, `git checkout main` | post-checkout fires again |
| 5 | `git merge test-branch` | post-merge hook fires (check for stale plans) |

**Note:** pre-push requires a remote. If using local bare harness, add an
origin: `git remote add origin /tmp/qa-test-bare.git && git init --bare /tmp/qa-test-bare.git`

| 6 | `git push origin main` | pre-push hook fires (touched-areas generation) |

**Result:** PASS / FAIL _______________

---

### T7: Feedback — inline text with `gh`

| Step | Action | Expected |
|------|--------|----------|
| 1 | `reins feedback "QA test issue — please ignore"` | Attempts `gh issue create --repo mnbiehl/reins` |
| 2 | Observe output | If `gh` succeeds: "Issue created." with URL. If fails: falls back to browser. |
| 3 | Check GitHub | Issue exists with title, body contains Environment section |

**Clean up:** Close the test issue on GitHub.

**Result:** PASS / FAIL _______________

---

### T8: Feedback — editor mode

| Step | Action | Expected |
|------|--------|----------|
| 1 | Ensure `$EDITOR` or `$VISUAL` is set | e.g. `export EDITOR=vim` |
| 2 | `reins feedback` (no arguments) | Opens editor with HTML-commented template |
| 3 | Verify template | Contains `<!-- ... -->` instructions about title/body format |
| 4 | Write a title on first line, body below, save & close | Proceeds to issue creation |
| 5 | Include a `# Markdown Header` in the body | Header is preserved (not stripped) |
| 6 | Observe the created issue | Title matches first line, body includes markdown header |

**Clean up:** Close the test issue on GitHub.

**Result:** PASS / FAIL _______________

---

### T9: Feedback — empty input

| Step | Action | Expected |
|------|--------|----------|
| 1 | `reins feedback` → leave editor empty, save & close | Prints "Feedback text cannot be empty.", exits 1 |
| 2 | `reins feedback` → no editor set, enter empty string at prompt | Same error |

**Result:** PASS / FAIL _______________

---

### T10: Feedback — browser fallback (no `gh`)

| Step | Action | Expected |
|------|--------|----------|
| 1 | `PATH=/usr/bin:/bin reins feedback "browser test"` (hide `gh`) | Falls back to browser |
| 2 | Observe terminal | No GTK/toolkit noise printed after browser opens |
| 3 | Observe browser | GitHub new-issue form pre-filled with title, body, label |

**Result:** PASS / FAIL _______________

---

### T11: Skills content verification

| Step | Action | Expected |
|------|--------|----------|
| 1 | Use attached repo from T2 | — |
| 2 | `find .claude/skills -name "*.md" \| wc -l` | Count matches reins source skills |
| 3 | Spot-check a skill file | Contains valid SKILL.md content (not empty) |
| 4 | Open Cursor in this repo | Skills appear in Cursor's skill list |

**Result:** PASS / FAIL _______________

---

### T12: Wheel asset bundling (non-editable install)

| Step | Action | Expected |
|------|--------|----------|
| 1 | `pip install "git+...@feature/mvp-unified-attach"` (NOT `-e`) | Non-editable install |
| 2 | `python -c "from reins.assets import get_asset_path; print(get_asset_path('AGENTS.md'))"` | Returns a path inside site-packages `_data/` |
| 3 | `reins attach` in a fresh repo | AGENTS.md and skills copied from bundled assets |

**Result:** PASS / FAIL _______________

---

## Summary

| Test | Description | Result |
|------|-------------|--------|
| T1 | Install from git URL | |
| T2 | Attach — local bare harness | |
| T3 | Attach — existing harness URL | |
| T4 | Attach — idempotence | |
| T5 | Attach — outside git repo | |
| T6 | Git hooks fire | |
| T7 | Feedback — inline + gh | |
| T8 | Feedback — editor mode | |
| T9 | Feedback — empty input | |
| T10 | Feedback — browser fallback | |
| T11 | Skills content | |
| T12 | Wheel asset bundling | |

## Notes

- After QA, close any test issues created on GitHub
- Delete scratch repos: `rm -rf /tmp/qa-test /tmp/qa-test-harness.git /tmp/qa-test-bare.git`
- If any test fails, file via `reins feedback` (dogfooding!)
