# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Template repository storing reusable AI skills for guided development workflows.

## Skill File Format

Each `SKILL.md` uses YAML frontmatter followed by Markdown:

```markdown
---
name: skill-name
description: One-line description used by the AI to decide when to load this skill.
---
```

The `description` field determines when the skill auto-loads based on prompt relevance.

## Available Skills

### Project Bootstrap

- **`new-project`** — Lists available starters (TypeScript, Go), asks which to use, confirms before creating any files, and optionally applies an architecture guide at the end. Invoke with `/new-project`.

### Architecture Guides (loaded on demand by project type)

- **`backend-architecture`** — Clean Architecture for backend/API projects.
- **`frontend-architecture`** — Next.js modular architecture with dependency inversion for drivers and adapters.

### Testing

- **`unit-testing`** — Unit testing guide. Identifies context (frontend, backend TS, backend Go) and applies the corresponding reference: Vitest + RTL for frontend, Vitest for backend TS, Go `testing` + testify for backend Go.
