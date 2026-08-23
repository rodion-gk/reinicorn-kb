---
type: idea
title: Feature mood board — Agent OS, BMAD, SpecKit, govctl
slug: 2026-05-04-feature-mood-board-from-agent-os-bmad-speckit
lifecycle: active
status: open
created: 2026-05-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Feature mood board — Agent OS, BMAD, SpecKit, govctl

## Description

Comparison of Reinicorn against the other harnesses in this space. Reinicorn's
positioning is deliberately distinct (team harness, not synthetic-team
simulator — see `project_reins_positioning` memory), but several features in
the others are worth borrowing. This is a research/inspiration doc, not a
commitment to implement any of these.

Originally captured 2026-05-04 covering Agent OS, BMAD, and SpecKit. Revised
2026-07-30: added govctl, refreshed the SpecKit findings against its current
README (it has moved on considerably), and added the two sections that were
missing — what Reinicorn does that none of them do, and a ranked adopt list.
Tool names updated to current `rcorn`/Reinicorn naming throughout; the original
doc said `reins`.

## The four tools at a glance

| | **Agent OS** | **BMAD** | **SpecKit** | **govctl** |
|---|---|---|---|---|
| **Mental model** | Standards-first single-agent | Persona-driven agile | Spec-as-executable governance | Governance-as-code, phase-gated |
| **Anchor doc** | `mission.md` + `standards/` | PRD + Architecture | `constitution.md` | RFC-0000 (governs the framework itself) |
| **Unit of work** | Spec shaped by extracted standards | User story passed between personas | Spec → Plan → Tasks artifacts | RFC → work item → guarded impl |
| **Storage** | Markdown | Markdown | Markdown in `specs/` | TOML SSOT in `gov/`, markdown rendered + SHA-256 signed |
| **Multi-agent** | No | Yes (Analyst/PM/Architect/SM/Dev/QA…) | No (agent-agnostic) | No |
| **Language** | — | Node | Python (`uv tool install specify-cli`) | Rust (`cargo install govctl`) |
| **Sources** | github.com/buildermethods/agent-os | github.com/bmad-code-org/BMAD-METHOD | github.com/github/spec-kit | github.com/govctl-org/govctl |

govctl is the closest of the four to Reinicorn's actual design — CLI-enforced
phases, decisions separated from specs, dogfooded on itself, no MCP and no
vector database. It had never come up in prior research before 2026-07-30.

## What Reinicorn does that none of them do

Worth writing down, because it is easy to lose sight of while cataloguing other
people's features.

1. **The kb is a separate repo, shared across projects and teams.** Everyone
   else writes their artifacts into the project directory. SpecKit's docs do
   not address teams, review, or multi-repo use at all. One kb serving many
   repos, so domain knowledge sits in one place every agent can reach, is
   unique to Reinicorn among all four.
2. **A human review checkpoint on intent.** `rcorn review` opens a single-doc
   PR for a spec, tracks review status in frontmatter, and has a
   solo-maintainer gate and a cleanup workflow. None of the other four has any
   review workflow — specs are artifacts you generate and then implement
   against. Validating intent before implementation exists is the actual bet.
3. **Enforcement that is not a prompt.** SpecKit's checks (`analyze`,
   `checklist`, `clarify`) are slash commands: the agent runs them if it feels
   like it, and a model that skips one produces no signal. Reinicorn enforces
   through git hooks (`post-checkout`, `post-merge`, `pre-push`), editor hooks
   (`block-raw-kb-git.sh`, `enforce-doc-templates.sh`), and a linter framework
   with rules and structural tests — all outside the model's control. govctl
   gets partway there with `govctl check`; SpecKit does not try. **This is the
   clearest place Reinicorn is simply better.**
4. **Lifecycle past "implement."** Ideas, decisions, retros, principles
   (`rcorn principle add`). SpecKit ends at `implement`/`converge` — nothing
   captures what was learned or why a choice was made. govctl covers the "why"
   with ADRs but has no retro.
5. **Provenance as a property, not a convention.** Every doc comes from a
   template with structured frontmatter, and `docmeta`/`manifest`/`validation`
   enforce it. The others' templates produce a starting shape; nothing checks
   the result stayed in shape.
6. **AXI output surface.** The CLI's output format is itself a reviewed spec,
   designed to be parsed by agents and read by humans. The others have
   conventional human-facing CLI output.

The honest summary: Reinicorn is ahead on the human-in-the-loop parts — review
lane, shared knowledge, retros, hard enforcement — and behind on distribution:
extensions, agent breadth, and templates that update themselves.

## Ranked: what to adopt

Broken out as standalone idea docs, most urgent first.

1. **[[templates-should-ship-in-the-tool-not-the-user-s-kb-from-spe]]** — the
   only item on this list that gets *harder* to fix by waiting, because moving
   where templates live is a migration for every existing kb. Do this one
   first.
