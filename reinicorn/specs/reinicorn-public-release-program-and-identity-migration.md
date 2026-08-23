---
type: spec
title: Reinicorn public release program and identity migration
slug: reinicorn-public-release-program-and-identity-migration
lifecycle: active
status: approved
created: 2026-07-15
author: Michael Biehl
origin: ai-assisted
human_validated: true
review_pr: https://github.com/mnbiehl/reins-kb/pull/9
---

# Reinicorn public release program and identity migration

## Problem

The private Reins repository is being prepared for archival and replacement by a
fresh-history, public, MIT-licensed project. The current name cannot be carried into
that release: several unrelated projects already use Reins, including a directly
competing LLM harness, and the Python distribution name is occupied.

The rename also exposes an asset-ownership defect. The source repository's root
`AGENTS.md` is packaged as the generic file copied into downstream repositories.
Populating it for this source repository would leak source-specific instructions to
users; leaving it generic makes the source repository permanently incomplete. The
same downstream file is currently tracked by the update manifest even though it is
intended to become project-specific after initialization.

Renaming alone is not sufficient for public release. The audit in
`public-release-readiness-and-naming-collision-audit.md` (private-only, not published)
also identified licensing, security, documentation, history-sanitization, and
anonymous-clone release gates. These concerns need one stable program contract but
must not be forced into one unreviewable implementation plan.

## Design Goals

1. Replace the Reins identity completely with Reinicorn while exposing `rcorn` as
   the sole CLI executable.
2. Provide no compatibility aliases, deprecated commands, fallback imports, legacy
   configuration discovery, or legacy environment-variable support.
3. Make every downstream root `AGENTS.md` user-owned immediately after it is
   created.
4. Move the canonical KB navigation map into a scope-level `README.md` that GitHub
   renders automatically.
5. Keep package-owned assets updateable without modifying user-owned instructions
   or team-owned KB content.
6. Divide public-release preparation into bounded, sequential gates with explicit
   completion criteria.
7. Export fresh public histories instead of exposing or mirroring the private
   repositories and their deleted or unreachable history.

## Design

### 1. Frozen identity contract

The public identity is fixed as follows:

| Surface | Final identity |
|---|---|
| Product and public brand | Reinicorn |
| Source repository | `reinicorn` |
| KB repository | `reinicorn-kb` |
| Python distribution | `reinicorn` |
| Python import package | `reinicorn` |
| CLI executable | `rcorn` |
| Local state directory | `.reinicorn/` |
| Repository config | `.reinicorn-config` |
| Environment prefix | `REINICORN_*` |
| KB scope | `kb/reinicorn/` |
| Skills, workflows, platform files, and manifest fields | Reinicorn-derived names |

Runtime identity values live in a single `reinicorn.identity` module: product
display name, CLI name, state directory, config filename, and environment prefix.
Python distribution metadata and source-repository coordinates remain sourced from
`pyproject.toml`.

No public artifact may ship a `reins` executable, `reins` Python package, `.reins*`
fallback, `REINS_*` fallback, old skill alias, or deprecated compatibility path.
Historical occurrences are permitted only in the archived private repositories, not
in either fresh public export.

### 2. Instruction and KB ownership

The source file and downstream template become separate assets:

```text
root AGENTS.md
  Reinicorn source-repository instructions; source-owned

templates/AGENTS.md
  Generic downstream scaffold
    -> packaged as reinicorn/_data/templates/AGENTS.md
    -> rendered once by rcorn init

kb/<scope>/README.md
  Team-owned KB entry point and navigation map
```

The generic template contains project-description, build, and convention
placeholders plus a stable instruction to read `kb/<scope>/README.md`. It does not
embed the KB directory table. The population skill fills the project-specific
sections collaboratively and removes its unpopulated marker.

`rcorn init` preserves any existing root `AGENTS.md`. If none exists, it renders the
generic template with the target scope and then relinquishes ownership. Neither
`AGENTS.md` nor any KB file is recorded in `.reinicorn/manifest.json` or considered
by `rcorn update`.

When a KB scope is first seeded, `rcorn init` creates
`kb/<scope>/README.md`. The README links to golden principles, architecture, specs,
PRDs, active execution plans, quality scores, technical debt, and the required KB
commands. It is never overwritten when the scope already contains one and becomes
team-owned after creation.

The Reinicorn source repository gets a populated root `AGENTS.md` only after the
generic template has moved to its dedicated packaged location. This ordering
prevents source-specific content from entering downstream projects.

### 3. Identity migration sequence

Gate 1 is implemented through reviewable checkpoints:

1. Rename `src/reins/` to `src/reinicorn/`, update all imports and tests, update
   package metadata, and expose only the `rcorn` entry point.
2. Rename state, configuration, hooks, workflows, skills, platform files, manifest
   fields, generated messages, and environment variables.
3. Split the root `AGENTS.md` from `templates/AGENTS.md` and remove downstream
   `AGENTS.md` from update management.
4. Add scope-level KB README generation and rename the project KB scope from
   `kb/reins/` to `kb/reinicorn/`.
5. Update current public-facing documentation and run a legacy-identity and
   private-coordinate sweep across runtime code, package inputs, generated assets,
   and release-facing docs. Private historical KB records are handled by the export
   allowlist in Gate 4 rather than rewritten merely to erase history.

