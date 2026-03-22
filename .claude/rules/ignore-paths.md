---
name: ignore-paths
description: Paths and files that must never be read, searched, or included in any analysis. Applied globally to reduce token usage.
---

# Ignored Paths

Never read, glob, grep, or include the content of the following paths in any response or analysis. Skip them silently without mentioning their existence.

## Dependencies and package caches

- `node_modules/`
- `.yarn/`
- `.pnp.*`
- `vendor/`
- `go/pkg/`

## Build outputs

- `dist/`
- `build/`
- `out/`
- `.next/`
- `.nuxt/`
- `.output/`
- `bin/`
- `coverage/`
- `*.out`
- `*.test` (Go binary)

## Generated files

- `*.generated.ts`
- `*.generated.go`
- `*.pb.go`
- `*.pb.ts`
- `__generated__/`
- `.prisma/`
- `prisma/migrations/` (read only when explicitly asked)

## Tooling and IDE internals

- `.git/`
- `.idea/`
- `.vscode/`
- `*.log`
- `*.lock` (read only when explicitly asked about dependency versions)

## OS artifacts

- `.DS_Store`
- `Thumbs.db`
- `desktop.ini`

## Environment and secrets

- `.env`
- `.env.*`
- `*.pem`
- `*.key`
- `*.p12`
- `secrets.*`
