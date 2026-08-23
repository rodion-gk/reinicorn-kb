---
type: idea
title: Flesh out workflow lifecycle commands — plan create/status/complete naming is co
slug: 2026-02-17-flesh-out-workflow-lifecycle-commands-plan-create-status-com
lifecycle: active
status: new
created: 2026-02-17
author: mnbiehl
---

# Flesh out workflow lifecycle commands — plan create/status/complete naming is co

## Description

Flesh out workflow lifecycle commands — plan create/status/complete naming is confusing from user perspective. Consider top-level commands like 'reins done' or 'reins finish', or rethink the plan subcommand naming to better match how users think about branch lifecycle (start work, check progress, wrap up).

## Additional considerations

### Rename `reins idea`

The `idea` command name is too generic — it's easy to confuse with "jot down a note about the current task." Its intended scope is capturing **standalone ideas unrelated to the current work** (backlog items, future features, shower thoughts). When this becomes a skill trigger word, the name needs to clearly convey that scope. Consider: `reins backlog`, `reins spark`, `reins capture`, or similar. The help text should also make the scope explicit.

### Configurable commit/push behavior

Future: make auto-commit and auto-push behavior configurable per command via `.reins-config`. For now, the design is auto-commit always + push only on `reins kb publish`.
