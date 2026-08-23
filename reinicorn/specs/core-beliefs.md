---
type: spec
title: 'Core Beliefs: Agent-First Operating Principles'
slug: core-beliefs
lifecycle: active
status: approved
created: 2026-02-13
author: Michael + Claude
---

# Core Beliefs: Agent-First Operating Principles

> **This is a living document.** These principles are a starting point drawn from the OpenAI harness engineering article and the reins design. Teams should keep what resonates, modify what doesn't fit their context, and add beliefs that reflect their own hard-won lessons. The goal is a shared foundation your agents and humans can align on — not a rigid doctrine.

---

## 1. Repository is the source of truth

If the agent can't see it in-repo, it doesn't exist. Slack threads, Google Docs, and tacit knowledge are invisible to agents. Every decision, convention, and piece of context that matters must live in version-controlled files. When you find yourself explaining something verbally for the second time, that's a signal it should be written down in the repo.

## 2. Progressive disclosure over comprehensive instruction

Give agents a map, not a 1,000-page manual. Teach them where to look, not everything at once. A sparse entry point (like AGENTS.md) that links to deeper documents lets agents load context on demand. Front-loading every detail causes confusion and wastes context window; layered navigation lets agents drill into exactly what they need.

## 3. Mechanical enforcement over documented conventions

When docs fall short, promote the rule into code. Linters, structural tests, and CI checks are more reliable than instructions that agents might miss or misinterpret. A convention that only exists in prose will eventually be violated. A convention enforced by a failing build is self-correcting and requires no ongoing human vigilance.

## 4. Agents replicate what exists

Agents pattern-match on the codebase. If the existing code is clean and consistent, agents will produce clean, consistent output. If the codebase is messy, agents will faithfully replicate the mess — and compound it. Investing in code quality pays double dividends: it improves both the human and agent experience.

## 5. Cross-branch awareness prevents drift

Feature branches should publish what they're doing so other agents can coordinate. Without visibility into parallel work, agents on different branches will make conflicting assumptions, duplicate effort, or create merge conflicts. A lightweight mechanism for branches to advertise their plans and touched areas is cheaper than resolving collisions after the fact.

## 6. Tech debt is a high-interest loan

Pay it down continuously in small increments rather than letting it compound. In agent-assisted codebases, debt accumulates faster because agents produce code faster. A shortcut taken today becomes a pattern replicated across dozens of agent-written files tomorrow. Track debt explicitly, prioritize it visibly, and chip away at it alongside feature work.

## 7. Boring technology is agent-friendly technology

Composable, stable APIs with good training data representation are easier for agents to model. Agents perform best with well-documented, widely-used patterns that appear frequently in their training data. Exotic frameworks, clever metaprogramming, and cutting-edge abstractions may delight humans but confuse agents. When choosing technology, "Will an agent be able to work with this effectively?" is a legitimate selection criterion.

## 8. Enforce boundaries centrally, allow autonomy locally

Care deeply about interfaces and dependencies. Allow freedom in implementation details. Define strict rules about what can depend on what, how modules communicate, and where boundaries lie. Within those boundaries, let agents (and humans) choose the implementation approach that works best. This prevents architectural erosion while avoiding micromanagement that slows everyone down.

## 9. Human taste is captured once, then enforced continuously

Encode preferences into tooling so they apply to every line of code automatically. Style choices, naming conventions, error handling patterns, and architectural preferences should be expressed as linter rules, formatters, and structural tests. This transforms a one-time human decision into a permanent, zero-effort guardrail that applies equally to human-written and agent-written code.

## 10. Corrections are cheap, waiting is expensive

In high-throughput agent environments, fix-forward is often better than blocking. When an agent produces imperfect output, it is frequently faster to let it finish and then correct the result than to block progress with elaborate upfront validation. Design review processes that are lightweight and fast-feedback rather than heavyweight and gate-keeping. The cost of a quick correction is almost always lower than the cost of idle agents waiting for approval.
