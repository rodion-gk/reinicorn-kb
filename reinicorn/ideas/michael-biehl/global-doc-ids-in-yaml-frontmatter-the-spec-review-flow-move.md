---
type: idea
title: 'Global doc IDs in YAML frontmatter: the spec review flow moves docs (specs/draft'
slug: global-doc-ids-in-yaml-frontmatter-the-spec-review-flow-move
lifecycle: active
status: new
created: 2026-07-31
author: Michael Biehl
---

# Global doc IDs in YAML frontmatter: the spec review flow moves docs (specs/draft
## Description

Global doc IDs in YAML frontmatter: the spec review flow moves docs (specs/drafts/<slug> -> specs/<slug> on acceptance), so any path-based reference breaks. Give every kb doc a stable global ID in its frontmatter (and/or doc title) at creation time, and make cross-doc references (create-from links, related-doc links, review targets) resolve by ID, not path. Depends on the unified-kb-doc-frontmatter-schema spec; interacts with registry-driven-doc-types and the create-from graduation idea.

## Notes

_No additional notes yet._
