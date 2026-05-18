# go-make-examples

Keep GitHub markdown in sync with real, compilable Go example code.

This tool scans a README (or any markdown file) for `[example]` link-reference directives, extracts code from Go source files on disk, and rewrites the fenced `go` code blocks that follow each directive. The examples you point at should live in `Example*` functions (typically in `example_test.go`) so `go test` compiles and runs them—documentation stays honest because broken examples fail CI.

## Installation

```bash
go install github.com/kellegous/go-make-examples@latest
```

Or run it without installing:

```bash
go run github.com/kellegous/go-make-examples@latest README.md
```

## Usage

```text
 go-make-examples <readme-file>
```

The tool reads `<readme-file>`, updates every embedded example in place, and writes the file back. Paths in directives are resolved relative to the markdown file’s directory.

### Makefile

A common pattern is a `docs` or `update-readme` target that regenerates the README before commit:

```makefile
.PHONY: docs

docs:
	go run github.com/kellegous/go-make-examples@latest README.md
```

## Markdown directives

Place a link-reference comment immediately above the code block you want filled in. The ref names a Go file, optionally followed by a function:

    [example]: # "example_test.go:ExampleConn_SendTextMessage"

    ```go
    // filled in by update-readme from example_test.go
    ```

| Ref format           | What gets embedded                                                     |
| -------------------- | ---------------------------------------------------------------------- |
| `file.go`            | Entire file, `go/format`-ted                                           |
| `file.go:ExampleFoo` | Body of `ExampleFoo`, dedented, plus any imports used in that function |

Subdirectories work the same way: `pkg/example_test.go:ExampleFoo`.

On each run, the tool keeps the directive line, skips any existing code block after it, and writes a fresh ` ```go ` block. You normally commit both the directive and the generated block so GitHub renders correctly without running the tool.

## Writing examples

Use standard Go [example functions](https://pkg.go.dev/testing#hdr-Examples) in a `*_test.go` file. The function name after the colon in the directive must match.

```go
package mypkg_test

import (
	"fmt"
	"log"

	"github.com/example/mypkg"
)

func ExampleConn_SendTextMessage() {
	conn, err := mypkg.Connect()
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	fmt.Println("done")
	// Output: done
}
```

When the tool extracts `ExampleConn_SendTextMessage`, it includes only the function body (not the `func` line or braces) and adds an `import (...)` block for packages referenced in that body—matching what readers expect in a README snippet.

Run tests to verify examples compile and behave as documented:

```bash
go test ./...
```

## Workflow

1. Add or edit `Example*` functions in `example_test.go` (or other `.go` files).
2. Add `[example]: # "..."` directives in your README where snippets should appear.
3. Run `go-make-examples README.md` (or `make docs`).
4. Review the diff, then commit the markdown and test files together.

## How it works

The directive syntax is a [link reference definition](https://spec.commonmark.org/0.31.2/#link-reference-definitions): `[example]: # "ref"`. That keeps the marker valid markdown and invisible on GitHub except as the anchor for the following code block.

For function-level refs, the tool parses the Go AST, extracts and formats the function body, dedents one level, and collects imports used via selector expressions (`pkg.Name`) in that body.
