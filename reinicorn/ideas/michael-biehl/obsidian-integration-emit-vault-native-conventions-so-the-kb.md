---
type: idea
title: Obsidian Integration — Emit Vault-Native Conventions So the KB Lights Up
slug: obsidian-integration-emit-vault-native-conventions-so-the-kb
lifecycle: active
status: new
created: 2026-07-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Obsidian Integration — Emit Vault-Native Conventions So the KB Lights Up

## Summary

Obsidian is purpose-built for large directories of `.md` files, and `kb/` is
exactly that (121 docs across `specs/`, `ideas/`, `exec-plans/`, `tech-debt/`,
`prds/`, `references/`). This is a research capture — not a spec — of *how*
reinicorn should integrate with Obsidian, to feed a future spec.

**The one-line conclusion:** treat Obsidian as a **human-facing reading / graph
/ dashboard layer over a reinicorn-owned vault**, and make the integration
almost entirely **convention emission with zero runtime dependency on
Obsidian**. Everything Obsidian gives a human (graph, backlinks, queryable
dashboards) is derived from `[[wikilinks]]` and YAML frontmatter *in the plain
files* — which rcorn already emits (or is about to, via the frontmatter
schema). rcorn and AI agents keep touching the files directly; Obsidian is
never a dependency in any code path.

This is congruent with reinicorn's existing design, not a new axis:
- rcorn already owns the vault files and **all** git ops (there is a hook
  actively blocking raw git in `kb/`).
- The axi/agent-native output philosophy already favors structured, parseable
  files — which is exactly what makes Obsidian light up.

**Critical dependency:** this work is almost entirely *downstream* of
[[unified-kb-doc-frontmatter-schema]] (`specs/drafts/`). That draft already
defines `type`, `lifecycle`, `status`, `tags`, `related`, and per-type link
fields — those *are* the queryable fields and the graph edges. Obsidian
integration mostly needs **two or three format decisions inside that schema**,
not new machinery. See "Load-bearing decisions" below; one of them directly
answers that draft's open question #2.

---

## Key finding: Obsidian is human-facing only

Every agent-facing Obsidian mechanism — the Local REST API plugin, all MCP
servers built on it, the new official Obsidian CLI (GA Feb 2026), and the
`obsidian://` / Advanced URI schemes — **requires the Obsidian desktop GUI to
be running**. None is a headless service. For an agent-native, CLI-driven tool
that also runs in CI and worktrees, depending on "Obsidian happens to be open"
is a direct contradiction of the design.

The value Obsidian adds (link graph, Bases queries, backlinks) is derived
**entirely from the conventions in the plain files** — `[[wikilinks]]` + YAML
frontmatter — not from the application. An agent that understands those
conventions reconstructs the same graph by parsing files directly; Obsidian
just pre-computes it for a human GUI.

The heaviest real-world practitioners converge on exactly this split: Andrej
Karpathy's "LLM wiki" (Claude Code pointed at the folder; Obsidian only for
human browsing) and Eleanor Konik's 12M-word vault (plain Filesystem MCP on the
folder, Obsidian as the reading surface). This is the consensus among people
operating agents at vault scale, not a fallback.

**Implication for reinicorn:** do **not** invest in Obsidian's agent-facing
APIs. If anything, the only defensible investment is a thin, read-mostly
compatibility layer so a human teammate *can* open the same folder in Obsidian —
with zero reinicorn code path depending on Obsidian being installed, running,
or reachable.

---

## The surfaces, ranked by value-for-coupling

From most value / least coupling to least:

1. **Properties / YAML frontmatter** — CORE, zero-install. The single
   highest-leverage input: one clean typed frontmatter block feeds the
   Properties UI, search, sort, Bases, and Dataview simultaneously. Note the
   1.9.0 rule: `tags`, `aliases`, `cssclasses` **must be plural, list-valued**.
2. **Wikilinks + `aliases`** — CORE, zero-install. `[[Note]]` (and
   `[[Note#Heading]]`, `[[Note#^block]]`) unlock backlinks, unlinked mentions,
   and Graph View. Obsidian builds the graph **only** from `[[ ]]` syntax — not
   from relative Markdown links (which the kb uses in the ~2 files that link at
   all today).
