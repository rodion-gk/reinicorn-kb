---
type: spec
title: 'Enforce the review lane: draft-refs resolution, declared plan spec, and a push gate'
slug: enforce-the-review-lane-draft-refs-resolution-declared-plan
lifecycle: active
status: approved
created: 2026-07-25
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/5
---

# Enforce the review lane: draft-refs resolution, declared plan spec, and a push gate

Closes #23.

## Problem

The review lane is documented as a gate in three places — `writing-plans/SKILL.md`
("Gated docs"), `executing-plans/SKILL.md` (Step 1), and `kb/reinicorn/README.md`
(`specs/` = approved specs) — but nothing in the tooling enforces or even surfaces
it. On `feat/coverage-reports` the whole implementation was written and merged
before its spec review (`crystldm/reinicorn-kb#4`) was resolved; that review is
still open now that the code has shipped. The violation was caught by inspection,
not by any tool.

Three separate defects let it through:

**1. `kb/draft-refs` only matches `kb/`-prefixed paths.** `draft_refs.py:17` uses
`(?<![\w/])kb/[\w./-]+\.md`, so it sees a reference only when it is written as
`kb/<scope>/specs/…`. Verified against the current rule:

```text
'Spec: `specs/test-coverage-reports.md`.'                   -> []
'Spec: `reinicorn/specs/drafts/x.md`'                       -> []
'Spec: kb/reinicorn/specs/drafts/test-coverage-reports.md'  -> ['kb/reinicorn/…']
```

The plan in question wrote `specs/test-coverage-reports.md` — the natural
kb-relative form — so the rule matched nothing. It fired zero diagnostics across
the entire incident.

There is a second-order miss in the same rule: that plan cited the *future
approved* path (`specs/<slug>.md`) rather than the actual current one
(`specs/drafts/<slug>.md`). Even a matcher keyed on `/drafts/` would have missed
it. As written the rule catches the careful author who names the draft location
and misses the careless one who doesn't — exactly backwards.

**2. Plans have no structured spec reference.** `exec-plans/_template/plan.md`
carries Ticket / Author / Created / Status and no Spec field, so spec references
land in freeform prose inside `## Goal` or `## Approach` in whatever path style
the author picked. `plan_structure.py` does not look for one at all. This is the
root cause of defect 1: `draft-refs` is text-mining prose because there is no
declared field to read.

**3. The gate is advisory and on-demand.** `kb/draft-refs` is `severity: warning`,
so `rcorn kb lint` exits 0 and the Lint Kb workflow stays green even when it does
fire. Nothing in the commit or push path checks it — grepping
`.reinicorn/hooks/` and `hooks/` for `in-review|draft|approv` returns nothing. A
branch whose spec is sitting in review can be committed, pushed, PR'd and merged
with no signal at any step. By contrast `block-raw-kb-git.sh` hard-blocks raw git
in `kb/`, a far cheaper mistake — the stronger guard is on the smaller problem.

## Design Goals

- A plan's spec reference is **declared**, not mined from prose, so the check is
  deterministic rather than regex-dependent.
- A reference to an unapproved spec is caught **however it is written** —
  `kb/`-prefixed, kb-relative, or scope-relative — and whether it names the
  draft location or the future approved one.
- Every path referenced in a doc's frontmatter must be **known to git**. A
  reference that no tracked kb path can satisfy is itself a finding, so typos and
  stale references cannot pass as "nothing to check".
- Omitting the reference must not be a way to dodge the gate; the only escape is
  an explicit, reviewable one.
- The gate fails the Lint Kb workflow, not just `rcorn kb lint`'s output.
- The mistake is blocked at push time, where it becomes expensive, in the same
  spirit as `block-raw-kb-git.sh`.
- Implementation that legitimately must run ahead of approval has a supported,
  visible way to record that decision.
- The push gate must not become a new way to brick pushes: an internal error in a
  *policy* check should not block work the way a data-integrity check does.

## Design

### 1. Declared spec reference on plans

Add a `Spec` header field to `kb/reinicorn/exec-plans/_template/plan.md`:

```markdown
**Ticket:** [TICKET-ID or N/A]
**Spec:** [kb path to the spec this implements, or N/A]
**Author:** [developer or agent]
```

Add `FIELD_SPEC = "Spec"` to `docmeta.py` and read it with the existing
`get_field`, matching how `Status` / `Review-PR` are handled. `rcorn plan create`
emits the field from the template unchanged.

`N/A` is the explicit escape hatch. It answers the open policy question in #23
(direction 6): work that legitimately runs ahead of approval declares `N/A`,
which is visible in the plan, survives into review, and is greppable — rather
than happening silently as it does today.

### 2. `kb/draft-refs`: resolve references however they are written

Two changes to `linter/rules/draft_refs.py`.

