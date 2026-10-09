# Real-model tier (`-tags real_model`)

Semantic-recall assertions against the **live Python embedder** — proves hybrid search beats BM25 on real embeddings, not the deterministic `test_vector` fakes. [integration/semantic_recall_test.go](../../../../integration/semantic_recall_test.go), behind `//go:build real_model`.

**Run:** `just test -tags real_model ./integration/... -run TestSemanticRecall`

## Prerequisites

- `uv` installed and the Python embedder at `apps/embedder`.
- Postgres up (`just stack up`) — the test calls `open_search_store`, so `RETRIEVAL_ADMIN_DSN` gating applies (see [integration.md](integration.md)); `just test` loads it from `.env`.

## How it boots

`boot_real_embedder` (line 77) execs `uv run --directory ../apps/embedder python -m embedder --mode serve --port 18091` (`REAL_EMBEDDER_PORT`, line 24), polls TCP `127.0.0.1:18091` until bound (`REAL_EMBEDDER_BOOT_TIMEOUT = 120s`, line 25), and registers a `test.Cleanup` that SIGTERMs and Waits the process.

The corpus, Ley 25.326, six CSJN fallos and the expression twin Ley 99.001, is embedded live (`seed_corpus_live`, line 192) and queries embedded per case (`embed_query`, line 217).

## What it asserts

The cases are the fixture qrels set in [integration/testdata/qrels/fixture/](../../../../integration/testdata/qrels/fixture/), BEIR's `queries.jsonl` beside a `qrels.tsv` of query-id, corpus-id and score, read by `qrels.Load` ([internal/qrels](../../../../internal/qrels/)), which panics on a malformed row. A corpus-id is a unit IRI, or an expression IRI that judges a fallo whole, and its AKN type picks the genus section the case reads.

A query's `metadata` carries one of two flags. A control case is the lexical baseline: the unit must already rank within `CONTROL_RANK_CEILING = 5` under BM25 alone. A proven case is the semantic win: hybrid must rank the unit better than lexical does.

The expression-twin case q11 judges Ley 99.001's consolidation, while its original expression is stored but not current: for every query, no section of either lane may carry a unit of the original (`TWIN_ORIGINAL`), so only the consolidation answers.

Add a case when a real query regresses: a `queries.jsonl` line with the query and its control or proven flag, and a `qrels.tsv` row with the relevant unit or fallo.

## Why tagged

Boot (~up to 2 min) plus live embedding makes it the slowest tier; the tag criterion is time, not infrastructure.