3. **Bases** — CORE, zero-install (Obsidian ≥1.9). Turns a folder/query into a
   live filterable Table/List/Cards/Map view over frontmatter properties.
   Config is a plain-YAML `.base` file (also readable/editable without
   Obsidian). Needs nothing beyond good frontmatter. **This is the dashboard
   bet.**
4. **Obsidian CLI** (CORE, ≥1.12.7) — bidirectional automation, but
   auto-launches / requires the GUI. Not for us (see key finding).
5. **`obsidian://` URI** — CORE, fine for "open this note" deep links; also
   needs the GUI.
6. **Core Templates** — CORE, minor; only relevant if authoring *inside*
   Obsidian, which we should not do (rcorn owns creation).
7. **Local REST API (+MCP)** — COMMUNITY plugin, powerful but GUI-bound and a
   full-trust security ask. Not a dependency.
8. **Dataview** — COMMUNITY, **dormant** (no significant release in ~a year;
   maintainer on sabbatical). Still huge installed base and still works. Do not
   build against it; support it *passively* by keeping frontmatter clean.
9. **Advanced URI / Templater / QuickAdd** — COMMUNITY power-user niceties;
   optional at most.
10. **Obsidian Git** — COMMUNITY; an **anti-pattern here** (see guardrails).
11. **First-party plugin** — highest ceiling, highest cost (TS maintenance,
    review process, full-trust security ask, unsandboxed). Reserve for
    capability unreachable via 1–3. Almost certainly not worth it.

### The Bases-vs-Dataview call

**Bet on Bases.** It is core (zero install, no trust prompt), reads the same
frontmatter, and — critically — betting on it means **zero community plugins**,
which eliminates the entire plugin-vendoring supply-chain problem (below). Its
only gap vs. Dataview is complex computed/JS queries (DataviewJS, joins across
non-property data), which the kb dashboards do not need. Ship starter `.base`
files for the lanes; leave Dataview as a thing that *also* happens to work for
users who already have it.

---

## Load-bearing decisions (feed these into the frontmatter spec)

These are the format choices that determine whether the graph/dashboards light
up. All three touch [[unified-kb-doc-frontmatter-schema]].

