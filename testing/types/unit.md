# Unit tests

Colocated `*_test.go` beside source, one per package, with fixtures in a sibling `testdata/`.

A test lives at the narrowest scope that can exercise the behavior.

About 230 files; densest packages: `internal/akn`, `internal/source`.

**Run:** `just test` (the whole untagged suite, offline, seconds) or scoped: `just test ./internal/akn`, `just test "./internal/source -run TestDecode"`. A unit test needs no running service; one that opens a scratch database belongs to the integration tier.

## Shape 1 — named scenario

One `Test…` func per behavior, hand-built inputs, direct assertion. Dominant in decoders and pure functions. From [internal/httpx/httpx_test.go](../../../../internal/httpx/httpx_test.go) at line 5:

```go
func TestToUTF8Mapped_OffsetsPointAtSourceBytes(test *testing.T) {
	body := []byte{'l', 'e', 'y', ' ', 0xE1, 0x96, 'x'}
	text, raw_of := ToUTF8Mapped(body)
	if text != "ley á–x" {
		test.Fatalf("text = %q", text)
	}
	if raw_of[len(text)] != len(body) || string(body[raw_of[4]:raw_of[4]+1]) != "\xe1" {
		test.Fatalf("offset map broken: %v", raw_of)
	}
}
```

The func name is the spec: `TestSubject_BehaviorInCamelCase`, readable as a sentence.

Prefer several small named tests over one broad one — each failure names its behavior.

## Shape 2 — table-driven

`cases := []struct{…}` with **named fields**, iterated as `for _, scenario := range cases` with `test.Run(scenario.name, …)` subtests. From [internal/source/infoleg_test.go](../../../../internal/source/infoleg_test.go) at line 25:

```go
func TestDecode_Goldens(test *testing.T) {
	cases := []struct {
		name string
		file string
		row  Row
	}{
		{name: "plain-ley", file: "ley-25326.html", row: Row{Type: "Ley", Number: "25326", EnactmentDate: "2000-10-04"}},
		{name: "anexo-decreto", file: "decreto-1558.html", row: Row{Type: "Decreto", Number: "1558", EnactmentDate: "2001-11-29"}},
	}
	for _, scenario := range cases {
		test.Run(scenario.name, func(test *testing.T) {
			units := act_units(test, decode_one(test, scenario.row, scenario.file))
			if len(units) == 0 {
				test.Fatalf("%s produced no units", scenario.name)
			}
			check_golden(test, scenario.name, units)
		})
	}
}
```

The loop variable is always `scenario`, never `tc`/`tt`.

Use a table when cases differ only in data; use named scenarios when they differ in setup or assertion.

## Package-local helpers

Each package grows small `snake_case` helpers that call `test.Helper()` first and `test.Fatalf` on any setup failure, so tests read as intent only. Exemplars: `decode_one` ([internal/source/infoleg_test.go](../../../../internal/source/infoleg_test.go) line 13), `read_fixture` ([internal/source/harness_test.go](../../../../internal/source/harness_test.go) line 17).

## HTTP clients — httptest, never the network

The idiom is a `snake_case` fake-server helper that registers `test.Cleanup(server.Close)` and returns `server.URL`. From [internal/embed/embed_test.go](../../../../internal/embed/embed_test.go) at line 16:

```go
func fake_embedder(test *testing.T, reply func(request embed_request) embed_reply) string {
	server := httptest.NewServer(http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
		var parsed embed_request
		if error := json.NewDecoder(request.Body).Decode(&parsed); error != nil {
			http.Error(writer, error.Error(), http.StatusBadRequest)
			return
		}
		json.NewEncoder(writer).Encode(reply(parsed))
	}))
	test.Cleanup(server.Close)
	return server.URL
}
```

Used across 37 test files (`apps/acquire`, `apps/index`, `apps/server`, `integration`, `internal/embed`, `internal/httpx`, `internal/jev`, `internal/llm`, `internal/raw`, `internal/route`).

For the connection-refused path: `server := httptest.NewServer(nil)` then `server.Close()` before calling ([internal/embed/client_test.go](../../../../internal/embed/client_test.go) line 27).

## Crawl drains — function injection

`apps/acquire` drains are tested by injecting fetch/post functions into the engine struct and asserting ledger status transitions, raw-store keys, and counters (`crawler.got`, `acquire.failures`) — no httptest needed. Exemplar: `TestDrainLaws_ResolvesBodylessBOAndFailsUnresolved` ([apps/acquire/laws_test.go](../../../../apps/acquire/laws_test.go) line 46).