There is no runtime migration layer for old checkouts. The private source and KB are
transformed as part of the controlled release work, while users of the future public
project start from the new identity. This follows the pre-release rule against
deprecated aliases and prevents two identities from surviving indefinitely.

### 4. Initialization and update behavior

The final initialization flow is:

```text
rcorn init
  -> create or attach kb
  -> create kb/<scope>/README.md when absent
  -> render templates/AGENTS.md when root AGENTS.md is absent
  -> install managed skills, hooks, linters, workflows, and platform instructions
  -> write .reinicorn/manifest.json containing package-owned assets only
```

Re-running `rcorn init` preserves existing user and KB content byte-for-byte.
`rcorn update` updates only package-owned assets. Missing packaged templates or
malformed generated assets fail with an agent-readable error that names the expected
asset and gives a concrete installation-repair command. Existing user-owned files
produce an informational "keeping existing" message rather than an error.

### 5. Public-release gates

The release program has five sequential gates. This spec is both the umbrella
contract and the implementation contract for Gate 1. Each later gate receives its
own reviewed spec and focused execution plan; later gates do not compensate for
incomplete earlier gates.

#### Gate 1: Identity and asset ownership

Complete the identity migration, instruction/template split, KB README map, source
`AGENTS.md` population, and legacy-identifier removal described above.

#### Gate 2: Licensing and security

- Record authority to MIT-license all original work.
- Ship complete third-party notices and license metadata, including the upstream
  Superpowers license.
- Inventory other copied or adapted assets and record provenance or remove them.
- Validate allowed Git transports before invoking Git.
- Define and enforce the repository-local linter trust boundary.
- Remove GitHub Actions expression-to-script injection paths.
- Minimize workflow permissions and pin third-party actions.
- Add a public security policy and private reporting route.

#### Gate 3: Documentation and KB credibility

- Replace conflicting or unavailable installation instructions with a tested
  source-install path appropriate for a project not published to PyPI.
- Route all KB operations through `rcorn`; remove raw KB Git instructions.
- Align maturity, version, and tagging language.
- Refresh stale KB documents, malformed active-plan data, provenance fields, and
  quality scores, or record an explicit accepted baseline.
- Add contributor documentation and verify every quick-start command from scratch.

#### Gate 4: Sanitized public export

- Build explicit source and KB export allowlists.
- Create new `reinicorn` and `reinicorn-kb` repositories with fresh histories.
- Do not make either private repository public and do not mirror private branches,
  tags, reflogs, deleted objects, or commit metadata.
- Remove private coordinates and identity-specific settings.
- Run secret, PII, large-file, provenance, license, and private-string scans against
  the exported repositories before their first public push.

#### Gate 5: Release rehearsal

- Test clean anonymous recursive clones of both public repositories.
- Test source installation and the complete `rcorn init`, KB sync, KB publish, and
  update workflows without developer credentials or local configuration.
- Run unit, lint, type, structural, shell, packaging, and KB checks.
- Inspect final archives for required licenses and unwanted assets.
- Tag only after every mandatory gate passes and any remaining release risk is
  explicitly owner-approved.

### 6. Gate 1 verification contract

Gate 1 is complete only when all of the following are demonstrated:

- `import reinicorn` and `python -m reinicorn` work.
- The built distribution installs exactly one executable, `rcorn`.
- No `reins` Python namespace or CLI compatibility entry point is shipped.
- Wheel and source archives contain the Reinicorn package, generic AGENTS template,
  skills, hooks, workflows, linters, platform instructions, and required licenses.
- A clean temporary-repository initialization creates a correctly substituted
  `AGENTS.md`, `kb/<scope>/README.md`, and `.reinicorn/manifest.json` that excludes
  both user-owned and KB-owned files.
- Re-running initialization preserves existing root AGENTS and KB README content
  byte-for-byte.
- Updating package assets preserves those two files byte-for-byte.
- Configuration, mode, state, hooks, and platform integrations use only the final
  identity.
- Behavioral tests, Ruff, Pyright, structural tests, package builds, and
  extracted-wheel smoke tests pass.
- A targeted scan of runtime code, package inputs, current release-facing docs, and
  built archives finds no legacy identity or private repository coordinates, and
  finds no Reinicorn source-repository instructions in generated downstream assets.
  Private historical KB records are excluded from this Gate 1 scan and must either
  be omitted or sanitized by the Gate 4 export allowlist.

Dedicated secret/license scanners and anonymous-clone tests remain mandatory in the
later release gates; they do not replace Gate 1's package and initialization tests.

## Non-Goals

1. Publishing Reinicorn to PyPI. The distribution name is reserved in the identity
   contract, but the initial public release uses a tested source-install path.
2. Preserving compatibility for the private Reins CLI, import package, configuration,
   state files, skills, or environment variables.
3. Publishing, rewriting in place, or mirroring the private source or KB repository.
4. Automatically populating downstream `AGENTS.md` without human review.
5. Letting `rcorn update` manage user-owned root instructions or team-owned KB
   content.
6. Implementing all five gates in one branch or one execution plan. Gate 1 is the
   first implementation cycle; later gates receive separate reviewed specs and
   plans.
