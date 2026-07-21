# Doc-review CI cleanup cannot push to kb main: the reinicorn-doc-review ruleset r

**Date:** 2026-07-21
**Author:** Michael Biehl
**Status:** new

## Description

Doc-review CI cleanup cannot push to kb main: the reinicorn-doc-review ruleset requires PRs and GITHUB_TOKEN has no bypass. GitHub rejects adding the Actions integration as a bypass actor on this org (no app installation). Need a sanctioned push path for the cleanup workflow: fine-grained PAT secret (workflow already has the REINICORN_INSTALL_TOKEN pattern) or a deploy key with DeployKey bypass on the ruleset. Until fixed, browser-merged review PRs stay in-review and need manual rcorn review merge. Evidence: reinicorn-kb run 29701503171 failed on GH013 after PR 1 merged.

## Notes

_No additional notes yet._
