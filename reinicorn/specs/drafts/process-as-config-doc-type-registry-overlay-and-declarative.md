---
type: spec
title: 'Process-as-config: doc-type registry overlay and declarative type relations'
slug: process-as-config-doc-type-registry-overlay-and-declarative
lifecycle: active
status: in-review
created: 2026-08-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/18
---

# Process-as-config: doc-type registry overlay and declarative type relations

## Problem

Four retros written 2026-08-20/21 (mattpocock-adapter, registry stage 2,
enforce-review-lane, unified-frontmatter) found the same failures
independently: every audited plan was still `lifecycle: active` weeks after
its PR merged; `rcorn plan complete` only *warns* when the retro is empty and
the post-merge auto-archive completes plans with no retro at all; and spec
drift (stages 3-5 of `registry-driven-doc-types` never shipped, #27 cut
spec §6, #30 skipped the frontmatter back-fill) is recorded nowhere.

The first fix drafted for this (`enforce-plan-completion-pre-merge-retro-
gate-with-spec-drift`, kb PR #16, withdrawn) patched the symptom: a
`Spec Drift` literal on the retro row, a bespoke `_retro-gate` command, a
`kb/retro-structure` lint and a `kb/plan-lifecycle` lint — four more code
paths that name `spec`/`plan`/`retro` by key, on top of the thirteen files
that already do (`_branch_target` carries a comment saying "this coupling
stays code"; `spec_gate.py`, `plan_structure.py`, `post_merge.py`,
`plan.py`, `status.py`, `kb.py`, `spec_refs.py`, `frontmatter.PER_TYPE`).

That is the design gap. `registry-driven-doc-types` made the registry the
single source for *what a doc type is* but explicitly deferred two things:
"the config file itself" (a follow-up spec) and "plan lifecycle as data —
the retro-rides-with-plan coupling stays code". The process — which type
must point at which approved type, which type closes which, what a
reviewer must see before merge — therefore lives only in Python. Opting out
of a retro section, or running an RFC → ADR process instead of
spec → plan → retro, means forking the engine.

## Design Goals

- The process is data: a repo changes its doc process (sections, which
  types depend on or close which, what gates merge) by editing one config
  file, with zero engine changes. Deleting "Spec Drift" from a list is the
  whole opt-out.
- The engine's enforcement *events* stay fixed and few (lint, pre-push,
  pre-merge CI check, `complete`, post-merge archive); what varies is the
  type graph they read. No rule language, no custom events.
- Every built-in behavior today — spec gate, retro placement, plan
  completion — is expressible as a default config row, and the shipped
  defaults reproduce it exactly plus the retro gate and spec-drift
  accounting the retros asked for.
- A second process (RFC → ADR) is a worked example in this spec and a
  phantom-type test in the suite, not a promise.
- Branches without governed docs (dependabot, trivial docs) pass every gate
  untouched; the 11 legacy active plans get a stated migration path.

## Design

### 1. Registry overlay: `kb/<scope>/doc-types.yaml`

`doc_types.REGISTRY` becomes the **built-in defaults**. The effective
registry is `registry(root)`: defaults overlaid by an optional
`doc-types.yaml` in the kb scope dir, resolved once per process and
validated into the same frozen `DocType` objects, so every consumer keeps
its types and the module-level `REGISTRY` import sites become call sites
(mechanical sweep).

Placement in the kb scope dir, not the code repo: process is a property of
the doc corpus, it is reviewed where the docs are, a shared kb serving
several repos gets per-scope process, and the kb-side checks
(`kb-review-prs-get-real-github-status-checks`) see it without the
consuming repo checked out. The file is a non-doc (added to
`EXCLUDED_FILENAMES`), published with `rcorn kb publish` like any kb file.

Overlay semantics, by `doc_types.<key>`:

- **Override** a built-in row: only the listed fields change
  (`required_sections: [...]` replaces the list wholesale — no merge magic).
- **Add** a row: all required `DocType` fields present; enums by their
  string value.
- **Remove** a built-in row: `disabled: true`; refused if any other row's
  relation targets it.

Load-time validation fails **closed** with the file path and offending key
— a broken process config must not silently revert to defaults. Invariants:
existing ones (`gated` ⇒ slug addressing), relation targets exist, relation
fields are declared fields of the row, `filename` placeholders match
`addressing`, `closes` pairs are both branch-addressed.

The per-type frontmatter vocabulary moves into the row too — `fields` and
`required_fields` absorb `frontmatter.PER_TYPE` / `PER_TYPE_REQUIRED` — so
a custom type can declare the field its `depends_on` uses.

`rcorn doc-types show` prints the effective registry as YAML, each row
annotated `built-in` / `overlay`, so "copy the row, change one line" is
the documented way to customize.

### 2. Two relations, declared on the row

```yaml
doc_types:
  plan:
    depends_on: {field: spec, type: spec, status: approved}
  retro:
    closes: {type: plan, required: true}
    required_sections: [What Went Well, What Could Be Improved,
                        Lessons Learned, Action Items, Spec Drift]
```

- `depends_on: {field, type, status}` — the doc's `field:` must resolve
  (via `spec_refs`, unchanged) to a tracked doc of `type` with `status`, or
  be the N/A sentinel. Generalizes the plan → approved-spec rule.
- `closes: {type, required}` — this type is the closer of `type`. Implies:
  the closer is created *inside* the closee's active dir (today's
  `_branch_target` special case, now a graph lookup `closer_of(dt)`);
  `<closee> complete` moves `active/{branch}/` → `completed/{branch}/` with
  both docs; and, when `required`, the closee cannot complete without a
  filled closer. The closer's `filename` is its tail only (`retro.md`).

