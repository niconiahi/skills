---
name: plan
description: "Planning session for a GitHub issue labeled ready-for-planning: research until every change is pinned to a file, post the plan as an issue comment, flip the label to ready-for-agent."
disable-model-invocation: true
---

# Plan

Takes an issue number. Requires the `ready-for-planning` label — if the issue carries `ready-for-agent` instead, stop: it is already planned; run /implement.

This session exists to absorb the research cost so the implementation session starts cold and clean. Spend tokens freely.

**This skill never edits source code.** Its only outputs are evidence — `/probe` scripts under `/probe/`, `/research` docs under `docs/research/` — and the plan comment that links to them. No production file is created or modified in a planning session; every change the plan describes is executed later by `/implement` in a fresh session. If a question can only be answered by running code, that code is a probe in `/probe/`, never an edit to the tree.

1. Read the issue in full (body, comments, labels), then explain it back succinctly before doing anything else.
2. Research until every change is pinned to a file: codebase, `CONTEXT.md`, ADRs, official docs. Producing evidence is encouraged, not incidental: an open question about sources, docs, or facts → fire /research (lands in `docs/research/`); an empirical question the codebase or a live source can answer → fire /probe (lands in `/probe/`). These artifacts are what step 3 links to.
3. Post the plan as an issue comment. Completion criterion — the **cold-start test**: a fresh session reading only the issue description and this comment can implement with zero further research. That means every file to touch named with its concrete change, in execution order; the test seams where /tdd will run; commands to run; how to verify; and links to the evidence the plan rests on — research docs under `docs/research/`, probe scripts or probe tests under `/probe/`.

   **Gates are fast, TDD-shaped checks.** A plan's verification and acceptance gates are `just test` green, goldens, and targeted unit/integration tests — never a full-corpus replay, corpus-wide diff, or whole-database sweep. Probes cited as evidence run in seconds over a narrow slice (see the probe skill); anything longer is a maintainer-run operation the plan may only *mention* as optional, never require.

   **Speak code, not prose about code.** The issue body already carries the text; the plan comment is where snippets live. Every step that changes code shows the change as a snippet or diff against the real file (read the file first so the snippet matches actual lines) — never a paragraph describing what the code will do. Tests show their assertion code, not a description of what they assert. Prose is reserved for what code can't show: decisions, scope boundaries, why something is untouched. Reference: https://github.com/niconiahi/machina-iuris/issues/414#issuecomment-5308901342
4. Flip labels: remove `ready-for-planning`, add `ready-for-agent`.

Stop. Implementation happens in a fresh session via /implement.
