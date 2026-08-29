---
name: probe
description: "Settle an empirical question by writing and running throwaway code in /probe/. Use when a claim about the codebase, its data, or a live source can be answered by running a script or test, or when another skill (/plan) needs runnable evidence."
---

# Probe

A probe settles an empirical question by running code. Everything it produces — script, scratch data, output — lives in the gitignored `/probe/` directory at the repo root, never anywhere else in the tree. The dividing line: a committed `_test.go` beside its package protects behaviour the codebase must keep; a probe answers a question once.

A probe is **small and fast**: it exercises the narrowest slice of code or data that makes the point, and it finishes in seconds — a minute at the outside. A probe never sweeps the whole raw corpus or the whole database; if the question seems to need a full sweep, sample a handful of representative documents instead, or stop and ask the maintainer — a corpus-wide run is a maintainer-run operation, never something a probe (or a plan citing one) does on its own.

Inside `/probe/` liberty is total: Go, Python, Bash, Go tests — whatever settles the question fastest. Duplicate production code freely instead of refactoring it for reuse; the point is zero pollution of the real tree. A probe may import and call source packages, but never modifies source: the real tree stays untouched. If reuse would require refactoring production code, duplicate it inside `/probe/` instead.

Every probe is written to be re-run by a stranger:

- Go probes carry a `//go:build probe_<tag>` tag.
- The header comment states the question the probe answers and a `Run:` line with the exact command (e.g. `Run: go run -tags probe_anexo ./probe/anexo_census.go > probe/anexo_lines.tsv`).
- Captured data and output files stay beside the script.
- Each probe runs in isolation: it needs nothing but its own file(s) in `/probe/` and the source packages it imports — no edit to the tree, no hand-setup, no sibling probe.

Probes are evidence, not litter: leave them in place after answering. Future sessions read them as reference for how a question was settled, and plans link to them under `/probe/`.

Completion criterion: the question answered with the probe's actual output quoted, and the script sitting in `/probe/` with its `Run:` line so any session can reproduce the answer.
