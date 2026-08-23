---
type: idea
title: 'Origin stories: why reinicorn is shaped the way it is (mined from private-era PR'
slug: origin-stories-why-reinicorn-is-shaped-the-way-it-is-mined-f
lifecycle: active
status: new
created: 2026-08-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Origin stories: why reinicorn is shaped the way it is (mined from private-era PR

## Description

Design decisions whose *reasons* live only in the private-era PR history (mnbiehl/reins #1–#46, mnbiehl/reinicorn-kb #1–#9) and cannot be reconstructed from today's code. Captured from the 2026-08 full-history mining pass so agents stop re-litigating settled designs or "simplifying" load-bearing shapes. Candidate for promotion to a permanent kb reference doc.

## Notes

**The review lane's add-only mechanic defeats GitHub rename-detection.** Three spike PRs (kb #2/#3/#4, labeled `[SPIKE A/B/C]`, including a deliberate negative control) proved that a git-mv rename+edit PR collapses to a rename hunk with almost no commentable surface. The chosen design — draft lives at `specs/drafts/` on main, the review PR *adds* the file at its final path — exists solely to force a fully commentable new-file diff. Moving draft paths or "simplifying" to a rename would silently break review commentability.

**`doc_types.py` REGISTRY was born from scattered-path pain, not upfront design.** Private #9's review: "far too many places that have hard-coded lists, paths, etc." after the same literals spread across 8+ files (first flagged in #8's `git add harness` ×5). Today's P2/P13 principles are the scar tissue.

**The AST-walker/structural-lint lineage started with #28's `sanitize_branch` ban** — Michael's "no string literals in control flow, SOURCE OF TRUTH!" review. Notably, `sanitize_branch` misuse is the one mined bug class that never recurred — the argument the 2026-08 enforcement spec is built on.

**Overlap detection computes from git on demand because persisted derived state raced.** Private #18 removed `touched-areas.md`: the pre-push hook wrote it asynchronously while another process read it synchronously — infinite dirty-kb cycles and dangling pointers. Don't reintroduce persisted "computed" files the hooks write.

**Shell hooks are thin dispatchers into Python by design.** Private #3 found a bash hook and a Python worker independently implementing background push — the Python path was unreachable dead code, the bash path had a hidden bug. Everything routes through `rcorn _pre-push`-style entrypoints since.

**Golden principle P1 (no generated `python -c` source strings) comes from a real injection.** Private #3 interpolated a branch name into a generated script; branch names are user-controlled.

**Fail-closed guards are a lesson paid for twice.** Private #41 (`except Exception: return 0` defeating the pre-push guard) and #39 (`isatty()`-gated confirmation silently wiping the kb in non-interactive contexts, `rcorn kb remove-scope ""`). Origin of the "cannot-verify fails safe" rule.

**The tech-debt-catalog workflow demonstrably closes its loop.** #1 cataloged 60 items at MVP; #14 was a dedicated security remediation against that catalog five weeks later. Cataloging is not deferral theater here.

**Two-phase renames are the survival pattern.** The project renamed itself three times (reinicorn→reins, harness→kb, reins→rcorn/public crystldm). The working shape: PR 1 introduces internal indirection with zero behavior change, PR 2 flips the user-facing surface; finish with a grep sweep for the old identity (public #46 fixed three post-migration stragglers the gates missed).

**Go-public required a systematic blocker sweep** (#39–#44): destructive-path validation, Actions `${{ }}` injection, fail-open guards, sdist allowlisting (5.5MB→252KB after the private kb leaked into the archive), and license-text bundling (attribution alone doesn't satisfy MIT; upstream URL was wrong and had never been checked). This list is the template for any future repo this project takes public.

**Contribution policy:** human contributions only — AI-assisted work by a person is welcome; autonomous bot-submitted PRs/issues get closed without review (#38).