2. **[[converge-assess-the-codebase-against-specs-and-append-the-re]]** — the
   only answer any of the four has for "the code drifted from the spec," and
   the missing piece for brownfield adoption.
3. **[[guards-as-first-class-verification-artifacts-attached-to-pla]]** — the
   one place govctl goes further than Reinicorn's already-strong enforcement:
   per-work-item declared gates rather than global infrastructure.
4. **[[extension-and-preset-discovery-as-a-cli-surface-from-speckit]]** — the
   biggest capability gap versus SpecKit, but addable later without a
   migration, so it is on the list rather than at the top of it.

Two further items keep surfacing from more than one tool and deserve idea docs
of their own if they are ever picked up: **cross-artifact `analyze`** (SpecKit
`/speckit.analyze`, govctl `check`) and **a clarification/rigor gate before
planning** (SpecKit `/speckit.clarify`).

## Features worth borrowing

### From SpecKit

Refreshed 2026-07-30 against the current README — SpecKit has grown a
substantial ecosystem since the original capture.

- **Broad agent/IDE coverage out of the box.** 30+ integrations, both CLI and
  IDE-based, with `specify integration list` and both slash-command and
  agent-skills modes. Reinicorn ships three:
  `platform-instructions/{claude,copilot,cursor}.md`. Borrow: a per-agent
  template injection system so `rcorn init` produces the right command files
  for whatever agent the user runs.
- **Cross-artifact `/speckit.analyze` consistency check.** A formal step
  auditing spec/plan/tasks for drift before implementation. Reinicorn has
  structural lints for kb shape but no semantic check that exec-plan goals
  match progress.md tasks match the actual diff. Worth a `rcorn plan analyze`
  command. Note the distinction from converge: analyze compares documents to
  each other, converge compares documents to code.
- **`/speckit.converge`.** Assess the codebase against the specs and append
  remaining work. Nothing in Reinicorn does this — broken out as its own idea.
- **`/speckit.checklist`.** Custom quality checklists validating requirements.
  Overlaps the guards idea; the difference is that a checklist validates a
  *requirement's* quality while a guard validates *completion*.
- **`/speckit.taskstoissues`.** Tasks straight to GitHub issues. Reinicorn
  touches GitHub for review PRs and `feedback` but never projects plan tasks
  outward. Low priority, obvious once the plan format is stable.
- **Optional `/speckit.clarify` step.** Forces underspecified areas resolved
  before planning proceeds. Reinicorn exec-plans have no rigor gate — a
  clarification pass could be added to `rcorn plan create`.
- **Constitution as first artifact.** SpecKit creates `constitution.md` before
  anything else. Reinicorn has `golden-principles.md` but does not foreground
  it during init the same way. Could be promoted earlier in the setup flow.
- **`specify self check`.** A pre-flight verifying the install is healthy.
  Reinicorn has `rcorn status` and `rcorn kb status` for kb health — could
  expand to verify hook installation, agent template freshness, and manifest
  validity.
- **Extensions, presets, and bundles as first-class CLI surface**, with a
  documented resolution order (project overrides > presets > extensions > core).
  Broken out as its own idea. Note that the `extensions/stacks/` and
  `apply-variant.sh` the original 2026-05-04 capture compared against no longer
  exist in the repo — there is currently no customization surface at all.

### From Agent OS

- **Context-aware standards injection.** Instead of dumping the whole
  principles doc into context, only the standards relevant to the current
  spec/branch are surfaced. Reinicorn's `golden-principles.md` is currently
  all-or-nothing. Tagging principles by domain and injecting only the relevant
  subset would scale better as the list grows.
- **Standards extraction from existing code.** Agent OS's "Discover Standards"
  stage scans the codebase for patterns and turns them into documented
  standards. Reinicorn's `/analyze-codebase` skill does some of this but
  doesn't produce lintable rules. Worth pairing extraction with
  promotion-to-linter. Note this is the same underlying operation as SpecKit's
  converge, pointed at standards instead of specs.
- **Profile system.** `profiles/default/global/` holds per-agent config layers.
  Reinicorn has no notion of agent profiles. Could be useful as agent coverage
  broadens, and folds naturally into the extension-layering idea.

### From BMAD

Most of BMAD's distinctives (personas, party mode, persona-handoff workflow)
are explicit non-goals per the positioning memory. But a few mechanics are
still worth borrowing:

- **`npx bmad-method install`.** Zero-friction install with
  `--directory --modules --tools --yes` flags. Reinicorn requires
  `uv tool install` from git, which is fine for Python users but creates a
  dependency. A platform-agnostic installer (npx, curl-pipe-bash, or Homebrew)
  would lower the bar. Note SpecKit and Reinicorn have the same install story
  (`uv tool install`), so this is not a competitive gap so much as a shared one.
