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

## Step 2 — Scaffold the chosen starter

After the user chooses, follow the corresponding workflow:

- **TypeScript** → @.claude/skills/new-project/references/typescript.md
- **Go** → @.claude/skills/new-project/references/golang.md

If the user describes something outside these options, acknowledge the choice, ask clarifying questions, and proceed without a predefined starter.

## Step 3 — Post-scaffold: offer architecture guide

After scaffolding is complete, ask:

> Would you also like to apply an architecture guide to this project?
>
> - **Backend (Clean Architecture)** — layers: Domain, Application, Infrastructure, Presentation
> - **Frontend (Next.js Modular)** — module-based layout with ports & adapters
>
> Or skip if you prefer to design the architecture yourself.

Only apply an architecture skill if the user confirms.
