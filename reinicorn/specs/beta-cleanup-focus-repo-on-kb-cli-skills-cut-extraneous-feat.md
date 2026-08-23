---
type: spec
title: 'Beta cleanup: focus repo on kb CLI + skills, cut extraneous features'
slug: beta-cleanup-focus-repo-on-kb-cli-skills-cut-extraneous-feat
lifecycle: active
status: draft
created: 2026-07-05
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Beta cleanup: focus repo on kb CLI + skills, cut extraneous features

## Problem

The repo carries features and docs that predate or fell out of the current product
direction: legacy bootstrap shell scripts superseded by the CLI, an extension/overlay
system nothing consumes, a standalone Go tool referenced nowhere, dead Python modules,
orphaned and stale hooks, agent-task-dispatch remnants in the kb, 11 dead "active"
exec-plans, and a README describing a shape the project no longer has. This noise
obscures the actual first beta candidate: a **Skill Set + CLI tool for managing an AI
knowledgebase and enforcing a spec-driven development workflow across a team**.

## Design Goals

- Everything left in the repo serves the beta: `reins` CLI, `.claude/skills/` skill
  set, git/editor hooks, linters, platform instructions, upgrade notes, tests, docs.
- Every hook that ships is wired, delegates to tested Python where logic exists, and
  has test coverage.
- Cut features are deleted (git history preserves them) and filed as `reins idea`
  docs so the backlog lives in the kb, dogfooding the tool.
- The kb reflects reality: no dead active plans, superseded specs marked, cleanup
  queue references only code that exists.
- The workflow scaffolds only docs it actually enforces: spec → plan → execution →
  retro. No decisions.md/progress.md stubs that nothing fills.
- Platform support matches the beta audience: Claude Code, Cursor, GitHub Copilot,
  Codex. Gemini support removed. Skills install to the open-standard
  `.agents/skills/` location, with a `.claude/skills` symlink for Claude Code.
- README rewritten around the beta pitch.

## Design

### A. Code repo deletions

1. **`scripts/`** — delete all four (`init-greenfield.sh`, `attach-to-repo.sh`,
   `setup-hooks.sh`, `apply-variant.sh`). All are superseded by `reins init` /
   `reins hooks install`. File one idea doc: "pip-free shell bootstrap path".
2. **`extensions/`** (stacks, atlassian integration, `_template`) plus dead module
   `src/reins/extensions.py` and `tests/test_extensions.py`. File idea doc:
   "extension/overlay system (stacks + integrations)".
3. **`tools/block-dangerous-commands`** (Go tool, referenced nowhere). File idea doc.
4. **Legacy loose skills** in `.claude/skills/`: `analyze-codebase.md`,
   `publish-plan.md`, `sync-context.md`, `update-kb.md`. Old single-file format,
   referenced nowhere; publish/sync are CLI commands now. File one idea doc for an
   `analyze-codebase`-style attach workflow if worth reviving.
5. Sweep stray references to any of the above (README, GETTING-STARTED, AGENTS.md,
   `doc_create.py` shared-dir skip list once kb dirs are gone).

### B. Hook consolidation

1. **`hooks/post-checkout`**: replace the 77-line bash reimplementation with a thin
   stub that delegates to `reins _post-checkout` (same pattern as pre-push and
   post-merge). Move the new-branch suggestion into the Python command and update its
   text from the defunct `/create-exec-plan` to `reins plan create`. Keep ticket-id
   detection, reading the pattern via `config_get`.
2. **Tests**: add coverage for `_post-checkout` and `_post-merge` (currently zero;
   only `_pre-push` is tested).
3. **`editor-hooks/block-raw-kb-git.sh`**: wire into `reins hooks install` alongside
   `enforce-doc-templates.sh` for all three editor configs (Claude Code, Cursor,
   Copilot — Bash/terminal tool matcher), and add tests mirroring the
   enforce-doc-templates coverage.

### C. Platform support

1. Remove Gemini: delete `platform-instructions/gemini.md`, drop `gemini` from
   `PLATFORM_FILES`, `PLATFORM_TEMPLATES`, and the init picker; update tests.
2. Add Codex to the init picker. Codex reads `AGENTS.md` natively and init always
   installs AGENTS.md, so selecting Codex installs no extra file — init confirms
   "Codex: uses AGENTS.md". Add a short AGENTS.md note that platforms without a
   skill-invocation tool read skills directly from the skills directory.
