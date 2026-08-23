---
type: spec
title: 'Markdown linting: a docs/markdown rule and a config baseline'
slug: markdown-linting-a-docs-markdownlint-rule-and-a-config-basel
lifecycle: active
status: approved
created: 2026-07-26
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/6
---

# Markdown linting: a docs/markdown rule and a config baseline

Closes #26.

> **Revised 2026-07-28.** The original draft specified `markdownlint-cli2`.
> This revision switches the tool to [rumdl](https://github.com/rvben/rumdl) and
> re-measures everything against it. The rule is renamed `docs/markdown` so the
> lint surface does not carry a vendor name. All numbers below were measured
> against the same 206 git-tracked markdown files (repo + kb, excluding
> `presentation/`), and re-verified 2026-07-29. Note that this spec is itself
> in the corpus it measures, so the default-config total drifts by a few counts
> as the doc is edited.

## Problem

Markdown is this project's primary artifact — the kb, the skills, every doc
surface — and nothing lints it. There is no markdown lint config, no lint rule,
and no tool installed. The only thing reviewing markdown today is CodeRabbit:
external, advisory, and PR-only.

The gap surfaced on `reinicorn-kb#5`, where CodeRabbit flagged a fenced block
with no language (`MD040`) that no local check would have caught.

`linters/rules/scripts/shellcheck.sh` already establishes the pattern for an
external linter — a `.sh` rule taking `$PROJECT_ROOT`, emitting
`path:line — message`, and skipping with exit 0 when the tool is absent.
Markdown has no equivalent.

## Tool choice: rumdl, not markdownlint-cli2

Both tools were run against the same 206 files with an equivalent tuned config.

| | markdownlint-cli2 0.23.2 | rumdl 0.2.44 |
| --- | --- | --- |
| Violations (tuned) | 926 | 912 |
| Of which identical (`file:line:rule`) | 844 | 844 |
| Fixed automatically (`--fix` / `rumdl fmt`) | 810 of 926 (87%) | 851 of 912 (93%) |
| Left for a human after `--fix` | 116 | 61 |
| Wall clock, cold | 3.9 s (via `npx`) | 0.12 s |
| Runtime dependency | node + npm | none (static binary) |
| Install | `npm i -g` / `npx --yes` | `pip install rumdl` |
| Rules | markdownlint set | same IDs, 79 rules (superset) |
| License | MIT | MIT |

**The two tools substantially agree.** 844 of ~920 findings are identical down
to file, line, and rule ID. The divergences are small and explained:

- **68 `MD022` reported only by markdownlint.** These are headings immediately
  followed by a list. markdownlint reports the same defect twice — once as
  `MD022` (no blank line below heading) and once as `MD032` (list not
  surrounded by blanks). rumdl reports it once, as `MD032`. Same defect, less
  duplicate noise.
- **58 `MD076` and 1 `MD077` reported only by rumdl.** These are rumdl-only
  rules (list item spacing, list continuation indentation) with no markdownlint
  equivalent.
- **3 `MD057`** (relative links to files that do not exist) found only by rumdl.
  These are real broken links.
- Residual single-digit differences in `MD005`, `MD009`, `MD010`, `MD037`,
  `MD038`.

**The install story is the deciding argument.** Reinicorn is a Python project
distributed on PyPI. It already ships `ruff` — a Rust linter distributed as a
pip wheel — in `[dependency-groups] dev`. rumdl has exactly that distribution
model, so adopting it is a one-line dependency addition with a locked version in
`uv.lock`. markdownlint-cli2 requires node, which Reinicorn does not otherwise
need, which is why the original draft had to design around `npx --yes` and an
absent-tool skip as the *primary* path rather than a fallback.

**Rule IDs are markdownlint's**, so the decision is cheap to reverse. rumdl also
reads `.markdownlint.json`/`.yaml` directly and ships `rumdl import` to convert
configs. Switching back later is a config translation, not a rewrite.

### Risk: rumdl is pre-1.0 and effectively single-maintainer

Honest accounting: rumdl is at 0.2.44, labelled beta, MIT, 1,370 stars, and one
maintainer holds 2,929 of ~2,980 commits. It is actively developed — five
releases in the week ending 2026-07-28 — but a single-maintainer pre-1.0
dependency in the lint path is a real risk.

It is an acceptable one here because the rule is registered at `warning`, the
tool is not on any critical path, and the shared rule IDs make a fallback to
markdownlint-cli2 a config change. Pin the version in `uv.lock` and treat
upgrades as deliberate.

### Measurements

rumdl 0.2.44 over 206 markdown files:

| Config | Violations |
| --- | --- |
| Default | 2,933 |
| Tuned (below) | 912 |
| — auto-fixable via `rumdl fmt` | 851 |
| — needing human judgment | 61 (58 `MD076`, 3 `MD057`) |

Top offenders under the default config:

```text
1813  MD013/line-length
 452  MD032/blanks-around-lists
 197  MD036/no-emphasis-as-heading
 172  MD031/blanks-around-fences
 113  MD040/fenced-code-language
  58  MD076/list-item-spacing
  56  MD022/blanks-around-headings
   8  MD033/no-inline-html
```

### The house style is illegal under the default config

The two largest rules fire on deliberate conventions, not defects. `MD036`
(`no-emphasis-as-heading`, 197) hits the `**Bold lead-in.**` paragraph opener
used throughout specs and skills. `MD013` (`line-length` at 80, 1,813) is
unmeetable for tables, code blocks, and long URLs, and at that volume is pure
noise.

Config is therefore a prerequisite, not polish. A rule shipped without one would
report 2,933 violations on a clean checkout and be ignored within a day.

### Reinicorn's own generated stamp fails the lint — and the frontmatter migration already fixes it

`rcorn review start` stamps a bare URL into every gated doc:

```text
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/5
```

That is an `MD034/no-bare-urls` violation (9 across the corpus today), and it
**cannot be fixed in the document** — the next `review push` rewrites the line.
The original draft therefore specified a generator change: emit `<url>` instead,
and teach `docmeta.get_field` to parse both forms.

**That workstream is unnecessary.** Two things were verified on 2026-07-28:

1. rumdl does not scan YAML frontmatter for `MD034`. A URL carried as a
   frontmatter key produces no violation.
2. The in-flight frontmatter work already moves the stamp there.
   `reinicorn#30` deletes `docmeta.py` outright and replaces it with
   `frontmatter.py`, which defines `review_pr`, `approved_by`, and
   `review_cancelled` as frontmatter fields. The migrated corpus on
   `reinicorn-kb#9` confirms it:

```yaml
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/3
---
```

So once those two land, `MD034` on generated stamps disappears with no work from
this spec. No generator change, no dual-form parsing, no test matrix. This spec
gets smaller by waiting rather than by building.

**This is the dependency behind the "wait on the YAML frontmatter work" note on
the review PR.** It is now resolved, in the direction that removes scope.

## Design Goals

- Markdown structure is checked locally, by the same `rcorn kb lint` surface that
  already runs the other rules — not only by an external reviewer on a PR.
- The shipped config produces **zero** violations from the project's own
  documented conventions. A linter that flags house style trains people to ignore
  it.
- The linter is a Python-ecosystem dependency, consistent with `ruff`. No node.
- Reinicorn stops emitting markdown that Reinicorn would reject.
- Landing the rule does not turn `main` red, and does not require an 851-file
  diff to be reviewed alongside it.
- Downstream projects that adopt Reinicorn inherit a working baseline, not 2,933
  violations on day one.

## Design

### 1. Config baseline

Add `.rumdl.toml` at the repo root:

```toml
[global]
disable = ["MD013", "MD025", "MD033", "MD036", "MD041"]
exclude = ["node_modules", ".claude/worktrees", "presentation"]
```

| Rule | Why disabled |
| --- | --- |
| `MD013` line-length | Unmeetable for tables/code/URLs; 1,813 hits |
| `MD025` single-h1 | Frontmatter `title:` + body H1 is the schema's convention (§6) |
| `MD033` no-inline-html | Inline HTML is used deliberately in docs |
| `MD036` no-emphasis-as-heading | `**Bold lead-in.**` is house style |
| `MD041` first-line-heading | Not all fragments begin with a heading |

**The disable list shrinks from seven rules to five.** The original draft
disabled `MD060`, `MD024`, and `MD029`. Under rumdl, `MD060` is opt-in (not in
the default set) and `MD024`/`MD029` produce zero hits on the current corpus.
Disabling a rule that never fires is dead config; leave them on and revisit if
they ever trigger. `MD025` is new — it is not needed today, but becomes needed
the moment the frontmatter migration lands.

**The exclude list shrinks too.** rumdl honours `.gitignore` by default
(`respect-gitignore = true`), which already covers `.venv/` and `htmlcov/`.
Only `node_modules/`, `.claude/worktrees/`, and `presentation/` — which are not
gitignored — need naming.

Ship it as a Reinicorn asset (`assets.py` / `_data/`) alongside
`linters/.lint-config.json`, so `rcorn init` seeds it into downstream projects.

### 2. The dependency

Add `rumdl>=0.2.44` to `[dependency-groups] dev` in `pyproject.toml`, next to
`ruff`. Contributors get a locked version via `uv.lock`; no node, no `npx`, no
network fetch at lint time.

Downstream projects that install Reinicorn from PyPI do **not** inherit dev
dependencies, so the rule must still degrade gracefully there — see below.

### 3. The lint rule

Add `linters/rules/docs/markdown.sh`, modelled directly on
`scripts/shellcheck.sh` — same shape, same contract, same output format:

1. Take `$PROJECT_ROOT` as `$1`, defaulting to the script's grandparent as
   shellcheck does.
2. Resolve the tool into a runner variable: if `command -v rumdl` succeeds the
   runner is `rumdl`; otherwise, if `uv run --no-sync rumdl --version` succeeds
   the runner is `uv run --no-sync rumdl` (treating failure as "not found").
   The same wrapper must be reused for the `check` invocation in step 3 — in
   the fallback path a bare `rumdl` is not on `PATH`. If neither resolves,
   print `rumdl not found — skipping. Install with: pip install rumdl`
   and **exit 0** — matching shellcheck's absent-tool behavior exactly, so
   `rcorn kb lint` never hard-fails for a downstream user who has not installed
   it.
3. Run `check --output-format concise "$PROJECT_ROOT"` via the resolved
   runner. File selection and
   exclusions come from `.rumdl.toml`, not from a `find` invocation — unlike
   `shellcheck.sh`, the rule does not build its own file list. `kb/` is **not**
   excluded; the kb is the main thing worth linting here.
4. Reformat rumdl's concise output into the framework's
   `path:line — [MDxxx] message`. Verified against rumdl 0.2.44 — `concise`
   emits `path:line:col: [MDxxx] message`, rule ID in brackets, identical to
   `text` except that it drops the trailing `[*]` fixable marker:

   ```text
   README.md:77:1: [MD040] Code block (```) missing language
   ```

   becomes:

   ```text
   README.md:77 — [MD040] Code block (```) missing language
   ```

5. Exit 1 if any violation, 0 otherwise.

The runner already discovers `linters/rules/**/*.sh` and derives the rule name
from the relative path, so `docs/markdown.sh` becomes `docs/markdown` with no
runner change.

**The rule is named `docs/markdown`, not `docs/rumdl`.** The lint surface should
name the thing being checked, not the vendor — `scripts/shellcheck` is the
outlier, and given this spec is already swapping one markdown linter for
another, baking the second vendor name into the rule ID would be a mistake.

### 4. Registration at warning severity

`linters/.lint-config.json` gains:

```json
"docs/markdown": { "enabled": true, "severity": "warning" }
```

Warning, not error, so the rule lands green against the existing 912-violation
backlog. Promotion to `error` happens after the cleanup, as a separate change —
the same warning-then-error sequencing `kb/draft-refs` is going through in the
review-lane spec.

### 5. Sequencing: land after the frontmatter migration

This spec has a hard ordering dependency on `reinicorn#30` and
`reinicorn-kb#9`. Landing markdown lint first would mean building the `MD034`
generator workaround described above and then deleting it a week later.

### 6. `MD025` must be disabled — measured against the migrated corpus

One rule fires on the migrated docs and needs a decision here, because the
config baseline is this spec's deliverable.

**`MD025/single-h1`.** Both linters treat a frontmatter `title:` key as the
document's H1. The frontmatter schema deliberately retains the `# Title` H1 in
the body *and* makes `title:` the tooling source of truth, so every doc carrying
both reports "multiple top-level headings". Confirmed by linting five migrated
docs from `reinicorn-kb#9`: three `MD025` violations, one per doc that has both.

The design goal — *the shipped config produces zero violations from the
project's own documented conventions* — settles it. `title:` plus a body H1 is a
documented convention of an approved spec, so `MD025` goes in the §1 disable
list.

**`MD071`** (blank line after frontmatter) was checked and does **not** fire —
the migration already emits the separator. No action needed.

## Non-Goals

- The `rumdl fmt` cleanup sweep over the 851 mechanical violations. It touches
  nearly every markdown file in the repo and kb, and that diff must not be
  reviewed alongside the rule that motivates it. Separate PR, after this one.
  **Caveat for that PR:** rumdl marks all 113 `MD040` violations fixable and
  fixes them by inserting `text` as the fence language, regardless of whether
  the block is bash, json, or output. That is a judgment call being made
  mechanically; the cleanup PR should fix `MD040` by hand and run `rumdl fmt`
  with `MD040` disabled.
- Promotion to `severity: error`. Follows the cleanup, not this change.
- The 61 `MD076`/`MD057` violations needing human judgment — part of the
  cleanup, not of landing the rule.
- Enabling rumdl's opt-in extras (`MD060`, `MD063`, and the rest of the
  `extend-enable` set). Default rules only, for now.
