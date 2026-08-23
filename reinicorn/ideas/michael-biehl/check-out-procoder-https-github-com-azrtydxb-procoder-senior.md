---
type: idea
title: 'Check out procoder (https://github.com/azrtydxb/procoder): ''senior-developer dis'
slug: check-out-procoder-https-github-com-azrtydxb-procoder-senior
lifecycle: active
status: new
created: 2026-08-20
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Check out procoder (https://github.com/azrtydxb/procoder): 'senior-developer dis

## Description

Check out procoder (https://github.com/azrtydxb/procoder): 'senior-developer discipline for AI coding agents' — a single Go binary (Apache-2.0, v1.0.2) providing a commit gate that treats unchecked work as failing, quality controllers that refuse to call unfinished work done, and a lessons loop that closes each escaped bug's class; ships adapters for 20+ agents incl. Claude Code plugin, Codex, Copilot, anything reading AGENTS.md. Relevant to Reinicorn's hooks/push-gate + golden-principles enforcement: possible complement (gate layer) or a pattern source for the lessons-loop → principles/retro flow.

## Notes

### Deep-dive analysis (2026-08-21)

Small correction first: the project is called procoder, not procode. It is one Go binary with zero dependencies, Apache-2.0, written by one person (Pascal Watteel).

How this analysis was made: five parallel subagents. One audited the Go code and ran the build, vet, and test suite. One compared their superpowers adaptation to ours, file by file. One built the binary from source and tried to cheat every quality controller. One ran the gate and hooks live, on a scratch clone of reinicorn and on a demo repo rigged with known problems. One did git-history and supply-chain forensics. Everything below was run or read, not guessed from the README.

#### How old it really is

The repo started on August 16. The first two days produced a JavaScript plugin. On August 18 the author threw that away and rewrote everything in Go. So the Go codebase was about three days old when it reached version 1.0.2. There are 381 commits, all from one author, peaking at 60 commits in a single hour. The first 322 commits went straight to main without a PR; after that, nearly everything went through PRs. Twenty release tags landed in about thirty hours.

#### Code quality: same league as ours, not better

The audit came back friendlier than the commit velocity suggests, scoring it 7 out of 10. This is not slop. It builds cleanly, `go vet` passes, and 605 tests pass in eight seconds. The tests are real: they use temp dirs, put fake tool binaries on PATH, and carry comments naming the regression each one pins. External tools are always exec'd as argument lists, never through `sh -c`, and every call has a timeout. The core promise, that a file the tool could not check is never reported clean, is actual enforced code with its own regression tests, not just README prose.

The weaknesses are the ones you'd expect from code written this fast. `main.go` is 1,065 lines of hand-rolled dispatch: a forty-case switch, flags parsed by comparing `args[1]` to string literals, and almost no test coverage on that file. The same exec helper is copy-pasted seven times. There are four separate parsers for `git status --porcelain`, and one of them drops the quote handling the others have, so `check` and `security` can disagree about a filename with a space in it. That is exactly the drift the README claims is impossible, one layer down. The zero-dependency choice also forced a hand-rolled TOML subset that silently skips lines it can't parse. And all the workflow state lives in markdown parsed by regex, with statuses stored as free strings.

Overall coverage is 68 percent, but it collapses to 30-40 percent in the packages that wrap external tools, which is where the bugs will live. Our numbers for comparison: reinicorn has 11.2k lines of source and 18.3k lines of tests, 1,164 tests, 87 percent coverage, and four CI workflows including the architecture and kb lints. One caveat: reinicorn was not audited at the same depth in this pass. On what we can measure, the two are in the same league. We are ahead on test discipline and CI. They have far more feature surface. Their code is not better than ours.

#### The superpowers adaptation: we did this better

Procoder claims to have "absorbed" superpowers. In practice there is no vendored superpowers text anywhere in the repo. Every skill was rewritten by hand, in their own voice, at about a third of the original length. Their TDD command is 3KB; upstream is 9KB with 25 rationalization-table rows, of which they kept none. The rationalization tables and red-flag lists, which are the part of superpowers that actually stops an agent mid-excuse, are mostly gone. There is no pin and no lineage, so now that upstream is at 6.3.0 they have no way to diff against it or pull updates. There is also a plain bug: 16 of the 34 command files tell the model "the user invoked this with arguments" and then never include the `$ARGUMENTS` placeholder, so the user's arguments never reach the prompt.

The "20+ agents" support is one AGENTS.md copied to twelve host-specific paths, plus the 33 command files duplicated three times with a one-string substitution. The generator that supposedly keeps the copies in sync exists only as a helper inside a test. Parity is pinned by `go test`, and the agent is expected to retype all twelve copies whenever the master file changes.

Our adapter model is better on every axis of the adaptation itself: a pinned upstream commit, seven small patches, appends kept out of the patches so rebases stay small, a per-file hash lockfile, a second adapter (mattpocock-skills) proving the slot is generic, and wiring generated from the doc-type registry.

Where they beat us is enforcement. Their hooks cover session start, session stop, pre-compact, every Bash call, and every Write/Edit. The commit gate actually denies a `git commit` through the hook's permission decision, in about 30 milliseconds, and when the gate itself fails it fails open with an explicit "this commit was NOT checked" message. Our skills are advice; their controllers refuse.

#### The controllers: structural refusals are real, quality refusals are theater

One subagent built the binary and tried to cheat every controller. The pattern that emerged is clean. Anything structural genuinely refuses: one active sprint at a time, sprint close refuses while a story is open and uncarried, the next sprint refuses to open until the retro is written, ADR checks catch dangling supersede references, release refuses on a version mismatch across files, and epic close cascades down to open stories. All verified live.

Anything that claims to judge quality turned out to be a non-emptiness check wearing a refusal message. `spec check` passed lorem ipsum. Its "untestable criterion" detector is a ten-phrase regex anchored to end of line, so "The login page works well." passes because of the period. `todo close` accepted "Evidence: trust me", and a story closed with evidence "lol". The lessons ledger marked "we will be more careful" as a landed adaptation. An agent facing these refusals will learn to satisfy the regex, not to fix the work.

The interesting twist is that their lessons ledger produced real results anyway. It has 32 entries, each naming the layer that should have caught the bug. Of 110 test names cited as adaptations, 108 exist in the tree. Their review rubric grew from 9 generic lines to 22 repo-specific ones. But the binary didn't do that. The prompt layer (a mandatory reflection step in their merge command) and one disciplined human did.

#### The gate in practice

The gate is fast and honest. Incremental checks run in about 30ms because it only looks at changed files. `format` prints the result and provably never writes the file. Missing tools are reported loudly as unchecked, with a failing exit code.

It is also thin. It orchestrates external tools rather than analyzing anything itself, so on a reinicorn clone with a normal machine setup, 214 of 237 files came back "unchecked" and the gate stays red until you install around 13 tools. Without gitleaks there is no secret detection at all, only an honest "not checked". Other live findings: wrapping a commit in `sh -c` slips past the commit hook, a file of exactly 5MB slips the "over 5MB" check, `procoder test` runs bare `pytest` instead of `uv run pytest`, and the session-stop hook silently writes a handoff file into the repo, which contradicts their own "nothing touches your tree" pitch.

#### Trust: do not install this as a plugin

This was the worst finding. The plugin ships prebuilt binaries in `dist/`, and the hook launcher executes them on every tool call. Those binaries were built from a dirty working tree at a commit that does not exist in the public repo. No checksums, no CI rebuild, no provenance; CI uploads the committed binaries as-is. On top of that, SECURITY.md still describes the deleted JavaScript version, including a daily phone-home the Go binary does not do, and their own docs-drift gate never caught it. The git history deliberately scrubs AI attribution, the repo's .git weighs 128MB from committed binaries, and the bus factor is one. If we want to experiment, build from source and never point hooks at dist/.

The star count is astroturfed. The repo had 56 stars at three days old and 105 at five days, with one watcher and no promotion anywhere we could find (nothing on Hacker News or Reddit). The author's other new repo, Fastllm-proxy, sits at 104 stars, 0 watchers, and 97 forks. His fourteen older repos earned 9 stars combined over a year. His 99 followers, every one we sampled, were created in the same three-day window in June, have 0 followers of their own, and hold only forked repos under faker-style names (VicentaKoelp, Cletus1Pagac, Eric2Smitham). One of those follower accounts forked Fastllm-proxy, and the Fastllm-proxy fork owners have the same kind of names. The follower farm, the fork farm, and the star deliveries are the same operation. We could not query the stargazer list directly from this environment, so the star accounts themselves were not sampled; the inference rests on followers, fork owners, and the ratios. procoder's own five forks look like real users, so there is a sliver of genuine interest, but the badge number is bought. Taken with the scrubbed attribution and the unverifiable binaries, the pattern is consistent: an experienced developer choosing to make the surface look better than the provenance. Treat every claim from this account as unchecked until verified.

#### Worth taking

In order of value:

1. The escape-to-adaptation loop. Every bug that escapes must name the layer that should have caught it, and a rubric line, lint rule, or pinning test must land in the same PR. This is the one mechanism with demonstrated results in their repo, and it maps directly onto our retro and principles flow.
2. A session-stop handoff note. Their Stop/PreCompact hook writes a small facts-plus-notes file for the next session. Cheap and useful; we would gitignore the state dir.
3. The commit-gate hook shape: deny through the hook decision, and fail open with an explicit "the gate did NOT run" message. If we ever gate closure on anything, gate on something executable, like a named test that exists and passed in this run, never on a non-empty evidence field.
4. A placeholder and vagueness lint for specs and plans ("TBD", "similar to task N", trailing "works well"), plus a fingerprint of the spec's acceptance criteria stored on derived plans so drift becomes visible. Ship these as smell lints, not gates.
5. A dogfood CI job that runs our own lints and gates over our own tree, and tagged releases with a human-readable changelog.
6. Their git hygiene checks as stdout-only helpers: conflict markers, junk files, oversized files, attribution scrubbing.

Worth skipping: the sprint/epic/milestone hierarchy (three state files and five subcommands for what a plan plus a branch already represents), story-per-criterion seeding (their 112 "stories" are one-line spec criteria), copilot-leak (Copilot-and-GitHub-specific plumbing), evidence-field gating (it invites fabricated evidence text), and rewriting upstream skills in your own voice.

#### Housekeeping

Found while comparing: this checkout has the superpowers lock (v6.1.1) but every file is missing under `.agents/skills/`. The adapter is effectively uninstalled here; `rcorn skills install` should fix it.
