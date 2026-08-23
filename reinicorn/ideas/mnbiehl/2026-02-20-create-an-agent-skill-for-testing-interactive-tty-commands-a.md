---
type: idea
title: Create an agent skill for testing interactive TTY commands. Agents currently run
slug: 2026-02-20-create-an-agent-skill-for-testing-interactive-tty-commands-a
lifecycle: active
status: new
created: 2026-02-20
author: mnbiehl
---

# Create an agent skill for testing interactive TTY commands. Agents currently run

## Description

Create an agent skill for testing interactive TTY commands. Agents currently run shell commands without a TTY, making it difficult to fully test features that change behavior based on TTY status (like 'reins feedback'). Consider teaching agents to use the 'script' utility, 'expect', 'socat', or Python's 'pexpect'/'pty' modules to simulate an interactive terminal for end-to-end testing.

## Notes

_No additional notes yet._
