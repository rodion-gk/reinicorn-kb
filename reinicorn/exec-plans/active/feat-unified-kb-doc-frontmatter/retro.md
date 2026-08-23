---
type: retro
title: 'Retro: feat/unified-kb-doc-frontmatter'
slug: feat-unified-kb-doc-frontmatter
lifecycle: active
status: draft
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat/unified-kb-doc-frontmatter
---

# Retro: feat/unified-kb-doc-frontmatter

## What Went Well

- Round-trip stability was treated as a correctness property from the first
  commit (b0598f8): `dumps(*parse(text)) == text` is pinned by test because
  `review.push_candidate` asserts a one-file diff and
  `candidate_matches_draft` compares exact text. kb PR #9 then verified the
  byte-exact round trip on all 123 docs (358 distinct values) before
  committing, and the review lane kept working after landing.
- Measuring the corpus before designing the migration paid off: splitting
  `**Key:**` runs on whether they precede the first `##` separated 47
  distinct names into a short header allow-list and body prose
  (`**Why:**`, `**Decision:**`), so the migration moved 0 body lines
  (kb PR #9: "0 lost, 0 added, 0 reordered").
- The orphan-detection fix shipped in both places, not only the one the
  spec named: `doc-gardening.yml` (reports) and `post_merge.py` (archives,
  destructive). b92a3c4's regression test was confirmed to fail on the old
  `tr '/' '-'` comparison. The exact-ref sweep is live: issue #44
  (2026-08-09) flagged three orphans by their real branch names.
- The five review findings that were answered got structural fixes
  (239293a): `frontmatter.render()` validates-then-serializes and all three
  create paths go through it, so "a doc cannot be born failing the lint
  that guards it" is a property of the seam, not a convention. The same
  commit replaced per-branch `ls-remote --exit-code` with one batched query
  that treats failure as "cannot verify" -- a transient outage would
  otherwise have reported every active plan as orphaned.
- The branch tip briefly pinned a local-only kb commit left behind by a
  smoke test (fixed in 3f78f53 before CI saw it); catching a dangling
  submodule pointer pre-push is exactly the failure the concurrent
  remove-kb-submodule spec was written for.

## What Could Be Improved

- Ten of the fifteen CodeRabbit threads opened on PR #30 on 2026-07-27 were
  never replied to or fixed; the response comment covered five. One of the
  unanswered ones -- `plan.py:103`, "never write a template plan without
  validated frontmatter" (Major), posted seven minutes after the "all five
  fixed" comment -- came back three days later as issue #41 / PR #42:
  `rcorn plan create` wrote frontmatter-less plans from any pre-migration
  template and the pre-push spec gate rejected every plan the tool made.
- Still open from that review, verified against current main: `parse()`
  rejects CRLF fences and a closing `---` at EOF (silently "no frontmatter"
  for the whole corpus); `read`/`write` use the platform-default encoding
  despite `allow_unicode=True`; `is_doc` matches `EXCLUDED_DIRS` against
  absolute `path.parts`, so a checkout under a dir named `references`
  validates nothing; `draft_refs.py:46` still `read_text()`s around
  `frontmatter.read`; `kb_seed.py:102` still emits unquoted
  `created: [date]` (a YAML list); `FrontmatterRule` and `is_doc` split the
  exclusion rule between them.
- The two PR bodies stated opposite merge orders (#30: "land #9 first";
  kb #9: "do not merge before #30"); the kb PR's rebase comment had to
  correct it. A hard interlock belongs in one place, with the CI reason.
- The plan said the migration script would be "appended verbatim to this
  plan when it runs"; it was not -- kb PR #9 holds only `.md` files and the
  plan body has no script. The plan's ACs and tasks are all unticked,
  status is still `planning`, the doc sits in `active/` 17 days after the
  merge, and the remote branch was never deleted, so the new orphan sweep
  cannot flag it.
- The migration snapshot went stale while the PRs sat open for eight days:
  six docs created on kb main in legacy format had to be migrated again
  (kb #9 0dc63b3, de52479), plus two body conflicts. The migration also
  synthesized `fix-security-critical-mvp/plan.md` as `done` when the doc
  said `abandoned`: its header block sat after a `##`, the deliberate
  first-`##` stop skipped it, and the checker did not notice -- CodeRabbit
  on kb #9 did.
- The spec lagged the implementation and still does: review-lane fields
  (`review_pr`/`approved_by`/`review_cancelled`), the `docmeta.py` deletion
  and the two `review` modules are absent from
  `specs/unified-kb-doc-frontmatter-schema.md` (own PR comment 2026-07-28,
  "worth a follow-up commit", never made). `origin`/`human_validated` were
  also made optional (`CORE_REQUIRED` excludes them) for 87 legacy docs, a
  deviation flagged "for the maintainer" on kb #9 and recorded nowhere.

## Lessons Learned

- Triage every review thread to a verdict -- fixed, declined with reason,
  or deferred to a tracked item -- before calling a review closed. The
  untouched threads here mixed a real bug (#41) with cheap hardening; an
  unreplied thread costs more later than a one-line decline.
- When a change forks the on-disk format, every writer that a *template* or
  *legacy state* can reach must be tested with stale inputs, not only fresh
  ones. PR #42's three-path parity test (template-less / stale / freshly
  seeded) is the shape that was missing.
- A mechanical migration of a live corpus needs a freeze or an explicit
  re-run step in the plan; "six docs added since the snapshot" and two
  conflicts were predictable for an eight-day window.
- A migration's synthesis fallback ("no header found, derive from path")
  needs an assertion that the doc truly has no header, not just that the
  anchored scan came up empty; the `done`-vs-`abandoned` error hid in that
  false negative.
- Spec-vs-implementation divergence is invisible to diff review. The
  `branch:` -> plan -> `spec:` chain is now three field reads (idea
  `code-review-should-be-spec-aware-...`), which is what would have made
  the missing review-lane fields visible to a reviewer.

## Action Items

- Fix or explicitly decline the still-open #30 threads: CRLF/EOF fence
  tolerance in `frontmatter.parse`; `encoding="utf-8"` in `read`/`write`;
  `is_doc` on kb-relative parts with the rule's dot/underscore/`generated`
  checks folded in; `draft_refs` via `frontmatter.read`; quote `[date]` in
  `kb_seed`; and the three test gaps (legacy fixture in
  `test_plan_complete_updates_status`, vacuous
  `test_template_dir_is_skipped`, no cases for the `YAMLError` and
  non-mapping `parse` fallbacks).
- Update `specs/unified-kb-doc-frontmatter-schema.md`: add the three
  review-lane fields, the `docmeta.py` removal and the two `review` modules
  to "Consumers to repoint"; record the `type:`-is-REGISTRY-key and
  optional-`origin` deviations.
- Decide `origin`/`human_validated`: keep optional, or require them with an
  explicit `unknown` origin for the 87 legacy docs. Record the outcome as a
  spec amendment or a debt doc either way.
- Close out the plan: tick ACs/tasks, `rcorn plan complete`, delete the
  merged remote branch `feat/unified-kb-doc-frontmatter`.
- Untracked items deferred in the kb #9 triage: the idea title `[:80]` hard
  slice with no word boundary (about ten truncated titles in the corpus),
  and a gardening pass over `lifecycle: done` plans with unchecked
  checklists.
