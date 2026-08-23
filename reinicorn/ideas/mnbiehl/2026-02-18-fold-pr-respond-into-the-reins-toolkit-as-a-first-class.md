---
type: idea
title: Fold pr-respond into the reins toolkit as a first-class PR review workflow.
slug: 2026-02-18-fold-pr-respond-into-the-reins-toolkit-as-a-first-class
lifecycle: active
status: new
created: 2026-02-18
author: mnbiehl
---

# Fold pr-respond into the reins toolkit as a first-class PR review workflow. 

## Description

Fold pr-respond into the reins toolkit as a first-class PR review workflow. Key things to build out: (1) Agent attribution footers on all agent-drafted PR comments — opt-in via session config, makes agent vs human comments distinguishable for trust and harness insight ingestion. (2) PR comment ingestion pipeline — ingest PR review results to extract harness rule candidates and track agent accuracy over time. (3) Full PR review workflow: fetch, triage, fix, reply, post — currently spread across pr-respond CLI and manual steps, should be a reins subcommand (e.g. reins pr review). (4) Worker/spawner placement rule — background worker modules should live next to the domain they serve, not in the layer that spawns them. (5) No dynamically generated Python source strings rule is already in golden-principles.md but the pr review tooling gave us the data to discover it — validates the dogfooding approach.

## Notes

_No additional notes yet._
