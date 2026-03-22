---
name: unit-testing
description: Guide for writing unit tests. Use when the user wants to add, structure, or review unit tests in a project. Covers frontend (Vitest + React Testing Library) and backend (TypeScript with Vitest, or Go with the standard testing package + testify).
---

# Unit Testing

## Step 1 — Identify the context

Before writing any tests, determine the project context:

- **Frontend** (Next.js, React components, hooks, services) → @.claude/skills/unit-testing/references/frontend.md
- **Backend TypeScript** (Clean Architecture: domain, use cases, controllers) → @.claude/skills/unit-testing/references/backend/typescript.md
- **Backend Go** (Clean Architecture: domain, use cases, handlers) → @.claude/skills/unit-testing/references/backend/golang.md

If unclear, ask:

> Is this for a frontend (React/Next.js) or backend project? If backend, are you using TypeScript or Go?

## Step 2 — Follow the reference for the identified context

Load and apply the corresponding reference. Do not mix patterns across contexts.

## Step 3 — General rules (apply to all contexts)

These rules apply regardless of language or layer:

- **Test behavior, not implementation.** A test should break when observable behavior changes, not when internal code is refactored.
- **One scenario per test.** Each `it` / `func Test` block covers a single case. Use table-driven tests (Go) or parameterized tests for multiple inputs.
- **Fakes over mocks.** For dependency inversion ports, use hand-written in-memory fakes that implement the interface. Reserve spies/mocks for verifying call behavior on collaborators within the same layer.
- **Independent tests.** No test depends on the execution order of another. Reset all state in `beforeEach` / at the start of each test function.
- **Separate unit from integration.** Tests that require a real database, network, or filesystem are integration tests and must not run in the default unit test suite.
