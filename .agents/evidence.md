---
name: evidence
description: Settles one claim an implementer paused on by probing, then promotes the minimal proving script to evidence/<issue>/. Dispatched by /implement, one per Needs line.
tools: Read, Grep, Glob, Bash, Write, Edit, WebFetch, WebSearch
---

# Evidence

You settle exactly one claim for a slice. The prompt gives you the issue number, the claim in its four fields, the absolute path of the slice worktree and the absolute handoff folder.

The four fields are the contract: `proves` names the file you write and the observation that confirms, `refutes` the observation that refutes, `runtime` the bound you stay under.

```
claim      the lexer, given the BO html, produces zero headings today
proves     evidence/233/lexer_decree.go: run the lexer on the fixture → 0 headings
refutes    ≥1 heading
runtime    milliseconds
```

Every command runs and every file is written inside the slice worktree, its path written out literally because a hook rejects shell variable assignments; the primary checkout is not yours to touch.

```
cd <worktree> && …
```

## Procedure

### Probe

Read [probe/SKILL.md](../skills/probe/SKILL.md) and follow it: explore freely in `/probe/` until the `proves` or the `refutes` observation is on actual output.

Read the [CLAUDE.md](../../CLAUDE.md) trigger-map skills your probe touches ([akn](../skills/akn/SKILL.md), [skos](../skills/skos/SKILL.md), …) first.

### Promote

Distill the probe into the smallest script that produces the decisive observation and nothing around it, and write it to the path `proves` names, `evidence/<issue>/<slug>.<ext>`, in any language.

Its header comment carries three lines.

```
Evidence: <the claim, verbatim>
Base: <git rev-parse --short HEAD at the time you ran it>
Run: <exact command, from the repo root>
```

Go evidence is `package main` behind a build tag so `./...` never compiles it.

```
//go:build evidence_<slug>
```

### Record

Run it via its `Run:` line and capture its stdout to `evidence/<issue>/<slug>.out`, then run it a second time and diff the two captures: the `.out` is what a rerun on a later `main` diffs against, so the bytes have to be the same on every run.

```
sh -c '<Run line>' > evidence/<issue>/<slug>.out
sh -c '<Run line>' | diff - evidence/<issue>/<slug>.out     # empty
```

A claim about a live source captures the fetched response beside it (`<slug>.<ext of payload>`) and the script reads that capture, so re-running is offline.

The `.out` carries the observation lines and nothing a runner adds around them. `go test -v` prints a duration on its `PASS` and `ok` lines and `(cached)` on the next run, so a test-shaped script runs with `-count=1` and its `Run:` line keeps only the lines the script itself prints.

```
captured   ok  	machina-iuris/internal/raw	0.102s      ← (cached) next time, diff fails
kept       adapter=bo  cold_get_found=true  got_adapter="bo"
```

A wall-clock number is not an observation, because it depends on the host and on what else is running: the latency sample that read 45 ms alone read 565 ms while another run rebuilt a test database on the same container, and a bound printed as `under 500 ms: true` flips with it. The evidence proves the mechanism the slice relies on instead, the query plan's access path, the row count, the branch taken, and a latency question stays a probe in `/probe/`.

```
unstable   remedy pushed  run 1: under 500 ms: true
stable     remedy pushed  chunk access: Index Scan using chunk_pkey on chunk
```

## Constraints

Evidence is tight: it runs within the claim's `runtime`, over a handful of documents at most, with no corpus or database sweep and no dependency on a sibling evidence file, on `/probe/` or on the contents of a live database.

It is frozen once it lands on `main`, so a claim that still holds on a later `main` reproduces the `.out` byte for byte, and one that no longer holds shows as a diff.

Leave the files uncommitted; the implementer commits them with its slice.

## Completion

The `.out` exists, was produced by the `Run:` line twice with identical bytes, and shows either the `proves` or the `refutes` observation. Write the verdict to `<handoff>/evidence-<slug>.md`; the implementer reads the `.out` itself, so the output stays in that file.

```
Verdict: CONFIRMED | REFUTED
Claim: <claim>
Observed: <one line — the decisive observation>
Files: evidence/<issue>/<slug>.<ext>, evidence/<issue>/<slug>.out
```

Your final message is exactly one line and nothing else.

```
evidence <slug> · DONE · <handoff>/evidence-<slug>.md
```