- **Greenfield vs. brownfield distinction.** BMAD scale-adapts its workflow to
  project complexity and explicitly handles both new and existing projects.
  govctl does the same with migration tooling for brownfield adoption.
  Reinicorn's `rcorn init` does not surface the distinction. Worth promoting to
  top-level init modes — and closely related to converge, which is what
  brownfield adoption actually needs.
- **`@bmad-help` skill.** A first-class help skill that explains the framework
  conversationally. Reinicorn has the README, `GETTING-STARTED.md`, CLI
  `--help`, and the `using-reinicorn` skill — this is closer to covered than it
  was in May, but the skill is oriented at agents rather than at a human asking
  "what does this kb directory do."
- **Trigger codes (CP, CA, DS, QD…).** Short two-letter aliases for common
  workflows. Lower priority but a nice ergonomic touch for power users.

### From govctl

New 2026-07-30.

- **Guards as first-class artifacts.** `GUARD-*.toml` files as declared,
  executable completion gates attached to work items. Broken out as its own
  idea — the single most interesting thing govctl has.
- **Structured SSOT with rendered, signed markdown.** Artifacts are TOML;
  markdown is generated by `govctl render` and SHA-256 signed so hand-edits are
  detectable. This makes artifacts machine-queryable and tamper-evident.
  Reinicorn deliberately keeps markdown as the source and puts rigor in
  templates, frontmatter, and lints — the opposite trade (human-editable, less
  verifiable). Not obviously worth switching, but the *signature* idea is
  separable from the TOML idea: a checksum over generated sections would catch
  hand-edits to generated content without giving up markdown as the source.
  Worth considering alongside [[registry-driven-doc-types]].
- **Hard phase gates.** SPEC → IMPL → TEST → STABLE, phases cannot be skipped,
  and `govctl check` fails if you implement against a draft RFC. Reinicorn's
  lifecycle is the same shape but softer — the review lane gates specs, and
  nothing gates implementation against an unreviewed spec. The
  `feat-enforce-review-lane` work is the closest active equivalent.
- **Schema versioning with migrations.** govctl is on schema v5 with migration
  tooling. Reinicorn has `upgrades/` but no versioned doc schema. This becomes
  necessary the moment there is more than one kb in the world that needs a
  format change — related to the template-propagation problem.
- **`govctl tui`.** A read-only interactive dashboard over the artifact set.
  Reinicorn's `rcorn kb status` is the same information as a flat report.
  Probably not worth building, but a useful reminder that the kb is now large
  enough (150 stale docs as of 2026-07-30) that browsing matters.

## Cross-cutting themes

Patterns showing up across all four projects:

1. **Top-level CLI verb for every artifact type.** SpecKit: `specify`, `plan`,
   `tasks`. BMAD: `*pm`, `*architect`, `*sm`. govctl: `rfc new`, `adr new`,
   `work new`. Reinicorn's CLI is already shaped this way (`idea`, `plan`,
   `retro`, `principle`) but could fill in gaps (`rcorn decision add`,
   `rcorn debt add`).
2. **First artifact is governance.** Constitution / standards / mission /
   RFC-0000 all come before any feature work, and govctl goes furthest by
   governing itself with its own RFCs. Reinicorn should consider whether
   `rcorn init` should *require* a pass through `golden-principles.md` and
   `DESIGN.md` rather than leaving them as templates.
3. **Templates ship inside the tool, not the repo.** Now the top-ranked adopt
   item — see [[templates-should-ship-in-the-tool-not-the-user-s-kb-from-spe]].
4. **Everyone is converging on the same lifecycle.** spec → plan → implement →
   verify, with decisions recorded separately from specs. Four independent
   projects landed on the same shape, which is decent evidence the shape is
   right. The differentiation is entirely in *enforcement* and *where the human
   checkpoint sits* — which is exactly where Reinicorn has chosen to be
   distinct.

## Explicit non-goals (per positioning memory)

Do **not** borrow from BMAD:

- Agent personas (PM-agent, Architect-agent, QA-agent, etc.)
- Multi-agent orchestration / "party mode" / role-played debates
- Anything that replaces a human role rather than augmenting it

Added 2026-07-30: SpecKit's **bundles** ("role-oriented setups so a whole team
persona can be provisioned with one command") are adjacent to this non-goal.
Package the workflows, not the roles.

## Notes

- 2026-05-04 — original capture: Agent OS, BMAD, SpecKit.
- 2026-07-30 — added govctl, refreshed SpecKit against its current README,
  added the Reinicorn-unique and ranked-adopt sections, broke the top four
  items out into standalone idea docs, and updated `reins` → `rcorn` naming.
