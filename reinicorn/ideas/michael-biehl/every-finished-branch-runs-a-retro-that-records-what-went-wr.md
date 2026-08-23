---
type: idea
title: Every finished branch runs a retro that records what went wrong — including fric
slug: every-finished-branch-runs-a-retro-that-records-what-went-wr
lifecycle: active
status: new
created: 2026-08-16
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Every finished branch runs a retro that records what went wrong — including fric

## Description

Every finished branch runs a retro that records what went wrong — including friction with rcorn itself — and the retro flow ends with a CTA prompting the agent to file rcorn feedback for any tool friction. Today nothing wires this: finishing-a-development-branch never mentions retros, the retro template has no tool-friction section, and rcorn feedback is only listed in the command table. Wiring: (1) add a 'Tool Friction' (rcorn/harness) section to the retro template, (2) finishing-a-development-branch step to create the retro, (3) retro create output ends with 'file anything under Tool Friction via rcorn feedback'. This is the qualitative twin of the telemetry spec (opt-in-anonymous-telemetry): telemetry says where friction is, retro-driven feedback says why.

## Notes

_No additional notes yet._
