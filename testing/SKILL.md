# Testing Strategy

The monorepo's test taxonomy. Each tier has a complete reference file — read the one matching your task before writing or running that kind of test.

| Tier | When it fires | Reference |
|------|---------------|-----------|
| Unit | Any package logic: decoders, parsers, HTTP clients, drains; offline, no running service | [unit.md](types/unit.md) |
| Integration (`-tags integration`) | Anything against a real ParadeDB/Postgres store | [integration.md](types/integration.md) |
| Golden | Decoder output pinned as `testdata/*.golden.json`; the pending → graduate fixture pipeline (vocabulary: [red-ledger/SKILL.md](../../../.claude/skills/red-ledger/SKILL.md)) | [golden.md](types/golden.md) |
| Qrels | Retrieval relevance judgements under `integration/` — TREC/BEIR `(query_id, doc_id, relevance)`, never called goldens | [retrieval-telemetry/SKILL.md](../../../.claude/skills/retrieval-telemetry/SKILL.md) |
| Real model (`-tags real_model`) | Semantic recall against the live Python embedder | [real_model.md](types/real_model.md) |

## Entrypoint

`just test *args` is the single entrypoint: it defaults the target to `go test ./...` and, in a linked worktree, exports `TEST_DATABASE_SUFFIX` (`_<worktree dir>`) so each worktree's Postgres tests use their own scratch databases.

The justfile loads `.env`, so `RETRIEVAL_ADMIN_DSN` (the paradedb container on port `5434`) reaches the tests from there.

Every tier is selected purely by the args you pass.

```
just test                              unit tier, offline, seconds
just test -tags integration ./...      unit + Postgres tiers
just test -tags real_model ./integration/...
just test "./internal/akn -run TestX"
```

A `-run` that names a test living in a tagged file needs the tag too: without it `go test` compiles the file out and reports `ok … [no tests to run]`, a green that ran nothing.

```
just test "./internal/store/ -run TestSearch_Bands"                    [no tests to run]   ← false green
just test "-tags integration ./internal/store/ -run TestSearch_Bands"  runs it
```

## Always run the most localized set — the full tagged run is a once-per-task gate

Bare `just test` is the offline unit tier and runs in seconds, so it is fine in the inner loop. The full run, `just test -tags integration ./...`, is the once-per-task gate: **run it at most once per task, as the final gate right before `/code-review`, never as an inner-loop check.**

During the red→green loop, run the *narrowest* selection that exercises what you just changed, and no wider.

```
just test "./internal/source/ -run TestDecode_Goldens"    one test
just test ./internal/source/                              one package
just test -tags real_model ./integration/...              one tier
```

Quote the whole arg string when it contains `-run`/flags so it reaches `go test` intact.

After a one-line edit, re-run the one test or the one package — not `./...`. Widening the selection "just to be safe" after every small change is the anti-pattern: it burns minutes per iteration for no signal a targeted run wouldn't give.

The full sweep's job is to catch cross-package fallout once, at the end.

## House style (universal, every tier)

- Receiver is `test *testing.T` — never `t`.
- The error variable is `error` — never `err` (shadowing the builtin is the deliberate convention).
- Helpers are `snake_case` and call `test.Helper()` on the first line.
- Test funcs are `TestSubject_BehaviorInCamelCase`; constants are `SCREAMING_SNAKE_CASE`.
- Assertions are hand-rolled `if got != want { test.Errorf(...) }` — no testify, no assert library.

## Tiering rationale

The tag criterion is **infrastructure**: an untagged test needs nothing running, no Postgres, no embedder, no network, so bare `just test` is offline and runs in seconds. A test that opens a scratch database lives in a file behind `//go:build integration`; a test that needs the live Python embedder lives behind `//go:build real_model`. A file mixing both kinds is split, the Postgres half going to `<name>_integration_test.go`.

```go
//go:build integration

package store
```

`integration/` is the only global test dir; tiers inside it are build tags, never subdirectories.

A test lives at the narrowest scope that can exercise the behavior, and its data lives next to it; the pipeline-specific tests are in [pipeline.md](../../../docs/agents/pipeline.md) under "Testing".

Postgres tests run on every task's final gate because ranking correctness lives in SQL (RRF, bands, the authority tiebreak) and cannot be exercised in Go alone, so they run against a scratch database with a faked embedder. Mocking the database is forbidden: it tests nothing.

Acquire is tested lightly by design, since it is fetch, rate limit and ledger writes; everything that looks like acquire testing (carátula matching, volume extraction, SAIJ decode, PDF fixture equality) is offline function testing and carries the weight.

## Fixtures hold exactly what the test needs

A fixture holds only the slice of the document that proves its test's behaviour: the few articles, the one anexo, the pages a caption sits on, never the whole code, tomo or fallo. Decoding a whole document is what made unit tests take seconds; an excerpt decodes in milliseconds and its golden reads at a glance.

[tools/fixtures/size_test.go](../../../tools/fixtures/size_test.go) fails on any git-tracked `testdata/` file over 128 KB, with no exception list; hitting it means a document was committed where an excerpt belongs.

```
testdata/ley-25326.html                80 KB   ok
testdata/codigo-ccyc-consolidado.html  1.2 MB  FAIL: keep only the slice of the document the test needs
```

The cap catches the gross case; whether a fixture under it is still more than its test needs is judged in review, and a unit test over about 100ms is the sign to look.

Goldens are small and semantic: one committed fixture per document shape or proven failure mode, asserting the projection (units, IRIs, citations), never rendered bytes. A golden diff reads as a legal statement ("article 5 moved into the anexo"); if it needs scrolling, the golden is too big, and a rubber-stamped golden is worse than none.

## No corpus sweeps — ever

There is **no soak tier** and it must not come back. No test, probe, plan gate, or acceptance criterion may replay, decode, or diff the entire raw corpus (or the whole database).

Regressions are pinned the TDD way: a minimal fixture reproducing the failure, promoted into goldens/unit tests.

If a check needs more than ~1 minute of runtime, it is not a test — it is a maintainer-run operation that only the maintainer may request explicitly. Never write a plan or issue whose gate is "full-corpus run/diff green".

## Tests prove code, documentation is read

A test's subject is code and the fixtures it reads. Documentation, from [CONTEXT.md](../../../CONTEXT.md), [README.md](../../../README.md), `docs/` (the naming catalogue and the ADRs included) to a skill or an agent file, is checked by reading it in review, and no test opens a document to assert its wording, keys, names or links.

```
test     internal/akn/closing_corpus_test.go reads testdata/corpus/law/*.md    a fixture, code's input
review   docs/agents/names.yaml, CONTEXT.md, .claude/skills/*                  read, never tested
```

## Probes — not a test tier

Throwaway diagnostics live in gitignored `/probe/` behind a `//go:build probe_<name>` tag, per [probe/SKILL.md](../probe/SKILL.md); a committed test belongs in a `_test.go`.

## Process

Red→green loop, seams, anti-patterns: [tdd/SKILL.md](../tdd/SKILL.md).
