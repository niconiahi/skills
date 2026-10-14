# Integration tests

`integration/` package — the only global test dir. Task-named files, each testing one user task end-to-end against a real ParadeDB/Postgres store: `bloque_tier_completion_test.go`, `chunk_scoring_test.go`, `citation_stability_test.go`, `decree_body_fallback_test.go`, `fusion_test.go`, `law_body_fallback_test.go`, `piramide_ranking_test.go`, `quote_grounding_test.go`, `reset_rebuild_test.go`, `retrieval_test.go`, plus `main_test.go` and the tagged `semantic_recall_test.go`.

Mocking the DB is forbidden (locked principle: ranking correctness lives in SQL and can't be Go-unit-tested).

**Run:** `just test -tags integration ./...` for the whole tier, or scoped, `just test "-tags integration ./internal/store -run TestSearch"` — Postgres must be up (`just stack up`). `just test` loads `.env`, which carries `RETRIEVAL_ADMIN_DSN`.

Every file whose tests open a scratch database starts with `//go:build integration`, so bare `just test` never compiles them and stays offline. A package mixing pure and Postgres tests keeps the pure ones in `<name>_test.go` and the Postgres ones in `<name>_integration_test.go`.

## Harness — `integration/harness_test.go`

Every test starts with `open_search_store`, which hands the package's database name to [storetest.Scratch](../../../../internal/store/storetest/storetest.go) and pins tuning ([integration/harness_test.go](../../../../integration/harness_test.go) at line 20).

The database name takes the per-worktree suffix `just test` exports, so parallel worktrees never collide.

```go
var integration_database = "machina_iuris_integration_test" + os.Getenv("TEST_DATABASE_SUFFIX")

func open_search_store(test *testing.T) *store.Store {
	test.Helper()
	database, _ := storetest.Scratch(test, integration_database)
	database.Tuning = store.Tuning{HybridBandWidth: 0.0001, LexicalBandWidth: 0.5, EfSearch: 200}
	return database
}
```

`Scratch` skips the test when `RETRIEVAL_ADMIN_DSN` is unset. Its first call in a test binary takes a Postgres advisory lock on the database name, held on a pinned connection until the process exits, then builds `<database>_template` and bootstraps it through `store.Open` (migrate plus vocabulary reconcile). Every call after drops the scratch database, recreates it as a copy and attaches with `store.Connect`, which opens the connection and nothing else, so a test pays for a copy, never a bootstrap.

```
first call   pg_try_advisory_lock(hashtext('machina_iuris_integration_test'))
             CREATE DATABASE machina_iuris_integration_test_template  →  store.Open
every call   DROP DATABASE machina_iuris_integration_test WITH (FORCE)
             CREATE DATABASE machina_iuris_integration_test TEMPLATE machina_iuris_integration_test_template  →  store.Connect
```

The lock makes a second live run of the same package in the same checkout fail at once instead of force-dropping the first run's databases under it; Postgres releases the lock when the holding process's connection closes, so a killed run leaves nothing behind.

```
another test run is still using machina_iuris_index_test and its template in this checkout; wait for it to finish or stop it before running again
```

The copy uses the default `WAL_LOG` strategy, the efficient one for a small template per the PostgreSQL docs; `FILE_COPY` forces a checkpoint before and after every copy. A fresh copy per test keeps the rule of no shared state between tests, ever.

Every helper calls `test.Helper()` and `test.Fatalf` on failure. The ones the `real_model` tier shares stay in [harness_test.go](../../../../integration/harness_test.go); the ones only the integration tier calls live in [harness_integration_test.go](../../../../integration/harness_integration_test.go).

- `seed_units(test, database, works, units)` — upserts works + units, then `Consolidate` (`harness_integration_test.go:15`; delegates to `seed_units_with_vectors(…, nil)`).
- `seed_units_with_vectors(test, database, works, units, chunks map[string][][]float32)` — same with per-unit chunk vectors (`harness_test.go:27`).
- `must_search(test, database, query) []store.Result` — `SearchLexical(query).Legislacion` (`harness_integration_test.go:20`).
- `must_hybrid(test, database, query, query_vector) []store.Result` — `SearchHybrid(...).Legislacion` (`harness_integration_test.go:29`).
- `test_vector(seed float32) []float32` — deterministic 768-dim vector, all `seed` with `vector[0] = 1` (`harness_integration_test.go:43`).
- `result_ids(results) []string` (`harness_integration_test.go:52`), `rank_of(results, id) int` — index or -1 (`harness_test.go:48`), `same_id_set(first, second) bool` (`harness_integration_test.go:69`).

A test therefore reads: `database := open_search_store(test)` → `seed_units(...)` → `must_search(...)` → assert with `rank_of` / `same_id_set`.

## Scratch DBs outside `integration/`

Postgres tests also live beside their package, behind the same tag, each calling `storetest.Scratch` with its **own** DB name — one scratch DB per package: `machina_iuris_acquire_test`, `machina_iuris_index_test`, `machina_iuris_server_test`, each with the `TEST_DATABASE_SUFFIX` appended. A new Postgres package calls `storetest.Scratch` with a name of its own, never another package's. [internal/store/store_test.go](../../../../internal/store/store_test.go) (`machina_iuris_test`) keeps a local copy of the same lock, template and copy steps, because `package store` importing `storetest` would be an import cycle.

## No-Postgres integration

[integration/quote_grounding_test.go](../../../../integration/quote_grounding_test.go) is the odd one: it spins an httptest fake LLM (`llm.Client{BaseURL: server.URL, APIKey: "test-key"}`, line 51) and needs no store — integration means "one user task end-to-end", not "needs a database".

There is deliberately **no time-boxed full-pipeline integration test and no corpus-sweep tier**: retry and decode paths are pinned by fast deterministic tests over minimal fixtures, never by replaying the whole corpus ([pipeline.md](../../../../docs/agents/pipeline.md) under "Testing").
