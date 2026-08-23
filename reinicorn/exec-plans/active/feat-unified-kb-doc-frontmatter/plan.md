---
type: plan
title: 'Execution Plan: feat/unified-kb-doc-frontmatter'
slug: feat-unified-kb-doc-frontmatter
lifecycle: active
status: planning
created: 2026-07-27
author: Michael Biehl
branch: feat/unified-kb-doc-frontmatter
ticket: N/A
spec: specs/unified-kb-doc-frontmatter-schema.md
---

# Execution Plan: feat/unified-kb-doc-frontmatter

## Goal

Give every kb doc real fenced YAML frontmatter with a validated schema, and route
all metadata reads/writes through one module so no consumer regex-scans prose
again.

The concrete bug this closes: `doc-gardening.yml` orphan detection has no field
to read a plan's branch from, so it sanitizes remote heads (`tr '/' '-'`) and
compares them to sanitized plan-dir names. That mapping is lossy and many-to-one
— a deleted `feature/mvp` plan is not flagged as orphaned if a literal
`feature-mvp` branch exists.

## Acceptance Criteria

- [ ] `src/reinicorn/frontmatter.py` is the only module that touches the `---`
      block; `docmeta.py` is deleted.
- [ ] Every kb doc (minus the excluded non-docs) carries valid frontmatter.
- [ ] `dumps(*parse(text)) == text` holds for every doc the migration emits.
- [ ] `rcorn kb lint` fails at `error` severity on missing/invalid frontmatter.
- [ ] Orphan detection matches exact refs — a `feature-mvp` branch no longer
      masks a deleted `feature/mvp` plan.
- [ ] `rcorn review start` / `merge` / `cancel` round-trip without disturbing
      doc bodies.
- [ ] Full suite green; coverage stays at or above the 87% floor.

## Approach

Decisions taken (2026-07-27):

| Decision | Choice |
|---|---|
| YAML impl | `pyyaml>=6.0.3` as reinicorn's first runtime dep. Corpus values (`[TICKET-ID or N/A]`, `Tech Debt: Output Discipline`) require real scalar quoting in both directions; a hand-rolled writer would silently corrupt docs. |
| Migration | One-shot, no compat shim. `frontmatter.py` only ever knows YAML. |
| `type:` values | `doc_types.REGISTRY` keys — `plan`, `debt`, not the spec's `exec-plan`/`tech-debt`. Validation is `meta["type"] in REGISTRY`; no second vocabulary. |
| Lint severity | `kb/frontmatter` at `error` on landing; `kb/provenance` rule and its `.sh` deleted. |

Three gaps found in the spec during exploration:

1. **Review-lane fields are missing from the schema.** `review.py` writes
   `Review-PR`, `Approved-by`, `Review-cancelled` today. They become
   `review_pr` / `approved_by` / `review_cancelled`, allowed on gated types.
2. **The API needs a text-level core.** The spec words it as `read(path)` /
   `write(path, …)`, but `review.py` reads candidate content out of `git show`
   and never from a path. Core is `parse(text)` / `dumps(meta, body)`; the
   path forms are thin wrappers.
3. **Round-trip stability is load-bearing, not cosmetic.**
   `push_candidate` asserts the review ref differs from main by exactly one
   added file, and `candidate_matches_draft` compares exact text. Unstable
   serialization breaks every review push.

Measured baseline (2026-07-27, before any change): `rcorn kb lint` reports **89
`kb/provenance` failures** across the corpus. Both docs created for *this branch*
— by `rcorn plan create` and `rcorn idea create` — are among them, each missing
`Origin`. The create paths emit docs that violate the project's own lint rule,
and at `warning` severity nothing has ever failed the build over it. That is the
case for landing `kb/frontmatter` at `error`, and for `validate()` being shared
by the create paths rather than living only in CI.

Assumptions:

- `golden-principles.md` gets file-level `type: principle` frontmatter. The
  spec's per-principle `id`/`category` would need one file per principle, which
  its own Non-Goals rule out — deferred.
- Dates stay `datetime.date` objects in the meta dict. `yaml.safe_dump` of the
  *string* `"2026-07-19"` emits it quoted to preserve type; date objects emit
  bare.

## Tasks

### 1. `frontmatter.py`

- [ ] Add `pyyaml>=6.0.3` to `[project].dependencies`.
- [ ] Core: `parse(text) -> (meta, body)`, `dumps(meta, body) -> str`,
      `get(text, key)`, `set_meta(text, updates)` (value `None` removes).
- [ ] Path wrappers: `read(path)`, `write(path, meta, body)`,
      `validate(meta) -> list[str]`.
