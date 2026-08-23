---
type: idea
title: 'Feature Retros: Add retro.md artifact to completed exec-plans'
slug: 2026-02-20-feature-retros-add-retro-md-artifact-to-completed-exec-plans
lifecycle: done
status: resolved
created: 2026-02-20
author: mnbiehl
---

# Feature Retros: Add retro.md artifact to completed exec-plans

**Resolved:** 2026-05-05

## Resolution

Landed in M6. `reins doc create retro` scaffolds a retro.md alongside the
completed exec-plan, the plan-complete flow reminds the user to write one,
and the plan-structure lint rule covers retro placement.

- Templates: `_create_retro` in `src/reins/commands/doc_create.py`
- Design: `kb/reins/design-docs/m6-polish-trust-signals.md`

## Description

Feature Retros: Add retro.md artifact to completed exec-plans

## The Gap
The harness captures planning (exec-plans) and problems (tech-debt), but has no structured place for post-feature reflection. When a branch merges, the completed/ directory preserves *what we planned* but not *what we learned*.

## Proposal
Add a `retro.md` template that lives alongside completed exec-plans:

```
exec-plans/completed/{branch}/
  plan.md
  progress.md
  decisions.md
  touched-areas.md
  retro.md          <-- NEW
```

## Suggested Template Sections

### Outcome Assessment
- Goal achieved? (yes/partial/no + notes)
- Plan accuracy (scope creep, estimation accuracy)
- Unexpected complications

### What Worked
- Patterns worth repeating
- Good decisions that paid off
- Tools/approaches that helped

### What Didn't Work
- Anti-patterns to avoid
- Friction points encountered
- Decisions that turned out poorly

### Learnings
- Technical insights
- Process insights
- Agent-specific observations (for agentic dev context: where did the agent struggle/excel?)

### Actionable Follow-ups
- Ideas spawned (link to harness/reins/ideas/)
- Tech debt identified (link to harness/reins/tech-debt/)
- Process improvements to consider

## Integration Points
1. `reins plan complete` prompts for retro or auto-generates template
2. Retros reviewed during sprint digest alongside ideas/
3. Future: aggregate retros to surface systemic patterns across features

## Synergy with PR History Crawling

This idea connects directly with the pr-history-crawling feature. The PR crawl pipeline extracts review comments, and feature retros should leverage this data:

- **PR review mining for retros:** When generating a retro, crawl all PRs associated with the feature branch to extract:
  - Mistakes caught during review (request-changes comments, correction threads)
  - Patterns of what reviewers flagged (repeated issues = learning opportunity)
  - How corrections were made (the fix commits, follow-up discussions)
  
- **Automated "Review Learnings" section:** The distillation phase could auto-generate a section summarizing:
  - Code quality issues caught in review (before they hit main)
  - Architectural feedback that changed the approach
  - Reviewer suggestions that improved the implementation

- **Cross-feature pattern detection:** Aggregating review feedback across multiple feature retros could surface systemic issues (e.g., "auth-related PRs consistently have error handling flagged" → add to golden-principles or AGENTS.md).

This turns PR reviews from point-in-time feedback into durable institutional knowledge.

## Conceptually Similar (reviewed before proposing)
- ideas/ — forward-looking, not reflective
- decisions.md — captures *during* not *after*
- tech-debt — reactive problem catalog, not learnings
- quality-scores — domain-level health, not feature-level

## Notes

_No additional notes yet._
