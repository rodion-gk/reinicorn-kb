---
type: idea
title: lavish-axi as the Visual Review Surface for kb Docs
slug: lavish-axi-as-the-visual-review-surface-for-kb-docs
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# lavish-axi as the Visual Review Surface for kb Docs

## Summary

`lavish-axi` (`github.com/kunchenguid/lavish-axi`, ~2.2k stars) is an **official
AXI implementation by the same author as the AXI spec reinicorn already built its
output surface against** ([[agent-native-output-surface-axi-principles]] cites
`github.com/kunchenguid/axi` directly). Its pitch: *"HTML is the new markdown.
Lavish is the new editor for your HTML artifacts."*

The concrete claim that matters to reinicorn: **instead of a human reading a giant
markdown plan, render it as a visual HTML artifact where they leave inline
feedback, so they can tell the agent exactly what to change before coding starts.**

That is a precise description of the current bottleneck in reinicorn's review
lane. `rcorn review` puts a spec on a `reinicorn-kb` PR and a human reads a long
markdown document in a GitHub diff view. Feedback comes back as prose comments
that an agent then has to interpret. Lavish replaces that with element-level and
text-range-level annotations returned to the agent as structured data.

**Naming note:** this was investigated under the name "laravel-axi", which does
not exist. There is no Laravel/PHP implementation in the official or community
AXI catalog. Laravel's own agent story is `Laravel Boost` — an MCP server, i.e.
the architecture AXI explicitly positions against. If a Laravel angle is wanted
later it is a separate, unrelated idea.

## How it actually works

Zero-install for agents — `npx -y lavish-axi`, no database, no auth; session
state in `~/.lavish-axi/`. There is also a skill: `npx skills add
kunchenguid/lavish-axi --skill lavish`.

```sh
lavish-axi path/to/artifact.html          # spawn detached server, open browser, return immediately
lavish-axi poll path/to/artifact.html     # long-poll, blocks until feedback arrives
lavish-axi --agent-reply "message"        # reply in-session without another poll cycle
lavish-axi export                         # self-contained HTML, local assets inlined
lavish-axi share [--password ...]         # publish to ht-ml.app (third-party host)
```

Feedback returned to the agent is tagged and structured:

- `tag: "text"` — annotation on a selected text range, with boundaries
- `tag: "mermaid-node"` — annotation on a diagram node, by node ID
- `tag: "whiteboard"` — Mermaid diagram edited as an Excalidraw whiteboard,
  returned as a scene diff summary + source hash
- layout warnings — a render-time audit flags severe findings (clipped text,
  overflow, unreachable viewport); **severe findings block human review until the
  agent fixes and re-checks**
- session state — `status: "feedback" | "ended"`, `ended_by: "user" | "agent"`

Design properties that matter here: it injects **no design system** (the saved
HTML renders identically opened directly in a browser), and the only thing added
to the artifact is a `<script src="/sdk.js">` tag. Local-first — no cloud in the
core feedback loop. TOON output and long polling for token efficiency.

## Why this fits reinicorn specifically

1. **Same design lineage.** reinicorn's output surface is already an AXI
   implementation. Adopting a sibling AXI tool is not a new architectural axis.
2. **Mermaid → whiteboard is the sleeper feature.** Architecture and flow
   diagrams in specs are exactly where markdown review breaks down, and editing
   the diagram directly beats describing the change in prose.
3. **The layout-audit gate is a review-lane primitive.** "Agent must fix severe
   findings before a human is allowed to review" is structurally the same shape
   as the solo-maintainer self-review gate that just landed (`32f4ba3`).
4. **Structured feedback closes the loop.** Prose PR comments are the lossiest
   part of the current lane.

## Open questions

1. **Which lane?** Review is specs-only today. Do visual reviews apply to specs
   only, or also PRDs and exec-plans (where the "giant plan" problem is worst)?
2. **Relationship to the GitHub PR lane.** Does lavish *replace* the kb PR for
   solo review, or run *before* it as a pre-PR iteration step? A public kb repo
   still wants the PR as the audit trail. **Proposed answer: neither — it feeds
   the PR.** See [[bridge-lavish-annotations-to-github-pr-reviews]].
3. **Node dependency.** reinicorn is Python/uv. `npx` adds a Node runtime to a
   previously Node-free tool. Acceptable as an optional, lazily-invoked
   integration; not acceptable as a hard dependency in any agent code path — the
   same guardrail [[obsidian-integration-emit-vault-native-conventions-so-the-kb]]
   applied to Obsidian.
4. **`share` and the public kb.** `ht-ml.app` is third-party hosting, public by
   default unless `--password`. Almost certainly out of scope — the kb is already
   a public GitHub repo and does not need a second publish surface.
5. **Where do the HTML artifacts live?** This is the whole of
   [[html-as-a-kb-doc-type]] — and the answer there ("render target, not storage")
   determines whether this integration is cheap or expensive.
6. **License unverified.** The README states no explicit license. Needs checking
   before any vendoring or dependency.

## Relationship to other ideas

- [[html-as-a-kb-doc-type]] — the storage-format question this depends on.
- [[configurable-open-in-markdown-editor-env-var]] — if "open this doc for a
  human" becomes a pluggable verb, lavish is simply one editor backend alongside
  Obsidian and `$EDITOR`.
- [[obsidian-integration-emit-vault-native-conventions-so-the-kb]] — the same
  architectural conclusion should apply: human-facing layer, zero runtime
  dependency, rcorn stays the only writer.

## Sources

- `github.com/kunchenguid/lavish-axi` — README and AGENTS.md
- `axi.md` — the 10 AXI principles and the official implementation list
  (gh-axi, chrome-devtools-axi, lavish-axi, quota-axi)
- Peter Yang, "use visual plans, not walls of markdown" (Threads)
