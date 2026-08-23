---
type: idea
title: Bridge lavish Annotations to GitHub PR Reviews
slug: bridge-lavish-annotations-to-github-pr-reviews
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Bridge lavish Annotations to GitHub PR Reviews

## Summary

Render a kb doc to HTML, review it visually in `lavish-axi`, and have the
element- and text-range annotations land on the **GitHub PR as real review
comments** anchored to the right lines of the markdown source.

This resolves the weakest point in [[html-as-a-kb-doc-type]] (its open question
4: annotation round-tripping) and answers open question 2 in
[[lavish-axi-as-the-visual-review-surface-for-kb-docs]] (whether visual review
replaces or precedes the PR lane). The answer this proposes: **neither — it feeds
the PR.** The human never reads a raw markdown diff, and the PR still holds the
complete audit trail, which is non-negotiable for a public kb repo.

## The planning-time framing (this may be the real use)

The framing here was "HTML as a useful **planning** tool" — and that is a sharper
fit than spec review. lavish's own pitch is *"tell the agent exactly what to
change **before coding starts**."* That is exec-plan review, not spec review.

Note the tension with current scope: the review lane is **specs-only** today, and
exec-plans do not go through review at all. So the highest-value target for
visual review is the lane that currently has no review step. Two readings:

- Visual review is a **new pre-implementation gate** for exec-plans, separate
  from the specs review lane and not necessarily PR-backed.
- Or exec-plans get pulled into the review lane, and visual review comes with
  them.

This needs deciding before the PR-bridge design matters, because a plan review
that never becomes a PR does not need the GitHub half at all.

## Verified mechanics

### The line-anchoring constraint — decides the design

GitHub's REST API **cannot create review comments on unchanged lines**. The
"Files changed" UI gained this in Sept 2025; the API has not followed
(`github/roadmap#347`). There is also a live report of the documented
`start_line`/`side` payload returning 422 (`cli/cli#13358`) — the review-payload
form works where the direct comment call does not.

What this means concretely:

- **New spec or plan draft → entire file is an addition → every line is
  commentable.** This is the dominant review-lane case and it works cleanly.
- **Second-round review of an edited doc → only changed lines are anchorable.**
  An annotation on untouched prose cannot be posted inline and must degrade to a
  general PR comment quoting the passage.

That degradation path is a required part of the design, not an edge case.

### Source-line provenance

The bridge needs `HTML element → markdown source line`. The standard mechanism is
markdown-it's `token.map` (`[startLine, endLine]`) with a small plugin injecting
`data-source-line` onto block elements; `markdown-it-py` exposes the same
`Token.map`. Annotation → nearest `[data-source-line]` ancestor → source line →
PR diff position.

Mermaid nodes map back to a line within the fenced ```mermaid block by node ID —
tractable, and the same lookup.

### The zero-dependency constraint

`pyproject.toml` declares `dependencies = []`. reinicorn has **no runtime Python
dependencies at all**, and the sdist allowlist shows that is deliberate. Adding
`markdown-it-py` would break that stance.

The clean way out: **do the rendering on the Node side.** `lavish-axi` is already
invoked via `npx`, so markdown-it plus a ~10-line `data-source-line` plugin costs
nothing extra in Python. Alternative is an optional extra
(`reinicorn[visual]`), which keeps the default install clean but splits the
install story.

### Existing GitHub pattern

`src/reinicorn/github.py` already shells out to `gh` via `subprocess` — so
posting a PR review is `gh api --input -` with an existing, tested helper. No new
machinery, and comments post as the reviewer's own GitHub identity, which is the
right provenance for an audit trail.

## Does it actually need an API layer?

The proposal was "a small API layer". Worth stating plainly: **for the
solo-maintainer case it needs no service at all.**

**Architecture A — local shim (recommended first).** `rcorn <type> review
--visual` renders to HTML, runs `lavish-axi`, polls, and on session end
translates annotations into one `gh api` PR review call. Fully local, zero deps,
no hosting, no GitHub App, no secrets. Fits the local-first / zero-runtime-
dependency stance that [[obsidian-integration-emit-vault-native-conventions-so-the-kb]]
already settled on.

**Architecture B — hosted API layer (later, conditional).** A service earns its
keep only when the reviewer is **not at the machine**: a teammate or an outside
contributor on the public repo. `lavish-axi share` already publishes artifacts to
ht-ml.app, so the shape exists — a service would receive annotations from a
shared artifact and post to the PR via a GitHub App. That means hosting, app
registration, token custody, and an abuse surface on a public repo.

Given the solo-maintainer self-review gate that just landed (`32f4ba3`), remote
reviewers are not a current reality. **Recommendation: build A, and treat B as an
unlock gated on actually wanting outside review.** Raising the trade-off once —
if the hosted layer is the interesting part, it should be built deliberately and
not arrived at by accident.

## The cheap reverse direction

The other direction is read-only and much cheaper: **pull existing PR comments
into the HTML artifact as annotations.** No API write constraints, no line-
anchoring problem, no service.

This is especially interesting because the repo already ingests CodeRabbit review
threads via the `autofix` skill — so CodeRabbit's findings could surface as
visual annotations on the rendered doc rather than as a wall of PR threads. This
may be the highest value-per-effort piece of the whole idea and could ship
independently of everything above.

## Open questions

1. **Which lane first** — exec-plans (best fit for the planning framing, but no
   review lane today) or specs (has the lane, weaker fit)? See the framing
   section; this gates everything else.
2. Does a plan-review flow need GitHub at all, or is the local annotate → agent
   loop sufficient without a PR?
3. Round-trip fidelity: is `data-source-line` granular enough for *text-range*
   annotations, or does inline-level provenance need character offsets too?
4. What happens to annotations that cannot be anchored — batch into one general
   PR comment, or fail loudly?
5. Does the agent apply annotation fixes before or after the PR review is posted?
   Posting first preserves the audit trail; applying first is a shorter loop.
6. Node-side rendering vs. an optional Python extra — which breaks less?
7. Is `lavish-axi share` + a GitHub App ever worth it for a solo public repo, or
   is architecture B permanently theoretical here?

## Relationship to other ideas

- [[html-as-a-kb-doc-type]] — this is the concrete answer to its round-tripping
  question, and reinforces its "render target, not storage" conclusion.
- [[lavish-axi-as-the-visual-review-surface-for-kb-docs]] — the tool being bridged.
- [[configurable-open-in-markdown-editor-env-var]] — same pluggable "open this doc
  for a human" seam.

## Sources

- GitHub Changelog, "Files changed public preview now supports commenting on
  unchanged lines" (2025-09-25); `github/roadmap#347`; `cli/cli#13358`
- GitHub REST docs — pull request review comments
- `github.com/kunchenguid/lavish-axi` — README, AGENTS.md