1. **Link-field format — the big one (answers that draft's open question #2).**
   Obsidian draws graph/backlink edges **only** from `[[wikilink]]` syntax,
   including inside frontmatter. The draft currently stores `related:` and the
   per-type link fields (`spec`, `plan`, `promoted_to`, `supersedes`, …) as
   bare slugs/paths. Bare slugs give **no** graph.
   - Complication: kb filenames are date-prefixed (`2026-02-22-…-plan.md`) and
     do **not** equal the `slug`, so `[[slug]]` would not resolve to the file.
   - **Recommended reconciliation:** emit link fields as `[[slug]]` **and** put
     `aliases: [<slug>]` in every doc's frontmatter. Obsidian resolves
     `[[slug]]` via the alias table → real graph edges, backlinks, and
     hover-preview, while rcorn's own tooling still parses the slug out of the
     `[[…]]` wrapper trivially. This keeps a single source of truth and makes
     both consumers (rcorn + Obsidian) happy.
   - Portability note (the "Foam discipline"): keeping links as resolvable
     wikilinks + aliases is the portable choice; avoid Obsidian-only link
     syntax so the files stay clean in plain Markdown tools too.

2. **Plural list properties.** `tags: []` (already in the draft) and the new
   `aliases: []` must be plural lists — required since Obsidian 1.9.0.

3. **Which fields are the dashboard axes.** `type`, `lifecycle`, `status`,
   `severity` (tech-debt), `created`/`updated` are exactly the Bases
   filter/sort/group axes. The draft already carries all of them — so the four
   canonical lanes (open specs, plans in flight, tech-debt by severity, ideas
   awaiting triage) are near-free once the schema lands.

---

## Integration architecture options

- **A — Pure convention (recommended core).** rcorn emits good frontmatter +
  `[[wikilink]]`/`aliases` (mostly via the frontmatter schema work). No
  reinicorn code depends on Obsidian. A human who opens `kb/<repo>` in Obsidian
  gets graph + backlinks + search for free. Lowest cost, lowest risk, fully
  agent-native. This is the floor and probably 80% of the value.

- **B — Ship a curated read-only vault (recommended enhancement).** On top of
  A, rcorn generates/vendors a small, **core-only** Obsidian setup per
  namespace: starter `.base` files for the lanes, a `_dashboard.md` hub note,
  and an allowlisted `.obsidian/` (theme + core-plugins config, **no community
  plugins**). Could be a `rcorn obsidian init` command or part of `rcorn init`.
  Prior art: **ArchitectKB** ships exactly this shape (`.obsidian` + dashboards
  + named query notes) — dev-doc/ADR tooling is otherwise an Obsidian gap
  reinicorn could be a first-mover in filling.

- **C — First-party Obsidian plugin (rejected).** High cost, full-trust
  security ask, unsandboxed, GUI-bound. No capability we need is unreachable
  via A+B.
  - *Update 2026-07-27:* one candidate capability has since been identified —
    commenting on **rendered** markdown and posting to a GitHub PR, which cannot
    be emitted as a file convention. See
    [[rendered-markdown-pr-review-via-obsidian]]. The conclusion still holds
    (existing third-party plugins cover it; do not write one), but this is the
    reserved exception firing.

- **D — Agent-facing APIs / MCP / REST / CLI (rejected).** All GUI-bound;
  contradicts agent-native. Agents read the folder directly.

---

## `.obsidian/` git handling

If we commit any Obsidian config into the kb submodule (so a team shares it),
use the **allowlist** strategy and never commit UI state:

```gitignore
# Ignore everything under .obsidian, then allowlist shared, stable config
.obsidian/*
!.obsidian/app.json
!.obsidian/appearance.json
!.obsidian/core-plugins.json
!.obsidian/community-plugins.json
# Never commit UI/layout state (top source of merge conflicts):
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache/
.obsidian/updates.json
.trash/
```

Notes and gotchas:
- **`workspace.json` / `workspace-mobile.json` — always ignore.** Highest-
  confidence finding across every source; pure pane/layout state, the #1 cause
  of `.obsidian` merge conflicts.
- **`graph.json` is not safe either** — live zoom/pan state is serialized into
  it and the maintainer declined to fix it; expect noise. Prefer leaving graph
  settings out.
- **Committing community-plugin folders = vendoring executable, unsandboxed JS**
  into the shared repo (full filesystem + network privilege on next launch,
  pulled silently via `git`). This is a real supply-chain surface. **Choosing
  Bases (core) avoids it entirely** — no community plugins to vendor, so the
  allowlist can stop at core config.
- `community-plugins.json` only lists enabled plugin *ids*; it does not install
  anything. (Moot if we stay core-only.)
- The current vault already has a fresh, empty `.obsidian/` on `kb/reinicorn/`
  (default config, no plugins) — a clean starting point.

---

## Guardrails / anti-patterns (non-negotiable)

- **Obsidian Git plugin — forbid / disable.** Two independent git automations
  (Obsidian Git's auto-commit vs. rcorn) on one working tree = races, dirty
  worktrees, interleaved commits — and it fights the existing block-raw-kb-git
  hook. rcorn must remain the only writer.
- **Templater / QuickAdd / in-Obsidian note creation — forbid.** Creating docs
  inside Obsidian bypasses rcorn templates and provenance stamping, violating
  "all harness docs are created via the CLI from templates." Obsidian stays
  **read-mostly**; authoring goes through rcorn.
- **External-write staleness.** rcorn writes files while Obsidian is open;
  Obsidian's file watcher can show stale content until the tab is
  refocused/reopened (worse under Flatpak / network drives). Options: document
  a manual reload, or accept it. (A community "Vault File Refresh" plugin exists
  but reintroduces the plugin-vendoring problem — prefer documenting over
  depending.)
- **Concurrency / worktrees.** Multiple Obsidian instances against one working
  tree race on `.obsidian` state files. Relevant to reinicorn's worktrees — the
  informal fix is a separate vault/clone per concurrent opener rather than
  sharing one working directory. Worth a decision when worktree + vault use
  overlap.

---

## Open questions for the future spec

1. **Vault scope.** Is the Obsidian vault the whole `kb/` (cross-repo graph
   spanning every `kb/<repo>` namespace — richer graph, but cross-repo noise)
   or per-namespace `kb/<repo>` (current setup; cleaner, no cross-repo edges)?
2. **Link format ratification.** Confirm the `[[slug]]` + `aliases: [slug]`
   reconciliation in the frontmatter schema (its open question #2), or choose
   an alternative (bare slugs + a rcorn-side graph export; or matching filenames
   to slugs). This is the single decision that gates the graph.
3. **How far to go past convention (A vs A+B).** Do we ship starter `.base`
   files + a dashboard note + allowlisted core `.obsidian`, and via what command
   (`rcorn obsidian init` vs. folded into `rcorn init`)? Committed to the kb
   submodule (shared) or generated locally (per-user)?
4. **Which lanes get dashboards**, and their exact Bases queries (open specs;
   plans in flight by `lifecycle`; tech-debt by `severity`; ideas awaiting
   triage). Draft the `.base` definitions.
5. **Staleness + worktree concurrency stance** — document-only, or tooling?
6. **Migration coupling.** The graph only lights up once existing docs carry
   frontmatter + aliases + wikilink-format links — so this rides the frontmatter
   migration; sequence accordingly.

---

## Prior art worth mining

- **ArchitectKB** (`DavidROliverBA/ArchitectKB`) — a dev-doc vault that ships
  `.obsidian` + Dataview/Templater config, dashboard hub notes, and named query
  notes. Closest structural template for option B. (Maturity unverified; treat
  as an example, not a standard.)
- **Foam** — editor-agnostic; keeps links portable by avoiding Obsidian-only
  syntax and auto-emitting link-reference-definitions. Good discipline model.
- **Quartz / obsidian-digital-garden** — frontmatter-flag publish gating
  (`publish: true` / `dg-publish: true`) — a pattern to borrow if the kb ever
  wants selective external publishing.
- **Dev-doc / ADR / spec tooling is an Obsidian gap** — the canonical ADR-tool
  index lists zero Obsidian/Dataview integrations. Reinicorn filling this well
  is genuinely novel positioning.

---

## Recommended MVP (for the spec to confirm)

1. Land the frontmatter schema with the **`[[slug]]` link format + `aliases:
   [slug]`** decision baked in (option A — pure convention). This alone lights
   up graph, backlinks, search, and Properties for any human who opens the
   vault, with zero Obsidian dependency.
2. Then (option B) ship a **core-only** curated setup per namespace: starter
   `.base` dashboards for the four lanes + a `_dashboard.md` hub + an
   allowlisted `.obsidian/` — no community plugins, so no plugin-vendoring risk.
3. Document the guardrails (no Obsidian Git, no in-Obsidian authoring,
   reload-on-external-write) in the vault README.
4. Invest **nothing** in Obsidian's agent-facing APIs.

---

## Sources

Captured from four parallel research passes (2026-07-22). Full URL lists live in
the session transcript; key primary sources:

- Obsidian Properties / 1.9.0 plural rule, Bases, CLI, plugin security —
  `obsidian.md/help`, `docs.obsidian.md`, `obsidian.md/changelog`.
- Dataview dormancy — `github.com/blacksmithgu/obsidian-dataview/releases`;
  "Dataview is dead, long live Bases" (Medium).
- Local REST API + MCP, agent patterns — `github.com/coddingtonbear/obsidian-local-rest-api`,
  ContextBolt Obsidian-MCP guide, Eleanor Konik, Karpathy "LLM wiki" writeups.
- `.obsidian` git practice — Obsidian forum consensus threads,
  `github.com/Vinzent03/obsidian-git`, `kristoffer.dev/blog/obsidian-gitignore`,
  `trustedsec/Obsidian-Vault-Structure`.
- Prior art — `github.com/DavidROliverBA/ArchitectKB`, Foam, Quartz,
  `oleeskild/obsidian-digital-garden`, `adr.github.io/adr-tooling`.