The `complete` verb is generated for every type that something `closes`
(today only plan), alongside `create`/`show`. Queries on the effective
registry — `closer_of(type)`, `closable_types()`, `dependencies_of(type)`
— replace the literal `REGISTRY["plan"]`/`["retro"]`/`["spec"]` lookups in
all thirteen files. A relation with no match returns `None` and the caller
skips, which is how a repo with no `closes` row gets no retro behavior.

### 3. Gates: fixed events, graph-driven

| event | reads | rule |
|---|---|---|
| `rcorn kb lint` | `required_sections` | `kb/required-sections`: every doc of every type with a non-empty list carries the headers (generalizes `plan_structure`, which was `registry-driven-doc-types` stage 4, never shipped) |
| `rcorn kb lint` | `depends_on` | `kb/draft-refs` unchanged in behavior, now iterates every row with `depends_on` |
| `rcorn kb lint` | `closes` | `kb/closer-filled`: an active closee with a required closer whose closer is missing or has only placeholder sections → error (`_retro_is_empty` moves from `commands/plan.py` to the linter as `sections_empty`) |
| `rcorn kb lint` | `closes` | `kb/lifecycle`: active closee whose `branch:` is gone from origin or whose `origin/<branch>` is an ancestor of `origin/main` → error "merged/deleted but still active — rcorn <type> complete". Network facts fail open as "cannot verify", mirroring `_live_remote_branches` |
| pre-push | `depends_on` | `spec_gate.ensure_plan_spec_approved` becomes `ensure_dependencies_approved`: for each pushed branch, every branch-addressed type with `depends_on` is checked. Same fail-open-loud contract |
| pre-merge CI | `closes` + sections | new job "Process gate" in `lint-kb.yml` running `rcorn _process-gate <branch>`: the three lint rules above scoped to that branch's docs; no governed docs → pass. Added to `main-pr-gate` required checks after the implementation merges (repo-settings action, recorded in the plan) |
| `<type> complete` | `closes` | exit 1 without a filled required closer, next-step `rcorn <closer> create`; `--abandon` stamps `status: abandoned` / `lifecycle: dropped` and needs no closer |
| post-merge | `closes` | `_archive_stale_plans` calls the generic `complete`; the hook script stops piping stdout to `/dev/null` so a refusal and its next-step are visible in merge output |

### 4. Shipped defaults

