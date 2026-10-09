---
name: probe
description: "Settle an empirical question by writing and running throwaway code in /probe/. Use when a claim about the codebase, its data, or a live source can be answered by running a script or test, or when another skill (/implement) needs runnable evidence."
---

# Probe

A probe settles an empirical question by running code. Everything it produces, script, scratch data and output, lives in the gitignored `/probe/` directory at the repo root and nowhere else in the tree. The dividing line: a committed `_test.go` beside its package protects behaviour the codebase must keep, while a probe answers a question once.

## Small and fast

A probe exercises the narrowest slice of code or data that makes the point and finishes in seconds, a minute at the outside.

It never sweeps the whole raw corpus or the whole database; when the question seems to need a full sweep, a handful of representative documents is sampled instead, or the maintainer is asked, because a corpus-wide run is a maintainer-run operation that neither a probe nor a plan citing one does on its own.

## Total liberty, zero pollution

Inside `/probe/` anything goes: Go, Python, Bash, Go tests, whatever settles the question fastest.

Production code is duplicated freely rather than refactored for reuse, because the point is zero pollution of the real tree; a probe may import and call source packages but never modifies source, and when reuse would require refactoring production code the code is duplicated inside `/probe/` instead.

## Written for a stranger to re-run

Every probe filename carries the timestamp it was written, `YYYYMMDD-HHMM-`, so the directory lists in chronological order, and captured data and output files sit beside the script sharing its prefix.

```
probe/20260829-1503-anexo_census.go
probe/20260829-1503-anexo_lines.tsv
```

A Go probe carries a `//go:build probe_<tag>` tag, where the tag stays `probe_<tag>` without the timestamp, and its header comment states the question it answers and a `Run:` line with the exact command, which uses the prefixed filename.

```go
//go:build probe_anexo

// Question: how many anexo lines does each sampled norm carry?
// Run: go run -tags probe_anexo ./probe/20260829-1503-anexo_census.go > probe/20260829-1503-anexo_lines.tsv
```

Each probe runs in isolation, needing nothing but its own files in `/probe/` and the source packages it imports: no edit to the tree, no hand setup, no sibling probe.

## Disposable, promoted only as evidence

Probes stay in place after answering as a local trail, but nothing links to `/probe/`.

When a plan rests on the answer, the proving core is promoted to committed evidence under `evidence/<issue>/`, in the format [evidence.md](../.agents/evidence.md) fixes.

The completion criterion is the question answered with the probe's actual output quoted, and the script sitting in `/probe/` with its `Run:` line so any session reproduces the answer.
