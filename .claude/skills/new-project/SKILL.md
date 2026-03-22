---
name: new-project
description: Start a new project from scratch. Use when the user wants to create or bootstrap a new project. Lists available language starters, asks which one to use, confirms before creating any files, and optionally applies an architecture guide at the end.
---

# New Project

## Purpose

Guide the user through bootstrapping a new project. Always ask before creating any files.

## Step 1 — Present available starters

Always present the options first. Do not assume a language or stack.

> I can help you bootstrap a new project. Which starter would you like to use?
>
> | # | Starter | What it sets up |
> |---|---------|-----------------|
> | 1 | **TypeScript** | Yarn, strict `tsconfig.json`, `@` → `src` alias, ESLint, `build`/`start`/`lint` scripts |
> | 2 | **Go (Golang)** | Go modules, `cmd/`+`internal/` layout, `golangci-lint`, Makefile |
>
> Reply with the number or name, or describe something different if none of these fit.

## Step 2 — Confirm before scaffolding

After the user chooses, present a summary of what will be created and ask for confirmation:

**TypeScript:**
> I'll set up the following:
> - **Package manager**: Yarn
> - **TypeScript**: strict mode, `@` → `src` path alias
> - **Tooling**: ESLint with `@typescript-eslint`
> - **Scripts**: `build` (tsc), `start`, `lint`
>
> Confirm to proceed?

**Go:**
> I'll set up the following:
> - **Go version**: latest stable (via `.go-version`)
> - **Layout**: `cmd/` for entry points, `internal/` for private packages
> - **Tooling**: `golangci-lint`
> - **Makefile**: `build`, `run`, `lint`, `test`, `tidy` targets
>
> Confirm to proceed?

Only scaffold after the user confirms. If they decline, ask what they'd prefer instead.

## Step 3 — TypeScript scaffold

Follow this workflow if the user chose TypeScript.

### 3.1 Clarify minimal requirements

Ask only if unclear:
- **Runtime target**: "Node script / backend", "library", or "browser app"? (default: Node)
- **Package manager**: default Yarn; switch only if user asks.

If vague, assume: Node LTS, Yarn, single-package project.

### 3.2 Initialize the project

- Check NVM is available.
- Decide Node LTS version (e.g. `22`). Create `.nvmrc` with the version string.
- Run `nvm install` then `nvm use`.
- If no `package.json`: `yarn init -y`. Never overwrite existing — extend it.

### 3.3 Install dependencies

```bash
yarn add -D typescript @types/node eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
# only if user wants direct TS execution:
yarn add -D tsx
```

### 3.4 Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "es2015",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

The `@` alias resolves to `src/`. Inform the user that some runtimes need `tsconfig-paths` or a bundler alias config to honor this at runtime.

For browser-oriented projects, adjust `module` and `lib` only if asked.

### 3.5 Create folder and entry file

```
src/index.ts
```

```ts
export function main() {
  console.log("Hello from TypeScript!");
}

if (require.main === module) {
  main();
}
```

Do not introduce `controllers`, `services`, or `modules` — the user designs the architecture.

### 3.6 Add scripts to `package.json`

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "lint": "eslint \"src/**/*.{ts,tsx}\""
  }
}
```

If the user wants hot reload: `"dev": "tsx watch src/index.ts"`.

### 3.7 Add `.eslintrc.cjs`

```js
module.exports = {
  root: true,
  env: { es2021: true, node: true },
  parser: "@typescript-eslint/parser",
  parserOptions: { ecmaVersion: "latest", sourceType: "module" },
  plugins: ["@typescript-eslint"],
  extends: ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
  ignorePatterns: ["dist"],
};
```

### 3.8 Optional (only on request)

- **Prettier**: add with default config.
- **Tests**: add Vitest or Jest with a basic `tests/` folder.

---

## Step 4 — Go scaffold

Follow this workflow if the user chose Go.

### 4.1 Clarify minimal requirements

Ask only if unclear:
- **Project type**: "CLI tool", "HTTP service", or "library"?
- **Module path**: e.g. `github.com/user/project-name`

### 4.2 Verify Go installation

- Run `go version` to check availability.
- Create `.go-version` with the chosen version string (e.g. `1.23`).

### 4.3 Initialize Go module

```bash
go mod init <module-path>
```

Never overwrite an existing `go.mod` — extend it instead.

### 4.4 Create project layout

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

### 4.5 Add golangci-lint

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

### 4.6 Add `Makefile`

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

### 4.7 Add `.gitignore`

```
bin/
*.out
*.test
```

### 4.8 Optional (only on request)

- **Air** for hot reload.
- **Testify** for assertion helpers.
- **godotenv** for `.env` loading.

---

## Step 5 — Post-scaffold: offer architecture guide

After scaffolding is complete, ask:

> Would you also like to apply an architecture guide to this project?
>
> - **Backend (Clean Architecture)** — layers: Domain, Application, Infrastructure, Presentation
> - **Frontend (Next.js Modular)** — module-based layout with ports & adapters
>
> Or skip if you prefer to design the architecture yourself.

Only apply an architecture skill if the user confirms.