**Read the declared field first.** For each active plan, read `**Spec:**`. If the
field is absent, or still holds the template placeholder, emit a diagnostic — a
plan with no declared spec reference is itself a finding, so omission is not a
loophole. If the value is `N/A`, the plan is exempt and the rule moves on.

**Broaden the prose matcher as a backstop.** Keep scanning the body (fence-skipping
as today) so a plan that declares `N/A` but references a draft in prose is still
caught. Anchor the pattern on a known doc-type directory rather than on the `kb/`
prefix, which bounds false positives while accepting all three path styles:

```python
_DOC_DIRS = "|".join(re.escape(d) for d in sorted(
    {dt.dir_path for dt in REGISTRY.values() if dt.dir_path != "."}
))
_REF_RE = re.compile(
    rf"(?<![\w/])(?:{KB_DIR_NAME}/)?(?:[\w.-]+/)*(?:{_DOC_DIRS})/[\w./-]+\.md"
)
```

This matches `specs/x.md`, `reinicorn/specs/drafts/x.md` and
`kb/reinicorn/specs/x.md`, and ignores unrelated `.md` paths in prose.

**Resolution is against git, not the filesystem.** Every path referenced in
frontmatter must be known to git. Build the kb's tracked-path set once per lint
run — `run_git("ls-files", "-z", cwd=kb_dir)`, the same plumbing `pre_push.py`
and `kb.py` already use — and resolve references by exact lookup into that set
rather than by probing the filesystem.

This is both cheaper and stricter than stat-ing candidates, and it disposes of
two problems by construction:

- **Containment.** `git ls-files` only ever emits repo-relative paths under the
  kb root. A reference containing `..`, an absolute path, or anything outside the
  kb simply fails to match — there is no traversal to defend against, because no
  attacker-controlled string is ever joined onto a filesystem path before it has
  been proven to be a tracked kb file.
- **Untracked files.** A doc that exists on disk but was never committed is not a
  real reference. Git is the authority the review lane already runs on, so the
  lint agrees with what a reviewer would actually see on the branch.

**Candidate forms.** A reference is interpreted three ways, and each is looked up
in the tracked set:

1. `kb/`-prefixed — strip the `kb/` prefix, look up the remainder.
2. kb-relative — `reinicorn/specs/x.md`, look up as written.
3. scope-relative — `specs/x.md`, prefixed with the plan's own repo-scope
   directory: `<scope>/specs/x.md`.

**Precedence and ambiguity.** Evaluate all three; collect the distinct tracked
paths they hit. Exactly one distinct hit resolves. More than one distinct hit is
an **ambiguity diagnostic** naming every candidate — the rule never silently
picks a winner, because guessing is how the original bug was born. Zero hits is
an **unresolved-reference diagnostic**: a plan whose declared spec matches no
tracked kb path is a finding, not a pass. This closes the `Spec: specs/typo.md`
hole where a misspelling would otherwise sail through both checks.

**Drafts fallback — the second-order miss.** When none of the three forms hits a
tracked path, retry each with `drafts/` inserted before the filename. A plan
citing `specs/<slug>.md` while only `specs/drafts/<slug>.md` is tracked
therefore resolves to the draft and is reported. The fallback runs *only* after
exact lookup fails, so a reference to a genuinely approved `specs/x.md` still
resolves to the approved doc even when a same-named draft is also tracked.

**Diagnosis** is unchanged in shape: report when the resolved path sits under
`drafts/`, or when the resolved file's `Status` is `draft` or `in-review`.
`draft` is added to the existing `in-review` check because a plan built on a
never-submitted draft is the same violation.

**Generalization.** The tracked-path check is not `Spec`-specific: any frontmatter
field whose value is a path is validated the same way. `Spec` is the only such
field today, but the helper takes a field name so the next one is free.

### 3. `kb/plan-structure`: require the field

`plan_structure.py` gains a check that `**Spec:**` is present and not the
template placeholder, alongside its existing required-section checks. It stays
`severity: warning` — it reports document shape. The gating is `draft-refs`'
job, and `draft-refs` diagnoses a missing field independently so the gate does
not depend on plan-structure's severity.

### 4. Severity: error

`linters/.lint-config.json`: `kb/draft-refs` moves from `warning` to `error`, so
`rcorn kb lint` exits non-zero and the Lint Kb workflow fails. If the lane is a
real gate, a plan built on an unapproved spec should fail CI rather than warn.

**Migration.** The three currently-active plans predate the field and would fail
immediately. Backfilling their `**Spec:**` values (real path or `N/A`) is part of
this change, not a follow-up — the severity flip and the backfill land together
so `main` is never red.

### 5. Push gate

Add `_ensure_plan_spec_approved(root)` to
`commands/internal/pre_push.py`, run after `_ensure_kb_pushed`:

