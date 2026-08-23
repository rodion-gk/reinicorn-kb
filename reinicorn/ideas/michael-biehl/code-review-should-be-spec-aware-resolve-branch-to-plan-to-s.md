---
type: idea
title: 'Code review should be spec-aware: resolve branch to plan to spec and review the'
slug: code-review-should-be-spec-aware-resolve-branch-to-plan-to-s
lifecycle: active
status: new
created: 2026-07-28
author: Michael Biehl
---

# Code review should be spec-aware: resolve branch to plan to spec and review the

## Description

Code review should be spec-aware: resolve branch to plan to spec and review the
diff against it.

Today the default review path sees **only the diff**. `.agents/skills/code-review`
(CodeRabbit) reviews changed code for bugs, security, and quality. Nothing in it
loads the spec the work implements, so "does this match what we said we'd build?"
is a question no reviewer is asked.

## Notes

### The gap surfaced concretely

`reinicorn#30` (unified YAML frontmatter) defines `review_pr`, `approved_by`, and
`review_cancelled` in `frontmatter.py`. The approved spec it implements,
`specs/unified-kb-doc-frontmatter-schema.md`, never names those three fields —
neither in the shared-core list nor in per-type fields.

The implementation is *right* and the spec is stale. But nothing flagged the
divergence, because no reviewer had both artifacts in hand at once. Found by
accident while revising an unrelated spec.

### The linkage already exists — the frontmatter work makes it a field read

Post-`reinicorn#30`, the chain is fully mechanical. From the exec-plan's
frontmatter:

```yaml
type: plan
branch: feat/unified-kb-doc-frontmatter
spec: specs/unified-kb-doc-frontmatter-schema.md
```

So: current branch → the plan whose `branch:` matches → that plan's `spec:` →
the spec doc. Before frontmatter this meant regex-scraping prose and was not
worth building. After it, it is three field reads. The timing is the point.

### Half of it is already built and unwired

`.agents/skills/requesting-code-review/code-reviewer.md` already has the slots:

```text
## Requirements/Plan
{PLAN_REFERENCE}
...
- All plan requirements met?
- Implementation matches spec?
```

They are manual template placeholders. Nothing resolves them, and the skill that
actually runs on PRs (`code-review`) has no equivalent section at all. The
capability is designed but not connected.

### Sketch of the work

- A resolver — likely on `frontmatter.py` — that takes a branch and returns the
  plan and spec paths, or nothing.
- `code-review` loads the resolved spec into review context and adds an explicit
  "does the diff match the spec, and where does it diverge?" pass.
- Degrade quietly: no plan, or a plan with no `spec:`, means review proceeds
  exactly as today. Most branches will not resolve, and that must be silent.

### Design note: divergence is not automatically a defect

Implementation going past its spec often means **the spec should be updated**,
which is what happened in `reinicorn#30`. The output should name the divergence
and ask for reconciliation, not assume the code is wrong. A reviewer that treats
every gap as a code defect will push people to under-implement.

### Related

- Depends on the frontmatter migration: `reinicorn#30`, `reinicorn-kb#9`.
- Strong prior art in `feat/enforce-review-lane` (`reinicorn#27`): it adds a
  shared resolver, `linter/spec_refs.py`, that resolves a plan's spec reference
  against the kb's git-tracked path set. That is most of the "plan → spec" half
  of the chain already written and tested. Reuse it rather than growing a second
  resolver; this idea mainly adds the "branch → plan" hop and the review wiring.
