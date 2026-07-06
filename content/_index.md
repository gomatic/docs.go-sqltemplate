---
title: go-sqltemplate
---
**A pure-Go library that rewrites a SQL template into a parameterized query, turning `{{name}}` variables into ordered `$1, $2, …` bind placeholders (with their values returned alongside) and substituting `{{.name}}` variables verbatim into the SQL text.** It performs only string and `text/template` work — it never touches a database or driver.

- **Source:** [gomatic/go-sqltemplate](https://github.com/gomatic/go-sqltemplate)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-sqltemplate](https://pkg.go.dev/github.com/gomatic/go-sqltemplate)

## Install

```sh
go get github.com/gomatic/go-sqltemplate
```

## Usage

A statement carries two kinds of variables:

- `{{name}}` — rewritten into ordered bind placeholders (`$1`, `$2`, …) whose values are returned in `Result.Bindings`. Identical values are deduplicated to a single placeholder, and bind values reach the driver untouched (the `$N` placeholder makes them injection-safe regardless of content).
- `{{.name}}` — substituted verbatim into the statement, for composing trusted SQL fragments such as a sub-query source.

`Parameterize` renders a `Statement` against `Params` into a `Result`:

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-sqltemplate"
)

func main() {
	result, err := sqltemplate.Parameterize(
		"select * from ({{.source}}) as s where name={{name}}::text and value={{value}}::text",
		sqltemplate.Params{
			"source": "select 1",
			"name":   "abc",
			"value":  "123",
		},
	)
	if err != nil {
		panic(err)
	}

	fmt.Println(result.SQL)      // select * from (select 1) as s where name=$1::text and value=$2::text
	fmt.Println(result.Bindings) // [abc 123]
}
```

`Result` is JSON-serializable:

```go
type Result struct {
	SQL      Query   `json:"sql"`
	Bindings []Value `json:"bindings"`
}
```

Pass `Result.SQL` and `Result.Bindings` to your driver's query call — the library produces the text and the ordered values but does not execute anything.

### Behavior

- **Bind values are never mutated.** `O'Brien`, unicode, empty strings, and injection-shaped payloads (`'; DROP TABLE users; --`) all reach the driver byte-for-byte behind a `$N` placeholder; none of their characters leak into the SQL text.
- **Values are deduplicated by content.** `select {{a}}, {{b}}, {{c}}` with `a` and `b` equal renders `select $1, $1, $2`. A name referenced twice (`select {{a}}, {{a}}`) likewise collapses to one binding.
- **Bindings are allocated lazily.** A valid but unreferenced param produces no placeholder — only variables the template actually emits are bound.
- **Invalid names are dropped.** Empty names, names longer than 30 characters, and internal names prefixed with `.` or `_` are rejected. A dropped name that the statement references leaves its bind unprovided, so `Parameterize` returns `ErrInvalidStatement`.
- **Missing statics are preserved.** An unprovided `{{.name}}` survives rendering unchanged, so the statement can be parameterized again later.

### Normalize

`Normalize` collapses runs of whitespace in a statement to single spaces (`Parameterize` applies it internally):

```go
sqltemplate.Normalize("select\t1\n  from  t") // "select 1 from t"
```

### Errors

Every failure `Parameterize` returns wraps the `ErrInvalidStatement` sentinel, matchable with `errors.Is`, while keeping the underlying cause recoverable with `errors.As`:

```go
_, err := sqltemplate.Parameterize("select {{missing}}", nil)
if errors.Is(err, sqltemplate.ErrInvalidStatement) {
	// handle an unparseable or unrenderable statement
}
```

## Design

Bind variables (`{{name}}`) resolve through a `text/template` function map: each name is a function that, when the template emits it, allocates or reuses an ordered `$N` placeholder and appends the value to the bindings slice. This is what makes deduplication content-based and binding allocation lazy — a name only binds when the template actually calls it.

Static variables (`{{.name}}`) are substituted before template execution. Provided statics are inserted verbatim (with `;`, `'`, and `"` stripped as a backstop); unprovided statics are marked, carried through execution, and restored to `{{.name}}` afterward.

Verbatim `{{.name}}` values must be **trusted** fragments. The character strip is only a backstop, not a defense against injection — never feed untrusted input through `{{.name}}`; use a `{{name}}` bind placeholder instead.
