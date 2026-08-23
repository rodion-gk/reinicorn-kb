---
type: spec
title: 'Reins: Agentic Engineering Harness Template'
slug: reins-design
lifecycle: active
status: Approved
created: 2026-02-13
author: Michael + Claude
---

# Reins: Agentic Engineering Harness Template

## Overview

Reins is a GitHub repo template that encodes agentic engineering best practices into a reusable, extensible scaffolding. Inspired by OpenAI's "Harness Engineering" article (Feb 2026), adapted for Claude Code (with conventions that carry to Cursor and Copilot natively).

Two workflows: start new projects from scratch (greenfield), or bolt onto existing repositories (attach). Stack-agnostic core with a variant system for specific stacks.

Early-stage, fluid, "take what you need."

## Key Philosophy

- **Humans steer, agents execute.** The harness encodes intent, constraints, and context so agents can do reliable work.
- **Progressive disclosure.** Agents get a sparse map (AGENTS.md), not a 1,000-page manual. They're taught where to look, not told everything up front.
- **Mechanical enforcement over documentation.** When docs fall short, promote the rule into code (linters, structural tests, CI).
- **Zero-friction adoption.** The system must not impede developer workflow. Passive tracking by default, richer engagement opt-in.
- **Repository is the source of truth.** If the agent can't see it in-repo, it doesn't exist. Slack threads, Google Docs, tacit knowledge — all illegible to agents.
- **Cross-branch awareness.** Feature branches publish plans to a shared space so agents on other branches can detect conflicts and coordinate.

## Repository Structure

```
reins/
├── README.md                           # Comprehensive guide: philosophy, decisions, usage
├── AGENTS.md                           # ~100 lines. Sparse map. Universal agent entry point.
├── CLAUDE.md                           # Claude Code behavior/personality config
├── .claude/
│   ├── skills/                         # Reusable agent workflows
│   │   ├── analyze-codebase.md         # Hybrid crawl -> synthesize -> verify
│   │   ├── create-exec-plan.md         # Write a new execution plan
│   │   ├── update-harness.md           # Maintain docs, quality scores, tech debt
│   │   ├── sync-context.md             # Pull latest from harness submodule
│   │   ├── publish-plan.md             # Push branch plans to shared submodule
│   │   ├── review-pr.md               # Agent-to-agent review workflow
│   │   └── capture-idea.md            # Quick idea capture
│   ├── hooks/                          # Claude Code hooks (pre/post tool execution)
│   └── mcp.json                        # MCP server configs
├── harness/                            # The knowledge base (becomes submodule for attach)
│   ├── architecture/
│   │   ├── ARCHITECTURE.md             # Top-level domain + layer map
│   │   ├── domains/                    # Per-domain architecture docs
│   │   └── dependency-rules.md         # What can depend on what
│   ├── design-docs/
│   │   ├── index.md                    # Catalog with status
│   │   ├── core-beliefs.md             # Agent-first operating principles
│   │   └── ...
│   ├── exec-plans/
│   │   ├── active/                     # In-flight work, namespaced per branch
│   │   │   └── {branch-name}/
│   │   │       ├── plan.md
│   │   │       ├── progress.md
│   │   │       ├── decisions.md
│   │   │       └── touched-areas.md
│   │   └── completed/
│   ├── product-specs/
│   │   ├── index.md
│   │   └── ...
│   ├── references/                     # LLM-friendly docs (llms.txt files, etc.)
│   ├── generated/                      # Machine-generated context
│   │   ├── dependency-graph.md
│   │   ├── domain-map-draft.md
│   │   ├── dead-code-report.md
│   │   ├── external-deps.md
│   │   ├── test-coverage-summary.md
│   │   └── file-structure.md
│   ├── tech-debt/
│   │   ├── index.md                    # Summary dashboard, priority rankings
│   │   ├── by-domain/                  # Debt per architectural domain
│   │   ├── by-category/               # Cross-cutting debt patterns
│   │   └── cleanup-queue.md            # Prioritized list for garbage collection agents
│   ├── ideas/
│   │   ├── index.md                    # Auto-generated catalog
│   │   ├── {username}/                 # Per-developer namespace
│   │   └── shared/                     # Team-wide ideas
│   ├── golden-principles.md            # Opinionated rules, mechanically enforced
│   ├── quality-scores.md               # Per-domain/layer quality grades
│   └── DESIGN.md                       # Product design philosophy
├── linters/
│   ├── README.md                       # How to write a lint rule
│   ├── framework/
│   │   ├── run-lints.sh                # Entry point: discovers and runs all rules
│   │   └── lint-rule-template.md       # Template for new rules
│   ├── rules/
│   │   └── harness/                    # Core rules (docs freshness, cross-links, etc.)
│   ├── structural-tests/               # Architecture conformance tests
│   ├── ci-recovery/
│   │   ├── README.md
│   │   ├── recovery-protocol.md        # Generic agent decision tree for CI failures
│   │   ├── handlers/                   # Per-failure-type diagnosis + fix instructions
│   │   │   ├── lint-failure.md
│   │   │   ├── test-failure.md
│   │   │   ├── build-failure.md
│   │   │   └── deploy-failure.md
│   │   └── platforms/                  # CI platform-specific log fetching + re-triggering
│   │       ├── github-actions.md
│   │       └── _template.md
│   └── .lint-config.json               # Active rules, severity levels
├── variants/
│   ├── README.md                       # How to contribute a variant
│   ├── _template/                      # Starter template for new variants
│   │   └── manifest.json
│   ├── typescript-node/
│   ├── python/
│   ├── rust/
│   ├── go/
│   └── atlassian/                      # Issue tracker integration variant
├── scripts/
│   ├── init-greenfield.sh              # Interactive new project setup
│   ├── attach-to-repo.sh              # Add harness to existing repo
│   ├── setup-hooks.sh                  # Install git hooks for auto-sync
│   └── apply-variant.sh               # Apply a stack variant overlay
├── hooks/
│   ├── post-checkout                   # Auto-pull harness submodule + issue tracker prompt
│   ├── post-merge                      # Auto-pull harness submodule
│   └── pre-push                        # Auto-update touched-areas.md
└── .github/
    ├── workflows/
    │   ├── lint-harness.yml            # Validate docs freshness, cross-links
    │   ├── lint-architecture.yml       # Structural conformance tests
    │   └── doc-gardening.yml           # Scheduled: agent scans for stale docs
    └── PULL_REQUEST_TEMPLATE.md        # Pre-filled by CI, links to exec plans
```

