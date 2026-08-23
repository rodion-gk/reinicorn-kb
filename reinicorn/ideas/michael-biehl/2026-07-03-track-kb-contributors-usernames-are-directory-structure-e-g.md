---
type: idea
title: 'Track kb contributors: usernames are directory structure (e.g. ideas/{username}/'
slug: 2026-07-03-track-kb-contributors-usernames-are-directory-structure-e-g
lifecycle: active
status: new
created: 2026-07-03
author: Michael Biehl
---

# Track kb contributors: usernames are directory structure (e.g. ideas/{username}/

## Description

Track kb contributors: usernames are directory structure (e.g. ideas/{username}/), but nothing registers or validates them. The same person can end up with parallel dirs under two usernames across machines (e.g. michael-biehl on one, mnbiehl on another). Need a contributor registry/identity check — e.g. a contributors file in the kb mapping canonical username to git identities (name/email), validated at doc-create/publish time so a new unrecognized username prompts confirmation instead of silently forking identity.

## Notes

Escalation flow for an unrecognized username (per Michael, 2026-07-03):

1. Don't fail or silently create. First, check existing contributor dirs/registry for
   similar usernames (e.g. shared git email, name similarity: michael-biehl vs mnbiehl).
2. If a likely match exists, the CLI output should suggest using that existing username —
   agent-consumed, so the agent can self-correct without bothering the user.
3. Only if no match is found (or the agent/user rejects the suggestion) does it raise to
   the user for confirmation before registering a genuinely new contributor.
