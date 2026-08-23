---
type: idea
title: Configurable "open in Markdown editor" env var
slug: configurable-open-in-markdown-editor-env-var
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
---

# Configurable "open in Markdown editor" env var

## Description

Configurable 'open in Markdown editor' env var for rcorn. Today docs presumably open with $EDITOR/$VISUAL, which forces one editor for every file type. Add a markdown-specific override so rcorn can launch a dedicated MD editor (e.g. Obsidian) while $EDITOR stays vim for everything else in the terminal. Resolution order idea: $RCORN_VISUAL / $RCORN_EDITOR (rcorn-specific), then a generic $MD_EDITOR, then fall back to $VISUAL / $EDITOR as today. Open questions: whether to support a generic $MD_EDITOR at all vs rcorn-namespaced only; GUI editors detach rather than block, so rcorn needs to decide whether to wait for the editor to exit before continuing (Obsidian returns immediately); and whether a config file key should mirror the env var.

## Notes

Related: [[obsidian-integration-emit-vault-native-conventions-so-the-kb]] — that
idea keeps Obsidian read-mostly with authoring going through rcorn. This one is
the complementary escape hatch: rcorn still owns doc creation, but the *editing*
step can hand off to a GUI MD editor instead of the terminal `$EDITOR`.

See also [[html-as-a-kb-doc-type]] and
[[lavish-axi-as-the-visual-review-surface-for-kb-docs]] — if "open this doc for a
human" becomes one pluggable seam, then `$EDITOR`, Obsidian, and a rendered-HTML
review surface are all just backends for the same verb, and the env-var
resolution order above is the mechanism that picks between them.
