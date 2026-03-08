---
name: typescript-project-starter
description: Scaffold a minimal, conventional TypeScript project using Yarn, tsconfig with a baseUrl/paths alias so that `@` points to `src`, and a basic src structure and scripts, while leaving framework and architecture decisions to the user. Use when starting a new TypeScript project from scratch and you want a simple, customizable base with an `@` → `src` alias.
---

# TypeScript Project Starter

## Purpose

This skill helps set up a **small, clean TypeScript project skeleton** without imposing a specific application architecture or framework.

The goal is:
- **Do**: Initialize Yarn, TypeScript, a basic `src` layout, configure `@` to point to `src` via `tsconfig.json`, add a minimal ESLint setup, and add common scripts.
- **Do not**: Choose frameworks (React, Express, etc.) or high-level architecture unless the user explicitly asks.

## When to Apply This Skill

Use this skill when:
- The user says they want to **start a TypeScript project from scratch**.
- They want a **simple or minimal structure**, and prefer to design the architecture themselves.
- They want to use **Yarn** as the package manager (classic Yarn, not necessarily Plug'n'Play).

Avoid using this skill when:
- The user asks for a **framework-specific starter** (e.g. Next.js, NestJS); follow framework docs instead.
- The project is part of an existing monorepo with its own conventions.

## Workflow

Follow this workflow end to end.

### 1. Clarify minimal requirements (without over-questioning)

Ask only what is necessary:
- **Runtime target** (default to Node if unclear):
  - "Node script / backend", "library", or "browser app"?
- **Package manager**:
  - Default to **Yarn** for this skill. Only switch to another manager if the user explicitly asks.

If the user is vague, assume:
- **Node LTS** runtime (latest available LTS major)
- **Yarn** as package manager
- Single-package project (no monorepo).

### 2. Initialize the project

In the project root:
- Ensure **NVM** is available (this skill assumes NVM is installed and used to manage Node versions).
- Decide on the exact **Node LTS version** to use:
  - Prefer a specific LTS major or version string (for example: `20` or `v20.11.0`), either provided by the user or chosen as the latest LTS.
- Configure `.nvmrc`:
  - If `.nvmrc` does not exist, create it in the project root with the chosen Node LTS version string as its only content.
  - If `.nvmrc` already exists and clearly points to a non-LTS version, suggest updating it to the desired LTS version.
- Use NVM to install/use that version:
  - Run `nvm install` (reads `.nvmrc`) and then `nvm use`.

- If there is no `package.json`, run:
  - `yarn init -y`

Never overwrite an existing `package.json` without checking; if one exists, **extend** it instead.

### 3. Install TypeScript and basic tooling

Add core dev dependencies (for Node-oriented projects):
- `typescript`
- `@types/node`
- `eslint`
- `@typescript-eslint/parser`
- `@typescript-eslint/eslint-plugin`

Optional (only if the user wants them):
- `ts-node` or `tsx` for running TS directly

Example with Yarn (Node default):
- `yarn add -D typescript @types/node eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin`
- Optionally (if user wants easy execution): `yarn add -D tsx`

### 4. Create a minimal `tsconfig.json`

If `tsconfig.json` does not exist, create a **simple, strict** config.

For Node 18+ default:

```json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": "src",
    "paths": {
      "@/*": ["*"]
    }
  },
  "include": ["src"]
}
```

This config:
- Makes `src` the **root** of the TypeScript sources.
- Sets `baseUrl` to `src` and defines a `paths` alias so that you can import modules using the `@` prefix, for example:
  - `import { something } from "@/utils/something";`
  - which resolves to `src/utils/something`.

Note: some runtimes or bundlers need extra configuration to honor `tsconfig` path aliases at runtime. This skill only sets up the TypeScript side; the user may configure bundlers or runtime helpers (like `tsconfig-paths` or bundler-specific alias options) later as they design the architecture.

For browser-oriented projects, adjust only if the user asks (e.g. `module: "ESNext"`, appropriate `lib` settings).

### 5. Create a simple folder and entry file structure

Create the minimal structure:
- `src/`
- `src/index.ts` (or another entry file if the user specifies)

Default content for `src/index.ts` should be **neutral** and non-opinionated, for example:

```ts
export function main() {
  console.log("Hello from TypeScript!");
}

if (require.main === module) {
  main();
}
```

Do **not** introduce additional layers like `controllers`, `services`, or `modules` unless the user asks. The user is responsible for designing the domain architecture.

### 6. Add useful npm scripts (minimal)

Update `package.json` scripts to include at most:

- `build`: `tsc`
- `start`: a simple way to run compiled code or a dev runner

For a Node project using `dist/index.js`:

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

If the user wants direct TS execution with `tsx`:

```json
{
  "scripts": {
    "dev": "tsx src/index.ts"
  }
}
```

Also add a `lint` script for ESLint:

```json
{
  "scripts": {
    "lint": "eslint \"src/**/*.{ts,tsx}\""
  }
}
```

Avoid adding additional formatting or testing scripts unless the user explicitly requests them, or you have already confirmed they want "basic tooling".

### 7. Add a minimal ESLint configuration

After dependencies are installed, always add a **basic TypeScript-aware ESLint setup**:

- Create a `.eslintrc.cjs` file in the project root with content similar to:

```js
module.exports = {
  root: true,
  env: {
    es2021: true,
    node: true,
  },
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module",
  },
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
  ],
  ignorePatterns: ["dist"],
};
```

This configuration is intentionally minimal:
- It lints TypeScript files using recommended ESLint and `@typescript-eslint` rules.
- It ignores the compiled `dist` output.
- It does **not** enforce any project-specific style or architecture; the user can extend rules later.

### 8. Optional: Prettier, tests (only on request)

If the user asks for **additional quality tooling**, you may:

- Add **Prettier** with a default configuration.
- Add a **test runner** (e.g. Vitest or Jest) if they mention tests.

Keep these choices lightweight and do not enforce any test structure beyond a simple `tests/` or `__tests__/` folder and one example test.

If the user does **not** ask for them, skip this step.

### 9. Verify and explain what was created

After scaffolding:

- List the created or modified files (e.g. `package.json`, `tsconfig.json`, `src/index.ts`).
- Explain briefly:
  - **What** each file is for.
  - That the **architecture is intentionally minimal**, and they are free to design modules, folders, and patterns as they prefer.
- Optionally, suggest **next steps** in 1–2 sentences (e.g. "create a `src/app` module", or "set up routes/controllers") but do not implement them unless the user asks.

## Examples

### Example 1: New Node CLI tool

User: "Create a simple TypeScript project from scratch for a small CLI that I'll design later."

Agent should:
- Assume Node 18+ and Yarn.
- Run `yarn init -y`.
- Install `typescript @types/node` as dev dependencies using `yarn add -D`.
- Create `tsconfig.json` as above.
- Create `src/index.ts` with a basic `main()` and console log.
- Add `"build"` and `"start"` scripts.
- Tell the user what was created and that they can extend the structure as they design the CLI.

### Example 2: Browser library skeleton

User: "I want a minimal TypeScript library for the browser, I'll handle bundling and architecture."

Agent should:
- Ask for confirmation that bundling (Vite/Rollup/Webpack) will be handled separately.
- Initialize Yarn if needed.
- Install `typescript` only (no `@types/node` unless they request node-specific APIs) using `yarn add -D typescript`.
- Create `tsconfig.json` tuned for browser (e.g. `module: "ESNext"`, appropriate `target` and `lib`).
- Create `src/index.ts` that exports a placeholder function or object.
- Avoid picking any bundler or test framework unless the user asks.

