# Golden-file tests

Marshal-to-JSON comparison against `testdata/<name>.golden.json`, regenerated with `-update`.

Goldens pin the **projection** — units, IRIs, citations — never rendered bytes. Keep them small and semantic: one fixture per shape or failure mode, and a golden diff must read as a legal statement.

A rubber-stamped golden is worse than none — always review the diff before committing a regeneration.

**Run:** `go test ./internal/source` to verify · `go test ./internal/source -update` to regenerate (then review the diff). Same for `./internal/akn`.

## The harness

Two twin implementations of `check_golden`, both gated on a package-local `var update_golden = flag.Bool("update", …)`: [internal/akn/golden_test.go](../../../../internal/akn/golden_test.go) at line 14 and [internal/source/harness_test.go](../../../../internal/source/harness_test.go) at line 26.

```go
func check_golden(test *testing.T, name string, value any) {
	test.Helper()
	got, error := json.MarshalIndent(value, "", "  ")
	if error != nil {
		test.Fatal(error)
	}
	got = append(got, '\n')
	path := filepath.Join("testdata", name+".golden.json")
	if *update_golden {
		if error := os.WriteFile(path, got, 0o644); error != nil {
			test.Fatal(error)
		}
		return
	}
	want, error := os.ReadFile(path)
	if error != nil {
		test.Fatalf("read golden %s: %v (run: go test -update)", path, error)
	}
	if string(got) != string(want) {
		test.Errorf("%s mismatch (run: go test -update)", name)
	}
}
```

Goldens are named by scenario slug (`plain-ley`, `anexo-decreto`, `format-era-legacy`, `fallo-divisions`).

`internal/akn` goldens are built from hand-authored `Act`/`Judgment` structs; `internal/source` goldens decode a `testdata/*.html|json|pdf` fixture first, then golden the resulting units (see the `TestDecode_Goldens` table in [unit.md](unit.md)).

## Pending specimens — the table harness

[internal/source/pending_test.go](../../../../internal/source/pending_test.go) holds two tables, `pending_cases` and `documented_defects`, not a filename-prefix switch.

Each row names committed `testdata/` bytes (`specimens`) and a **chain** function that re-runs the real indexer pipeline slice from raw bytes (decode → parse → units → span verify), so deep conditions (zero units, span mismatch) surface as errors instead of hiding behind a single decode call.

`pending_cases` holds specimens the decoder must eventually handle: each row is **red until the decoder is fixed** and carries a `represents` count (how many corpus failures that specimen stands for), so the table's length is the open-defect census.

A row's `represents` is the cluster count from a live census pass. A count taken any other way, from a research note or before a rebuild, carries its date and source in [PROVENANCE.md](../../../../internal/source/testdata/PROVENANCE.md) beside the specimen, and the row holds the number only.

The assertions nest. The chain must succeed always; the moment `testdata/pending/<name>.golden.json` exists, the row also **golden-matches**, and before that chain success is the only assertion, so the green flip is the graduation signal.

```go
value, error := item.chain(specimens...)
if error != nil {
	return fmt.Errorf("decoder does not yet handle %s (represents %d corpus failures): %w", item.name, item.represents, error)
}
…
want, error := os.ReadFile(path)
if os.IsNotExist(error) {
	return nil
}
```

`TestPendingCases_RedUntilFixed` runs the table; `go test ./internal/source -update` writes the golden once the decoder handles the row.

`documented_defects` pins **source-side** failures (a truncated attachment, a body the source never published): green only while the failure stays **exactly as classified** (the `contains` fragment), red the moment it drifts or heals. `TestDocumentedDefects_PinnedClassification` runs it, and each row carries a dated `reason` verdict.

Each chain function in the file mirrors the indexer slice call-for-call. `ocr_judgment_chain` and `appellate_docket_chain` spell the refusals they raise themselves `index settles <outcome>: <cause>` with the index outcome token, so a documented-defect `contains` fragment matches the census outcome that ties a row to its cluster. The zero-units refusals of `act_chain` and `judgment_chain` keep their own wording, and an error from anything a chain calls, a decoder, an extractor or `judgment_chain`, passes through unchanged. A slice that reads bytes an acquire leg derives re-derives them the way that leg does, as `ocr_judgment_chain` OCRs the stored attachment through a copy of the acquire OCR assembly at its confidence floor. A new deep condition gets a new chain, not a new dispatch case.

**Graduation**: when the decoder fix lands, the specimen and its reviewed golden move out to the permanent per-source scenario table in the **same commit** as the fix — a promoted pair is a permanent regression test, and a row left in the pending table after its fix rots.

The runner mechanics (`evaluate_pending_case`, `evaluate_documented_defect`) are themselves pinned by fake-chain tests in the same file (`TestEvaluatePendingCase_*`, `TestEvaluateDocumentedDefect_*`), no fixtures.

## Where specimens come from — the acquisition-table census

Golden fixtures are minted from real crawl failures.

The census is the `acquisition_status` view, and the curation of census rows into table rows is [red-ledger/SKILL.md](../../../../.claude/skills/red-ledger/SKILL.md). Three pieces feed it.

A **smoke crawl**, `just acquire smoke <type> [cap]` (cap defaults to 25; the types are the acquire drains in [just/SKILL.md](../../../../.claude/skills/just/SKILL.md)), is a bounded live crawl that prints this run's seam delta `N decode-golden · M network-signal`, read from the store before and after.

```
just acquire smoke <type> [cap]
```

The **failure census** is a live query over failed acquisition rows.

```
just db query "SELECT error, seam, count(*) FROM acquisition_status WHERE status='failed' AND seam <> 'network-signal' GROUP BY error, seam"
```

Each failed row carries `error` (the globally-unique event name) and `seam`. The vocabulary lives in [internal/event/event.go](../../../../internal/event/event.go): `event.Event{Name, Seam, Params, Engine}` with four seams — `decode-golden` (deterministic decoder refusal, harvestable), `network-signal` (transient, retriable, excluded from the census), `coverage-gap`, `data-defect`.

Free-text detail (params, Go error text) lives in telemetry (`just obs logs`), not in any file.

Event names are globally unique and schema-checked by AST-walking tests in [internal/event/catalogue_test.go](../../../../internal/event/catalogue_test.go).

**Curation** resolves specimen bytes from the failed row's own `id` via the document keys (`<id>#document`, `<id>#evidence`, `<id>#record`) or the row's `url`; they are copied by hand into `testdata/pending/` during red-ledger curation, and the table row is the registration.
