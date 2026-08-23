---
type: idea
title: Add 'rcorn <doctype> create from <other-doc>' e.g. 'rcorn spec create from <idea
slug: add-rcorn-doctype-create-from-other-doc-e-g-rcorn-spec-creat
lifecycle: active
status: new
created: 2026-07-31
author: Michael Biehl
---

# Add 'rcorn <doctype> create from <other-doc>' e.g. 'rcorn spec create from <idea
## Description

Add 'rcorn <doctype> create from <other-doc>' e.g. 'rcorn spec create from <idea-slug>': graduate a kb doc into another type, carrying over its content/context as the starting point and linking back to the source doc.

## Notes

This melds perfectly with `rcorn plan create from <spec>` — the same graduation mechanism covers the whole pipeline: idea → spec → plan. Each step seeds the new doc from its source and links back, so `create from` becomes the uniform way a doc advances to the next stage.

This also ties in with the custom-docs-from-schemas direction (`specs/drafts/registry-driven-doc-types.md`): relationships between doc types can be defined in the schemas themselves. A doc type's registry/schema entry would declare which source types it can be created `from` (e.g. spec: from [idea], plan: from [spec]), so the graduation graph is data, not hardcoded CLI logic — and custom user-defined doc types get `create from` support for free.
