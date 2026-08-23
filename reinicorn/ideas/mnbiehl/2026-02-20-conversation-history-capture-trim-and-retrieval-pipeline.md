---
type: idea
title: 'Conversation History: Capture, Trim, and Retrieval Pipeline'
slug: 2026-02-20-conversation-history-capture-trim-and-retrieval-pipeline
lifecycle: active
status: new
created: 2026-02-20
author: mnbiehl
---

# Conversation History: Capture, Trim, and Retrieval Pipeline

## Description

Integrate AI conversation history capture into the reins workflow as a low-effort fallback knowledge layer. Engaged developers use exec-plans, decisions.md, and retros. Disengaged developers at minimum get their conversations preserved. Both benefit from searchable history when debugging old decisions.

## The Gap

Reins's structured workflow (specs → plans → decisions → retros) produces high-quality knowledge but requires developer discipline. Developers who don't follow the full workflow leave no trace of their reasoning. When something breaks months later, there's no record of *why* code was written a certain way. The conversations where those decisions were made vanish when the session ends.

## Prior Art Reviewed

### SpecStory (specstoryai/getspecstory, MIT license)
- Go CLI that wraps Claude Code/Cursor/Codex/Gemini and captures sessions to `.specstory/history/` as Markdown
- Provider SPI cleanly abstracts across 5 agent backends
- Parses Claude Code's `~/.claude/projects/<hash>/<uuid>.jsonl` format including DAG reconstruction, warmup filtering, sidechain handling, resumed-session merging
- Cloud product is just storage + search — no distillation or analysis
- Agent-skills repo is largely prompt-only (no backing scripts)
- 1,063 stars — validated developer demand for conversation capture
- Key technical details worth noting: path hashing is `[^a-zA-Z0-9-]` → `-`, symlinks resolved first; JSONL records form DAGs via `parentUuid`; tool results are merged into preceding tool-use messages; `bufio.Reader` not `Scanner` for >64KB lines

### claude-code-cmv (CosmoNaught/claude-code-cmv)
- Strips mechanical bloat from Claude Code session JSONL: tool result bodies >500 chars, thinking signatures, file-history metadata
- Typical session is ~85-90% bloat, ~10-15% actual conversation
- Claims 50-70% reduction; measured mean ~10.5% on 33 sessions (real-world ~2x model predictions)
- Snapshot/branch/tree model (git-like version control for sessions)
- Honest caveat: quality impact of trimming has NOT been measured

## Proposal

A three-layer pipeline, each layer independent and incrementally adoptable:

### Layer 1: Capture (near-zero effort)
- Recommend or bundle SpecStory CLI alongside reins
- `.specstory/history/` committed to the repo (or a dedicated submodule to keep repo size manageable)
- Git hook or `reins` command to auto-commit conversation history on branch push
- Alternatively: build a minimal capture mechanism directly in reins using SpecStory's JSONL parsing as reference (avoid Go dependency, reimplement key parts in Python)

### Layer 2: Trim (pre-commit processing)
- Before committing conversation files, strip mechanical bloat CMV-style:
  - Tool result bodies over a configurable threshold (default 500 chars)
  - Thinking block signatures
  - File-history snapshot metadata
  - `<system-reminder>` blocks
  - ANSI escape sequences in bash output
- Keep: all user messages, all assistant responses, all tool-use requests (inputs, not outputs), complete reasoning
- This is mechanical — a Python script that processes JSONL or Markdown, not an LLM task
- Result: conversation files small enough to commit without bloating the repo

### Layer 3: Retrieval (the hard part — design TBD)
- The value of captured conversations is only realized when an agent can find relevant ones
- Options (not mutually exclusive, roughly ordered by complexity):
  1. **Brute force**: With 1M token context windows, an agent can ingest many trimmed conversations when investigating a specific question. A reins skill that finds conversations touching specific files or time ranges and loads them.
  2. **File-based index**: Each trimmed conversation gets a header with: date, branch, files touched, one-line summary (LLM-generated at trim time). A simple grep-able index.
  3. **Semantic search**: Embeddings over trimmed conversations, vector store, retrieval at query time. This is what SpecStory Cloud does. More infrastructure than reins should probably own.
  4. **Distillation into existing artifacts**: An LLM pass that extracts decisions, patterns, and mistakes from conversations and proposes additions to `decisions.md`, `golden-principles.md`, or `tech-debt/`. Human reviews before merge. This is the highest-value option but also the least reliable.

## Key Design Decisions To Make

1. **SpecStory dependency vs. native capture?** SpecStory is MIT and well-engineered. Using it avoids reimplementing JSONL parsing. But it's a Go binary — reins is Python. Could vendor their parsing logic, or just recommend installing SpecStory alongside reins.

2. **Repo vs. submodule for history?** Conversations accumulate fast. Even trimmed, a team of 5 over 6 months could produce hundreds of MB. A git submodule (like the harness itself) keeps the main repo clean but adds submodule management complexity.

3. **How aggressive is trimming?** CMV's approach (keep tool inputs, strip outputs) is conservative. A more aggressive approach would also summarize long assistant responses. But aggressive trimming risks losing the exact reasoning you're trying to preserve.

4. **When does retrieval happen?** Proactively (agent always checks history before starting work) vs. on-demand (agent searches history only when stuck or asked). Proactive is more useful but costs tokens every session.

## Relationship to Existing Reins Features

- **Exec plans / decisions.md**: The structured, high-quality path. Conversation capture is the unstructured fallback for when developers skip the structured path.
- **PR history crawling** (design-docs/pr-history-crawling.md): Complementary — PR crawling extracts review feedback, conversation capture preserves the development reasoning.
- **Retros** (ideas/2026-02-20-feature-retros): Retros could be seeded from conversation history — "what did we discuss and struggle with during this feature?"
- **Golden principles**: Layer 3 distillation could propose new principles based on patterns across many conversations.

## What NOT To Build

- A cloud sync service (that's SpecStory's business, not ours)
- A conversation viewer/UI (Markdown files + editor is fine)
- Real-time analysis during sessions (out of scope — reins is a harness, not an IDE extension)

## Rough Implementation Phases

1. Document the recommended SpecStory + reins setup (Layer 1 — just docs)
2. Build a `reins trim` command that processes `.specstory/history/` files (Layer 2 — Python script)
3. Build a `reins history search` skill that finds and loads relevant conversations (Layer 3, option 1 — brute force)
4. Evaluate whether more sophisticated retrieval is worth the complexity (Layer 3, options 2-4)

## Notes

- SpecStory's provider SPI pattern is worth studying if reins ever needs to support agents beyond Claude Code
- The cross-platform path canonicalization in SpecStory (case-insensitive matching for macOS) is a gotcha worth noting for any future JSONL parsing
- CMV's own docs admit they haven't measured quality impact of trimming — we should do this before relying on trimmed conversations for critical decisions
