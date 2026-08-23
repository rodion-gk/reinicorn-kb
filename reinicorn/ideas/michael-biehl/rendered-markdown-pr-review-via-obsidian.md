---
type: idea
title: Rendered-markdown PR Review via Obsidian
slug: rendered-markdown-pr-review-via-obsidian
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Rendered-markdown PR Review via Obsidian

## The actual problem

The pain is narrow and specific: **GitHub's web UI will not let you comment on
rendered markdown.** You can toggle the rich diff to *read* a doc as prose, but
inline comments are bound to source-diff line numbers, so to say anything you
must switch back to the raw document and comment there. It is clumsy for exactly
the docs reinicorn produces — specs, plans, PRDs — which are prose, not code.

This is confirmed and long-standing, not a misconfiguration:

- Rich diff renders prose but comments cannot be left on it
  (community discussion #186730).
- Existing inline comments **do not even display** on the rich diff view
  (community discussion #160981).
- GitHub's own docs describe the rich diff as a viewing feature only.

This applies to **any doc type** — specs, plans, whatever — not just the
specs-only review lane. That is a scope note for
[[bridge-lavish-annotations-to-github-pr-reviews]], which assumed the lane
mattered.

### Why the editor-embedded PR UIs don't already solve it

VS Code (GitHub Pull Requests extension) and JetBrains both ship PR review UIs,
which proves the "review inside the tool you're already in" pattern. But both are
**diff-based** — they render source diffs, not prose. They inherit the same
limitation rather than fixing it.

So the precedent supports the *embedding* pattern, not the *rendered-comment*
capability. Nothing mainstream solves the latter. That is the gap.

## This hits the exception the earlier Obsidian research reserved

[[obsidian-integration-emit-vault-native-conventions-so-the-kb]] rejected a
first-party plugin — but explicitly reserved the option for "capability
unreachable via 1–3" (frontmatter / wikilinks / Bases). Commenting on rendered
markdown and posting to a GitHub PR **is** such a capability: it cannot be
emitted as a file convention. So this is that document's stated exception firing,
not a contradiction of it.

## Prior art — this largely exists already

Three options found, cheapest first:

### 1. "Markdown Rich Review" browser extension (zero effort, try first)

A browser extension (sabbour.me, March 2026) that bridges GitHub's rich diff and
the source diff — injecting line numbers, comment indicators, and click-to-source
navigation into the rich preview. It does not change reinicorn at all and it
targets the exact complaint. **This is the thing to try today** before building
anything.

### 2. "GitHub Review" Obsidian plugin (closest to the ask)

`community.obsidian.md/plugins/github-review` — "read, comment on, and
approve/request-changes on GitHub issues & PRs inside Obsidian." Crucially it
**renders PR markdown files as formatted text rather than diffs**, which is
precisely the missing capability. Submits real GitHub reviews
(Comment / Approve / Request changes), has collapsible diffs with "Viewed"
checkboxes, and even integrates with Claude via a local MCP server.

Maturity and risk are the problem:

- **113 downloads, version 0.1.4** — very early, effectively unproven.
- **Desktop only.**
- **Auth: a fine-grained PAT stored unencrypted in plaintext in `data.json`.**

That last point is a live hazard here. Plugin data lives at
`.obsidian/plugins/<id>/data.json`, and the vault is the **public** kb repo. A
plaintext GitHub token inside a directory that gets committed and pushed is a
credential-leak vector. The deny-all-then-allowlist `.gitignore` strategy already
recommended in [[obsidian-integration-emit-vault-native-conventions-so-the-kb]]
would exclude it — but only if that strategy is actually in place first. **Treat
the gitignore work as a prerequisite, not a follow-up.**

### 3. "Redline" Obsidian plugin (the interesting one to build on)

`github.com/nicolasassi/redline` — PR-style review comments anchored to
paragraphs, headings, images, code blocks, callouts, and tables. Two properties
make it a strong fit:

- **It anchors to Obsidian native `^blockref` markers**, injecting one on the
  target block if absent. This is semantic anchoring, not line numbers, so
  comments survive content shifting. It is also the same `[[Note#^block]]` syntax
  the earlier Obsidian research already bet on.
- **Comments live in a sibling `.review.md` sidecar** in a documented,
  tool-agnostic format. The author calls it "intentionally protocol-based… does
  not bind to any specific downstream tool," explicitly intended for "a script, an
  AI assistant, a coworker" to consume.

It has **no GitHub integration** — which is the opening. rcorn reading
`.review.md` and posting to the PR is a small, well-bounded bridge that reuses
the existing `gh` subprocess helper in `src/reinicorn/github.py`. No plugin to
write, no token in the vault, no Node dependency.

Worth noting: block-ref anchoring is more robust than the line-number mapping
proposed in [[bridge-lavish-annotations-to-github-pr-reviews]], though it does not
escape the GitHub API's inability to comment on unchanged lines — that constraint
lives on GitHub's side regardless of how the anchor is computed.

Also relevant: `Fevol/obsidian-criticmarkup` (CriticMarkup suggestions/comments in
markdown) as an alternative annotation syntax, and `github-pr-tracker` for
sidebar PR tracking only.

## Recommendation

**Do not build a plugin.** In order:

1. Try the browser extension — it may simply solve the stated pain, today.
2. Evaluate `GitHub Review` in a throwaway vault, **after** the `.obsidian`
   gitignore allowlist is in place. Judge whether 113 downloads / v0.1.4 is
   acceptable for something holding a PAT.
3. If a build is warranted, build the **Redline `.review.md` → GitHub PR bridge**
   in rcorn. Smallest surface, no new runtime, no credential in the vault, and it
   composes with the Obsidian bet already made.

## The overlap that needs resolving

There are now **two competing surfaces for the same job**: this and
[[lavish-axi-as-the-visual-review-surface-for-kb-docs]] / [[html-as-a-kb-doc-type]].
Both let a human annotate a rendered doc and feed structured edits back. Running
both is duplicated machinery.

A plausible split rather than a winner-takes-all:

- **Obsidian / Redline** — prose-heavy specs and plans; markdown stays canonical;
  no Node; local; fits the existing vault bet.
- **lavish-axi / HTML** — diagram-heavy or visual planning docs, where the mermaid
  whiteboard and layout audit are the actual value.

But this should be an explicit decision, not accretion. It is the main open
question across all four ideas in this cluster.

## Open questions

1. Does the browser extension resolve the pain outright? If so, most of this
   cluster is unnecessary.
2. Obsidian vs. lavish — pick one, or ratify the prose/visual split above?
3. Is `GitHub Review`'s plaintext-PAT model acceptable at all given a public kb
   vault, or is it disqualifying regardless of maturity?
4. Redline's `.review.md` format — stable and documented enough to build a bridge
   against, and is the author receptive to it?
5. Does the bridge post as a batched PR review, or comment-by-comment?
6. Round-2 reviews still hit GitHub's unchanged-line API limitation. Does the
   bridge degrade to a general comment, as proposed in
   [[bridge-lavish-annotations-to-github-pr-reviews]]?

## Sources

- GitHub community discussions #186730 (commenting on formatted markdown) and
  #160981 (inline comments don't show on rich diff); GitHub Docs, "rendering
  differences in prose documents"
- `community.obsidian.md/plugins/github-review`
- `github.com/nicolasassi/redline`
- sabbour.me (2026-03-23), "A browser extension for better Markdown reviews in
  GitHub pull requests"