3. **Canonical `.agents/skills/` install location.** Codex, Cursor, and Copilot all
   read `.agents/skills/` natively (it is the Agent Skills open-standard location);
   Claude Code still reads only `.claude/skills/`. Therefore:
   - `reins init` and `reins update` install skills to `.agents/skills/` and create
     `.claude/skills` as a relative symlink to it (Claude Code supports a symlinked
     skills dir; Codex/Cursor follow symlinks).
   - On platforms where symlink creation fails (e.g. Windows without developer
     mode), fall back to copying and warn about the duplicate location.
   - Update every reference to the skills path: `SKILLS_ASSET` dest in init,
     update.py, manifest.py, AGENTS.md, platform-instruction templates, README,
     GETTING-STARTED.
   - Convert this repo itself to the same layout (`.agents/skills/` canonical,
     `.claude/skills` symlink) and adjust the pyproject wheel force-include.

### D. Kb cleanup (submodule)

1. Delete `kb/agent-tasks/` and `kb/generated/agent-jobs/` (agent-task-dispatch
   remnants; the feature's code is already gone).
2. Archive dead active exec-plans: move all merged/abandoned plans out of
   `exec-plans/active/` to `completed/` (abandoned ones get a status note). Verify
   each plan's branch merge state via git before archiving; only genuinely
   in-flight work stays active.
3. Mark superseded specs with a `**Status:** superseded` header line (mvp-roadmap,
   agent-task-dispatch, agent-runner-layer, m2–m6 milestone specs,
   team-presentation, harness-to-kb rename docs, and any others describing removed
   or shipped-and-replaced features). No deletions — kb history stays browsable.
4. Prune `tech-debt/cleanup-queue.md`: remove completed rows and rows referencing
   deleted code (`commands/agent.py`, `job.json`, extension manifests); keep real
   items. The internal-hook test gaps (TEST-03/04) are resolved by workstream B;
   publish/sync test gaps (TEST-01/02) stay queued.
5. File the idea docs from workstream A via `reins idea create`.

### E. Workflow doc honesty (decisions/progress/retro)

Audit finding: `plan create` scaffolds `decisions.md` and `progress.md`, and retro
stubs exist in most plan dirs, but the skills that drive the workflow
(writing-plans, executing-plans, finishing-a-development-branch) never reference
them. Result: zero filled-in retros ever; decisions/progress are placeholder stubs
in 13 of 17 plan dirs. Scaffold only what the workflow enforces:

1. **Drop `decisions.md` and `progress.md`** from plan scaffolding and the
   `exec-plans/_template/`. Decisions are recorded in the plan (or spec) where they
   are made; progress is the plan's task checkboxes and Status field. Delete the
   untouched stubs from existing plan dirs during the archive pass (keep the few
   with real content).
2. **Make retro real.** Keep the `reins retro` noun. `finishing-a-development-branch`
   gains a retro step: invoke `reins retro create` and fill What Went Well / What
   Could Be Improved / Lessons Learned / Action Items from the session before
   `reins plan complete`. `plan complete` warns when retro.md is missing or contains
   only empty bullets (upgrade from the current passive "Tip:" hint).
3. Align `retro create` with this flow: it must work on the current branch's active
   plan (the retro travels to `completed/` at archive time) — resolves the existing
   idea doc about retro create only working post-completion.

### F. Docs

1. **README.md**: full rewrite around the beta pitch — Skill Set + CLI for AI
   knowledgebase management and team spec-driven development. Drop sections on the
   extension system, scripts, workflows/doc-gardening CI, MCP servers, adoption
   tiers, and the "7 skills" table; describe the actual skill set and CLI surface.
2. **HELLO.md → HUMANS.md**: replace HELLO.md with a new HUMANS.md, written by
   Michael (content is his; the cleanup just renames the file, carries the existing
   text over as a starting point, and points the README link at HUMANS.md).
3. **GETTING-STARTED.md**: verify commands still match post-cleanup surface (it is
   already CLI-centric; expected minimal changes).
4. `pyproject.toml`: no force-include changes expected (extensions/tools/scripts were
   never shipped in the wheel) — verify.

## Non-Goals

- No new features beyond the Codex picker entry and the `.agents/skills/`
  canonical location — otherwise this is subtraction and repair.
- No renaming of nouns, commands, or directory layout.
- No resolution of open tech-debt items beyond the hook test gaps named above.
- No kb-spec deletions — superseded specs are marked, not removed.
- No changes to the superpowers-forked skills' content.
