# Go Scaffold

## Confirmation summary

Before creating any files, present this to the user:

> I'll set up the following:
> - **Go version**: latest stable (via `.go-version`)
> - **Layout**: `cmd/` for entry points, `internal/` for private packages
> - **Tooling**: `golangci-lint`
> - **Makefile**: `build`, `run`, `lint`, `test`, `tidy` targets
>
> Confirm to proceed?

Only scaffold after the user confirms. If they decline, ask what they'd prefer instead.

## Step 1 — Clarify minimal requirements

Ask only if unclear:
- **Project type**: "CLI tool", "HTTP service", or "library"?
- **Module path**: e.g. `github.com/user/project-name`

## Step 2 — Verify Go installation

- Run `go version` to check availability.
- Create `.go-version` with the chosen version string (e.g. `1.23`).

## Step 3 — Initialize Go module

```bash
go mod init <module-path>
```

Never overwrite an existing `go.mod` — extend it instead.

## Step 4 — Create project layout

For a **service or CLI**:

```
cmd/<project-name>/main.go
internal/app/
```

For a **library**:

```
<package-name>.go
internal/
```

Default `cmd/<project-name>/main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello from Go!")
}
```

Do not introduce `handlers`, `services`, or `repositories` — the user designs the architecture.

## Step 5 — Add golangci-lint

```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

Create `.golangci.yml`:

```yaml
linters:
  enable:
    - gofmt
    - govet
    - errcheck
    - staticcheck
    - unused

linters-settings:
  gofmt:
    simplify: true

issues:
  exclude-use-default: false
```

## Step 6 — Add `Makefile`

```makefile
BINARY_NAME=app
MAIN_PATH=./cmd/$(BINARY_NAME)

.PHONY: build run lint test tidy

build:
	go build -o bin/$(BINARY_NAME) $(MAIN_PATH)

run:
	go run $(MAIN_PATH)

lint:
	golangci-lint run ./...

test:
	go test ./... -v -race

tidy:
	go mod tidy
```

## Step 7 — Add `.gitignore`

```
bin/
*.out
*.test
```

## Step 8 — Optional (only on request)

- **Air** for hot reload.
- **Testify** for assertion helpers.
- **godotenv** for `.env` loading.