1. Skip when mode is `incognito` or `disabled`, matching `_ensure_kb_pushed`.
2. Resolve the current branch to `kb/<scope>/exec-plans/active/<sanitize_branch(branch)>/plan.md`.
   No plan → nothing to check, return 0.
3. Read `**Spec:**`. `N/A` → return 0.
4. Resolve the path with the same helper `draft-refs` uses, drafts-fallback
   included. If it lands under `drafts/` or its `Status` is `draft` /
   `in-review`, block the push naming the doc, its status, and
   `rcorn review status <slug>`, with `git push --no-verify` as the documented
   bypass.

The resolution logic is shared, not duplicated — extracted into a helper both
`draft_refs.py` and `pre_push.py` import, so the lint rule and the push gate can
never disagree about whether a reference is approved.

**Failure mode: open, not closed.** `_ensure_kb_pushed` fails *closed* because it
guards data integrity — a dangling submodule pointer breaks every downstream
checkout. This gate guards a process norm, and a parse hiccup that silently
bricks every push in the repo is worse than a missed policy warning. An
unexpected exception here warns and returns 0. This is a deliberate asymmetry
between the two checks in the same file and should be commented as such.

**Fail-open must be loud.** A gate that degrades silently is indistinguishable
from a gate that was never wired up — which is precisely the failure mode of the
`reins`-era hooks in #24. The exception path prints a warning naming the branch,
the plan path, and the exception, and says plainly that the approval gate did not
run. A regression that disables the gate must be visible in the push output, not
inferred later by inspection.

### 6. Next-step hints carry the gate

Small wording changes so the CLI stops presenting an unapproved spec as a neutral
fact (#23 direction 5):

- `rcorn review start` adds a line that the spec is now in review and
  implementation should wait for approval.
- `rcorn kb status` flags "current branch has an in-review spec" rather than
  listing in-review docs neutrally.

Lowest-value part of this change and the first thing to cut if review wants the
scope tighter.

## Non-Goals

- Blocking `git commit`. The push is where the mistake becomes expensive and
  where `_ensure_kb_pushed` already lives; adding a second commit-time hook is
  more friction for less benefit.
- Linting archived plans. `draft-refs` scans `exec-plans/active/` only; once
  `rcorn plan complete` archives a plan the evidence leaves the lint surface.
  Worth fixing, but it is a separate change with its own migration cost.
- Enforcing the lane server-side (branch protection, required status checks on
  the code repo). Local hooks plus Lint Kb are the scope here.
- Reconciling the already-shipped `feat/coverage-reports` violation or the still
  -open `reinicorn-kb#4` review. That is a cleanup decision, not a tooling one.
- Any change to which doc types are gated. Only `spec` is gated today and that
  stays true.

## Acceptance Criteria

- `specs/x.md`, `reinicorn/specs/drafts/x.md` and `kb/reinicorn/specs/x.md` all
  resolve to the same doc and are all diagnosed when that doc is unapproved.
- A plan citing `specs/<slug>.md` when only `specs/drafts/<slug>.md` is tracked
  is reported — the second-order miss from the incident.
- A reference matching no tracked kb path (`Spec: specs/typo.md`) is reported as
  unresolved, not silently passed.
- A reference whose candidate forms hit two distinct tracked paths is reported as
  ambiguous, naming both; the rule never picks one.
- A doc present on disk but untracked does not satisfy a reference.
- A reference containing `..`, or an absolute path, resolves to nothing and is
  reported as unresolved — no filesystem access occurs for it.
- A reference to an approved `specs/x.md` still resolves to the approved doc when
  a same-named `specs/drafts/x.md` is also tracked.
- A plan with no `**Spec:**` field, or with the template placeholder still in
  place, is reported by `draft-refs`.
- A plan declaring `**Spec:** N/A` is exempt from the field check but still
  scanned for draft references in prose.
- A reference inside a fenced code block is still ignored.
- A plan referencing an approved spec produces no diagnostic.
- `rcorn kb lint` exits non-zero on a draft reference, and the Lint Kb workflow
  fails.
- The three currently-active plans carry a backfilled `**Spec:**` and `main` is
  green immediately after the severity flip.
- `git push` is blocked on a branch whose plan references a draft or in-review
  spec, naming the doc and `rcorn review status`; `--no-verify` bypasses it.
- `git push` is not blocked when the plan declares `N/A`, when the branch has no
  plan, when the spec is approved, or when mode is `incognito` / `disabled`.
- An unexpected exception in the spec-approval gate warns — naming the branch,
  the plan, and the exception, and stating that the gate did not run — and allows
  the push, while `_ensure_kb_pushed` continues to fail closed.
- Tests cover each path style, the drafts fallback, unresolved and ambiguous
  references, untracked files, traversal strings, missing/placeholder/`N/A`
  fields, the fence skip, the severity change, and every push-gate branch above.
