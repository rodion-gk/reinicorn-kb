---
type: idea
title: PR History Knowledge Crawling for Attach Workflow
slug: 2026-02-15-pr-history-knowledge-crawling-for-attach-workflow-mine-merge
lifecycle: active
status: new
created: 2026-02-15
author: mnbiehl
---

# PR History Knowledge Crawling for Attach Workflow

## Summary

When reins attaches to an established repo, the current `analyze-codebase`
skill examines static code (file structure, dependencies, coverage, CI). But
established repos have years of tribal knowledge encoded in PR review comments —
coding conventions, architectural decisions, quality standards, and domain
expertise — that static analysis can't capture.

Build a `/crawl-pr-history` skill that mines a repo's PR history to extract
structured knowledge, feeding into the harness documentation that agents use for
future work.

## Architecture

### New Components

| Component | Path | Purpose |
|-----------|------|---------|
| Skill | `.claude/skills/crawl-pr-history.md` | Agent workflow for three-phase crawl |
| CLI command | `bin/reins` (add `crawl-prs` subcommand) | CLI entry point with `--resume`, `--distill`, `--status` flags |
| Raw storage | `harness/generated/pr-knowledge/raw/` | Per-PR JSON files and aggregate stats |
| Distilled output | `harness/generated/pr-knowledge/distilled/` | Four structured knowledge artifacts |
| Config | `.reins-config` | New `REINS_PR_CRAWL_*` settings |

### Integration Points

- **analyze-codebase** Phase 2 reads `distilled/` artifacts as additional input
- **review-pr** skill can reference `conventions.md` and `quality-standards.md`
- **CLI** dispatches to crawl script, same pattern as other commands

## Phase 1: Discovery & Scoring

Score every merged PR: `signal_score = (comments * 3) + (min(lines_changed, LINE_CAP) * 0.1) + (review_duration_days * 2) + recency_bonus`

- Use `gh pr list --state merged --json ...` for initial scoring pass
- Present top N to developer before committing to full crawl
- Developer can adjust sample size and weights

## Phase 2: Extraction (Raw)

For each selected PR, extract full content via `gh pr view` and `gh api`.

- Per-PR JSON: body, diff stats, all comments, reviews, timeline
- `pr-index.json`: all sampled PRs with metadata
- `reviewer-stats.json`: who reviewed what, volume, domains
- Resumable: write each PR JSON as extracted, track progress in `crawl-metadata.json`

## Phase 3: Distillation (Hybrid)

Agent processes raw data into four knowledge artifacts:

1. **`conventions.md`** — What reviewers consistently enforce (naming, error handling, etc.)
2. **`architectural-decisions.md`** — Why things are the way they are (lightweight ADRs)
3. **`quality-standards.md`** — What the bar looks like (derived from request-changes patterns)
4. **`domain-expertise.md`** — Who knows what (reviewer-to-domain mapping, bus factor)

Each finding tagged HIGH_CONFIDENCE or NEEDS_REVIEW for developer triage.

## Configuration

```ini
REINS_PR_CRAWL_SAMPLE_SIZE=150
REINS_PR_CRAWL_MONTHS=6
REINS_PR_CRAWL_MIN_COMMENTS=2
REINS_PR_CRAWL_LINE_CAP=2000
```

## CLI Interface

```bash
reins crawl-prs              # Full run: discover -> extract -> distill
reins crawl-prs --resume     # Resume interrupted extraction
reins crawl-prs --distill    # Re-run distillation on existing raw data
reins crawl-prs --status     # Show last crawl info and stats
```

## Files to Create/Modify

| Action | File |
|--------|------|
| Create | `.claude/skills/crawl-pr-history.md` |
| Create | `scripts/crawl-pr-history.sh` |
| Modify | `bin/reins` — add `crawl-prs` subcommand |
| Modify | `.reins-config` — add `REINS_PR_CRAWL_*` defaults |
| Modify | `.claude/skills/analyze-codebase.md` — add optional pr-knowledge input |
| Modify | `CLAUDE.md` — add `/crawl-pr-history` to Skills Reference table |

## Notes

Full detailed plan — scoring formula details, JSON schemas, distillation
flow, and verification steps — to be expanded from this outline when picked up.
