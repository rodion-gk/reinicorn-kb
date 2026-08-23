---
type: spec
title: Opt-in anonymous telemetry
slug: opt-in-anonymous-telemetry
lifecycle: active
status: draft
created: 2026-08-16
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Opt-in anonymous telemetry

## Problem

Reinicorn's guardrails (skills, hooks, doc workflows, golden principles) are
tuned by intuition and community lore. We have no empirical signal about which
commands and workflows are actually used, where they break, or which guardrails
agents override — let alone how any of that varies by agent harness or model.

Prior art (researched 2026-08-16) shows the pieces exist but nobody has closed
the loop in the open:

- **Copilot Arena** (arXiv 2502.09328) proved in-the-wild usage signal has
  near-zero correlation with static benchmarks — real usage is the only
  ground truth for tool behavior.
- **VS Code/Copilot harness team** (code.visualstudio.com blog, 2026-05/07)
  documented telemetry → A/B → shipped per-model prompt changes. Closed
  source, single vendor.
- **aider, Cline, Goose, Roo Code** all run opt-in anonymous PostHog telemetry
  in agent CLIs, but none has published learnings from it.

General product tuning needs almost no scale ("nobody runs `debt create`",
"the session-start hook errors on 15% of installs" are actionable at N≈30
users). Per-model guardrail tuning needs volume we don't have yet — but if
events carry `harness` and `model` dimensions from day one, cross-model
analysis becomes a later query, not a later system.

## Design Goals

Priority order is explicit: **1 and 2 outrank everything else.** Any conflict
resolves in favor of anonymity and security, at the cost of data richness.

1. **True anonymity, enforced client-side.** The collection server never
   receives anything linkable to a person or install long-term, and never
   receives anything that would need scrubbing before publication. Publishing-
   side controls are defense-in-depth, never the primary protection.
2. **Security.** Minimal attack surface: one first-party endpoint, no third-
   party analytics vendor, open-source auditable collector code, all-enum
   schema so injection is structurally impossible.
3. **Strict opt-in.** Nothing is ever sent without an explicit yes. Copy the
   aider consent pattern, not the Homebrew/Next.js opt-out pattern.
4. **Public dataset.** All collected data is published openly (daily batch) —
   radical transparency doubles as the trust story and the crowdsourced-
   tuning story.
5. **Inspectable.** Published schema with literal example JSON (Next.js
   pattern); a debug mode prints events instead of sending (GitHub CLI
   `GH_TELEMETRY=log` pattern).
6. **Cross-model-ready.** Events carry harness + model dimensions so per-model
   tuning is a later query, not a later system.
7. **Actionable at small N.** v1 questions must be answerable with tens of
   opted-in installs.

## Design

### Consent

- Default: fully off. No event, no network call, no salt file.
- `rcorn telemetry enable|disable|status` manages state; `enable` prints the
  schema summary and the docs URL before confirming.
- One-time interactive prompt is allowed only at `rcorn init` (never
  mid-command), and declining is permanent — never re-prompt.
- `DO_NOT_TRACK=1` is an unconditional override, even if previously enabled.
- CI environments: off regardless of config (Nx pattern) — detect via `CI`.

### Identity: rotating, client-side, unlinkable

No persistent UUID — a lifetime pseudonym is not anonymity. Instead:

- On enable, generate a random 256-bit **salt** stored at
  `~/.rcorn/telemetry-salt`. The salt never leaves the machine.
- Every event's `client_id = HMAC-SHA256(salt, current_UTC_month)` truncated
  to 8 bytes. IDs rotate monthly by construction; the server and the public
  dataset cannot link one install across months, and nothing server-side can
  undo this — the linking key never left the client.
- Within a month, `client_id` supports dedup, funnels, and per-install weight
  caps. Across months, installs are unlinkable.
- Deleting the salt file (or disable→enable) is an immediate identity reset.

### Data minimization at the client (the primary control)

- **All-enum schema.** Every field is a closed enum or a version string from
  a known set. No free text exists anywhere in the pipeline, so neither PII
  leakage nor content injection is expressible in a valid event.
- **Timestamps coarsened client-side** to the UTC hour before send. Precise
  timing (a working-hours fingerprint correlatable with public GitHub
  activity) never exists server-side.
- **Rare-value suppression at the client**: `model` is mapped to a maintained
  enum of known model families; anything else is sent as `other`. Same for
  `harness` and `os`. A custom model name can never become a fingerprint
  because it is never transmitted.
- Hard never-collected list: doc content, prompts, code, file/dir paths, repo
  names, branch names, hostnames, usernames, env values, keys, IP-derived
  geo, precise timestamps.

### Event schema (schema-first, versioned)

Events are declared in one registry module (mirroring `doc_types.REGISTRY`
style) and validated client-side before send; the collector re-validates and
drops anything malformed or carrying unknown fields. Schema carries a
`schema_version`.

Common dimensions on every event: `schema_version`, `rcorn_version`, `os`,
`harness`, `model`, `client_id`, `hour_bucket`.