- Prose and grammar linting (Vale, LanguageTool). Structure only.
- `rumdl fmt` as a formatter run in CI or a hook. Lint only.
- Making rumdl a hard runtime dependency of Reinicorn. Dev dependency; the rule
  degrades to a skip for downstream installs.

## Acceptance Criteria

- `rcorn kb lint` runs `docs/markdown` and reports violations in the
  framework's `path:line — message` format.
- With `rumdl` unavailable and no `uv`, the rule prints an install hint and
  exits 0; `rcorn kb lint` still succeeds.
- With the tool available, a file containing a fence with no language is
  reported; a clean file is not.
- The shipped config yields zero violations from `MD036` and `MD013` across the
  existing repo and kb.
- `rumdl` is present in `[dependency-groups] dev` and pinned in `uv.lock`.
- The rule respects `.rumdl.toml` exclusions: no findings from `node_modules/`,
  `.claude/worktrees/`, `presentation/`, or gitignored paths.
- A doc stamped by `rcorn review start` produces no `MD034` violation — met by
  the frontmatter migration, not by code in this spec. Verify, don't build.
- A migrated doc carrying both `title:` in frontmatter and a body `# Title`
  produces no `MD025` violation under the shipped config.
- The rule is registered at `warning` and `rcorn kb lint` exits 0 against the
  current backlog.
- `rcorn init` seeds `.rumdl.toml` into a fresh project.
- Tests cover: tool-present, tool-absent, violations-found, clean, and the
  exclusion set.

## Resolved during revision

- **Does the frontmatter schema carry the Review-PR URL?** Yes. The published
  schema's field list never names it, which suggested a gap — but `reinicorn#30`
  defines `review_pr`/`approved_by`/`review_cancelled` in `frontmatter.py`, and
  the migrated corpus on `reinicorn-kb#9` carries `review_pr:` in the `---`
  block. The implementation went past the spec's field list. The `MD034`
  generator workstream is deleted rather than deferred.

## Open Questions

1. **Does this file get renamed?** The slug still says `markdownlint`. Renaming
   it would break the open review PR's branch; leaving it means the filename
   names a tool the spec rejects. Suggest leaving it and letting the next
   `spec create` naming pass clean it up.