- [ ] `dumps` passes `sort_keys=False, allow_unicode=True, width=float("inf"),
      default_flow_style=False`. **`width` is load-bearing** — the default of 80
      folds long values, and tech-debt docs carry full-sentence `Remediation:`
      values. `allow_unicode` keeps existing em-dashes from becoming escapes.
- [ ] Key order per spec: `type, title, slug, lifecycle, status, created,
      updated, author, origin, human_validated, <per-type>, tags, related`.
      Unknown keys appended sorted, never dropped, so `validate` can report them.
- [ ] Per-type fields keyed by REGISTRY key: `plan` → `branch` (required,
      unsanitized), `ticket`, `spec`, `retro` · `retro` → `branch` (required),
      `plan` · `idea` → `promoted_to` · `spec`/`prd` → `supersedes`,
      `superseded_by`, `implemented_by` · `debt` → `id`, `category`, `severity`,
      `remediation` · `principle` → none. Gated types also allow `review_pr`,
      `approved_by`, `review_cancelled`.
- [ ] `validate()`: required core present; `type in REGISTRY`; `lifecycle in
      {active, done, dropped}`; `origin in {human, ai-assisted}`;
      `created`/`updated` are dates; `human_validated` bool; `tags`/`related`
      string lists; per-type required present; no unknown keys.
- [ ] Single exclusion list for non-docs (currently duplicated in
      `provenance.sh`): `README.md`, `index.md`, `ATTRIBUTION.md`,
      `quality-scores.md`, `cleanup-queue.md`, `progress.md`, `decisions.md`,
      anything under `_template/`.
- [ ] `tests/test_frontmatter.py` replaces `tests/test_docmeta.py`, including a
      round-trip stability test and the raw-literal pinning `test_docmeta.py`
      does on purpose.

### 2. Migration

- [ ] **Ad-hoc script, not a CLI command** (decided 2026-07-27). The earlier
      plan argued for `rcorn kb migrate-frontmatter` on the grounds that
      adopting repos carry the same legacy blocks. There are no adopting repos:
      `rcorn kb list` reports two scopes, `reinicorn/` and `generated/`, and
      `generated/` is excluded from doc scans. This is a one-shot over one
      scope in one kb. A permanent command would mean cli.py surface, a
      dispatch entry, idempotency guarantees, and wheel/sdist weight — forever,
      for an operation that runs once.
      The script lives in the session scratchpad and is appended verbatim to
      this plan when it runs, so the kb records how the corpus was rewritten
      without shipping dead code.
- [ ] Scan **all** leading `**Key:** value` runs, not just the first —
      `_header_span` stops at a blank line, but tech-debt docs carry a second
      block (`**Severity:** / **Domain:** / **Remediation:**`).
- [ ] **Do not slurp body prose.** A corpus probe (2026-07-27) found 47
      distinct `**Key:**` names across the kb, most of them body prose —
      `**Why:**`, `**Bad:**`, `**Good:**`, `**Decision:**`, `**Steps:**`,
      `**Trigger:**`, `**Rules:**`, `**post-checkout:**`. Only a minority are
      header fields. Keep `docmeta._header_span`'s anchor test (a run counts as
      a header only if it contains `Date`/`Author`/`Created`/`Status`/`Origin`)
      and port it into the migration, extended to accept a second run. Migrate
      an explicit allow-list of header names; anything else stays in the body.
- [ ] Migrate this allow-list and **nothing else** — measured from the corpus
      2026-07-27 by splitting `**Key:**` runs on whether they precede the first
      `##` heading:

      | legacy header field | → | occurrences |
      | --- | --- | --- |
      | `Status` | `status` (+ derived `lifecycle`) | 117 |
      | `Author` | `author` | 114 |
      | `Date`, `Created` | `created` | 115 |
      | `Origin` | `origin` | 37 |
      | `Human-validated` | `human_validated` | 36 |
      | `Ticket` | `ticket` | 13 |
      | `Spec` | `spec` | 8 |
      | `Review-PR` | `review_pr` | 7 |
      | `Severity`, `Domain`→`category`, `Remediation` | debt fields | 9 |
      | `Branch` | `branch` | 2 |

      Header-position names that must **stay in the body**: `Files` (26 — a
      label introducing a list), `Goal` (7), `Architecture` (7), `Resolved`
      (13), `Category` (5 — prose, not the debt enum), `Reference`, `PR`,
      `Tester`, `License`, `Predecessor`, `Trigger`. Removing only allow-listed
      lines from the header run keeps the rest in place.
- [ ] Derive `lifecycle` from `status` per the spec's table, keeping `status`
      verbatim; `slug` from filename stem; `title` from H1 (H1 stays in body).
