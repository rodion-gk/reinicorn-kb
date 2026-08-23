---
type: retro
title: 'Retro: feature/beta-cleanup'
slug: feature-beta-cleanup
lifecycle: active
status: draft
created: 2026-07-06
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feature-beta-cleanup
---

# Retro: feature/beta-cleanup

## What Went Well

- Subagent-per-task with two-stage review caught three real production bugs the plan
  didn't know about: `_post-merge` had no mode gating (archived plans while reins was
  disabled), `block-raw-kb-git.sh` failed open on Cursor/Copilot tool names, and the
  Windows symlink-fallback message promised a sync that `reins update` doesn't do.
- The audit-before-spec approach paid off: discovering that decisions/progress/retro
  were never filled in (13 of 17 plan dirs pure template) turned a vague "clean up the
  kb" into a concrete workflow change (scaffold only what's enforced).
- Dogfooding closed the loop live: the stale `/create-exec-plan` hook message fired
  during this branch's own `plan create`, and this retro was created by the exact
  active-dir flow the branch implements.

## What Could Be Improved

- The plan's placeholder-grep patterns for stub deletion missed 10 header-only files
  (single-line `# Decisions`), needing a review round-trip; content-based emptiness
  rules beat pattern-matching specific template strings.
- Two plan details were wrong on arrival: `_post-merge`'s real contract (origin/*
  refs, not merge status) and AGENTS.md having no skills section to append to.
  Both were absorbed by implementers, but plan-time verification of assumed contracts
  would have been cheaper.
- Task 6's new test had no red phase (pre-existing code already no-opped unknown
  platforms) — acceptable as a regression lock, but the plan sold it as TDD.

## Lessons Learned

- Wiring a previously-inert guard is the moment its assumptions get tested: the
  tool-name allowlist looked fine for months because nothing installed the hook.
  Presence-based gating (does the payload carry a command?) survives editor renaming;
  name allowlists don't.
- "Filed as idea + deleted" is a genuinely comfortable way to cut features — git
  history plus a searchable idea doc removed all deletion anxiety across 3,800 lines.
- A committed relative symlink (`.claude/skills -> ../.agents/skills`) is a clean
  compatibility bridge: git tracks it, Claude Code follows it, and the live session
  survived its own skills directory moving.

## Action Items

- [ ] Decide whether the plan template should carry `**Origin:**` provenance — fresh
      plans currently trip the kb/provenance lint warning (flagged in Task 13).
- [ ] Backlog: block-raw-kb-git regex false-positives on quoted strings such as
      `grep "cd kb && git"` (fails closed with clear guidance; low priority).
- [ ] Michael rewrites HUMANS.md in his own voice before the beta announcement.