## Component Design

### AGENTS.md vs CLAUDE.md

**AGENTS.md is the map. CLAUDE.md is the personality.**

AGENTS.md (~100 lines) tells any agent where things are. Read by Claude Code, Cursor, Copilot, Codex — anything. Points to deeper sources of truth in `harness/`.

CLAUDE.md tells Claude Code how to behave — tool preferences, style conventions, behavioral rules. This is where "always run tests before PRs" or "use structured logging" lives.

For the attach workflow, the `analyze-codebase` skill generates both as drafts; the human refines.

### Harness as Submodule

For the attach workflow (bolting onto an existing repo), `harness/` becomes a git submodule pointing to a separate repo.

**Rules:**
- Harness repo is **main-only, always linear, strict rebasing.** No branches, no PRs against it.
- Agents push directly to main, namespaced to their own area (e.g., exec-plans/active/{branch-name}/).
- Conflicts are rare (different branches write to different directories). When they occur, rebase and push.
- Consumers always `git submodule update --remote` to get latest.

**Cross-branch awareness:**
Each feature branch maintains `touched-areas.md` listing files/domains being modified. Before starting work, agents check other branches' touched-areas for overlap and flag conflicts to the developer:

> "Heads up: feature/payment-refactor is also modifying src/services/billing/. Their plan says they're restructuring the invoice model. You may want to coordinate."

**Lifecycle:**
- Merged branches: CI/hook moves plans from active/ to completed/
- Stale branches: doc-gardening agent flags them

### Adoption Tiers

**Tier 1: Passive (zero effort)**
Developer changes nothing. Git hooks silently:
- Track changed files per branch -> auto-maintain touched-areas.md
- On PR merge, CI auto-generates branch summary in completed/
- Doc-gardening agent maintains harness health

**Tier 2: Lightweight (minimal effort)**
Developer optionally:
- Jots ideas via `/idea` skill (one command)
- Writes a short plan.md when starting a branch
- Reviews auto-generated summaries before merge

**Tier 3: Full agentic**
Full exec plans, agent-to-agent reviews, quality score maintenance, the whole system.

**Tier 1 is the default.** Hooks install once, everything else is additive.

**Git hooks must be fast (< 2 seconds), never block a push.** They warn, suggest, append — never reject.

### Issue Tracker Integration

Generic interface, not tracker-specific. The core defines:

