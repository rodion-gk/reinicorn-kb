---
type: spec
title: AGENTS.md Template & Population Skill
slug: agents-md-template-and-population
lifecycle: done
status: implemented
created: 2026-03-14
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# AGENTS.md Template & Population Skill

## Problem

`reins init` copies a generic AGENTS.md that describes reins conventions but nothing about the target project — its tech stack, build commands, or project-specific conventions. Agents starting work in a newly-initialized repo get the harness map but no project context.

The OpenAI Harness Engineering article (see `references/harness-engineering-openai.md`) establishes that AGENTS.md should be a ~100-line table of contents with progressive disclosure. Ours follows this pattern for harness navigation but leaves the project-specific half empty.

## Design Goals

1. AGENTS.md template is a terse (~100 line) map with pre-filled reins sections and clearly marked placeholders for project-specific content
2. A population skill guides agents through filling in the placeholders collaboratively (brainstorming-style: gather context, ask questions, draft, get approval)
3. A StartSession hook nudges agents to run the skill on first session after init
4. Init is additive-only — never overwrites existing AGENTS.md, CLAUDE.md, or other agent config files
5. The skill respects and incorporates existing agent config files (CLAUDE.md, GEMINI.md, .cursorrules) as baseline context

## Design

### Component 1: AGENTS.md Template

Replaces the current `AGENTS.md` asset copied by `reins init`. Structure:

```
<!-- UNPOPULATED: Run the populate-agents-md skill to complete this file. -->
# {repo-name}

<!-- TODO: One paragraph — what this repo does, its purpose, who it serves. -->

## Tech Stack & Build

<!-- TODO: Language, framework, package manager, build/test/run commands. -->

## Knowledge Base                    ← pre-filled by init
[harness pointer table]

## CLI                               ← pre-filled by init
[reins command table]

## Project Conventions
<!-- TODO: Naming, code style, key architectural patterns. -->

## Hard Rules                        ← pre-filled by init
[universal reins rules]

## Progressive Disclosure            ← pre-filled by init
```

Template variables `{repo}` and `{repo-name}` are substituted by init using the slug. The `<!-- UNPOPULATED -->` marker is what the hook checks for.

### Component 2: Population Skill

`.claude/skills/populate-agents-md.md` — brainstorming-style collaborative skill.

**Flow:**

1. **Gather context silently** — read AGENTS.md, scan for CLAUDE.md/GEMINI.md/.cursorrules, read package files (package.json, pyproject.toml, Cargo.toml, go.mod), check directory structure, read CI config, sample a few source files
2. **Use existing agent config as baseline** — if CLAUDE.md etc. exist, extract relevant info (tech stack, conventions, build commands)
3. **Present findings** — "Here's what I found: [tech stack, build commands, test runner, structure]"
4. **Ask questions one at a time** (brainstorming style):
   - What does this project do in one sentence?
   - Any conventions or patterns not captured in config files?
   - Anything agents should avoid?
5. **Draft populated AGENTS.md** — present for approval
6. **Write it** — fill TODO sections, remove `<!-- UNPOPULATED -->` marker
7. **If existing platform files found** — suggest whether to merge into AGENTS.md or keep both

**Hard constraint:** Only modifies TODO/placeholder sections. Never touches pre-filled reins sections (Knowledge Base, CLI, Hard Rules).

### Component 3: StartSession Hook

Registered by `reins init`. Checks for the UNPOPULATED marker and prints a nudge:

```
if AGENTS.md contains "<!-- UNPOPULATED":
    print "⚠️ AGENTS.md has not been populated yet."
    print "Run the populate-agents-md skill to set up this repo for agents."
```

Self-disabling: once the skill removes the marker, the hook prints nothing.

### Component 4: Init Changes

Modifications to `reins init` and the copy-assets flow:

1. **New AGENTS.md template** replacing the current one
2. **Template variable substitution** — `{repo}` → slug, `{repo-name}` → slug
3. **Skip-if-exists** for all copied files (AGENTS.md, skills, hooks)
4. **Hook registration** — install the StartSession hook (idempotent)
5. **Print nudge** at end of init: "Run the populate-agents-md skill to finish setup"

The skill handles scanning for existing CLAUDE.md/GEMINI.md itself — no need to thread state through init.

## Non-Goals

- **CLAUDE.md generation** — the skill may suggest content, but generating a full CLAUDE.md is out of scope
- **Auto-population without human interaction** — the skill is collaborative, not autonomous
- **Platform-specific skill variants** — `.claude/skills/` is the only format; other platforms that support it can use it
- **CI enforcement of population** — the hook nudges, but we don't block builds on unpopulated AGENTS.md
- **Migrating existing repos** — this covers new `reins init` runs; migrating repos that already have a populated AGENTS.md is not in scope
