# Go Naming Conventions

Every Go identifier in this repo is a full English word or phrase, cased by its role: `SCREAMING_SNAKE_CASE` for constants, error variables and package-level variables never written after init, `PascalCase` for exported functions and struct fields, `snake_case` for everything unexported (functions, parameters, locals, mutable package-level state). Spanish never enters an identifier, and no comment enters the code. The sections below show each rule as code.

## Full Names Over Abbreviations

Always prefer full, descriptive names over contractions.

```go
// Good
error, user, config, response, request, context, message, result

// Bad
err, usr, cfg, resp, req, ctx, msg, res
```

The same rule applied to a whole function.

```go
// Good
func GetUser(user_id string) (*User, error) {
    user, error := db.Find(user_id)
    if error != nil {
        return nil, error
    }
    return user, nil
}

// Bad
func GetUser(uid string) (*User, error) {
    u, err := db.Find(uid)
    if err != nil {
        return nil, err
    }
    return u, nil
}
```

## Constants

Use `SCREAMING_SNAKE_CASE` for all constants, exported or not.

```go
const MAX_RETRIES = 3
const DEFAULT_TIMEOUT = 30
const ERROR_INVALID_FORMAT = "invalid format"
```

## Exported Functions/Methods

Use `PascalCase`.

```go
func CreateNewUser(name string) *User
func GetActiveLicense() (*License, error)
func ParseConfig(path string) (*Config, error)
```

## Internal (Unexported) Functions

Use `snake_case`.

```go
func download_tarball(url string) (string, error)
func extract_docs(path string) error
func validate_input(data []byte) bool
```

## Parameters and Local Variables

Use `snake_case`.

```go
func CreateUser(user_name string, is_admin bool) *User {
    user_id := generate_id()
    created_at := time.Now()
    return &User{
        id:   user_id,
        name: user_name,
    }
}
```

## Struct Fields

Use `PascalCase` for exported fields (required for JSON, etc).

```go
type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}
```

## Package-Level Variables

A package-level variable that is never written after init, a read-only lookup table, set or map, uses `SCREAMING_SNAKE_CASE`, because it is a constant Go cannot declare `const`.

```go
var RANGE_WORDS = map[string]bool{"a": true, "al": true}
var DOCUMENT_ADAPTERS = []string{"saij", "bo", "infoleg", "csjn", "cij"}
```

Mutable package-level state, anything assigned, appended to or filled after init, uses `snake_case`.

```go
var error_cache = make(map[string]error)
var template_once sync.Once
```

## Error Variables

Use `SCREAMING_SNAKE_CASE`, because they are constants.

```go
var ERROR_NOT_FOUND = errors.New("not found")
var ERROR_INVALID_INPUT = errors.New("invalid input")
```

## Identifiers Are English

Every Go identifier (types, funcs, fields, consts, params, locals) is English. Spanish survives only in non-identifier positions: string literals, struct tags (`json:"actor"`), regex patterns, presentation labels (`"Fallo"`, `"Sumario"`), and data cells. This is a hard boundary: rename a name, never a value.

Two audited exceptions exist where a stored value is also English, because it round-trips a contract: the census/class tokens `"judgment"`/`"volume_part"` (ledger `class` column, the census-key grammar in [akn/SKILL.md](../../../.claude/skills/akn/SKILL.md)) and the código status enum `"in-force"`/`"repealed"`/`"superseded-in-progress"`. Everything else the codebase stores or displays stays Spanish.

## No Comments

Never add comments to Go code. No doc comments, no inline comments, no commented-out code. The code speaks for itself.

## Closed Dispatches

A switch or map lookup over a closed set, an akn enum such as `akn.EdgeLabel` or a table the binary carries, has two halves that ship together. The dispatch ends in a trailing panic naming the table the value fell outside of, so a value off the set stops the program at the line that met it instead of flowing on as a silent default.

```go
panic(fmt.Sprintf("<package>: <Function> carries <type> %q, not one of <package>.<TABLE>", value))
panic(fmt.Sprintf("store: edge %s->%s carries akn.EdgeLabel %q, not one of akn.EDGE_LABELS", from, to, label))
```

Beside it sits a totality test that ranges over the exported value slice and drives every member through the dispatch, so a member added to the slice without its arm fails the test before it can reach the panic in production.

```go
for _, label := range akn.EDGE_LABELS {
	dispatch(label)
}
```

A `json.Marshal` over a value the code built itself, a struct, slice or map of plain fields, cannot fail, so its error branch is dead code. It goes through `jsonx.MustMarshal`, which returns the bytes and panics naming the value's type if the encoder ever refuses one.

```go
before   body, error := json.Marshal(embed_request{IDs: ids, Texts: texts})
         if error != nil {
             return nil, error
         }
after    body := jsonx.MustMarshal(embed_request{IDs: ids, Texts: texts})
```
