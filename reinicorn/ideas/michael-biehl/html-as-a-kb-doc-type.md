---
type: idea
title: HTML as a kb Doc Type
slug: html-as-a-kb-doc-type
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# HTML as a kb Doc Type

## Summary

Should the kb be able to hold HTML docs, not just markdown? The motivating pull
is [[lavish-axi-as-the-visual-review-surface-for-kb-docs]] — *"HTML is the new
markdown"* — where a spec or plan rendered as a rich HTML artifact gets
element-level human annotation instead of prose PR comments.

**Tentative conclusion: make HTML a render target, not a storage format.** Keep
markdown as the single source of truth in git; add a rendering + review step that
produces ephemeral HTML. The full case for and against is below; this is the
recommendation the open questions should try to falsify, not a decision.

## What "HTML as a doc type" would mechanically require

Encouragingly cheap at the type layer: `src/reinicorn/doc_types.py` declares a
`filename` pattern per type (`"{slug}.md"`, `"active/{branch}/plan.md"`, …), so
the extension is already a per-type field rather than a global constant.

What is *not* cheap is everything downstream, which assumes markdown:

- `src/reinicorn/review.py` globs `*.md` when discovering drafts and resolving
  review targets (`review.py:51`, `review.py:101`, `review.py:109`).
- The linter lane, and the in-review
  `markdown-linting-a-docs-markdownlint-rule-and-a-config-basel` spec, are
  markdown-only by construction. HTML docs would silently sidestep all doc
  quality enforcement.
- `kb_seed.py` and the doc templates emit markdown bodies.
- `docmeta.py` parses the `**Date:** / **Author:** / **Status:**` header block.

## The case against HTML as a *storage* format

1. **Diffability — the decisive cost.** kb docs live in git and go through PRs on
   the public `reinicorn-kb` repo. A one-sentence edit to an HTML spec produces
   an unreadable diff. The PR *is* the review artifact and the audit trail; HTML
   destroys it.
2. **Token cost, against AXI principle 1.** Agents read kb docs directly. HTML
   markup is pure overhead per unit of content — the opposite of the
   token-efficiency principle reinicorn's own output surface was built on
   ([[agent-native-output-surface-axi-principles]]).
3. **It breaks the frontmatter bet.** The in-review
   `unified-kb-doc-frontmatter` spec and
   [[obsidian-integration-emit-vault-native-conventions-so-the-kb]] both assume
   `.md` + YAML frontmatter + `[[wikilinks]]`. HTML docs would not appear in the
   Obsidian graph, would not be queryable by Bases, and would not carry the
   typed schema.
4. **Grep-ability.** Plain-text search across the kb is a real daily affordance
   and degrades badly against markup.

## The case *for* — and the narrow exception

The genuine win is not storage, it is **review ergonomics**: mermaid diagrams
that can be edited rather than described, decision matrices that render as
tables, layout-audited artifacts, and annotation anchored to a specific text
range. All of that is achievable from markdown-as-source with an HTML render
step.

The one class where HTML may deserve to be a real stored type is a doc whose
content *is* inherently visual and has no meaningful markdown source — a PRD with
interactive mockups, or a retro dashboard. Even there, the cost is losing the
graph, the lint lane, and diffability, so the bar should be high.

## Recommended shape (to be confirmed or falsified)

Markdown stays canonical. Add a verb along the lines of:

```
rcorn spec review --visual <slug>
```

which renders the markdown doc to HTML, hands it to `lavish-axi`, collects
structured annotations, and applies them back **to the markdown source**. The
HTML artifact is ephemeral — a scratch/cache path, gitignored, never committed.

This is the same architectural conclusion the Obsidian research reached: a
**human-facing layer over rcorn-owned markdown, with zero runtime dependency** in
any agent code path, and rcorn remaining the only writer. Note the symmetry —
Obsidian is the *reading* layer, lavish is the *reviewing* layer, and both sit on
the same plain-markdown substrate.

## Open questions

1. Does anything actually require HTML at *rest*, or does render-on-demand cover
   every real case? Find a counterexample or close this.
2. If a stored HTML type is added anyway, does it need frontmatter (in an HTML
   comment? a `<meta>` block?) to stay in the schema and the Obsidian graph?
3. Markdown → HTML rendering: which renderer, and does mermaid need to be
   pre-rendered for lavish's whiteboard feature to engage?
4. Round-tripping is the hard part — can text-range annotations on rendered HTML
   be mapped back to source line ranges in the markdown reliably enough for an
   agent to apply them without ambiguity? **A concrete answer is proposed in
   [[bridge-lavish-annotations-to-github-pr-reviews]]** (markdown-it `token.map`
   → `data-source-line`).
5. Where do ephemeral artifacts live — scratch dir, `.reinicorn/`, or a cache
   under `~`? They must never land in the kb submodule.
6. Does the lint lane need an explicit "no HTML docs" rule to keep the invariant
   from eroding by accident?

## Relationship to other ideas

- [[lavish-axi-as-the-visual-review-surface-for-kb-docs]] — the tool that
  motivates this; depends on the answer here.
- [[obsidian-integration-emit-vault-native-conventions-so-the-kb]] — same
  conclusion shape, and the source of the constraints HTML would break.
- [[configurable-open-in-markdown-editor-env-var]] — "render and open for a human"
  and "open in an editor" are plausibly the same pluggable seam.
