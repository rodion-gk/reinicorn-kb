---
type: idea
title: 'pr-respond misses review-body feedback: the tool only fetches inline PR comments'
slug: 2026-02-18-pr-respond-misses-review-body-feedback-the-tool-only-fetches
lifecycle: active
status: new
created: 2026-02-18
author: mnbiehl
---

# pr-respond misses review-body feedback: the tool only fetches inline PR comments

## Description

pr-respond misses review-body feedback: the tool only fetches inline PR comments via /pulls/{pr}/comments but GitHub review summaries live at /pulls/{pr}/reviews. Non-line-specific feedback (like 'missing tests') posted in the review body is invisible to pr-respond list/stats, causing a false 'all addressed' signal. Fix: fetch reviews endpoint in addition to comments and surface review bodies as first-class items in list output.

## Notes

_No additional notes yet._