**Contract — what an integration must provide:**
- Ticket ID parsing from branch name (regex pattern, configurable)
- Ticket data: title, description, acceptance criteria
- Linked tickets / epic context
- Status

**On branch creation:**
1. Parse branch name for ticket pattern
2. Hit tracker API -> pull context
3. Offer to generate exec plan from ticket context
4. If declined, save ticket context for passive reference

Atlassian (JIRA) is a variant implementing this contract via Atlassian MCP. Other trackers (Linear, GitHub Issues, etc.) can be contributed as variants.

### Attach Workflow (Existing Repos)

Hybrid: automated crawl + agent synthesis + human verification.

**Phase 1: Automated Crawl**
Agent runs analysis tools against existing codebase:
- CodeGraphContext MCP — dependency graph, call chains, class hierarchies, dead code
- code-index-mcp — semantic search index
- Built-in analysis — file tree, package deps, CI config, test coverage

Output to `harness/generated/`.

**Phase 2: Agent-Assisted Synthesis**
Agent reads crawl output and drafts harness docs, presenting each for human review:
1. AGENTS.md draft
2. ARCHITECTURE.md draft
3. dependency-rules.md draft
4. golden-principles.md draft
5. tech-debt/index.md draft
6. quality-scores.md draft

Each step is a conversation. Agent proposes, human corrects, agent updates.

**Phase 3: Harness Setup**
Initialize submodule repo, commit verified docs, add submodule, install hooks, configure MCP, generate CLAUDE.md.

**Phase 4: Ongoing Maintenance**
Doc-gardening on schedule, quality re-evaluation, tech debt catalog updates.

### Linter/Enforcement Framework

Stack-agnostic framework; actual lint rules come from variants.

**Core provides:**
- `run-lints.sh` — discovers and runs all active rules
- Harness-specific rules (docs freshness, cross-links, plan structure validation)
- `lint-rule-template.md` — every rule must produce agent-actionable error messages with remediation instructions

**Lint error convention:**
```
BAD:  "Import violation in billing/service.ts"
GOOD: "Import violation in billing/service.ts:12 — Service layer cannot
       import from UI layer. See harness/reins/architecture/dependency-rules.md.
       Fix by extracting the type to billing/types/"
```

### CI Recovery

Generic protocol for agents to diagnose and fix CI failures automatically.

**Decision tree (recovery-protocol.md):**
1. Identify failed job, pull logs
2. Classify failure (lint, test, build, deploy, infra/flaky)
3. Read relevant handler for that failure type
4. If flaky/infra -> re-trigger, flag for tech-debt
5. If real failure -> diagnose, attempt fix, push, verify
6. If fix fails -> escalate to human with diagnosis context

**Platform files** teach agents how to interact with specific CI systems (fetch logs, re-trigger, read artifacts). GitHub Actions provided; others contributed as variants.

### Variant/Template System

```json
// variants/typescript-node/manifest.json
{
  "name": "typescript-node",
  "description": "TypeScript + Node.js web applications",
  "append": {
    "CLAUDE.md": "claude-md-additions.md",
    "harness/reins/golden-principles.md": "golden-principles-additions.md"
  },
  "copy": {
    "linters/": "linters/rules/typescript/",
    "structural-tests/": "linters/structural-tests/typescript/",
    "references/": "harness/reins/references/"
  },
  "hooks": {
    "pre-push": "hooks/pre-push"
  }
}
```

Applied via `./scripts/apply-variant.sh typescript-node`. Idempotent. Multiple variants composable.

Contributing: create directory under variants/, add manifest.json, submit PR. Low bar — can start with just a claude-md-additions.md.

### Ideas Space

```
harness/reins/ideas/
├── index.md          # Auto-generated catalog
├── {username}/       # Per-developer namespace
└── shared/           # Team-wide, unattributed
```

Capture via `/idea` skill — one command, agent formats, files, pushes.

End-of-sprint digest collects each developer's ideas, branch summaries, encountered debt. Delivered as summary for review: keep, plan, or discard.

### README.md

The template repo's front door. Must cover:
- What this is and the philosophy behind it
- Early-stage / fluid / take what you need
- Quick start for greenfield and attach
- Component guide with rationale for every design decision
- Adoption guide (the three tiers)
- Contributing guide (variants, lint rules, CI handlers, ideas)

## References

- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/) — primary inspiration
- [CodeGraphContext MCP](https://github.com/CodeGraphContext/CodeGraphContext) — codebase graph indexing
- [code-index-mcp](https://github.com/johnhuang316/code-index-mcp) — semantic code search
- [Anthropic: Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