- [ ] **5 docs need synthesis** (no anchored header): the four completed plans
      `feature-cross-platform-support/plan.md`, `fix-security-critical-mvp/plan.md`,
      `mvp-unified-attach/2026-02-19-mvp-unified-attach-plan.md`, and
      `skills-fork-template-docs/plan.md` — type `plan`, `branch` from the dir
      name, `lifecycle: done` (they are under `completed/`), `created`/`author`
      from git history — and `golden-principles.md` (type `principle`).
- [ ] `# Execution Plan: <branch>` → `branch:` field + a generic H1.
- [ ] Unmapped legacy keys are reported and fail the run, never silently dropped.
- [ ] **Verification moves from the script to its output.** A throwaway script
      gets no unit tests; correctness is asserted on the 118 rewritten files:
      1. **Body preservation (the safety property):** for every doc, the new
         body is byte-identical to the original text minus exactly the
         allow-listed header lines. Nothing else may move.
      2. `frontmatter.validate()` returns `[]` for all 123 docs.
      3. `dumps(*parse(text)) == text` for every migrated file.
      4. `rcorn kb lint` green with `kb/frontmatter` at `error`.
      5. Human review of the kb PR diff.
      Run 1–3 as a checker script over the working tree *before* committing, so
      a bad rewrite never reaches a commit.
- [ ] 118 of 144 kb `.md` files carry a legacy block; the rest are 21 excluded
      non-docs plus the 5 header-less docs above, which get frontmatter
      synthesized from path + git history. 118 + 5 = the 123 migrated docs.

### 3. Repoint consumers

- [ ] `commands/doc_show.py:86` — `_title_and_status` reads from frontmatter.
- [ ] `commands/status.py` — reads `lifecycle`; `collect_gated_drafts` rows too.
- [ ] `commands/plan.py:180` — drop the `re.sub` on `**Status:**`.
- [ ] `review.py:55,129,245-250` — `candidate_text` / `_finalize_tree` use
      `set_meta` / `get`.
- [ ] `commands/review.py:129` — `_stamp_draft`'s remove/set loop collapses to
      one `set_meta(text, fields)`.
- [ ] `linter/rules/draft_refs.py:54` — `get(text, "status") == "in-review"`.
- [ ] `commands/doc_create.py:27` (`_provenance`), `commands/idea.py`,
      `kb_seed.py` — emit frontmatter; `kb_seed`'s plan template gains `branch:`.
- [ ] Delete `docmeta.py`.

### 4. Lint + CI

- [ ] `linter/rules/frontmatter.py` → `FrontmatterRule`, registered as
      `kb/frontmatter` in `linter/rules/__init__.py`.
- [ ] `linters/.lint-config.json` — replace the `kb/provenance` entry with
      `kb/frontmatter` at `error`; delete `linters/rules/kb/provenance.sh`.
- [ ] `doc-gardening.yml` "Check for orphaned plans" — read `branch:` from each
      active plan, then `git ls-remote --exit-code --heads origin
      "refs/heads/$branch"`; flag only on exact-ref miss. Keep the existing
      env-var passing so branch names reach the report as data, never
      interpolated into shell source.

## Dependencies

**Decided 2026-07-27: `spec/remove-kb-submodule` (kb PR #8) lands first. Build
against the post-#8 world.**

What that changes here:

- **Task 2 loses its second PR.** With no submodule pointer, the migration is a
  single PR against `crystldm/reinicorn-kb`; there is no pointer bump in this
  repo to pair with it.
- **Task 4 rebases onto a rewritten checkout.** #8 §9 drops
  `submodules: recursive` from all four workflows and adds an explicit kb clone
  step. The orphan-detection fix sits on top of that step rather than beside
  `submodules: recursive`. The change itself is unaffected — it alters *how the
  branch is read*, not how kb is checked out — but write it against the clone
  step and expect to rebase if #8 slips.
- **Tasks 1 and 3 are unaffected.** #8 §2 moves kb discovery behind
  `get_kb_dir()`, and roughly twenty `commands/` call sites reach the kb through
  `get_kb_dir()` / `require_kb_dir()` and need no edit. `frontmatter.py` and the
  consumer repointing sit above that seam.
- **Do not depend on `stage_kb_pointer()`.** #8 §6 deletes it and its four call
  sites outright.

Also in flight: `fix/kb-remote-resolution-and-publish-errors` — kb remote URL
resolution and `push_main_with_retry` error classification. Touches `kb.py`,
`post_checkout.py`, `submodule.py`. No overlap with this branch's files, but
both land near #8's blast radius.