The built-in rows encode the current flow plus what the retros asked for:
`plan.depends_on` = approved spec (unchanged behavior); `retro.closes`
plan, `required: true` (new); `Spec Drift` appended to retro
`required_sections` (new). The `Spec Drift` placeholder states the content
contract: enumerate every deviation between the plan's declared `spec:` and
what shipped, each with a disposition — **amended** (link the spec review
PR), **debted** (link the debt doc), **accepted** (one-line reason) — or
the single word "None."

### 5. Worked example: RFC → ADR

```yaml
doc_types:
  spec:  {disabled: true}
  plan:  {disabled: true}
  retro: {disabled: true}
  rfc:
    dir_path: rfcs
    filename: "{slug}.md"
    addressing: slug
    gated: true
    required_sections: [Summary, Motivation, Detailed Design, Drawbacks, Alternatives]
    fields: [superseded_by]
  adr:
    dir_path: decisions
    filename: "{slug}.md"
    addressing: slug
    required_sections: [Context, Decision, Consequences]
    fields: [rfc]
    depends_on: {field: rfc, type: rfc, status: approved}
```

Gets: `rcorn rfc create` through the review lane, `rcorn adr create`, the
draft-refs lint and pre-push gate on `adr.rfc`, required-section lint on
both, no retro or completion machinery at all. Zero engine changes. The
"phantom type" test from `registry-driven-doc-types` is extended with a
synthetic closer/dependency pair and asserts placement, gate, lint and
`complete` appear with no other change.

### 6. Review rules: AGENTS.md + PR template

Repo prose, not engine:

- AGENTS.md "Pull requests": the body states spec-drift status — "matches
  spec" or the deviation list with dispositions, mirroring the retro.
- AGENTS.md "Reviewing": the reviewer diffs the change against the plan's
  declared spec; undisclosed drift is a blocking finding; the retro is part
  of the reviewed diff and held to the same bar as code.
- PR template: the stale kb checklist (`progress.md`, `decisions.md`,
  `kb/exec-plans/` paths) is replaced with "retro filled incl. Spec Drift"
  and "drift dispositions recorded".

### 7. Stages (one PR each onto a `feat-process-as-config` integration branch)

1. **Loader.** `registry(root)`, overlay parsing + validation, `fields`
   absorption of `PER_TYPE`, `rcorn doc-types show`. Behavior-preserving.
2. **Relations.** `depends_on`/`closes` fields, graph queries, literal sweep
   of the thirteen files, generic `complete`, phantom-pair test.
   Behavior-preserving (defaults unchanged).
3. **Gates.** The four lint rules, `complete` refusal + `--abandon`, hook
   stdout, `_process-gate` + CI job. Lints ship at error severity against a
   kb made clean in the same PR: backfill the 7 retro-less legacy plans
   (Spec Drift where determinable), `plan complete` every merged branch.
4. **Defaults + rules.** Flip `retro.closes.required` and add `Spec Drift`;
   AGENTS.md / PR template; ruleset required-check last.

## Non-Goals

- **No rule language.** No conditions, custom events, or scripting in the
  overlay; if a process needs a gate the engine lacks, that is an engine
  change with its own spec. The two relations are the whole vocabulary.
- **Overlay is not review-gated.** `doc-types.yaml` ships with
  `kb publish`; gating config changes through the review lane is a
  follow-up if it proves necessary.
- **Overlay does not live in the code repo.** Rejected above; revisit only
  if a per-repo override of a shared scope is needed.
- No `.coderabbit.yaml` (decided 2026-08-21); no change to the
  solo-maintainer approval gate
  (`review-lane-solo-maintainer-self-review-gate`); no doc-gardening
  consolidation (its orphan step and `kb/lifecycle` dedupe later).
- No retro *content* quality gate beyond structure and non-emptiness; no
  retroactive `Spec Drift` sections in already-completed retros.
- Supersedes only the "plan lifecycle as data" non-goal of
  `registry-driven-doc-types`; that spec's unshipped stages 3 and 5
  (literal sweep, shipped-docs genericization) are absorbed by stage 2 here
  where they overlap and otherwise remain that spec's.
