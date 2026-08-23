---
type: idea
title: Extension and preset discovery as a CLI surface — from SpecKit
slug: extension-and-preset-discovery-as-a-cli-surface-from-speckit
lifecycle: active
status: new
created: 2026-07-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Extension and preset discovery as a CLI surface — from SpecKit
## Description

SpecKit ships a full customization ecosystem as CLI surface:

- **Extensions** add new capabilities — commands and workflows.
- **Presets** customize existing workflows without adding commands, and stack
  with priority ordering.
- **Bundles** package curated sets of extensions, presets, and steps behind a
  `bundle.yml` manifest, so a whole role-oriented setup is provisioned with one
  command.
- Discovery and lifecycle: `specify extension search|add`, `specify preset
  search|add`, `specify bundle search|install|info|validate`.
- Resolution order is documented: project-local overrides > presets >
  extensions > core defaults.

Reinicorn has no extension surface at all. The `extensions/stacks/` directory
and `apply-variant.sh` that the 2026-05-04 mood board referred to no longer
exist in the repo, so as of 2026-07-30 there is no customization mechanism to
discover, share, or install.

## Why this is worth something, and why it is not first

SpecKit's ecosystem is the main thing it has that Reinicorn does not — that,
and 30+ agent integrations against Reinicorn's three
(`platform-instructions/{claude,copilot,cursor}.md`). Both are distribution
problems: they matter for adoption by people who are not already sold.

But unlike [[templates-should-ship-in-the-tool-not-the-user-s-kb-from-spe]],
this is a feature that can be added later without a migration. Nothing gets
harder by waiting. It belongs on the list, not at the top of it.

The piece worth borrowing early is the **layering discipline** rather than the
package manager: a documented resolution order for "the tool's default vs. what
this kb overrode vs. what this project overrode." That question already exists
today for templates and platform instructions, and it currently has no answer.
Settling the layering model first makes an extension surface a natural
extension of it later, rather than a second competing mechanism.

## Sketch

- Start with resolution order as a stated rule, applied to templates and
  platform instructions.
- Then `rcorn extension list` over whatever ships in the tool, before anything
  remote or searchable.
- A remote catalog and `search`/`add` only once there is something worth
  distributing — this is not urgent while the harness has a single serious
  user.
- Bundles are explicitly a later question, and possibly never: SpecKit's
  role-oriented provisioning is adjacent to the persona thinking that is an
  explicit non-goal per the positioning memory. Package the workflows, not the
  roles.

## Notes

Surfaced during the 2026-07-30 four-tool comparison — see
[[2026-05-04-feature-mood-board-from-agent-os-bmad-speckit]]. Supersedes the
"extensions and presets as first-class CLI surface" bullet in that doc, which
described a `extensions/stacks/` layout that has since been removed.
