---
type: idea
title: Externalize doc templates from inline Python strings to editable files in the ha
slug: 2026-02-26-externalize-doc-templates-from-inline-python-strings-to-edit
lifecycle: active
status: new
created: 2026-02-26
author: mnbiehl
---

# Externalize doc templates from inline Python strings to editable files in the ha

## Description

Externalize doc templates from inline Python strings to editable files in the harness. Each doc type would have a template file (e.g. harness/{repo}/templates/design.md) that the creator function reads instead of hard-coding the markdown. Enables per-repo customization of doc structure without code changes.

## Notes

_No additional notes yet._