v1 event types (deliberately coarse):

1. `session_start` — emitted by the session-start hook: kb present, plan
   present, mode flags. One event per session; the hook is the single
   instrumentation point, not per-command patching.
2. `command` — command name + subcommand + exit class (ok / user-error /
   internal-error). No arguments, no values.
3. `doc_lifecycle` — doc type + verb (create/show/complete/archive). Enables
   funnels like plan create → complete without touching content.
4. `guardrail_event` — guardrail kind (skill / hook / principle / lint) +
   guardrail id (from the shipped registry, an enum) + outcome (fired /
   followed / overridden / errored) + workflow phase (session-start /
   pre-commit / mid-task / finish). Phase matters: field data (arXiv
   2601.10253) shows workflow-boundary interventions get ~52% engagement vs
   62% dismissal mid-task.
5. `error` — component + error class (no messages, no paths).

### Collection and hosting

- **Ingest**: a single first-party Cloudflare Worker (free tier) is the only
  endpoint. It schema-validates, enforces enums, rate-limits per `client_id`,
  and appends to R2/D1. ~100 lines; its source lives in the public telemetry
  repo so the exact server behavior is auditable.
- **No third-party analytics vendor** (no PostHog). Fewer parties holding
  data is the point; the public dataset is the primary and only store.
- **No IP retention**: the worker never logs or stores IPs or headers;
  Cloudflare-edge logging is disabled/minimized and the residual "Cloudflare
  sees connection metadata in transit" is documented honestly in the threat
  model rather than papered over.
- Fire-and-forget with a short timeout — telemetry failure must never slow or
  break a command. `RCORN_TELEMETRY_DEBUG=1` prints the exact JSON to stderr
  instead of sending. Endpoint overridable for users who want to run their
  own collector.

### Public dataset (daily batch — decided 2026-08-16)

- A public repo (`crystldm/reinicorn-telemetry`) holds daily NDJSON files
  committed by a scheduled GitHub Action pulling from the worker, plus
  regenerated aggregate JSON. Versioned, diffable, anyone can clone and
  analyze.
- **Daily batch, not a live raw feed**: the ~24h delay is the remediation
  window, and batch publication kills timing-correlation attacks that a live
  feed would enable. Live data on the dashboard is aggregate counters only.
- **k-anonymity gate at publish** (defense-in-depth): a dimension combination
  (harness × model × os × rcorn_version) appearing for fewer than K=5
  distinct client_ids in a batch is generalized to `other` in the raw export.
- Dashboard: static GitHub Pages reading the aggregate files. No backend, no
  accounts, no cookies.
- Retention: the public dataset is the retention (git history). Because
  publication is irreversible, client-side minimization (above) is sized so
  that nothing in a valid event is sensitive even in perpetuity.

### Threat model (maintained in docs, summarized here)

- **Deanonymization**: mitigated by monthly client-side ID rotation, hour
  coarsening, client-side rare-value suppression, publish-time k-anonymity.
  Residual risk stated honestly: within a single month, an active install's
  event pattern is a weak behavioral signature.
- **Malicious injection / poisoning**: all-enum schema (junk can only inflate
  counts), per-`client_id` rate limits, per-install weight caps in every
  aggregate (Chatbot Arena vote-rigging lesson, arXiv 2501.17858), anomaly
  review before each batch publish.
- **Collector compromise**: worst case is loss/corruption of coarse enum
  counts — by construction there is nothing sensitive to steal. Worker code
  is public and reviewed via PR like any other change.
- **Sample bias**: before generalizing, diff the opted-in sample's
  harness/model mix against known ecosystem distributions (arXiv 2604.05100
  audit pattern); treat per-model findings as hypotheses to confirm on a
  fixed benchmark harness (aider leaderboard pattern).

### Docs

New docs page: what is collected (with literal example events), the never-
collected list, the threat model, how to enable/disable/inspect, the publish
pipeline, and the privacy contact. Linked from README and from the
`telemetry enable` output.

## Non-Goals

- **Per-model A/B power in v1.** The install base can't support it; v1 buys
  directional signal only. No live prompt experiments.
- **No content collection ever** — not even opt-in flags for it (unlike
  Claude Code's layered `OTEL_LOG_*` redaction opts). Events are metadata-
  only by construction.
- **No third-party analytics** (PostHog, GA, etc.) in any tier.
- **No live raw event feed.** Aggregates may be live; raw NDJSON is daily.
- **No formal differential privacy** in v1 — anonymity comes from data
  minimization and unlinkability, stated plainly rather than implied by a
  mechanism we don't have; revisit if the dataset grows enough to need it.
- **No arena / pairwise preference UI.** Different mechanism, different spec.
- **No OpenTelemetry conformance.** The OTel `gen_ai.*` conventions are still
  "Development" maturity and aimed at user-side observability; revisit if
  they stabilize.
- **Qualitative feedback flow** — covered by the retro→`rcorn feedback` CTA
  idea (see ideas/every-finished-branch-runs-a-retro…), not this spec.
